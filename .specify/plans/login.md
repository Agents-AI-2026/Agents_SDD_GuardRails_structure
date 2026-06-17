# Plan técnico — Autenticación y Login

> Versión: v1
> Fecha creación: 2026-06-17
> Fecha última modificación: 2026-06-17
> Spec de referencia: `.specify/specs/login.md`
> Estado: BORRADOR

---

## 1. Stack tecnológico

### Backend
| Librería | Versión | Propósito |
|----------|---------|-----------|
| Spring Security | 6.x | Autenticación, autorización y filtros HTTP |
| jjwt (io.jsonwebtoken) | 0.12.x | Generación y validación de JWT |
| BCrypt (Spring Security) | — | Hash de contraseñas, factor de coste 12 |
| Spring Mail + Thymeleaf | 3.3.x | Envío de emails HTML (confirmación, reset) |
| Nimbus JOSE + JWT | 9.x | Validación de ID tokens OAuth (JWKS) |
| Spring OAuth2 Client | 6.x | Gestión del flujo Authorization Code + PKCE |
| Bucket4j | 8.x | Rate limiting (intentos fallidos, resets) |

### Frontend
| Librería | Versión | Propósito |
|----------|---------|-----------|
| React Hook Form + Zod | 7.x / 3.x | Formularios de login, registro y reset |
| jose | 5.x | Generación de `code_verifier` / `code_challenge` para PKCE |
| Redux Toolkit (authSlice) | 2.x | Estado de sesión (access token en memoria) |

---

## 2. Modelo de datos

```
User
  id              BIGINT PK AUTO_INCREMENT
  email           VARCHAR(255) UNIQUE NOT NULL
  password_hash   VARCHAR(255)            -- NULL para usuarios solo OAuth
  full_name       VARCHAR(200) NOT NULL
  role            ENUM('CUSTOMER','ADMIN') DEFAULT 'CUSTOMER'
  status          ENUM('ACTIVE','BLOCKED') DEFAULT 'ACTIVE'
  email_verified  BOOLEAN DEFAULT FALSE
  failed_attempts SMALLINT DEFAULT 0      -- intentos fallidos de login
  blocked_until   TIMESTAMP               -- NULL si no está bloqueado
  created_at      TIMESTAMP NOT NULL
  updated_at      TIMESTAMP NOT NULL

RefreshToken
  id              BIGINT PK AUTO_INCREMENT
  user_id         BIGINT NOT NULL FK → User ON DELETE CASCADE
  token_hash      VARCHAR(255) UNIQUE NOT NULL  -- hash SHA-256 del token (nunca el raw)
  expires_at      TIMESTAMP NOT NULL
  revoked         BOOLEAN DEFAULT FALSE
  created_at      TIMESTAMP NOT NULL

PasswordResetToken
  id              BIGINT PK AUTO_INCREMENT
  user_id         BIGINT NOT NULL FK → User ON DELETE CASCADE
  token_hash      VARCHAR(255) UNIQUE NOT NULL  -- hash SHA-256
  expires_at      TIMESTAMP NOT NULL             -- creado_at + 1 hora
  used            BOOLEAN DEFAULT FALSE
  created_at      TIMESTAMP NOT NULL

OAuthIdentity
  id              BIGINT PK AUTO_INCREMENT
  user_id         BIGINT NOT NULL FK → User ON DELETE CASCADE
  provider        ENUM('GOOGLE','MICROSOFT','APPLE') NOT NULL
  sub             VARCHAR(255) NOT NULL           -- identificador único del proveedor
  email           VARCHAR(255) NOT NULL
  name            VARCHAR(200)
  picture_url     VARCHAR(500)
  created_at      TIMESTAMP NOT NULL
  updated_at      TIMESTAMP NOT NULL
  UNIQUE (provider, sub)
```

### Migraciones Flyway

| Versión | Descripción |
|---------|-------------|
| `V3__create_users.sql` | Tabla `User` (ya incluida en plan general) |
| `V7__create_refresh_tokens.sql` | Tabla `RefreshToken` |
| `V9__create_password_reset_tokens.sql` | Tabla `PasswordResetToken` |
| `V10__create_oauth_identities.sql` | Tabla `OAuthIdentity` |

