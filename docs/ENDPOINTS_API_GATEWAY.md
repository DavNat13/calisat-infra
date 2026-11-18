# Endpoints de todos los microservicios (para las rutas del API Gateway)

Convención del gateway (HTTP API, payload 2.0): cada ruta se crea como `MÉTODO /ruta` y apunta a la integración del servicio (`http://<IP-servicio>:<puerto>/{proxy}`).
Columna **Auth**: 🔓 pública (sin authorizer) · 🔐 JWT (authorizer + validación interna) · 🤖 servicio→servicio (`X-Service-Token`, no se publica en el gateway).
Roles: `CLIENTE`, `ADMINISTRADOR`, `LOGISTICA`, `SERVICIO`.

---

## A. Rutas consolidadas (la forma más rápida: 1 ruta por prefijo)

Crea estas rutas con `ANY` + `{proxy+}`; la autorización fina la hace cada microservicio.

| Ruta en API Gateway | Integración (puerto) | Authorizer |
|---|---|---|
| `ANY /api/v1/usuarios/{proxy+}` | usuarios (8081) | 🔐 |
| `GET /api/v1/catalogo` · `GET /api/v1/catalogo/{sku}` · `GET /api/v1/catalogo/categoria/{categoria}` | catalogo (8082) | 🔓 **sin** authorizer |
| `ANY /api/v1/catalogo/{proxy+}` | catalogo (8082) | 🔐 (el resto del catálogo: POST/PUT/DELETE/inactivos/reactivar) |
| `ANY /api/v1/stock/{proxy+}` | inventario (8083) | 🔐 |
| `ANY /api/v1/carrito/{proxy+}` y `ANY /api/v1/carrito` | carrito (8084) | 🔐 |
| `ANY /api/v1/ordenes/{proxy+}` y `ANY /api/v1/ordenes` | orden (8085) | 🔐 |
| `GET /api/v1/envios/seguimiento/{numeroGuia}` | envios (8086) | 🔓 **sin** authorizer |
| `ANY /api/v1/envios/{proxy+}` y `ANY /api/v1/envios` | envios (8086) | 🔐 |
| `ANY /api/v1/notificaciones/{proxy+}` y `ANY /api/v1/notificaciones` | notificaciones (8087) | 🔐 |
| `ANY /api/v1/auditoria` · `ANY /api/v1/auditoria/{proxy+}` | notificaciones (8087) | 🔐 |
| `ANY /api/v1/destinatarios` · `ANY /api/v1/destinatarios/{proxy+}` | notificaciones (8087) | 🔐 |
| `ANY /api/v1/plantillas` · `ANY /api/v1/plantillas/{proxy+}` | notificaciones (8087) | 🔐 |
| `ANY /api/v1/preferencias` · `ANY /api/v1/preferencias/{proxy+}` | notificaciones (8087) | 🔐 |
| **`ANY /api/v1/rabbit/{proxy+}`** | **rabbit-admin (8088)** | 🔐 (ADMINISTRADOR) |
| **`ANY /api/v1/pagos/{proxy+}`** | **pago (8089)** | 🔐 |
| **`ANY /api/v1/recomendaciones/{proxy+}`** | **analytics (8090)** | 🔐 |
| **`ANY /api/v1/analytics/{proxy+}`** | **analytics (8090)** | 🔐 (ADMINISTRADOR) |
| **`ANY /api/v1/kafka/{proxy+}`** | **kafka-admin (8091)** | 🔐 (ADMINISTRADOR) |
| `OPTIONS /{proxy+}` | (CORS lo resuelve el gateway) | 🔓 |

> Las rutas **más específicas ganan** en HTTP API (`GET /api/v1/catalogo/{sku}` sin authorizer prevalece sobre `ANY /api/v1/catalogo/{proxy+}` con authorizer).
> Nuevo respecto de EV2 (en negrita): rabbit, pagos, recomendaciones, analytics, kafka.
> **WebSocket** de métricas: no pasa por el HTTP API → `ws(s)://<host-analytics>:8090/ws/metricas?token=<JWT>` (o API WebSocket de la guía AWS).

---

## B. Detalle por microservicio

