# Guía de despliegue en AWS (paso a paso)

Región **us-east-1** (AWS Academy / Learner Lab, rol `LabRole`, perfil de instancia `LabInstanceProfile`).
Sigue el patrón de las guías del profesor: **un contenedor por máquina EC2**, API Gateway como único punto público.

> Convenciones: `<IP-xxx>` = IP **privada** de la EC2 (dentro de la VPC); `<IP-PUB-xxx>` = IP pública.
> En Learner Lab las IP cambian al detener/iniciar la sesión: asigna **Elastic IP** a Kafka y a las EC2 que expone el API Gateway, o actualiza las integraciones tras cada reinicio.

---

## 0. Mapa de lo que se despliega

| # | Máquina EC2 (nombre) | Tipo | Contenido | Puertos que escucha |
|---|---|---|---|---|
| 1 | `calisat-rabbit` | t3.small | RabbitMQ (1 nodo o clúster) | 5672, 15672 |
| 2 | `calisat-kafka` | **t3.medium** (Kafka+ZK+UI consumen ~2 GB) | Zookeeper + Kafka + Kafka-UI | 2181, 9092, 8080 |
| 3 | `calisat-usuarios` | t3.small | ms-usuarios + su Postgres | 8081 |
| 4 | `calisat-catalogo` | t3.small | ms-catalogo + Postgres | 8082 |
| 5 | `calisat-inventario` | t3.small | ms-inventario + Postgres | 8083 |
| 6 | `calisat-carrito` | t3.small | ms-carrito + Postgres | 8084 |
| 7 | `calisat-orden` | t3.small | ms-orden + Postgres | 8085 |
| 8 | `calisat-envios` | t3.small | ms-envios + Postgres | 8086 |
| 9 | `calisat-notificaciones` | t3.small | ms-notificaciones + Postgres | 8087 |
| 10 | `calisat-rabbit-admin` | t3.small | ms-rabbit-admin | 8088 |
| 11 | `calisat-pago` | t3.small | ms-pago + Postgres | 8089 |
| 12 | `calisat-analytics` | t3.small | ms-analytics + Postgres | 8090 |
| 13 | `calisat-kafka-admin` | t3.small | ms-kafka-admin | 8091 |
| 14 | `calisat-frontend` | t3.micro/small | frontend (Nginx) | 80 |

Si el Learner Lab limita instancias, agrupa: (a) RabbitMQ + rabbit-admin, (b) Kafka + kafka-admin, (c) varios microservicios por EC2 con sus puertos distintos (cada uno ya tiene un puerto único).

Servicios gestionados (no son EC2): **API Gateway HTTP API**, **API Gateway WebSocket**, **Lambda** (authorizer + 3 lambdas WebSocket), **DynamoDB**, Cognito, Entra ID (ya existentes).

---

## 1. Red y grupos de seguridad (Security Groups)

Usa la **VPC por defecto**. Crea **un Security Group por rol** (EC2 → Security Groups → Create). Regla general: **solo el API Gateway/Internet entra a los servicios HTTP; los brokers solo reciben tráfico de la VPC**.

### 1.1 `sg-microservicios` (todas las EC2 de microservicios 8081-8091)
| Tipo | Puerto | Origen | Motivo |
|---|---|---|---|
| TCP personalizado | 8081-8091 | `0.0.0.0/0` | El API Gateway **HTTP API público** llama a la IP pública/DNS de la EC2 (no tiene IP fija). Para endurecer: usa un **VPC Link** + ALB interno y deja el origen = `sg-alb`. |
| TCP personalizado | 8090 | `0.0.0.0/0` | WebSocket directo del dashboard (`/ws/metricas`) |
| SSH 22 | — | **no abrir** | Entra por **Session Manager (SSM)** con `LabInstanceProfile` (como en la guía del profesor) |
| Salida | todo | `0.0.0.0/0` | JWKS de Entra/Cognito, SMTP, Transbank, Docker Hub |

### 1.2 `sg-rabbit`
| Puerto | Origen | Motivo |
|---|---|---|
| 5672 (AMQP) | `sg-microservicios` | productores/consumidores |
| 15672 (Management) | `sg-microservicios` + **tu IP /32** | `ms-rabbit-admin` y tú (UI) |
| 4369, 25672 | `sg-rabbit` (a sí mismo) | solo si montas **clúster** de varios nodos |

