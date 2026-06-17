# Especificación — Juvise: Tienda de Muebles Online (Identidad y Experiencia)

> Versión: v1
> Fecha creación: 2026-06-17
> Fecha última modificación: 2026-06-17
> Estado: BORRADOR
> Autor: equipo
> Plan de referencia: `.specify/plans/juvise-frontend.md`

---

## 1. Descripción general

**Juvise** es una tienda de muebles para viviendas ubicada en Madrid (Calle Agentes IA 10).
Este módulo define la identidad visual, la estructura de la página principal y los elementos de navegación y descubrimiento de la tienda online.

Juvise diferencia su oferta mediante:
- Una estética **vanguardista** (diseño limpio, tipografía contemporánea, paleta neutra con acentos de color)
- Organización del catálogo por **habitáculos** del hogar (salón, dormitorio, cocina, baño, terraza, oficina)
- Navegación avanzada por **estilos** (minimalista, nórdico, industrial, mediterráneo, clásico) y **marcas**
- Presencia física en Madrid con localización integrada en la web

---

## 2. Página principal (Home)

### 2.1 Header

El header es fijo (sticky) y visible en todo momento durante el scroll. Contiene:

- **Logo Juvise** — enlaza a la página principal
- **Menú de categorías** — desplegable con las categorías de muebles agrupadas por habitáculo
- **Menú de estilos** — desplegable con los estilos decorativos disponibles
- **Menú de marcas** — listado de marcas comercializadas
- **Buscador** — búsqueda por texto libre sobre productos, categorías y marcas; resultados en tiempo real
- **Área de usuario** — acceso a login/registro, perfil, historial de pedidos y lista de favoritos
- **Icono de carrito** — con contador de ítems y acceso al carrito

En móvil, el menú principal se condensa en un menú hamburguesa.

### 2.2 Sección Hero + Carrusel de más vendidos

La sección principal de la home contiene:

- **Carrusel automático** con los productos más vendidos del período actual
- Cada tarjeta del carrusel muestra: imagen principal del producto, nombre, precio y botón "Ver producto"
- El carrusel avanza automáticamente cada 5 segundos y permite navegación manual (flechas + puntos indicadores)
- El usuario puede pausar el avance automático al pasar el cursor sobre el carrusel
- Sobre el carrusel puede superponerse una **faixa promocional** (banner con texto y CTA configurables desde el panel de admin)

### 2.3 Secciones por habitáculo

Debajo del carrusel se presentan secciones independientes, una por cada habitáculo principal:

| Habitáculo | Descripción breve visible en la sección |
|------------|----------------------------------------|
| Salón | Sofás, mesas de centro, estanterías, TV units |
| Dormitorio | Camas, mesitas, armarios, cómodas |
| Cocina & Comedor | Mesas de comedor, sillas, bancos, vajilleros |
| Baño | Muebles de baño, espejos, accesorios |
| Terraza & Jardín | Muebles exterior, hamacas, parasoles |
| Oficina & Estudio | Escritorios, sillas de trabajo, estanterías |

Cada sección muestra:
- Título del habitáculo con icono representativo
- Rejilla de 4 productos destacados (los más vendidos o seleccionados manualmente por admin)
- Botón "Ver todo" que lleva al listado filtrado por ese habitáculo

### 2.4 Banner de estilo / colección destacada

Entre las secciones de habitáculos puede insertarse un banner visual de ancho completo que destaca una **colección o estilo** (ej.: "Nueva colección Nórdica"). Es configurable desde el panel de admin.

### 2.5 Footer

El footer contiene:

- **Información de la tienda física:**
  - Nombre: Juvise
  - Dirección: Calle Agentes IA 10, Madrid
  - Teléfono de contacto
  - Horario de atención
  - **Mapa embebido de Google Maps** centrado en la ubicación exacta de la tienda
- **Columnas de navegación:**
  - Sobre Juvise (quiénes somos, sostenibilidad, trabaja con nosotros)
  - Atención al cliente (devoluciones, envíos, FAQs, contacto)
  - Categorías principales (acceso rápido)
  - Síguenos (redes sociales: Instagram, Pinterest, TikTok)
- **Aviso legal, política de privacidad y cookies**
- **Copyright Juvise**

---

## 3. Catálogo y navegación

### 3.1 Filtros en listado de productos

El listado de productos del catálogo incluye un panel de filtros lateral (escritorio) o en drawer inferior (móvil) con:

- **Rango de precio** — slider doble (precio mínimo / máximo)
- **Categoría / habitáculo** — árbol jerárquico seleccionable
- **Estilo** — checkboxes (minimalista, nórdico, industrial, mediterráneo, clásico)
- **Marca** — checkboxes con búsqueda dentro del panel
- **Material** — checkboxes (madera maciza, DM, metal, tapizado, cristal, ratán)
- **Disponibilidad** — toggle "Solo en stock"
- **Valoración** — mínimo de estrellas (1–5)

Los filtros son **acumulables** y la URL refleja el estado de los filtros (compartible por enlace).

### 3.2 Paginación

- El listado de productos usa **paginación numérica** con botones anterior/siguiente y acceso directo a páginas
- El tamaño de página por defecto es 24 productos; el usuario puede cambiarlo a 12, 48 o "Ver todos"
- La paginación se refleja en la URL (`?page=2`) para compatibilidad con la navegación del historial del navegador

