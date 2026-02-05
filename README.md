## 1. Prueba en local

Antes de dockerizar la aplicación, se prueba en local, creamos un entorno virtual de Python, instalando las dependencias y ejecutando la aplicación.

```bash
$ python3 -m venv venv
$ source venv/bin/activate
(venv) $ pip install -r requirements.txt
(venv) $ python src/app.py
```

Para comprobar que funciona:

```bash
$ curl http://localhost:5000
$ curl http://localhost:5000/status
```


---

## 1.2. Creación de la imagen Docker

El archivo `Dockerfile` utilizado es el siguiente:

```dockerfile
FROM python:3-slim
WORKDIR /app
COPY ./requirements.txt ./
RUN pip install -r requirements.txt
COPY src .
CMD gunicorn --bind 0.0.0.0:5000 app:app
```

Para crear y probar la imagen:

```bash
(venv) $ docker build -t <usuario_dockerhub>/galeria:latest .
(venv) $ docker run -d -p 80:5000 --name testgaleria <usuario_dockerhub>/galeria
(venv) $ curl http://localhost/status
```

Para detener y eliminar el contenedor:

```bash
(venv) $ docker stop testgaleria && docker rm testgaleria
```

---

## 1.3. Prueba de la imagen con docker-compose.yml

Creamos un archivo `docker-compose.yml` para ejecutar el servicio:

```yaml
services:
  galeria:
    container_name: galeria
    image: <usuario_dockerhub>/galeria:latest
    ports:
      - 80:5000
    restart: always
```

Probamos a iniciar y detener la aplicación:

```bash
(venv) $ docker compose up -d
(venv) $ curl http://localhost
(venv) $ docker compose down
```

---

## 1.4. Subida de la imagen a DockerHub

Para publicar la imagen en DockerHub:

```bash
(venv) $ docker push <usuario_dockerhub>/galeria:latest
```

---

## 2. GitHub Actions - Build and Push

Se crea un workflow en GitHub Actions que construye y sube automáticamente la imagen a DockerHub cuando se realicen cambios en el directorio `src`.

### 2.1. Secretos necesarios

En los ajustes del repositorio `Settings / Security / Secrets and Variables / Actions` se crean los secretos:

- `DOCKERHUB_USERNAME`: usuario en DockerHub  
- `DOCKERHUB_TOKEN`: token que hemos generado en nuestra cuenta de DockerHub

### 2.2. Crear el Action

Creamos un archivo para el workflow (`.github/workflows/devops02.yml`) con el contenido:

```yaml
name: Docker CI/CD

on:
  push:
    branches: [ main ]
    paths:
      - src/**
  pull_request:
    branches: [ main ]
    paths:
      - src/**

jobs:
  build:
    name: build and push Docker image
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Login to DockerHub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}
      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ secrets.DOCKERHUB_USERNAME }}/galeria:latest
```
Para probar el fichero tendremos que:
1. Crear el fichero del Action en GitHub.  
1. Crear una nueva rama local y modificar algún fichero del directorio `src` (por ejemplo `index.html`).  
1. Hacer push de la rama y, si es necesario, crear un Pull Request.  
1. Comprobar en la pestaña **Actions** que los pasos se están ejecutando correctamente.  
1. Verificar en DockerHub que la imagen se ha subido.

---

## 3. Despliegue en AWS (opcional)

Se puede añadir un segundo job al Action anterior para desplegar la imagen en una instancia EC2 de AWS.  
Es necesario tener Docker instalado en la instancia y crear los siguientes secretos en el repositorio:

- `AWS_USERNAME`: usuario en la instancia (en este caso **admin**).
- `AWS_HOSTNAME`: IP pública de la instancia.
- `AWS_PRIVATEKEY`: contenido del fichero `.pem` con la clave privada SSH (completo).

### 3.1. Código añadido al Action
**IMPORTANTE!! ASEGURARSE QUE EL SERVICIO AWS QUEDE A LA MISMA ALTURA QUE EL BUILD**
```yaml
jobs:
  build:
    # ...
  aws:
    name: Deploy image to aws
    needs: build
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - name: copy docker compose via ssh key  
      uses: appleboy/scp-action@v0.1.7
      with:
        host: ${{ secrets.AWS_HOSTNAME }}
        username: ${{ secrets.AWS_USERNAME }}
        port: 22 
        key: ${{ secrets.AWS_PRIVATEKEY }}
        source: "docker-compose.yml"
        target: /home/admin
    - name: script deploy docker services 
      uses: appleboy/ssh-action@v1.0.3
      with:
        host: ${{ secrets.AWS_HOSTNAME }}
        username: ${{ secrets.AWS_USERNAME }}
        key: ${{ secrets.AWS_PRIVATEKEY }}
        port: 22 
        script: |
            sleep 60
            docker compose down --rmi all
            docker compose up -d
```
---
