# Plan técnico — Juvise: Frontend, Home y Catálogo

> Versión: v1
> Fecha creación: 2026-06-17
> Fecha última modificación: 2026-06-17
> Spec de referencia: `.specify/specs/juvise-frontend.md`
> Estado: BORRADOR

---

## 1. Stack tecnológico

Extiende el stack ya definido en `plans/tienda-muebles-online.md`. Librerías adicionales específicas de este módulo:

| Librería | Versión | Propósito |
|----------|---------|-----------|
| Embla Carousel | 8.x | Carrusel de productos más vendidos (5 KB, sin CSS global, accesible) |
| `@googlemaps/react-wrapper` | 1.x | Wrapper React oficial para Google Maps JS API v3 |
| Nuqs | 1.x | Sincronización tipada de filtros y paginación con query params de la URL |
| `react-slider` | 2.x | Slider doble para rango de precio |
| Google Fonts | — | Playfair Display (títulos) + Inter (cuerpo), cargadas con `display=swap` |
| `react-helmet-async` | 2.x | Meta tags y structured data (Schema.org) por página |

---

## 2. Extensión del modelo de datos

### Nuevas tablas (Flyway V14–V19)

```sql
-- V14__extend_product_brand_style_room.sql
ALTER TABLE product ADD COLUMN brand_id    BIGINT REFERENCES brand(id);
ALTER TABLE product ADD COLUMN room_id     BIGINT REFERENCES room(id);
ALTER TABLE product ADD COLUMN is_new      BOOLEAN DEFAULT FALSE;
ALTER TABLE product ADD COLUMN discount    SMALLINT DEFAULT 0;    -- % descuento (0-100)
ALTER TABLE product ADD COLUMN sales_count INT DEFAULT 0;         -- para ranking más vendidos

-- Tabla relación producto-estilo (M:N)
CREATE TABLE product_style (
  product_id  BIGINT NOT NULL REFERENCES product(id) ON DELETE CASCADE,
  style_id    BIGINT NOT NULL REFERENCES style(id)   ON DELETE CASCADE,
  PRIMARY KEY (product_id, style_id)
);

-- V15__create_brands.sql
CREATE TABLE brand (
  id          BIGINT PRIMARY KEY AUTO_INCREMENT,
  name        VARCHAR(100) NOT NULL,
  slug        VARCHAR(100) NOT NULL UNIQUE,
  logo_url    VARCHAR(500),
  description TEXT,
  is_active   BOOLEAN DEFAULT TRUE,
  created_at  TIMESTAMP NOT NULL DEFAULT NOW()
);

-- V16__create_styles.sql
CREATE TABLE style (
  id          BIGINT PRIMARY KEY AUTO_INCREMENT,
  name        VARCHAR(100) NOT NULL,
  slug        VARCHAR(100) NOT NULL UNIQUE,
  banner_url  VARCHAR(500),
  is_active   BOOLEAN DEFAULT TRUE
);

-- V17__create_rooms.sql
CREATE TABLE room (
  id            BIGINT PRIMARY KEY AUTO_INCREMENT,
  name          VARCHAR(100) NOT NULL,
  slug          VARCHAR(100) NOT NULL UNIQUE,
  icon          VARCHAR(50),              -- nombre de icono Lucide (ej: "sofa")
  display_order SMALLINT DEFAULT 0,
  is_active     BOOLEAN DEFAULT TRUE
);

-- V18__create_promo_banners.sql
CREATE TABLE promo_banner (
  id         BIGINT PRIMARY KEY AUTO_INCREMENT,
  title      VARCHAR(200),
  subtitle   VARCHAR(300),
  cta_text   VARCHAR(100),
  cta_url    VARCHAR(500),
  image_url  VARCHAR(500),
  position   ENUM('HERO','BETWEEN_ROOMS') DEFAULT 'BETWEEN_ROOMS',
  is_active  BOOLEAN DEFAULT TRUE,
  starts_at  TIMESTAMP,
  ends_at    TIMESTAMP
);

-- V19__seed_rooms.sql
INSERT INTO room (name, slug, icon, display_order) VALUES
  ('Salón',            'salon',          'sofa',        1),
  ('Dormitorio',       'dormitorio',     'bed-double',  2),
  ('Cocina & Comedor', 'cocina-comedor', 'utensils',    3),
  ('Baño',             'bano',           'bath',        4),
  ('Terraza & Jardín', 'terraza',        'tree-pine',   5),
  ('Oficina & Estudio','oficina',        'monitor',     6);
```

---

## 3. Nuevas APIs — Backend

### Base URL: `/api/v1`

