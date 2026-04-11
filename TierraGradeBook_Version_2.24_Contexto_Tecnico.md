# TierraGradeBook — Contexto Técnico Completo
**Versión del código:** v2.24  
**Fecha:** Abril 2026  
**Archivo principal:** `index.html` (archivo único, ~2700 líneas)

---

## 1. Descripción del Sistema

TierraGradeBook es una aplicación web de gestión de notas universitarias organizada por **Resultados de Aprendizaje (RA)** y **Criterios de Evaluación (CE)**. Permite importar notas desde CSV de Moodle, cargar notas manuales con prioridad sobre las importadas, y generar reportes en PDF y Excel coloreados por nivel de desempeño.

**Usuarios:** Solo profesores. El acceso se controla manualmente por el administrador en Firebase. Los alumnos no tienen acceso a la app.

**Instancia activa:**
- Materia: `Análisis de Sistemas de Información - FRCU UTN`
- Año: 2026
- ID Firebase: `ANALISIS_2026`
- Segunda materia: `SIMULACION_2026`
- Admin: `bonninmiguel@gmail.com`
- Co-profe: `mvansorena@gmail.com`

---

## 2. Stack Tecnológico

| Componente | Tecnología | Versión |
|---|---|---|
| Frontend | HTML + CSS + JavaScript puro (sin frameworks) | — |
| Módulos JS | ES Modules (`<script type="module">`) | — |
| Base de datos | Firebase Realtime Database | Plan Spark (gratuito) |
| Autenticación | Firebase Auth — Google Sign-In | 10.12.0 |
| Hosting | GitHub Pages | — |
| PDF | jsPDF + jsPDF-autoTable | 2.5.1 / 3.8.2 |
| Tipografía | Google Fonts: Playfair Display + IBM Plex Sans | — |

**Por qué no hay frameworks:** La app es un único `index.html` desplegado en GitHub Pages sin proceso de build. Mantener todo en un solo archivo simplifica el deploy — basta con subir el archivo al repositorio.

---

## 3. URLs e Infraestructura

```
App:        https://bonninmigueldamian.github.io/tierragradebook/
Repo:       https://github.com/bonninmigueldamian/tierragradebook
Firebase:   https://console.firebase.google.com → proyecto: tierragradebook
DB URL:     https://tierragradebook-default-rtdb.firebaseio.com
```

**Deploy:** Subir `index.html` a la raíz del repositorio en GitHub. GitHub Pages publica automáticamente. No hay proceso de build, compilación ni dependencias locales.

---

## 4. Firebase Configuration

```javascript
const firebaseConfig = {
  apiKey: "AIzaSyAGVal6rv6EN7qGgubdJTKX7WRKwFmMASg",
  authDomain: "tierragradebook.firebaseapp.com",
  databaseURL: "https://tierragradebook-default-rtdb.firebaseio.com",
  projectId: "tierragradebook",
  storageBucket: "tierragradebook.firebasestorage.app",
  messagingSenderId: "448999535428",
  appId: "1:448999535428:web:6c2ab5eca4a93c6bc46736"
};
```

> **Nota de seguridad:** El `apiKey` de Firebase es público por diseño. La seguridad real la proveen las Security Rules en Firebase, no las credenciales de conexión. Cualquiera puede ver el `apiKey` en el código fuente del navegador — eso es normal e inevitable en apps web.

---

## 5. Firebase Security Rules (actuales)

```json
{
  "rules": {
    "materias": { ".read": "auth != null", ".write": false },
    "permisos": { ".read": "auth != null", ".write": false },
    "datos": {
      "$materiaId": {
        ".read": "auth != null && root.child('permisos').child($materiaId).child(auth.token.email.replace('.','_').replace('.','_').replace('.','_').replace('.','_')).exists()",
        ".write": "auth != null && root.child('permisos').child($materiaId).child(auth.token.email.replace('.','_').replace('.','_').replace('.','_').replace('.','_')).exists()"
      }
    }
  }
}
```

**Explicación de la regla:**
- `materias` y `permisos`: legibles por cualquier usuario autenticado con Google. Necesario porque la app lee `/permisos/` para saber a qué materias tiene acceso antes de poder verificar permisos.
- `datos`: solo accesible si el email del usuario autenticado aparece como key en `/permisos/{materiaId}/`.
- Los emails se guardan en Firebase con `_` reemplazando los `.` (Firebase no acepta puntos en keys). La regla encadena 4 `.replace('.','_')` porque Firebase Rules no soporta regex con flags. Cubre hasta 4 puntos en el email (ej: `pablo.colombo@gmail.com` → `pablo_colombo@gmail_com`).
- Nadie puede modificar `/permisos/` desde la app (`.write: false`). Solo el administrador desde la consola de Firebase.

---

## 6. Estructura de Datos en Firebase

### IMPORTANTE: IDs con caracteres especiales

Firebase **no acepta puntos (`.`) en los keys** de los nodos. Tampoco acepta otros caracteres especiales como `$`, `#`, `[`, `]`, `/`.

Los IDs de RA y CE tienen puntos en su definición original (ej: `C.E.01.01`, `R.A.01`). La solución:
- El **key** de Firebase usa guiones bajos en lugar de puntos: `CE_01_01`, `RA01`
- Dentro del nodo, el campo **`id`** guarda el valor original para mostrar: `"C.E.01.01"`
- La función `objArr()` agrega `_key` al objeto para que el código pueda referirse al key de Firebase

