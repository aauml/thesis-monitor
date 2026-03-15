# WEBAPP-MEMORY — Memoria de desarrollo del Thesis Dashboard

**Archivo:** `pwa/WEBAPP-MEMORY.md`
**Propósito:** Registro acumulativo de decisiones de diseño, estructura técnica, y evolución del dashboard. Consultar SIEMPRE antes de hacer cambios, mejoras o actualizaciones al webapp.

---

## 1. ARQUITECTURA

### Archivos y ubicación
```
thesis/
├── dashboard.html          # Entrada principal (URL fija, no mover)
├── sw.js                   # Service worker (DEBE estar en raíz — scope técnico)
├── pwa/
│   ├── manifest.json       # PWA manifest
│   ├── icon-192.png        # Icono Android/PWA
│   ├── icon-512.png        # Icono splash/large
│   ├── apple-touch-icon.png # Icono iOS (180px)
│   └── WEBAPP-MEMORY.md    # Este archivo
└── index.html              # KB webapp (proyecto separado, no tocar desde PM)
```

### PWA
- **Display:** standalone (pantalla completa, sin barra Safari)
- **Theme color:** `#1a3a5c` (accent del dashboard)
- **Scope:** `/thesis/` (cubre dashboard.html e index.html)
- **Cache strategy:** Stale-while-revalidate. Sirve cache inmediatamente, actualiza en background. Nombre del cache: `thesis-v1`.
- **Para forzar actualización en todos los dispositivos:** cambiar el nombre del cache en `sw.js` (ej: `thesis-v1` → `thesis-v2`).
- **Restricción de scope:** `sw.js` DEBE vivir en la raíz. Un SW solo controla rutas de su directorio hacia abajo. Si se mueve a `pwa/sw.js`, pierde control de `dashboard.html`.

### Icono
- Fondo navy (`#1a3a5c`), esquinas redondeadas, barra accent izquierda (`#4a7aac`), texto "TD" blanco.
- Generado con Pillow. Si hay que regenerar, usar el script `map_final.py` como referencia de estilo.

---

## 2. DESIGN SYSTEM

### CSS Variables (definidas en `:root`)
```css
--text: #2c2c2c;
--text-light: #606060;
--text-muted: #8a8a8a;
--accent: #1a3a5c;        /* navy principal */
--accent-soft: #e8eef4;   /* fondo claro accent */
--accent-mid: #4a7aac;    /* accent intermedio */
--border: #d4d4d4;
--bg: #fdfdfd;
--serif: 'Source Serif 4', Georgia, serif;
--sans: 'DM Sans', system-ui, sans-serif;
--mono: 'JetBrains Mono', 'Fira Mono', 'Consolas', monospace;
--warn-bg: #fffbe6; --warn-border: #e8d060; --warn-text: #5a4500;
--success-bg: #e6f4ea; --success-text: #2d7a3f;
--info-bg: #e8f0fe; --info-text: #1a56db;
```

### Tipografía
- **Headings:** Source Serif 4 (serif), 700 weight
- **Body:** DM Sans (sans-serif)
- **Code/monospace:** JetBrains Mono

### Colores por componente del mapa conceptual
| Componente | Color principal | Background | Border |
|---|---|---|---|
| Teórica | `#534ab7` | `#eeedfe` | `#afa9ec` |
| Metodológica | `#6b6b68` | `#f1efe8` | `#b4b2a9` |
| RIA (UE) | `#7f3f98` (púrpura) | `#f3ecf7` | `#c9a5d8` |
| NIST AI RMF | `#0f6e56` (verde) | `#e1f5ee` | `#5dcaa5` |
| Evaluativa | `#993c1d` (terracota) | `#faece7` | `#f0997b` |
| Caso de estudio | `#854f0b` (ámbar) | `#faeeda` | `#fac775` |
| Eje procesal | `#0f6e56` | `#e1f5ee` | `#5dcaa5` |
| Fundamentación (contenedor) | — | `#f8f7f4` | `#c8c7c2` |