### 1.3 `sg-kafka`
| Puerto | Origen | Motivo |
|---|---|---|
| 9092 (listener externo) | `sg-microservicios` + tu IP /32 | clientes Kafka |
| 29092 | `sg-kafka` (a sí mismo) | listener interno entre contenedores (no necesita abrirse a fuera) |
| 2181 | `sg-kafka` | Zookeeper (solo interno) |
| 8080 (Kafka-UI) | **tu IP /32** | administración visual. **No** dejarlo en `0.0.0.0/0` (la guía del profesor lo abre, pero la UI no tiene login) |

### 1.4 `sg-frontend`
| Puerto | Origen |
|---|---|
| 80 (HTTP) | `0.0.0.0/0` |

> Asocia cada SG a su EC2 en: *Instancia → Seguridad → Grupos de seguridad → Cambiar*.
> Si algo "no conecta", la causa 1 es siempre un puerto o un origen faltante en el SG.

---

## 2. Preparar una EC2 (plantilla para todas)

1. **EC2 → Lanzar instancia**. Nombre (tabla 0), AMI *Amazon Linux 2023*, tipo según tabla.
2. **Par de claves**: *Continuar sin un par de claves* (se usa SSM).
3. **Red**: subnet disponible, **asignar IP pública automáticamente = Habilitar**, SG del rol.
4. **Almacenamiento**: 30 GB.
5. **Detalles avanzados → Perfil de instancia de IAM** = `LabInstanceProfile`.
6. **Datos de usuario** (instala Docker y Compose):
```bash
#!/bin/bash
dnf update -y
dnf install -y git docker
systemctl enable docker && systemctl start docker
usermod -aG docker ec2-user
mkdir -p /usr/local/lib/docker/cli-plugins/
curl -SL https://github.com/docker/compose/releases/latest/download/docker-compose-linux-x86_64 -o /usr/local/lib/docker/cli-plugins/docker-compose
chmod +x /usr/local/lib/docker/cli-plugins/docker-compose
ln -s /usr/local/lib/docker/cli-plugins/docker-compose /usr/bin/docker-compose
```
7. Lanzar → *Conectar → Sesión SSM*. Verificar: `sudo docker -v && sudo git -v`.
8. (Recomendado para Kafka y servicios que cuelga el API Gateway) **EC2 → Direcciones IP elásticas → Asignar → Asociar**.

---

## 3. RabbitMQ en AWS

1. Lanza `calisat-rabbit` (sección 2) con `sg-rabbit`.
2. En la EC2:
```bash
sudo git clone https://github.com/DavNat13/calisat-infra.git
cd calisat-infra/rabbitmq-cluster
sudo docker compose up -d rabbit1          # 1 nodo basta para la demo
# Clúster completo (3 nodos en la misma EC2): sudo docker compose up -d
```
3. Abre `http://<IP-PUB-rabbit>:15672` (usuario `calisat-app`, clave `Calisat2026!`; cámbiala con `RABBITMQ_USER/RABBITMQ_PASSWORD` si quieres).
4. En **todos** los microservicios que usan RabbitMQ (`usuarios`, `orden`, `envios`, `notificaciones`, `rabbit-admin`) define:
```
RABBITMQ_HOST=<IP-privada-rabbit>
RABBITMQ_USER=calisat-app
RABBITMQ_PASSWORD=Calisat2026!
```
   (ya están mapeadas a `SPRING_RABBITMQ_*` en sus `docker-compose.yml`; basta con exportarlas antes de `docker compose up`).
5. Las colas/exchange se **declaran solas** al arrancar `ms-notificaciones` (exchange `calisat.exchange` **topic**, colas y DLQ).

> **IMPORTANTE – broker ya existente:** el exchange `calisat.exchange` antes era *direct*. Si ese broker ya tenía datos, RabbitMQ rechaza redeclararlo como *topic* (`PRECONDITION_FAILED`). Solución: borra el exchange viejo (UI → Exchanges → `calisat.exchange` → Delete, o con `ms-rabbit-admin`) **antes** de levantar los servicios nuevos. En un broker nuevo no hay problema.

