
# Documento Maestro de Producto, Arquitectura y Operaciones — EventHub

**Versión:** 2.0 (Consolidado: Ecosistema Microservicios, Async Event-Driven, AWS S3 & Neon DB)

**Fecha de Última Actualización:** Octubre de 2026

**Estado:** Documento Vivo de Especificación de Arquitectura

---

## 1. Modelo de Negocio

**EventHub** es una plataforma **SaaS B2B Multi-tenant** de gestión de eventos, ticketing y analítica en tiempo real. Permite a empresas organizadoras de eventos operar bajo su propia marca (White-Label / Custom Branding) mediante un esquema de suscripción recurrente sumado a una comisión parametrizable por entrada vendida.

---

## 2. Flujo Vertical Crítico (End-to-End Walkthrough)

```text
[Comprador / Frontend] ──> (1. Reserva Entrada) ──> [API Gateway (8000)]
                                                         │
                                                         ▼
                                             [booking-service (8084)]
                                                         │
                                                         ├── (2. Lock Fila / DB Transaction en Neon Postgres)
                                                         ├── (3. Estado: AVAILABLE ──> HELD)
                                                         │
                                                         ▼
                                             [payment-service (8085)]
                                                         │
                                                         ├── (4. Procesamiento de Pago / Webhook)
                                                         │
                                                         ▼
                                             [booking-service (8084)]
                                                         │
                                                         ├── (5. Estado: HELD ──> SOLD)
                                                         ├── (6. Publica evento AMQP: 'booking.confirmed')
                                                         │
                                                         ▼
                                               [RabbitMQ Broker]
                                                         │
                                                         ▼
                                         [notification-service (8086)]
                                                         │
                                                         ├── (7. Obtiene Branding & Assets de AWS S3)
                                                         ├── (8. Compila plantilla HTML con Thymeleaf)
                                                         └── (9. Genera PDF/QR y envía correo vía SMTP)

```

---

## 3. Matriz de Microservicios y Componentes

| Repositorio / Microservicio | Tecnología / Stack | Responsabilidad Principal | Persistencia / Storage |
| --- | --- | --- | --- |
| `eventhub-api-gateway` | Java 21, Spring Boot 3.x / WebFlux | Enrutamiento dinámico, verificación de JWT, inyección de `X-Gateway-Secret` y propagación de `X-Tenant-Id`. | Stateless |
| `eventhub-config-service` | Java 21, Spring Boot 3.x | Parámetros de negocio por tenant, esquemas dinámicos de formularios de registro (`formSchema`). | MongoDB (`eventhub_config_db`) |
| `eventhub-auth-service` | Java 21, Spring Boot 3.x | Autenticación, registro de usuarios/organizadores, emisión de JWT con claims de tenant. | Neon PostgreSQL |
| `eventhub-event-service` | Java 21, Spring Boot 3.x | Gestión de eventos, localidades/tiers, precios y subida de imágenes de branding (logos, banners). | MongoDB + AWS S3 (`eventhub-branding-prod`) |
| `eventhub-booking-service` | Java 21, Spring Boot 3.x | Motor de reserva de boletas, máquinas de estado (`AVAILABLE` → `HELD` → `SOLD`), control de expiración. | Neon PostgreSQL + RabbitMQ |
| `eventhub-payment-service` | Java 21, Spring Boot 3.x | Integración y procesamiento de pagos / simulador de pasarela y webhooks de confirmación. | Neon PostgreSQL + RabbitMQ |
| `eventhub-notification-service` | Java 21, Spring Boot 3.x | Consumidor AMQP, generación de tickets en PDF con código QR y envío de correos con branding. | MongoDB + AWS S3 |

---

## 4. Estrategia Multirrepo y Visión CI/CD

El proyecto sigue un enfoque **Multirrepo**:

* Un repositorio independiente por cada microservicio (`eventhub-api-gateway`, `eventhub-auth-service`, `eventhub-event-service`, etc.).
* Repositorio de orquestación `eventhub-dev-environment` para desarrollo local y orquestación con Docker Compose.

### Pipeline CI/CD Objetivo (por microservicio)

1. **Trigger:** `push` / `pull_request` sobre la rama `main`.
2. **Build & Test:** Compilación e integración continua utilizando Java 21 y Gradle.
3. **Containerization:** Construcción de imagen OCI (Docker) optimizada.
4. **Publishing:** Publicación automática de la imagen etiquetada en Docker Hub / GitHub Container Registry (GHCR).

---

## 5. Contratos entre Booking, Payment y Notification

Esta sección documenta los contratos implementados actualmente en `eventhub-booking-service`, `eventhub-payment-service` y `eventhub-notification-service`. Los mensajes AMQP son JSON; los importes se serializan como números decimales y las fechas `LocalDateTime` como fecha-hora ISO sin desplazamiento de zona. Los nombres de campos son sensibles a mayúsculas y minúsculas.

### Solicitudes HTTP internas

