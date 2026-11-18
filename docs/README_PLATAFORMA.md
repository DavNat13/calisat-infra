# Calisat — Plataforma e-commerce Cloud Native (EV1 + EV2 + EV3 / EP3 + EP5)

Frontend React 19 + microservicios Spring Boot 4.1 (Java 21) + Lambda authorizer + RabbitMQ + Apache Kafka.

## Repositorios y puertos

| Carpeta / repo | Puerto | Rol |
|---|---|---|
| `calisat-frontend` | 5173 / 80 | SPA React (MSAL Entra ID + Cognito), checkout con WebPay, dashboard y paneles admin |
| `calisat-ms-usuarios` | 8081 | Perfil/direcciones; publica `usuario.registrado` (RabbitMQ) |
| `calisat-ms-catalogo` | 8082 | Catálogo; publica evento `view` (Kafka) |
| `calisat-ms-inventario` | 8083 | Stock (reservar/liberar/confirmar) |
| `calisat-ms-carrito` | 8084 | Carrito; publica evento `cart` (Kafka) |
| `calisat-ms-orden` | 8085 | Órdenes; RabbitMQ + evento `purchase` (Kafka); aborta (409) si no hay stock |
| `calisat-ms-envios` | 8086 | Envíos/guías; publica `envio.*` (RabbitMQ) |
| `calisat-ms-notificaciones` | 8087 | Consume RabbitMQ (correo SMTP), DLQ con logs, auditoría |
| **`calisat-ms-rabbit-admin`** | 8088 | Admin RabbitMQ: crear/listar/eliminar colas, exchanges y bindings |
| **`calisat-ms-pago`** | 8089 | WebPay (Transbank integración) + antifraude (Velocity Check) con Kafka |
| **`calisat-ms-analytics`** | 8090 | Consume `interacciones-user`: Top-5 recomendaciones, métricas, WebSocket, DynamoDB opcional |
| **`calisat-ms-kafka-admin`** | 8091 | Admin Kafka (`AdminClient`): tópicos y consumer groups |
| **`calisat-infra`** | — | Composes: clúster RabbitMQ (3 nodos), Kafka + Zookeeper, Kafka-UI |
| **`calisat-aws-lambdas`** | — | Lambdas Java 21 (WebSocket connect/disconnect, broadcast desde DynamoDB Streams) |
| `calisat-validador-lambda` | — | Authorizer JWT dual (Entra + Cognito) |

(en **negrita**: nuevos de la EV3)

## Arranque local
```
cd calisat-infra
docker compose -f rabbitmq-cluster/docker-compose.yml -f kafka/docker-compose.yml -f kafka-ui/docker-compose.yml up -d
```
Luego en cada microservicio: `docker compose up -d --build` (todos comparten la red `calisat-net`).
UIs: RabbitMQ http://localhost:15672 · Kafka-UI http://localhost:8080.
Variables útiles: `RABBITMQ_HOST` (def. `calisat-rabbit1`), `KAFKA_BOOTSTRAP` (def. `kafka:29092`), `WEBPAY_ENABLED` (`false` = pago simulado).

## Mensajería: convivencia RabbitMQ + Kafka
- **RabbitMQ** (transaccional, consumo destructivo): exchange `calisat.exchange` (**topic**), colas `notificaciones.queue`, `ordenes.queue`, `envios.queue`, `auditoria.queue`; fallidos → `calisat.dlx` → `dlq.general.queue`, consumida por `DlqListener` (log de causa `x-death`). Reintentos 3x con backoff.
- **Kafka** (streaming, retención/offsets):

| Tópico | Key | Particiones | Retención | Cleanup | Productor → Consumidor (group) |
|---|---|---|---|---|---|
| `interacciones-user` | userId | 3 | 7 d | delete | catalogo/carrito/orden → analytics (`analytics-recommendations-group`, `analytics-metrics-group`) |
| `payment-attempts` | hash tarjeta | 3 | 1 d | delete | pago → pago (`payment-audit-group`) |
| `payment-alerts` | hash tarjeta | 1 | 7 d | compact,delete | pago → pago (`payment-alerts-group`), analytics (`analytics-metrics-group`) |
| `*.DLT` | — | = origen | 7 d | delete | registros que agotan reintentos o son corruptos |

Errores: `ErrorHandlingDeserializer` + `DefaultErrorHandler` (3 reintentos, backoff exponencial) + `DeadLetterPublishingRecoverer`; payloads inválidos no se reintentan.

## Rutas a publicar en el API Gateway (HTTP API, con el authorizer)
| Ruta | Destino |
|---|---|
| `ANY /api/v1/pagos/{proxy+}` | ms-pago :8089 |
| `ANY /api/v1/recomendaciones/{proxy+}`, `/api/v1/analytics/{proxy+}` | ms-analytics :8090 |
| `ANY /api/v1/rabbit/{proxy+}` | ms-rabbit-admin :8088 |
| `ANY /api/v1/kafka/{proxy+}` | ms-kafka-admin :8091 |
| WebSocket `/ws/metricas?token=<JWT>` | ms-analytics :8090 (directo, `VITE_METRICAS_WS_URL`) |

Seguridad de los admin/analytics: JWT dual (Entra + Cognito), rol `ADMINISTRADOR`. Variables de entorno de URL del antifraude en `calisat-ms-pago/src/main/resources/application.yaml`.

## Mapa pauta EP5 → código
| Indicador | Dónde |
|---|---|
| Config cliente Kafka (12 %) | `KafkaConfig` de ms-pago / ms-analytics; `KafkaProducerConfig` en catalogo/carrito/orden |
| 2 tópicos con DTO (12 %) | `InteraccionEvent`, `IntentoPagoEvent`, `AlertaFraudeEvent` |
| Admin básico (6 %) / avanzado (6 %) | `calisat-ms-kafka-admin` (`TopicAdminService`, `ConsumerGroupAdminService`) |
| Productores (14 %) | `PagoService`, `InteraccionPublisher` (key = userId / hash tarjeta) |
| Consumidores (14 %) | `PagoKafkaListeners`, `AnalyticsKafkaListeners` → servicios de dominio |
| Errores (12 %) | `KafkaConfig.kafkaErrorHandler` + DLT, validación en servicios |
| Parámetros de tópicos (12 %) | `KafkaTopicsConfig` (particiones, réplicas, `retention.ms`, `cleanup.policy`) |
| Consumer groups (12 %) | 2 groups sobre `interacciones-user`; `concurrency=3` |

## Pruebas
`./mvnw clean test` en cada servicio (los de Kafka incluyen un test de integración con broker embebido: serialización, grupos y DLT) y `npm run lint && npm run build` en el frontend.