#### GET `/home`
Endpoint de composición para la home — reduce las peticiones del cliente a una sola.
**Acceso:** Público  
**Cache:** Spring Cache (`@Cacheable("home")`) con TTL de 5 minutos  
**Response `200 OK`:**
```json
{
  "bestsellers": [
    { "id": 1, "name": "Sofá Chester", "price": 899.00, "imageUrl": "...", "slug": "sofa-chester" }
  ],
  "rooms": [
    {
      "id": 1, "name": "Salón", "slug": "salon", "icon": "sofa",
      "featuredProducts": [ /* top 4 por sales_count del room */ ]
    }
  ],
  "heroBanner": { "title": "...", "subtitle": "...", "ctaText": "...", "ctaUrl": "...", "imageUrl": "..." },
  "interBanner": { /* banner BETWEEN_ROOMS activo, o null */ }
}
```

---

#### GET `/products` — nuevos query params

| Param | Tipo | Descripción |
|-------|------|-------------|
| `roomId` | Long | Filtrar por habitáculo |
| `styleId` | Long[] | Uno o varios estilos (multi-valor: `?styleId=1&styleId=3`) |
| `brandId` | Long[] | Una o varias marcas |
| `minPrice` | Decimal | Precio mínimo (inclusivo) |
| `maxPrice` | Decimal | Precio máximo (inclusivo) |
| `material` | String[] | Materiales (WOOD, MDF, METAL, FABRIC, GLASS, RATTAN) |
| `onlyInStock` | Boolean | Solo productos con `stock > 0` |
| `minRating` | Integer | Valoración media mínima (1–5) |
| `sort` | Enum | `BESTSELLER` (defecto), `PRICE_ASC`, `PRICE_DESC`, `NEWEST`, `TOP_RATED` |
| `page` | Integer | Página 0-based (defecto: 0) |
| `size` | Integer | Tamaño: 12, 24 (defecto), 48 |

---

#### GET `/brands`
**Acceso:** Público | **Response:** `[{ id, name, slug, logoUrl }]`

#### GET `/styles`
**Acceso:** Público | **Response:** `[{ id, name, slug, bannerUrl }]`

#### GET `/rooms`
**Acceso:** Público | **Response:** `[{ id, name, slug, icon, displayOrder }]` ordenado por `display_order`

#### CRUD Admin

| Método | Endpoint | Descripción |
|--------|----------|-------------|
| `POST/PUT/DELETE` | `/admin/brands/{id?}` | Gestión de marcas |
| `POST/PUT/DELETE` | `/admin/styles/{id?}` | Gestión de estilos |
| `POST/PUT/DELETE` | `/admin/rooms/{id?}` | Gestión de habitáculos |
| `POST/PUT/DELETE` | `/admin/banners/{id?}` | Gestión de banners promocionales |
| `PUT` | `/admin/rooms/{id}/featured-products` | Asignar los 4 productos destacados del room |

---

## 4. Estructura de componentes — Frontend

```
src/
├── pages/
│   ├── HomePage.tsx                   # Composición de todas las secciones home
│   ├── CatalogPage.tsx                # Listado con filtros + paginación
│   └── StorePage.tsx                  # Página "Encuéntranos" con mapa completo
│
├── components/
│   ├── layout/
│   │   ├── Header/
│   │   │   ├── Header.tsx             # Sticky header
│   │   │   ├── NavDropdown.tsx        # Mega-menú desplegable (categorías/estilos/marcas)
│   │   │   ├── MobileMenu.tsx         # Menú hamburguesa para móvil
│   │   │   ├── SearchBar.tsx          # Búsqueda con resultados en tiempo real (debounce 300ms)
│   │   │   └── CartIcon.tsx           # Icono con badge de cantidad
│   │   └── Footer/
│   │       ├── Footer.tsx             # Layout completo del footer
│   │       ├── StoreInfo.tsx          # Dirección, teléfono, horario
│   │       └── StoreMap.tsx           # Google Maps embebido + fallback
│   │
│   ├── home/
│   │   ├── HeroBanner.tsx             # Banner promocional configurable superpuesto
│   │   ├── BestsellerCarousel.tsx     # Carrusel Embla con autoplay
│   │   ├── CarouselDots.tsx           # Puntos indicadores del carrusel
│   │   ├── RoomSection.tsx            # Sección individual de habitáculo
│   │   ├── RoomProductGrid.tsx        # Rejilla de 4 productos del habitáculo
│   │   └── InterBanner.tsx            # Banner de colección entre secciones
│   │
│   ├── catalog/
│   │   ├── FilterPanel.tsx            # Panel lateral (escritorio) / drawer (móvil)
│   │   ├── PriceRangeSlider.tsx       # Slider doble de precio con react-slider
│   │   ├── FilterCheckboxGroup.tsx    # Grupo de checkboxes reutilizable (estilos, marcas…)
│   │   ├── FilterTreeGroup.tsx        # Árbol jerárquico de categorías/habitáculos
│   │   ├── ProductGrid.tsx            # Vista rejilla (CSS Grid, responsive)
│   │   ├── ProductList.tsx            # Vista lista (horizontal)
│   │   ├── SortSelect.tsx             # Selector de ordenación
│   │   ├── ViewToggle.tsx             # Botones rejilla / lista
│   │   ├── PageSizeSelect.tsx         # Selector 12 / 24 / 48
│   │   └── Pagination.tsx             # Paginación numérica accesible
│   │
│   └── product/
│       └── ProductCard.tsx            # Tarjeta reutilizable:
│                                      # hover→imagen alternativa, badge, precio tachado,
│                                      # botón "Añadir al carrito" en hover
│
├── hooks/
│   ├── useHomeData.ts                 # RTK Query → GET /home
│   ├── useProducts.ts                 # RTK Query → GET /products (con filtros)
│   ├── useBrands.ts                   # RTK Query → GET /brands
│   ├── useStyles.ts                   # RTK Query → GET /styles
│   └── useRooms.ts                    # RTK Query → GET /rooms
│
└── store/
    └── filtersSlice.ts                # Estado Redux de filtros activos (sync con Nuqs)
```

