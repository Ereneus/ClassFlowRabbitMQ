# ClassFlow — Cluster RabbitMQ

Cluster RabbitMQ de **2 nodos** de ClassFlow: la base de mensajería de la EP2, levantada con Docker Compose. Lleva los comandos punto a punto entre microservicios (mensaje creado → notificación), con DLQ.

Los productores y consumidores están en el repositorio del backend ([ClassFlowBack](https://github.com/Ereneus/ClassFlowBack)). El streaming de eventos va por la plataforma Kafka ([ClassFlowKafka](https://github.com/Ereneus/ClassFlowKafka)), que complementa a RabbitMQ sin modificarlo.

## Arquitectura

| Componente | Imagen | Rol |
|---|---|---|
| `rabbitmq-1` | `rabbitmq:3.13-management-alpine` | Primer nodo del cluster. Panel en `http://<host>:15672`. |
| `rabbitmq-2` | `rabbitmq:3.13-management-alpine` | Se une al cluster cuando `rabbitmq-1` está sano. Panel en `http://<host>:15673`. |

- Los nodos forman el cluster con *peer discovery* `classic_config` ([rabbitmq/rabbitmq.conf](rabbitmq/rabbitmq.conf)) y comparten la Erlang cookie (`RABBITMQ_ERLANG_COOKIE`).
- `cluster_partition_handling = autoheal`: con 2 nodos ningún lado tiene mayoría tras una partición de red. `autoheal` deja seguir a ambos y, al reconectarse, reinicia el lado perdedor para volver a un solo estado.
- Los microservicios conocen los 2 nodos (`amqp://...@rabbitmq-1:5672,amqp://...@rabbitmq-2:5672`), se conectan a uno al azar y se pasan al otro si cae.

**Topología** (la declaran `ms-message` y `ms-notification` al arrancar):

| Recurso | Tipo | Uso |
|---|---|---|
| `classflow.events` | exchange `topic` | Eventos de mensajería |
| `classflow.messages.created` | cola (routing key `message.created`) | Mensajes nuevos que `ms-notification` convierte en alertas |
| `classflow.dlx` | exchange `topic` | Dead letter exchange |
| `classflow.messages.dlq` | cola | Mensajes rechazados, para revisarlos en el panel |

Son colas clásicas: cada una vive en el nodo donde se declaró y el otro nodo la expone a los clientes.

## Puesta en marcha

Requisitos: Docker con Docker Compose v2.

```bash
cp .env.example .env                              # y completar las claves
docker network create classflow-back-net          # una vez por host (compartida con backend y Kafka)
docker compose up -d
docker compose exec rabbitmq-1 rabbitmq-diagnostics cluster_status   # deben aparecer los 2 nodos
```

Variables de `.env`:

| Variable | Descripción |
|---|---|
| `RABBITMQ_USER` / `RABBITMQ_PASSWORD` | Usuario del broker y del panel. El backend debe usar los mismos valores. Clave: `openssl rand -hex 16` (solo letras y números, va dentro de la URL AMQP). |
| `RABBITMQ_ERLANG_COOKIE` | Secreto compartido por los nodos para formar el cluster. `openssl rand -hex 32`. |

**En AWS EC2:** el puerto AMQP (5672) solo es accesible dentro de la red Docker. Abrir `15672-15673` (paneles) solo para la IP del administrador.

## Pruebas

- **Cluster:** en el panel, *Overview → Nodes* muestra los 2 nodos en ejecución.
- **Caída de un nodo:** `docker compose stop rabbitmq-2`. Los microservicios conectados a ese nodo se reconectan a `rabbitmq-1`. Una cola que vivía en `rabbitmq-2` no está disponible hasta que el nodo vuelve con `docker compose start rabbitmq-2`.
- **DLQ:** un evento inválido queda en `classflow.messages.dlq` (*Queues* en el panel).

## Estructura

```
.
├── docker-compose.yml        # rabbitmq-1 y rabbitmq-2
├── rabbitmq/rabbitmq.conf    # Formación del cluster y manejo de particiones
├── .env.example              # Variables a copiar en .env
└── README.md
```
