# n8n-workshop
Ejemplo N8n en EC2 de  AWS, instalación, configuracion y ejemplo.

# Deploy de n8n en AWS (EC2 + PostgreSQL + Redis)
## Introducción

Este repositorio/documentación describe cómo desplegar n8n, una plataforma de automatización de workflows, en AWS EC2 usando una arquitectura desacoplada para mejorar la escalabilidad, resiliencia y mantenimiento.

Se implementa una infraestructura basada en contenedores con Docker, separando los servicios en dos instancias:

Servidor de Aplicación (n8n)

Servidor de Datos (PostgreSQL + Redis)

Este enfoque permite soportar cargas de trabajo más complejas, como ejecuciones en cola (queue mode), automatizaciones intensivas y agentes inteligentes.

## Objetivo

Implementar una infraestructura funcional y escalable que permita:

Ejecutar n8n en producción o laboratorio

Persistir datos en PostgreSQL

Gestionar colas de ejecución con Redis

Evitar pérdida de datos

Facilitar crecimiento futuro (workers, clustering)

Asegurar conectividad estable (Elastic IP)

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

1. Crear instancias EC2

## EC2 #1 — n8n

AMI: Ubuntu 24.04 LTS

Tipo: t3.large (mínimo)

Disco: 30 GB

Security Group:

SSH (22) → tu IP

HTTP (80) → 0.0.0.0/0

HTTPS (443) → 0.0.0.0/0

5678 → tu IP (pruebas)

## EC2 #2 — Base de datos

AMI: Ubuntu 24.04 LTS

Tipo: t3.medium

Disco: 20–30 GB

Security Group:

SSH (22) → tu IP

PostgreSQL (5432) → IP privada EC2 n8n

Redis (6379) → IP privada EC2 n8n

** Nunca abrir 5432 o 6379 a internet.**

## 2. Configurar Elastic IP

Ir a EC2 → Elastic IPs

Click en Allocate Elastic IP

Click en Associate Elastic IP

Seleccionar instancia n8n

Verificar:

curl ifconfig.me

# 3. Conectarse por SSH
´´´bash
ssh -i tu-key.pem ubuntu@TU_ELASTIC_IP
´´´
## 4. Preparar el servidor

Ejecutar en ambas instancias:

sudo apt update && sudo apt upgrade -y

## 5. Configurar SWAP
´´´bash
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
´´´
Verificar:

free -h

## 6. Instalar Docker

sudo apt install -y ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo tee /etc/apt/keyrings/docker.gpg > /dev/null
sudo chmod a+r /etc/apt/keyrings/docker.gpg
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu noble stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

Activar Docker:

sudo systemctl enable docker
sudo systemctl start docker
sudo usermod -aG docker ubuntu

Reiniciar sesión SSH.


## 7. Configurar PostgreSQL y Redis (EC2 #2)

mkdir ~/data && cd ~/data
nano docker-compose.yml
version: "3.8"

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

Levantar servicios:

docker compose up -d
🔍 Obtener IP privada
hostname -I

Ejemplo: 172.31.x.x

⚙️ 8. Configurar n8n (EC2 #1)
mkdir ~/n8n && cd ~/n8n
nano docker-compose.yml
version: "3.8"

services:
  n8n:
    image: docker.n8n.io/n8nio/n8n
    restart: always
    ports:
      - "5678:5678"

    environment:
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
      - QUEUE_BULL_REDIS_HOST=IP_PRIVADA_DB
      - QUEUE_BULL_REDIS_PORT=6379
      - EXECUTIONS_MODE=queue

      # Configuración general
      - GENERIC_TIMEZONE=America/Bogota
      - N8N_RUNNERS_ENABLED=true

    volumes:
      - ~/.n8n:/home/node/.n8n
## 9. Levantar n8n
docker compose up -d

Verificar:

docker ps

Logs:

docker logs -f n8n

## 10. Acceso
http://TU_ELASTIC_IP:5678


## 11. Seguridad
EC2 n8n

22 → tu IP

80 → público

443 → público

5678 → solo tu IP

EC2 DB

5432 → solo IP privada n8n

6379 → solo IP privada n8n

## 12. Validación
Probar PostgreSQL
telnet IP_PRIVADA_DB 5432
Probar Redis
telnet IP_PRIVADA_DB 6379

** Problemas comunes **

Problema	Causa
Webhooks no funcionan	IP dinámica
n8n pierde datos	SQLite en uso
Errores en ejecución	Redis no configurado
Timeout	puertos bloqueados
🚀 Mejoras futuras

Nginx + HTTPS (Let's Encrypt)

Dominio personalizado

n8n workers (escalado horizontal)

Backups automáticos

Monitoreo (Prometheus/Grafana)

📌 Conclusión

Con esta arquitectura:

Se separa lógica de aplicación y datos

Se mejora escalabilidad

Se habilita uso de colas y agentes

Se reduce riesgo de pérdida de datos

🧠 Autor

Guía técnica para despliegue de n8n en AWS con enfoque en producción y escalabilidad.