---

## 3. Arquitectura de tokens

```
┌──────────────┐   POST /api/auth/login   ┌──────────────────────┐
│   Frontend   │ ─────────────────────── ▶│   AuthController     │
│  (React SPA) │                          │   AuthService        │
│              │ ◀─────────────────────── │                      │
│  access_token│   body: { access_token } │  BCryptPasswordEncoder│
│  en memoria  │   cookie: refresh_token  │  JwtTokenProvider    │
│  (Redux)     │   (HttpOnly, Secure,     │  RefreshTokenRepo    │
│              │    SameSite=Strict)       └──────────────────────┘
└──────────────┘

Expiración:
  access_token  → 15 minutos  (JWT firmado con clave HMAC-SHA-256)
  refresh_token → 7 días      (opaque token; hash guardado en BD)

Rotación: el refresh_token NO se rota en cada uso.
Se invalida completamente al hacer logout o reset de contraseña.
```

---

## 4. Contratos de API

### Base URL: `/api/v1/auth`

#### POST `/register`
**Acceso:** Público  
**Request:**
```json
{
  "email": "usuario@ejemplo.com",
  "password": "MiContraseña123",
  "fullName": "Juan García"
}
```
**Responses:**
| Código | Descripción |
|--------|-------------|
| `201 Created` | Usuario creado; body con `access_token` |
| `400 Bad Request` | Validación fallida (email, password, nombre) |
| `409 Conflict` | Email ya registrado |
| `429 Too Many Requests` | Rate limit IP: 5 intentos/hora |

---

#### POST `/login`
**Acceso:** Público  
**Request:**
```json
{ "email": "usuario@ejemplo.com", "password": "MiContraseña123" }
```
**Responses:**
| Código | Descripción |
|--------|-------------|
| `200 OK` | `{ "access_token": "...", "token_type": "Bearer", "expires_in": 900 }` + cookie `refresh_token` (HttpOnly, Secure, SameSite=Strict, Max-Age=604800) |
| `401 Unauthorized` | Credenciales inválidas (mensaje genérico) |
| `429 Too Many Requests` | Cuenta bloqueada temporalmente (header `Retry-After`) |

---

#### POST `/refresh`
**Acceso:** Público (refresh token en cookie)  
**Request:** Sin body; refresh token llega automáticamente en la cookie `refresh_token`  
**Responses:**
| Código | Descripción |
|--------|-------------|
| `200 OK` | `{ "access_token": "...", "expires_in": 900 }` |
| `401 Unauthorized` | Token expirado, revocado o inválido |

---

#### POST `/logout`
**Acceso:** Autenticado (`Authorization: Bearer <access_token>`)  
**Request:** Sin body  
**Responses:**
| Código | Descripción |
|--------|-------------|
| `204 No Content` | Refresh token revocado; cookie eliminada (`Max-Age=0`) |
| `401 Unauthorized` | Token inválido |

---

#### POST `/forgot-password`
**Acceso:** Público  
**Request:**
```json
{ "email": "usuario@ejemplo.com" }
```
**Responses:**
| Código | Descripción |
|--------|-------------|
| `200 OK` | Siempre el mismo mensaje: `{ "message": "Si el email existe, recibirás instrucciones" }` |
| `429 Too Many Requests` | Rate limit: 1 solicitud cada 5 minutos por usuario |

---

#### POST `/reset-password`
**Acceso:** Público  
**Request:**
```json
{ "token": "<reset_token>", "newPassword": "NuevaContraseña456" }
```
**Responses:**
| Código | Descripción |
|--------|-------------|
| `200 OK` | Contraseña actualizada; todos los refresh tokens revocados |
| `400 Bad Request` | Token expirado, ya usado o nueva contraseña inválida |

---