Lo mismo aplica para los emails en `/permisos/`: `pablo_colombo@gmail_com: true`

```
/materias/
  {materiaId}: { nombre: "Análisis de Sistemas de Información - FRCU UTN" }
  // Ej: ANALISIS_2026: { nombre: "..." }

/permisos/
  {materiaId}/
    {email_con_guiones_bajos}: true
    // Ej: bonninmiguel@gmail_com: true
    // Ej: pablo_colombo@gmail_com: true
    // Los puntos del email se reemplazan por _ (Firebase no acepta . en keys)

/datos/{materiaId}/

  config/
    materia: "Análisis de Sistemas de Información - FRCU UTN"
    año: 2026

  alumnos/
    {key_autogenerado}/
      apellido: "GARCIA"        // siempre en MAYÚSCULAS (lo convierte al importar)
      nombre: "JUAN PABLO"      // siempre en MAYÚSCULAS
      email: "juan.garcia@gmail.com"
      legajo: "14150216"
      año: 2026
      grupo: 6                  // 0 = sin grupo

  ra/
    {key}/                      // key = ID sin puntos (ej: RA01)
      id: "RA01"                // valor a mostrar (igual al key en este caso)
      nombre: "Reconoce los distintos interesados en el proceso de desarrollo"
      peso: 0.3

  ce/
    {key}/                      // key = ID con _ en lugar de . (ej: CE_01_01)
      id: "CE_01_01"            // en este caso el id y el key son iguales
      nombre: "C.E.01.01. Reconoce los distintos interesados..."  // nombre completo descriptivo
      raId: "RA01"              // referencia al RA padre
      peso: 0.3

  ev/                           // Instancias de evaluación
    {key}/                      // key = código corto (ej: PT1, PP1, R1PT1)
      nombre: "Parcial Teorico 1"

  niveles/
    {code}/                     // P, B, A, V (configurables desde la UI)
      code: "P"
      nombre: "Principiante"
      desde: 0
      hasta: 3.99

  notas/
    {key_autogenerado}/
      email: "juan.garcia@gmail.com"   // email del alumno (con puntos originales)
      raId: "RA01"                     // key del RA
      ceId: "CE_01_01"                 // key del CE
      evId: "PT1"                      // key de la instancia
      nota: 8.33                       // convertida a escala /10
      notaoriginal: 1.67               // valor exacto del CSV (escala original)
      notamanual: null                 // null = no hay; número = tiene prioridad sobre nota
      ts: 1712345678000                // timestamp Unix

  etiquetas/
    {key_autogenerado}/
      tipo: "Accion"            // nombre del conjunto (ej: Accion, Estado)
      nivel_code: "P"           // código del nivel al que aplica
      etiqueta: "Recuperar"     // texto a mostrar en celdas de reportes
```

---

## 7. Arquitectura del Código

### 7.1 El Problema Fundamental de ES Modules

La app usa `<script type="module">`. En módulos ES, **todas las funciones tienen scope local al módulo** y no son accesibles desde el scope global (`window`). Los atributos HTML `onclick="miFuncion()"` buscan en `window` y no las encuentran → `ReferenceError`.

**Solución obligatoria:** Toda función que sea llamada desde HTML (tanto atributos inline como HTML generado dinámicamente) debe ser explícitamente expuesta al scope global:

```javascript
// Al final del módulo, antes del cierre de </script>:
window.miFuncion = miFuncion;

// O definida directamente en window:
window.miFuncion = async function() { ... };
```

**Diagnóstico rápido:** Si un botón no responde y la consola muestra `ReferenceError: X is not defined`, la causa es que `window.X` no está expuesta. Es el error más frecuente en esta app.

### 7.2 Patrón de Renderizado (innerHTML + addEventListener)

La app no usa frameworks — todo el HTML dinámico se genera como strings y se asigna via `innerHTML`. El patrón estándar:

```javascript
function renderAlgo() {
  // 1. Generar HTML como string y asignar
  document.getElementById('panel').innerHTML =
    '<button id="mi-btn">Click</button>';

  // 2. DESPUÉS del innerHTML, adjuntar eventos
  document.getElementById('mi-btn').addEventListener('click', function() {
    // handler
  });
}
```

**Nunca usar onclick inline** en el HTML generado:
```javascript
// ❌ INCORRECTO - no funciona en ES modules
'<button onclick="miFuncion()">Click</button>'

// ✅ CORRECTO - addEventListener después del render
document.getElementById('btn').addEventListener('click', miFuncion);

// ✅ TAMBIÉN CORRECTO si miFuncion está en window
'<button onclick="window.miFuncion()">Click</button>'
```

### 7.3 Event Delegation para Evitar Stacking

Cuando un panel se re-renderiza múltiples veces (ej: cada vez que se selecciona un filtro), los listeners se acumulan causando que haya que hacer múltiples clicks para cerrar modales. Solución con flag:

```javascript
const panel = document.getElementById('panel-importar');
if(!panel._delegated) {
  panel._delegated = true;
  panel.addEventListener('click', function(e) {
    const btn = e.target.closest('[data-action]');
    if(!btn) return;
    const { action, ra, ce, ev } = btn.dataset;
    if(action === 'edit-notas') window.openEditNotas(ra, ce, ev);
    if(action === 'del-notas')  window.delGrpNotas(ra, ce, ev);
  });
}
```

