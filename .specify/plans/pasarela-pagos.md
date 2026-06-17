# Plan técnico — Pasarela de Pagos

> Versión: v1
> Fecha creación: 2026-06-17
> Fecha última modificación: 2026-06-17
> Spec de referencia: `.specify/specs/pasarela-pagos.md`
> Estado: BORRADOR

---

## 1. Stack tecnológico

### Backend
| Librería | Versión | Propósito |
|----------|---------|-----------|
| Stripe Java SDK | 25.x | Integración con Stripe (PaymentIntents, Customers, Refunds) |
| Spring Web | 3.3.x | Endpoint de webhook |
| Spring Events | 3.3.x | Publicación de `OrderPaidEvent` desacoplada del webhook |

### Frontend
| Librería | Versión | Propósito |
|----------|---------|-----------|
| `@stripe/stripe-js` | 4.x | Carga de Stripe.js (PCI DSS compliant) |
| `@stripe/react-stripe-js` | 2.x | `<Elements>`, `<PaymentElement>` — formulario de pago |

### Proveedor principal
**Stripe** actúa como orquestador de todos los métodos de pago. Bizum, Apple Pay, Google Pay, Klarna y SEPA se activan como `payment_methods` dentro de la plataforma Stripe para cuentas con cuenta bancaria española (Stripe ES).

---

## 2. Modelo de datos

```
PaymentMethod               -- métodos de pago guardados por usuario
  id                    UUID PK DEFAULT gen_random_uuid()
  user_id               BIGINT NOT NULL FK → User ON DELETE CASCADE
  stripe_customer_id    VARCHAR(64) NOT NULL    -- cus_...
  stripe_pm_id          VARCHAR(64) NOT NULL    -- pm_...
  type                  ENUM('CARD','BIZUM','APPLE_PAY','GOOGLE_PAY','KLARNA','SEPA') NOT NULL
  brand                 VARCHAR(20)             -- visa, mastercard... (solo CARD)
  last_four             CHAR(4)                 -- (solo CARD)
  exp_month             SMALLINT                -- (solo CARD)
  exp_year              SMALLINT                -- (solo CARD)
  is_default            BOOLEAN DEFAULT FALSE
  created_at            TIMESTAMP NOT NULL
  UNIQUE (user_id, stripe_pm_id)

ProcessedWebhookEvent       -- idempotencia de webhooks Stripe
  stripe_event_id       VARCHAR(64) PK           -- evt_...
  event_type            VARCHAR(64) NOT NULL
  processed_at          TIMESTAMP NOT NULL DEFAULT NOW()
```

### Extensión de tabla `Order` (migración adicional)

```sql
-- V11__extend_orders_payment.sql
ALTER TABLE orders ADD COLUMN stripe_payment_intent_id VARCHAR(64);
ALTER TABLE orders ADD COLUMN payment_method_type      VARCHAR(20);
ALTER TABLE orders ADD COLUMN paid_at                  TIMESTAMP;
ALTER TABLE orders ADD COLUMN refunded_amount          DECIMAL(10,2) DEFAULT 0.00;
ALTER TABLE orders ADD COLUMN status ... -- añadir valores: FALLO_PAGO, DEVOLUCION_PARCIAL, DEVUELTO
```

### Migraciones Flyway

| Versión | Descripción |
|---------|-------------|
| `V11__extend_orders_payment.sql` | Nuevas columnas de pago en `orders` |
| `V12__create_payment_methods.sql` | Tabla `payment_methods` |
| `V13__create_processed_webhook_events.sql` | Tabla `processed_webhook_events` |

---

## 3. Flujo técnico: Checkout estándar

```
Frontend (React)                Backend (Spring)              Stripe
     │                               │                           │
     │ POST /api/v1/payments/intent  │                           │
     │ { orderId }                   │                           │
     │ ─────────────────────────────▶│                           │
     │                               │ PaymentIntents.create(    │
     │                               │   amount, currency,       │
     │                               │   idempotencyKey=         │
     │                               │   "order-{id}-{userId}",  │
     │                               │   payment_method_types    │
     │                               │   customer=stripeCustomer │
     │                               │ ) ────────────────────────▶
     │                               │ ◀────────────────────────│
     │                               │  { id: pi_...,           │
     │                               │    client_secret }        │
     │ { clientSecret }              │                           │
     │ ◀─────────────────────────────│                           │
     │                               │                           │
     │ stripe.confirmPayment(        │                           │
     │   clientSecret, elements)     │                           │
     │ ─────────────────────────────────────────────────────────▶│
     │                               │                      3DS2 / Bizum
     │                               │                      / Apple Pay
     │ ◀─────────────────────────────────────────────────────────│
     │  return_url redirect           │                           │
     │                               │                           │
     │                               │ ◀── POST /api/v1/payments/webhook
     │                               │     payment_intent.succeeded
     │                               │     (verificar Stripe-Signature)
     │                               │                           │
     │                               │ order.status = PAGADO     │
     │                               │ publish OrderPaidEvent    │
     │                               │   → reduce stock          │
     │                               │   → send email            │
```

