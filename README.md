# back-Ventas — Microservicio de Ventas

API REST desarrollada en Spring Boot 3 que gestiona las ventas del sistema Innovatech Chile. Desplegada en **AWS ECS Fargate** dentro de una subred privada, con acceso expuesto únicamente a través del **Application Load Balancer (ALB)**.

---

## Tabla de contenidos

1. [Arquitectura en producción](#1-arquitectura-en-producción)
2. [Endpoints de la API](#2-endpoints-de-la-api)
3. [Variables de entorno](#3-variables-de-entorno)
4. [Contenedor Docker](#4-contenedor-docker)
5. [Pipeline CI/CD](#5-pipeline-cicd)
6. [Gestión de secretos](#6-gestión-de-secretos)
7. [Escalado automático](#7-escalado-automático)
8. [Monitoreo y logs](#8-monitoreo-y-logs)
9. [Validación funcional](#9-validación-funcional)
10. [Ejecución local](#10-ejecución-local)

---

## 1. Arquitectura en producción

```
Internet
    │  HTTPS 443
    ▼
┌─────────────────────────────────────────────────────┐
│  Application Load Balancer (ALB)                    │
│  innovatech-alb-516038279.us-east-1.elb.amazonaws.com │
│  Regla: /api/v1/ventas* → back-ventas-svc :8080     │
└──────────────────────────┬──────────────────────────┘
                           │ HTTP interno
                           ▼
            ┌──────────────────────────────┐
            │  ECS Fargate — back-ventas-svc │
            │  Cluster: innovatech-ecs-cluster │
            │  Tareas: 2–5 (autoscaling)   │
            │  Puerto: 8080                │
            │  Subred: privada us-east-1   │
            └──────────────┬───────────────┘
                           │ JDBC MySQL
                           ▼
            ┌──────────────────────────────┐
            │  Amazon RDS MySQL 8.0        │
            │  innovatech-ecs-db           │
            │  Subred privada              │
            └──────────────────────────────┘
```

**Componentes de infraestructura:**

| Recurso | Nombre / Valor |
|---|---|
| Cluster ECS | `innovatech-ecs-cluster` |
| Servicio ECS | `back-ventas-svc` |
| ECR Repository | `back-ventas` |
| Puerto de contenedor | `8080` |
| Región | `us-east-1` |
| ALB | `innovatech-alb-516038279.us-east-1.elb.amazonaws.com` |
| RDS endpoint | `innovatech-ecs-db.cyihboqkds5h.us-east-1.rds.amazonaws.com` |
| Tipo de lanzamiento | Fargate (serverless) |

**¿Por qué ECS Fargate y no EC2?**

Fargate elimina la gestión de servidores: AWS provisiona, parchea y escala la infraestructura de cómputo. El equipo solo define la imagen Docker y los recursos (CPU/RAM); el resto es gestionado por la plataforma.

---

## 2. Endpoints de la API

Base path: `/api/v1/ventas`

| Método | Ruta | Descripción |
|---|---|---|
| `GET` | `/api/v1/ventas` | Listar todas las ventas |
| `GET` | `/api/v1/ventas/{idVenta}` | Obtener venta por ID |
| `POST` | `/api/v1/ventas` | Crear nueva venta |
| `PUT` | `/api/v1/ventas/{idVenta}` | Actualizar venta existente |

**Documentación interactiva (Swagger UI):**

```
https://innovatech-alb-516038279.us-east-1.elb.amazonaws.com/swagger-ui.html
```

---

## 3. Variables de entorno

La aplicación no contiene credenciales en el código ni en archivos versionados. Todas las variables sensibles se inyectan en tiempo de ejecución a través de la **ECS Task Definition** (ver sección 6).

| Variable | Descripción | Ejemplo |
|---|---|---|
| `DB_ENDPOINT` | Host del servidor RDS | `innovatech-ecs-db.cyihboqkds5h.us-east-1.rds.amazonaws.com` |
| `DB_PORT` | Puerto MySQL | `3306` |
| `DB_NAME` | Nombre de la base de datos | `ventasdb` |
| `DB_USERNAME` | Usuario de la base de datos | *(secreto)* |
| `DB_PASSWORD` | Contraseña de la base de datos | *(secreto)* |

Configuración en `application.properties`:

```properties
spring.datasource.url=jdbc:mysql://${DB_ENDPOINT}:${DB_PORT}/${DB_NAME}?useSSL=false&serverTimezone=UTC&createDatabaseIfNotExist=true
spring.datasource.username=${DB_USERNAME}
spring.datasource.password=${DB_PASSWORD}
spring.jpa.hibernate.ddl-auto=update
management.endpoints.web.exposure.include=health,info
management.endpoint.health.show-details=never
```

> **Nota de seguridad:** `show-details` se configuró en `never` (no `always`) para evitar exponer públicamente sin autenticación información sensible del sistema —motor y estado de la base de datos, espacio en disco, rutas internas del filesystem— a través de `/actuator/health`. El ALB solo necesita el código HTTP 200/503 para determinar salud, no el detalle interno. Ver Incidente 3 más abajo para el detalle completo de este hallazgo.

---

## 4. Contenedor Docker

El `Dockerfile` implementa un **build multi-stage** para separar el entorno de compilación del de producción:

```dockerfile
# Stage 1: Build — Maven compila el JAR
FROM maven:3.9-eclipse-temurin-17-alpine AS builder
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline -B        # cachea dependencias
COPY src ./src
RUN mvn package -DskipTests -B          # genera target/*.jar

# Stage 2: Runtime — solo la JRE mínima
FROM eclipse-temurin:17-jre-alpine AS runtime
WORKDIR /app
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
COPY --from=builder /app/target/*.jar app.jar
RUN chown appuser:appgroup app.jar
USER appuser                            # principio de mínimo privilegio
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

**Decisiones de diseño:**

- **Multi-stage build:** La imagen `maven:3.9` (~500MB) solo se usa para compilar. La imagen final solo contiene la JRE (`eclipse-temurin:17-jre-alpine`, ~85MB), sin Maven, sin código fuente, sin herramientas de desarrollo.
- **Usuario no-root (`appuser`):** Si el contenedor fuera comprometido, el atacante no tendría privilegios de administrador sobre el sistema host. Aplica el principio de mínimo privilegio.
- **`mvn dependency:go-offline` antes de copiar `src/`:** Permite a Docker cachear la capa de dependencias. Si solo cambia el código fuente (no el `pom.xml`), Docker reutiliza la capa cacheada y el build es ~5x más rápido.
- **`-DskipTests`:** Las pruebas de integración requieren base de datos; el pipeline de CI/CD valida el deploy observando el estado del servicio ECS, no ejecutando tests dentro del contenedor.

**Tamaño comparado:**

| Etapa | Imagen base | Tamaño aprox. |
|---|---|---|
| Builder | `maven:3.9-eclipse-temurin-17-alpine` | ~500 MB |
| Runtime final | `eclipse-temurin:17-jre-alpine` | ~85 MB |

---

## 5. Pipeline CI/CD

El archivo `.github/workflows/deploy.yml` automatiza el ciclo completo de integración y despliegue. Se activa en cada `git push` a la rama `deploy`.

```
git push origin deploy
        │
        ▼
┌────────────────────────────────────────┐
│  GitHub Actions                        │
│                                        │
│  1. Checkout del repositorio           │
│  2. Configurar AWS credentials         │
│     (ACCESS_KEY_ID + SESSION_TOKEN     │
│      desde GitHub Secrets)             │
│  3. Login a Amazon ECR                 │
│  4. docker build -t registry/back-ventas:sha │
│                  -t registry/back-ventas:latest │
│  5. docker push ambos tags             │
│  6. aws ecs update-service             │
│     --force-new-deployment             │
│  7. Verificar rolloutState             │
│     (COMPLETED o IN_PROGRESS = OK)     │
└────────────────────────────────────────┘
        │
        ▼
ECS Fargate descarga la nueva imagen
y reemplaza tareas gradualmente (rolling update)
```

**Variables del pipeline:**

```yaml
env:
  AWS_REGION: us-east-1
  ECR_REPOSITORY: back-ventas
  ECS_CLUSTER: innovatech-ecs-cluster
  ECS_SERVICE: back-ventas-svc
```

**¿Por qué dos tags (`sha` y `latest`)?**

- `sha` (ej. `a3f7c9d`): permite rollback a una versión exacta anterior sin ambigüedad.
- `latest`: permite referenciar la imagen más reciente sin conocer el SHA. La ECS Task Definition usa `:latest` para el despliegue automático.

**¿Por qué `--force-new-deployment`?**

Aunque el tag `:latest` ya existe en ECR, ECS no detecta automáticamente que la imagen cambió. `--force-new-deployment` fuerza a ECS a re-lanzar las tareas descargando la imagen más reciente.

**Rolling Update:** ECS reemplaza tareas gradualmente (nunca baja las tareas activas antes de levantar las nuevas), garantizando **cero downtime** durante el despliegue.

---

## 6. Gestión de secretos

El proyecto maneja dos capas de secretos:

### GitHub Actions Secrets

Las credenciales de AWS para que el pipeline pueda publicar en ECR y desplegar en ECS se almacenan como **GitHub Secrets** (cifrados, nunca expuestos en logs):

| Secret | Descripción |
|---|---|
| `AWS_ACCESS_KEY_ID` | Access Key de AWS Academy |
| `AWS_SECRET_ACCESS_KEY` | Secret Access Key de AWS Academy |
| `AWS_SESSION_TOKEN` | Session Token de AWS Academy (credenciales temporales) |

**¿Por qué Session Token?** AWS Academy usa credenciales temporales (STS). Cada sesión genera nuevas credenciales que expiran en ~4 horas. GitHub Secrets permite actualizarlas sin modificar el código.

### ECS Task Definition — Variables de entorno

Las credenciales de base de datos (`DB_USERNAME`, `DB_PASSWORD`) y el endpoint RDS se inyectan directamente en la Task Definition de ECS. **Nunca aparecen en el repositorio Git, Dockerfile, ni logs del pipeline.**

```
GitHub Secrets → solo disponibles dentro del runner de GitHub Actions
ECS Task Def env vars → solo disponibles dentro del contenedor en ejecución
```

---

## 7. Escalado automático

Application Auto Scaling está configurado sobre el servicio ECS para ajustar la capacidad según la demanda:

| Parámetro | Valor |
|---|---|
| Mínimo de tareas | 2 |
| Máximo de tareas | 5 |
| Servicio | `back-ventas-svc` |

**¿Por qué mínimo 2?** Alta disponibilidad: si una tarea falla o una zona de disponibilidad tiene problemas, siempre hay otra tarea activa atendiendo tráfico.

**¿Por qué máximo 5?** Límite de costo para el entorno académico (AWS Academy tiene cuotas). En producción real este valor se definiría según el throughput esperado.

Las políticas de escalado se basan en métricas como utilización de CPU o número de peticiones por tarea, configuradas en AWS Application Auto Scaling.

---

## 8. Monitoreo y logs

### CloudWatch Container Insights

Container Insights está habilitado en el cluster `innovatech-ecs-cluster`:

```bash
aws ecs update-cluster-settings \
  --cluster innovatech-ecs-cluster \
  --settings name=containerInsights,value=enabled
```

Métricas disponibles en CloudWatch → Container Insights:

- **CPU Utilization** por servicio/tarea
- **Memory Utilization** por servicio/tarea
- **Network I/O** (bytes enviados/recibidos)
- **Task count** (tareas activas)

### Logs del contenedor

Los logs de stdout/stderr del contenedor se envían automáticamente a **CloudWatch Logs**:

```
Log Group: /ecs/back-ventas
Log Stream: ecs/back-ventas/<task-id>
```

### Health Check — Spring Actuator

Spring Boot Actuator expone el endpoint de salud utilizado por el ALB para verificar que la tarea está lista para recibir tráfico:

```
GET /actuator/health
```

Respuesta esperada (sin detalle, por configuración de seguridad):

```json
{
  "status": "UP"
}
```

El ALB marca la tarea como **healthy** cuando responde `200 OK`. Si una tarea falla el health check repetidamente, ECS la reemplaza automáticamente.

---

## 9. Validación funcional

### Verificar que el servicio está operativo

```bash
# Health check via ALB
curl https://innovatech-alb-516038279.us-east-1.elb.amazonaws.com/actuator/health

# Listar ventas
curl https://innovatech-alb-516038279.us-east-1.elb.amazonaws.com/api/v1/ventas

# Crear venta
curl -X POST https://innovatech-alb-516038279.us-east-1.elb.amazonaws.com/api/v1/ventas \
  -H "Content-Type: application/json" \
  -d '{"idCompra": 1, "direccionCompra": "Av. Providencia 123", "valorCompra": 50000}'

# Actualizar venta
curl -X PUT https://innovatech-alb-516038279.us-east-1.elb.amazonaws.com/api/v1/ventas/1 \
  -H "Content-Type: application/json" \
  -d '{"estado": "DESPACHADO"}'
```

### Verificar estado del servicio ECS

```bash
aws ecs describe-services \
  --cluster innovatech-ecs-cluster \
  --services back-ventas-svc \
  --query "services[0].{Running:runningCount,Desired:desiredCount,Status:status}" \
  --output table
```

---

## 10. Ejecución local

Requiere Java 17+ y Maven 3.9+.

### Sin Docker

```bash
cd Springboot-API-REST

# Configurar variables de entorno (base de datos local)
export DB_ENDPOINT=localhost
export DB_PORT=3306
export DB_NAME=ventasdb
export DB_USERNAME=root
export DB_PASSWORD=

mvn spring-boot:run
```

API disponible en `http://localhost:8080/api/v1/ventas`

### Con Docker

```bash
cd Springboot-API-REST

docker build -t back-ventas:local .

docker run -p 8080:8080 \
  -e DB_ENDPOINT=host.docker.internal \
  -e DB_PORT=3306 \
  -e DB_NAME=ventasdb \
  -e DB_USERNAME=root \
  -e DB_PASSWORD= \
  back-ventas:local
```

### Con Docker Compose (stack completo)

```bash
# Desde la raíz del repositorio
docker compose up --build
```

---

## Tecnologías

| Tecnología | Versión | Rol |
|---|---|---|
| Spring Boot | 3.4.4 | Framework principal |
| Spring Data JPA | — | ORM sobre MySQL |
| Spring Boot Actuator | — | Health checks y métricas |
| springdoc-openapi | — | Documentación Swagger |
| MySQL Connector/J | — | Driver JDBC |
| Java | 17 | Runtime |
| Maven | 3.9 | Build y dependencias |
| Docker | multi-stage | Empaquetado |
| GitHub Actions | — | CI/CD |
| Amazon ECS Fargate | — | Ejecución en la nube |
| Amazon ECR | — | Registro de imágenes |
| Amazon RDS | MySQL 8.0 | Base de datos |
| ALB | — | Balanceo y SSL termination |

---

## Incidentes resueltos en producción

---

### Incidente 1 — Timeout en todas las peticiones (ECONNABORTED 10000ms)

**Síntoma:**

El frontend devolvía `timeout of 10000ms exceeded` en todas las peticiones a `/api/v1/ventas`. La aplicación cargaba pero no mostraba datos.

**Causa raíz:**

Las reglas del ALB estaban configuradas con paths incorrectos:

```
Regla incorrecta: /api/ventas* → back-ventas-svc
```

El controller Spring Boot expone:

```java
@RequestMapping("api/v1/ventas")  // path real: /api/v1/ventas
```

Las peticiones a `/api/v1/ventas` no coincidían con la regla `/api/ventas*`. Caían a la regla `default` del ALB, que las enviaba al `frontend-svc`. El nginx del frontend intentaba hacer proxy a una IP de EC2 antigua que no existía en la VPC de ECS. Después de 10 segundos, axios devolvía timeout.

**Solución:**

Se actualizó la regla del ALB via AWS CLI para que el path coincida con el `@RequestMapping` real del controller:

```bash
aws elbv2 modify-rule \
  --rule-arn arn:aws:elasticloadbalancing:...:rule/492229a1e1291e1f \
  --conditions '[{"Field":"path-pattern","Values":["/api/v1/ventas*"]}]'
```

El cambio fue inmediato, sin redeploy.

**Lección aprendida:**

El path de la regla ALB debe coincidir exactamente con el `@RequestMapping` del controller, incluyendo el segmento `/v1/`.

---

### Incidente 2 — Métricas ECS sin datos en CloudWatch Dashboard

**Síntoma:**

CloudWatch mostraba "No hay datos disponibles" para CPU y memoria del servicio.

**Causa raíz:**

`containerInsights: disabled` en el cluster (configuración por defecto de AWS). Sin Container Insights activo, ECS no envía métricas detalladas a CloudWatch.

**Solución:**

```bash
aws ecs update-cluster-settings \
  --cluster innovatech-ecs-cluster \
  --settings name=containerInsights,value=enabled
```

Las métricas aparecieron en CloudWatch ~5 minutos después.

**Lección aprendida:**

Container Insights debe habilitarse explícitamente. No está activo por defecto al crear un cluster ECS.

---

### Incidente 3 — Exposición de información sensible en Actuator y Health Check Grace Period insuficiente

**Síntoma:**

El endpoint `/actuator/health` exponía públicamente, sin autenticación, el motor de base de datos, espacio en disco y rutas internas del sistema de archivos. Adicionalmente, tras desplegar la corrección de seguridad junto con un VPC Endpoint de Secrets Manager configurado el mismo día, el servicio quedó con dos deployments concurrentes sin reconciliar: tareas nuevas eran marcadas como `unhealthy` por el ALB segundos antes de completar su arranque, generando un ciclo de reemplazos fallidos.

**Causa raíz:**

Dos problemas independientes coincidieron:
1. `management.endpoint.health.show-details=always` exponía detalle innecesario del sistema sin autenticación, una superficie de ataque evitable ya que el ALB solo requiere el código HTTP 200/503.
2. `healthCheckGracePeriodSeconds` del servicio ECS estaba en `0`, mientras Spring Boot tarda 44-47 segundos en completar su arranque. El Target Group evaluaba la salud cada 30 segundos con un umbral de 2 intentos fallidos (~30-45s), una ventana que se solapaba con el arranque real de la aplicación.

**Solución:**

```bash
# Fix 1: ocultar detalles del Actuator
# Cambio en application.properties: show-details=always → never

# Fix 2: dar margen de arranque suficiente al health check
aws ecs update-service \
  --cluster innovatech-ecs-cluster \
  --service back-ventas-svc \
  --health-check-grace-period-seconds 90 \
  --region us-east-1
```

**Validación:**

```bash
curl -sk https://innovatech-alb-516038279.us-east-1.elb.amazonaws.com/actuator/health
# {"status":"UP"}

curl -sk https://innovatech-alb-516038279.us-east-1.elb.amazonaws.com/api/v1/ventas
# Respuesta con datos reales desde RDS
```

**Lección aprendida:**

El grace period debe calibrarse según el tiempo de arranque real de la aplicación, no dejarse en su valor por defecto (0). Frameworks con cold start lento como Spring Boot requieren explícitamente este margen. Además, los endpoints de monitoreo deben exponer solo lo mínimo necesario para su función operativa, sin filtrar detalles internos del sistema.

---
