# Especificación — Autenticación y Login

> Fecha: 2026-06-16
> Estado: BORRADOR
> Autor: equipo

---

## 1. Descripción general

El módulo de **Autenticación** gestiona el registro de nuevos usuarios, login seguro con JWT, recuperación de contraseña, logout y **autenticación social** mediante OAuth 2.0 (Google, Microsoft, Apple).
Proporciona a los clientes y administradores un acceso controlado a la plataforma mediante:
- Credenciales propias (email + contraseña hasheada)
- Proveedores de identidad externos (Google, Microsoft, Apple)

Requisitos de seguridad obligatorios:
- Contraseñas hasheadas con BCrypt (factor de coste 12) para login tradicional
- JWT con expiración (access token: 15 min, refresh token: 7 días)
- Refresh token almacenado en cookie HttpOnly (no accesible desde JavaScript)
- Validación de todas las entradas contra inyección XSS y SQL
- Email de confirmación y recuperación de contraseña
- OAuth 2.0 con validación de PKCE (Proof Key for Code Exchange)
- Validación de firma ID token (OpenID Connect)
- CSRF protection en flujos OAuth

---

## 2. Flujos funcionales

### 2.1 Registro de usuario

**Actor:** Usuario anónimo  
**Entrada:** Email, contraseña (mínimo 8 caracteres), nombre completo  
**Salida:** Usuario creado, tokens JWT, email de confirmación enviado  

**Pasos:**
1. Usuario envía POST `/api/auth/register` con email, password, nombre
2. Backend valida email (formato correcto, no duplicado)
3. Backend valida password (mínimo 8 caracteres, complejidad mínima)
4. Backend hasea contraseña con BCrypt (factor 12)
5. Backend crea usuario con rol `CUSTOMER`
6. Backend genera tokens JWT y refresh token
7. Backend almacena refresh token en BD con expiración (7 días)
8. Backend envía email de confirmación a la dirección registrada
9. Frontend recibe tokens y redirige a dashboard del usuario

**Restricciones:**
- Email duplicado → 409 Conflict
- Password < 8 caracteres → 400 Bad Request
- Email inválido → 400 Bad Request
- Fallo al enviar email → log de error, pero permite continuar
- Máximo 5 intentos de registro fallidos en 1 hora desde mismo IP → 429 Too Many Requests

---

### 2.2 Login

**Actor:** Usuario registrado  
**Entrada:** Email, contraseña  
**Salida:** Access token (JWT), refresh token (cookie HttpOnly)  

**Pasos:**
1. Usuario envía POST `/api/auth/login` con email + password
2. Backend busca usuario por email (case-insensitive)
3. Backend compara password recibido con hash almacenado (BCrypt)
4. Si coincide:
   - Genera new access token (15 min de expiración)
   - Genera new refresh token
   - Almacena refresh token en BD
   - Retorna access token en body JSON
   - Retorna refresh token en cookie HttpOnly (flags: Secure, SameSite=Strict)
5. Si NO coincide:
   - Incrementa contador de intentos fallidos del usuario
   - Si intentos ≥ 5 en 15 minutos → bloquea cuenta temporalmente (15 min)
   - Retorna 401 Unauthorized (mensaje genérico: "Credenciales inválidas")
6. Si usuario no existe:
   - Retorna 401 Unauthorized (no revelar si email existe)

**Restricciones:**
- No revelar si email existe (security: timing attack mitigation)
- Máximo 5 intentos fallidos en 15 minutos → bloqueo temporal
- Contraseña en texto plano solo durante transmisión HTTPS
- Log de todos los logins fallidos para auditoría

---

### 2.3 Refresh Token

**Actor:** Usuario autenticado con token expirado  
**Entrada:** Refresh token (cookie)  
**Salida:** Nuevo access token  

**Pasos:**
1. Frontend detecta que access token expiró (401 o jwt.io validation)
2. Frontend envía POST `/api/auth/refresh` (refresh token va automáticamente en cookie)
3. Backend valida refresh token:
   - Existe en BD
   - No ha expirado (< 7 días)
   - Firma JWT es válida
4. Si válido:
   - Genera nuevo access token (15 min)
   - Retorna en body JSON
   - NO renueva refresh token (refresh solo cada 7 días)
