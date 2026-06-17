# Tareas — Juvise: Frontend, Home y Catálogo

> Fecha: 2026-06-17
> Plan de referencia: `.specify/plans/juvise-frontend.md`
> Spec de referencia: `.specify/specs/juvise-frontend.md`
> Estado: PENDIENTE

---

## Épica 1 — Modelo de datos extendido

### TASK-J001 — Migraciones Flyway V14–V19
- **Estado**: `TODO`
- **Descripción**: Crear todas las migraciones para las entidades nuevas de Juvise.
- **Subtareas**:
  - [ ] `V14__extend_product_brand_style_room.sql` — añadir `brand_id`, `room_id`, `is_new`, `discount`, `sales_count` a `product`; crear tabla `product_style` (M:N)
  - [ ] `V15__create_brands.sql` — tabla `brand` con `slug` UNIQUE
  - [ ] `V16__create_styles.sql` — tabla `style` con `slug` UNIQUE
  - [ ] `V17__create_rooms.sql` — tabla `room` con `icon`, `display_order`
  - [ ] `V18__create_promo_banners.sql` — tabla `promo_banner` con ENUM `position` (`HERO`, `BETWEEN_ROOMS`)
  - [ ] `V19__seed_rooms.sql` — insertar los 6 habitáculos (Salón, Dormitorio, Cocina & Comedor, Baño, Terraza & Jardín, Oficina & Estudio)
- **Criterio de aceptación**: Flyway aplica V14–V19 sin errores; datos de rooms semilla presentes tras arranque.

### TASK-J002 — Entidades JPA y repositorios
- **Estado**: `TODO`
- **Descripción**: Implementar las entidades y repositorios correspondientes a las nuevas tablas.
- **Subtareas**:
  - [ ] Entidades: `Brand`, `Style`, `Room`, `PromoBanner`, `ProductStyle` (clave compuesta)
  - [ ] Actualizar entidad `Product`: relaciones `@ManyToOne` a `Brand` y `Room`; `@ManyToMany` a `Style` via `product_style`; campos `isNew`, `discount`, `salesCount`
  - [ ] Repositorios: `BrandRepository`, `StyleRepository`, `RoomRepository`, `PromoBannerRepository`
  - [ ] Tests de repositorio con H2 en memoria
- **Criterio de aceptación**: Relaciones JPA correctas; no hay N+1 queries en los endpoints de home y catálogo.

---

## Épica 2 — APIs backend

### TASK-J003 — `GET /home` (endpoint de composición)
- **Estado**: `TODO`
- **Descripción**: Endpoint único que agrega datos para la home: bestsellers, rooms, hero banner e inter banner.
- **Subtareas**:
  - [ ] `HomeService.compose()`:
    - [ ] Top 10 bestsellers ordenados por `sales_count DESC`
    - [ ] Todos los rooms activos con sus 4 productos destacados (por `sales_count`)
    - [ ] Banner `HERO` activo (dentro de rango `starts_at`/`ends_at`)
    - [ ] Banner `BETWEEN_ROOMS` activo (o `null`)
  - [ ] `@Cacheable("home")` con TTL de 5 minutos (Spring Cache + Caffeine)
  - [ ] Invalidar caché en operaciones admin de banners, rooms y productos
  - [ ] Tests unitarios de `HomeService` con mocks
- **Criterio de aceptación**: Respuesta conforme al contrato JSON del plan. Segunda llamada en < 5 min usa caché. Tiempo de respuesta < 100 ms en caché hit.

### TASK-J004 — `GET /products` extendido
- **Estado**: `TODO`
- **Descripción**: Añadir los nuevos query params de filtrado y ordenación al endpoint de catálogo.
- **Subtareas**:
  - [ ] Nuevos params: `roomId`, `styleId[]`, `brandId[]`, `minPrice`, `maxPrice`, `material[]`, `onlyInStock`, `minRating`
  - [ ] Nuevo param `sort`: `BESTSELLER` (defecto), `PRICE_ASC`, `PRICE_DESC`, `NEWEST`, `TOP_RATED`
  - [ ] Tamaños de página: 12, 24 (defecto), 48
  - [ ] Implementar con `JPA Criteria` o `Specification` para combinaciones dinámicas
  - [ ] Tests de integración por cada combinación de filtro
- **Criterio de aceptación**: Filtros combinados devuelven resultados correctos. `onlyInStock=true` excluye productos con `stock=0`.

### TASK-J005 — `GET /brands`, `GET /styles`, `GET /rooms`
- **Estado**: `TODO`
- **Descripción**: Endpoints de referencia para poblar los filtros del catálogo.
- **Subtareas**:
  - [ ] `GET /api/v1/brands` → `[{ id, name, slug, logoUrl }]` — solo marcas activas
  - [ ] `GET /api/v1/styles` → `[{ id, name, slug, bannerUrl }]` — solo estilos activos
  - [ ] `GET /api/v1/rooms` → `[{ id, name, slug, icon, displayOrder }]` ordenados por `display_order`
  - [ ] Tests
