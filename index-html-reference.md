# index.html — Reference Log
**Archivo:** `index.html` en https://github.com/aauml/thesis
**GitHub Pages:** https://aauml.github.io/thesis/
**Última actualización de este log:** 2026-03-08

---

## Qué es este archivo

`index.html` es la Knowledge Base webapp de la tesis. Muestra las entradas del **NewsLog** (Google Sheet) en forma de tarjetas filtrables. Es un archivo HTML/CSS/JS autocontenido (sin dependencias externas, sin build process) que se sirve desde GitHub Pages.

---

## Arquitectura general

```
GitHub Pages (index.html)
       │
       │  fetch() — CORS OK (API devuelve Access-Control-Allow-Origin: *)
       ▼
Apps Script Web App (WebApp.gs, deployment v28)
       │
       ▼
Google Sheet → pestaña NewsLog (1,041+ entradas)
```

### Carga de datos

El webapp carga **todos los datos de una vez** al inicio (`limit=2000&offset=0`). Los datos se cachean en `_cache` (variable global). Todo el filtrado, ordenamiento y paginación ocurre en el browser (client-side). Esto resuelve las limitaciones del API (ver sección "Limitaciones del API" más abajo).

```javascript
// Petición única al cargar la página
GET ?action=getAll&limit=2000&offset=0
→ { rows: [...1041 entries...], total: 1041 }
```

### Flujo de datos

```
loadAllRows()          → fetch API, guarda en _cache
fetchData()            → loadAllRows() → filter → sort → paginate → render
refresh()              → borra _cache, llama fetchData()
toggleStar()           → fetch updateByUrl (GET), actualiza _cache y state.rows
```

---

## Limitaciones conocidas del API (WebApp.gs)

Documentadas mediante pruebas directas con curl (2026-03-08):

| Parámetro | Soportado server-side | Notas |
|---|---|---|
| `limit` | ✅ Sí | Funciona. Máx testeado: 2000 (devuelve todos los rows) |
| `offset` | ✅ Sí | Funciona para paginación |
| `importance` | ✅ Sí (parcial) | Filtra rows correctamente PERO `total` siempre devuelve el total global (1041), nunca el total filtrado |
| `capa` | ✅ Sí (parcial) | Igual que importance — filtra rows pero `total` no refleja el filtro |
| `search` | ❌ No | Ignorado por el API. Se hace client-side |
| `sort` | ❌ No | Ignorado por el API. Se hace client-side |
| `capa_detail` | ❌ No | Ignorado por el API. Se hace client-side |
| `callback` (JSONP) | ❌ No | El API devuelve JSON plano, no wrappea en callback. Usar fetch() |

**Consecuencia:** Como `total` siempre es 1041, la paginación server-side no puede ser confiable cuando hay filtros activos. Por eso se carga todo y se filtra client-side.

### Formato de respuesta del API

```json
{
  "rows": [ { ...campos del NewsLog... } ],
  "total": 1041
}
```

**No incluye:** `total_pages`, `has_next`, `has_prev`, `stats` (breakdown por importancia), ni `page`.

### Valores de campos a tener en cuenta

- `starred`: El Sheet devuelve el **entero** `1` (no el string `"1"`). Comparar con `==` (loose equality) o verificar ambos tipos.
- `capa`: Algunas entradas tienen acentos (`"Teórica"`) y otras no (`"Teorica"`). Usar `normStr()` para comparaciones.
- `tier`: Puede ser número (`2`) o string (`"2"`). El webapp lo muestra directamente.

---

## Funciones principales del JS

### `loadAllRows()`
Fetches todos los rows del NewsLog una sola vez y los cachea en `_cache`. Calcula los stats de importancia (Alta/Media/Baja) a partir de los datos.

**Primera llamada:** hace el fetch. **Llamadas subsecuentes:** devuelve el caché.

Para forzar un re-fetch (e.g., el usuario clickea Refresh): poner `_cache = null` antes.

### `fetchData()`
El corazón de la webapp. Flujo:
1. Llama `loadAllRows()` (usa caché si existe)
2. Aplica filtros en orden: `importance → capa → capa_detail → evaluativa → content_type → search → starFilter`
3. Ordena según `state.sort`
4. Pagina: corta el array con `slice(start, start + PAGE_LIMIT)`
5. Actualiza `state` (total, total_pages, has_next, has_prev)
6. Renderiza header, cards y paginación

