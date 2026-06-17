# Especificación — Pasarela de Pagos

> Versión: v1
> Fecha creación: 2026-06-17
> Fecha última modificación: 2026-06-17
> Estado: BORRADOR
> Autor: equipo

---

## 1. Descripción general

Módulo de pasarela de pagos para la Tienda de Muebles Online. Centraliza todos los flujos de cobro, devolución y conciliación, integrando métodos de pago modernos adaptados al mercado europeo y español.

**Stack de pagos seleccionado:**

| Proveedor | Rol |
|-----------|-----|
| **Stripe Payment Element** | Orquestador principal: tarjetas, wallets, BNPL |
| **Bizum** (vía Stripe o Redsys) | Método local España — pago instantáneo |
| **Apple Pay / Google Pay** | Wallets digitales en checkout web y móvil |
| **Klarna** | Buy Now Pay Later (BNPL) — clave en venta de muebles |
| **SEPA Direct Debit** | Domiciliación bancaria para pedidos corporativos |

**Requisitos de seguridad:**
- Cumplimiento **PCI DSS nivel 1** — datos de tarjeta nunca tocan los servidores propios
- **3D Secure 2.0 (3DS2)** obligatorio para todas las transacciones con tarjeta
- Tokenización de métodos de pago para compras recurrentes / guardado de tarjeta
- Firma y verificación de todos los webhooks entrantes
- Cifrado TLS 1.3 en todos los endpoints de pago
- Idempotency keys en todas las peticiones a APIs externas

---

## 2. Métodos de pago soportados

### 2.1 Tarjeta de crédito / débito (Stripe)
- Visa, Mastercard, Amex
- Interfaz: **Stripe Payment Element** (componente React preintegrado, PCI DSS out-of-the-box)
- 3DS2 gestionado automáticamente por Stripe
- Guardado de tarjeta tokenizada para futuros pagos (opt-in explícito del usuario)

### 2.2 Wallets digitales
- **Apple Pay**: activado vía Stripe si el navegador lo soporta; requiere verificación de dominio con Apple
- **Google Pay**: activado vía Stripe; disponible en Chrome/Android
- Detección automática en frontend — el botón del wallet se muestra solo si el dispositivo lo soporta

### 2.3 Bizum
- Integración mediante **Stripe + Bizum** (disponible desde 2024 en la plataforma Stripe España) o alternativamente mediante **Redsys** (pasarela de referencia para Bizum en España)
- Flujo: usuario introduce número de teléfono → confirmación en app Bizum → webhook de confirmación
- Solo disponible para cuentas bancarias españolas
- Límite por transacción: 1.000 € (límite Bizum actual)

### 2.4 Klarna (Buy Now Pay Later)
- Modalidades disponibles:
  - **Paga en 3** — 3 plazos sin intereses
  - **Paga en 30 días** — período de gracia
  - **Financiación** — hasta 36 meses (sujeto a aprobación de Klarna)
- Integración vía **Stripe + Klarna** (Klarna como payment method en Payment Element)
- El vendedor recibe el importe completo al instante; Klarna asume el riesgo de crédito
- Disponible para pedidos entre 35 € y 10.000 €

### 2.5 SEPA Direct Debit
- Para clientes B2B con pedidos recurrentes o corporativos
- Mandato SEPA generado y firmado digitalmente durante el checkout
- Liquidación en 5 días hábiles (no inmediata)
- Gestión de devoluciones (chargebacks) SEPA dentro del panel de admin

---

## 3. Flujos funcionales

### 3.1 Checkout estándar

**Actor:** Usuario autenticado con carrito no vacío  
**Entrada:** Carrito, dirección de entrega, método de pago seleccionado  
**Salida:** Pedido creado en estado `PENDIENTE_PAGO`, PaymentIntent creado en Stripe  