6. Verificación: UI → *Queues*: deben existir `notificaciones.queue`, `ordenes.queue`, `envios.queue`, `auditoria.queue`, `dlq.general.queue`.

---

## 4. Kafka en AWS (como en la guía del profesor)

1. Lanza `calisat-kafka` (**t3.medium**, 30 GB, `sg-kafka`, `LabInstanceProfile`) y asígnale **Elastic IP** (`<IP-PUB-kafka>`).
2. Por SSM:
```bash
sudo git clone https://github.com/DavNat13/calisat-infra.git
cd calisat-infra
export KAFKA_PUBLIC_HOST=<IP-PUB-kafka>      # lo que Kafka anuncia a clientes externos
sudo -E docker compose -f kafka/docker-compose.yml -f kafka-ui/docker-compose.yml up -d
sudo docker ps      # zookeeper, kafka, kafka-ui "Up"
```
   - `KAFKA_PUBLIC_HOST` es **crítico**: Kafka devuelve esa dirección al cliente; si es incorrecta, el cliente conecta al bootstrap pero falla al producir/consumir.
   - Si usas IP **privada** para los microservicios (misma VPC): pon `KAFKA_PUBLIC_HOST=<IP-privada>` y úsala como bootstrap.
3. Kafka-UI: `http://<IP-PUB-kafka>:8080` → debe verse el clúster `calisat`.
4. **Los tópicos no se autocrean** (`AUTO_CREATE_TOPICS=false`): los crean los microservicios al arrancar (beans `NewTopic` en `ms-pago` y `ms-analytics`) o `ms-kafka-admin`. Tras arrancar `ms-analytics` y `ms-pago`, en Kafka-UI → *Topics* verás: `interacciones-user` (3 part., retención 7 d), `payment-attempts` (3 part., 1 d), `payment-alerts` (1 part., compact+delete) y sus `.DLT`.
5. **Variable para todos los servicios que usan Kafka** (`catalogo`, `carrito`, `orden`, `pago`, `analytics`, `kafka-admin`):
```
SPRING_KAFKA_BOOTSTRAP_SERVERS=<IP-kafka>:9092
```
   - Los composes de `pago`, `analytics`, `kafka-admin` usan `KAFKA_BOOTSTRAP` (por defecto `kafka:29092` = misma red Docker). En EC2 distintas: `export KAFKA_BOOTSTRAP=<IP-kafka>:9092`.
   - `catalogo`, `carrito` y `orden` leen directamente `SPRING_KAFKA_BOOTSTRAP_SERVERS`: agrégala en su `environment:` del compose (o `export` + `-e`). Si falta, usan `localhost:9092` y **solo registran un warning** (el flujo de negocio no se bloquea).
6. Probar desde Kafka-UI: *Topics → interacciones-user → Messages* mientras ves un producto en el frontend (evento `view`, key = id de usuario).
7. Operación con `ms-kafka-admin` (rol ADMINISTRADOR): crear/ampliar/eliminar tópicos y grupos vía REST o desde `/admin/kafka` del frontend.

---

## 5. DynamoDB (historial de analítica + conexiones WebSocket)  → puntos extra +2 / +3

### 5.1 Tabla `UserInteractions`
1. Consola AWS → **DynamoDB → Tablas → Crear tabla**.
2. Nombre: `UserInteractions`. Clave de partición: `userId` (Cadena). Clave de ordenación: `sk` (Cadena).
3. Configuración: *Personalizar* → capacidad **bajo demanda** (on-demand).
4. Crear. Luego *Tabla → Exportaciones y streams → Activar **DynamoDB Streams*** → tipo **Imagen nueva (New image)**.

### 5.2 Tabla `ConexionesWebSocket`
- Nombre `ConexionesWebSocket`, clave de partición `connectionId` (Cadena), on-demand. Sin streams.

### 5.3 Que `ms-analytics` escriba en DynamoDB
1. En `calisat-analytics` define `ANALYTICS_PROFILE=dynamodb` (activa `SPRING_PROFILES_ACTIVE=dynamodb`), `AWS_REGION=us-east-1`, `DYNAMODB_TABLE=UserInteractions`.
2. Credenciales: **no escribas claves**; la EC2 usa `LabInstanceProfile` (cadena de credenciales por defecto del SDK).
3. `docker compose up -d --build` y verifica: DynamoDB → *Explorar elementos* → `UserInteractions` debe llenarse al navegar la tienda.
4. Con el perfil `dynamodb` el historial vive en DynamoDB; las recomendaciones Top-5 siguen en Postgres.

