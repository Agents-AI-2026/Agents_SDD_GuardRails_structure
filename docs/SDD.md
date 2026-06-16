# Software Design Document (SDD)

> Versión: 0.1 — Fecha: 2026-06-16
> Estado: BORRADOR

## 1. Resumen ejecutivo

**Tienda de Muebles Online** es un sistema de e-commerce especializado en la venta de muebles para el hogar y la oficina. Permite a los clientes explorar un catálogo de productos con filtros avanzados, gestionar un carrito de compra persistente, realizar pedidos y pagar online mediante tarjeta de crédito/débito a través de Stripe.

El sistema está dirigido a consumidores finales que buscan muebles de calidad desde cualquier dispositivo, y a los administradores del negocio que necesitan gestionar el catálogo, el inventario y los pedidos desde un panel dedicado.

La plataforma se construye con una API REST en Spring Boot (Java 17) como backend y React + TypeScript en el frontend, garantizando una separación clara de responsabilidades, seguridad desde el diseño y capacidad de evolución hacia una arquitectura de mayor escala.

## 2. Objetivos

- [ ] Ofrecer un catálogo de muebles navegable con filtros por categoría, precio y disponibilidad
- [ ] Implementar un flujo de compra seguro y sin fricciones (carrito → checkout → pago con Stripe)
- [ ] Proporcionar un panel de administración para gestionar productos, stock, pedidos y usuarios
- [ ] Garantizar la seguridad de los datos de pago (PCI DSS — datos de tarjeta nunca tocan el backend)
- [ ] Cumplir con los requisitos de rendimiento: p95 < 300 ms en condiciones normales

## 3. Alcance

### Incluido
- Catálogo público de productos con búsqueda y filtros
- Autenticación JWT con roles CUSTOMER y ADMIN
- Carrito de compra persistente
- Checkout y pago online con Stripe
- Historial de pedidos del usuario
- Panel de administración (productos, categorías, pedidos, usuarios)
- Almacenamiento de imágenes en S3/MinIO
- Notificaciones por email (confirmación de pedido, recuperación de contraseña)

### Excluido (v1.0)
- App móvil nativa
- Sistema de reseñas y valoraciones
- Programa de fidelización / puntos
- Chat en vivo con soporte
- Comparador de productos
- Multi-tienda / multi-tenant

## 4. Arquitectura del sistema

```
┌─────────────────────────────────────────────────────────┐
│                      Internet                           │
└──────────────────────┬──────────────────────────────────┘
                       │ HTTPS
              ┌────────▼────────┐
              │  Reverse Proxy  │  (Nginx / AWS ALB)
              └────────┬────────┘
           ┌───────────┴───────────┐
           │                       │
  ┌────────▼────────┐     ┌────────▼────────┐
  │   Frontend      │     │   Backend API   │
  │  React / Vite   │◄────►  Spring Boot    │
  │  (puerto 3000)  │     │  (puerto 8080)  │
  └─────────────────┘     └────────┬────────┘
                          ┌────────┼────────────┐
                          │        │             │
               ┌──────────▼─┐  ┌───▼────┐  ┌───▼───┐
               │ PostgreSQL │  │  S3/   │  │ SMTP  │
               │   (16)     │  │ MinIO  │  │ server│
               └────────────┘  └────────┘  └───────┘
                                   │
               ┌───────────────────▼─────────────────┐
               │             Stripe API               │
               │  (pagos externos — PCI DSS)          │
               └─────────────────────────────────────┘
```

### Componentes principales

| Componente | Responsabilidad | Tecnología |
|------------|-----------------|------------|
| Frontend SPA | Interfaz de usuario pública y panel admin | React 18 + TypeScript + Vite |
| Backend API | Lógica de negocio, persistencia, seguridad | Spring Boot 3.3 + Java 17 |
| Base de datos | Persistencia relacional | PostgreSQL 16 |
| Almacenamiento de imágenes | Upload y servicio de imágenes de productos | AWS S3 / MinIO |
| Pasarela de pago | Procesamiento de pagos con tarjeta | Stripe |
| Servidor de email | Envío de notificaciones transaccionales | SMTP (Spring Mail) |

## 5. Stack tecnológico

| Capa | Tecnología | Versión |
|------|------------|---------|
| Lenguaje backend | Java | 17 |
| Framework backend | Spring Boot | 3.3.x |
| Seguridad | Spring Security | 6.x |
| Persistencia | Spring Data JPA + Hibernate | 3.3.x |
| Base de datos | PostgreSQL | 16 |
| Migraciones BD | Flyway | 10.x |
| Mapeo DTO | MapStruct | 1.5.x |
| JWT | jjwt (io.jsonwebtoken) | 0.12.x |
| Pagos | Stripe Java SDK | 25.x |
| Almacenamiento | AWS SDK v2 / MinIO client | 2.x |
| Email | Spring Mail | 3.3.x |
| Lenguaje frontend | TypeScript | 5.x |
| Framework frontend | React | 18.x |
| Build tool | Vite | 5.x |
| Estado global | Redux Toolkit + RTK Query | 2.x |
| Formularios | React Hook Form + Zod | 7.x / 3.x |
| Estilos | Tailwind CSS + shadcn/ui | 3.x |
| Tests backend | JUnit 5 + Mockito + Testcontainers | incluido / 1.19.x |
| Tests frontend | Vitest + React Testing Library | latest |