**Pasos:**
1. Usuario confirma carrito y dirección → pulsa "Ir a pagar"
2. Backend crea un **Stripe PaymentIntent** con el importe exacto e idempotency key = `pedido-{orderId}-{userId}`
3. Frontend monta el **Stripe Payment Element** con el `client_secret` recibido
4. Usuario selecciona método de pago y confirma
5. Stripe gestiona 3DS2 si es requerido
6. Stripe confirma el pago → dispara webhook `payment_intent.succeeded`
7. Backend recibe webhook, verifica firma (`Stripe-Signature` header con secret de endpoint), actualiza pedido a `PAGADO`
8. Backend publica evento interno `OrderPaidEvent` → trigger de email de confirmación y reducción de stock
9. Frontend recibe confirmación y redirige a página de éxito

**Restricciones:**
- El PaymentIntent expira en 24 horas si no se completa
- No se confirma ningún pedido sin webhook verificado (no se confía solo en el redirect del frontend)

---

### 3.2 Guardado de método de pago

- El usuario puede optar (opt-in explícito con checkbox) por guardar la tarjeta para futuros pagos
- Backend crea un **Stripe Customer** vinculado al usuario si no existe
- El `PaymentMethod` se adjunta al `Customer` de Stripe; en base de datos se guarda solo el `paymentMethodId` y los últimos 4 dígitos + marca (nunca datos sensibles)
- El usuario puede gestionar (listar, eliminar) sus métodos guardados desde su perfil

---

### 3.3 Pago con método guardado

1. Frontend lista los `PaymentMethod` del usuario (datos no sensibles: marca, últimos 4, expiración)
2. Usuario selecciona uno → backend crea PaymentIntent con `customer` y `payment_method` pre-rellenados
3. Si requiere 3DS2, Stripe lo gestiona; si no, se confirma directamente
4. Flujo de webhook idéntico al checkout estándar

---

### 3.4 Devolución (Refund)

**Actor:** Admin  
**Tipos:** Total o parcial  

**Pasos:**
1. Admin selecciona pedido en estado `ENTREGADO` o `PAGADO` → pulsa "Emitir devolución"
2. Admin indica importe (total o parcial) y motivo
3. Backend crea un **Stripe Refund** contra el `PaymentIntent` original
4. Stripe procesa la devolución (3–5 días hábiles para tarjeta, inmediato para Bizum/wallets)
5. Webhook `charge.refunded` recibido y verificado → pedido actualizado a `DEVUELTO` (total) o `DEVOLUCION_PARCIAL`
6. Email automático al cliente con el detalle de la devolución

**Restricciones:**
- Solo admins pueden emitir devoluciones
- No se puede devolver más del importe original
- Los pedidos en estado `CANCELADO` sin cargo previo no generan refund

---

### 3.5 Gestión de webhooks

Todos los webhooks de Stripe se reciben en `POST /api/payments/webhook`.

| Evento | Acción en backend |
|--------|-------------------|
| `payment_intent.succeeded` | Pedido → `PAGADO`; publicar `OrderPaidEvent` |
| `payment_intent.payment_failed` | Pedido → `FALLO_PAGO`; notificar usuario |
| `payment_intent.canceled` | Pedido → `CANCELADO` |
| `charge.refunded` | Pedido → `DEVUELTO` o `DEVOLUCION_PARCIAL` |
| `customer.subscription.deleted` | (reservado para futuros planes) |

**Seguridad de webhooks:**
- Verificación obligatoria de `Stripe-Signature` usando `Stripe.constructEvent()` con el webhook secret almacenado en variable de entorno
- Si la firma no es válida → responder `400 Bad Request` y log de alerta
- Idempotencia: cada evento se procesa una sola vez (tabla `processed_webhook_events` con el `event.id`)

---

## 4. Modelo de datos (extensión)

### Tabla `payment_methods` (métodos guardados por usuario)
| Campo | Tipo | Descripción |
|-------|------|-------------|
| `id` | UUID | PK |
| `user_id` | UUID | FK → users |
| `stripe_payment_method_id` | VARCHAR(64) | ID en Stripe |
| `stripe_customer_id` | VARCHAR(64) | Customer en Stripe |
| `brand` | VARCHAR(20) | visa, mastercard, klarna… |
| `last_four` | CHAR(4) | Últimos 4 dígitos (solo tarjeta) |
| `exp_month` | SMALLINT | Mes expiración |
| `exp_year` | SMALLINT | Año expiración |
| `type` | ENUM | CARD, BIZUM, APPLE_PAY, GOOGLE_PAY, KLARNA, SEPA |
| `is_default` | BOOLEAN | Método por defecto del usuario |
| `created_at` | TIMESTAMP | — |