- **Criterio de aceptación**: Entidades inactivas no aparecen. Rooms ordenados correctamente.

### TASK-J006 — CRUD Admin (brands, styles, rooms, banners)
- **Estado**: `TODO`
- **Descripción**: Endpoints admin para gestionar el contenido editorial.
- **Subtareas**:
  - [ ] `POST/PUT/DELETE /api/v1/admin/brands/{id?}`
  - [ ] `POST/PUT/DELETE /api/v1/admin/styles/{id?}`
  - [ ] `POST/PUT/DELETE /api/v1/admin/rooms/{id?}`
  - [ ] `POST/PUT/DELETE /api/v1/admin/banners/{id?}` — validar solapamiento de fechas en banners activos del mismo `position`
  - [ ] `PUT /api/v1/admin/rooms/{id}/featured-products` — asignar array de 4 product IDs como destacados del room
  - [ ] `@PreAuthorize("hasRole('ADMIN')")` en todos; invalidar caché `home` en cada mutación
  - [ ] Tests de integración
- **Criterio de aceptación**: No-admin → 403. Caché de home se invalida al crear/editar/borrar cualquier entidad de estas.

---

## Épica 3 — Layout y navegación (Frontend)

### TASK-J007 — Header
- **Estado**: `TODO`
- **Descripción**: Header sticky con mega-menú, búsqueda y carrito.
- **Subtareas**:
  - [ ] `Header.tsx`: sticky con `position: sticky; top: 0; z-index: 50`; fuente Inter; logo Juvise
  - [ ] `NavDropdown.tsx`: mega-menú con categorías, estilos y marcas cargadas de RTK Query
  - [ ] `MobileMenu.tsx`: drawer lateral con hamburguesa (accesible: `aria-expanded`, `aria-controls`)
  - [ ] `SearchBar.tsx`: input con debounce 300 ms → `GET /products?q=`; resultados en popover
  - [ ] `CartIcon.tsx`: badge con cantidad de items del carrito (Redux)
  - [ ] Tests de componente
- **Criterio de aceptación**: Header visible en scroll. Menú mobile abre/cierra sin layout shift. Badge se actualiza al añadir al carrito.

### TASK-J008 — Footer
- **Estado**: `TODO`
- **Descripción**: Footer completo con información de tienda y mapa embebido.
- **Subtareas**:
  - [ ] `Footer.tsx`: layout de 3–4 columnas (responsive)
  - [ ] `StoreInfo.tsx`: dirección (Calle Agentes IA 10, Madrid), teléfono, horario (L–V 10–20, S 10–14)
  - [ ] `StoreMap.tsx` (mini): Google Maps embebido 200 px alto con marker; fallback link a Google Maps si la API falla
  - [ ] Tests
- **Criterio de aceptación**: Fallback visible si `VITE_GOOGLE_MAPS_API_KEY` está vacía o la API falla.

---

## Épica 4 — Home

### TASK-J009 — HeroBanner
- **Estado**: `TODO`
- **Descripción**: Banner hero configurable cargado desde el endpoint `/home`.
- **Subtareas**:
  - [ ] `HeroBanner.tsx`: imagen de fondo, overlay semitransparente, título (Playfair Display), subtítulo (Inter), CTA button
  - [ ] Props desde `homeData.heroBanner`; si `null`, mostrar banner por defecto (imagen placeholder)
  - [ ] Responsive: altura full-viewport en desktop, 60 vw en mobile
  - [ ] Tests
- **Criterio de aceptación**: Si no hay banner activo, se muestra el placeholder sin romper el layout.

### TASK-J010 — BestsellerCarousel (Embla)
- **Estado**: `TODO`
- **Descripción**: Carrusel de productos más vendidos con Embla + autoplay.
- **Subtareas**:
  - [ ] `BestsellerCarousel.tsx`: `useEmblaCarousel({ loop: true, align: 'start' }, [Autoplay({ delay: 5000, stopOnInteraction: true })])`
  - [ ] `CarouselDots.tsx`: puntos sincronizados con `selectedScrollSnap()`; clicables
  - [ ] Flechas prev/next accesibles (`aria-label`)
  - [ ] Accesibilidad: `role="region"` + `aria-label="Productos más vendidos"`; cada slide `role="group"` + `aria-roledescription="slide"`
  - [ ] Tests
- **Criterio de aceptación**: Sin salto visual al hacer loop. Autoplay se pausa al interactuar. Navegación por teclado funcional.

