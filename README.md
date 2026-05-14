# Back Ventas — Microservicio de Gestión de Ventas

API REST desarrollada con Spring Boot que gestiona las órdenes de compra del sistema Innovatech Chile. Es uno de los dos microservicios que componen el backend del proyecto, desplegado en AWS EC2 dentro de una subred privada.

---

## ¿Qué hace este servicio?

Permite registrar, consultar, actualizar y eliminar órdenes de compra. El frontend lo consume para:
- Listar las ventas que aún no tienen despacho generado
- Marcar una venta como despachada (`despachoGenerado = true`) una vez que se crea el despacho correspondiente

---

## Endpoints

| Método | Ruta | Descripción |
|---|---|---|
| GET | /api/v1/ventas | Listar todas las ventas |
| GET | /api/v1/ventas/{id} | Obtener venta por ID |
| POST | /api/v1/ventas | Crear nueva venta |
| PUT | /api/v1/ventas/{id} | Actualizar venta |
| DELETE | /api/v1/ventas/{id} | Eliminar venta |

Swagger UI disponible en: `http://localhost:8080/swagger-ui.html`

---

## Tecnologías

- Java 17
- Spring Boot 3.4.4
- Spring Data JPA + Hibernate
- MySQL 8.0
- Docker (multi-stage build)

---

## Variables de entorno

| Variable | Descripción | Ejemplo |
|---|---|---|
| DB_ENDPOINT | Host del servidor MySQL | mysql / IP EC2 |
| DB_PORT | Puerto MySQL | 3306 |
| DB_NAME | Nombre de la base de datos | ventas_db |
| DB_USERNAME | Usuario MySQL | innovatech |
| DB_PASSWORD | Contraseña MySQL | tu_password |

---

## Decisiones arquitectónicas

### ¿Por qué Docker con multi-stage build?
El Dockerfile usa dos etapas separadas:
- **Stage 1 (builder):** imagen `maven:3.9-eclipse-temurin-17-alpine` para compilar el JAR. Esta imagen es pesada (~500MB) pero solo se usa en tiempo de build.
- **Stage 2 (runtime):** imagen `eclipse-temurin:17-jre-alpine`, que solo incluye el JRE necesario para ejecutar el JAR (~180MB). El resultado final es una imagen pequeña y sin herramientas de compilación innecesarias.

**Beneficio:** la imagen de producción es significativamente más liviana y tiene menor superficie de ataque de seguridad.

### ¿Por qué usuario no root?
Se crea un usuario `appuser` sin privilegios dentro del contenedor:
```dockerfile
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser
```
Si un atacante logra explotar una vulnerabilidad en la aplicación, no tendrá acceso root al contenedor ni al sistema operativo del host. Es una práctica de mínimo privilegio requerida en entornos productivos.

### ¿Por qué MySQL 8.0 y no PostgreSQL?
El proyecto fue entregado con el driver JDBC de MySQL (`mysql-connector-j`) y el dialecto de Hibernate configurado para MySQL. Cambiar a PostgreSQL requeriría modificar `pom.xml`, `application.properties` y potencialmente el esquema. MySQL es ampliamente usado en aplicaciones Spring Boot y es compatible con AWS RDS.

### ¿Por qué `allowPublicKeyRetrieval=true` en la URL JDBC?
MySQL 8 cambió su plugin de autenticación por defecto a `caching_sha2_password`, que requiere recuperar la clave pública del servidor para autenticarse en conexiones no SSL. Este parámetro habilita esa recuperación. Sin él, el conector JDBC lanza `Public Key Retrieval is not allowed` al intentar conectarse.

### ¿Por qué `--default-authentication-plugin=mysql_native_password`?
Para garantizar compatibilidad total con el conector JDBC en un entorno Docker sin SSL configurado, se fuerza el plugin de autenticación clásico de MySQL. Esto asegura que el usuario `innovatech` sea creado con ese plugin desde el inicio, evitando problemas de handshake de autenticación.

### ¿Por qué `ddl-auto=update`?
Hibernate actualiza automáticamente el esquema de la base de datos al iniciar la aplicación. En un entorno de desarrollo y evaluación esto es conveniente porque no requiere ejecutar scripts SQL para crear las tablas — se crean solas a partir de las entidades JPA. En producción real se usaría `validate` o migraciones con Flyway/Liquibase.

---

## Cómo ejecutar localmente

### Con Docker (recomendado)

Desde la raíz del proyecto semestral:

```bash
cp .env.example .env   # editar con tus valores
docker compose up --build
```

Servicio disponible en `http://localhost:8080`

### Sin Docker

Requiere MySQL corriendo localmente con la base de datos `ventas_db` creada. Configurar en `application.properties`:

```properties
DB_ENDPOINT=localhost
DB_PORT=3306
DB_NAME=ventas_db
DB_USERNAME=tu_usuario
DB_PASSWORD=tu_password
```

Luego:
```bash
./mvnw spring-boot:run
```

---

## Pipeline CI/CD

El archivo `.github/workflows/deploy.yml` automatiza el despliegue al hacer `git push` sobre la rama `deploy`:

1. **Build:** compila la imagen Docker usando el Dockerfile multi-stage
2. **Push:** publica la imagen en Docker Hub como `{usuario}/back-ventas:latest`
3. **Deploy:** se conecta por SSH a la instancia EC2 del backend, descarga la nueva imagen y reinicia el contenedor

### Secrets requeridos en GitHub

| Secret | Descripción |
|---|---|
| DOCKERHUB_USERNAME | Usuario de Docker Hub |
| DOCKERHUB_TOKEN | Token de acceso Docker Hub |
| EC2_HOST | IP de la instancia EC2 backend |
| EC2_USER | Usuario SSH (ec2-user o ubuntu) |
| EC2_SSH_KEY | Contenido del archivo .pem |
| DB_ENDPOINT | IP o hostname del MySQL en EC2 |
| DB_USERNAME | Usuario MySQL |
| DB_PASSWORD | Contraseña MySQL |

---

## Estructura del proyecto

```
Springboot-API-REST/
├── src/main/java/com/citt/
│   ├── controller/        # VentaController — endpoints REST
│   ├── persistence/
│   │   ├── entity/        # Venta — entidad JPA
│   │   ├── repository/    # VentaRepository — acceso a datos
│   │   └── services/      # VentaService + VentaServiceImpl
│   ├── exceptions/        # Manejo de errores 404
│   └── config/            # Configuración OpenAPI/Swagger
├── src/main/resources/
│   └── application.properties
├── Dockerfile
└── pom.xml
```