El flag `_delegated` persiste en el elemento DOM entre re-renders porque se asigna al objeto DOM, no al innerHTML.

### 7.4 Helpers de Base de Datos

Todas las operaciones Firebase pasan por estos wrappers que muestran el indicador de guardado:

```javascript
function dbRef(path) {
  return ref(db, `datos/${CURRENT_MATERIA.id}/${path}`);
}

async function dbSet(path, val)    { showSaving(true); await set(dbRef(path), val);    showSaving(false); }
async function dbPush(path, val)   { showSaving(true); const r = await push(dbRef(path), val); showSaving(false); return r.key; }
async function dbRemove(path)      { showSaving(true); await remove(dbRef(path));       showSaving(false); }
async function dbUpdate(path, val) { showSaving(true); await update(dbRef(path), val);  showSaving(false); }
```

**CRÍTICO:** Siempre usar estos wrappers, nunca llamar a `set()`, `push()`, etc. de Firebase directamente. Aseguran que el indicador visual de guardado funcione.

### 7.5 Variables de Estado Global

```javascript
// ── Sesión ────────────────────────────────────────────────
let CURRENT_USER    = null;   // objeto usuario Firebase autenticado
let CURRENT_MATERIA = null;   // { id: "ANALISIS_2026", nombre: "..." }
let USER_MATERIAS   = [];     // array de materias accesibles para el usuario
let DB_LISTENERS    = [];     // refs de listeners Firebase activos (para detach)

// ── Espejo local de Firebase ──────────────────────────────
let DB = {
  config:    {},  // { materia, año }
  alumnos:   {},  // { [key]: { apellido, nombre, email, legajo, año, grupo } }
  ra:        {},  // { [key]: { id, nombre, peso } }
  ce:        {},  // { [key]: { id, raId, nombre, peso } }
  ev:        {},  // { [key]: { nombre } }
  niveles:   {},  // { [code]: { code, nombre, desde, hasta } }
  notas:     {},  // { [key]: { email, raId, ceId, evId, nota, notaoriginal, notamanual, ts } }
  etiquetas: {},  // { [key]: { tipo, nivel_code, etiqueta } }
};

// ── UI State ──────────────────────────────────────────────
let currentTab  = 'inicio';
let alumSearch  = '';          // texto de búsqueda en tab Alumnos
let alumGrupo   = 'todos';     // filtro de grupo en tab Alumnos

// ── Importar CSV State ────────────────────────────────────
let impRA       = '';          // key del RA seleccionado
let impCE       = '';          // key del CE seleccionado
let impEV       = '';          // key de la instancia seleccionada
let impStep     = 1;           // paso actual del wizard (1, 2, 3)
let impFile     = null;        // objeto File del CSV seleccionado
let impPreview  = [];          // array de filas procesadas para preview

// ── Reportes State ────────────────────────────────────────
let repType      = 'instancia'; // 'instancia' | 'race' | 'nivel'
let repEV        = '';          // instancia seleccionada en reporte 1
let repLevels    = [];          // niveles seleccionados en reporte 3
let repNivelMode = 'any';       // 'any' | 'all' para reporte 3
let repCellMode  = {};          // config de celdas (qué mostrar por nivel) — persistida en localStorage

// ── Datos temporales para PDF/Excel ──────────────────────
// Se llenan al generar el reporte y se usan por los botones de export
window._lastInstStudents  = [];
window._lastInstCeIds     = [];
window._lastRaceStudents  = [];
window._lastRaceCeIds     = [];
window._lastNivelStudents = [];
window._lastNivelCeIds    = [];
```

---

## 8. Funciones Clave — Implementación

### 8.1 objArr(obj)

```javascript
function objArr(obj) {
  if(!obj) return [];
  return Object.entries(obj).map(([k, v]) => Object.assign({}, v, { _key: k }));
}
```

Convierte un objeto Firebase `{ key1: {datos}, key2: {datos} }` en un array `[{datos, _key:'key1'}, {datos, _key:'key2'}]`. Es la función más usada en toda la app — toda iteración sobre datos de Firebase la pasa por aquí.

### 8.2 notaEfectiva(n)

```javascript
function notaEfectiva(n) {
  if(n.notamanual !== null && n.notamanual !== undefined && n.notamanual !== '')
    return parseFloat(n.notamanual);
  return parseFloat(n.nota) || 0;
}
```

**REGLA ABSOLUTA:** Toda operación que use la nota de un alumno (reportes, promedios, dashboard, exportaciones, PDF) DEBE usar `notaEfectiva(n)` en lugar de `n.nota`. Si `notamanual` tiene cualquier valor numérico (incluyendo 0), ese valor tiene prioridad absoluta. `n.nota` solo se usa para mostrar la nota original del CSV.

### 8.3 getBestNotaEfectiva(email, raId, ceId)

```javascript
function getBestNotaEfectiva(email, raId, ceId) {
  const gs = objArr(DB.notas).filter(n =>
    String(n.email||'').toLowerCase() === email.toLowerCase() &&
    n.raId === raId && n.ceId === ceId
  );
  if(!gs.length) return null;
  return Math.max(...gs.map(n => notaEfectiva(n)));
}
```

Retorna la mejor nota efectiva del alumno para una combinación RA+CE considerando **todas** las instancias de evaluación. Usada en el reporte "Mejor Nota por RA/CE". Si no hay notas, retorna `null`.

### 8.4 getNivelArr() y getNivel(nota)

