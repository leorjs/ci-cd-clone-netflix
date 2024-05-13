# CI/CD de Netflix sobre Kubernetes



→ Creación, configuración de las  EC2 que necesitaremos

→ Configuración API TMDB

→ Implementacion Security con Sonarqube y Trivy

→ Instalación, configuración e implementación de Jenkins (Con Pipeline)

→ Instalación, configuración e implementacion de Monitoreo > Prometheus & Grafana

→ Instalación, configuración del Node Exponer

→ Creación y configuración del Cluster-EKS

→ Instalación y configuración de helm

→ Instalación, configuración e implementacion de ArgoCD


![netflix-kubernetes (1)](https://github.com/leorjs/ci-cd-clone-netflix/assets/119978221/bcef6d6a-0703-4810-80fc-ed3fcdeb38f6)


## Pasos previos para ahorrar tiempo

👉 Tener creado la VPC

→ Agrear al Sg (Security Group) los siguiente port: 22, 9000, 8081, 9090, 9001, 30007, 9100, 3000
 
→ Estar registrado en las siguientes paginas:
		- https://www.themoviedb.org/
		- https://hub.docker.com/


## Stack 

→ Git / GitHub

→ Jenkins

→ Sonarqube

→ trivy

→ Docker

→ AWS / EC2

→ Kubernete

→ Grafana

→ Prometheus

→ argoCD

→ helm

→ aws-cli

### Paso 1 → Creación EC2 en AWS (t2.large)

1.- Vamos a nuestra consola de AWS y buscamos el servicio de EC2

+ Launch Instance
+ Name and tags: netflix-ec2
+ OS: Ubuntu >> versión 24.04 o 22.04
+ Instance type: t2.large
+ Key pair >> Agrega el key previamente creado (O esta la opción que la puedes crear y descargar)

→ Networking Setting → Edit:

+ VPC >> Agregar la VPC previamente creada
+ Subnet >> Agregar alguna subnet PUBLIC
+ Auto-assing public IP >> Enable
+ Select existing security group >> Seleccionar el SG previamente creado
+ Configure storage >> 25 GB >> gp2
   
→ Launch Instance.
	

![](https://miro.medium.com/v2/resize:fit:700/1*QW0DVqBqj-Bk72rVQFEf5w.png)

1.2-  Se puede conectar de dos maneras, elige cual te sientas a gusto:

→ Vía consola web desde AWS
  
+ Tilda la instance
+ En la parte superior derecha le das connect
+ Y en la primera opción: EC2 Instance connect >> Le das conectar la cual te lleva a otra pagina directo al ssh de la EC2
		
→ Vía terminal desde tu PC

+ Vas al terminal y te posicionas donde esta tu .pem que previamente descargaste
+ Conectate con el siguiente comando: $ `ssh -i <key.pem> ubuntu@<IP-PUBLIC>`


![](https://miro.medium.com/v2/resize:fit:700/1*3yKVCmrCXMbAeB0WjpHsvA.png)


### Paso 2 → Configuración de la EC2

Al momento de ingresar a nuestra EC2 comenzaremos una sería de instalaciones y configuraciones

→ $ `sudo apt update -y`
→ Vamos a clonar el repositorio de Netflix:

+ `git clone https://github.com/Aakibgithuber/Deploy-Netflix-Clone-on-Kubernetes`


Nota: Es importante que al crear la VPC creen un elastic IP Address y lo asocian a la instancia

> Buscar si la VPC crea la IP Elastic o investigar para agregar a la doc como se crea y se asocia a la instancia



2.1- Instalación docker sobre la EC2

→ $ `apt-get install docker.io` 
→ $ `usermod -aG docker ubuntu` 
→ $ `newgrp docker`
→ $ `sudo chmod 777 /var/run/docker.sock`

2.2- Configuración imagen docker sobre la EC2

→ $ `docker build -t netflix .`
→ $ `docker run -d — name netflix -p 8081:80 netflix:latest`

> Nota: Si no agregaste el port 8081 al SG este es el momento indicado asi vamos a probar nuestra web de Netflix 
<br>
<br>

![](https://miro.medium.com/v2/resize:fit:700/1*1NLxGqchmGxanD0QBp4uGw.png)

Vamos a ingresar a nuestro Netflix --> `http://<IP-PUBLIC>:8081`

![](https://miro.medium.com/v2/resize:fit:700/1*coo7bSMDThDciP68DM7IDg.png)

**Observaran la pagina sin datos ya que más adelante estaremos configurando y consumiendo la API que tiene la data**

<br>
API → (Interfaz de Programación de Aplicaciones) es un conjunto de reglas y especificaciones que permiten que diferentes aplicaciones de software interactúen entre sí. Facilita la comunicación y la integración de sistemas, permitiendo que un software utilice funcionalidades o datos de otro, de manera segura y controlada. Las APIs son fundamentales para el desarrollo moderno de aplicaciones, especialmente en arquitecturas de microservicios y plataformas en la nube
<br>
### Paso 3 → Configuración API (TMDB) para Netflix

→ Primero debe registrarte en la siguiente web: https://www.themoviedb.org/
→ Luego de estar registrador iremos a la sección → Ajustes
→ En la parte izquierda de Configuraciones vamos a la opción → API
→ Allí te va aparecer la opción de generar nueva api key
→ Copiamos la Clave de la API para más adelante usarla


![[Pasted image 20240509213426.png]]


### Paso 4 → Generar nueva Imagen con Key APi

Para este paso vamos eliminar y generar de nuevo la image en docker de Netflix pero con el API que creamos en TMDB, aplicar los siguientes comandos:


→ $ docker stop netflix 
→ $ docker rmi -f netflix

Ahora crearemos la nueva image con la API-KEY

> Recuerden siempre dejar el punto al final del comando

→ docker build -t netflix:latest --build-arg TMDB_V3_API_KEY=your_api_key .    
→ docker run -d -p 8081:80 netflix

Ya estaría listo nuestro netflix clonado consumiendo data desde TMDB, prueben conectandose con la `http://<IP-Public-EC2>:8081`

![[Pasted image 20240509214049.png]]



### Paso 5 → Implementación de security con Sonarqube & trivy

**sonarqube** → Es una plataforma de inspección continua de calidad de código que realiza análisis estático para detectar bugs, vulnerabilidades y malas prácticas en el código fuente. Proporciona herramientas de seguimiento del código a lo largo del tiempo para mejorar su mantenibilidad y adherirse a estándares de calidad.
<br>
**trivy** → Es una herramienta de escaneo de vulnerabilidades sencilla y completa, diseñada para detectar vulnerabilidades en contenedores y otros artefactos como imágenes de contenedores y sistemas de archivos. Es fácil de usar, rápida y puede integrarse en los procesos de CI/CD para la detección temprana de problemas de seguridad.
<br>
5.1- Configuraremos sonarqube sobre nuestra EC2

Vamos a correr el siguiente comando pero antes recuerda agregar el port 9000 a tu SG (security group)

→ $ docker run -d --name sonar -p 9000:9000 sonarqube:lts-community

![](https://miro.medium.com/v2/resize:fit:700/1*O3lEgiygLbgelwif4fJ7HA.png)

5.2- En la imagen siguiente se observa el port agregado en el SG

![](https://miro.medium.com/v2/resize:fit:700/1*oky6T792r0b6ZTxeI_IkEw.png)
<br>
Ahora prueben entrando a sonarqube >>  `<http://IP-PUBLIC-EC2:9000>` y veremos que sonarqube nos pedirá las credenciales

```
username = admin  
password = admin
```

![](https://miro.medium.com/v2/resize:fit:700/1*knLNWz6rcsu2iV-r0gYAxg.png)



5.3.- Ahora instalaremos trivy

→ $ `sudo apt-get install wget apt-transport-https gnupg lsb-release`

→ $ `wget -qO — [https://aquasecurity.github.io/trivy-repo/deb/public.key](https://aquasecurity.github.io/trivy-repo/deb/public.key) | sudo apt-key add -`

→ $ `echo deb [https://aquasecurity.github.io/trivy-repo/deb](https://aquasecurity.github.io/trivy-repo/deb) $(lsb_release -sc) main | sudo tee -a /etc/apt/sources.list.d/trivy.list`

→ $ `sudo apt-get update`

→ $ `sudo apt-get install trivy`
<br>
Ya estaría listo trivy para chequear y escanear nuestra imagen de cualquier vulnerabilidad, por ejemplo:


→ $ `trivy image <image id>`

![](https://miro.medium.com/v2/resize:fit:700/1*wxTmnkIjRWgpeq0u9CZ2ug.png)



### Paso 6 → Instalación de Jenkins

Para instalar la tools de Jenkins vamos a necesitar jdk17 y luego instalar la tools

6.1.- Instalación del java

→ $ `sudo apt update`
→ $ `sudo apt install fontconfig openjdk-17-jre`
→ $ java -version  
    **output →**openjdk version “17.0.8” 2023–07–18  
    OpenJDK Runtime Environment (build 17.0.8+7-Debian-1deb12u1)  
    OpenJDK 64-Bit Server VM (build 17.0.8+7-Debian-1deb12u1, mixed mode, sharing)
<br>
6.2.- Ahora si vamos a instalar **Jenkins**

→ $ ```sudo wget -O /usr/share/keyrings/jenkins-keyring.asc \
https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key```

→ $ ```echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \  
    https://pkg.jenkins.io/debian-stable binary/ | sudo tee \  
    /etc/apt/sources.list.d/jenkins.list > /dev/null```
    
→ $ `sudo apt-get update`
→ $ `sudo apt-get install jenkins`
→ $ `sudo systemctl start jenkins`
→ $ `sudo systemctl enable jenkins`
<br>
Ya podemos acceder a nuestro **Jenkins** con nuestra IP-Public del EC2 y el port 8080 (Recuerden habilitar el por 8080 en su Security group)

> Cuando entren por primera vez a Jenkins le pediran una clave, desde la EC2 colocan el siguiente comando:
> $ sudo cat /var/lib/jenkins/secrets/initialAdminPassword
<br>
6.3.- Al momento de entrar a Jenkins le damos instalar plugins recomendado. Ya dentro de Jenkins nos iremos a mano izquierda en Install Plugins:

+ Eclipse Temurin Installer (Install without restart)
+ SonarQube Scanner (Install without restart)
+ NodeJs Plugin (Install Without restart)
+ Email Extension Plugin
+ owasp
+ Prometheus metrics →to moniter jenkins on grafana dashboard
+ Download all the docker realated plugins
<br>
![](https://miro.medium.com/v2/resize:fit:700/1*im2CuWRgQ3I6f172ZoAu3Q.png)
<br>
![](https://miro.medium.com/v2/resize:fit:700/1*0i-TmajKoUTtsTuiIcPcFw.png)

### Paso 7 → Agregaremos credenciales Sonarqube & Docker

7.1.- Vamos a generar el token para sonarqube para luego agregarlo en Jenkins

→ Vamos a la pagina de nuestro sonarqube >> `http://<IP-EC2-SONAR:9000>`
→ Vamos a la opción: Security → users → token → generate token
→ Copiamos el token y nos vamos a dirigir a Jenkins

![](https://miro.medium.com/v2/resize:fit:700/1*_WbX8seku9mACh7i2_GPZg.png)

![](https://miro.medium.com/v2/resize:fit:700/1*cCbvnTsI4arkvM1g0IRZLg.png)

→ Al momento de ingresar a Jenkins buscamos la opción 

+ manage jenkins → credentials → global → add credentials

→ Vamos a seleccionar: → Kind: select secret text

+ Scope: Global
+ Secret: `<El token que previamente copiamos en sonarqube>`
+ ID: sona-token
+ Description: OPCIONAL

→ Luego le damos create

7.2.- Configuracion proyect en sonarqube para Jenkins

+ Regresamos a nuestra page de sonarqube
+ Cliqueamos en la opción Projects → Create project
+ Project display name: Netflix
+ Project Key: Netflix
+ Main branch name: main
+ click sobre >> set up
<br>
![](https://miro.medium.com/v2/resize:fit:481/1*hdIL9FQ7AUNCiQaVyuIkbw.png)

→ Le damos la opción de Locally

![](https://miro.medium.com/v2/resize:fit:436/1*mDTEg7ouRggVn2YiISt3aA.png)

→ En la siguiente pantalla vamos a generar un token de 30 días

![](https://miro.medium.com/v2/resize:fit:560/1*x0Gdfg80S6X6Rn7iYXJrTg.png)
<br>
Al momento de darle generar token, nos arroja nuestro token que más adelante estaremos usando. Guardar dicho token
→ Continuar

![](https://miro.medium.com/v2/resize:fit:700/1*YodhXj4sgTdIIHdx96-jFg.png)


7.3.- Configuracion credenciales de Docker

+ Vamos a nuestro Jenkins → manage jenkins → credentials → global → add credentials
+ Acá agregamos nuestro username y password de nuestro dockerhub
+ ID: docker

![](https://miro.medium.com/v2/resize:fit:700/1*69WUqyaxAJ-z9g19DKg8yg.png)
<br>
7.4. Ahora vamos a configurar todas las herramientas en Jenkins

→ Manage Jenkins

+ Tools

→  Agregamos el jdk17

+ click sobre **add jdk** y seleccionamos **installer adoptium.net**
+ En este caso escogemos **jdk 17.0.8.1+** y en **name section enter jdk 17**

![](https://miro.medium.com/v2/resize:fit:700/1*Gy5i8mSpdse0jQSP774ASw.png)
<br>
→  Ahora le toca a  **node js**

+ click sobre  **add nodejs**
+ Enter node16 in name section
+ Elegimos la versión  **nodejs 16.2.0**

![](https://miro.medium.com/v2/resize:fit:700/1*1J-5u_PTtGZk7_eqzan0gw.png)
<br>
→  Ahora toca agregar a Docker

+ Click sobre **add docker**
+ Name: **docker**
+ add installer >> download from docker.com

![](https://miro.medium.com/v2/resize:fit:700/1*peEiZWYTLUMUSFKMRPUOpg.png)

→ Ahora le toca  sonarqube

+ add sonar scanner
+ name: sonar-scanner

![](https://miro.medium.com/v2/resize:fit:700/1*N0h9KMBQiSu1NUEquQWr3A.png)

→ Agregamos **owasp** dependency check

+ add dependency check
+ name: DP-Check
+ from add installer select install from github.com

![](https://miro.medium.com/v2/resize:fit:700/1*yvnR7d8iKxQKQcv6L7wr4Q.png)


7.5.- Configurar el global setting para sonarqube

→ Ir al →  manage jenkins → Configure global setting → add sonarqube servers

→ name: sonar-server**

→ server_url: http:<IP-EC2:9000>

→ server authentication token == sonar-token

![](https://miro.medium.com/v2/resize:fit:700/1*dlOfq0oJeC6GrLsAwnwO4Q.png)
<br>
7.6.- Acá vamos armar nuestro **Pipeline**

> Nota importante: Verificar los siguientes stage:
> stage("Docker Build & Push") >> Agregar tu usuario de docker y tu API correctamente
> stage('Deploy to Container') >> Agregar tu usuario de docker


>Agregar en la EC2 donde está instalado Jenkins

```
sudo usermod -aG docker jenkins  
sudo systemctl restart jenkins
```

→ Ir a la selección:  new item → select pipeline in the name section type **netflix**

→ Busca la opción **pipeline script** y copia & pega el siguiente codigo con las consideraciones que antes mencione.

```
pipeline {
    agent any

    tools {
        jdk 'jdk17'
        nodejs 'node16'
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }

    stages {
        stage('Clean Workspace') {
            steps {
                cleanWs() // Limpia el espacio de trabajo antes de empezar
            }
        }

        stage('Checkout from Git') {
            steps {
                git branch: 'main', url: 'https://github.com/Aakibgithuber/Deploy-Netflix-Clone-on-Kubernetes.git'
            }
        }

        stage("Sonarqube Analysis") {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh """
                        $SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectName=Netflix \
                        -Dsonar.projectKey=Netflix
                    """
                }
            }
        }

        stage("Quality Gate") {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token'
                }
            }
        }

        stage('Install Dependencies') {
            steps {
                sh "npm install" // Instala dependencias NPM
            }
        }

        stage('OWASP FS SCAN') {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }

        stage('TRIVY FS SCAN') {
            steps {
                sh "trivy fs . > trivyfs.txt" // Escanea el sistema de archivos con Trivy
            }
        }

        stage("Docker Build & Push") {
            steps {
                script {
                    docker.withRegistry('https://registry.hub.docker.com', 'docker-credentials-id') {
                        def customImage = docker.build("luckygold/netflix:latest", "--build-arg TMDB_V3_API_KEY=e057b10a36f63969d2ba1b6389c929ef .")
                        customImage.push() // Construye y sube la imagen Docker
                    }
                }
            }
        }

        stage("TRIVY") {
            steps {
                sh "trivy image luckygold/netflix:latest > trivyimage.txt" // Escanea la imagen Docker con Trivy
            }
        }

        stage('Deploy to Container') {
            steps {
                script {
                    // Asegúrate de que no hay conflictos con contenedores existentes
                    sh 'docker stop netflix-clone || true' // Detiene el contenedor si existe
                    sh 'docker rm netflix-clone || true'   // Elimina el contenedor si existe
                    sh 'docker run -d --name netflix-clone -p 8082:80 luckygold/netflix:latest' // Ejecuta el nuevo contenedor
                }
            }
        }
    }
}

```



→ Al momento de darle Build a tu proyecto de Netflix debes esperar el proceso y al final tendrás la siguiente pantalla

![[Pasted image 20240512022600.png]]



### Paso 8 → Monitoreo con Prometheus y grafana

**Prometheus**: Es un sistema de monitoreo y alerta de código abierto que recopila y almacena métricas en tiempo real a través de un modelo de datos multidimensional. Ofrece consultas potentes y alertas basadas en condiciones para evaluar la salud de aplicaciones e infraestructura.
<br>
<br>
**Grafana**: Es una plataforma de visualización y análisis que permite crear dashboards interactivos para visualizar, explorar y entender grandes cantidades de datos métricos, integrándose con múltiples fuentes de datos como Prometheus, para facilitar la interpretación y el análisis en tiempo real.

<br>
8.1.- Vamos a crear otra EC2 donde instalaremos las herramientas de monitoreo
<br>
→ Vamos a nuestro servicio de EC2 y creamos una nueva instance como ya lo hemos hecho
+ t2.medium
+ ubuntu

→ Minimos requisitos para instalar Prometheus que debe tener la nueva EC2

+ 2 CPU cores.
+ 4 GB of memory.
+ 20 GB of free disk space.

8.2.- Instalación Prometheus

→ sudo useradd --system --no-create-home --shell /bin/false prometheus
<br>
→ wget https://github.com/prometheus/prometheus/releases/download/v2.47.1/prometheus-2.47.1.linux-amd64.tar.gz

 → Extraemos los archivos de Prometheus y creamos directorios necesarios
 
+ tar -xvf prometheus-2.47.1.linux-amd64.tar.gz
+ cd prometheus-2.47.1.linux-amd64/
+ sudo mkdir -p /data /etc/prometheus
+ sudo mv prometheus promtool /usr/local/bin/
+ sudo mv consoles/ console_libraries/ /etc/prometheus/
+ sudo mv prometheus.yml /etc/prometheus/prometheus.yml
+ sudo chown -R prometheus:prometheus /etc/prometheus/ /data/

→ Ahora creamos el service para prometheus:

+ sudo vi  /etc/systemd/system/prometheus.service

Agregar el siguiente codigo al archivo que estamos creando `prometheus.service`

```
[Unit]  
Description=Prometheus  
Wants=network-online.target  
After=network-online.target  
  
StartLimitIntervalSec=500  
StartLimitBurst=5  
  
[Service]  
User=prometheus  
Group=prometheus  
Type=simple  
Restart=on-failure  
RestartSec=5s  
ExecStart=/usr/local/bin/prometheus \  
  --config.file=/etc/prometheus/prometheus.yml \  
  --storage.tsdb.path=/data \  
  --web.console.templates=/etc/prometheus/consoles \  
  --web.console.libraries=/etc/prometheus/console_libraries \  
  --web.listen-address=0.0.0.0:9090 \  
  --web.enable-lifecycle  
  
[Install]  
WantedBy=multi-user.target
```

> Nota: Para guardar el archivo le damos las siguientes teclas:  `shift + :`
> Luego colocamos lo siguiente: `wq!` 
> Luego le damos >> `enter`  





8.3.- Por ultimo vamos activar y levantar el servicio de Prometheus
```
→ sudo systemctl enable prometheus
→ sudo systemctl start prometheus
→ sudo systemctl status prometheus
```
![[Pasted image 20240512024037.png]]


→ Es importante agregar el Security group de la EC2 donde esta instalado el Prometheus el port 9090
<br>
→ Vamos a verificar que todo esta Ok con Prometheus y vamos a la web con la IP publica de la EC2 con el port 9090

![[Pasted image 20240512024230.png]]



### Paso 9 → Instalación de Node Exporter

Node Exporter es una herramienta de Prometheus que recopila métricas de hardware y sistema operativo de los hosts, proporcionando datos detallados como uso de CPU, memoria, disco, y redes. Se utiliza ampliamente para monitorizar el estado y el rendimiento de los servidores en tiempo real, facilitando la detección de problemas y la optimización del rendimiento.

→ Vamos a instalar la tools con los siguientes comandos:
```
	→ sudo useradd --system --no-create-home --shell /bin/false node_exporter
	→ wget https://github.com/prometheus/node_exporter/releases/download/v1.6.1/node_exporter-1.6.1.linux-amd64.tar.gz
```
→ Extraemos y movemos los archivos necesarios de Node  Exporter
```
	→ tar -xvf node_exporter-1.6.1.linux-amd64.tar.gz  
	→ sudo mv node_exporter-1.6.1.linux-amd64/node_exporter /usr/local/bin/  
```
→ Creamos el node exporter como servicio
```
	→ sudo vi  /etc/systemd/system/node_exporter.service
```
<br>
→ Agregamos el siguiente codigo al archivo que se esta creando

```
[Unit]  
Description=Node Exporter  
Wants=network-online.target  
After=network-online.target  
  
StartLimitIntervalSec=500  
StartLimitBurst=5  
  
[Service]  
User=node_exporter  
Group=node_exporter  
Type=simple  
Restart=on-failure  
RestartSec=5s  
ExecStart=/usr/local/bin/node_exporter --collector.logind  
  
[Install]  
WantedBy=multi-user.target
```

> Nota: Para guardar el archivo le damos las siguientes teclas:  `shift + :`
> Luego colocamos lo siguiente: `wq!` 
> Luego le damos >> `enter`  

→ Habilitamos y levantamos el servicio
```
	→ sudo systemctl enable node_exporter  
	→ sudo systemctl start node_exporter
	→ sudo systemctl status node_exporter
```
![[Pasted image 20240512024836.png]]


### Paso 10 → Configuración  Prometheus Plugin Integration:

→ Conectarse a vía ssh a la EC2 de Prometheus 

+ cd /etc/prometheus
+ Editamos el archivo con el siguiente comando:
  
  → sudo vi /etc/prometheus/prometheus.yml
  
  → Agregar el siguiente codigo (Adelanto un print de como quedaría el codigo final del proyecto)
		
```
# my global config
global:
  scrape_interval: 15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

# Alertmanager configuration
alerting:
  alertmanagers:
    - static_configs:
        - targets:
          # - alertmanager:9093

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"

# A scrape configuration containing multiple endpoints to scrape:
scrape_configs:
  # Scrape configuration for Prometheus itself
  - job_name: "prometheus"
    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.
    static_configs:
      - targets: ["localhost:9090"]

  # Scrape configuration for Node Exporter
  - job_name: 'node_exporter'
    static_configs:
      - targets: ['localhost:9100']

  # Scrape configuration for Jenkins
  - job_name: 'jenkins'
    metrics_path: '/prometheus'
    static_configs:
      - targets: ['ip-public-ec2:8080']

  - job_name: 'Netflix'
    metrics_path: '/metrics'
    static_configs:
      - targets: ['ip-public-ec2:9100']
```


> El archivo puedes validar la sintaxi con el siguiente  comando:
> → $promtool check config /etc/prometheus/prometheus.yml
> El resultado sería el siguiente: $ o/p →success



→ Luego de la configuración le damos un restart al servicio de Prometheus y luego vamos a la page de Prometheus
	→ sudo systemctl restart prometheus.service
	→ $http://ip-public-ec2:9090/

![[Pasted image 20240512025722.png]]

→  Al momento de ingresar a la web de prometheus vayan a Targets.

![](https://miro.medium.com/v2/resize:fit:700/1*22091ODfutUpZrPy5iB8Uw.png)


### Paso 11 → Configuración Grafana

→ Instalación de Dependencias
	→ sudo apt-get update  -y
	→ sudo apt-get install -y apt-transport-https software-properties-common

→ Agregamos el GPG key y el repositorio para Grafana
	→ wget -qO- [https://packages.grafana.com/gpg.key](https://packages.grafana.com/gpg.key) | sudo gpg --dearmor -o /usr/share/keyrings/grafana-archive-keyring.gpg
	→ echo "deb [signed-by=/usr/share/keyrings/grafana-archive-keyring.gpg] [https://packages.grafana.com/oss/deb](https://packages.grafana.com/oss/deb) stable main" | sudo tee /etc/apt/sources.list.d/grafana.list

→ Updateamos todo e instalamos
	→ sudo apt-get update  
	→ sudo apt-get -y install grafana
	→ sudo systemctl enable grafana-server
	→ sudo systemctl start grafana-server
	→ sudo systemctl status grafana-server

![[Pasted image 20240512030156.png]]


→ Importante agregamos el port 3000 a nuestro SG de la EC2

![](https://miro.medium.com/v2/resize:fit:700/1*0CPTI9FYcgzBggfcZPcZdg.png)

→ Vamos a la pagina de grafana con la IP-publica-ec2 donde se encuentra instalado con el port 3000

![[Pasted image 20240512030322.png]]

→ username = admin & password = admin

![[Pasted image 20240512030351.png]]

9.1.- Agregaremos el DataSource necesario para Prometheus

Para visualizar las metrics necesitamos crear un datasource en grafana

→ Dashboard Grafana:
	→ Click sobre el icono (⚙️) para abrir  “Configuration” menu.
	→ Seleccionamos “Data Sources.”
	→ Click sobre the “Add data source” button.

![](https://miro.medium.com/v2/resize:fit:700/1*-ke0lv81U6srYeaZY0UhnA.png)
	→ Elegimos  “Prometheus” como  tipo de datasource
	→ En la sección de: “HTTP”
	→ Configuramos la “URL”  `http://localhost:9090` (se coloca localhost ya que esta instalado en el mismo servidor).
	→ Clickeamos  “Save & Test”

9.2.- Importamos el  Dashboard

Vamos a usar un template que generalmente de usa para estos laboratorios, pero es posible que se puede personalizar.

→ Click sobe el “+” (plus) y se abrira  “Create” menu.

![](https://miro.medium.com/v2/resize:fit:297/1*Z7S2RztDw-ZlxJ5K9LrixA.png)
	→ Seleccionamos “Dashboard.”
	→ Click sobre  “Import” dashboard option
	→ Vamos agregar el siguiente codigo: 1860
	→ Click en “Load” button.

![](https://miro.medium.com/v2/resize:fit:700/1*YWdTF81oQQVIDR33BwMTUw.png)
	→ Seleccionamos el DataSource que creamos (Prometheus)
	→ Click en  “Import” button.
![[Pasted image 20240512031505.png]]


9.3.- Configuración global para Prometheus

Nos vamos al panel de Jenkins  → manage jenkins → system → search for prometheus → apply → save

9.4.- Ahora vamos a importar el dashboard para Jenkins
	→ Click sobre el “+” (plus) icon → “Create” menu.
	→ Seleccionamos “Dashboard.”
	→ Click sobre el “Import” dashboard option.
	→ Vamos agregar el siguiente code:  9964
	→ Click → “Load” button.
	→ Acá seleccionamos el DataSource de Prometheus

![[Pasted image 20240512031638.png]]



### Paso 12 → Creación de EKS

A continuación vamos a crear el Cluster EKS pero previamente vamos a crear el Role que usaremos y vamos a verificar la conexión desde tu terminal hacia la cuenta AWS

#### Creación de Role
	→ Ir al servicio de IAM
	→ Access management → Roles
	→ Create Role
	→ Custom trust policy:

Agregar el primer Custom trust:

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ec2:CreateSecurityGroup",
                "ec2:DescribeSecurityGroups",
                "ec2:AuthorizeSecurityGroupIngress",
                "ec2:AuthorizeSecurityGroupEgress",
                "ec2:DeleteSecurityGroup"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "ec2:CreateTags"
            ],
            "Resource": "arn:aws:ec2:*:*:instance/*",
            "Condition": {
                "ForAnyValue:StringLike": {
                    "aws:TagKeys": "kubernetes.io/cluster/*"
                }
            }
        },
        {
            "Effect": "Allow",
            "Action": [
                "ec2:DescribeInstances",
                "ec2:DescribeAvailabilityZones",
                "ec2:DescribeNetworkInterfaces",
                "ec2:DescribeVpcs",
                "ec2:DescribeDhcpOptions",
                "kms:DescribeKey"
            ],
            "Resource": "*"
        }
    ]
}
```

→ En la siguiente pantalla agregaremos los siguientes permisos:
	→ AmazonEC2ContainerRegistryReadOnly
	→ AmazonEKS_CNI_Policy
	→ AmazonEKSClusterPolicy
	→ AmazonEKSWorkerNodePolicy
→ Colocamos nombre y creamos.

#### Conexión terminal hacia AWS
	→ Desde la terminal: 
		$ aws configure
	→ Durante la configuración, se te pedirá que ingreses:
		- AWS Access Key ID: (usuario de la aws para conectarse)
		- AWS Secret Access Key: (el secret key al momento de crear el ID en aws > IAM)
		- Default region name: (por ejemplo, `us-east-1`)
		- Default output format: (por ejemplo, `json` o le das enter)

→ Para probar que estás conectado ejecutamos el siguiente comando:
	$ aws s3 ls

#### Ahora si vamos a crear nuestro cluster de EKS

→ Buscamos el servicio en Amazon AWS como >> EKS
	→ Opción Cluster
	→ Create Cluster
	→ Cluster Configuration
		→Name : Agregar un nombre
		→Kubernetes version: 1.29
		→ Cluster service role: (Crear un role) (agregalo en la docu al final)
	![[Pasted image 20240510213831.png]]
	→ Cluster access
		→Allow cluster administrator access
		→ EKS API and ConfigMap
![[Pasted image 20240510213934.png]]

Luego vamos a darle al  Boton de Next


→ Specify networking
	→  VPC: Agregar las 2  subnets Public
	→ Security groups: El que previamente se creo
	→ Choose cluster IP address family: IPv4
	→  Cluster endpoint access: Public and private
	![[Pasted image 20240510233109.png]]
→ Configure observability: Se deja todo por default

→ Select add-ons:
	→ CoreDNS
	→ kube-proxy
	→ Amazon VPC CNI
	→ Amazon EKS Pod Identity Agent

→ El resto le dan Next and create

![[Pasted image 20240512031841.png]]

#### Creación de Node-group

→Luego de crear el Cluster, vamos al proximo paso que es crear el node group

→ Vamos a la pestaña de **Compute**
→ Node groups >> Add node group
	→ Node group configuration: Name: Agrega el nombre de tu node
	→ Node IAM role: Ya fue creado posteriormente
	→ De resto se deja por default en este ejemplo pero por buenas practicas se puede agregar:
		→ Labels: (Buscar info y agregar)
		→ Taints: (Buscar info y agregar)
		→ Tags: (Buscar info y agregar)

→ Set compute and scaling configuration:
	→ Ami type: Amazon linux 2
	→ Capacity type: On-Demand
	→ Instance types: t3.medium
	→ Disk: 20 gb

→ Node group scaling configuration: Para este ejemplo lo dejaremos todo en 1

→ Node group network configuration: Agregar las 4 subnets

→ Al final de la pagina le damos **Create**

![[Pasted image 20240510223543.png]]

### Paso 13 → Instalación  aws -cli en tu PC

Para esta instalación usaremos un script para facilitar la productividad:
	→ Crear el archivo .sh y lo ejecutas:
		→ vi install_cli.sh → Copias el codigo que está más abajo
		→ $ chmod +x install_cli.sh   ## Le damos permisos al script
		→ $ ./install_cli.sh   ## ejecutamos el script

```
#!/bin/bash

# Verificar e instalar unzip si es necesario
if ! command -v unzip &> /dev/null
then
    echo "unzip no está instalado. Instalando unzip..."
    sudo apt-get update
    sudo apt-get install -y unzip
fi

# Descargar AWS CLI
echo "Descargando AWS CLI..."
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"

# Descomprimir el archivo descargado
echo "Instalando AWS CLI..."
unzip awscliv2.zip
sudo ./aws/install

# Verificar y mostrar la versión de AWS CLI instalada
echo "Verificación de la instalación de AWS CLI:"
aws --version

# Mostrar estado de la instalación
if [ $? -eq 0 ]; then
    echo "AWS CLI instalado correctamente."
else
    echo "Hubo un error instalando AWS CLI."
fi
```



### Paso → (Opcional) Instalación de Open Lens

OpenLens es una herramienta de gestión de clústeres de Kubernetes que proporciona una interfaz visual para interactuar con múltiples clústeres. Facilita la observación, el diagnóstico y la administración de los recursos y cargas de trabajo de Kubernetes, integrándose con el sistema de autenticación de Kubernetes para proporcionar un acceso seguro y simplificado.

Dejo el link para la instalación de la tool.
https://flathub.org/es/apps/dev.k8slens.OpenLens

### Conexión Terminal al AWS -Eks

> Con el siguiente comando vamos agregar nuestra configuración del EKS y poder conectanos desde la terminal a la cuenta AWS 
> $ aws eks update-kubeconfig --name Nombre-EKS --region us-east-1



### Paso 14 → Instalación de helm en tu PC
### Install Node Exporter using Helm

To begin monitoring your Kubernetes cluster, you’ll install the Prometheus Node Exporter by Using Helm to install Node Exporter in Kubernetes makes it easy to set up and manage the tool for monitoring your servers by providing a pre-packaged, customizable, and consistent way to do it.

→ Agregamos el repositorio necesario:
	→ helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
	→ Creamos en namespace:
		→ kubectl create namespace prometheus-node-exporter

→  Instalamos el Node Exporter con helm:
	→ $ helm install prometheus-node-exporter prometheus-community/prometheus-node-exporter --namespace prometheus-node-exporter

→ kubectl get pods -n prometheus-node-exporter



**Vamos agregar el Job-Scrape Metrics en el archivo prometheus.yml**

Con el siguiente comando vamos a editar y agregar el Job:
	$ sudo vi /etc/prometheus/prometheus.yml
	$ cat /etc/prometheus/prometheus.yml   ## Verificamos si esta todo ok

→  Agregamos los siguiente al final del archivo
    job_name: 'Netflix'  
	    metrics_path: '/metrics'  
	    static_configs:  
	      - targets: ['node1Ip:9100']

### Paso 15 → Instalar argoCD

ArgoCD es una herramienta de despliegue continuo, basada en GitOps, para Kubernetes. Utiliza Git como fuente única de verdad para la definición y el estado deseado de las aplicaciones e infraestructura, permitiendo la automatización y sincronización de las configuraciones de los recursos de Kubernetes desde repositorios Git. Argo CD facilita la implementación, supervisión y gestión de aplicaciones multicapa en diferentes entornos de Kubernetes, proporcionando una interfaz gráfica y una API para el control operativo. Su diseño ayuda a reducir la complejidad de gestionar lanzamientos y promociones de aplicaciones, asegurando que el entorno de producción se mantenga siempre en sincronía con los estados definidos en Git.

Vamos a instalar nuestro argoCD en EKS

→ Agregamos el repositorio de argoCD
	→ helm repo add argo-cd  https://argoproj.github.io/argo-helm

→ Updateamos el repo de helm:
	→ helm repo update

→ Creamos el namespace para argoCD
	→ kubectl create namespace argocd

→ Instalamos argoCD desde Helm:
	→ helm install argocd argo-cd/argo-cd -n argocd

![[Pasted image 20240510225258.png]]

 → Verificamos que todo este en Running con el siguiente comando:
	 → $kubectl get all -n argocd

![[Pasted image 20240510225429.png]]


### Paso 16 → Exponer argocd-server

**By default argocd-server is not publicaly exposed.** For the purpose of this workshop, we will use a Load Balancer to make it usable:

→ $ kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'  ##Crea el loadBalancer

→ $ export ARGOCD_SERVER=$(kubectl get svc argocd-server -n argocd -o json | jq --raw-output '.status.loadBalancer.ingress[0].hostname')

→ $ echo $ARGOCD_SERVER

![](https://miro.medium.com/v2/resize:fit:700/1*GlUsLVl_cPT64vUGw6GitQ.png)

El resultado del echo vamos a copiar esa URL y abrimos una browser.

![](https://miro.medium.com/v2/resize:fit:700/1*_aNE07vM-TOsi2t6wFJGfw.png)

El username sería por default que es:  admin

Con respecto a la password seguimos las siguientes instrucciones:

1. export ARGO_PWD=`kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath=”{.data.password}” | base64 -d`
2. echo $ARGO_PWD
3. output == password for argocd

![](https://miro.medium.com/v2/resize:fit:700/1*wq01A8suQre7JEilGlWn1g.png)



### Paso 17 → Deployar la aplicación con ArgoCD

→ Vamos a Settings
→ Luego repositories → connect repo

![](https://miro.medium.com/v2/resize:fit:700/1*YaRKhjJ4AUrBREpD6NDwFw.png)

![](https://miro.medium.com/v2/resize:fit:700/1*KemrtXfcNH_0-ghsxCRLEQ.png)

→ Selecionamos las siguientes opciones:
	→ Choose your connection methos: VIA HTTPS
	→ Type: git
	→ Project: default
	→ Repositorio URL: https://github.com/Aakibgithuber/Deploy-Netflix-Clone-on-Kubernetes.git

→ Click sobre connect output → successful 

→ Ahora vamos a →  application → newapp

![](https://miro.medium.com/v2/resize:fit:700/1*gemR-J5JohW4sIKe5xGIRw.png)

![](https://miro.medium.com/v2/resize:fit:700/1*j9a-75d-KHVVNu8Sigb-oQ.png)

→ Selecionamos sobre la opción de Create

→ Click sobre  sync → force → sync → ok

![](https://miro.medium.com/v2/resize:fit:700/1*TA-1exAaUPy8W5XVsKvlHg.png)

![](https://miro.medium.com/v2/resize:fit:700/1*29w1qYcB2PXkfnq342dkDw.png)

→ Click sobre la app de netflix

![](https://miro.medium.com/v2/resize:fit:700/1*p1GX4dVcCkt5bqBwouYo9A.png)

![[Pasted image 20240513175238.png]]

Agregar el port 30007 al Security group de la EC2 creado por el node-group de EKS


![](https://miro.medium.com/v2/resize:fit:700/1*AlXssgb8FXmjpNHMoq-BXg.png)

Ahora vamos al  browse >> ec2-public-ip:30007/browser

![[Pasted image 20240513175251.png]]


🥇  Terminamos 🥇