5. Si inválido:
   - Retorna 401 Unauthorized
   - Frontend redirige a login

**Restricciones:**
- Refresh token NO se renueva automáticamente (solo al login)
- Si refresh token expiró → usuario DEBE hacer login nuevamente
- La cookie debe estar presente (validación HttpOnly)

---

### 2.4 Logout

**Actor:** Usuario autenticado  
**Entrada:** Access token  
**Salida:** Confirmación de logout, token invalidado  

**Pasos:**
1. Usuario hace clic en "Logout"
2. Frontend envía POST `/api/auth/logout` con access token en header `Authorization: Bearer <token>`
3. Backend:
   - Valida access token (firma y expiración)
   - Busca el refresh token asociado y lo marca como `revoked` en BD
   - Limpia la cookie HttpOnly (Set-Cookie con Max-Age=0)
   - Retorna 200 OK
4. Frontend:
   - Borra el access token del localStorage/sessionStorage
   - Redirige a homepage
5. Usuario intenta acceder a recurso protegido → 401 Unauthorized

**Restricciones:**
- El token se revoca inmediatamente (no espera expiración)
- Logout debe ser posible sin internet (frontend lo maneja localmente)

---

### 2.5 Recuperación de contraseña

**Actor:** Usuario que olvidó contraseña  
**Entrada:** Email  
**Salida:** Email con enlace de reset  

**Pasos (request):**
1. Usuario envía POST `/api/auth/forgot-password` con email
2. Backend valida email existe
3. Si existe:
   - Genera token de reset (JWT con expiración 1 hora)
   - Almacena en BD asociado al usuario
   - Envía email con enlace: `https://app.tiendamuebles.com/reset-password?token=<jwt>&email=<email>`
   - Retorna 200 OK (mensaje: "Si el email existe, recibirá instrucciones")
4. Si NO existe:
   - Retorna 200 OK igual (no revelar si email existe)

**Pasos (reset):**
1. Usuario recibe email con enlace
2. Usuario hace clic → frontend muestra formulario de nueva contraseña
3. Usuario envía POST `/api/auth/reset-password` con token + nueva password
4. Backend valida:
   - Token existe en BD
   - Token no ha expirado (< 1 hora)
   - Nueva password ≠ anterior (complejidad)
5. Si válido:
   - Hashea nueva contraseña
   - Reemplaza contraseña antigua
   - Marca token como `used`
   - Retorna 200 OK
6. Si inválido:
   - Retorna 400 Bad Request (token expirado, inválido o ya usado)

**Restricciones:**
- Token de reset válido por 1 hora solamente
- No permitir reset de contraseña más de 1 vez cada 5 minutos (mismo usuario)
- Revocar todos los refresh tokens del usuario tras reset (fuerza nuevo login)

---

### 2.6 Login con proveedores OAuth (Google, Microsoft, Apple)

**Actor:** Usuario anónimo con cuenta en Google, Microsoft o Apple  
**Entrada:** Selección de proveedor (Google / Microsoft / Apple)  
**Salida:** Usuario autenticado, access token (JWT), refresh token (cookie HttpOnly)  

**Pasos (Flujo Authorization Code + PKCE):**

1. **Frontend inicia flow:**
   - Genera `code_verifier` (43-128 caracteres alfanuméricos)
   - Calcula `code_challenge = base64url(sha256(code_verifier))`
   - Redirige a proveedor: `https://provider.com/oauth/authorize?client_id=...&redirect_uri=...&scope=openid email profile&response_type=code&code_challenge=...&code_challenge_method=S256&state=<random>`
   - Almacena `state` en sessionStorage para validación CSRF
   - Almacena `code_verifier` en sessionStorage

2. **Usuario autoriza en proveedor:**
   - Proveedor muestra pantalla de consentimiento
   - Usuario autoriza acceso a email y perfil
   - Proveedor redirige a `https://app.tiendamuebles.com/auth/callback?code=<auth_code>&state=<state>`

3. **Frontend procesa callback:**
   - Valida `state` coincide con el almacenado (previene CSRF)
   - Extrae `auth_code`
   - Envía POST `/api/auth/oauth/callback` con `{ provider, auth_code, code_verifier }`