---

## 4. Contratos de API

### Base URL: `/api/v1/payments`

#### POST `/intent`
**Acceso:** `CUSTOMER` autenticado  
**Request:**
```json
{ "orderId": 42 }
```
**Response `200 OK`:**
```json
{ "clientSecret": "pi_3abc_secret_xyz" }
```
| Código | Descripción |
|--------|-------------|
| `200 OK` | PaymentIntent creado; devuelve `clientSecret` para Stripe.js |
| `400 Bad Request` | Pedido no existe o no pertenece al usuario |
| `409 Conflict` | El pedido ya tiene un pago completado |

---

#### GET `/methods`
**Acceso:** `CUSTOMER` autenticado  
**Response `200 OK`:**
```json
[
  {
    "id": "uuid",
    "type": "CARD",
    "brand": "visa",
    "lastFour": "4242",
    "expMonth": 12,
    "expYear": 2027,
    "isDefault": true
  }
]
```

---

#### DELETE `/methods/{id}`
**Acceso:** `CUSTOMER` autenticado (solo sus propios métodos)  
| Código | Descripción |
|--------|-------------|
| `204 No Content` | Método eliminado en Stripe y BD |
| `404 Not Found` | No existe o no pertenece al usuario |

---

#### POST `/webhook`
**Acceso:** Público — verificado con `Stripe-Signature` header  
**Flujo de procesamiento:**
```java
// 1. Verificar firma
Event event = Webhook.constructEvent(
    payload, sigHeader, System.getenv("STRIPE_WEBHOOK_SECRET")
);
// 2. Comprobar idempotencia
if (processedWebhookEventRepo.existsById(event.getId())) return 200;

// 3. Procesar según tipo
switch (event.getType()) {
    case "payment_intent.succeeded"    -> orderService.markAsPaid(pi.getMetadata("orderId"));
    case "payment_intent.payment_failed" -> orderService.markAsFailed(pi.getMetadata("orderId"));
    case "payment_intent.canceled"     -> orderService.markAsCanceled(pi.getMetadata("orderId"));
    case "charge.refunded"             -> orderService.processRefund(chargeId, refundedAmount);
}
// 4. Registrar evento procesado
processedWebhookEventRepo.save(new ProcessedWebhookEvent(event.getId(), event.getType()));
```
| Código | Descripción |
|--------|-------------|
| `200 OK` | Evento procesado (o ya procesado — idempotente) |
| `400 Bad Request` | Firma inválida |

---

#### POST `/refunds`
**Acceso:** `ADMIN`  
**Request:**
```json
{
  "orderId": 42,
  "amount": 150.00,
  "reason": "REQUESTED_BY_CUSTOMER"
}
```
**Stripe `reason` values:** `DUPLICATE`, `FRAUDULENT`, `REQUESTED_BY_CUSTOMER`  
| Código | Descripción |
|--------|-------------|
| `200 OK` | Devolución creada en Stripe |
| `400 Bad Request` | Importe > importe original o pedido sin pago |
| `403 Forbidden` | No es admin |

---

#### GET `/transactions`
**Acceso:** `ADMIN`  
**Query params:** `page`, `size`, `status`, `from`, `to`, `paymentMethodType`  
**Response `200 OK`:** Lista paginada con `orderId`, `amount`, `status`, `paymentMethodType`, `paidAt`

---

## 5. Configuración de métodos de pago en Stripe

```yaml
# Activar en Stripe Dashboard > Settings > Payment methods:
# - Cards (Visa, Mastercard, Amex)        → automático
# - Apple Pay / Google Pay                → wallet: activar Payment Request Button
# - Bizum                                 → requiere cuenta Stripe España
# - Klarna                                → activar en Stripe Dashboard
# - SEPA Direct Debit                     → requiere mandato en checkout

# Configuración PaymentElement en frontend:
stripe.elements({
  clientSecret,
  appearance: { theme: 'stripe' }
})
# El PaymentElement muestra automáticamente los métodos disponibles
# según país del usuario y configuración del PaymentIntent.
```

