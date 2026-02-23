# n8n-workshop

Ejemplo n8n en EC2 de AWS: instalación, configuración y despliegue escalable.

## Deploy de n8n en AWS (EC2 + PostgreSQL + Redis)
### Introducción
Este documento describe cómo desplegar n8n en AWS EC2 utilizando una arquitectura desacoplada para mejorar:

Escalabilidad
Resiliencia
Mantenimiento

Se implementa una infraestructura basada en Docker con dos instancias:

Servidor de Aplicación (n8n)
Servidor de Datos (PostgreSQL + Redis)

## Objetivo

Implementar una infraestructura que permita:

Ejecutar n8n en producción o laboratorio
Persistir datos en PostgreSQL
Gestionar colas con Redis
Evitar pérdida de datos
Escalar con workers en el futuro

## Arquitectura
              ┌───────────────────────────┐
              │      Internet / Users     │
              └─────────────┬─────────────┘
                            │
                    Elastic IP (EC2 #1)
                            │
                ┌───────────▼───────────┐
                │       EC2 - n8n       │
                │    Docker Container   │
                └───────┬───────┬───────┘
                        │       │
                        │       │
        ┌───────────────▼──┐ ┌──▼───────────────┐
        │   PostgreSQL     │ │      Redis       │
        │   EC2 #2         │ │    EC2 #2         │
        └──────────────────┘ └──────────────────┘
## Requisitos

Cuenta en AWS

Llave SSH (.pem)

Conocimientos básicos de Linux

Acceso a consola AWS

## Crear instancias EC2
EC2 #1 — n8n
Ubuntu 24.04 LTS
t3.large
30 GB disco
Security Group
Puerto	Acceso
22	Tu IP
80	Público
443	Público
5678	Tu IP

## EC2 #2 — Base de datos

Ubuntu 24.04 LTS
t3.medium
20–30 GB disco
Security Group
Puerto	Acceso
22	Tu IP
5432	IP privada n8n
6379	IP privada n8n

***Nunca abrir 5432 o 6379 a internet***

#3 Elastic IP

Asignar Elastic IP a EC2 #1
```bash
curl ifconfig.me
```

## Conexión SSH
```bash
ssh -i tu-key.pem ubuntu@TU_ELASTIC_IP
```

## Preparar servidor (AMBAS EC2)
```bash

sudo apt update && sudo apt upgrade -y
```

## Configurar SWAP
```bash
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

Verificar:
```bash
free -h
```

## Instalar Docker
```bash
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
```
```bash
sudo chmod a+r /etc/apt/keyrings/docker.gpg
```
```bash
echo \
"deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
https://download.docker.com/linux/ubuntu noble stable" \
| sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```
```bash
sudo apt update
```
```bash
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

### Activar Docker
```bash
sudo systemctl enable docker
sudo systemctl start docker
```

Permisos
```bash
sudo usermod -aG docker ubuntu
```

Cerrar sesión y volver a entrar.

Verificar
```bash
docker --version
```
```bash
docker compose version
```
```bash
docker run hello-world
```

## Servidor de Datos (EC2 #2)
```bash
mkdir ~/data && cd ~/data
```
```bash

nano docker-compose.yml
```bash

docker-compose.yml

```yaml

services:
  postgres:
    image: postgres:15
    restart: always
    environment:
      POSTGRES_USER: n8n
      POSTGRES_PASSWORD: password_seguro
      POSTGRES_DB: n8n
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"

  redis:
    image: redis:7
    restart: always
    command: redis-server --appendonly yes
    volumes:
      - redis_data:/data
    ports:
      - "6379:6379"

volumes:
  postgres_data:
  redis_data:
```
Levantar:
```bash

docker compose up -d
```
Obtener IP privada:
```bash

hostname -I
```

## Servidor n8n (EC2 #1)

mkdir ~/n8n && cd ~/n8n
```bash
mkdir n8n_data
```

## CORRECCIÓN DE PERMISOS (OBLIGATORIO)
```bash
sudo chown -R 1000:1000 n8n_data
sudo chmod -R 755 n8n_data
```bash

nano docker-compose.yml
```bash

docker-compose.yml

```yaml

services:
  n8n:
    image: docker.n8n.io/n8nio/n8n
    container_name: n8n_app
    restart: always
    user: "1000:1000"

    ports:
      - "5678:5678"

    environment:
      # Seguridad
      - N8N_SECURE_COOKIE=false

      # Autenticación
      - N8N_BASIC_AUTH_ACTIVE=true
      - N8N_BASIC_AUTH_USER=admin
      - N8N_BASIC_AUTH_PASSWORD=PasswordFuerte123!

      # URL
      - N8N_HOST=TU_ELASTIC_IP
      - N8N_PORT=5678
      - N8N_PROTOCOL=http
      - WEBHOOK_URL=http://TU_ELASTIC_IP:5678/

      # Base de datos
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=IP_PRIVADA_DB
      - DB_POSTGRESDB_PORT=5432
      - DB_POSTGRESDB_DATABASE=n8n
      - DB_POSTGRESDB_USER=n8n
      - DB_POSTGRESDB_PASSWORD=password_seguro

      # Redis
      - EXECUTIONS_MODE=queue
      - QUEUE_BULL_REDIS_HOST=IP_PRIVADA_DB
      - QUEUE_BULL_REDIS_PORT=6379

      # Zona horaria
      - GENERIC_TIMEZONE=America/Bogota

    volumes:
      - ./n8n_data:/home/node/.n8n

## Levantar n8n
```bash
docker compose up -d
```

Ver:
```bash

docker ps
```
Logs:
```bash

docker logs -f n8n_app
```

## Acceso
http://TU_ELASTIC_IP:5678

## Seguridad
EC2 n8n
Puerto	Acceso
22	Tu IP
80	Público
443	Público
5678	Tu IP
EC2 DB
Puerto	Acceso
5432	IP privada n8n
6379	IP privada n8n

 ## Validación
telnet IP_PRIVADA_DB 5432
telnet IP_PRIVADA_DB 6379

## Problemas comunes
|Problema	|Causa|
|----|-----|
|Permiso denied	|carpeta no es 1000:1000|
|n8n no guarda datos	|volumen incorrecto|
|Error cookies	|falta N8N_SECURE_COOKIE|
|Webhooks fallan	|IP dinámica|
|Redis error	puerto bloqueado|