## 6. Modelo de datos

Las entidades principales y sus relaciones se detallan en `.specify/plans/tienda-muebles-online.md` (sección 2).

Resumen de entidades:

| Entidad | Descripción |
|---------|-------------|
| `Category` | Árbol jerárquico de categorías de productos |
| `Product` | Mueble con precio, stock, imágenes y dimensiones |
| `User` | Usuario con roles CUSTOMER / ADMIN |
| `Address` | Direcciones de envío asociadas al usuario |
| `Cart` / `CartItem` | Carrito persistente con líneas de producto |
| `Order` / `OrderItem` | Pedido con snapshot de productos y estado del ciclo de vida |
| `RefreshToken` | Tokens de renovación de sesión almacenados en BD |

## 7. APIs e integraciones externas

| Integración | Propósito | Documentación |
|-------------|-----------|---------------|
| Stripe API | Procesamiento de pagos y devoluciones | https://stripe.com/docs/api |
| Stripe Webhooks | Notificación de eventos de pago | https://stripe.com/docs/webhooks |
| AWS S3 / MinIO | Almacenamiento de imágenes de productos | https://docs.aws.amazon.com/s3 |
| SMTP | Envío de emails transaccionales | Configurable vía `application.yml` |

El contrato completo de la API REST propia (endpoints, DTOs, códigos de respuesta) se documenta en `.specify/plans/tienda-muebles-online.md` (sección 3) y se expone como Swagger UI en `/api/docs` (SpringDoc OpenAPI).

## 8. Seguridad

| Control | Implementación |
|---------|----------------|
| Autenticación | JWT: access token (15 min) + refresh token (7 días, cookie HttpOnly) |
| Autorización | Spring Security con `@PreAuthorize`; roles CUSTOMER y ADMIN |
| Contraseñas | BCrypt con factor de coste 12 |
| CORS | Permitido solo desde el origen del frontend |
| HTTPS | Obligatorio en producción (TLS en reverse proxy) |
| PCI DSS | Datos de tarjeta 100 % gestionados por Stripe.js; nunca llegan al backend |
| Webhook Stripe | Validación obligatoria de firma `Stripe-Signature` |
| Uploads | Validación de tipo MIME y tamaño ≤ 5 MB |
| SQL Injection | JPA + prepared statements; prohibido `@NativeQuery` con concatenación |
| XSS | Sanitización de input con OWASP Java HTML Sanitizer antes de persistir |
| Secrets | Toda configuración sensible via variables de entorno; prohibido en código |

## 9. Rendimiento y escalabilidad

| Atributo | Objetivo |
|----------|----------|
| Latencia p95 | < 300 ms bajo carga normal |
| Disponibilidad | 99,5 % uptime mensual |
| Paginación | Obligatoria en todos los listados (page size máximo: 50) |
| Caché | Cache de categorías en memoria (TTL 10 min) con Spring Cache |
| Imágenes | Servidas directamente desde S3/MinIO (sin pasar por el backend) |
| Escalabilidad horizontal | Backend sin estado (stateless JWT) — preparado para múltiples réplicas |

## 10. Decisiones de arquitectura (ADRs)

| # | Decisión | Alternativas consideradas | Razón |
|---|----------|--------------------------|-------|
| 1 | Refresh token en cookie HttpOnly | localStorage | Mitigación XSS; localStorage expuesto a scripts |
| 2 | Flyway para migraciones | Liquibase | Simplicidad; integración nativa con Spring Boot |
| 3 | MapStruct para mapeo DTO | ModelMapper / manual | Type-safe en tiempo de compilación; sin reflection en runtime |
| 4 | Soft delete en productos | Delete físico | Historial de pedidos consistente; los OrderItems referencian productos históricos |
| 5 | `SELECT FOR UPDATE` para stock | Optimistic locking | Garantía fuerte frente a compras simultáneas del último item |
| 6 | Stripe Elements en frontend | Pasarela propia | PCI DSS: datos de tarjeta nunca tocan nuestros servidores |
| 7 | RTK Query para fetching | React Query | Integración natural con Redux; evita duplicar estado servidor/cliente |

## 11. Dependencias nuevas

Todas las dependencias están documentadas en `.specify/plans/tienda-muebles-online.md` (sección 4).

| Paquete | Versión | Propósito | Aprobado por |
|---------|---------|-----------|--------------|
| stripe-java | 25.x | SDK Java para pagos | pendiente |
| aws-sdk-java-v2 (s3) | 2.x | Upload de imágenes | pendiente |
| flyway-core | 10.x | Migraciones de BD | pendiente |
| mapstruct | 1.5.x | Mapeo entidad ↔ DTO | pendiente |
| jjwt-api / jjwt-impl | 0.12.x | JWT | pendiente |
| testcontainers-postgresql | 1.19.x | Tests de integración | pendiente |
| @stripe/react-stripe-js | latest | Formulario de pago seguro | pendiente |
| @reduxjs/toolkit | 2.x | Estado global + fetching | pendiente |
| react-hook-form | 7.x | Formularios | pendiente |
| zod | 3.x | Validación de esquemas | pendiente |
| tailwindcss | 3.x | Estilos utility-first | pendiente |