| Productor → consumidor | Solicitud | Contrato de respuesta y validación |
| --- | --- | --- |
| Booking → Event | `GET /api/v1/events/{eventId}` con `X-Gateway-Secret` | Sobre `{ success, data }`. Booking usa `data.id`, `status`, `eventDate`, `ticketTiers`, `sessions` y `registrationFields` para verificar publicación, sesión, tarifa, precio, capacidad, asientos y datos de asistentes. |
| Payment → Booking | `GET /api/v1/bookings/{bookingId}` con `X-Gateway-Secret`, `X-User-Id` y `X-User-Roles` | Sobre `{ success, data }`. Payment requiere `data.id`, `userId`, `totalPrice`, `status`, `expiresAt` e `inventoryHeld`; Checkout solo continúa si el usuario es propietario, la reserva sigue en `PENDING_PAYMENT`, el inventario está retenido y la reserva no ha expirado. |

Los servicios internos deben enviar el secreto compartido por los canales de red de confianza; los valores del secreto nunca forman parte del body ni de los logs.

### Eventos RabbitMQ

Los exchanges `booking.exchange` y `payment.exchange` son topic exchanges. El body es el payload del evento, no un sobre con el nombre del evento. Las tablas listan los campos del DTO productor actual; `messageId` es una propiedad AMQP, no un campo JSON.

| Productor → consumidor | Exchange / routing key (cola del consumidor) | Campos JSON |
| --- | --- | --- |
| Booking → consumidores de reservas | `booking.exchange` / `booking.created` | `bookingId`, `eventId`, `sessionId`, `ticketTierId`, `userId`, `quantity`, `totalPrice`, `createdAt`. No se identificó un listener de aplicación para este routing key en los servicios revisados. |
| Booking → Notification | `booking.exchange` / `booking.confirmed` (`notification.booking.confirmed.queue`) | `bookingId`, `eventId`, `sessionId`, `userId`, `userEmail`, `transactionId`, `totalPrice`, `tickets[]`; cada ticket lleva `ticketId`, `qrCodeHash`, `ticketTierId` y `seatId`. |
| Booking → Payment | `booking.exchange` / `booking.cancelled` (`payment.booking-cancellation.queue`) | `bookingId`, `transactionId`, `amount`, `currency`. Solicita el reembolso; no confirma que Stripe lo haya completado. |
| Payment → Booking | `payment.exchange` / `payment.completed` (`payment.completed.queue`) | `paymentId`, `bookingId`, `transactionId`, `amount`, `currency`, `paymentProvider`, `completedAt`. Booking confirma la reserva y emite tickets solo si coinciden el importe y la moneda y el hold continúa vigente. |
| Payment → Booking | `payment.exchange` / `payment.failed` (`payment.failed.queue`) | `paymentId`, `bookingId`, `reason`, `failedAt`. Booking conserva el hold hasta su expiración para permitir otro intento. |
| Payment → Booking | `payment.exchange` / `payment.refunded` (`payment.refunded.queue`) | `paymentId`, `bookingId`, `transactionId`, `refundId`, `refundedAt`. Booking cancela tickets y libera inventario vendido solo cuando el reembolso corresponde a la cancelación pendiente. |
| Payment → Booking | `payment.exchange` / `payment.refund.failed` (`payment.refund.failed.queue`) | `paymentId`, `bookingId`, `transactionId`, `reason`, `failedAt`. Una solicitud de cancelación pendiente pasa a revisión manual. |

`booking.confirmed` contiene datos necesarios para emitir tickets y enviar el correo, incluido `userEmail` y los hashes QR. Limite el acceso al exchange y a sus colas a los servicios autorizados; no copie estos payloads a logs.

### Entrega, idempotencia y evolución

Los productores guardan el evento en un outbox con los cambios de dominio y el relay establece `messageId` con el ID persistido del evento. La entrega es al menos una vez: Booking deduplica los eventos `payment.*`, Payment deduplica `booking.cancelled` y Notification deduplica `booking.confirmed` mediante inboxes persistidas. Los consumidores deben confirmar una repetición como procesada sin repetir efectos. Los mensajes históricos sin `messageId` usan claves de compatibilidad derivadas de los identificadores de negocio.

Los listeners reintentan con tres reintentos adicionales, backoff exponencial desde 1 segundo hasta 10 segundos, y envían fallos al DLQ configurado para cada flujo. Los eventos de outbox también pueden quedar en estado `FAILED` tras alcanzar el límite de intentos del relay y requieren intervención operativa; un mensaje en DLQ o un outbox fallido debe inspeccionarse antes de redirigirse o republicarse.

Los payloads actuales **no incluyen un campo `schemaVersion`**; `messageId` identifica la entrega lógica y no la versión del esquema. Para cambios compatibles, agregue solo campos opcionales y mantenga consumidores que toleren su ausencia y campos desconocidos. No cambie el significado ni elimine o renombre campos publicados. Para un cambio incompatible, publique una nueva routing key versionada, despliegue primero consumidores capaces de procesar ambos contratos y cambie después a los productores; retire el contrato anterior solo cuando no queden productores ni mensajes pendientes que lo usen.