---

## 6. WebSocket de métricas (+3)

Hay **dos canales**; con uno basta para la evaluación:

### Opción A (la más simple, ya implementada): WebSocket de `ms-analytics`
- URL: `ws://<IP-PUB-analytics>:8090/ws/metricas?token=<JWT>` (solo rol ADMINISTRADOR; el JWT va por query porque el navegador no puede mandar cabeceras).
- Frontend: al construir define `VITE_METRICAS_WS_URL=ws://<IP-PUB-analytics>:8090/ws/metricas`.
- Mixed content: si el frontend se sirve por **HTTPS**, el WS debe ser **`wss://`** → publica `ms-analytics` detrás de un ALB/CloudFront con certificado, o sirve el frontend por HTTP en la demo. El CSP de `nginx.conf` ya permite `wss://*.execute-api…` y `ws://localhost:8090`; **si usas otra IP/host añádelo a `connect-src`** y reconstruye la imagen.

### Opción B (serverless, guía del profesor): API Gateway WebSocket + Lambdas + DynamoDB Streams
1. Compila `calisat-aws-lambdas` (`./mvnw clean package`) → `target/calisat-aws-lambdas-1.0.0.jar`.
2. **Lambda** → Crear función → Java 21, rol `LabRole`. Crea **tres** funciones subiendo el mismo JAR:
   | Nombre | Handler |
   |---|---|
   | `ws-connect` | `com.califorge.lambdas.WebSocketConnectHandler::handleRequest` |
   | `ws-disconnect` | `com.califorge.lambdas.WebSocketDisconnectHandler::handleRequest` |
   | `interaccion-stream` | `com.califorge.lambdas.InteraccionStreamHandler::handleRequest` |
   Memoria 512 MB, timeout 30 s.
3. **API Gateway → Crear API → API WebSocket**: nombre `calisat_ws`, expresión de ruta `request.body.action`. Rutas `$connect` → `ws-connect`, `$disconnect` → `ws-disconnect`. Crear etapa `prod` y copiar **URL WebSocket** (`wss://…`) y **URL de conexión** (`https://…/prod`).
4. En `interaccion-stream` → Configuración → Variables de entorno: `WEBSOCKET_API_ENDPOINT=https://<api-id>.execute-api.us-east-1.amazonaws.com/prod`, `CONNECTIONS_TABLE=ConexionesWebSocket`.
5. `interaccion-stream` → *Agregar desencadenador* → **DynamoDB** → tabla `UserInteractions`, tamaño de lote 10, posición inicial *Latest*.
6. Usa la URL `wss://…/prod` como `VITE_METRICAS_WS_URL` (el payload serverless es el agregado por lote; el de la Opción A es el snapshot completo con las 4 métricas).

---

## 7. Desplegar los microservicios (cada uno igual)

En cada EC2 (por SSM):
```bash
sudo git clone https://github.com/DavNat13/calisat-ms-<nombre>.git
cd calisat-ms-<nombre>
# variables específicas (ver tabla 7.1) y luego:
sudo -E docker compose up -d --build
sudo docker logs -f calisat-ms-<nombre>
curl http://localhost:<puerto>/actuator/health      # {"status":"UP"}
```
> Los secretos van **en el compose / variables exportadas** (decisión del proyecto: son de asignatura). No uses `.env` dentro de la EC2 si te dio problemas: usa `export VAR=valor` y `sudo -E`, o escribe el valor directamente en `environment:` del compose.

