# Plan técnico — Tienda de Muebles Online

> Fecha: 2026-06-16
> Spec de referencia: `.specify/specs/tienda-muebles-online.md`
> Estado: BORRADOR

---

## 1. Estructura de repositorio

```
tienda-muebles/
├── backend/                   # Spring Boot — Java 17
│   ├── src/
│   │   ├── main/
│   │   │   ├── java/com/muebles/
│   │   │   │   ├── MueblesApplication.java
│   │   │   │   ├── config/          # SecurityConfig, CorsConfig, S3Config, StripeConfig
│   │   │   │   ├── domain/          # Entidades JPA
│   │   │   │   ├── repository/      # Interfaces JPA Repository
│   │   │   │   ├── service/         # Lógica de negocio
│   │   │   │   ├── controller/      # REST controllers
│   │   │   │   ├── dto/             # Request/Response DTOs
│   │   │   │   ├── mapper/          # MapStruct mappers
│   │   │   │   ├── exception/       # GlobalExceptionHandler + excepciones custom
│   │   │   │   └── util/            # Utilidades (validadores, constantes)
│   │   │   └── resources/
│   │   │       ├── application.yml
│   │   │       ├── application-dev.yml
│   │   │       └── db/migration/    # Flyway migrations
│   │   └── test/
│   │       └── java/com/muebles/    # Tests unitarios e integración
│   ├── pom.xml
│   └── Dockerfile
│
├── frontend/                  # React + TypeScript
│   ├── src/
│   │   ├── api/               # Clientes axios por dominio
│   │   ├── components/        # Componentes reutilizables
│   │   ├── pages/             # Páginas / rutas
│   │   ├── store/             # Redux Toolkit slices
│   │   ├── hooks/             # Custom hooks
│   │   ├── types/             # TypeScript interfaces/types
│   │   └── utils/             # Helpers
│   ├── public/
│   ├── package.json
│   ├── vite.config.ts
│   └── Dockerfile
│
├── docker-compose.yml         # H2 (modo servidor) + MinIO + backend + frontend
└── docs/
```

---

## 2. Modelo de datos

### Entidades principales

```
Category
  id            BIGINT PK
  name          VARCHAR(100) NOT NULL
  slug          VARCHAR(100) UNIQUE NOT NULL
  parent_id     BIGINT FK → Category (nullable, para jerarquía)
  created_at    TIMESTAMP

Product
  id            BIGINT PK
  name          VARCHAR(200) NOT NULL
  description   TEXT
  price         DECIMAL(10,2) NOT NULL
  stock         INT NOT NULL DEFAULT 0
  category_id   BIGINT FK → Category
  images        TEXT[]          -- URLs en S3
  dimensions    JSONB           -- { width, height, depth, weight }
  is_active     BOOLEAN DEFAULT TRUE
  created_at    TIMESTAMP
  updated_at    TIMESTAMP

User
  id            BIGINT PK
  email         VARCHAR(255) UNIQUE NOT NULL
  password_hash VARCHAR(255) NOT NULL  -- BCrypt
  full_name     VARCHAR(200)
  role          ENUM('CUSTOMER','ADMIN') DEFAULT 'CUSTOMER'
  status        ENUM('ACTIVE','INACTIVE') DEFAULT 'ACTIVE'
  created_at    TIMESTAMP

Address
  id            BIGINT PK
  user_id       BIGINT FK → User
  street        VARCHAR(300)
  city          VARCHAR(100)
  postal_code   VARCHAR(10)
  country       VARCHAR(2)      -- ISO 3166-1 alpha-2
  is_default    BOOLEAN DEFAULT FALSE

Cart
  id            BIGINT PK
  user_id       BIGINT FK → User (nullable para carrito anónimo)
  session_id    VARCHAR(100)    -- para carrito anónimo
  updated_at    TIMESTAMP

CartItem
  id            BIGINT PK
  cart_id       BIGINT FK → Cart
  product_id    BIGINT FK → Product
  quantity      INT NOT NULL
  unit_price    DECIMAL(10,2)   -- precio en el momento de añadir

Order
  id            BIGINT PK
  user_id       BIGINT FK → User
  status        ENUM('PENDIENTE_PAGO','PAGADO','EN_PREPARACION','ENVIADO','ENTREGADO','CANCELADO','DEVOLUCION_PARCIAL')
  shipping_address_id BIGINT FK → Address
  subtotal      DECIMAL(10,2)
  tax           DECIMAL(10,2)
  total         DECIMAL(10,2)
  stripe_payment_intent_id VARCHAR(200)
  created_at    TIMESTAMP
  updated_at    TIMESTAMP

OrderItem
  id            BIGINT PK
  order_id      BIGINT FK → Order
  product_id    BIGINT FK → Product
  quantity      INT NOT NULL
  unit_price    DECIMAL(10,2)
  product_name  VARCHAR(200)   -- snapshot al momento del pedido

RefreshToken
  id            BIGINT PK
  user_id       BIGINT FK → User
  token         VARCHAR(500) UNIQUE NOT NULL
  expires_at    TIMESTAMP
  revoked       BOOLEAN DEFAULT FALSE
```

