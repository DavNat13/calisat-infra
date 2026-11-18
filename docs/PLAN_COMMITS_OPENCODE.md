# Plan de repositorios y commits (tarea exclusiva de **opencode**)

> Esta tarea la ejecuta **solo opencode**. Claude **no** hizo ningún commit: todos los cambios están sin commitear en `C:\Users\David\Desktop\Duoc2026\DC1`.

## 1. Cuántos repositorios quedamos

**15 repositorios** = 9 existentes + 6 nuevos.

| # | Repo | Estado | Carpeta local | Remote (GitHub `DavNat13/…`) |
|---|---|---|---|---|
| 1 | `calisat-frontend` | existente, modificado | `calisat-frontend` | ya configurado |
| 2 | `calisat-ms-usuarios` | existente, modificado | `calisat-ms-usuarios` | ya configurado |
| 3 | `calisat-ms-catalogo` | existente, modificado | `calisat-ms-catalogo` | ya configurado |
| 4 | `calisat-ms-carrito` | existente, modificado | `calisat-ms-carrito` | ya configurado |
| 5 | `calisat-ms-orden` | existente, modificado | `calisat-ms-orden` | ya configurado |
| 6 | `calisat-ms-envios` | existente, modificado | `calisat-ms-envios` | ya configurado |
| 7 | `calisat-ms-notificaciones` | existente, modificado | `calisat-ms-notificaciones` | ya configurado |
| 8 | `calisat-ms-inventario` | existente, modificado (1 archivo) | `calisat-ms-inventario` | ya configurado |
| 9 | `calisat-validador-lamdba` | existente, **sin cambios** | `calisat-validador-lambda` | **no se toca** |
| 10 | `calisat-infra` | **NUEVO** | `calisat-infra` | crear en GitHub |
| 11 | `calisat-ms-rabbit-admin` | **NUEVO** | `calisat-ms-rabbit-admin` | crear en GitHub |
| 12 | `calisat-ms-kafka-admin` | **NUEVO** | `calisat-ms-kafka-admin` | crear en GitHub |
| 13 | `calisat-ms-pago` | **NUEVO** | `calisat-ms-pago` | crear en GitHub |
| 14 | `calisat-ms-analytics` | **NUEVO** | `calisat-ms-analytics` | crear en GitHub |
| 15 | `calisat-aws-lambdas` | **NUEVO** | `calisat-aws-lambdas` | crear en GitHub |

Además, `C:\Users\David\Desktop\Duoc2026\DC1\README.md` (visión general) **no es un repo**: copiarlo a `calisat-infra/docs/README_PLATAFORMA.md` y commitearlo en `calisat-infra`.

## 2. Reglas generales para opencode

1. **Antes de empezar, en cada repo**: `git status`, `git log --oneline -5`, y **leer la versión actual** (`pom.xml` → `<version>` del proyecto; frontend → `package.json` `version`) y el último tag (`git tag --sort=-v:refname | head`). Esa es la **versión base**; la nueva se calcula con SemVer sobre ella (no usar los números de esta guía a ciegas: si la versión actual difiere, recalcula).
2. **Qué sube cada tipo de cambio (SemVer)**
   - `fix:` → PATCH (1.3.1 → 1.3.2)
   - `feat:` (compatible) → MINOR (1.3.0 → 1.4.0)
   - `feat!:` / `fix!:` / cambio que **rompe compatibilidad** (p. ej. cambia el tipo de exchange de RabbitMQ, cambia un contrato HTTP) → MAJOR (2.4.0 → 3.0.0)
   - `docs:`, `test:`, `chore:`, `refactor:` → sin bump por sí solos
3. **Estilo de mensaje**: Conventional Commits en español, igual que el historial (`feat: …`, `fix: …`). Una línea ≤ 72 caracteres; cuerpo opcional con el porqué.
4. **Bump de versión** = commit propio `chore(release): vX.Y.Z` que cambia `pom.xml`/`package.json`, **agrega entrada en `CHANGELOG.md`** (si el repo lo versiona) y se etiqueta: `git tag -a vX.Y.Z -m "…"`.
5. **Cada tag debe compilar y pasar tests** (`./mvnw clean test` / `npm run lint && npm run build`). Si un commit intermedio no compila porque comparte `pom.xml`/`application.yaml`, **agrúpalo con el anterior**; lo que importa es que cada *release* (tag) esté verde.
6. **Nunca** `git add .` a ciegas: añadir por rutas (`git add <ruta>`). Verificar con `git diff --cached --stat` antes de cada commit.
7. **No commitear**: `target/`, `node_modules/`, `dist/`, `*.jar`, `.env`, `*.pem`, `HELP.md`. Los `.gitignore` de los repos nuevos ya los excluyen; comprobar con `git status --ignored`.
8. Push: `git push origin main --follow-tags`. Repos nuevos: `git init -b main`, `git remote add origin https://github.com/DavNat13/<repo>.git` (crear antes el repo vacío en GitHub), primer push con `-u`.
9. Al terminar cada repo: `git status` limpio y `git log --oneline` revisado.