```javascript
function getNivelArr() {
  return objArr(DB.niveles).sort((a, b) => a.desde - b.desde);
}

function getNivel(nota) {
  if(nota === null || nota === undefined || isNaN(nota)) return null;
  return getNivelArr().find(l => nota >= l.desde && nota <= l.hasta);
}
```

Los niveles se cargan desde Firebase (son configurables). `getNivel()` retorna el objeto de nivel completo `{ code, nombre, desde, hasta }` o `null` si la nota no entra en ningún rango.

### 8.5 sortAlumnos(arr)

```javascript
function sortAlumnos(arr) {
  return [...arr].sort((a, b) =>
    (a.apellido + a.nombre).localeCompare(b.apellido + b.nombre, 'es')
  );
}
```

Ordena por apellido+nombre en español (respeta ñ, acentos, etc.).

### 8.6 tgbConfirm({ title, msg, okLabel, danger })

```javascript
function tgbConfirm({ title='¿Confirmar?', msg='', okLabel='Aceptar', danger=false } = {}) {
  return new Promise(resolve => {
    // Crea overlay con modal de confirmación
    // Resuelve true si acepta, false si cancela
  });
}
```

Reemplaza `window.confirm()` con un modal visual consistente con el diseño de la app. Se usa con `await`:

```javascript
const ok = await tgbConfirm({ title: 'Eliminar', msg: '¿Seguro?', okLabel: 'Eliminar', danger: true });
if(!ok) return;
```

### 8.7 toast(msg, type)

```javascript
function toast(msg, type='ok') {
  // type: 'ok' (verde) | 'err' (rojo) | 'warn' (amarillo)
  // Aparece en la esquina inferior derecha por 3 segundos
}
```

Notificación no bloqueante. Usada para confirmar acciones exitosas y reportar errores menores.

### 8.8 openModal(id, html) y closeModal(id)

```javascript
function openModal(id, html) {
  // Busca elemento con ese id y coloca el HTML dentro
  // Si ya existe un modal previo en el mismo id, lo elimina primero
}
function closeModal(id) {
  const el = document.getElementById(id);
  if(el) el.innerHTML = '';
}
```

Sistema de modales. El `id` suele ser `'config-modal'` (para modales de configuración) o `'alumnos-modal'`. CRÍTICO: `closeModal` debe estar expuesta como `window.closeModal`.

### 8.9 loadAllData()

```javascript
async function loadAllData() {
  const mid = CURRENT_MATERIA.id;
  const snap = await get(ref(db, `datos/${mid}`));
  const data = snap.val() || {};
  DB.config    = data.config    || { materia: CURRENT_MATERIA.nombre, año: new Date().getFullYear() };
  DB.alumnos   = data.alumnos   || {};
  DB.ra        = data.ra        || {};
  DB.ce        = data.ce        || {};
  DB.ev        = data.ev        || {};
  DB.notas     = data.notas     || {};
  DB.etiquetas = data.etiquetas || {};
  // Niveles: si no existen, crea defaults y los guarda
  if(data.niveles) { DB.niveles = data.niveles; }
  else { DB.niveles = getDefaultNiveles(); dbSet('niveles', DB.niveles); }
}
```

Carga todo el árbol de datos de la materia activa en una sola lectura. Se ejecuta al seleccionar materia. Los datos quedan en el objeto `DB` como espejo local — las operaciones CRUD actualizan tanto Firebase como `DB` localmente para evitar re-lecturas.

---

## 9. Tabs y Funciones de Renderizado

| Tab | Panel ID | Función render | Tab inicial |
|---|---|---|---|
| 🏠 Inicio | `panel-inicio` | `renderInicio()` | **Sí** |
| 👥 Alumnos | `panel-alumnos` | `renderAlumnos()` | No |
| 📋 Notas | `panel-importar` | `renderImportar()` | No |
| 📊 Reportes | `panel-reportes` | `renderReportes()` | No |
| ⚙️ Config | `panel-config` | `renderConfig()` | No |
| ❓ Ayuda | `panel-ayuda` | `renderAyuda()` | No |

```javascript
function switchTab(tab) {
  currentTab = tab;
  document.querySelectorAll('.panel').forEach(p => p.classList.remove('active'));
  document.querySelectorAll('.tab-btn').forEach(b => b.classList.remove('active'));
  document.getElementById('panel-'+tab).classList.add('active');
  document.querySelector(`[data-tab="${tab}"]`).classList.add('active');
  if(tab === 'inicio')    renderInicio();
  if(tab === 'alumnos')   renderAlumnos();
  if(tab === 'importar')  renderImportar();
  if(tab === 'reportes')  renderReportes();
  if(tab === 'config')    renderConfig();
  if(tab === 'ayuda')     renderAyuda();
}
```

---

## 10. Sistema de Notas — Diseño Completo

### 10.1 Los tres campos de nota

| Campo | Escala | Quién lo escribe | Cuándo cambia |
|---|---|---|---|
| `notaoriginal` | Escala original del CSV | `confirmImport` | Cada reimportación |
| `nota` | 1-10 | `confirmImport` | Cada reimportación |
| `notamanual` | 1-10 | `openEditNotas` | Solo el profe manualmente |

### 10.2 Lógica de nota efectiva

```
notaEfectiva =
  notamanual !== null  →  notamanual  (prioridad absoluta, incluyendo 0)
  notamanual === null  →  nota
```

### 10.3 Reimportación de CSV — Opción A