### 7.1 Variables por servicio
| Servicio | Variables a definir |
|---|---|
| usuarios | `RABBITMQ_HOST` (+ `RABBITMQ_USER/PASSWORD`) |
| catalogo | `SPRING_KAFKA_BOOTSTRAP_SERVERS` |
| inventario | — |
| carrito | `SPRING_KAFKA_BOOTSTRAP_SERVERS`, `CALISAT_URLS_CATALOGO`/`CALISAT_URLS_INVENTARIO` si no usas el gateway |
| orden | `RABBITMQ_HOST…`, `SPRING_KAFKA_BOOTSTRAP_SERVERS` |
| envios | `RABBITMQ_HOST…` |
| notificaciones | `RABBITMQ_HOST…` (SMTP ya configurado en `application.yaml`) |
| rabbit-admin | `RABBITMQ_HOST…`, `RABBITMQ_MANAGEMENT_URL=http://<IP-rabbit>:15672` |
| pago | `KAFKA_BOOTSTRAP`, `WEBPAY_ENABLED=true`, `WEBPAY_RETURN_URL=https://<dominio-front>/checkout/retorno` |
| analytics | `KAFKA_BOOTSTRAP`, `ANALYTICS_PROFILE` (`dynamodb` opcional), `WS_ALLOWED_ORIGINS=http://<IP-front>` |
| kafka-admin | `KAFKA_BOOTSTRAP` |

Orden recomendado de arranque: **RabbitMQ → Kafka → notificaciones → usuarios/catalogo/inventario → carrito → envios → orden → pago → analytics → admin(s) → frontend**.

---

## 8. API Gateway (HTTP API) — qué cambiar

El gateway actual es `ho5p58iyu7` (HTTP API) con el authorizer `calisat-validador-lambda`. Hay que **agregar rutas e integraciones** para los servicios nuevos y verificar las existentes.

### 8.1 Integraciones (una por servicio)
API Gateway → tu API → **Integraciones → Crear** → tipo *URL HTTP* (HTTP proxy), método `ANY`, URL:
`http://<IP-PUB-servicio>:<puerto>/{proxy}`  (ej. `http://3.90.1.20:8089/{proxy}`).
Crea: `int-usuarios` (8081), `int-catalogo` (8082), `int-inventario` (8083), `int-carrito` (8084), `int-orden` (8085), `int-envios` (8086), `int-notificaciones` (8087), **`int-rabbit-admin` (8088)**, **`int-pago` (8089)**, **`int-analytics` (8090)**, **`int-kafka-admin` (8091)**.

### 8.2 Rutas
Crea las rutas del archivo **`ENDPOINTS_API_GATEWAY.md`** (sección "Rutas consolidadas") y asocia cada una a su integración y al authorizer:
- Rutas **protegidas** → *Autorización → Adjuntar* `calisat-validador-lambda` (REQUEST, payload 2.0, respuesta simple, caché 300 s).
- Rutas **públicas** (sin authorizer): `GET /api/v1/catalogo`, `GET /api/v1/catalogo/{sku}`, `GET /api/v1/catalogo/categoria/{categoria}`, `GET /api/v1/envios/seguimiento/{numeroGuia}`, `GET /actuator/health` (opcional).
- Lo **nuevo** a crear: `/api/v1/pagos/*`, `/api/v1/recomendaciones/*`, `/api/v1/analytics/*`, `/api/v1/rabbit/*`, `/api/v1/kafka/*`.

### 8.3 CORS (API → CORS)
- Orígenes permitidos: `https://ezeh839whh.execute-api.us-east-1.amazonaws.com` (origen actual del frontend) **y el dominio/IP donde publiques el nuevo frontend**.
- Métodos: `GET, POST, PUT, PATCH, DELETE, OPTIONS`. Cabeceras: `Authorization, Content-Type, Idempotency-Key`. Credenciales: sí.
- Aviso: además del gateway, cada microservicio responde CORS con su propia lista (`SecurityConfig`): si cambias el origen del frontend, actualiza `ALLOWED_ORIGIN` en cada servicio.

### 8.4 Authorizer: ¿hay que tocarlo?
- **No** hay cambios de código. Revisa que la Lambda tenga configurados los **dos proveedores** (Entra y Cognito) con sus `aud`/`client_id` (variables `APP_SECURITY_JWT_PROVIDERS_*`, ver README de la lambda).
- Si cambias la caché (300 s), recuerda que un rol cambiado tarda hasta 5 min en reflejarse.