### TASK-J011 — RoomSection + RoomProductGrid
- **Estado**: `TODO`
- **Descripción**: Secciones de habitáculos con rejilla de 4 productos destacados.
- **Subtareas**:
  - [ ] `RoomSection.tsx`: título del room con icono Lucide, CTA "Ver todo" → `/catalogo?roomId={id}`
  - [ ] `RoomProductGrid.tsx`: CSS Grid 4 columnas desktop / 2 mobile; usa `ProductCard`
  - [ ] Iterar sobre `homeData.rooms` para renderizar todas las secciones
  - [ ] Tests
- **Criterio de aceptación**: Cada room muestra exactamente 4 productos. El CTA filtra el catálogo por room.

### TASK-J012 — InterBanner
- **Estado**: `TODO`
- **Descripción**: Banner promocional entre secciones de habitáculos.
- **Subtareas**:
  - [ ] `InterBanner.tsx`: aparece entre la 2ª y 3ª sección de room (configurable)
  - [ ] Datos desde `homeData.interBanner`; si `null`, no renderiza nada (no deja hueco)
  - [ ] Tests
- **Criterio de aceptación**: Si no hay inter banner activo, el layout no deja espacio vacío.

---

## Épica 5 — Catálogo

### TASK-J013 — Panel de filtros
- **Estado**: `TODO`
- **Descripción**: Panel lateral (desktop) / drawer (mobile) con todos los controles de filtrado.
- **Subtareas**:
  - [ ] `FilterPanel.tsx`: sidebar fijo en desktop; drawer `<Dialog>` accesible en mobile
  - [ ] `FilterCheckboxGroup.tsx`: reutilizable para estilos, marcas y materiales; muestra badge con cantidad seleccionada
  - [ ] `FilterTreeGroup.tsx`: árbol jerárquico de categorías expandible con `<details>/<summary>`
  - [ ] `PriceRangeSlider.tsx`: slider doble con `react-slider`; inputs numéricos sincronizados
  - [ ] Botón "Limpiar filtros" resetea todos a defaults
  - [ ] Tests
- **Criterio de aceptación**: Aplicar filtros actualiza la URL y la lista de productos sin recargar página. Filtros se preservan al compartir URL.

### TASK-J014 — Grid, lista, ordenación y paginación
- **Estado**: `TODO`
- **Descripción**: Vistas de productos y controles de presentación del catálogo.
- **Subtareas**:
  - [ ] `ProductGrid.tsx`: CSS Grid responsivo 4/3/2/1 col según breakpoint
  - [ ] `ProductList.tsx`: vista horizontal con imagen pequeña + datos
  - [ ] `ViewToggle.tsx`: botones rejilla/lista que cambian la vista activa
  - [ ] `SortSelect.tsx`: selector de ordenación (BESTSELLER, PRICE_ASC, PRICE_DESC, NEWEST, TOP_RATED)
  - [ ] `PageSizeSelect.tsx`: selector 12 / 24 / 48
  - [ ] `Pagination.tsx`: numérica, accesible (`aria-label`, `aria-current`), máximo 7 páginas visibles
  - [ ] Tests
- **Criterio de aceptación**: Cambio de ordenación resetea página a 0. Paginación no muestra más de 7 botones.

### TASK-J015 — URL state de filtros con Nuqs
- **Estado**: `TODO`
- **Descripción**: Sincronizar todos los filtros y la paginación con los query params de la URL.
- **Subtareas**:
  - [ ] `useQueryStates` con parsers tipados: `page` (Integer, default 0), `size` (Integer, default 24), `sort` (String), `roomId` (Integer), `styleId` (ArrayOf Integer), `brandId` (ArrayOf Integer), `minPrice`/`maxPrice` (Float), `onlyInStock` (Boolean), `minRating` (Integer)
  - [ ] Al cambiar cualquier filtro distinto de `page` → resetear `page` a 0
  - [ ] `useProducts` hook: deriva los params de RTK Query desde los query states
  - [ ] Tests del hook
- **Criterio de aceptación**: La URL `/catalogo?roomId=1&styleId=2&sort=PRICE_ASC&page=2` carga el catálogo con esos filtros activos. URL compartible reproduce el mismo estado.

---

## Épica 6 — ProductCard

### TASK-J016 — `ProductCard`
- **Estado**: `TODO`
- **Descripción**: Tarjeta de producto reutilizable con hover states y accesiones de carrito.
- **Subtareas**:
  - [ ] Imagen principal; en hover → imagen alternativa (segunda imagen si existe)
  - [ ] Badges: `NUEVO` (si `isNew`), `-%` (si `discount > 0`)
  - [ ] Precio tachado + precio con descuento calculado
  - [ ] Botón "Añadir al carrito" visible en hover (con animación `fade-in`)
  - [ ] Accesibilidad: foco visible, `aria-label` descriptivo, botón carrito con `aria-label`
  - [ ] Tests de componente
