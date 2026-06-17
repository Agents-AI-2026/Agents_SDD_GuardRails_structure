# Especificación — Autenticación y Login

> Versión: v2
> Fecha creación: 2026-06-16
> Fecha última modificación: 2026-06-17
> Estado: BORRADOR
> Autor: equipo
> Plan de referencia: `.specify/plans/login.md`

---

## 1. Descripción general

El módulo de **Autenticación** gestiona el registro de nuevos usuarios, login seguro, recuperación de contraseña, logout y **autenticación social** mediante proveedores de identidad externos (Google, Microsoft, Apple).
Proporciona a los clientes y administradores un acceso controlado a la plataforma mediante:
- Credenciales propias (email + contraseña)
- Proveedores de identidad externos (Google, Microsoft, Apple)

Requisitos de seguridad obligatorios:
- Las contraseñas se almacenan de forma segura y nunca en texto plano
- Las sesiones tienen una duración limitada y expiran automáticamente
- La sesión activa se mantiene de forma segura, inaccesible desde scripts de la página
- Todas las entradas se validan para prevenir ataques de inyección
- Los flujos de autenticación social incluyen protección contra ataques de falsificación de solicitudes

---

## 2. Flujos funcionales

### 2.1 Registro de usuario

**Actor:** Usuario anónimo  
**Entrada:** Email, contraseña (mínimo 8 caracteres), nombre completo  
**Salida:** Usuario creado, sesión iniciada, email de confirmación enviado  

**Pasos:**
1. Usuario introduce email, contraseña y nombre completo
2. El sistema valida que el email tiene formato correcto y no está registrado
3. El sistema valida que la contraseña cumple los requisitos mínimos
4. El sistema crea la cuenta con rol `CUSTOMER`
5. El usuario recibe un email de confirmación
6. El usuario queda autenticado y accede a su área personal

**Restricciones:**
- Email duplicado → error indicando que ya existe una cuenta con ese email
- Contraseña < 8 caracteres → error de validación
- Email inválido → error de validación
- Fallo al enviar email → no bloquea el registro; se reintenta en segundo plano
- Máximo 5 intentos de registro fallidos en 1 hora desde la misma IP → bloqueo temporal

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

**Actor:** Usuario autenticado con sesión próxima a expirar  
**Entrada:** Sesión activa  
**Salida:** Sesión renovada  

**Pasos:**
1. El sistema detecta que la sesión de corta duración ha expirado
2. El sistema renueva la sesión automáticamente usando la sesión de larga duración, sin interrumpir al usuario
3. Si la sesión de larga duración también ha expirado, el usuario es redirigido al login

**Restricciones:**
- La sesión de larga duración no se renueva automáticamente; solo se genera al hacer login
- Si la sesión de larga duración expiró, el usuario debe autenticarse nuevamente

---

### 2.4 Logout

**Actor:** Usuario autenticado  
**Entrada:** Sesión activa  
**Salida:** Sesión cerrada  

**Pasos:**
1. Usuario hace clic en “Cerrar sesión”
2. El sistema invalida la sesión activa de forma inmediata
3. El usuario es redirigido a la página de inicio
4. Cualquier intento posterior de acceder a recursos protegidos con la sesión cerrada es rechazado

**Restricciones:**
- La sesión se invalida de inmediato, no al expirar
- El cierre de sesión funciona aunque no haya conectividad (el cliente elimina los datos locales de sesión)

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
- [ ] Usuario registrado con email válido queda autenticado y accede a su área personal
- [ ] Email duplicado rechazado con mensaje de error claro
- [ ] Contraseña con menos de 8 caracteres rechazada con error de validación
- [ ] Email de confirmación enviado en menos de 2 minutos
- [ ] Máximo 5 intentos fallidos por IP en 1 hora → bloqueo temporal

### Login
- [ ] Login exitoso da acceso al área personal del usuario
- [ ] Credenciales inválidas muestran error genérico sin revelar si el email existe
- [ ] Tras varios fallos consecutivos, la cuenta queda bloqueada temporalmente
- [ ] La sesión de corta duración se renueva automáticamente mientras el usuario está activo
- [ ] La sesión de larga duración expira y obliga a volver a hacer login

### Renovación de sesión
- [ ] La sesión expirada no permite acceso sin una nueva renovación o login
- [ ] La renovación no interrumpe la experiencia del usuario si la sesión larga sigue activa

### Cierre de sesión
- [ ] El cierre de sesión invalida el acceso de forma inmediata
- [ ] Intentos de acceso post-logout son rechazados

### Recuperación de contraseña
- [ ] Email de recuperación enviado en menos de 2 minutos
- [ ] El enlace de recuperación expira tras un tiempo limitado
- [ ] El restablecimiento cierra todas las sesiones activas del usuario
- [ ] No se pueden solicitar resets en ráfaga (protección contra abuso)

### Inicio de sesión con proveedores externos
- [ ] El usuario puede iniciar sesión con Google, Microsoft y Apple
- [ ] Un email no registrado con proveedores externos crea una cuenta nueva automáticamente
- [ ] Un intento de login con un email ya registrado con otro proveedor es rechazado con mensaje claro
- [ ] El cierre de sesión invalida también la sesión iniciada con proveedor externo

---

## 4. Casos edge

| Caso | Comportamiento esperado |
|------|------------------------|
| Usuario intenta registrar con email de otro usuario | Error indicando que ya existe una cuenta con ese email |
| Usuario hace login antes de confirmar email | Login permitido (la confirmación de email es opcional) |
| Sesión larga expirada (navegador sin uso prolongado) | El usuario es redirigido al login |
| Usuario en cuenta bloqueada intenta login | Mensaje claro indicando que la cuenta está bloqueada temporalmente |
| Email de recuperación no llega (error SMTP) | El usuario puede reintentar pasado un tiempo; el fallo se registra para auditoría |
| Enlace de recuperación ya usado | Error claro indicando que el enlace ya fue utilizado |
| Usuario intenta iniciar sesión con proveedor externo usando email ya registrado con otro proveedor | Error claro con indicación del método original |
| Usuario cancela la autorización en el proveedor externo | El sistema redirige al login sin error, permitiendo reintentar |
| El proceso de autenticación con proveedor externo expira por inactividad | El usuario es redirigido al login con mensaje para intentar de nuevo |

---

## 5. No incluido (alcance v1)

- [ ] Autenticación de dos factores (2FA)
- [ ] Captcha en formularios
- [ ] Single Sign-On (SSO) empresarial
- [ ] Otros proveedores de identidad externos (GitHub, LinkedIn, etc.)
- [ ] Vinculación manual de proveedor externo a cuenta existente
- [ ] Revocación de acceso al proveedor externo desde la app

---
## Changelog

| Versi�n | Fecha | Descripci�n del cambio |
|---------|-------|------------------------|
| v1 | 2026-06-16 | Creación inicial |
| v2 | 2026-06-17 | Eliminado contenido técnico (algoritmos, endpoints HTTP, flags, tokens internos, tabla de proveedores OIDC); spec refactorizada para contener solo contenido funcional |