### 8.5 Etapa y despliegue
- Usa la etapa `$default` con *auto-deploy* activado. Prueba:
```bash
curl -i https://<api-id>.execute-api.us-east-1.amazonaws.com/api/v1/catalogo        # 200 sin token
curl -i https://<api-id>.execute-api.us-east-1.amazonaws.com/api/v1/pagos/mis-pagos # 401 sin token
```

---

## 9. Frontend

1. Si cambió la URL del gateway: edita `src/config/api.js` (`URL_BASE`) y el `connect-src` de `nginx.conf`.
2. Para `/checkout` con WebPay, el CSP ya incluye `form-action https://webpay3gint.transbank.cl https://webpay3g.transbank.cl`.
3. Build y despliegue (EC2 con Docker):
```bash
sudo git clone https://github.com/DavNat13/calisat-frontend.git && cd calisat-frontend
sudo docker build --build-arg VITE_METRICAS_WS_URL=ws://<IP-PUB-analytics>:8090/ws/metricas -t calisat-frontend .   # si el Dockerfile no declara el ARG, edita WS_METRICAS_URL en src/config/api.js
sudo docker run -d -p 80:80 --name calisat-frontend calisat-frontend
```
4. **Redirect URIs**: en **Entra ID** (registro de app → Autenticación) y **Cognito** (App client → URLs de retorno) agrega la URL final del frontend.
5. `WEBPAY_RETURN_URL` (ms-pago) debe apuntar a `https://<frontend>/checkout/retorno` y esa URL debe existir como origen válido.

---

## 10. Lista de verificación (en orden)

| # | Prueba | Resultado esperado |
|---|---|---|
| 1 | RabbitMQ UI | 5 colas creadas, `calisat.exchange` tipo topic |
| 2 | Kafka-UI | tópicos `interacciones-user`, `payment-attempts`, `payment-alerts` (+DLT) con 3/3/1 particiones |
| 3 | `curl /actuator/health` en cada EC2 | `UP` |
| 4 | Registrar usuario nuevo | mensaje en `notificaciones.queue` → correo recibido |
| 5 | Detener `notificaciones`, crear orden | `201` y mensaje encolado; al levantar se consume |
| 6 | Mensaje inválido a una cola | termina en `dlq.general.queue` y aparece `[DLQ]` en logs de notificaciones |
| 7 | Ver producto / agregar al carrito / pagar | eventos en `interacciones-user` (Kafka-UI), key = userId |
| 8 | `GET /api/v1/recomendaciones/me` | Top-5 del usuario |
| 9 | Pagar con la misma tarjeta 4 veces en <10 s | 4.º intento: **429**, alerta en `payment-alerts` y en `/admin/metricas` |
| 10 | `/admin/metricas` | WebSocket conectado, 4 métricas moviéndose |
| 11 | DynamoDB `UserInteractions` | filas nuevas (si perfil `dynamodb`) |
| 12 | `/admin/rabbitmq` y `/admin/kafka` | crear y eliminar una cola / un tópico de prueba |
| 13 | Orden con SKU sin stock | `409` y la orden no se crea |

---

## 11. Problemas frecuentes

| Síntoma | Causa | Solución |
|---|---|---|
| Servicio arranca pero Kafka "no produce" | `KAFKA_PUBLIC_HOST` incorrecto | recrear Kafka con la IP correcta; `docker compose down && up -d` |
| `PRECONDITION_FAILED inequivalent arg 'type'` | exchange viejo *direct* | borrar `calisat.exchange` y reiniciar servicios |
| `ACCESS_REFUSED` en Rabbit | usuario/clave distintos | alinear `RABBITMQ_USER/PASSWORD` en todos |
| 403 en rutas admin | token sin rol `ADMINISTRADOR` | revisar roles de aplicación en Entra / grupos en Cognito |
| 401 en todo | authorizer sin `aud` correcto | ver variables de la Lambda |
| WebSocket se cierra al conectar | token no-admin o `WS_ALLOWED_ORIGINS` sin tu origen | revisar ambos |
| Pagos 502 | Transbank inalcanzable (sin salida a internet) o `WEBPAY_ENABLED=true` sin red | usar `WEBPAY_ENABLED=false` (simulado) |
| Los datos desaparecen al reiniciar el Lab | EC2 sin volumen persistente / IP cambió | usar volúmenes Docker (ya definidos) + Elastic IP |
