# Especificación — Tienda de Muebles Online

> Versión: v2
> Fecha creación: 2026-06-16
> Fecha última modificación: 2026-06-17
> Estado: BORRADOR
> Autor: equipo

---

## 1. Descripción general

Sistema de e-commerce especializado en la venta de muebles para el hogar y oficina.
Permite a los usuarios explorar un catálogo de productos, gestionar un carrito de compra, realizar pedidos y efectuar pagos online. Los administradores gestionan el catálogo, el stock, los pedidos y los usuarios desde un panel de administración.

---

## 2. Módulos funcionales

### 2.1 Catálogo de productos
- Listado de muebles paginado, con filtros por categoría, precio, material y disponibilidad
- Vista detalle de producto con imágenes, descripción, dimensiones, precio y stock
- Búsqueda por texto libre (nombre, descripción)
- Categorías jerárquicas (ej.: Salón > Sofás > Sofás de 3 plazas)

### 2.2 Autenticación y usuarios
- Registro de usuario con email y contraseña (contraseña almacenada de forma segura)
- Login con JWT (access token + refresh token)
- Roles: `CUSTOMER`, `ADMIN`
- Perfil de usuario: dirección de envío, historial de pedidos
- Recuperación de contraseña por email

### 2.3 Carrito de compra
- Añadir / eliminar productos del carrito
- Modificar cantidades
- El carrito persiste en base de datos para usuarios autenticados
- Cálculo en tiempo real de subtotal, impuestos (IVA 21 %) y total

### 2.4 Pedidos
- Checkout: confirmación de dirección, método de pago y revisión del pedido
- Estados del pedido: `PENDIENTE_PAGO` → `PAGADO` → `EN_PREPARACION` → `ENVIADO` → `ENTREGADO` | `CANCELADO`
- Email de confirmación al realizar el pedido
- Historial de pedidos del usuario con detalle de cada línea

### 2.5 Pagos
- Integración con pasarela de pago externa (Stripe)
- Pago con tarjeta de crédito/débito
- Webhooks de Stripe para actualización de estado del pedido
- Devoluciones parciales o totales desde el panel de admin

### 2.6 Panel de administración
- CRUD de productos (nombre, descripción, imágenes, precio, stock, categoría)
- Gestión de categorías
- Gestión de pedidos: consulta, cambio de estado, cancelación
- Gestión de usuarios: listado, bloqueo/desbloqueo

---

## 3. Criterios de aceptación

### Catálogo
- [ ] Un usuario anónimo puede navegar el catálogo sin autenticarse
- [ ] Los filtros combinados devuelven resultados coherentes
- [ ] La paginación funciona correctamente con cualquier combinación de filtros
- [ ] Un producto sin stock aparece como "Agotado" y no puede añadirse al carrito

### Autenticación
- [ ] El registro falla si el email ya existe (error 409)
- [ ] El JWT expira a los 15 minutos; el refresh token a los 7 días
- [ ] Los endpoints de admin rechazan tokens de rol `CUSTOMER` con 403

### Carrito
- [ ] No es posible añadir más unidades que el stock disponible
- [ ] El carrito se asocia al usuario al hacer login (merge de carrito anónimo si aplica)

### Pedidos
- [ ] El stock se decrementa atómicamente al confirmar el pedido
- [ ] Si el pago falla, el pedido no se crea y el stock no se descuenta
- [ ] El usuario recibe email de confirmación en menos de 2 minutos

### Pagos
- Los datos de tarjeta nunca son procesados por los servidores propios
    - El webhook valida la autenticidad del proveedor de pagos antes de procesar el evento
- [ ] Una devolución actualiza el estado del pedido a `CANCELADO` o `DEVOLUCION_PARCIAL`

### Panel de admin
- [ ] Solo usuarios con rol `ADMIN` pueden acceder
- [ ] La modificación de stock refleja cambios en el catálogo en tiempo real

---

## 4. Casos edge

| Caso | Comportamiento esperado |
|------|------------------------|
| Usuario añade al carrito un producto que se agota entre sesiones | Al hacer checkout se muestra error y se bloquea la compra del item agotado |
| Dos usuarios compran el último stock simultáneamente | Solo uno completa la compra; el otro recibe error de stock insuficiente |
| Webhook de Stripe llega duplicado | Idempotencia: el segundo webhook no altera el estado |
| Usuario borra su cuenta con pedido en curso | El pedido se mantiene; el usuario queda en estado `INACTIVO` |
| Sesión expirada durante checkout | Se redirige a login conservando el carrito |
| Imagen de producto mayor de 5 MB | El backend rechaza el upload con error 413 |
| XSS en nombre de producto | El input se sanitiza antes de guardarse y al renderizarse |
| Inyección SQL en búsqueda | Los parámetros se pasan siempre como bind parameters (JPA/Prepared Statements) |

---

## 5. Requisitos no funcionales

| Atributo | Requisito |
|----------|-----------|
| Rendimiento | Tiempo de respuesta p95 < 300 ms bajo carga normal |
| Disponibilidad | 99,5 % uptime mensual |
| Seguridad | OWASP Top 10 mitigado; HTTPS obligatorio |
| Accesibilidad | WCAG 2.1 nivel AA en el front |
| Internacionalización | Español (ES) como idioma base; preparado para i18n |
| Almacenamiento imágenes | S3-compatible (AWS S3 o MinIO para desarrollo local) |

---

## 6. Fuera de alcance (v1.0)

- App móvil nativa
- Sistema de reseñas y valoraciones
- Programa de fidelización / puntos
- Chat en vivo con soporte
- Comparador de productos
- Multi-tienda / multi-tenant

---
## Changelog

| Versi�n | Fecha | Descripci�n del cambio |
|---------|-------|------------------------|
| v1 | 2026-06-16 | Creaci�n inicial || v2 | 2026-06-17 | Eliminadas referencias a tecnologías específicas (BCrypt, Stripe.js); lenguaje de criterios de pago refactorizado a términos funcionales |