## 3. Versiones resultantes (calculadas sobre la versión actual al 2026-11-18)

| Repo | Versión actual | Nueva | Motivo del salto |
|---|---|---|---|
| calisat-ms-usuarios | 1.4.0 | **2.0.0** | MAJOR: exchange `calisat.exchange` pasa de direct a **topic** (incompatible con brokers ya declarados) |
| calisat-ms-catalogo | 1.3.0 | **1.4.0** | MINOR: feat Kafka (evento `view`) |
| calisat-ms-carrito | 2.1.0 | **2.2.0** | MINOR: feat Kafka + fix JWT/rol |
| calisat-ms-orden | 2.2.0 | **3.0.0** | MAJOR: exchange topic **y** nuevo contrato (409 si no hay stock) |
| calisat-ms-envios | 2.3.0 | **3.0.0** | MAJOR: exchange topic |
| calisat-ms-notificaciones | 2.4.0 | **3.0.0** | MAJOR: exchange topic + se elimina la cola `inventario.alertas.queue` |
| calisat-ms-inventario | 1.3.1 | **1.3.2** | PATCH: CORS |
| calisat-frontend | 2.4.0 | **2.5.0** | MINOR: pago, métricas, paneles |
| calisat-validador-lamdba | 1.0.0 | 1.0.0 | sin cambios |
| calisat-infra | — | **1.0.0 → 1.1.0** | nuevo |
| calisat-ms-rabbit-admin | — | **1.0.0** | nuevo |
| calisat-ms-kafka-admin | — | **1.0.0 → 1.1.0** | nuevo |
| calisat-ms-pago | — | **1.0.0 → 1.1.0 → 1.2.0** | nuevo |
| calisat-ms-analytics | — | **1.0.0 → 1.1.0 → 1.2.0** | nuevo |
| calisat-aws-lambdas | — | **1.0.0** | nuevo |

## 4. Orden recomendado de trabajo
1. `calisat-infra` (los demás lo referencian) → 2. servicios existentes (RabbitMQ) → 3. nuevos servicios → 4. productores Kafka en catalogo/carrito/orden → 5. frontend → 6. aws-lambdas.

---

## 5. Repos EXISTENTES — commits por repo

> Rutas relativas a cada repo. Los nombres de commit son sugeridos; ajustar al estilo del historial.

### 5.1 calisat-ms-usuarios (1.4.0 → **2.0.0**)
1. `feat!: exchange calisat.exchange de tipo topic en el productor de usuarios`
   - `src/main/java/com/califorge/msusuarios/config/RabbitConfig.java`
2. `fix: no perder el registro de usuario si falla la publicación en RabbitMQ`
   - `src/main/java/com/califorge/msusuarios/controller/UsuarioController.java`
3. `feat: broker RabbitMQ configurable por entorno y publisher confirms`
   - `src/main/resources/application.yaml` (host/usuario/clave por `SPRING_RABBITMQ_*`, `publisher-confirm-type: correlated`)
4. `fix: aceptar Idempotency-Key en CORS`
   - `src/main/java/com/califorge/msusuarios/config/SecurityConfig.java`
5. `fix: usar la red calisat-net y el broker del clúster; reactivar health de RabbitMQ`
   - `docker-compose.yml`
6. `chore(release): v2.0.0` → `pom.xml` (+ `CHANGELOG.md`) · tag `v2.0.0`
> Pendiente conocido: el repo no tiene tests (el audit lo marca). Opcional: commit `test:` aparte con tests del controlador.