### `normStr(s)`
Normaliza acentos para comparaciones robustas: convierte a lowercase y reemplaza vocales acentuadas. Se usa en los filtros de capa, búsqueda, y evaluativa.

### `toggleStar(event, url, cardStarBtnId)`
Escribe al Sheet via `action=updateByUrl` (GET request con campos en query string). Actualiza `_cache.rows` y `state.rows` inmediatamente (optimistic UI). Si el star filter está activo y se desestrella un item, llama `fetchData()`.

---

## Estado global (`state`)

```javascript
let state = {
  page: 1,          // página actual (1-indexed)
  search: "",       // texto de búsqueda
  importance: "",   // "" | "ALTA" | "MEDIA" | "BAJA"
  capa: "",         // valor display, e.g. "teórica"
  capaKey: "",      // key interno, e.g. "teorica"
  sort: "found-desc",
  starFilter: false,
  rows: [],         // rows de la página actual (slice del array filtrado)
  total: 0,         // total de items tras filtrar
  total_pages: 1,
  has_next: false,
  has_prev: false,
  loading: false,
  lastFetch: null,
};
```

---

## Historial de bugs y fixes

### Bug 1: `action=getPage` no existe — 2026-03-07
**Síntoma:** La webapp cargaba pero mostraba "Loading…" indefinidamente.
**Causa:** `buildUrl()` llamaba `action=getPage`, que no existe en WebApp.gs v28.
**Fix:** Cambiar a `action=getAll` con `offset=(page-1)*limit`.

### Bug 2: JSONP sin soporte server-side — 2026-03-07
**Síntoma:** Sin datos a pesar de usar `action=getAll`.
**Causa:** La webapp usaba JSONP (inyección de `<script src="...&callback=_cb_xyz">`). El API ignora `callback` y devuelve JSON plano. El browser ejecutaba `{"rows":...}` como JavaScript → SyntaxError silencioso.
**Fix:** Reemplazar `jsonp()` con `fetch()`. El API soporta CORS (`Access-Control-Allow-Origin: *`). El request hace un redirect 302→200; ambas respuestas tienen CORS headers.

### Bug 3: Filtros/search/sort/starred no funcionaban — 2026-03-08
**Síntoma:** Total siempre 1041, filtros no cambiaban resultados, starred vacío.
**Causas:**
- API devuelve `total: 1041` siempre (no filtrado)
- `search`, `sort`, `capa_detail` ignorados server-side
- Valores `starred` en el Sheet son entero `1`, no string `"1"` → `=== "1"` fallaba
- Filtro starred solo escaneaba 50 rows de la página actual

**Fix:** Cargar todos los datos una vez (`limit=2000`), cachear, filtrar/ordenar/paginar todo client-side.

---

## Modificar este archivo

### Agregar un filtro nuevo
1. Añadir el elemento UI en el HTML
2. En `fetchData()`, agregar el filtro en la sección "── Filter ──"
3. Si debe limpiar con "✕ Clear", añadirlo en `updateClearBtn()` y `clearAllFilters()`

### Agregar una columna nueva del NewsLog
El API devuelve todos los campos automáticamente. En `renderCard()`, acceder via `item.nombre_campo`.

### Agregar subcapas al dropdown
Modificar `SUBCAPA_OPTIONS`. Tiene keys: `all`, `teorica`, `analitica`, `evaluativa`, `metodologica`.

### Cambiar items por página
Modificar `PAGE_LIMIT` (actualmente `50`).

### Si el Apps Script cambia
Actualizar `API_URL`. Verificar que el nuevo deployment devuelva `{ rows: [...], total: N }` con `access-control-allow-origin: *`.

---

## Acceso y deployment

- **Webapp:** https://aauml.github.io/thesis/
- **Repo:** https://github.com/aauml/thesis
- **API:** `SHEET_API` en `config.txt` del proyecto (WebApp.gs, deployment v28)
- **Deploy:** Automático — push a `main` → GitHub Pages actualiza en ~1-2 min
