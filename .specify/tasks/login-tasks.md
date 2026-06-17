# Tareas — Autenticación y Login

> Fecha: 2026-06-17
> Plan de referencia: `.specify/plans/login.md`
> Spec de referencia: `.specify/specs/login.md`
> Estado: PENDIENTE

---

## Épica 1 — Infraestructura de autenticación

### TASK-L001 — Migraciones Flyway (auth)
- **Estado**: `TODO`
- **Descripción**: Crear las migraciones de BD necesarias para el módulo de autenticación.
- **Subtareas**:
  - [ ] `V7__create_refresh_tokens.sql` — tabla `RefreshToken` (token_hash, expires_at, revoked)
  - [ ] `V9__create_password_reset_tokens.sql` — tabla `PasswordResetToken` (token_hash, expires_at, used)
  - [ ] `V10__create_oauth_identities.sql` — tabla `OAuthIdentity` (provider, sub, UNIQUE constraint)
  - [ ] Verificar que `V3__create_users.sql` incluye `email_verified`, `failed_attempts`, `blocked_until`
- **Criterio de aceptación**: Flyway aplica todas las migraciones en orden sin errores en H2 (modo servidor y embebido).

### TASK-L002 — JWT y ciclo de vida de tokens
- **Estado**: `TODO`
- **Descripción**: Implementar `TokenService` y el filtro de autenticación JWT.
- **Subtareas**:
  - [ ] Entidades JPA: `RefreshToken`, `PasswordResetToken`
  - [ ] Repositorios: `RefreshTokenRepository`, `PasswordResetTokenRepository`
  - [ ] `TokenService`: generar y validar access JWT (HMAC-SHA-256, 15 min), generar opaque refresh token (7 días, hash SHA-256 en BD), revocar todos los tokens de un usuario
  - [ ] `JwtAuthenticationFilter`: extraer Bearer, validar JWT, cargar `UserDetails`
  - [ ] `SecurityConfig`: rutas públicas vs. protegidas; excluir `/api/v1/auth/**` de CSRF; endpoints de webhook con verificación propia
  - [ ] Tests unitarios de `TokenService` (generación, validación, expiración, revocación)
- **Criterio de aceptación**: Endpoint protegido devuelve 401 sin token; 200 con JWT válido. Token expirado devuelve 401.

### TASK-L003 — Rate limiting con Bucket4j
- **Estado**: `TODO`
- **Descripción**: Proteger endpoints críticos contra fuerza bruta y abuso.
- **Subtareas**:
  - [ ] Dependencia `bucket4j-spring-boot-starter` en `pom.xml`
  - [ ] Rate limit IP en `POST /login`: 10 intentos / 15 min
  - [ ] Rate limit cuenta en `POST /login`: bloquear `blocked_until` tras 5 fallos
  - [ ] Rate limit IP en `POST /register`: 5 registros / hora
  - [ ] Rate limit usuario en `POST /forgot-password`: 1 solicitud / 5 min
  - [ ] Header `Retry-After` en respuesta 429
  - [ ] Tests con mock de repositorio de contadores
- **Criterio de aceptación**: Después de 5 intentos fallidos de login, la cuenta queda bloqueada y devuelve 429 con `Retry-After`.

---

## Épica 2 — Autenticación local (email + contraseña)

### TASK-L004 — Registro de usuario (`POST /register`)
- **Estado**: `TODO`
- **Descripción**: Endpoint de registro con validaciones, hash de contraseña y email de verificación.
- **Subtareas**:
  - [ ] DTO `RegisterRequest` con validaciones Bean Validation (email válido, password ≥ 8 chars, fullName no vacío)
  - [ ] `AuthService.register()`: comprobar email duplicado (409), hashear con BCrypt factor 12, crear `User` con `email_verified=false`
  - [ ] Generar token de verificación de email y enviar con Spring Mail
  - [ ] `AuthController.register()`: devuelve `201 Created` con `access_token` + cookie `refresh_token`
  - [ ] Tests de integración con H2 en memoria
- **Criterio de aceptación**: Email duplicado → 409. Contraseña débil → 400. Registro exitoso → 201 con JWT.

### TASK-L005 — Login (`POST /login`)
- **Estado**: `TODO`
- **Descripción**: Endpoint de login con protección contra timing attacks y bloqueo de cuenta.
- **Subtareas**:
  - [ ] `AuthService.login()`: cargar usuario, verificar `status=ACTIVE`, comparar BCrypt (tiempo constante), incrementar `failed_attempts` en fallo, resetear en éxito
  - [ ] Lógica de bloqueo temporal: `failed_attempts >= 5` → `blocked_until = now + 15 min`
  - [ ] Devolver `access_token` en body + `refresh_token` en cookie HttpOnly, Secure, SameSite=Strict, Max-Age=604800
  - [ ] Mensaje de error genérico (no revelar si email existe o no)
  - [ ] Tests de integración
- **Criterio de aceptación**: Credenciales inválidas → 401 con mensaje genérico. Cuenta bloqueada → 429. Login correcto → 200 con JWT.

