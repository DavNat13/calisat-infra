# calisat-infra
Composes de infraestructura (EP3 + EP5). Red compartida `calisat-net`.

| Stack | Archivo | Puertos |
|---|---|---|
| Cluster RabbitMQ (3 nodos) | `rabbitmq-cluster/docker-compose.yml` | AMQP 5672/5673/5674, UI 15672 |
| Kafka + Zookeeper | `kafka/docker-compose.yml` | Kafka 9092 (host) / 29092 (interno), ZK 2181 |
| Kafka-UI | `kafka-ui/docker-compose.yml` | 8080 |

Levantar todo:
```
docker compose -f rabbitmq-cluster/docker-compose.yml -f kafka/docker-compose.yml -f kafka-ui/docker-compose.yml up -d
```
En EC2: `export KAFKA_PUBLIC_HOST=<ip publica>` (abrir 8080, 9092, 2181, 5672, 15672 en el Security Group).
Usuario RabbitMQ por defecto: `calisat-app` / `Calisat2026!` (override con `RABBITMQ_USER`/`RABBITMQ_PASSWORD`).
Los tópicos Kafka no se autocrean; los crea cada microservicio al arrancar (`NewTopic`) o `calisat-ms-kafka-admin`.