### Variables de entorno Stripe

```
STRIPE_SECRET_KEY=sk_live_...          # Nunca exponer en frontend
STRIPE_PUBLISHABLE_KEY=pk_live_...     # Seguro para frontend
STRIPE_WEBHOOK_SECRET=whsec_...        # Secret del endpoint de webhook
```

---

## 6. Guardado de método de pago — Stripe Customer

```
Al primer checkout con opt-in guardado:
  1. Stripe Customer ya existe (creado en registro de usuario)
     o crear: Customers.create({ email, metadata: { userId } })
  2. PaymentIntent con setup_future_usage = "off_session"
  3. Tras payment_intent.succeeded:
     → recuperar PaymentMethod del PI
     → guardar en tabla payment_methods (datos no sensibles)
     → asociar al Stripe Customer

Pago con método guardado:
  PaymentIntents.create({
    customer: stripeCustomerId,
    payment_method: stripePmId,
    confirm: true,
    off_session: false   // usuario presente, puede hacer 3DS2
  })
```

---

## 7. Estructura de ficheros

```
payment/
├── controller/
│   └── PaymentController.java           # /api/v1/payments/**
├── service/
│   ├── PaymentService.java              # Lógica de PaymentIntents, métodos guardados
│   ├── WebhookService.java              # Procesamiento de eventos Stripe
│   └── RefundService.java               # Devoluciones
├── repository/
│   ├── PaymentMethodRepository.java
│   └── ProcessedWebhookEventRepository.java
├── domain/
│   ├── PaymentMethod.java
│   └── ProcessedWebhookEvent.java
├── dto/
│   ├── CreatePaymentIntentRequest.java
│   ├── PaymentIntentResponse.java
│   ├── PaymentMethodDto.java
│   ├── RefundRequest.java
│   └── TransactionSummaryDto.java
├── config/
│   └── StripeConfig.java               # Stripe.apiKey = env(STRIPE_SECRET_KEY)
└── exception/
    ├── PaymentNotFoundException.java
    └── RefundException.java
```

---

## 8. Consideraciones de seguridad

| Aspecto | Implementación |
|---------|----------------|
| PCI DSS | Stripe.js / Payment Element — datos de tarjeta nunca llegan al backend |
| 3DS2 | Gestionado automáticamente por Stripe; backend no interviene |
| Webhook auth | `Webhook.constructEvent()` con `STRIPE_WEBHOOK_SECRET` — rechaza si firma inválida |
| Idempotencia | `ProcessedWebhookEvent` en BD con `stripe_event_id` como PK |
| Concurrencia | `idempotencyKey = "order-{orderId}-{userId}"` en `PaymentIntents.create()` |
| Secretos | `STRIPE_SECRET_KEY` solo en backend; `STRIPE_PUBLISHABLE_KEY` en frontend |
| SEPA mandato | PDF generado por Stripe; firmado digitalmente por el usuario en el checkout |

---

## 9. Decisiones técnicas

| # | Decisión | Alternativas | Razón |
|---|----------|--------------|-------|
| 1 | Stripe como orquestador único | Redsys + Klarna por separado | Una sola integración cubre tarjeta, wallets, Bizum (ES), Klarna y SEPA; menos mantenimiento |
| 2 | Payment Element (Stripe UI) | Elementos individuales CardElement | Soporta todos los métodos en un componente; PCI DSS sin configuración extra |
| 3 | Confirmación por webhook (no por redirect) | Confiar en `return_url` redirect | El redirect puede ser interceptado; solo el webhook garantiza el cobro real |
| 4 | `ProcessedWebhookEvent` en BD | Flag en `Order` | Desacopla idempotencia de pagos del modelo de pedidos; funciona para todos los eventos |
| 5 | `setup_future_usage = "off_session"` al guardar | Segundo PaymentIntent de setup | Un solo flow para cobrar y guardar a la vez |
| 6 | Stripe Customer creado en registro | Crear en primer checkout | Permite guardar métodos de pago sin que el usuario haya hecho un pedido previo |

---
## Changelog

| Versión | Fecha | Descripción del cambio |
|---------|-------|------------------------|
| v1 | 2026-06-17 | Creación inicial — plan técnico completo de pasarela de pagos (Stripe, modelo de datos, APIs, webhooks, decisiones) |