### Colores de clasificación de la matriz
| Tipo | Color | Significado |
|---|---|---|
| Coincidencia | `#2d7a3f` (verde) | Control comparable en el NIST |
| Diferencia | `#b03030` (rojo) | Sin correspondencia o insuficiente |
| Complemento | `#185fa5` (azul) | Enfoques que se refuerzan mutuamente |
| Punto activo (pendiente) | `#2c2c2a` (negro) | Correspondencia existe, sin clasificar |
| Punto inactivo | `#e8e8e4` (gris claro) | Sin correspondencia |

**Nota:** Los puntos son negros neutros hasta que la Fase 1 los clasifique. Cuando se clasifiquen, cambiar la clase `mtx-on` por `mtx-co`, `mtx-di`, o `mtx-cp` y agregar los CSS correspondientes.

---

## 3. ESTRUCTURA DEL DASHBOARD

### Secciones en página
1. **Header:** Título, botones (Mapa conceptual, KB Report), meta
2. **Briefing de Estado:** Dónde está el doctorando, qué sigue
3. **Progreso:** Timeline visual (Q1–Q4 2026, 2027, 2028) + Siguientes Pasos
4. **Hoja de Ruta:** Lecturas activas con contexto, milestones
5. **Obsidian Digest:** Notas del vault con sugerencias
6. **Registro de Decisiones:** Colapsable, cronológico inverso

### Modales
1. **Mapa Conceptual** (`#mapModal`): Estructura completa de la tesis (ver §4)
2. **KB Intelligence Report** (`#kbModal`): Panorama del Knowledge Base

### Modal compartido CSS
- Max-width: `820px` (general), `880px` para `#mapModal`
- Header sticky, overlay con blur, cierre con Escape/click-fuera
- Clase base: `.modal-overlay` → `.modal` → `.modal-header` + `.modal-body`

---

## 4. MAPA CONCEPTUAL — Diseño v6

### Flujo vertical (orden de lectura)
```
Banner epistemológico
  ↓
Pregunta central
  ↓  "fundamentan y delimitan"
Fundamentación [teórica 2×2 | metodológica 2×2+1]
  ↓
Capa analítica [RIA púrpura | Matriz puntos | NIST verde]
  ↓  "hallazgos del análisis"
Capa evaluativa [4 criterios con iconos]
  ↓  "se aplican al"
Caso de estudio [pendiente + criterios de selección]
```

### Decisiones de diseño (historial)

**RIA en púrpura, NIST en verde** — Originalmente ambos eran azules, lo que los confundía con la clasificación "Complemento" (también azul). Se asignaron colores propios para evitar ambigüedad.

**Puntos de la matriz sólidos negros** — Versiones anteriores usaban puntos azules con centro blanco. El azul se confundía con "complemento". Negro neutro = correspondencia sin clasificar.

**Sin duplicación de artículos** — La v5 repetía los nombres de artículos tanto en el pilar RIA como en las filas de la matriz. Se eliminaron las etiquetas de fila en la matriz; los artículos se leen del pilar izquierdo por alineación vertical.

**Leyenda como pie de matriz, alineada a izquierda** — Versiones anteriores la ponían centrada entre los pilares, donde se confundía con un nodo de conexión. Ahora es un bloque vertical al pie de la tabla con descripciones.

**Teórica y metodológica visualmente iguales** — La teórica usa grid 2×2 de tarjetas; la metodológica replica el mismo formato (2×2 + 1 tarjeta full-width). Ambas dentro del contenedor "Fundamentación".

**Fundamentación soporta todo** — No una capa a cada pilar. Teórica y metodológica fundamentan el análisis completo (ambos pilares + matriz + evaluación + caso).

**Evaluativa como barra horizontal** — No envuelve todo el estudio. Vive entre la analítica y el caso. Es una rúbrica derivada de la teórica que mide los hallazgos. Los 4 criterios son dimensiones de evaluación, no pilares teóricos.

**Caso como pendiente** — Sin candidatos específicos. Solo criterios de selección del plan de investigación: alto riesgo, documentación pública, relevancia procesal.