### Tabla `processed_webhook_events` (idempotencia)
| Campo | Tipo | Descripción |
|-------|------|-------------|
| `stripe_event_id` | VARCHAR(64) | PK — ID único del evento Stripe |
| `event_type` | VARCHAR(64) | Tipo de evento |
| `processed_at` | TIMESTAMP | Cuándo se procesó |

### Extensión tabla `orders`
| Campo nuevo | Tipo | Descripción |
|-------------|------|-------------|
| `stripe_payment_intent_id` | VARCHAR(64) | ID del PaymentIntent |
| `payment_method_type` | ENUM | CARD, BIZUM, APPLE_PAY, GOOGLE_PAY, KLARNA, SEPA |
| `paid_at` | TIMESTAMP | Timestamp del cobro efectivo |
| `refunded_amount` | DECIMAL(10,2) | Importe devuelto (0 si ninguno) |

---

## 5. APIs expuestas

| Método | Endpoint | Auth | Descripción |
|--------|----------|------|-------------|
| `POST` | `/api/payments/intent` | `CUSTOMER` | Crea PaymentIntent para un pedido |
| `GET` | `/api/payments/methods` | `CUSTOMER` | Lista métodos de pago guardados |
| `DELETE` | `/api/payments/methods/{id}` | `CUSTOMER` | Elimina método de pago guardado |
| `POST` | `/api/payments/webhook` | Pública (firma Stripe) | Recibe eventos de Stripe |
| `POST` | `/api/payments/refunds` | `ADMIN` | Emite devolución total o parcial |
| `GET` | `/api/payments/transactions` | `ADMIN` | Listado de transacciones con filtros |

---

## 6. Criterios de aceptación

- [ ] Un usuario puede completar un pago con tarjeta pasando 3DS2
- [ ] Un usuario puede pagar con Apple Pay / Google Pay si su dispositivo lo soporta
- [ ] Un usuario español puede pagar con Bizum
- [ ] Un usuario puede seleccionar Klarna y fraccionar el pago en 3 plazos
- [ ] Un usuario puede guardar una tarjeta y usarla en el siguiente pedido sin reintroducirla
- [ ] Un admin puede emitir una devolución total o parcial desde el panel
- [ ] Un webhook duplicado de Stripe no genera un doble cobro ni doble actualización de estado
- [ ] Un webhook con firma inválida es rechazado con 400 y registrado en logs
- [ ] El PaymentIntent caducado no puede completar el pedido
- [ ] Los datos de tarjeta nunca son procesados ni almacenados en los servidores propios

---

## 7. Casos edge

| Caso | Comportamiento esperado |
|------|------------------------|
| Pago aprobado pero webhook llega con retraso | El pedido permanece en `PENDIENTE_PAGO` hasta recibir el webhook; el frontend muestra estado "verificando" |
| Usuario cierra el navegador durante 3DS2 | El PaymentIntent queda en `requires_action`; el pedido no se confirma |
| Devolución después de que el banco ya liquidó | Stripe gestiona el refund igualmente; puede tardar hasta 10 días en algunas entidades |
| Klarna rechaza la financiación al usuario | El checkout muestra error claro con alternativas de pago; el pedido queda en `FALLO_PAGO` |
| SEPA devuelto por fondos insuficientes | Webhook `charge.failed` recibido → pedido → `FALLO_PAGO`; email de notificación al cliente |
| Bizum no disponible (mantenimiento) | El método se oculta en el frontend si Stripe reporta el método como no disponible |
| Concurrencia: dos tabs del mismo usuario confirman el mismo pedido | Idempotency key en el PaymentIntent impide doble cobro |

---
## Changelog

| Versión | Fecha | Descripción del cambio |
|---------|-------|------------------------|
| v1 | 2026-06-17 | Creación inicial — pasarela de pagos moderna con Stripe, Bizum, Apple Pay, Google Pay, Klarna y SEPA |