### 3.3 Ordenación

El usuario puede ordenar el listado por:
- Más vendidos (defecto)
- Precio: menor a mayor
- Precio: mayor a menor
- Novedades
- Mejor valorados

### 3.4 Vistas del listado

El usuario puede alternar entre:
- **Vista rejilla** (2, 3 o 4 columnas según dispositivo)
- **Vista lista** (imagen + descripción resumida + precio en horizontal)

---

## 4. Localización física — Google Maps

La tienda Juvise dispone de una sección "Encuéntranos" accesible desde el footer y desde la página "Sobre Juvise". Contiene:

- **Mapa interactivo embebido** con un marcador en la dirección exacta: **Calle Agentes IA 10, Madrid**
- El mapa permite zoom, desplazamiento y apertura en Google Maps en nueva pestaña
- Junto al mapa se muestra la tarjeta de información de la tienda (dirección, teléfono, horario)
- Botón **"Cómo llegar"** que abre Google Maps con navegación activa hacia la tienda
- Si el mapa no puede cargarse, se muestra la dirección en texto plano con un enlace externo a Google Maps

---

## 5. Identidad visual y estilo

### 5.1 Estilo general
- Estética **vanguardista**: espacios en blanco amplios, tipografía moderna, fotografía de producto de alta calidad
- **Paleta de color neutra** con un color de acento configurable (por defecto: negro mate + dorado)
- Transiciones y microanimaciones sutiles en hover de tarjetas de producto y elementos del header
- Diseño completamente **responsivo** (mobile-first): mobile, tablet, escritorio

### 5.2 Tipografía
- Títulos: fuente contemporánea de estilo geométrico/humanista
- Cuerpo de texto: fuente sans-serif de alta legibilidad
- Ambas cargadas desde Google Fonts

### 5.3 Tarjeta de producto
- Imagen principal con efecto hover que muestra la imagen alternativa del producto
- Nombre, precio tachado (si hay descuento) y precio actual
- Badge de novedad / descuento / más vendido superpuesto sobre la imagen
- Botón "Añadir al carrito" visible al hacer hover

---

## 6. Criterios de aceptación

### Header y navegación
- [ ] El header es sticky y funcional durante el scroll en todos los dispositivos
- [ ] Los desplegables de categorías, estilos y marcas son accesibles en escritorio y en menú hamburguesa en móvil
- [ ] El buscador devuelve resultados relevantes mientras el usuario escribe (búsqueda en tiempo real)
- [ ] El icono de carrito refleja el número de ítems actual

### Home — Carrusel
- [ ] El carrusel muestra los productos más vendidos y avanza automáticamente cada 5 segundos
- [ ] El usuario puede pausar el avance automático al pasar el cursor sobre el carrusel
- [ ] La navegación manual (flechas y puntos indicadores) funciona correctamente
- [ ] Cada tarjeta lleva al detalle del producto al hacer clic

### Home — Secciones por habitáculo
- [ ] Se muestran las 6 secciones de habitáculos en la home
- [ ] Cada sección muestra 4 productos destacados
- [ ] El botón "Ver todo" lleva al listado filtrado por ese habitáculo

### Filtros y catálogo
- [ ] Los filtros de precio, categoría, estilo, marca, material, disponibilidad y valoración funcionan de forma acumulable
- [ ] La URL se actualiza con los filtros activos (enlace compartible)
- [ ] La paginación funciona y la URL refleja la página actual
- [ ] El usuario puede ordenar el listado y cambiar entre vista rejilla y lista

### Localización
- [ ] El mapa de Google Maps se muestra en el footer y en la sección "Encuéntranos"
- [ ] El marcador está centrado en Calle Agentes IA 10, Madrid
- [ ] El botón "Cómo llegar" abre Google Maps con navegación activa hacia la tienda

### Diseño responsivo
- [ ] La web es completamente funcional en móvil, tablet y escritorio
- [ ] El menú hamburguesa funciona correctamente en móvil
- [ ] Las tarjetas de producto muestran el efecto hover correctamente

---

## 7. Casos edge

| Caso | Comportamiento esperado |
|------|------------------------|
| Carrusel con menos de 3 productos más vendidos | Se muestran los disponibles sin romper el layout |
| Habitáculo sin productos en stock | La sección se oculta de la home hasta tener productos disponibles |
| Filtros combinados sin resultados | Se muestra mensaje "Sin resultados" con opción de limpiar todos los filtros |
| Usuario accede a una página de número mayor al total disponible | Redirige a la última página disponible |
| Google Maps no carga (sin conexión o API key caducada) | Se muestra la dirección en texto plano con enlace externo como fallback |
| Imagen de producto no disponible | Se muestra imagen de placeholder con el logo Juvise |
| Admin no ha configurado ningún banner promocional | El espacio del banner no se renderiza; el layout no deja huecos vacíos |

---
## Changelog

| Versión | Fecha | Descripción del cambio |
|---------|-------|------------------------|
| v1 | 2026-06-17 | Creación inicial — identidad Juvise, home, carrusel, secciones por habitáculo, filtros, paginación, Google Maps, estilo vanguardista |
