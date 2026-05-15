# Back Ventas — Microservicio de Gestión de Ventas

API REST desarrollada con Spring Boot 3 que gestiona las órdenes de compra del sistema Innovatech Chile. Es uno de los dos microservicios del backend, desplegado en AWS EC2 dentro de una subred **privada** — no es accesible directamente desde Internet.

---

## ¿Qué hace este servicio?

Permite registrar, consultar, actualizar y eliminar órdenes de compra. El frontend lo consume para:
- Listar las ventas que aún no tienen despacho generado
- Marcar una venta como despachada (`despachoGenerado = true`) al crear el despacho correspondiente

Swagger UI disponible en: `http://localhost:8080/swagger-ui.html`

---

## Endpoints

| Método | Ruta | Descripción |
|---|---|---|
| GET | `/api/v1/ventas` | Listar todas las ventas |
| GET | `/api/v1/ventas/{id}` | Obtener venta por ID |
| POST | `/api/v1/ventas` | Crear nueva venta |
| PUT | `/api/v1/ventas/{id}` | Actualizar venta |
| DELETE | `/api/v1/ventas/{id}` | Eliminar venta |

---

## Tecnologías

| Tecnología | Versión | Rol |
|---|---|---|
| Java | 17 | Lenguaje |
| Spring Boot | 3.4.4 | Framework principal |
| Spring Data JPA | 3.4.4 | Acceso a base de datos |
| Hibernate | (incluido en JPA) | ORM |
| MySQL | 8.0 | Base de datos |
| SpringDoc OpenAPI | 2.7.0 | Swagger UI |
| Lombok | 1.18.36 | Reducción de boilerplate |
| Docker | multi-stage | Empaquetado |
| Maven | 3.9 | Gestión de dependencias |

---

## Arquitectura en producción (AWS)

```
Internet
    │
    ▼
┌──────────────────────────────────────────┐
│  ec2-web  (subred pública)               │
│  Elastic IP: 52.73.73.226               │
│  Contenedor: frontend (nginx)            │
│  nginx hace proxy → 10.0.9.120:8080     │
└──────────────────┬───────────────────────┘
                   │ VPC privada
    ┌──────────────▼──────────────────────┐
    │  ec2-app  (10.0.9.120, privada)     │
    │  Contenedor: back-ventas  :8080  ◄──┤
    │  Contenedor: back-despachos :8081   │
    └──────────────┬──────────────────────┘
                   │
    ┌──────────────▼──────────────┐
    │  ec2-datos  (10.0.7.237)    │
    │  MySQL 8.0  :3306           │
    │  Base: ventas_db            │
    └─────────────────────────────┘
```

`ec2-app` no tiene IP pública. Solo es accesible desde dentro de la VPC, lo que significa que el backend nunca está expuesto a Internet directamente.

---

## Estructura del proyecto

```
Springboot-API-REST/
├── src/main/java/com/citt/
│   ├── controller/        # VentaController — endpoints REST
│   ├── persistence/
│   │   ├── entity/        # Venta — entidad JPA (tabla ventas)
│   │   ├── repository/    # VentaRepository — CRUD con Spring Data
│   │   └── services/      # VentaService + VentaServiceImpl
│   ├── exceptions/        # Manejo de errores 404
│   └── config/            # Configuración OpenAPI/Swagger
├── src/main/resources/
│   └── application.properties  # Variables DB_ENDPOINT, DB_NAME, etc.
├── Dockerfile             # Multi-stage: maven builder + JRE runtime
└── pom.xml                # Dependencias Maven
```

---

## Decisiones arquitectónicas

### ¿Por qué Docker con multi-stage build?