### 1. calisat-ms-usuarios — puerto 8081 (`/api/v1/usuarios`)
| Método | Ruta | Auth / rol | Descripción |
|---|---|---|---|
| POST | `/api/v1/usuarios/registro` | 🔐 cualquier autenticado | Alta/sync del usuario desde el JWT. 201 nuevo / 200 existente. Publica `usuario.registrado` (RabbitMQ) |
| GET | `/api/v1/usuarios/perfil` | 🔐 CLIENTE | Perfil |
| PUT | `/api/v1/usuarios/perfil` | 🔐 CLIENTE | Actualizar perfil |
| DELETE | `/api/v1/usuarios/perfil` | 🔐 CLIENTE | Baja |
| GET | `/api/v1/usuarios/perfil/direcciones` | 🔐 CLIENTE | Listar direcciones |
| POST | `/api/v1/usuarios/perfil/direcciones` | 🔐 CLIENTE | Crear dirección |
| PUT | `/api/v1/usuarios/perfil/direcciones/{id}` | 🔐 CLIENTE | Editar dirección |
| DELETE | `/api/v1/usuarios/perfil/direcciones/{id}` | 🔐 CLIENTE | Eliminar dirección |
| PUT | `/api/v1/usuarios/perfil/direcciones/{id}/predeterminada` | 🔐 CLIENTE | Marcar predeterminada |

### 2. calisat-ms-catalogo — puerto 8082 (`/api/v1/catalogo`)
| Método | Ruta | Auth / rol | Descripción |
|---|---|---|---|
| GET | `/api/v1/catalogo` | 🔓 | Listar activos (paginado). |
| GET | `/api/v1/catalogo/{sku}` | 🔓 | Detalle por SKU. **Publica evento `view` en Kafka** |
| GET | `/api/v1/catalogo/categoria/{categoria}` | 🔓 | Filtrar por categoría |
| GET | `/api/v1/catalogo/inactivos` | 🔐 ADMINISTRADOR | Productos dados de baja |
| POST | `/api/v1/catalogo` | 🔐 ADMINISTRADOR | Crear |
| PUT | `/api/v1/catalogo/{sku}` | 🔐 ADMINISTRADOR | Editar |
| DELETE | `/api/v1/catalogo/{sku}` | 🔐 ADMINISTRADOR | Baja lógica |
| POST | `/api/v1/catalogo/{sku}/reactivar` | 🔐 ADMINISTRADOR | Reactivar producto |

### 3. calisat-ms-inventario — puerto 8083 (`/api/v1/stock`)
| Método | Ruta | Auth / rol | Descripción |
|---|---|---|---|
| GET | `/api/v1/stock` | 🔐 autenticado | Listar stock |
| GET | `/api/v1/stock/sku/{sku}` | 🔐 autenticado | Stock por SKU |
| GET | `/api/v1/stock/{id}` | 🔐 autenticado | Por id |
| POST | `/api/v1/stock` | 🔐 ADMINISTRADOR / 🤖 SERVICIO | Crear registro de stock |
| PUT | `/api/v1/stock/{id}` | 🔐 ADMINISTRADOR | Actualizar |
| DELETE | `/api/v1/stock/{id}` | 🔐 ADMINISTRADOR | Eliminar |
| POST | `/api/v1/stock/{sku}/reservar` | 🔐 ADMIN / 🤖 SERVICIO | Reservar (409 sin stock) |
| POST | `/api/v1/stock/{sku}/liberar` | 🔐 ADMIN / 🤖 SERVICIO | Liberar reserva |
| POST | `/api/v1/stock/{sku}/confirmar` | 🔐 ADMIN / 🤖 SERVICIO | Confirmar descuento |

### 4. calisat-ms-carrito — puerto 8084 (`/api/v1/carrito`) — todos 🔐 CLIENTE|SERVICIO|ADMINISTRADOR
| Método | Ruta | Descripción |
|---|---|---|
| GET | `/api/v1/carrito` | Ver carrito |
| POST | `/api/v1/carrito/items` | Agregar ítem. **Publica evento `cart` en Kafka** |
| PUT | `/api/v1/carrito/items/{sku}` | Cambiar cantidad |
| DELETE | `/api/v1/carrito/items/{sku}` | Quitar ítem |
| DELETE | `/api/v1/carrito` | Vaciar |
| POST | `/api/v1/carrito/validar` | Validar carrito (precio/stock) |