#### POST `/oauth/callback`
**Acceso:** Público  
**Request:**
```json
{
  "provider": "GOOGLE",
  "authCode": "<authorization_code>",
  "codeVerifier": "<pkce_code_verifier>",
  "state": "<state_value>"
}
```
**Responses:**
| Código | Descripción |
|--------|-------------|
| `200 OK` | `{ "access_token": "..." }` + cookie `refresh_token` |
| `400 Bad Request` | Estado inválido, code_verifier incorrecto, email no verificado |
| `409 Conflict` | Email ya registrado con otro proveedor o método |

---

## 5. Flujo OAuth 2.0 + PKCE — Detalle de implementación

```
Frontend                           Backend                       Proveedor (Google/MS/Apple)
   │                                  │                                    │
   │ 1. Genera code_verifier (random) │                                    │
   │    code_challenge = base64url(   │                                    │
   │      sha256(code_verifier))      │                                    │
   │    state = random (5 min TTL)    │                                    │
   │    Guarda en sessionStorage      │                                    │
   │                                  │                                    │
   │ 2. Redirige a proveedor ──────────────────────────────────────────────▶│
   │    ?code_challenge=...           │                                    │
   │    &code_challenge_method=S256   │                                    │
   │    &state=...                    │                                    │
   │                                  │                                    │
   │ 3. Usuario autoriza en proveedor │                                    │
   │ ◀────────────────────────────────────────────────────────────────────│
   │    callback: ?code=...&state=... │                                    │
   │                                  │                                    │
   │ 4. POST /oauth/callback ─────────▶                                    │
   │    { authCode, codeVerifier,     │                                    │
   │      state, provider }           │                                    │
   │                                  │ 5. POST /token ────────────────────▶│
   │                                  │    { code, code_verifier,          │
   │                                  │      client_secret, redirect_uri } │
   │                                  │ ◀──────────────────────────────────│
   │                                  │    { id_token, access_token }      │
   │                                  │                                    │
   │                                  │ 6. Valida id_token contra JWKS     │
   │                                  │    del proveedor:                  │
   │                                  │    - Firma RS256                   │
   │                                  │    - aud = client_id               │
   │                                  │    - exp no caducado (< 5 min)     │
   │                                  │    - email_verified = true         │
   │                                  │                                    │
   │                                  │ 7. Busca/crea User + OAuthIdentity │
   │                                  │                                    │
   │ ◀────────────────────────────────│ 8. { access_token } + cookie       │
```

### Configuración de proveedores

| Proveedor | OIDC Discovery Endpoint | Scopes |
|-----------|------------------------|--------|
| **Google** | `https://accounts.google.com/.well-known/openid-configuration` | `openid email profile` |
| **Microsoft** | `https://login.microsoftonline.com/common/v2.0/.well-known/openid-configuration` | `openid email profile` |
| **Apple** | `https://appleid.apple.com/.well-known/openid-configuration` | `openid email name` |

**Variables de entorno requeridas (por proveedor):**
```
OAUTH_GOOGLE_CLIENT_ID=
OAUTH_GOOGLE_CLIENT_SECRET=
OAUTH_MICROSOFT_CLIENT_ID=
OAUTH_MICROSOFT_CLIENT_SECRET=
OAUTH_APPLE_CLIENT_ID=
OAUTH_APPLE_TEAM_ID=
OAUTH_APPLE_KEY_ID=
OAUTH_APPLE_PRIVATE_KEY=        # p8 key en base64
```

---

## 6. Estructura de ficheros

```
auth/
├── controller/
│   └── AuthController.java             # Endpoints /api/v1/auth/**
├── service/
│   ├── AuthService.java                # Orquesta registro, login, logout
│   ├── TokenService.java               # JWT + refresh token lifecycle
│   ├── PasswordResetService.java       # Flujo forgot/reset password
│   └── OAuthService.java               # Flujo OAuth + PKCE
├── repository/
│   ├── UserRepository.java
│   ├── RefreshTokenRepository.java
│   ├── PasswordResetTokenRepository.java
│   └── OAuthIdentityRepository.java
├── domain/
│   ├── User.java
│   ├── RefreshToken.java
│   ├── PasswordResetToken.java
│   └── OAuthIdentity.java
├── dto/
│   ├── RegisterRequest.java
│   ├── LoginRequest.java
│   ├── AuthResponse.java
│   ├── ForgotPasswordRequest.java
│   ├── ResetPasswordRequest.java
│   └── OAuthCallbackRequest.java
├── security/
│   ├── JwtAuthenticationFilter.java    # Extrae y valida JWT de cada request
│   ├── JwtTokenProvider.java           # Genera y valida JWT
│   └── SecurityConfig.java            # Spring Security config, CORS, CSRF
└── exception/
    ├── InvalidTokenException.java
    ├── AccountBlockedException.java
    └── OAuthProviderException.java
```

