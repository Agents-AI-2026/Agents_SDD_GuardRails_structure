# Tareas — Tienda de Muebles Online

> Fecha: 2026-06-16
> Plan de referencia: `.specify/plans/tienda-muebles-online.md`
> Estado: PENDIENTE

---

## Épica 1 — Infraestructura base

### TASK-001 — Scaffolding del proyecto
- **Estado**: `TODO`
- **Descripción**: Inicializar los proyectos backend y frontend con la estructura de carpetas definida en el plan técnico.
- **Subtareas**:
  - [ ] Crear proyecto Spring Boot con Spring Initializr (Java 17, dependencias: Web, Security, Data JPA, Validation, Mail, Flyway, Lombok)
  - [ ] Crear proyecto React + TypeScript con Vite
  - [ ] Configurar `docker-compose.yml` con H2 (modo servidor para dev) y MinIO
  - [ ] Configurar `application.yml` y `application-dev.yml` (sin secrets hardcodeados; usar variables de entorno)
  - [ ] Configurar Flyway en el backend
- **Criterio de aceptación**: `docker compose up` levanta la BD, MinIO, backend y frontend sin errores.

### TASK-002 — Seguridad base (Spring Security + JWT)
- **Estado**: `TODO`
- **Descripción**: Implementar la capa de autenticación y autorización.
- **Subtareas**:
  - [ ] Configurar `SecurityConfig` con rutas públicas / protegidas
  - [ ] Implementar `JwtService` (generación, validación, extracción de claims)
  - [ ] Implementar `JwtAuthenticationFilter`
  - [ ] Implementar `RefreshTokenService`
  - [ ] Configurar cookie HttpOnly para el refresh token
  - [ ] Tests unitarios de `JwtService`
- **Criterio de aceptación**: Endpoint protegido devuelve 401 sin token; devuelve 200 con token válido.

---

## Épica 2 — Autenticación y usuarios

### TASK-003 — Registro y login
- **Estado**: `TODO`
- **Descripción**: Endpoints `POST /api/v1/auth/register` y `POST /api/v1/auth/login`.
- **Subtareas**:
  - [ ] Migración V3 (tabla User)
  - [ ] Entidad `User` + `UserRepository`
  - [ ] `UserService.register()` con validación de email único y hash BCrypt
  - [ ] `AuthController` con DTOs de request/response
  - [ ] Tests de integración con H2 en memoria
- **Criterio de aceptación**: Registro con email duplicado devuelve 409. Login correcto devuelve JWT.

### TASK-004 — Refresh token y logout
- **Estado**: `TODO`
- **Descripción**: Endpoints `POST /api/v1/auth/refresh` y `POST /api/v1/auth/logout`.
- **Subtareas**:
  - [ ] Migración V7 (tabla RefreshToken)
  - [ ] `RefreshToken` entidad + repositorio
  - [ ] Lógica de rotación de refresh token
  - [ ] Logout revoca el refresh token en BD
  - [ ] Tests unitarios
- **Criterio de aceptación**: Refresh token expirado devuelve 401. Después de logout el refresh token no funciona.

### TASK-005 — Recuperación de contraseña
- **Estado**: `TODO`
- **Descripción**: Endpoints `forgot-password` y `reset-password`.
- **Subtareas**:
  - [ ] Generar token de recuperación (UUID, expiración 1 hora) almacenado en BD
  - [ ] Envío de email con Spring Mail
  - [ ] Validar token y cambiar contraseña
  - [ ] Tests unitarios con mock de email
- **Criterio de aceptación**: Token expirado devuelve 400. Token usado una segunda vez devuelve 400.

### TASK-006 — Perfil y direcciones
- **Estado**: `TODO`
- **Descripción**: Endpoints de perfil (`/users/me`) y gestión de direcciones.
- **Subtareas**:
  - [ ] Migración V4 (tabla Address)
  - [ ] Entidad `Address` + repositorio
  - [ ] CRUD de direcciones asociadas al usuario autenticado
  - [ ] Tests
- **Criterio de aceptación**: Un usuario no puede ver ni editar direcciones de otro usuario (403).

---

## Épica 3 — Catálogo

### TASK-007 — Categorías
- **Estado**: `TODO`
- **Descripción**: Modelo jerárquico de categorías y sus endpoints.
- **Subtareas**:
  - [ ] Migración V1 (tabla Category con self-reference)
  - [ ] Entidad `Category` + repositorio
  - [ ] Endpoint `GET /categories` devuelve árbol completo
  - [ ] CRUD admin de categorías
  - [ ] Tests