---

## 5. Carrusel — Embla Carousel

```tsx
// BestsellerCarousel.tsx
import useEmblaCarousel from 'embla-carousel-react'
import Autoplay from 'embla-carousel-autoplay'

const autoplayPlugin = Autoplay({ delay: 5000, stopOnInteraction: true })

const [emblaRef, emblaApi] = useEmblaCarousel(
  { loop: true, align: 'start', dragFree: false },
  [autoplayPlugin]
)

// Dots: selectedIndex = emblaApi.selectedScrollSnap()
// Flechas: emblaApi.scrollPrev() / emblaApi.scrollNext()
// Accesibilidad: role="region" aria-label="Productos más vendidos"
// Cada slide: role="group" aria-roledescription="slide"
```

---

## 6. Filtros y URL state — Nuqs

```tsx
// CatalogPage.tsx
import { useQueryStates, parseAsInteger, parseAsFloat, parseAsArrayOf } from 'nuqs'

const [filters, setFilters] = useQueryStates({
  page:        parseAsInteger.withDefault(0),
  size:        parseAsInteger.withDefault(24),
  sort:        parseAsString.withDefault('BESTSELLER'),
  roomId:      parseAsInteger,
  styleId:     parseAsArrayOf(parseAsInteger),
  brandId:     parseAsArrayOf(parseAsInteger),
  minPrice:    parseAsFloat,
  maxPrice:    parseAsFloat,
  onlyInStock: parseAsBoolean.withDefault(false),
  minRating:   parseAsInteger,
})

// Al cambiar cualquier filtro (excepto page) → resetear page a 0
const handleFilterChange = (key: string, value: unknown) => {
  setFilters({ [key]: value, page: 0 })
}

// URL resultante:
// /catalogo?roomId=1&styleId=2&styleId=4&minPrice=100&maxPrice=800&sort=PRICE_ASC&page=0
```

---

## 7. Google Maps — Integración técnica

```tsx
// StoreMap.tsx
import { Wrapper, Status } from '@googlemaps/react-wrapper'

// Coordenadas de Calle Agentes IA 10, Madrid
const STORE_COORDINATES = { lat: 40.4168, lng: -3.7038 }
const API_KEY = import.meta.env.VITE_GOOGLE_MAPS_API_KEY

const render = (status: Status) => {
  if (status === Status.LOADING) return <Skeleton className="h-64 w-full" />
  if (status === Status.FAILURE) return (
    <MapFallback
      address="Calle Agentes IA 10, Madrid"
      mapsUrl="https://maps.google.com/?q=Calle+Agentes+IA+10,Madrid"
    />
  )
  return <GoogleMapComponent coords={STORE_COORDINATES} />
}

// GoogleMapComponent: new google.maps.Map(ref, { center, zoom: 16 })
//   + new google.maps.Marker({ position: STORE_COORDINATES, map })
//   + InfoWindow con nombre y dirección de la tienda

// Enlace "Cómo llegar"
const DIRECTIONS_URL =
  'https://www.google.com/maps/dir/?api=1&destination=Calle+Agentes+IA+10,Madrid'
```

**Restricción de API Key en Google Cloud Console:**
- Tipo de restricción: Referrers HTTP
- Dominios permitidos: `juvise.com/*`, `*.juvise.com/*`