- **Criterio de aceptación**: Sin hover el botón de carrito no ocupa espacio (no cambia altura de card). Precio con descuento calculado correctamente.

---

## Épica 7 — Integración Google Maps

### TASK-J017 — `StoreMap` (Google Maps completo)
- **Estado**: `TODO`
- **Descripción**: Mapa interactivo de la tienda con marker e info window.
- **Subtareas**:
  - [ ] Dependencia `@googlemaps/react-wrapper` 1.x
  - [ ] `StoreMap.tsx`: `<Wrapper apiKey={VITE_GOOGLE_MAPS_API_KEY} render={render}>` con estados `LOADING`, `FAILURE`, `SUCCESS`
  - [ ] `GoogleMapComponent`: `new google.maps.Map(ref, { center: STORE_COORDINATES, zoom: 16 })` + `Marker` + `InfoWindow` con nombre y dirección
  - [ ] `MapFallback`: enlace a Google Maps con `?q=Calle+Agentes+IA+10,Madrid`; mostrar dirección en texto
  - [ ] API Key restringida a `juvise.com/*` (documentado en `.env.example`)
  - [ ] Tests con mock de `@googlemaps/react-wrapper`
- **Criterio de aceptación**: Si la API Key es inválida o está vacía, el fallback se muestra sin error en consola. El marker apunta a Calle Agentes IA 10, Madrid.

### TASK-J018 — `StorePage` ("Encuéntranos")
- **Estado**: `TODO`
- **Descripción**: Página completa de localización de la tienda.
- **Subtareas**:
  - [ ] Layout: mapa grande (altura 400 px), datos de contacto, horario, enlace "Cómo llegar"
  - [ ] URL: `/tienda`
  - [ ] Schema.org `LocalBusiness` con `openingHoursSpecification`
  - [ ] Tests
- **Criterio de aceptación**: Página indexable con datos estructurados correctos.

---

## Épica 8 — Diseño y SEO

### TASK-J019 — Tokens Tailwind + tipografías
- **Estado**: `TODO`
- **Descripción**: Configurar el sistema de diseño de Juvise en Tailwind.
- **Subtareas**:
  - [ ] `tailwind.config.ts`: familias `font-display` (Playfair Display) y `font-body` (Inter)
  - [ ] Paleta `brand`: `black #1A1A1A`, `gold #C9A96E`, `cream #F5F0E8`, `gray #8A8A8A`
  - [ ] Keyframe `fadeIn` + clase `animate-fade-in`
  - [ ] `index.html`: `<link>` Google Fonts con `display=swap`; preconnect a `fonts.googleapis.com` y `fonts.gstatic.com`
  - [ ] Tests visuales / Storybook (opcional)
- **Criterio de aceptación**: Tokens disponibles en todas las páginas. No hay CLS por carga de fuentes (`display=swap`).

### TASK-J020 — SEO (meta tags y Schema.org)
- **Estado**: `TODO`
- **Descripción**: Implementar meta tags y datos estructurados por página.
- **Subtareas**:
  - [ ] Dependencia `react-helmet-async` 2.x; `<HelmetProvider>` en el root
  - [ ] `HomePage`: `<title>`, `description`, `og:title`, `og:image`, Schema.org `FurnitureStore` con dirección, geo y `openingHoursSpecification`
  - [ ] `CatalogPage`: título dinámico según filtros activos, `noindex` si página > 1
  - [ ] `StorePage`: Schema.org `LocalBusiness`
  - [ ] Tests con `@testing-library/jest-dom`
- **Criterio de aceptación**: Google Rich Results Test valida el JSON-LD de `FurnitureStore`. Meta tags correctos por página.

---

## Épica 9 — Variables de entorno y configuración

### TASK-J021 — Variables de entorno adicionales
- **Estado**: `TODO`
- **Descripción**: Documentar y configurar todas las variables de entorno nuevas de este módulo.
- **Subtareas**:
  - [ ] Añadir al `.env.example`:
    - `VITE_GOOGLE_MAPS_API_KEY` (restringida a dominio juvise.com)
    - `VITE_STORE_LATITUDE`, `VITE_STORE_LONGITUDE`
    - `VITE_STORE_ADDRESS`, `VITE_STORE_PHONE`, `VITE_STORE_EMAIL`
  - [ ] Variable backend para TTL de caché home: `HOME_CACHE_TTL_SECONDS=300`
  - [ ] Documentar en `docker-compose.override.yml` cómo pasar `VITE_GOOGLE_MAPS_API_KEY` al contenedor de frontend en desarrollo
  - [ ] Tests de smoke: arrancar la app sin la clave de Maps no produce error fatal (solo muestra fallback)
- **Criterio de aceptación**: `.env.example` tiene todas las variables; `README.md` del proyecto referencia `.env.example`.