### 5.2 calisat-ms-catalogo (1.3.0 → **1.4.0**)
1. `feat: dependencia spring-kafka y configuración del productor de interacciones`
   - `pom.xml`, `src/main/resources/application.yaml`, `src/main/java/com/calisat/mscatalogo/config/KafkaProducerConfig.java`, `src/main/java/com/calisat/mscatalogo/dto/InteraccionEvent.java`
2. `feat: publicar evento view en Kafka (interacciones-user) al consultar un producto`
   - `src/main/java/com/calisat/mscatalogo/service/InteraccionPublisher.java`, `src/main/java/com/calisat/mscatalogo/controller/ProductoController.java`
3. `fix: aceptar Idempotency-Key en CORS`
   - `src/main/java/com/calisat/mscatalogo/config/SecurityConfig.java`
4. `chore(release): v1.4.0` · tag `v1.4.0` (la versión del `pom.xml` va en este commit; cuidado: el commit 1 también toca `pom.xml`, deja la versión sin cambiar hasta el commit 4)

### 5.3 calisat-ms-carrito (2.1.0 → **2.2.0**)
1. `fix: AudienceValidator acepta client_id de access tokens de Cognito`
   - `src/main/java/com/califorge/mscarrito/config/AudienceValidator.java`
2. `fix: rol por defecto CLIENTE para usuarios Entra sin claim roles (evita 403)` + `fix: Idempotency-Key en CORS`
   - `src/main/resources/application.yaml` (solo el bloque `default-role`), `src/main/java/com/califorge/mscarrito/config/SecurityConfig.java`
3. `feat: dependencia spring-kafka y productor de interacciones`
   - `pom.xml`, `config/KafkaProducerConfig.java`, `dto/InteraccionEvent.java` (+ `spring.kafka.bootstrap-servers` en `application.yaml`)
4. `feat: publicar evento cart en Kafka al agregar un ítem`
   - `service/InteraccionPublisher.java`, `service/CarritoService.java`
5. `chore(release): v2.2.0` · tag `v2.2.0`

### 5.4 calisat-ms-orden (2.2.0 → **3.0.0**)
1. `feat!: exchange calisat.exchange de tipo topic en el productor de órdenes`
   - `config/RabbitConfig.java`
2. `feat: broker RabbitMQ configurable por entorno y publisher confirms`
   - `src/main/resources/application.yaml` (solo bloque `rabbitmq`), `docker-compose.yml`
3. `fix: AudienceValidator acepta client_id de access tokens de Cognito`
   - `config/AudienceValidator.java`
4. `feat!: abortar la orden con 409 si el inventario rechaza la reserva (evita sobreventa, OR-05)`
   - `exception/StockInsuficienteException.java`, `client/InventarioClient.java`, `service/OrdenService.java` (bloque de reserva/compensación), `exception/GlobalExceptionHandler.java`
5. `feat: dependencia spring-kafka y productor de interacciones`
   - `pom.xml`, `config/KafkaProducerConfig.java`, `dto/InteraccionEvent.java`, `service/InteraccionPublisher.java`, `application.yaml` (`spring.kafka`)
6. `feat: publicar evento purchase en Kafka al pagar la orden`
   - `service/OrdenService.java` (bloque `PAGADA`)
7. `chore(release): v3.0.0` · tag `v3.0.0` — *(rutas bajo `src/main/java/com/califorge/msorden/`)*
> Nota: los commits 4 y 6 tocan el mismo archivo (`OrdenService.java`); usa `git add -p` para separarlos o agrúpalos en uno.

### 5.5 calisat-ms-envios (2.3.0 → **3.0.0**)
1. `feat!: exchange calisat.exchange de tipo topic en el productor de envíos` — `src/main/java/com/calisat/msenvios/config/RabbitConfig.java`
2. `feat: broker RabbitMQ configurable por entorno y publisher confirms` — `src/main/resources/application.yaml`, `docker-compose.yml`
3. `fix: AudienceValidator acepta client_id de access tokens de Cognito` — `config/AudienceValidator.java`
4. `fix: aceptar Idempotency-Key en CORS` — `config/SecurityConfig.java`
5. `chore(release): v3.0.0` · tag `v3.0.0`

### 5.6 calisat-ms-notificaciones (2.4.0 → **3.0.0**)
1. `feat!: exchange calisat.exchange topic y eliminación de la cola inventario.alertas.queue (sin consumidor)`
   - `src/main/java/com/califorge/msnotificaciones/config/RabbitConfig.java`, `src/test/java/com/califorge/msnotificaciones/config/RabbitConfigTest.java`