Al reimportar para la misma combinación RA+CE+EV:
1. Se borran **todas** las notas existentes de esa combinación (incluyendo `notamanual`)
2. Se insertan las notas del nuevo CSV con `notamanual: null`

```javascript
// En confirmImport:
const existentes = Object.entries(DB.notas||{})
  .filter(([,n]) => n.raId===impRA && n.ceId===impCE && n.evId===impEV)
  .map(([k]) => k);
for(const k of existentes) { await dbRemove(`notas/${k}`); delete DB.notas[k]; }
// Luego insertar nuevas...
const data = {
  email, raId: impRA, ceId: impCE, evId: impEV,
  nota: row.nota,
  notaoriginal: row.rawNota,
  notamanual: null,
  ts: Date.now()
};
```

### 10.4 Panel de edición de notas manuales

**Acceso:** Tab Notas → "Notas cargadas" → botón ✏️ Editar en la fila deseada

**Columnas:** Alumno | Grp | Orig.CSV | Nota(/10) | Nota Manual | Efectiva | Borrar(✕)

**Comportamiento del botón ✕ (borrar nota manual):**
- **No guarda inmediatamente** — solo limpia el campo visualmente y lo marca en `clearedKeys`
- La columna "Efectiva" se actualiza en tiempo real para mostrar el fallback a `nota`
- Al hacer click en "Guardar": procesa todos los inputs (guarda los con valor, borra los marcados como limpiados)

**Después de guardar:** Recarga las notas desde Firebase antes de reabrir el panel para garantizar datos frescos.

### 10.5 Conversión de escala al importar

```javascript
// Detecta columna y máximo del CSV de Moodle
const gradeCol = fields.find(h => /calificaci[oó]n\//i.test(h));
const maxNota  = gradeCol
  ? parseFloat(gradeCol.match(/\/(\d+[,.]\d*)/)[1].replace(',','.')) || 2
  : 2;
const nota = Math.round((rawNota / maxNota) * 10 * 100) / 100;
```

Ejemplo: CSV con nota `1.67` sobre un máximo de `2` → `nota = (1.67/2)*10 = 8.35`

---

## 11. Reportes

### 11.1 Tres tipos de reporte

| Tipo | Variable estado | Función generar | Almacena datos en |
|---|---|---|---|
| Por Instancia | `repEV` | `genInstResult()` | `window._lastInstStudents/CeIds` |
| Mejor Nota RA/CE | — | `genRaceResult()` | `window._lastRaceStudents/CeIds` |
| Filtro por Nivel | `repLevels`, `repNivelMode` | `genNivelResult()` | `window._lastNivelStudents/CeIds` |

El flujo es: el usuario configura filtros → click "Generar reporte" → la función genera la tabla HTML y almacena los datos raw en `window._lastXxx` → los botones PDF/Excel usan esos datos almacenados.

### 11.2 Panel "Mostrar en cada celda"

Configura qué aparece en cada celda de los reportes según el nivel del alumno. Persiste en `localStorage` como `repCellMode`.

```javascript
// Estructura de repCellMode:
{
  P: { nota: true,  types: ['Accion'] },
  B: { nota: false, types: ['Estado', 'Accion'] },
  A: { nota: true,  types: [] },
  V: { nota: true,  types: [] }
}
```

Funciones relacionadas:
- `loadCellMode()` / `saveCellMode()` — lee/escribe localStorage
- `getCellMode(code)` — retorna config para un nivel
- `toggleCellNota(code)` — activa/desactiva nota numérica para un nivel
- `toggleCellTipo(code, tipo)` — activa/desactiva una etiqueta para un nivel
- `renderCfgPanel()` — renderiza el panel visual de configuración

### 11.3 buildCell(nota)

```javascript
function buildCell(nota) {
  const nv   = getNivel(nota);
  const code = nv ? nv.code : 'N';
  const mode = getCellMode(code);
  const tipos = getEtqTipos();
  // Construye contenido de la celda según modo
  // Retorna '<td class="nota-cell">...</td>'
}
```

Construye el HTML de una celda del reporte aplicando colores por nivel y etiquetas configuradas. **SIEMPRE** llamar con `buildCell(notaEfectiva(n))`, nunca con `buildCell(n.nota)`.

### 11.4 generatePDF(reportType, reportDetail, students, ceIds)

```javascript
generatePDF(
  'Reporte Por Instancia',          // tipo (aparece como título)
  'PT1 - Parcial Teorico 1',        // detalle (aparece en subtítulo)
  students,                          // array de alumnos con _cells calculadas
  ceIds                              // array de keys de CE
)
```

Genera PDF A4 horizontal. Características:
- Header negro/dorado en **todas** las páginas (via `willDrawPage` — ejecuta antes que autoTable)
- `margin: { top: HDR_H+14 }` reserva espacio para el header en todas las páginas
- Nombres truncados a 26 caracteres para evitar wrapping
- Columna nombre: 46mm, Grp: 9mm, CE: calculado automáticamente (máx 30mm, mín 15mm)
- Celdas coloreadas por nivel igual que la UI
- Footer: fecha+hora a la izquierda, número de página a la derecha

### 11.5 exportTableToExcel(title, students, ceIds)

Genera CSV con BOM UTF-8 (compatible con Excel sin configuración). Estructura:
```
Fila 1: título del reporte
Fila 2: nombre de materia, año, fecha
Fila 3: vacía
Fila 4: encabezado RA (Apellido, Nombre, Grupo, RA01, "", "", RA02, ...)
Fila 5: encabezado CE ("", "", "", CE_01_01, CE_01_02, ...)
Filas 6+: datos de alumnos
```