```dockerfile
# Stage 1: Build — imagen con Maven para compilar
FROM maven:3.9-eclipse-temurin-17-alpine AS builder
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline -B     # descarga dependencias primero (cache)
COPY src ./src
RUN mvn package -DskipTests -B       # compila el JAR

# Stage 2: Runtime — imagen mínima solo con JRE
FROM eclipse-temurin:17-jre-alpine AS runtime
WORKDIR /app
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
COPY --from=builder /app/target/*.jar app.jar
RUN chown appuser:appgroup app.jar
USER appuser
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

- **Stage 1 (builder):** `maven:3.9-eclipse-temurin-17-alpine` (~500MB) descarga dependencias y compila el JAR. Esta imagen pesada nunca llega a producción.
- **Stage 2 (runtime):** `eclipse-temurin:17-jre-alpine` (~180MB) solo incluye el JRE mínimo para ejecutar el JAR. No contiene Maven, código fuente, ni dependencias de compilación.

El `COPY pom.xml` seguido de `RUN mvn dependency:go-offline` antes de copiar el código fuente aprovecha el **caché de capas de Docker**: si el código cambia pero `pom.xml` no, Docker reutiliza la capa de dependencias y no las vuelve a descargar.

### ¿Por qué usuario no root en el contenedor?

```dockerfile
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser
```

Por defecto, los contenedores Docker corren como root. Si la aplicación fuera comprometida, el atacante tendría control total del contenedor. Corriendo como `appuser` (sin privilegios), el daño potencial queda confinado.

Es el principio de **mínimo privilegio** aplicado a contenedores.

### ¿Por qué `ddl-auto=update`?

En `application.properties`:
```properties
spring.jpa.hibernate.ddl-auto=update
```

Hibernate crea y actualiza automáticamente las tablas en la base de datos al iniciar la aplicación. En un entorno de evaluación/desarrollo esto es conveniente porque no requiere ejecutar scripts SQL manualmente — las tablas se crean solas a partir de las entidades JPA (`@Entity`).

En producción real se usaría `validate` con migraciones controladas (Flyway o Liquibase).

### ¿Por qué variables de entorno para la base de datos?

```properties
spring.datasource.url=jdbc:mysql://${DB_ENDPOINT}:${DB_PORT}/${DB_NAME}?...
spring.datasource.username=${DB_USERNAME}
spring.datasource.password=${DB_PASSWORD}
```

Las credenciales de base de datos nunca se hardcodean en el código. Se pasan como variables de entorno al contenedor en runtime. En el pipeline de GitHub Actions se inyectan desde los **secrets** del repositorio, que son cifrados y nunca visibles en los logs.

### ¿Por qué MySQL 8 y no PostgreSQL?

El proyecto usa el driver JDBC de MySQL (`mysql-connector-j`) y el dialecto de Hibernate configurado para MySQL. MySQL es ampliamente compatible con AWS RDS y es el motor de base de datos más usado en Spring Boot en Latinoamérica.

---

## Variables de entorno

| Variable | Descripción | Ejemplo producción |
|---|---|---|
| `DB_ENDPOINT` | Hostname o IP del servidor MySQL | `10.0.7.237` (ec2-datos) |
| `DB_PORT` | Puerto MySQL | `3306` |
| `DB_NAME` | Nombre de la base de datos | `ventas_db` |
| `DB_USERNAME` | Usuario MySQL | valor en secret |
| `DB_PASSWORD` | Contraseña MySQL | valor en secret |

---

## Pipeline CI/CD

El archivo `.github/workflows/deploy.yml` automatiza el despliegue al hacer `git push` sobre la rama `deploy`:

```
git push → GitHub Actions → Docker Hub → ec2-app (vía SSH proxy)
```

### Pasos del pipeline

```yaml
1. Checkout del repositorio
2. Login a Docker Hub (benjazzx)
3. Build y Push imagen Docker
   - Contexto: ./Springboot-API-REST
   - Publica benjazzx/back-ventas:latest en Docker Hub
4. Despliegue en ec2-app (subred privada, IP 10.0.9.120)
   - Conexión SSH a través de ec2-web como bastion (proxy)
   - docker pull  → descarga nueva imagen
   - docker stop  → para contenedor anterior
   - docker rm    → elimina contenedor anterior
   - docker run   → inicia con variables de entorno DB_*
```

### Patrón bastion (SSH proxy) — decisión clave

`ec2-app` está en una subred **privada** sin IP pública. Para que GitHub Actions pueda conectarse, debe hacer SSH primero a `ec2-web` (subred pública, Elastic IP 52.73.73.226) y desde ahí saltar a `ec2-app`.

```yaml
- uses: appleboy/ssh-action@v1
  with:
    host: ${{ secrets.EC2_HOST }}           # IP privada de ec2-app: 10.0.9.120
    username: ${{ secrets.EC2_USER }}
    key: ${{ secrets.EC2_SSH_KEY }}
    proxy_host: ${{ secrets.EC2_PROXY_HOST }}  # Elastic IP de ec2-web: 52.73.73.226
    proxy_username: ${{ secrets.EC2_USER }}
    proxy_key: ${{ secrets.EC2_SSH_KEY }}