2. `feat: consumidor de la DLQ que registra en logs los mensajes no entregados`
   - `src/main/java/com/califorge/msnotificaciones/listener/DlqListener.java`
3. `feat: reintentos del consumidor (3, backoff exponencial), requeue deshabilitado y broker por entorno`
   - `src/main/resources/application.yaml` (solo bloque `rabbitmq`)
4. `fix: usar el clúster RabbitMQ de calisat-infra en lugar de un broker local guest/guest`
   - `docker-compose.yml`
5. `chore(release): v3.0.0` · tag `v3.0.0`

### 5.7 calisat-ms-inventario (1.3.1 → **1.3.2**)
1. `fix: aceptar Idempotency-Key en CORS` — `src/main/java/com/califorge/msinventario/config/SecurityConfig.java`
2. `chore(release): v1.3.2` · tag `v1.3.2`

### 5.8 calisat-frontend (2.4.0 → **2.5.0**)
1. `feat: servicio y panel de pago con WebPay en el checkout`
   - `src/modules/pagos/` (service, `PagoWebpay.jsx`), `src/modules/carrito/pages/CheckoutPage.jsx`
2. `feat: página de retorno de WebPay que confirma el pago y marca la orden como pagada`
   - `src/modules/pagos/pages/RetornoPagoPage.jsx`, `src/routes/PublicRoutes.jsx`, `src/routes/lazyPages.js` (entrada `RetornoPagoPage`)
3. `feat: recomendaciones personalizadas (Top-5 de ms-analytics) en la ficha de producto`
   - `src/modules/catalogo/services/recomendacionService.js`, `src/modules/catalogo/hooks/useProductoDetalle.js`
4. `feat: dashboard de métricas en tiempo real por WebSocket y alertas de fraude`
   - `src/modules/admin/pages/MetricasPage.jsx`, `src/modules/admin/pages/PlataformaAdmin.css`, `src/config/api.js`, `src/modules/admin/services/apiAdmin.js` (`getToken`)
5. `feat: paneles de administración de RabbitMQ y Kafka`
   - `src/modules/admin/pages/ColasRabbitPage.jsx`, `KafkaPage.jsx`, `src/modules/admin/layout/AdminSidebar.jsx`, `src/routes/AdminRoutes.jsx`, `src/routes/lazyPages.js` (entradas admin)
6. `fix: CSP permite el POST a WebPay y las conexiones WebSocket` — `nginx.conf`
7. `chore(release): v2.5.0` · `package.json` (+ lock si cambia) + `CHANGELOG.md` · tag `v2.5.0`
> `lazyPages.js` se toca en 2 y 5: usar `git add -p`. Verificar antes de cada tag: `npm ci && npm run lint && npm run build`.

### 5.9 calisat-validador-lamdba — **sin commits** (no hubo cambios; las rutas nuevas se configuran en la consola de API Gateway, ver `ENDPOINTS_API_GATEWAY.md`).

---

## 6. Repos NUEVOS — commits por repo (`git init -b main`)

> Cada repo nuevo incluye: `.gitignore`, `.gitattributes`, `mvnw`, `mvnw.cmd`, `.mvn/`, `Dockerfile`, `pom.xml`, `docker-compose.yml`, `README.md`, `src/`. **No** subir `target/`.

### 6.1 calisat-infra (1.0.0 → 1.1.0)
1. `chore: estructura inicial del repositorio de infraestructura` — `README.md`
2. `feat: docker-compose del clúster RabbitMQ de 3 nodos` — `rabbitmq-cluster/docker-compose.yml`
3. `feat: docker-compose de Apache Kafka con Zookeeper (tópicos sin autocreación)` — `kafka/docker-compose.yml`
4. `feat: docker-compose de Kafka-UI` — `kafka-ui/docker-compose.yml`
5. `chore(release): v1.0.0` · tag `v1.0.0`
6. `docs: guía de despliegue en AWS (security groups, API Gateway, DynamoDB, Kafka)` — `docs/GUIA_DESPLIEGUE_AWS.md`
7. `docs: listado de endpoints para rutas del API Gateway` — `docs/ENDPOINTS_API_GATEWAY.md`
8. `docs: plan de commits y visión general de la plataforma` — `docs/PLAN_COMMITS_OPENCODE.md`, `docs/README_PLATAFORMA.md` (copia de `DC1/README.md`)
9. `chore(release): v1.1.0` · tag `v1.1.0`

