# Tareas — Pasarela de Pagos

> Fecha: 2026-06-17
> Plan de referencia: `.specify/plans/pasarela-pagos.md`
> Spec de referencia: `.specify/specs/pasarela-pagos.md`
> Estado: PENDIENTE

---

## Épica 1 — Infraestructura de pagos

### TASK-P001 — Migraciones Flyway (pagos)
- **Estado**: `TODO`
- **Descripción**: Crear migraciones de BD para el módulo de pagos.
- **Subtareas**:
  - [ ] `V11__extend_orders_payment.sql` — añadir a `orders`: `stripe_payment_intent_id`, `payment_method_type`, `paid_at`, `refunded_amount`; ampliar ENUM `status` con `FALLO_PAGO`, `DEVOLUCION_PARCIAL`, `DEVUELTO`
  - [ ] `V12__create_payment_methods.sql` — tabla `payment_methods` (UUID PK, stripe_customer_id, stripe_pm_id, type, brand, last_four, exp_month, exp_year, is_default, UNIQUE constraint)
  - [ ] `V13__create_processed_webhook_events.sql` — tabla `processed_webhook_events` (stripe_event_id PK, event_type, processed_at)
- **Criterio de aceptación**: Flyway aplica V11–V13 sin errores; rollback manual verificado.

### TASK-P002 — Configuración Stripe
- **Estado**: `TODO`
- **Descripción**: Inicializar el SDK de Stripe y gestionar las variables de entorno.
- **Subtareas**:
  - [ ] Dependencia `stripe-java` 25.x en `pom.xml`
  - [ ] `StripeConfig.java`: `Stripe.apiKey = env("STRIPE_SECRET_KEY")` en `@PostConstruct`; nunca hardcodear
  - [ ] Añadir al `.env.example`: `STRIPE_SECRET_KEY`, `STRIPE_PUBLISHABLE_KEY`, `STRIPE_WEBHOOK_SECRET`
  - [ ] Añadir al `application.yml` la clave pública para exponerla al frontend via endpoint `/api/v1/config/stripe-key` (solo publishable key)
  - [ ] Tests de contexto Spring para verificar que la configuración carga correctamente
- **Criterio de aceptación**: La aplicación no arranca si `STRIPE_SECRET_KEY` está vacía. La secret key nunca aparece en logs ni respuestas.

---

## Épica 2 — PaymentIntents

### TASK-P003 — `POST /payments/intent`
- **Estado**: `TODO`
- **Descripción**: Crear un PaymentIntent en Stripe para un pedido.
- **Subtareas**:
  - [ ] Entidad + repositorio: `PaymentMethod`
  - [ ] `PaymentService.createIntent()`:
    - [ ] Verificar que el pedido existe y pertenece al usuario autenticado (400 si no)
    - [ ] Verificar que el pedido no tiene ya un pago completado (409)
    - [ ] Obtener o crear `Stripe Customer` (`cus_...`) vinculado al usuario
    - [ ] Crear `PaymentIntent` con `idempotencyKey = "order-{orderId}-{userId}"`, `amount`, `currency = "eur"`, `payment_method_types` configurables por entorno
    - [ ] Guardar `stripe_payment_intent_id` en la orden
  - [ ] `PaymentController.createIntent()`: devuelve `{ clientSecret }` — nunca la secret key
  - [ ] Tests con mock del Stripe SDK
- **Criterio de aceptación**: Mismo pedido llamado dos veces devuelve el mismo `clientSecret` (idempotente). Pedido ajeno devuelve 400.

### TASK-P004 — Frontend: Stripe.js + `PaymentElement`
- **Estado**: `TODO`
- **Descripción**: Integrar el formulario de pago de Stripe en el checkout.
- **Subtareas**:
  - [ ] Dependencias `@stripe/stripe-js` 4.x y `@stripe/react-stripe-js` 2.x
  - [ ] `loadStripe(publishableKey)` una sola vez (singleton, fuera del render)
  - [ ] `CheckoutForm`: `<Elements stripe={stripePromise} options={{ clientSecret }}>` wrapping `<PaymentElement>`
  - [ ] `stripe.confirmPayment({ elements, confirmParams: { return_url } })` al enviar
  - [ ] Página de `return_url`: leer `payment_intent_client_secret` de URL, mostrar estado
  - [ ] Soporte visual para Apple Pay / Google Pay (Payment Request Button habilitado automáticamente por `PaymentElement`)
  - [ ] Tests de componente
- **Criterio de aceptación**: El formulario muestra métodos de pago según configuración Stripe ES. Pago de prueba con tarjeta `4242 4242 4242 4242` completa el flujo.