### Migraciones Flyway (orden)

| Versión | Descripción |
|---------|-------------|
| V1__create_categories.sql | Tabla Category |
| V2__create_products.sql | Tabla Product |
| V3__create_users.sql | Tabla User |
| V4__create_addresses.sql | Tabla Address |
| V5__create_carts.sql | Tablas Cart + CartItem |
| V6__create_orders.sql | Tablas Order + OrderItem |
| V7__create_refresh_tokens.sql | Tabla RefreshToken |
| V8__seed_categories.sql | Datos iniciales de categorías |

---

## 3. APIs REST — Backend

### Autenticación (`/api/v1/auth`)
| Método | Endpoint | Acceso | Descripción |
|--------|----------|--------|-------------|
| POST | `/register` | Público | Registro de usuario |
| POST | `/login` | Público | Login → JWT |
| POST | `/refresh` | Público | Renovar access token |
| POST | `/logout` | Autenticado | Revocar refresh token |
| POST | `/forgot-password` | Público | Enviar email recuperación |
| POST | `/reset-password` | Público | Cambiar contraseña con token |

### Catálogo (`/api/v1/products`)
| Método | Endpoint | Acceso | Descripción |
|--------|----------|--------|-------------|
| GET | `/` | Público | Listado paginado con filtros |
| GET | `/{id}` | Público | Detalle de producto |
| GET | `/search?q=` | Público | Búsqueda por texto |
| POST | `/` | ADMIN | Crear producto |
| PUT | `/{id}` | ADMIN | Actualizar producto |
| DELETE | `/{id}` | ADMIN | Desactivar producto (soft delete) |
| POST | `/{id}/images` | ADMIN | Upload de imagen |

### Categorías (`/api/v1/categories`)
| Método | Endpoint | Acceso | Descripción |
|--------|----------|--------|-------------|
| GET | `/` | Público | Árbol de categorías |
| POST | `/` | ADMIN | Crear categoría |
| PUT | `/{id}` | ADMIN | Editar categoría |
| DELETE | `/{id}` | ADMIN | Eliminar categoría |

### Carrito (`/api/v1/cart`)
| Método | Endpoint | Acceso | Descripción |
|--------|----------|--------|-------------|
| GET | `/` | Autenticado | Ver carrito actual |
| POST | `/items` | Autenticado | Añadir item |
| PUT | `/items/{itemId}` | Autenticado | Actualizar cantidad |
| DELETE | `/items/{itemId}` | Autenticado | Eliminar item |
| DELETE | `/` | Autenticado | Vaciar carrito |

### Pedidos (`/api/v1/orders`)
| Método | Endpoint | Acceso | Descripción |
|--------|----------|--------|-------------|
| POST | `/` | Autenticado | Crear pedido (checkout) |
| GET | `/` | Autenticado | Historial del usuario |
| GET | `/{id}` | Autenticado | Detalle de pedido |
| GET | `/admin` | ADMIN | Todos los pedidos |
| PUT | `/admin/{id}/status` | ADMIN | Cambiar estado |

### Pagos (`/api/v1/payments`)
| Método | Endpoint | Acceso | Descripción |
|--------|----------|--------|-------------|
| POST | `/intent` | Autenticado | Crear PaymentIntent en Stripe |
| POST | `/webhook` | Público (verificado) | Webhook de Stripe |
| POST | `/refund/{orderId}` | ADMIN | Iniciar devolución |

### Usuarios (`/api/v1/users`)
| Método | Endpoint | Acceso | Descripción |
|--------|----------|--------|-------------|
| GET | `/me` | Autenticado | Perfil propio |
| PUT | `/me` | Autenticado | Actualizar perfil |
| GET | `/me/addresses` | Autenticado | Listado de direcciones |
| POST | `/me/addresses` | Autenticado | Añadir dirección |
| PUT | `/me/addresses/{id}` | Autenticado | Editar dirección |
| DELETE | `/me/addresses/{id}` | Autenticado | Eliminar dirección |
| GET | `/admin` | ADMIN | Listado de usuarios |
| PUT | `/admin/{id}/status` | ADMIN | Bloquear/desbloquear usuario |