---

## 7. Configuración de seguridad (SecurityConfig)

```java
// Rutas públicas
.requestMatchers("/api/v1/auth/**").permitAll()
.requestMatchers(HttpMethod.GET, "/api/v1/products/**").permitAll()
.requestMatchers(HttpMethod.GET, "/api/v1/categories/**").permitAll()

// Rutas de admin
.requestMatchers("/api/v1/users/admin/**").hasRole("ADMIN")
.requestMatchers(HttpMethod.POST, "/api/v1/products/**").hasRole("ADMIN")

// Todo lo demás requiere autenticación
.anyRequest().authenticated()

// Cookie config para refresh token
ResponseCookie.from("refresh_token", token)
  .httpOnly(true)
  .secure(true)
  .sameSite("Strict")
  .maxAge(Duration.ofDays(7))
  .path("/api/v1/auth/refresh")  // solo se envía en el endpoint de refresh
  .build();
```

---

## 8. Decisiones técnicas

| # | Decisión | Alternativas | Razón |
|---|----------|--------------|-------|
| 1 | Refresh token opaque (hash en BD) | JWT stateless | Permite revocación inmediata (logout, reset, compromiso) |
| 2 | Access token en memoria del cliente (Redux) | localStorage | Mitiga XSS: no persiste entre pestañas ni en storage |
| 3 | Refresh token en cookie HttpOnly SameSite=Strict | localStorage / memory | Mitiga XSS; SameSite=Strict elimina CSRF sin necesitar token CSRF adicional |
| 4 | BCrypt factor 12 | Argon2, scrypt | Spring Security lo incluye nativo; factor 12 ≈ 300ms (balance seguridad/UX) |
| 5 | PKCE obligatorio en OAuth (S256) | Flujo implícito | Mitiga interceptación de auth_code; requerido por OAuth 2.1 |
| 6 | `path` del cookie restringido a `/api/v1/auth/refresh` | Path raíz | El cookie solo se transmite cuando es necesario, reduciendo superficie de ataque |
| 7 | Rate limiting con Bucket4j en memoria | Redis, DB | Suficiente para instancia única; migrar a Redis en escala horizontal |

---

## 9. Variables de entorno

```
# JWT
JWT_SECRET=<clave-hmac-minimo-256bits>
JWT_ACCESS_EXPIRATION_MINUTES=15
JWT_REFRESH_EXPIRATION_DAYS=7

# OAuth
OAUTH_REDIRECT_URI=https://app.tiendamuebles.com/auth/callback
OAUTH_GOOGLE_CLIENT_ID=
OAUTH_GOOGLE_CLIENT_SECRET=
OAUTH_MICROSOFT_CLIENT_ID=
OAUTH_MICROSOFT_CLIENT_SECRET=
OAUTH_APPLE_CLIENT_ID=
OAUTH_APPLE_TEAM_ID=
OAUTH_APPLE_KEY_ID=
OAUTH_APPLE_PRIVATE_KEY=

# Email
MAIL_HOST=
MAIL_PORT=587
MAIL_USERNAME=
MAIL_PASSWORD=
MAIL_FROM=noreply@tiendamuebles.com

# App
APP_BASE_URL=https://app.tiendamuebles.com
```

---
## Changelog

| Versión | Fecha | Descripción del cambio |
|---------|-------|------------------------|
| v1 | 2026-06-17 | Creación inicial — plan técnico completo de autenticación (JWT, OAuth PKCE, modelo de datos, APIs, decisiones) |
