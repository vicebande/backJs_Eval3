# Backend 1 - User Service (Node.js)

Servicio backend desarrollado en Node.js (Express) para la gestión de usuarios con persistencia en una base de datos MySQL. Este servicio forma parte de la **Evaluación Parcial N°3 de Introducción a Herramientas DevOps**.

---

## 🚀 Ejecución Local

### Opción 1: Tradicional (Manual)

1. **Requisitos**: Node.js 18+ y MySQL 8.0+.
2. **Configuración**:
   ```bash
   cp .env.example .env
   ```
   Configure sus credenciales en `.env` (puerto por defecto `8081`).
3. **Inicialización**:
   ```bash
   npm install
   npm start
   ```

### Opción 2: Con Docker (Individual)

1. **Construir la imagen**:
   ```bash
   docker build -t users-backend:latest .
   ```
2. **Ejecutar el contenedor** (requiere una base de datos externa accesible):
   ```bash
   docker run -d -p 8081:8081 \
     -e DB_HOST=host.docker.internal \
     -e DB_PORT=3306 \
     -e DB_USER=root \
     -e DB_PASSWORD=tu_password \
     -e DB_NAME=users_db \
     users-backend:latest
   ```

---

## 🏛️ Arquitectura de Despliegue en AWS (ECS Fargate)

Este servicio está diseñado para ejecutarse en la nube utilizando la siguiente infraestructura de Amazon Web Services (AWS):

* **AWS ECS Fargate**: Cómputo serverless para contenedores. Evita administrar instancias EC2 y reduce costos en laboratorios.
* **AWS ECR**: Repositorio privado para almacenar las imágenes de contenedor (`eva3-users-repo`).
* **Application Load Balancer (ALB)**: Distribuye el tráfico hacia las tareas de ECS. Para este proyecto, se implementa en conjunto con el Frontend y el otro Backend:
  * Tráfico hacia `/api/users/*` se enruta al puerto `8081` de `user-service`.
* **Amazon RDS (MySQL)**: Base de datos relacional administrada, compartida o exclusiva para persistencia.
* **AWS Systems Manager (SSM) Parameter Store**: Almacenamiento seguro de credenciales (`DB_PASSWORD`).

---

## 🛡️ Gestión de Secrets y Seguridad (IE5)

Para dar cumplimiento estricto a las normas de seguridad del indicador **IE5** (Gestión de Secrets y credenciales sin exposición):
1. **SSM Parameter Store**: El password de base de datos se guarda de forma segura en el parámetro `/eva3/db/password` tipo `SecureString`.
2. **Inyección en ECS**: En la definición de tarea (`task-definition.json`), no se expone el password en texto plano. Se referencia mediante el bloque `secrets`:
   ```json
   "secrets": [
     {
       "name": "DB_PASSWORD",
       "valueFrom": "arn:aws:ssm:us-east-1:<ACCOUNT_ID>:parameter/eva3/db/password"
     }
   ]
   ```
3. **AWS Academy / LabRole**: Durante el despliegue en AWS Academy Learner Lab, se debe utilizar el rol pre-creado `LabRole` como **Task Execution Role** y **Task Role**. Este rol tiene permisos asignados para leer de Systems Manager.

---

## 📈 Configuración de Autoscaling (IE3)

Para garantizar la alta disponibilidad y tolerancia a fallos del servicio de usuarios:
* **Métrica objetivo**: CPU Utilization promedio de las tareas.
* **Umbral de escala (Target Tracking)**: **50%**.
  * **Justificación del umbral**: Un umbral del 50% es ideal para entornos de producción de carga variable. Permite absorber ráfagas repentinas de tráfico mientras Fargate aprovisiona nuevas tareas (lo cual toma entre 1 y 2 minutos) sin saturar las tareas existentes ni degradar la experiencia de usuario.
* **Límites de escala**: Mínimo 1 tarea, Máximo 3 tareas (diseñado para mantener el presupuesto controlado en AWS Academy).

---

## 🔄 Pipeline CI/CD con GitHub Actions (IE4)

El archivo `.github/workflows/deploy.yml` automatiza todo el ciclo de entrega continua (build ➡️ push ➡️ deploy):

1. **Gatillo**: Empujar cambios a las ramas `main` o `master`.
2. **Inicio y autenticación**: Configura credenciales usando `aws-actions/configure-aws-credentials` y Secrets de GitHub.
3. **Login en ECR**: Se loguea al registro privado con `aws-actions/amazon-ecr-login`.
4. **Construcción y etiquetado**: Genera la imagen Docker etiquetándola con el hash del commit (`github.sha`) y la etiqueta `latest`.
5. **Subida**: Empuja ambas imágenes a Amazon ECR.
6. **Actualización de ECS**: Modifica la definición de tarea (`task-definition.json`) con la nueva imagen y despliega la actualización en ECS actualizando el servicio (`user-service`).

### Configuración de GitHub Secrets Necesarios:
* `AWS_ACCESS_KEY_ID`: Credencial temporal de AWS.
* `AWS_SECRET_ACCESS_KEY`: Credencial temporal de AWS.
* `AWS_SESSION_TOKEN`: Token de sesión de AWS Academy (obligatorio en Learner Labs).
* `AWS_REGION`: Región de despliegue (ej. `us-east-1`).

---

## 📊 Logs y Monitoreo (IE6)

Los logs de la aplicación se transmiten en tiempo real a **Amazon CloudWatch Logs** mediante el driver `awslogs` configurado en `task-definition.json`.
* **Grupo de Logs**: `/ecs/user-service`
* **Prefijo de Logs**: `ecs`
* Esto permite realizar el análisis de errores e inspeccionar llamadas HTTP del endpoint `/api/users/register` o `/api/users`.