### 5. calisat-ms-orden — puerto 8085 (`/api/v1/ordenes`) — 🔐 CLIENTE|ADMINISTRADOR
| Método | Ruta | Descripción |
|---|---|---|
| POST | `/api/v1/ordenes` | Crear orden (cabecera `Idempotency-Key`). **409 si no hay stock** |
| GET | `/api/v1/ordenes` | Mis órdenes (paginado) |
| GET | `/api/v1/ordenes/todas` | Todas (ADMINISTRADOR) |
| GET | `/api/v1/ordenes/{id}` | Detalle |
| PUT | `/api/v1/ordenes/{id}/estado` | Cambiar estado (`PAGADA` publica `orden.confirmada` y evento `purchase` en Kafka) |
| POST | `/api/v1/ordenes/{id}/cancelar` | Cancelar |

### 6. calisat-ms-envios — puerto 8086 (`/api/v1/envios`)
| Método | Ruta | Auth / rol | Descripción |
|---|---|---|---|
| POST | `/api/v1/envios/cotizar` | 🔐 autenticado | Cotizar envío |
| POST | `/api/v1/envios` | 🔐 ADMIN/LOGISTICA / 🤖 SERVICIO | Crear envío y guía `CAL-…` |
| GET | `/api/v1/envios` | 🔐 autenticado | Listar |
| GET | `/api/v1/envios/{id}` | 🔐 autenticado | Detalle |
| PUT | `/api/v1/envios/{id}/estado` | 🔐 ADMIN/LOGISTICA | Cambiar estado (publica `envio.*`) |
| GET | `/api/v1/envios/seguimiento/{numeroGuia}` | 🔓 pública | Seguimiento por guía |

### 7. calisat-ms-notificaciones — puerto 8087
| Método | Ruta | Auth / rol | Descripción |
|---|---|---|---|
| POST | `/api/v1/notificaciones/eventos` | 🤖 `X-Service-Token` (ROLE_SERVICIO) | Ingesta de evento (no se publica) |
| POST | `/api/v1/notificaciones/enviar` | 🔐 ADMINISTRADOR | Envío manual |
| GET | `/api/v1/notificaciones` | 🔐 ADMINISTRADOR | Listado |
| GET | `/api/v1/notificaciones/{id}` | 🔐 ADMINISTRADOR | Detalle |
| POST | `/api/v1/notificaciones/{id}/reintentar` | 🔐 ADMINISTRADOR | Reintentar |
| GET | `/api/v1/notificaciones/mis-notificaciones` | 🔐 autenticado | Las del usuario |
| GET | `/api/v1/auditoria` | 🔐 ADMINISTRADOR | Auditoría (paginada, filtros `tipo`,`resultado`) |
| GET/POST/PUT/DELETE | `/api/v1/plantillas[/{codigo}]` | 🔐 GET autenticado · escritura ADMINISTRADOR | CRUD plantillas |
| GET/POST | `/api/v1/destinatarios` | 🔐 ADMINISTRADOR | Directorio de destinatarios |
| GET/PUT | `/api/v1/preferencias` | 🔐 autenticado | Preferencias del usuario |

### 8. **calisat-ms-rabbit-admin — puerto 8088 (`/api/v1/rabbit`) — 🔐 ADMINISTRADOR** *(nuevo)*
| Método | Ruta | Descripción |
|---|---|---|
| POST | `/api/v1/rabbit/queues` | Crear cola `{name, durable, deadLetterExchange, deadLetterRoutingKey, messageTtlMs, quorum}` |
| GET | `/api/v1/rabbit/queues` | Listar colas (vía Management API) |
| GET | `/api/v1/rabbit/queues/{name}` | Propiedades de una cola |
| DELETE | `/api/v1/rabbit/queues/{name}` | Eliminar cola |
| POST | `/api/v1/rabbit/exchanges` | Crear exchange `{name, type: topic\|direct\|fanout\|headers, durable}` |
| GET | `/api/v1/rabbit/exchanges` | Listar exchanges |
| DELETE | `/api/v1/rabbit/exchanges/{name}` | Eliminar exchange |
| POST | `/api/v1/rabbit/bindings` | Crear binding `{exchange, queue, routingKey}` |
| GET | `/api/v1/rabbit/bindings` | Listar bindings |
| DELETE | `/api/v1/rabbit/bindings` | Eliminar binding (cuerpo JSON con exchange/queue/routingKey) |