- **Criterio de aceptación**: El árbol de categorías se serializa correctamente (padre → hijos anidados).

### TASK-008 — Productos
- **Estado**: `TODO`
- **Descripción**: Modelo de productos y endpoints de catálogo.
- **Subtareas**:
  - [ ] Migraciones V2 (tabla Product)
  - [ ] Entidad `Product` + repositorio
  - [ ] Listado paginado con filtros (categoría, precio min/max, disponibilidad)
  - [ ] Búsqueda por texto con `LIKE` (H2-compatible) en nombre y descripción
  - [ ] CRUD admin con validaciones
  - [ ] Tests de integración
- **Criterio de aceptación**: Filtros combinados devuelven resultados correctos. Producto inactivo no aparece en catálogo público.

### TASK-009 — Upload de imágenes
- **Estado**: `TODO`
- **Descripción**: Upload de imágenes de producto a S3/MinIO.
- **Subtareas**:
  - [ ] Configurar cliente AWS SDK / MinIO
  - [ ] `ImageService.upload()` con validación de tipo MIME (solo `image/jpeg`, `image/png`, `image/webp`) y tamaño ≤ 5 MB
  - [ ] Endpoint `POST /products/{id}/images`
  - [ ] Tests con mock de S3
- **Criterio de aceptación**: Archivo > 5 MB devuelve 413. Tipo no permitido devuelve 415.

---

## Épica 4 — Carrito

### TASK-010 — Gestión del carrito
- **Estado**: `TODO`
- **Descripción**: Lógica de carrito persistente para usuarios autenticados.
- **Subtareas**:
  - [ ] Migraciones V5 (Cart + CartItem)
  - [ ] Entidades + repositorios
  - [ ] `CartService`: add, update, remove, clear
  - [ ] Validación de stock al añadir item
  - [ ] Cálculo de totales (subtotal, IVA 21 %, total)
  - [ ] Merge de carrito anónimo al hacer login
  - [ ] Tests
- **Criterio de aceptación**: No se puede añadir más cantidad que el stock disponible. El IVA se calcula correctamente.

---

## Épica 5 — Pedidos y pagos

### TASK-011 — Checkout y creación de pedido
- **Estado**: `TODO`
- **Descripción**: Flujo de checkout: validación, creación de pedido y decremento de stock.
- **Subtareas**:
  - [ ] Migraciones V6 (Order + OrderItem)
  - [ ] Entidades + repositorios
  - [ ] `OrderService.checkout()` con `SELECT FOR UPDATE` en stock
  - [ ] Si stock insuficiente → excepción; pedido no se crea
  - [ ] Tests de integración con H2 en memoria (race condition simulada)
- **Criterio de aceptación**: Dos peticiones simultáneas al último item; solo una crea el pedido.

### TASK-012 — Integración Stripe (PaymentIntent)
- **Estado**: `TODO`
- **Descripción**: Crear PaymentIntent y procesar el resultado desde el front.
- **Subtareas**:
  - [ ] `PaymentService.createIntent()` usando Stripe Java SDK
  - [ ] Endpoint `POST /payments/intent` devuelve `client_secret`
  - [ ] Front: integración con `@stripe/react-stripe-js` y `CardElement`
  - [ ] Tests con mock de Stripe SDK
- **Criterio de aceptación**: El `client_secret` se devuelve al front sin exponer la API key de Stripe.

### TASK-013 — Webhook de Stripe
- **Estado**: `TODO`
- **Descripción**: Endpoint que procesa eventos de Stripe y actualiza el estado de pedidos.
- **Subtareas**:
  - [ ] Endpoint `POST /payments/webhook` (excluido de CSRF y autenticación JWT)
  - [ ] Validación de firma `Stripe-Signature` con `Webhook.constructEvent()`
  - [ ] Manejo de eventos: `payment_intent.succeeded`, `payment_intent.payment_failed`
  - [ ] Idempotencia: verificar que el evento no fue procesado ya
  - [ ] Tests con payload de Stripe simulado
- **Criterio de aceptación**: Webhook con firma inválida devuelve 400. Evento duplicado no cambia el estado.