### 6.2 calisat-ms-rabbit-admin (1.0.0)
1. `chore: proyecto Spring Boot 4.1 (Java 21) con Docker y Maven Wrapper` — `pom.xml`, `mvnw*`, `.mvn/`, `Dockerfile`, `.gitignore`, `.gitattributes`, `CalisatMsRabbitAdminApplication.java`
2. `feat: validación JWT dual Entra ID + Cognito y rol ADMINISTRADOR` — `src/main/java/com/califorge/msrabbitadmin/config/*`, `src/main/resources/application.yaml`
3. `feat: crear, listar y eliminar colas, exchanges y bindings (AmqpAdmin + Management API)` — `dto/`, `service/`, `controller/`, `exception/`
4. `test: pruebas del servicio de administración` — `src/test/…/RabbitAdminServiceTest.java`
5. `chore: docker-compose y README` — `docker-compose.yml`, `README.md`
6. `chore(release): v1.0.0` · tag `v1.0.0`

### 6.3 calisat-ms-kafka-admin (1.0.0 → 1.1.0)
1. `chore: proyecto Spring Boot 4.1 (Java 21) con Docker y Maven Wrapper` — igual que 6.2 con `CalisatMsKafkaAdminApplication.java`
2. `feat: seguridad JWT dual y AdminClient configurable` — `config/*` (incl. `KafkaAdminConfig.java`), `application.yaml`
3. `feat: administración de tópicos (crear, listar, describir, ampliar particiones, configs, eliminar)` — `dto/Topic*.java`, `service/TopicAdminService.java`, `service/KafkaAdminException.java`
4. `chore(release): v1.0.0` · tag `v1.0.0`
5. `feat: administración de consumer groups (listar, describir con lag, reiniciar offsets, eliminar)` — `service/ConsumerGroupAdminService.java`, `dto/ResetOffsetsRequest.java`, `controller/KafkaAdminController.java`, `exception/GlobalExceptionHandler.java`
6. `test: pruebas de TopicAdminService con AdminClient simulado` — `src/test/…`
7. `chore: docker-compose y README` — `docker-compose.yml`, `README.md`
8. `chore(release): v1.1.0` · tag `v1.1.0`
> Para que el tag 1.0.0 compile, el controlador de la 1.0.0 debe contener solo los endpoints de tópicos; si es complicado separarlo, haz un único release `v1.0.0` con todo.

### 6.4 calisat-ms-pago (1.0.0 → 1.1.0 → 1.2.0)
1. `chore: proyecto Spring Boot 4.1 (Java 21) con Docker y Maven Wrapper` — `pom.xml` (con jpa, postgres, kafka, transbank-sdk-java 6.2.1), `mvnw*`, `.mvn/`, `Dockerfile`, `.gitignore`
2. `feat: seguridad JWT dual y configuración (Postgres, Kafka, WebPay)` — `config/` (JWT/Security/AppConfig), `application.yaml`
3. `feat: modelo de pagos y repositorios (Pago, EstadoPagoCliente, AlertaFraude, IntentoPago)` — `model/`, `repository/`
4. `feat: integración WebPay Plus (Transbank integración) con modo simulado` — `service/Webpay*.java`, `PagoGatewayException`, `TarjetaUtil`
5. `feat: API de pagos iniciar/confirmar/mis-pagos` — `dto/`, `service/PagoService.java`, `controller/PagoController.java`, `exception/`
6. `chore(release): v1.0.0` · tag `v1.0.0`
7. `feat: antifraude Velocity Check (>3 intentos de la misma tarjeta en 10 s → bloqueo 30 min, HTTP 429)` — `service/VelocityCheckService.java`, `test/VelocityCheckServiceTest.java`
8. `feat: productor Kafka de intentos y alertas (key = hash de tarjeta) con DTOs tipados` — `config/KafkaConfig.java`, `config/KafkaTopicsConfig.java`, `dto/IntentoPagoEvent.java`, `dto/AlertaFraudeEvent.java`
9. `chore(release): v1.1.0` · tag `v1.1.0`
10. `feat: consumidores Kafka en dos grupos (auditoría y alertas) delegando a servicios` — `listener/PagoKafkaListeners.java`, `service/AuditoriaPagoService.java`, endpoint `/alertas`
11. `feat: errores de streaming: reintentos con backoff y dead-letter topics` — `KafkaConfig` (error handler)
12. `test: integración con Kafka embebido (tipado, grupos, DLT)` — `test/KafkaIntegrationTest.java`
13. `chore: docker-compose y README` — `docker-compose.yml`, `README.md`
14. `chore(release): v1.2.0` · tag `v1.2.0`
> Si un tag intermedio no compila por dependencias cruzadas (PagoService ↔ Velocity/Kafka), fusionar los commits 4-5-7-8 en un único release `v1.0.0`; lo obligatorio es que cada tag compile.