### 9. **calisat-ms-pago — puerto 8089 (`/api/v1/pagos`)** *(nuevo)*
| Método | Ruta | Auth / rol | Descripción |
|---|---|---|---|
| POST | `/api/v1/pagos/iniciar` | 🔐 CLIENTE/ADMIN | `{ordenId, monto, tarjeta}` → 201 `{pagoId, token, url, simulado}`; **429** si el antifraude bloquea la tarjeta |
| POST | `/api/v1/pagos/confirmar` | 🔐 CLIENTE/ADMIN | `{token}` → estado `PAGADO`/`RECHAZADO` |
| GET | `/api/v1/pagos/mis-pagos` | 🔐 CLIENTE/ADMIN | Historial propio |
| GET | `/api/v1/pagos/alertas` | 🔐 ADMINISTRADOR | Últimas 50 alertas de fraude |

### 10. **calisat-ms-analytics — puerto 8090** *(nuevo)*
| Método | Ruta | Auth / rol | Descripción |
|---|---|---|---|
| GET | `/api/v1/recomendaciones/me` | 🔐 CLIENTE/ADMIN | Top-5 del usuario autenticado |
| GET | `/api/v1/recomendaciones/{userId}` | 🔐 propio o ADMINISTRADOR | Top-5 de un usuario |
| GET | `/api/v1/analytics/metricas` | 🔐 ADMINISTRADOR | Snapshot REST de métricas |
| WS | `/ws/metricas?token=<JWT>` | JWT ADMINISTRADOR (handshake) | Métricas en vivo (4) — **no va por el HTTP API** |

### 11. **calisat-ms-kafka-admin — puerto 8091 (`/api/v1/kafka`) — 🔐 ADMINISTRADOR** *(nuevo)*
| Método | Ruta | Descripción |
|---|---|---|
| POST | `/api/v1/kafka/topics` | Crear tópico `{name, partitions, replicationFactor, configs{retention.ms, cleanup.policy…}}` |
| GET | `/api/v1/kafka/topics` | Listar tópicos |
| GET | `/api/v1/kafka/topics/{name}` | Describir (particiones, líder, ISR, configs) |
| PUT | `/api/v1/kafka/topics/{name}` | Ampliar particiones y/o cambiar configs |
| DELETE | `/api/v1/kafka/topics/{name}` | Eliminar tópico |
| GET | `/api/v1/kafka/groups` | Listar consumer groups |
| GET | `/api/v1/kafka/groups/{groupId}` | Describir grupo y lag por partición |
| POST | `/api/v1/kafka/groups/{groupId}/reset-offsets` | `{topic, to: earliest\|latest}` (grupo sin miembros activos) |
| DELETE | `/api/v1/kafka/groups/{groupId}` | Eliminar grupo |

### 12. Comunes a TODOS los servicios (no publicar o solo para pruebas)
| Método | Ruta | Nota |
|---|---|---|
| GET | `/actuator/health` | Health check (público). Úsalo solo para pruebas; no es necesario en el gateway |
| GET | `/v3/api-docs`, `/swagger-ui.html` | OpenAPI/Swagger (públicos en cada MS; **no publicarlos** en el gateway) |

---

## C. Checklist rápido en la consola de API Gateway
1. Integraciones: 11 (usuarios, catalogo, inventario, carrito, orden, envios, notificaciones, **rabbit-admin, pago, analytics, kafka-admin**).
2. Rutas públicas primero (catálogo GET ×3, seguimiento GET) **sin** authorizer.
3. Resto de rutas con authorizer `calisat-validador-lambda`.
4. CORS: orígenes del frontend; métodos `GET,POST,PUT,PATCH,DELETE,OPTIONS`; cabeceras `Authorization,Content-Type,Idempotency-Key`.
5. Probar con Postman/curl cada prefijo nuevo: sin token → 401; token de CLIENTE en `/api/v1/kafka/topics` → 403; token ADMIN → 200.