---

## 8. Diseño — Tipografías y tokens Tailwind

### Fuentes (`index.html`)
```html
<link rel="preconnect" href="https://fonts.googleapis.com">
<link href="https://fonts.googleapis.com/css2?family=Playfair+Display:wght@400;600;700&family=Inter:wght@300;400;500;600&display=swap" rel="stylesheet">
```

### `tailwind.config.ts`
```ts
theme: {
  extend: {
    fontFamily: {
      display: ['Playfair Display', 'serif'],
      body:    ['Inter', 'sans-serif'],
    },
    colors: {
      brand: {
        black:  '#1A1A1A',
        gold:   '#C9A96E',
        cream:  '#F5F0E8',
        'gray': '#8A8A8A',
      }
    },
    keyframes: {
      fadeIn: { from: { opacity: '0', transform: 'translateY(4px)' }, to: { opacity: '1', transform: 'none' } },
    },
    animation: {
      'fade-in': 'fadeIn 0.25s ease-out',
    }
  }
}
```

---

## 9. SEO — Structured data y meta tags

```tsx
// HomePage.tsx — Schema.org FurnitureStore
<Helmet>
  <title>Juvise — Muebles de diseño en Madrid</title>
  <meta name="description" content="Tienda de muebles vanguardistas en Madrid. Salón, dormitorio, cocina, baño, terraza y oficina. Estilos nórdico, industrial, minimalista y más." />
  <meta property="og:title" content="Juvise — Muebles de diseño en Madrid" />
  <meta property="og:image" content="/og-juvise.jpg" />
  <script type="application/ld+json">{JSON.stringify({
    "@context": "https://schema.org",
    "@type": "FurnitureStore",
    "name": "Juvise",
    "url": "https://juvise.com",
    "address": {
      "@type": "PostalAddress",
      "streetAddress": "Calle Agentes IA 10",
      "addressLocality": "Madrid",
      "addressCountry": "ES"
    },
    "geo": {
      "@type": "GeoCoordinates",
      "latitude": 40.4168,
      "longitude": -3.7038
    },
    "openingHoursSpecification": [
      { "@type": "OpeningHoursSpecification", "dayOfWeek": ["Monday","Tuesday","Wednesday","Thursday","Friday"], "opens": "10:00", "closes": "20:00" },
      { "@type": "OpeningHoursSpecification", "dayOfWeek": "Saturday", "opens": "10:00", "closes": "14:00" }
    ]
  })}</script>
</Helmet>
```

---

## 10. Variables de entorno adicionales

```
# Google Maps
VITE_GOOGLE_MAPS_API_KEY=AIza...        # Restringida a dominio juvise.com

# Datos de tienda (usados en StoreInfo, SEO y fallback)
VITE_STORE_LATITUDE=40.4168
VITE_STORE_LONGITUDE=-3.7038
VITE_STORE_ADDRESS=Calle Agentes IA 10, Madrid
VITE_STORE_PHONE=+34 91 000 00 00
VITE_STORE_EMAIL=hola@juvise.com
```

---

## 11. Decisiones técnicas

| # | Decisión | Alternativas | Razón |
|---|----------|--------------|-------|
| 1 | Embla Carousel | Swiper, Keen Slider | 5 KB sin CSS global; no rompe Tailwind; accesible por defecto con ARIA roles |
| 2 | Endpoint `/home` de composición | N peticiones paralelas en el cliente | Reduce waterfall de red; el backend cachea el resultado 5 min con Spring Cache |
| 3 | Nuqs para URL state de filtros | `useState` + `useSearchParams` manual | Serialización tipada, parsing automático, URLs limpias y compartibles |
| 4 | `@googlemaps/react-wrapper` | `react-google-maps/api`, `@vis.gl/react-google-maps` | Librería oficial de Google; sin dependencias adicionales; fallback declarativo por `Status` |
| 5 | API Key restringida a referrer HTTP | Key sin restricciones | Obligatorio en producción; evita consumo no autorizado de la cuota |
| 6 | Schema.org `FurnitureStore` en JSON-LD | Microdatos HTML | Formato preferido por Google Search; no contamina el HTML semántico |
| 7 | Tailwind tokens de marca centralizados | Valores inline o CSS variables | Coherencia visual garantizada; cambio de paleta en un solo fichero |

---
## Changelog

| Versión | Fecha | Descripción del cambio |
|---------|-------|------------------------|
| v1 | 2026-06-17 | Creación inicial — plan técnico de Juvise frontend (home, carrusel Embla, filtros Nuqs, Google Maps, modelo de datos extendido, nuevas APIs, componentes React, tokens Tailwind, SEO) |