4. **Backend intercambia código por tokens:**
   - Valida `state` y `code_verifier`
   - Envía POST a proveedor: `{ client_id, client_secret, code, code_verifier, redirect_uri }`
   - Recibe ID token + access token del proveedor
   - Valida firma del ID token contra JWKS del proveedor
   - Extrae: `email`, `email_verified`, `name`, `picture` (sub del token)

5. **Backend busca/crea usuario:**
   - Busca usuario por email
   - Si NO existe:
     - Crea usuario nuevo con rol `CUSTOMER`
     - Vincula identidad OAuth: almacena `{ provider, sub, email, name, picture }` en tabla `OAuthIdentity`
     - Marca email como verificado (porque viene del proveedor)
   - Si existe:
     - Valida que `sub` coincida (previene account takeover)
     - Si `sub` no coincide → error 400 "Email ya registrado con otro método"
     - Si coincide → actualiza `picture` y `name` si han cambiado

6. **Backend genera tokens y responde:**
   - Genera new access token (15 min)
   - Genera new refresh token (7 días)
   - Almacena refresh token en BD
   - Retorna access token en body JSON
   - Retorna refresh token en cookie HttpOnly (Secure, SameSite=Strict)
   - Retorna 200 OK

**Restricciones:**
- PKCE obligatorio (S256 hash)
- `state` válido por máximo 5 minutos
- `auth_code` válido por máximo 10 minutos (standard OAuth)
- ID token debe estar validado y sin expirar (< 5 min después de emisión)
- Solo aceptar email_verified=true (proveedor confirma propiedad del email)
- `sub` debe ser único y inmutable por proveedor (previene spoofing)
- Vinculación entre usuario local y OAuth identity es de 1-a-1 (un usuario, un `sub` por proveedor)
- No permitir cambio de proveedor sin re-autenticación
- Log de todos los logins OAuth exitosos para auditoría

**Proveedores y configuración:**

| Proveedor | OIDC Endpoint | Scopes requeridos | Validación |
|-----------|--------------|------------------|------------|
| **Google** | `https://accounts.google.com/.well-known/openid-configuration` | `openid email profile` | JWKS de Google, validar `aud` = client_id |
| **Microsoft** | `https://login.microsoftonline.com/common/v2.0/.well-known/openid-configuration` | `openid email profile` | JWKS de Microsoft, validar `aud` = client_id |
| **Apple** | `https://appleid.apple.com/.well-known/openid-configuration` | `openid email name` | JWKS de Apple, validar `aud` = client_id, generar `team_id.client_id.key_id` |

**Tokens de proveedor vs tokens de la app:**
- Los ID tokens de los proveedores se validan pero **NO se guardan** en BD
- La app emite sus propios JWT (compatibles con el resto del sistema)
- El refresh token de la app es independiente (gestión interna)

---

## 3. Criterios de aceptación

### Registro
- [ ] Usuario registrado con email válido recibe token JWT
- [ ] Email duplicado rechazado con 409 Conflict
- [ ] Password < 8 caracteres rechazado con 400 Bad Request
- [ ] Contraseña hasheada con BCrypt (factor 12) en BD
- [ ] Email de confirmación enviado dentro de 2 minutos
- [ ] Máximo 5 registros por IP en 1 hora → 429 Too Many Requests

### Login
- [ ] Login exitoso retorna access token en body y refresh token en cookie
- [ ] Credenciales inválidas retorna 401 sin revelar si email existe
- [ ] Tras 5 fallos en 15 min, cuenta bloqueada temporalmente (15 min)
- [ ] Access token expira a los 15 minutos
- [ ] Refresh token expira a los 7 días
- [ ] Cookie HttpOnly tiene flags: Secure, SameSite=Strict

### Refresh Token
- [ ] Token expirado rechazado con 401
- [ ] Refresh genera nuevo access token sin renovar el refresh
- [ ] Refresh token NO reutilizable después de expiración (logout fuerza login)

### Logout
- [ ] Logout revoca refresh token inmediatamente
- [ ] Acceso post-logout con token antiguo retorna 401
- [ ] Cookie se borra del cliente (Max-Age=0)

### Recuperación de contraseña
- [ ] Email de reset enviado en < 2 minutos
- [ ] Token de reset válido por 1 hora
- [ ] Contraseña reseteada invalida todos los refresh tokens activos
- [ ] Mismo usuario no puede solicitar reset > 1 vez cada 5 minutos

