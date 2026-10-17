# EventHub — Dev Environment

Entorno de desarrollo local para la plataforma EventHub (SaaS B2B multi-tenant de gestión de eventos y ticketing). Este repositorio contiene la orquestación en `docker-compose.yml` base y la configuración necesaria para levantar toda la infraestructura y los microservicios en tu máquina.

> Para la visión de producto, modelo de negocio y arquitectura distribuida completa del sistema, ver `PRODUCT_ARCHITECTURE.md`.

---

## Servicios incluidos

| Servicio | Descripción | Puerto |
|---|---|---|
| `api-gateway` | Punto de entrada único (Spring Cloud Gateway WebFlux, validación JWT, seguridad interna) | 8000 |
| `config-service` | Configuración por tenant, esquemas de registro dinámicos (`formSchema`) (MongoDB) | 8081 |
| `auth-service` | Registro/login, emisión y validación de JWT con claims de tenant (Neon PostgreSQL) | 8082 |
| `event-service` | Gestión del catálogo de eventos, tiers de boletas y branding multi-tenant (MongoDB + AWS S3) | 8083 |
| `booking-service` | Motor de reservas, gestión de estados (`AVAILABLE` → `HELD` → `SOLD`), control de expiración (Neon PostgreSQL + RabbitMQ) | 8084 |
| `payment-service` | Stripe Checkout, procesamiento de webhooks y publicación de pagos mediante outbox (Neon PostgreSQL + RabbitMQ) | 8085 |
| `notification-service` | Consumidor asíncrono RabbitMQ, envío de correos HTML con branding dinámico y tickets PDF (MongoDB) | 8086 |
| `mongodb` | Base de datos NoSQL para `config-service`, `event-service` y `notification-service` | 27017 |
| `rabbitmq` | Broker de mensajería asíncrona AMQP (Event-Driven architecture) | 5672 / 15672 (UI); 15692 (métricas internas) |
| `zipkin` | Backend de trazas distribuidas y observabilidad | 9411 |
| `prometheus` | Recolección y monitoreo de métricas | 9090 |
| `alertmanager` | Enrutamiento de alertas Prometheus (perfil opcional `alerting`) | 9093 |

> **Nota de Bases de Datos Relacionales:** Los microservicios `auth-service`, `booking-service` y `payment-service` se conectan directamente a **Neon PostgreSQL (Cloud Serverless)** configurado mediante variables de entorno en `.env`.

---

## Setup

### 1. Requisitos previos

- Docker y Docker Desktop instalados.
- Puertos `5672, 8000, 8081, 8082, 8083, 8084, 8085, 8086, 9090, 9093, 9411, 15672, 27017` libres en tu máquina.

### 2. Configuración de variables de entorno

Copia el archivo de ejemplo y configura las credenciales reales:

```bash
cp .env.example .env

```

Variables principales en `.env`:

| Variable | Descripción |
| --- | --- |
| `MONGO_ROOT_USER` / `MONGO_ROOT_PASS` | Credenciales root de la instancia MongoDB en Docker |
| `GATEWAY_INTERNAL_SECRET` | Clave secreta compartida entre el Gateway y los microservicios internos (`X-Gateway-Secret`) |
| `JWT_SECRET` | Clave HMAC-SHA256 para firmar/verificar tokens JWT |
| `STRIPE_SECRET_KEY` | Clave privada de Stripe para crear Checkout Sessions |
| `STRIPE_WEBHOOK_SECRET` | Secreto `whsec_` del endpoint de Stripe para verificar webhooks |
| `PAYMENT_CHECKOUT_SUCCESS_URL` | URL de retorno tras un pago; incluye `{CHECKOUT_SESSION_ID}` si el cliente necesita recuperar la sesión |
| `PAYMENT_CHECKOUT_CANCEL_URL` | URL de retorno cuando el cliente cancela Checkout |
| `PAYMENT_WEBHOOK_HMAC_SECRET` | Secreto opcional para el webhook HMAC interno heredado; no sirve para verificar webhooks de Stripe |
| `NEON_DB_*` | URLs y credenciales de las bases de datos relacionales en Neon PostgreSQL |
| `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` | Credenciales de AWS IAM para almacenamiento de branding y tickets en AWS S3 |
| `MAIL_USERNAME` / `MAIL_PASSWORD` | Configuración SMTP de correo para `notification-service` |

Stripe Checkout permanecerá deshabilitado hasta configurar las cuatro variables `STRIPE_*` y `PAYMENT_CHECKOUT_*` con valores del entorno. El endpoint público que debe registrarse en Stripe es `POST /api/v1/payments/webhook/stripe`; configura allí los eventos `payment_intent.succeeded` y `payment_intent.payment_failed`. No guardes claves reales en el repositorio.

Para probar webhooks en local, usa Stripe CLI y reenvía los eventos al API Gateway con `stripe listen --forward-to http://localhost:8000/api/v1/payments/webhook/stripe`; configura en `.env` el secreto `whsec_` que Stripe CLI entrega para esa escucha. El endpoint local no es accesible directamente desde Stripe.

### 3. Levantar el ecosistema

```bash
docker compose up -d --build

```

Verifica el estado de los contenedores:

```bash
docker compose ps

```

### 4. Accesos rápidos

* API Gateway: `http://localhost:8000`
* RabbitMQ Management UI: `http://localhost:15672` (usuario/password configurados en `.env`)
* Zipkin Traces: `http://localhost:9411`
* Prometheus Metrics: `http://localhost:9090`

Prometheus recopila las métricas de profundidad por cola de RabbitMQ y evalúa las reglas de `prometheus.rules.yml`: una DLQ con mensajes durante 5 minutos y cualquier outbox con eventos `FAILED` generan alertas. El endpoint de métricas RabbitMQ (15692) solo está disponible dentro de la red Docker.

Para enviar alertas fuera de Prometheus, configura un endpoint compatible con el formato webhook de Alertmanager en el archivo local `secrets\alertmanager-webhook-url` y activa el perfil opcional:

```powershell
docker compose --profile alerting up -d alertmanager
```

El archivo de URL del webhook está excluido de Git. No publiques su contenido ni lo incluyas en logs. Sin ese archivo y un endpoint válido, las reglas siguen evaluándose en Prometheus, pero Alertmanager no podrá entregar notificaciones.

---

## Operación diaria

### Reconstruir un solo servicio tras cambios de código

```bash
docker compose up -d --build <nombre-servicio>

```

### Ver logs en vivo de un microservicio

```bash
docker logs <nombre-contenedor> -f

```

### Reiniciar todo el ecosistema limpiamente

```bash
docker compose down --remove-orphans
docker compose up -d

```

---

## Arquitectura de seguridad interna (Gateway ↔ Microservicios)

Todos los microservicios internos integran el componente `GatewayAccessFilter` que exige la presencia del encabezado HTTP `X-Gateway-Secret` en cada solicitud.

El `api-gateway` inyecta este header automáticamente en todas las peticiones enrutadas. Intentar consumir un microservicio directamente por su puerto (ej. `http://localhost:8083`) sin pasar por el API Gateway rechazará la petición con un error de acceso no autorizado.

---