Las celdas respetan la configuración del panel "Mostrar en cada celda" igual que los reportes en pantalla.

---

## 12. Niveles de Desempeño

**Son configurables** desde Config → Niveles de Desempeño → ✏️ Editar. Se guardan en Firebase y se cargan al iniciar. Si no existen en Firebase, se crean con los defaults.

**Defaults:**

| Code | Nombre | Desde | Hasta |
|---|---|---|---|
| P | Principiante | 0 | 3.99 |
| B | Básico | 4 | 6.99 |
| A | Autónomo | 7 | 8.99 |
| V | Avanzado | 9 | 10 |

**Colores en UI y PDF:**

| Code | Fondo | Texto |
|---|---|---|
| P | `#fee2e2` | `#991b1b` |
| B | `#fef3c7` | `#7c2d12` |
| A | `#dbeafe` | `#1e40af` |
| V | `#dcfce7` | `#14532d` |
| N (sin nota) | `#f3f4f6` | `#6b7280` |

Si se agregan niveles nuevos con códigos distintos a P/B/A/V, hay que agregar sus colores en las funciones `generatePDF` y `buildCell`.

---

## 13. Etiquetas

Las etiquetas son textos que aparecen en las celdas de los reportes según el nivel del alumno. Se configuran en Config → Etiquetas por Nivel.

**Tipos comunes:** Accion (ej: "Recuperar", "Mejorar"), Estado (ej: "Regular", "Promovido")

**Funciones:**
```javascript
function getEtqTipos() {
  // Retorna array de tipos únicos ["Accion", "Estado", ...]
}
function getEtq(tipo, nivel_code) {
  // Retorna el texto de la etiqueta para ese tipo y nivel, o ''
}
```

---

## 14. Paleta de Colores (CSS Variables)

```css
--navy:    #111111   /* Negro principal — fondos de header, tablas, botones */
--navy2:   #1c1c1c   /* Negro secundario */
--navy3:   #2a2a2a   /* Negro terciario */
--gold:    #c9a84c   /* Dorado principal — logo, acentos, tabs activos */
--gold2:   #e8d08a   /* Dorado claro — textos sobre fondo negro */
--bg:      #f0f2f7   /* Fondo general de la app */
--white:   #fff
--text:    #111111
--muted:   #6b7280   /* Textos secundarios */
--border:  #dde2ee   /* Bordes de cards y tablas */
--border2: #c8d0e2   /* Bordes de inputs */
--ok:      #16a34a   /* Verde — éxito */
--warn:    #d97706   /* Amarillo — advertencia */
--err:     #dc2626   /* Rojo — error */
```

**REGLA:** El sistema usa **negro con dorado**, nunca azul navy. Si se agregan elementos nuevos, usar `var(--navy)` para fondos oscuros. Nunca usar colores como `#1a2744`, `#243256`, `#0f1c38` u otros azules navy hardcodeados.

Los colores en el PDF se usan como RGB:
```javascript
const navy = [17, 17, 17];   // #111
const gold  = [201, 168, 76]; // #c9a84c
```

---

## 15. Favicon

El favicon es un SVG embebido directamente en el `<head>` como data URL:

```html
<link rel="icon" type="image/svg+xml" href="data:image/svg+xml;base64,...">
```