### TASK-014 — Devoluciones
- **Estado**: `TODO`
- **Descripción**: Endpoint de devolución para admin.
- **Subtareas**:
  - [ ] `PaymentService.refund()` usando Stripe Refund API
  - [ ] Actualizar estado del pedido a `CANCELADO` o `DEVOLUCION_PARCIAL`
  - [ ] Tests
- **Criterio de aceptación**: Solo un ADMIN puede iniciar una devolución.

### TASK-015 — Email de confirmación de pedido
- **Estado**: `TODO`
- **Descripción**: Envío de email al completar un pedido.
- **Subtareas**:
  - [ ] Template de email HTML con resumen del pedido
  - [ ] `NotificationService.sendOrderConfirmation()` llamado tras pago exitoso (evento Stripe)
  - [ ] Tests con mock de JavaMailSender
- **Criterio de aceptación**: El email se envía dentro de 2 min del pago exitoso.

---

## Épica 6 — Panel de administración (Frontend)

### TASK-016 — Dashboard y rutas de admin
- **Estado**: `TODO`
- **Descripción**: Estructura de rutas y layout del panel admin en React.
- **Subtareas**:
  - [ ] Ruta protegida `AdminRoute` (redirige a login si rol ≠ ADMIN)
  - [ ] Layout de admin con navegación lateral
  - [ ] Tests de componente

### TASK-017 — Gestión de productos (admin UI)
- **Estado**: `TODO`
- **Descripción**: CRUD de productos desde el panel de administración.
- **Subtareas**:
  - [ ] Tabla de productos con paginación
  - [ ] Formulario crear/editar con React Hook Form + Zod
  - [ ] Upload de imágenes con preview
  - [ ] Tests

### TASK-018 — Gestión de pedidos (admin UI)
- **Estado**: `TODO`
- **Descripción**: Listado y gestión de estado de pedidos.
- **Subtareas**:
  - [ ] Tabla de todos los pedidos con filtros por estado
  - [ ] Detalle de pedido con líneas
  - [ ] Cambio de estado (select + botón confirmar)
  - [ ] Botón de devolución
  - [ ] Tests

---

## Épica 7 — Frontend público (tienda)

### TASK-019 — Catálogo público
- **Estado**: `TODO`
- **Descripción**: Página de listado de productos con filtros y búsqueda.
- **Subtareas**:
  - [ ] Componente `ProductGrid` con paginación
  - [ ] Sidebar de filtros (categoría, precio, disponibilidad)
  - [ ] Barra de búsqueda
  - [ ] Página de detalle de producto
  - [ ] Tests

### TASK-020 — Carrito y checkout UI
- **Estado**: `TODO`
- **Descripción**: Interfaz de carrito y flujo de checkout.
- **Subtareas**:
  - [ ] Panel de carrito (drawer lateral)
  - [ ] Página de checkout: dirección, resumen de pedido, formulario de pago Stripe
  - [ ] Página de confirmación de pedido
  - [ ] Tests

### TASK-021 — Historial de pedidos (usuario)
- **Estado**: `TODO`
- **Descripción**: Página de historial de pedidos del usuario.
- **Subtareas**:
  - [ ] Listado de pedidos con estado visual (badge)
  - [ ] Detalle de cada pedido con líneas de producto
  - [ ] Tests

---

## Épica 8 — Calidad y cierre

### TASK-022 — Cobertura de tests al 80 %
- **Estado**: `TODO`
- **Descripción**: Asegurar cobertura mínima en backend y frontend.
- **Subtareas**:
  - [ ] Configurar JaCoCo en backend con umbral 80 %
  - [ ] Configurar Vitest coverage con umbral 80 %
  - [ ] Corregir gaps de cobertura identificados

### TASK-023 — CI/CD pipeline
- **Estado**: `TODO`
- **Descripción**: Pipeline de integración continua.
- **Subtareas**:
  - [ ] GitHub Actions workflow: build + test backend (H2 en memoria)
  - [ ] GitHub Actions workflow: lint + test frontend
  - [ ] Bloquear merge si checks no pasan

### TASK-024 — Documentación de API
- **Estado**: `TODO`
- **Descripción**: Generar documentación interactiva de la API REST.
- **Subtareas**:
  - [ ] Integrar SpringDoc OpenAPI 3 (Swagger UI en `/api/docs`)
  - [ ] Anotar controllers con `@Operation` y `@ApiResponse`
  - [ ] Verificar que todos los endpoints están documentados