---

## 4. Stack tecnológico

### Backend
| Librería | Versión | Propósito |
|----------|---------|-----------|
| Spring Boot | 3.3.x | Framework principal |
| Java | 17 | Lenguaje |
| Spring Security | 6.x | Autenticación / autorización |
| Spring Data JPA | 3.3.x | Persistencia |
| H2 | 2.x | Base de datos (embebida en dev; modo servidor opcional) |
| Flyway | 10.x | Migraciones de BD |
| MapStruct | 1.5.x | Mapeo entidad ↔ DTO |
| jjwt (io.jsonwebtoken) | 0.12.x | Generación/validación JWT |
| Stripe Java SDK | 25.x | Integración pagos |
| AWS SDK S3 / MinIO | 2.x | Almacenamiento imágenes |
| Spring Mail | 3.3.x | Envío de emails |
| Lombok | 1.18.x | Reducción boilerplate |
| JUnit 5 + Mockito | incluido | Tests |
| H2 | 2.x | BD embebida para tests y desarrollo local |

### Frontend
| Librería | Versión | Propósito |
|----------|---------|-----------|
| React | 18.x | Framework UI |
| TypeScript | 5.x | Tipado estático |
| Vite | 5.x | Build tool |
| React Router | 6.x | Routing |
| Redux Toolkit | 2.x | Estado global |
| RTK Query | 2.x | Fetching / caché de datos |
| Stripe.js / React Stripe.js | latest | Formulario de pago seguro |
| React Hook Form | 7.x | Manejo de formularios |
| Zod | 3.x | Validación de esquemas |
| Tailwind CSS | 3.x | Estilos |
| shadcn/ui | latest | Componentes accesibles |
| i18next | 23.x | Internacionalización |
| Vitest + React Testing Library | latest | Tests |

---

## 5. Seguridad

- **Autenticación**: JWT (access 15 min + refresh 7 días almacenado en cookie HttpOnly)
- **Autorización**: Spring Security con roles `CUSTOMER` y `ADMIN`; anotaciones `@PreAuthorize`
- **Contraseñas**: BCrypt con factor de coste 12
- **CORS**: configurado solo para el origen del frontend
- **HTTPS**: obligatorio en producción (TLS terminado en reverse proxy / load balancer)
- **Pagos**: datos de tarjeta gestionados 100 % por Stripe.js (PCI DSS compliance)
- **Webhook Stripe**: validación obligatoria de firma (`Stripe-Signature` header)
- **Uploads**: validación de tipo MIME y tamaño máximo (5 MB) en backend
- **SQL Injection**: uso exclusivo de JPA/prepared statements; prohibido `@NativeQuery` con concatenación
- **XSS**: sanitización de input en backend con OWASP Java HTML Sanitizer antes de persistir

---

## 6. Diagrama de arquitectura

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
  │  React / Vite   │     │  Spring Boot    │
  │  (puerto 3000)  │     │  (puerto 8080)  │
  └─────────────────┘     └────────┬────────┘
                          ┌────────┼────────────┐
                          │        │             │
               ┌──────────▼─┐  ┌───▼────┐  ┌───▼───┐
               │    H2      │  │  S3/   │  │ SMTP  │
               │   (2.x)    │  │ MinIO  │  │ server│
               └────────────┘  └────────┘  └───────┘
                          │
               ┌──────────▼──────────┐
               │    Stripe API        │
               │  (pagos externos)    │
               └─────────────────────┘
```

---

## 7. Decisiones de arquitectura (ADRs)

| # | Decisión | Alternativas | Razón |
|---|----------|--------------|-------|
| 1 | JWT en cookie HttpOnly para refresh token | localStorage | Mitigación XSS; localStorage expuesto a scripts |
| 2 | Flyway para migraciones | Liquibase | Simplicidad; integración nativa Spring Boot |
| 3 | MapStruct para mapeo | ModelMapper / manual | Type-safe en tiempo de compilación, sin reflection en runtime |
| 4 | Soft delete en productos | Delete físico | Historial de pedidos consistente; los pedidos referencing productos deben seguir funcionando |
| 5 | Stock decrementado atómicamente con `SELECT FOR UPDATE` | Optimistic locking | Garantía fuerte ante compras simultáneas del último stock |
| 6 | Stripe Checkout / Elements en front | Pasarela propia | PCI DSS: datos de tarjeta nunca tocan nuestros servidores |
| 7 | RTK Query para fetching | React Query | Integración natural con Redux; evita duplicar estado |
