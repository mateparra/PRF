# Proyecto Integrador - Rama `amazonebs`


Este proyecto integrador demuestra la implementación de un entorno de aplicaciones web utilizando **Docker**, **NGINX** como **Proxy Reverso**, **Amazon EBS** para **Almacenamiento por Bloque**, y una estrategia de **Gestión de Ramas** en **Git**. A continuación, se detallan los componentes clave y cómo interactúan entre sí para proporcionar una solución escalable y persistente.


## Tabla de Contenidos


1. [Descripción General](#descripción-general)
2. [Estructura del Proyecto](#estructura-del-proyecto)
3. [Proxy Reverso con NGINX](#proxy-reverso-con-nginx)
4. [Almacenamiento por Bloque con Amazon EBS](#almacenamiento-por-bloque-con-amazon-ebs)
5. [Creación y Despliegue de Imágenes Docker](#creación-y-despliegue-de-imágenes-docker)
6. [Gestión de Ramas en Git](#gestión-de-ramas-en-git)
7. [Despliegue en AWS](#despliegue-en-aws)
8. [Consideraciones Adicionales](#consideraciones-adicionales)
9. [Autor](#autor)


---


## Descripción General


El objetivo de este proyecto es demostrar cómo desplegar aplicaciones web utilizando contenedores Docker, gestionar el tráfico mediante un proxy reverso con NGINX, asegurar la persistencia de datos utilizando Amazon EBS, y mantener una estructura de control de versiones eficiente con Git.


### Componentes Principales


- **app1 y app2**: Dos aplicaciones web independientes que sirven contenido estático.
- **NGINX**: Configurado como proxy reverso para gestionar y redirigir el tráfico hacia las aplicaciones.
- **Docker Compose**: Orquesta la construcción y despliegue de los contenedores.
- **Amazon EBS**: Proporciona almacenamiento persistente para las aplicaciones.
- **Git**: Gestiona las versiones del proyecto con ramas específicas.


---


## Estructura del Proyecto


La estructura del proyecto está organizada de la siguiente manera:




- **app1 y app2**: Contienen sus respectivos `Dockerfile` y archivos HTML.
- **nginx**: Contiene el `Dockerfile`, la configuración de NGINX (`nginx.conf`) y la página principal (`site.html`).
- **docker-compose.yml**: Define y configura los servicios de Docker.
- **README.md**: Documentación del proyecto.


---


## Proxy Reverso con NGINX


### ¿Qué es un Proxy Reverso?


Un **Proxy Reverso** es un servidor que se sitúa entre los clientes y uno o varios servidores backend. Su función principal es recibir las solicitudes de los clientes y redirigirlas al servidor backend adecuado. En este proyecto, NGINX actúa como proxy reverso para gestionar el tráfico hacia `app1` y `app2`.


### Configuración de NGINX


El archivo `nginx/nginx.conf` está configurado para definir dos **upstreams** (`app1` y `app2`) y gestionar las rutas de acceso.


```nginx
events {}


http {
    upstream app1 {
        server app1:80;
    }


    upstream app2 {
        server app2:80;
    }


    server {
        listen 80;


        # Redireccionar /app1 a /app1/
        location = /app1 {
            return 301 /app1/;
        }


        # Ruta para app1
        location /app1/ {
            proxy_pass http://app1/;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }


        # Redireccionar /app2 a /app2/
        location = /app2 {
            return 301 /app2/;
        }


        # Ruta para app2
        location /app2/ {
            proxy_pass http://app2/;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }


        # Página principal
        location / {
            root /usr/share/nginx/html;
            index site.html;
        }
    }
}


— 


¿Cómo Funciona?
1. Upstreams: Define dos grupos de servidores (app1 y app2) que NGINX utilizará para redirigir el tráfico.
2. Rutas Específicas:
   * http://<IP_PÚBLICA>/app1/ redirige al contenedor app1.
   * http://<IP_PÚBLICA>/app2/ redirige al contenedor app2.
3. Página Principal: Cualquier otra solicitud se dirige a la página principal definida en site.html.
Ventajas del Proxy Reverso
* Centralización: Facilita la gestión del tráfico en un solo punto.
* Escalabilidad: Permite añadir más servidores backend sin cambiar la configuración del cliente.
* Seguridad: Puede ocultar detalles de los servidores backend y manejar certificados SSL.
—


Almacenamiento por Bloque con Amazon EBS
¿Qué es Amazon EBS?
Amazon Elastic Block Store (EBS) proporciona almacenamiento de bloques persistente para usar con instancias de Amazon EC2. Es ideal para aplicaciones que requieren un sistema de archivos, almacenamiento de base de datos, o acceso directo a los bloques de datos.
Beneficios de Usar EBS
* Persistencia de Datos: Los datos almacenados en EBS persisten independientemente del estado de la instancia EC2.
* Separación de Datos y Lógica de Aplicación: Facilita la gestión y escalabilidad.
* Flexibilidad: Permite aumentar o disminuir el tamaño del volumen según las necesidades.
Configuración en el Proyecto
1. Montaje del Volumen EBS: El volumen EBS se monta en /ebs-data en la instancia EC2.
2. Bind Mounts en Docker Compose: Las aplicaciones app1 y app2 utilizan bind mounts para servir su contenido desde el volumen EBS.


```yaml
services:
  app1:
    build:
      context: ./app1
    image: app1_image:latest
    volumes:
      - /ebs-data/app1:/usr/share/nginx/html
    networks:
      - my-network


  app2:
    build:
      context: ./app2
    image: app2_image:latest
    volumes:
      - /ebs-data/app2:/usr/share/nginx/html
    networks:
      - my-network
```


¿Cómo Funciona?
* Persistencia: Los archivos servidos por app1 y app2 se almacenan físicamente en el volumen EBS (/ebs-data/app1 y /ebs-data/app2), asegurando que los datos persisten incluso si los contenedores se detienen o eliminan.
* Separación de Datos: La lógica de la aplicación (contenedores) está separada del almacenamiento de datos (EBS), permitiendo una gestión más eficiente y segura.
—


Creación y Despliegue de Imágenes Docker
¿Por Qué Docker?
Docker permite empaquetar aplicaciones y sus dependencias en contenedores, garantizando que funcionen de manera consistente en cualquier entorno.
Creación de Dockerfiles
Cada servicio (app1, app2, nginx) tiene su propio Dockerfile que define cómo se construye la imagen del contenedor.
Dockerfile de app1 y app2


```dockerfile
FROM nginx:latest


# Copiar el archivo HTML de la aplicación
COPY index.html /usr/share/nginx/html/


# Exponer el puerto 80
EXPOSE 80
```


NGINX:


```dockerfile
FROM nginx:latest


# Copiar el archivo de configuración de NGINX
COPY nginx.conf /etc/nginx/nginx.conf


# Copiar la página principal personalizada
COPY site.html /usr/share/nginx/html/site.html


# Exponer el puerto 80
EXPOSE 80
```


—


Uso de Docker Compose


El archivo docker-compose.yml orquesta la construcción y despliegue de los contenedores.
```yaml
version: '3.8'


services:
  app1:
    build:
      context: ./app1
    image: app1_image:latest
    volumes:
      - /ebs-data/app1:/usr/share/nginx/html
    networks:
      - my-network


  app2:
    build:
      context: ./app2
    image: app2_image:latest
    volumes:
      - /ebs-data/app2:/usr/share/nginx/html
    networks:
      - my-network


  nginx:
    build:
      context: ./nginx
    image: nginx_image:latest
    ports:
      - "80:80"
    depends_on:
      - app1
      - app2
    volumes:
      - ./nginx/site.html:/usr/share/nginx/html/site.html
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf
    networks:
      - my-network


networks:
  my-network:
    driver: bridge
```
¿Cómo Funciona?
1. Construcción de Imágenes:
   * docker-compose build construye las imágenes definidas en los Dockerfile de cada servicio.
2. Despliegue de Contenedores:
   * docker-compose up -d --build levanta los contenedores en segundo plano (-d) y fuerza la reconstrucción de las imágenes si es necesario (--build).
3. Redes de Docker:
   * my-network permite que los contenedores se comuniquen entre sí de manera segura y aislada.
—


## Gestión de Ramas en Git


Estrategia de Ramas
El proyecto utiliza dos ramas principales para mantener una separación clara entre la documentación básica y el código fuente completo.
* master:
   * Contenido: Solo contiene el README.md básico.
   * Propósito: Servir como rama de referencia inicial y documentación.
* amazonebs:
   * Contenido: Incluye todo el proyecto con configuraciones de Docker, NGINX, y Amazon EBS.
   * Propósito: Desarrollar y mantener el entorno de producción completo.
—


# Correr el proyecto


Construir y Levantar los Contenedores:


```bash
docker-compose up -d --build
```


Detener y Eliminar los Contenedores:


```bash
docker-compose down
```


Verificar Contenedores Activos:


```bash
docker ps
```


— 


## Despliegue en AWS


Prerequisitos
* Cuenta en AWS con permisos para crear instancias EC2 y volúmenes EBS.
* Clave SSH para acceder a la instancia EC2.
* Docker y Docker Compose instalados en la instancia EC2.
Pasos para el Despliegue
1. Crear y Configurar una Instancia EC2:
   * Selecciona una AMI basada en Linux (Amazon Linux 2 o Ubuntu).
   * Configura los grupos de seguridad para permitir tráfico en los puertos 22 (SSH) y 80 (HTTP).
   * Adjunta un volumen EBS adicional y móntalo en /ebs-data.
```bash
# Actualizar el sistema
sudo yum update -y  # Amazon Linux
# o
sudo apt update && sudo apt upgrade -y  # Ubuntu


# Instalar Docker
sudo amazon-linux-extras install docker -y  # Amazon Linux
# o
sudo apt install -y docker.io  # Ubuntu


# Iniciar y habilitar Docker
sudo systemctl start docker
sudo systemctl enable docker


# Agregar usuario al grupo Docker (opcional)
sudo usermod -aG docker ec2-user  # Amazon Linux
# o
sudo usermod -aG docker ubuntu  # Ubuntu


# Instalar Docker Compose
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
docker-compose --version
```


Clonar el Repositorio y Desplegar los Contenedores:


```bash
# Instalar Git
sudo yum install -y git  # Amazon Linux
# o
sudo apt install -y git  # Ubuntu


# Clonar la rama amazonebs
git clone -b amazonebs https://github.com/tu_usuario/ProyectoIntegrador.git
cd ProyectoIntegrador


# Asegurar que el volumen EBS está montado en /ebs-data
# Ya debe estar montado según la configuración previa


# Desplegar los contenedores
sudo docker-compose down  # Asegurar que no hay servicios corriendo
sudo docker-compose up -d --build
```


—


## Autor


Mateo Parra
Contacto: mateparra@gmail.com