---

## Épica 3 — Webhook

### TASK-P005 — `POST /payments/webhook`
- **Estado**: `TODO`
- **Descripción**: Endpoint que recibe y procesa eventos de Stripe con verificación de firma e idempotencia.
- **Subtareas**:
  - [ ] Endpoint excluido de `JwtAuthenticationFilter` y CSRF
  - [ ] Leer body como raw bytes (no deserializar con Jackson antes de verificar)
  - [ ] `Webhook.constructEvent(payload, sigHeader, env("STRIPE_WEBHOOK_SECRET"))` → 400 si firma inválida
  - [ ] Comprobar idempotencia: si `stripe_event_id` ya existe en `processed_webhook_events` → return 200 sin procesar
  - [ ] `WebhookService.process()`:
    - [ ] `payment_intent.succeeded` → `orderService.markAsPaid(orderId)`, publicar `OrderPaidEvent`
    - [ ] `payment_intent.payment_failed` → `orderService.markAsFailed(orderId)`
    - [ ] `payment_intent.canceled` → `orderService.markAsCanceled(orderId)`
    - [ ] `charge.refunded` → `orderService.processRefund(chargeId, refundedAmount)`
  - [ ] Guardar evento en `processed_webhook_events` tras procesarlo
  - [ ] Tests con payloads reales de Stripe (usar fixtures JSON)
- **Criterio de aceptación**: Firma inválida → 400. Evento duplicado → 200 sin doble procesamiento. Stripe recomienda responder en < 30 s.

---

## Épica 4 — Métodos de pago guardados

### TASK-P006 — Guardar método de pago post-checkout
- **Estado**: `TODO`
- **Descripción**: Persistir el método de pago del usuario tras un pago exitoso (opt-in).
- **Subtareas**:
  - [ ] En `payment_intent.succeeded`: si `setup_future_usage = "off_session"`, recuperar `PaymentMethod` del PI
  - [ ] Guardar en tabla `payment_methods` (solo metadatos no sensibles: type, brand, last_four, exp)
  - [ ] Asociar al `Stripe Customer` del usuario
  - [ ] Crear `Stripe Customer` si no existe aún
  - [ ] Tests unitarios
- **Criterio de aceptación**: Después del pago con opt-in, el método aparece en `GET /payments/methods`.

### TASK-P007 — `GET /methods` y `DELETE /methods/{id}`
- **Estado**: `TODO`
- **Descripción**: Endpoints para listar y eliminar métodos de pago guardados del usuario.
- **Subtareas**:
  - [ ] `GET /payments/methods`: devuelve lista de `PaymentMethodDto` del usuario autenticado
  - [ ] `DELETE /payments/methods/{id}`: verificar que pertenece al usuario (403 si no), eliminar en Stripe (`PaymentMethods.detach()`) y en BD
  - [ ] Tests
- **Criterio de aceptación**: Usuario A no puede eliminar método de usuario B (403). Método eliminado desaparece de Stripe y BD.

---

## Épica 5 — Devoluciones

### TASK-P008 — `POST /payments/refunds`
- **Estado**: `TODO`
- **Descripción**: Endpoint de devolución parcial o total para administradores.
- **Subtareas**:
  - [ ] `RefundService.createRefund()`: validar `amount ≤ importe original - ya_devuelto` (400 si supera), llamar `Refunds.create({ paymentIntent, amount, reason })`
  - [ ] Actualizar `refunded_amount` y `status` de la orden (→ `DEVOLUCION_PARCIAL` o `DEVUELTO` si total)
  - [ ] Solo accesible con rol `ADMIN` (`@PreAuthorize`)
  - [ ] Tests con mock del SDK
- **Criterio de aceptación**: Devolución > importe pagado → 400. No-admin → 403.

---

## Épica 6 — Administración

### TASK-P009 — `GET /payments/transactions`
- **Estado**: `TODO`
- **Descripción**: Listado paginado de transacciones para el panel admin.
- **Subtareas**:
  - [ ] Query paginada con filtros: `status`, `from`, `to`, `paymentMethodType`
  - [ ] DTO `TransactionSummaryDto`: `orderId`, `amount`, `status`, `paymentMethodType`, `paidAt`
  - [ ] Solo accesible con rol `ADMIN`
  - [ ] Frontend: tabla con filtros de fecha y estado, exportación CSV (opcional)
  - [ ] Tests
- **Criterio de aceptación**: Solo ADMIN puede acceder. Filtros funcionan de forma independiente y combinada.