### TASK-L006 — Refresh y logout
- **Estado**: `TODO`
- **Descripción**: Renovar access token y revocar sesión.
- **Subtareas**:
  - [ ] `POST /refresh`: leer cookie `refresh_token`, buscar hash en BD, verificar no expirado/revocado, emitir nuevo `access_token`
  - [ ] `POST /logout`: marcar refresh token como `revoked=true`, vaciar cookie (`Max-Age=0`)
  - [ ] Tests unitarios
- **Criterio de aceptación**: Refresh con token expirado → 401. Después de logout, el token no puede renovarse.

### TASK-L007 — Recuperación de contraseña
- **Estado**: `TODO`
- **Descripción**: Flujo `forgot-password` / `reset-password`.
- **Subtareas**:
  - [ ] `POST /forgot-password`: generar UUID, hash SHA-256, guardar en `PasswordResetToken` (TTL 1 hora), enviar email con template Thymeleaf
  - [ ] Respuesta siempre `200 OK` con mensaje genérico (no revelar si el email existe)
  - [ ] `POST /reset-password`: validar token (no expirado, no usado), cambiar contraseña, marcar token como `used=true`, revocar todos los refresh tokens del usuario
  - [ ] Tests unitarios con mock de `JavaMailSender`
- **Criterio de aceptación**: Token expirado → 400. Token ya usado → 400. Reset exitoso → todos los refresh tokens revocados.

---

## Épica 3 — OAuth 2.0 + PKCE

### TASK-L008 — Backend OAuth (`POST /oauth/callback`)
- **Estado**: `TODO`
- **Descripción**: Implementar el endpoint de callback para Google, Microsoft y Apple.
- **Subtareas**:
  - [ ] Dependencias: `spring-oauth2-client`, `nimbus-jose-jwt`
  - [ ] Variables de entorno por proveedor (CLIENT_ID, CLIENT_SECRET, TEAM_ID/KEY_ID/PRIVATE_KEY para Apple)
  - [ ] `OAuthService.handleCallback()`:
    - [ ] Verificar `state` (TTL 5 min, almacenado en caché / Redis o sesión efímera)
    - [ ] Intercambiar `authCode` + `codeVerifier` por `id_token` en endpoint del proveedor
    - [ ] Validar `id_token`: firma RS256 contra JWKS, `aud=client_id`, `exp` no caducado, `email_verified=true`
    - [ ] Buscar `OAuthIdentity` por `(provider, sub)` → si existe, login; si no, crear `User` + `OAuthIdentity`
    - [ ] Manejar colisión de email (email ya registrado con otro proveedor) → 409
  - [ ] Tests con mock de proveedor JWKS
- **Criterio de aceptación**: `state` inválido → 400. `email_verified=false` → 400. Email colisionado → 409. Flujo completo → JWT + cookie.

### TASK-L009 — Frontend OAuth (botones y PKCE)
- **Estado**: `TODO`
- **Descripción**: Implementar los botones de OAuth y el flujo PKCE en el cliente.
- **Subtareas**:
  - [ ] Dependencia `jose` 5.x para `code_verifier` / `code_challenge`
  - [ ] `OAuthButton` component (Google, Microsoft, Apple) con estilos de marca
  - [ ] `useOAuth` hook: generar `code_verifier` (random 43-128 chars), `code_challenge = base64url(sha256(code_verifier))`, `state` random, guardar en `sessionStorage`
  - [ ] Redirigir al proveedor con `response_type=code`, `code_challenge_method=S256`
  - [ ] Página de callback: leer `code` + `state` de URL, llamar `POST /oauth/callback`
  - [ ] Tests de componente y hook
- **Criterio de aceptación**: El flujo completo desde botón hasta sesión iniciada funciona end-to-end en dev.

---

## Épica 4 — Frontend de autenticación

### TASK-L010 — Formularios de login y registro
- **Estado**: `TODO`
- **Descripción**: Pantallas de login y registro con React Hook Form + Zod.
- **Subtareas**:
  - [ ] `LoginPage`: email + password, validación inline, mensaje de error del servidor, enlace a registro y a forgot-password
  - [ ] `RegisterPage`: email, password, fullName, confirmPassword, validación Zod
  - [ ] `ForgotPasswordPage` + `ResetPasswordPage`
  - [ ] Accesibilidad: labels correctos, `aria-invalid`, `aria-describedby` para errores
  - [ ] Tests de componente con React Testing Library
- **Criterio de aceptación**: Formulario no puede enviarse con campos inválidos. Errores del servidor se muestran inline. ARIA sin violaciones.

### TASK-L011 — Redux `authSlice` y silent refresh
- **Estado**: `TODO`
- **Descripción**: Gestión del estado de sesión en memoria y renovación automática del access token.
- **Subtareas**:
  - [ ] `authSlice`: `{ accessToken, user, status }` — token en memoria (nunca en localStorage)
  - [ ] RTK Query `baseQuery` con `prepareHeaders` que inyecta `Authorization: Bearer <token>`
  - [ ] `reAuth` middleware: si respuesta 401 → llamar `POST /refresh` → reintentar petición original
  - [ ] `AuthGuard` component: redirige a login si no hay sesión activa
  - [ ] Tests unitarios del slice y middleware
- **Criterio de aceptación**: Recarga de página → silent refresh automático. Refresh token expirado → redirect a login.