### 6.5 calisat-ms-analytics (1.0.0 → 1.1.0 → 1.2.0)
1. `chore: proyecto Spring Boot 4.1 (Java 21) con Docker y Maven Wrapper` — `pom.xml` (jpa, postgres, kafka, websocket, dynamodb-enhanced, spring-kafka-test), `mvnw*`, `.mvn/`, `Dockerfile`, `.gitignore`
2. `feat: seguridad JWT dual y configuración` — `config/` JWT/Security/AppConfig, `application.yaml`
3. `feat: modelo y repositorios de interacciones y recomendaciones` — `model/`, `repository/`, `store/InteraccionStore.java`, `store/JpaInteraccionStore.java`
4. `feat: motor de recomendaciones Top-5 (view=1, cart=2, purchase=3) y API REST` — `service/RecomendacionService.java`, `dto/`, `controller/RecomendacionController.java`, `test/RecomendacionServiceTest.java`
5. `chore(release): v1.0.0` · tag `v1.0.0`
6. `feat: consumidores Kafka de interacciones-user en dos consumer groups y de payment-alerts` — `config/KafkaConfig.java`, `config/KafkaTopicsConfig.java`, `listener/AnalyticsKafkaListeners.java`, `dto/InteraccionEvent.java`, `dto/AlertaFraudeEvent.java`
7. `feat: manejo de errores de streaming (reintentos, dead-letter topic, validación de payload)` — `KafkaConfig`
8. `chore(release): v1.1.0` · tag `v1.1.0`
9. `feat: métricas en tiempo real y WebSocket /ws/metricas con JWT en el handshake` — `service/MetricasService.java`, `ws/*`, `dto/MetricasSnapshot.java`, `test/MetricasServiceTest.java`
10. `feat: almacén opcional en DynamoDB (perfil dynamodb, tabla UserInteractions)` — `store/DynamoInteraccionStore.java`, `store/InteraccionDynamoItem.java`, `config/DynamoConfig.java`
11. `test: integración con Kafka embebido (JSON tipado, grupos y DLT)` — `test/KafkaIntegrationTest.java`
12. `chore: docker-compose y README` — `docker-compose.yml`, `README.md`
13. `chore(release): v1.2.0` · tag `v1.2.0`

### 6.6 calisat-aws-lambdas (1.0.0)
1. `chore: proyecto Maven Java 21 con shade para AWS Lambda` — `pom.xml`, `mvnw*`, `.mvn/`, `.gitignore`
2. `feat: handlers WebSocket $connect y $disconnect (tabla ConexionesWebSocket)` — `src/main/java/com/califorge/lambdas/WebSocketConnectHandler.java`, `WebSocketDisconnectHandler.java`
3. `feat: handler de DynamoDB Streams con broadcast a los dashboards conectados` — `InteraccionStreamHandler.java`
4. `docs: guía de despliegue de las Lambdas` — `README.md`
5. `chore(release): v1.0.0` · tag `v1.0.0`

---

## 7. Verificación final (opencode)
```bash
# En cada repo Java
./mvnw clean test            # BUILD SUCCESS
# Frontend
npm ci && npm run lint && npm run build
# Todos
git status                   # limpio
git tag --list               # tags de la sección 3
git log --oneline --decorate -15
```
Entregar al docente (AVA + correo) los **15 enlaces** de GitHub (14 si se excluye el authorizer, que no cambió, pero es parte del sistema). Indicar en el correo qué repos son nuevos en la EV3: `calisat-infra`, `calisat-ms-rabbit-admin`, `calisat-ms-kafka-admin`, `calisat-ms-pago`, `calisat-ms-analytics`, `calisat-aws-lambdas`.