**Clasificación pendiente (Fase 1)** — Los puntos negros se clasificarán como coincidencia/diferencia/complemento durante la Fase 1 de la investigación. La leyenda indica explícitamente que es "pendiente — producto de la Fase 1".

### Relaciones cruzadas
- **Contestabilidad = doble rol:** pilar teórico (#4) + criterio evaluativo. Anotada con línea punteada en el margen izquierdo de las versiones PNG; en HTML con la nota "Criterios derivados de la capa teórica (pilares 1–4)" al pie de la evaluativa.
- **Eje procesal:** pilares 3 y 4 comparten perspectiva procesal. Anotado con badge verde dentro de la capa teórica.

### CSS del mapa (prefijo `map-`)
Todas las clases del mapa usan el prefijo `map-` para no colisionar con el resto del dashboard. Referencia completa en el `<style>` de `dashboard.html`, bloque `/* ── Conceptual Map v6 ── */`.

Clases principales:
- `.map-question`, `.map-arrow` — pregunta central y conectores
- `.map-foundation`, `.map-found-grid`, `.map-found-block` — contenedor de fundamentación
- `.map-teo`, `.map-met` — bloques teórica/metodológica
- `.map-card-grid`, `.map-card` — tarjetas internas
- `.map-analytical`, `.map-ana-layout` — capa analítica (grid 3 columnas)
- `.map-pillar`, `.map-ria`, `.map-nist` — pilares normativos
- `.map-matrix`, `.map-mtx-table`, `.mtx-dot`, `.mtx-on`, `.mtx-off` — matriz de puntos
- `.map-legend`, `.lg-co`, `.lg-di`, `.lg-cp` — leyenda de clasificación
- `.map-evaluativa`, `.map-eva-grid`, `.map-eva-card`, `.map-eva-icon` — evaluativa
- `.map-case`, `.map-case-badge`, `.map-case-impl` — caso de estudio

---

## 5. RESPONSIVE

### Breakpoints
- **>700px:** Layout completo. Analítica en 3 columnas, evaluativa en 4, tarjetas en 2×2.
- **≤700px:** Fundamentación apila verticalmente. Analítica scroll horizontal (preserva relación RIA|Matriz|NIST con hint "← deslizar →"). Evaluativa 2×2. Descripciones de pilares se ocultan.
- **≤380px:** Todo a 1 columna.

### Decisión: scroll horizontal para la analítica en móvil
Se evaluaron tres opciones: (1) apilar todo verticalmente — pierde la relación visual entre RIA, matriz y NIST; (2) tabla responsiva — no funciona bien con puntos; (3) scroll horizontal — preserva la relación intacta. Se eligió la opción 3 con un hint visual.

---

## 6. CHANGELOG

| Fecha | Versión | Cambio |
|---|---|---|
| 2026-03-15 | v1 (pmlog) | PM Log original con árboles ASCII en `<pre>` dentro de modal |
| 2026-03-15 | v6 (mapa) | Rediseño completo: tarjetas expandibles, matriz de puntos, colores por componente, flujo vertical corregido |
| 2026-03-15 | PWA | manifest.json, sw.js, iconos. Display standalone, cache offline |
| 2026-03-15 | Reorg | Archivos PWA a `pwa/`, docs a ubicaciones existentes |

---

## 7. PRÓXIMOS CAMBIOS PREVISTOS

- **Clasificar puntos de la matriz:** Cuando la Fase 1 produzca resultados, cambiar puntos negros por verde/rojo/azul. Agregar clases CSS `.mtx-co { background: #2d7a3f; }`, `.mtx-di { background: #b03030; }`, `.mtx-cp { background: #185fa5; }`.
- **Actualizar caso de estudio:** Cuando se confirme con el director, reemplazar los criterios genéricos con el caso específico y sus detalles.
- **Cache versioning:** Al hacer cambios significativos al dashboard, incrementar `thesis-v1` → `thesis-v2` en `sw.js` para forzar recarga en dispositivos.