### Login OAuth (Google, Microsoft, Apple)
- [ ] PKCE S256 implementado correctamente
- [ ] `state` validado para prevenir CSRF (expiración 5 min)
- [ ] ID token validado contra JWKS del proveedor
- [ ] Solo se aceptan ID tokens con `email_verified=true`
- [ ] Usuario nuevo creado automáticamente con email del proveedor
- [ ] Vinculación OAuth: `{ provider, sub, email }` almacenada de forma única
- [ ] Intento de login con email existente pero diferente `sub` → 400 error
- [ ] Access token y refresh token retornados correctamente (mismo que login tradicional)
- [ ] Logout invalida todos los tokens de la sesión OAuth
- [ ] Proveedor de tokens (Google/Microsoft/Apple) no accesible desde frontend

---

## 4. Casos edge

| Caso | Comportamiento esperado |
|------|------------------------|
| Usuario intenta registrar con email de otro usuario | 409 Conflict, mensaje: "Email ya registrado" |
| Usuario hace login antes de confirmar email | Login exitoso (confirmación de email es optional) |
| Token JWT malformado | 401 Unauthorized, mensaje: "Token inválido" |
| Cookie HttpOnly perdida (navegador limpiado) | Refresh falla con 401, usuario redirigido a login |
| Usuario intenta reset de contraseña 6 veces en 5 minutos | 6ª solicitud rechazada, retry posible después de 5 min |
| Token de reset ya usado | 400 Bad Request, mensaje: "Token ya fue utilizado" |
| Acceso a endpoint protegido sin Authorization header | 401 Unauthorized |
| Usuario en cuenta bloqueada intenta login | 429 Too Many Requests, mensaje: "Cuenta temporalmente bloqueada, intente en 15 min" |
| Email del servidor SMTP no funciona en reset | Log de error, usuario no recibe email pero puede reintentar |
| XSS en campo email | Input sanitizado antes de almacenar, output escapado en templates |
| CSRF en formulario de login | Token CSRF en cada POST (si aplica; con SPA + JWT + HttpOnly es menos crítico) |
| Timing attack en login | Ambas rutas (user existe / contraseña incorrecta) tardan ~200ms (constant time hash) |
| Usuario hace login OAuth cuando ya tiene cuenta local | 400 Bad Request: "Email ya registrado. Usa contraseña o vincula OAuth" |
| `state` expirado (> 5 min después de iniciar flow) | 400 Bad Request: "Sesión expirada, intenta nuevamente" |
| `auth_code` interceptado en URL | Código válido solo 1 vez; segundo intento falla (estándar OAuth) |
| `code_verifier` no coincide con `code_challenge` | 400 Bad Request desde proveedor, backend rechaza |
| Proveedor retorna `email_verified=false` | 400 Bad Request: "Por favor verifica tu email en [proveedor]" |
| Usuario autoriza en proveedor pero cancela en app | Frontend maneja navegación; sesión expira en 5 min |
| Mismo email registrado en dos proveedores (Google y Microsoft) | Dos usuarios locales diferentes, cada uno vinculado a su proveedor |
| Usuario intenta revocar acceso OAuth después de login | Posible solo desde panel de proveedores externos (Google/Microsoft/Apple); backend solo desvincula al logout |
| ID token expirado (> 5 min de emisión) | 400 Bad Request: "ID token expirado, intenta nuevamente" |
| XSS en `redirect_uri` | Frontend valida que sea la URL correcta; backend valida contra whitelist |
| Llamada directa a `/api/auth/oauth/callback` sin `code` | 400 Bad Request |
| Refresh token de OAuth se agotan tras 7 días | Usuario debe hacer login nuevamente (flujo OAuth completo) |

---

## 5. No incluido (v1.0)

- [ ] Autenticación de dos factores (2FA)
- [ ] Capcha en formularios
- [ ] Session de usuario (solo JWT)
- [ ] Single Sign-On (SSO)
- [ ] Otros proveedores OAuth (GitHub, LinkedIn, etc.)
- [ ] Account linking manual (vincular OAuth a cuenta existente sin crear nueva)
- [ ] Revocación de acceso OAuth desde la app (solo desde el proveedor)
