# Especificación — Pasarela de Pagos

> Versión: v2
> Fecha creación: 2026-06-17
> Fecha última modificación: 2026-06-17
> Estado: BORRADOR
> Autor: equipo
> Plan de referencia: `.specify/plans/pasarela-pagos.md`

---

## 1. Descripción general

Módulo de pagos para la Tienda de Muebles Online. Centraliza todos los flujos de cobro, devolución y conciliación, con soporte para los métodos de pago más habituales en el mercado europeo y español.

Los métodos de pago disponibles son: tarjeta de crédito/débito, wallets digitales (Apple Pay, Google Pay), Bizum, Klarna (pago aplazado) y domiciliación bancaria SEPA.

**Requisitos de seguridad:**
- Los datos de tarjeta nunca son procesados ni almacenados en los servidores propios
- Los pagos con tarjeta requieren autenticación adicional del titular cuando el proveedor lo exige
- Los métodos de pago pueden guardarse de forma segura para futuras compras, con consentimiento explícito del usuario
- Las notificaciones de cobro del proveedor de pagos se verifican para garantizar su autenticidad antes de procesarlas
- Una misma notificación de cobro procesada dos veces no debe generar efectos duplicados

---

## 2. Métodos de pago soportados

### 2.1 Tarjeta de crédito / débito
- Visa, Mastercard, Amex
- El usuario introduce sus datos de tarjeta en un formulario de pago seguro
- Si el banco del titular lo requiere, se solicita una verificación adicional de identidad
- El usuario puede guardar la tarjeta para futuros pagos (opt-in explícito)

### 2.2 Wallets digitales
- **Apple Pay**: disponible en dispositivos y navegadores compatibles con Apple Pay
- **Google Pay**: disponible en dispositivos y navegadores compatibles con Google Pay
- El botón del wallet se muestra solo si el dispositivo del usuario lo soporta

### 2.3 Bizum
- El usuario introduce su número de teléfono y confirma el pago desde su app bancaria
- Solo disponible para cuentas bancarias españolas
- Límite por transacción: 1.000 €

### 2.4 Klarna (pago aplazado)
- Modalidades disponibles:
  - **Paga en 3** — 3 plazos sin intereses
  - **Paga en 30 días** — período de gracia
  - **Financiación** — hasta 36 meses (sujeto a aprobación de Klarna)
- La tienda recibe el importe completo al instante; Klarna asume el riesgo de crédito
- Disponible para pedidos entre 35 € y 10.000 €

### 2.5 Domiciliación bancaria SEPA
- Para clientes B2B con pedidos corporativos
- El usuario firma un mandato de domiciliación durante el proceso de pago
- La liquidación no es inmediata (5 días hábiles)
- Las devoluciones por cargo indebido se gestionan desde el panel de administración

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

- El usuario puede optar (opt-in explícito con checkbox) por guardar el método de pago para futuras compras
- El sistema asocia el método al perfil del usuario, almacenando únicamente datos no sensibles (marca, últimos 4 dígitos, fecha de expiración)
- El usuario puede gestionar (listar, eliminar) sus métodos guardados desde su perfil

---

### 3.3 Pago con método guardado

1. El usuario ve sus métodos de pago guardados (marca, últimos 4 dígitos, expiración)
2. Usuario selecciona uno y confirma el pago
3. Si el método requiere verificación adicional, el sistema la gestiona
4. El flujo de confirmación es idéntico al checkout estándar

---

### 3.4 Devolución (Refund)

**Actor:** Admin  
**Tipos:** Total o parcial  

**Pasos:**
1. Admin selecciona pedido en estado `ENTREGADO` o `PAGADO` → pulsa "Emitir devolución"
2. Admin indica importe (total o parcial) y motivo
3. El sistema procesa la devolución a través del proveedor de pagos
4. El proveedor confirma la devolución (3–5 días hábiles para tarjeta, antes para otros métodos)
5. El sistema actualiza el estado del pedido a `DEVUELTO` o `DEVOLUCION_PARCIAL`
6. El cliente recibe un email con el detalle de la devolución

**Restricciones:**
- Solo admins pueden emitir devoluciones
- No se puede devolver más del importe original
- Los pedidos en estado `CANCELADO` sin cargo previo no generan devolución

---

### 3.5 Notificaciones del proveedor de pagos

El sistema recibe notificaciones del proveedor de pagos para mantener el estado de los pedidos actualizado en tiempo real. El sistema reacciona a: confirmación de cobro, fallo de pago, cancelación y devolución.

Las notificaciones duplicadas son ignoradas para evitar inconsistencias. Las notificaciones con origen no verificado son rechazadas.

> Para la implementación técnica (modelo de datos, contratos de API, configuración del proveedor de pagos), ver `.specify/plans/pasarela-pagos.md`.

---

## 4. Criterios de aceptación

- [ ] Un usuario puede completar un pago con tarjeta con verificación adicional de identidad cuando se requiere
- [ ] Un usuario puede pagar con Apple Pay / Google Pay si su dispositivo lo soporta
- [ ] Un usuario español puede pagar con Bizum
- [ ] Un usuario puede seleccionar Klarna y fraccionar el pago en 3 plazos
- [ ] Un usuario puede guardar un método de pago y usarlo en el siguiente pedido sin reintroducirlo
- [ ] Un admin puede emitir una devolución total o parcial desde el panel
- [ ] Una notificación de cobro duplicada no genera un doble cobro ni doble actualización de estado
- [ ] Una notificación con origen no verificado es rechazada y registrada en logs
- [ ] Una sesión de pago caducada no puede confirmar el pedido
- [ ] Los datos de tarjeta nunca son procesados ni almacenados en los servidores propios

---

## 5. Casos edge

| Caso | Comportamiento esperado |
|------|------------------------|
| Cobro confirmado pero la notificación del proveedor llega con retraso | El pedido permanece en `PENDIENTE_PAGO` hasta recibir la notificación; el usuario ve estado "verificando" |
| Usuario cierra el navegador durante la verificación adicional de identidad | El pago no se confirma; el pedido permanece sin confirmar |
| Devolución después de que el banco ya liquidó | El proveedor gestiona la devolución igualmente; puede tardar hasta 10 días en algunas entidades |
| Klarna rechaza la financiación al usuario | El checkout muestra error claro con alternativas de pago; el pedido queda en `FALLO_PAGO` |
| SEPA devuelto por fondos insuficientes | El sistema recibe notificación de fallo → pedido → `FALLO_PAGO`; email de notificación al cliente |
| Bizum no disponible (mantenimiento) | El método se oculta en el checkout si el proveedor reporta que no está disponible |
| Dos tabs del mismo usuario confirman el mismo pedido simultáneamente | El sistema garantiza que solo un cobro se procesa |

---
## Changelog

| Versión | Fecha | Descripción del cambio |
|---------|-------|------------------------|
| v1 | 2026-06-17 | Creación inicial |
| v2 | 2026-06-17 | Eliminado contenido técnico (modelo de datos, APIs, tecnologías específicas); spec refactorizada para contener solo contenido funcional |