El SVG muestra el símbolo **ϕ** (phi) en dorado (#C9A84C) sobre fondo negro con bordes redondeados. Compatible con todos los browsers modernos. No requiere archivo externo.

---

## 16. Indicador de Guardado

```javascript
function showSaving(on) {
  document.getElementById('saving-bar').classList.toggle('show', on);
  document.getElementById('saving-overlay').classList.toggle('show', on);
}
```

Dos elementos visuales mientras se guarda en Firebase:
1. **Barra dorada** de 5px en el tope de la pantalla con animación de progreso
2. **Pill flotante** en el centro inferior: "💾 Guardando..."

Se activan automáticamente via los wrappers `dbSet/dbPush/dbRemove/dbUpdate`.

---

## 17. Funciones Expuestas a window (lista completa)

```javascript
// Auth y navegación
window.signInWithGoogle, window.signOut
window.switchTab, window._selectMateria, window.showMateria

// Alumnos
window.renderAlumnos
window.openAlumnoModal, window.saveAlumno, window.delAlumno
window.openImportListaModal, window.handleListaDrop, window.handleListaFile
window.confirmImportLista, window.readListaFile

// Notas / Importar CSV
window.renderImportar, window.parseCSV, window.confirmImport
window.delGrpNotas, window.renderGradesSummary, window.openEditNotas

// Reportes
window.renderReportes
window.renderRepInstancia, window.renderRepRACE, window.renderRepNivel
window.refreshRep, window.renderCfgPanel
window.genInstResult, window.genRaceResult, window.genNivelResult
window.toggleCellNota, window.toggleCellTipo
window.printInstancia, window.printRACE, window.printNivel
window.exportInstancia, window.exportRACE, window.exportNivel
window.buildRepTable

// Configuración
window.renderConfig, window.saveConfig
window.openRAModal, window.saveRA, window.delRA
window.openCEModal, window.saveCE, window.delCE
window.openEVModal, window.saveEV, window.delEV
window.openEtqModal, window.saveEtq, window.delEtq
window.openNivelModal, window.saveNivel
window.delAllNotas

// Pantallas y ayuda
window.renderAyuda, window.renderInicio

// Modales
window.closeModal

// Datos temporales para export/PDF
window._lastInstStudents, window._lastInstCeIds
window._lastRaceStudents, window._lastRaceCeIds
window._lastNivelStudents, window._lastNivelCeIds
```

---

## 18. Gestión de Permisos (Administración)

### Agregar un profesor a una materia

1. Ir a Firebase Console → Realtime Database
2. Navegar a `/permisos/{materiaId}/`
3. Click en el ícono `+` para agregar nodo
4. **Key:** email del profesor con todos los puntos reemplazados por `_`
   - `bonninmiguel@gmail.com` → `bonninmiguel@gmail_com`
   - `pablo.colombo@frcu.utn.edu.ar` → `pablo_colombo@frcu_utn_edu_ar`
5. **Value:** `true`
6. Guardar. El profesor puede entrar en su próximo login.

### Agregar una nueva materia

1. En `/materias/`: agregar `{ ID_MATERIA: { nombre: "Nombre completo" } }`
2. En `/permisos/{ID_MATERIA}/`: agregar los emails de los profesores
3. La app la mostrará automáticamente en el selector de materias al iniciar sesión

### Cómo funciona la verificación en la app

```javascript
// En loadUserMaterias():
USER_MATERIAS = Object.entries(materias)
  .filter(([id]) => {
    const mp = permisos[id] || {};
    return Object.keys(mp).some(k =>
      k.toLowerCase() === email.replace(/\./g, '_')
    );
  })
  .map(([id, dat]) => ({ id, nombre: dat.nombre }));
```

La app busca el email del usuario (con puntos reemplazados por `_`) como **key** en el objeto de permisos de cada materia.

---

## 19. Bugs Importantes Resueltos

### 19.1 Funciones no encontradas (ReferenceError)
**Síntoma:** `ReferenceError: renderConfig is not defined`  
**Causa:** La función existe pero no está expuesta como `window.xxx`  
**Solución:** Agregar `window.renderConfig = renderConfig` al bloque de exposición

### 19.2 Event listener stacking — múltiples clicks para cerrar
**Síntoma:** Hay que hacer 4-5 clicks en "Cerrar" para que cierre un modal  
**Causa:** Cada re-render del panel agrega un nuevo listener al mismo elemento  
**Solución:** Flag `panel._delegated = true` para agregar el listener solo la primera vez

### 19.3 Select pierde el valor seleccionado al re-renderizar
**Síntoma:** Se selecciona RA01, se re-renderiza, vuelve a `--RA--`  
**Causa:** `onchange` inline no funciona en ES modules; al re-renderizar se crea un nuevo select sin el valor  
**Solución:** Guardar el valor en variable de estado ANTES de re-renderizar; el select se renderiza con `selected` basado en el estado

### 19.4 `filtered is not defined` en genInstResult/genRaceResult
**Síntoma:** El reporte no muestra datos, error en consola  
**Causa:** Al agregar código de export, se pegó `window._lastNivelStudents = filtered` donde la variable se llama `students`  
**Solución:** Cada función guarda sus propios datos: `window._lastInstStudents = students`, `window._lastRaceStudents = students`

### 19.5 PDF: título superpuesto con tabla en páginas 2+
**Síntoma:** El header del reporte se dibuja encima de las primeras filas en la segunda página  
**Causa:** `didDrawPage` se ejecuta DESPUÉS de que autoTable renderiza las filas  
**Solución:** Usar `willDrawPage` (ejecuta ANTES) + `margin: { top: HDR_H+14 }` para reservar espacio

### 19.6 Emails con puntos en Firebase Security Rules
**Síntoma:** `Error: Permission denied` al aplicar reglas  
**Causa:** Firebase Rules no soporta regex con flags (`/./g`); `.replace()` sin flags solo reemplaza el primer punto  
**Solución:** Encadenar 4 `.replace('.','_')` — cubre hasta 4 puntos en el email (suficiente para cualquier dominio universitario)

### 19.7 buildRepTable missing
**Síntoma:** `ReferenceError: buildRepTable is not defined` al generar reporte  
**Causa:** La función fue borrada en un reemplazo masivo de código  
**Solución:** La función debe existir y estar expuesta como `window.buildRepTable`

### 19.8 Funciones duplicadas
**Síntoma:** `SyntaxError: Identifier 'X' has already been declared`  
**Causa:** Al reemplazar secciones de código, el código anterior no fue eliminado completamente  
**Diagnóstico:** Buscar con `re.finditer(r'function X\b', html)` para encontrar todas las ocurrencias  
**Solución:** Eliminar la definición duplicada (dejar la más nueva)

---

## 20. Estructura del Archivo index.html

El archivo tiene ~2700 líneas con la siguiente organización:

```
<head>
  Meta + favicon (SVG data URL)
  Google Fonts CDN
  jsPDF + autoTable CDN
  <style> (~800 líneas de CSS)
    - Variables CSS
    - Reset y base
    - Layout: header, tabs, panels
    - Componentes: cards, tables, buttons, forms, modals, toasts
    - Clases de niveles: .niv-P, .niv-B, .niv-A, .niv-V
    - Clases de notas: .nota-pill, .nota-P, .nota-B, .nota-A, .nota-V
    - Dashboard: .dash-*, .kpi-*, .ev-card, etc.
    - @media print: oculta controles, muestra todo el contenido
    - @keyframes: fadeUp, progress

<body>
  #toast-container
  #saving-bar + #saving-overlay
  #screen-loading
  #screen-login
  #screen-materias
  #app
    .hdr (header)
    .tabs-bar
    .main > .panels
      #panel-inicio
      #panel-alumnos
      #panel-importar
      #panel-reportes
      #panel-config
      #panel-ayuda

<script type="module">
  // Firebase imports
  // firebaseConfig + inicialización
  // Variables de estado global
  // Helpers de DB (dbRef, dbSet, dbPush, dbRemove, dbUpdate)
  // Auth (signInWithGoogle, signOut, onAuthStateChanged)
  // loadUserMaterias, loadAllData, enterMateria, detachListeners
  // Helpers: objArr, esc, notaFmtStr
  // notaEfectiva, getBestNotaEfectiva
  // getNivelArr, getNivel, getDefaultNiveles
  // getEtq, getEtqTipos
  // loadCellMode, saveCellMode, getCellMode
  // renderApp, switchTab, showScreen, setLoadingMsg
  // renderMateriaSelector, _selectMateria, showMateria
  // renderInicio
  // renderAlumnos + openAlumnoModal + saveAlumno + delAlumno
  // openImportListaModal + handleListaDrop/File + confirmImportLista + readListaFile
  // renderImportar + parseCSVLine + parseCSV + confirmImport
  // renderGradesSummary + openEditNotas
  // delGrpNotas
  // renderReportes + renderRepInstancia + renderRepRACE + renderRepNivel
  // genInstResult + genRaceResult + genNivelResult
  // generatePDF + printInstancia + printRACE + printNivel
  // exportTableToExcel + exportInstancia + exportRACE + exportNivel
  // buildRepTable + buildCell + renderCfgPanel + refreshRep
  // toggleCellNota + toggleCellTipo
  // renderConfig + saveConfig
  // openRAModal + saveRA + delRA
  // openCEModal + saveCE + delCE
  // openEVModal + saveEV + delEV
  // openEtqModal + saveEtq + delEtq
  // openNivelModal + saveNivel
  // delAllNotas
  // renderAyuda
  // renderInicio (dashboard)
  // tgbConfirm, toast, openModal, closeModal, showSaving, showSaving
  // sortAlumnos
  // window.xxx = ... (todas las exposiciones)
```

---

## 21. Versioning

La versión aparece una sola vez, en la pantalla de carga:
```html
<div style="font-size:.7rem;color:rgba(255,255,255,.3);...">v2.24</div>
```

Incrementar al generar una nueva versión del `index.html`. Formato: `vMAYOR.MENOR` (ej: v2.24 → v2.25 para cambios menores, v3.00 para refactors grandes).

**Historial relevante:**

| Versión | Cambio principal |
|---|---|
| v2.00 | Primera versión Firebase estable |
| v2.07 | Fix reportes (filtered is not defined) |
| v2.09 | PDF con jsPDF + botones Descargar PDF |
| v2.11 | notamanual + openEditNotas |
| v2.13 | Renombrar tab Importar → Notas |
| v2.17 | Dashboard Inicio + Tab Ayuda |
| v2.19 | PDF mejorado + niveles configurables |
| v2.22 | Paleta negro/dorado (reemplazo de navy) |
| v2.24 | Permisos Firebase por key + fix app |

---

## 22. Pendientes Conocidos

- La ayuda debe actualizarse manualmente al agregar funcionalidades nuevas
- No hay filtro por grupo en los reportes PDF/Excel
- No hay log de auditoría de cambios de notas (quién modificó qué y cuándo)
- No hay paginación en tablas (110 alumnos funcionan bien; con 500+ podría lentificarse)
- El panel de edición de notas manuales requiere saber el RA+CE+EV exacto para abrirse; no hay búsqueda global de nota por alumno
- Si se agregan niveles con códigos fuera de P/B/A/V, hay que agregar sus colores manualmente en `generatePDF` y en el CSS

---

## 23. Cómo Continuar el Trabajo

Para retomar desarrollo con una nueva sesión de IA:

1. Compartir este documento (`TierraGradeBook_Version_2.24_Contexto_Tecnico.md`)
2. Compartir el `index.html` de la versión vigente
3. Indicar qué se quiere hacer

**Los 5 puntos más críticos para no cometer errores:**

1. **Siempre exponer funciones nuevas a `window`** — si aparece `ReferenceError: X is not defined`, es esto
2. **Usar `addEventListener` nunca `onclick` inline** en HTML generado dinámicamente
3. **Usar `notaEfectiva(n)` nunca `n.nota`** en cualquier cálculo de nota para reportes/promedios
4. **Verificar duplicados** después de cualquier reemplazo grande de código con `re.finditer(r'function X\b', html)`
5. **La paleta es negro (#111) + dorado (#c9a84c)** — nunca agregar colores navy/azul nuevos

**Verificación de sintaxis antes de subir:**
```bash
# Extraer JS del HTML y verificar con Node.js
python3 -c "
import re
with open('index.html','r',encoding='utf-8',errors='replace') as f: html=f.read()
scripts=re.findall(r'<script type=\"module\">([\s\S]*?)</script>',html)
open('/tmp/check.js','w').write('\n'.join(scripts))
"
node --check /tmp/check.js
```