```

`appleboy/ssh-action` implementa un **SSH ProxyCommand** (jump host): abre un túnel SSH hacia `ec2-web` y desde ese túnel abre una segunda conexión SSH hacia `ec2-app`. Todo en una sola acción de GitHub.

### ¿Por qué la misma clave `.pem` para ambas instancias?

En AWS Academy, la misma clave `labsuser.pem` sirve para todas las instancias del laboratorio. Si el laboratorio se reinicia, la clave **cambia** — hay que actualizar el secret `EC2_SSH_KEY` en GitHub con la nueva `.pem`.

Comando para actualizar la clave:
```bash
cat labsuser.pem | gh secret set EC2_SSH_KEY --repo benjazzx/Backend_Ventas
```

### Secrets requeridos en GitHub

| Secret | Descripción |
|---|---|
| `DOCKERHUB_USERNAME` | Usuario de Docker Hub (`benjazzx`) |
| `DOCKERHUB_TOKEN` | Token de acceso Docker Hub |
| `EC2_HOST` | IP privada de ec2-app (`10.0.9.120`) |
| `EC2_PROXY_HOST` | Elastic IP de ec2-web (`52.73.73.226`) |
| `EC2_USER` | Usuario SSH (`ec2-user`) |
| `EC2_SSH_KEY` | Contenido del archivo `.pem` de AWS Academy |
| `DB_ENDPOINT` | IP de ec2-datos (`10.0.7.237`) |
| `DB_USERNAME` | Usuario MySQL |
| `DB_PASSWORD` | Contraseña MySQL |

---

## Cómo ejecutar localmente con Docker Compose

El `docker-compose.yml` levanta MySQL + back-ventas en un solo comando, sin necesidad de instalar MySQL por separado. MySQL persiste sus datos en el volumen nombrado `ventas_mysql_data`.

```bash
# Levantar ambos servicios (MySQL + Spring Boot)
docker compose up --build

# En segundo plano
docker compose up --build -d
```

Servicio disponible en `http://localhost:8080`

### ¿Por qué un volumen nombrado para MySQL?

```yaml
volumes:
  ventas_mysql_data:
    driver: local
```

Sin volumen, los datos de MySQL se pierden al hacer `docker compose down`. Con el volumen nombrado `ventas_mysql_data`, Docker persiste los archivos de la base de datos en el host. Al volver a levantar el stack, MySQL recupera todos los datos anteriores.

Diferencia clave:
- **Bind mount** (`./data:/var/lib/mysql`): mapea a una carpeta específica del host — depende del sistema de archivos del desarrollador.
- **Volumen nombrado** (`ventas_mysql_data:/var/lib/mysql`): Docker gestiona el almacenamiento internamente — portátil, funciona igual en cualquier máquina.

### ¿Por qué `depends_on` con `healthcheck`?

```yaml
depends_on:
  mysql:
    condition: service_healthy
```

Spring Boot falla al iniciar si MySQL aún no está listo para aceptar conexiones. El `healthcheck` de MySQL ejecuta `mysqladmin ping` cada 10 segundos. Docker Compose solo inicia `back-ventas` cuando MySQL responde correctamente — sin necesidad de `sleep` artificiales en el entrypoint.

### ¿Por qué `DB_ENDPOINT: mysql` en el compose?

En Docker Compose, los servicios se comunican por nombre de servicio dentro de la red interna que Compose crea automáticamente. El hostname `mysql` resuelve internamente a la IP del contenedor MySQL — no hay que hardcodear IPs.

En producción (AWS), `DB_ENDPOINT` es `10.0.7.237` (ec2-datos). En desarrollo local, es `mysql` (nombre del servicio en Compose).

---

## Cómo ejecutar localmente (sin Docker)

Requiere MySQL corriendo localmente con la base de datos `ventas_db` creada.

```bash
cd Springboot-API-REST
./mvnw spring-boot:run \
  -Dspring-boot.run.jvmArguments="-DDB_ENDPOINT=localhost -DDB_PORT=3306 -DDB_NAME=ventas_db -DDB_USERNAME=root -DDB_PASSWORD=password"
```

Servicio disponible en `http://localhost:8080`
