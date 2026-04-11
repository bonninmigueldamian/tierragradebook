# TierraGradeBook — Contexto Técnico Completo
**Versión del código:** v2.24  
**Fecha:** Abril 2026  
**Archivo principal:** `index.html` (archivo único, ~2700 líneas)

---

## ADVERTENCIA PARA EL LECTOR (IA o persona)

Este documento es la fuente de verdad del proyecto. Cuando una pregunta requiera un detalle muy específico que no esté aquí, la respuesta correcta es **"hay que verificarlo en el index.html"**, nunca inventar. Las respuestas incorrectas más probables son inventar nombres de clases CSS, nombres de variables, o comportamientos de funciones que suenan plausibles pero no están verificados.

---

## 1. Descripción del Sistema

TierraGradeBook es una aplicación web de gestión de notas universitarias organizada por **Resultados de Aprendizaje (RA)** y **Criterios de Evaluación (CE)**. Permite importar notas desde CSV de Moodle, cargar notas manuales con prioridad sobre las importadas, y generar reportes en PDF y Excel coloreados por nivel de desempeño.

**Usuarios:** Solo profesores. El acceso se controla manualmente por el administrador en Firebase. Los alumnos **no tienen acceso** a la app.

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

**Por qué no hay frameworks:** La app es un único `index.html` desplegado en GitHub Pages sin proceso de build. Mantener todo en un solo archivo elimina bundlers, dependencias locales y pasos de compilación.

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

> **Nota de seguridad:** El `apiKey` es público por diseño — solo identifica el proyecto, no autoriza nada. La seguridad real la proveen las Security Rules. Alguien que vea el apiKey sin estar autenticado y autorizado no puede leer ni escribir nada.

---

## 5. Firebase Security Rules (actuales y verificadas)

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

**Explicación:**
- `materias` y `permisos`: legibles por cualquier Gmail autenticado. Necesario porque la app lee estos nodos ANTES de saber si el usuario tiene permisos.
- `datos`: solo accesible si el email (con puntos reemplazados por `_`) existe como key en `/permisos/{materiaId}/`.
- Nadie puede modificar `/permisos/` desde la app (`.write: false`). Solo el admin desde la consola de Firebase.
- Firebase Rules no soporta regex con flags (`/\./g`). Por eso se encadenan 4 `.replace('.','_')` — cada uno reemplaza el primer punto restante. Cubre hasta 4 puntos en el email.

---

## 6. Estructura de Datos en Firebase

### REGLA CRÍTICA: Firebase no acepta puntos en keys

Firebase no acepta `.` en los keys de nodos. Tampoco acepta `$`, `#`, `[`, `]`, `/`.

**Solución implementada:**
- Los IDs de CE se guardan con `_` en lugar de `.` como key (ej: `CE_01_01`)
- Los emails en `/permisos/` usan `_` en lugar de `.`
- La función `objArr()` agrega `_key` a cada objeto con el key real de Firebase

```
/materias/
  ANALISIS_2026: { nombre: "Análisis de Sistemas de Información - FRCU UTN" }

/permisos/
  ANALISIS_2026/
    bonninmiguel@gmail_com: true    ← email con _ en lugar de .
    mvansorena@gmail_com: true
    pablo_colombo@gmail_com: true

/datos/ANALISIS_2026/

  config/
    materia: "Análisis de Sistemas de Información - FRCU UTN"
    año: 2026

  alumnos/
    {key_autogenerado}/
      apellido: "GARCIA"            ← siempre MAYÚSCULAS
      nombre: "JUAN PABLO"          ← siempre MAYÚSCULAS
      email: "juan.garcia@gmail.com" ← con puntos originales
      legajo: "14150216"
      año: 2026
      grupo: 6                      ← 0 = sin grupo

  ra/
    RA01/                           ← key sin puntos
      id: "RA01"                    ← valor a mostrar
      nombre: "Reconoce los distintos interesados..."
      peso: 0.3

  ce/
    CE_01_01/                       ← key con _ en lugar de .
      id: "CE_01_01"                ← valor a mostrar
      nombre: "C.E.01.01. Reconoce los distintos interesados..."
      raId: "RA01"
      peso: 0.3

  ev/
    PT1/                            ← código corto de la instancia
      nombre: "Parcial Teorico 1"

  niveles/
    P/
      code: "P"
      nombre: "Principiante"
      desde: 0
      hasta: 3.99
    B/ A/ V/  ← igual estructura

  notas/
    {key_autogenerado}/
      email: "juan.garcia@gmail.com"  ← con puntos originales
      raId: "RA01"
      ceId: "CE_01_01"
      evId: "PT1"
      nota: 8.33           ← convertida a /10 al importar
      notaoriginal: 1.67   ← valor exacto del CSV (escala original del Moodle)
      notamanual: null     ← null=no hay; número=tiene prioridad absoluta sobre nota
      ts: 1712345678000    ← timestamp Unix

  etiquetas/
    {key_autogenerado}/
      tipo: "Accion"          ← nombre del conjunto (ej: Accion, Estado)
      nivel_code: "P"         ← código del nivel al que aplica
      etiqueta: "Recuperar"   ← texto a mostrar en celdas de reportes
```

---

## 7. Sistema de Notas — Reglas Absolutas

### 7.1 Los tres campos de nota

| Campo | Escala | Quién lo escribe | Se pierde al reimportar |
|---|---|---|---|
| `notaoriginal` | Original del CSV (ej: sobre 2) | `confirmImport` | **SÍ** |
| `nota` | 1-10 | `confirmImport` | **SÍ** |
| `notamanual` | 1-10 | `openEditNotas` | **SÍ** |

### 7.2 DECISIÓN DE DISEÑO CRÍTICA: Reimportación borra TODO

**Al reimportar un CSV para una combinación RA+CE+EV:**
1. Se borran **TODAS** las notas existentes de esa combinación
2. Esto incluye `notamanual` — **las notas manuales SE PIERDEN**
3. Se insertan las nuevas notas del CSV con `notamanual: null`
4. El profe tiene que volver a cargar las notas manuales manualmente

Esta es la **Opción A**, elegida deliberadamente por simplicidad. No hay lógica de preservar notas manuales. Es una decisión de diseño documentada.

```javascript
// Código real de confirmImport — borra TODO:
const existentes = Object.entries(DB.notas||{})
  .filter(([,n]) => n.raId===impRA && n.ceId===impCE && n.evId===impEV)
  .map(([k]) => k);
for(const k of existentes) {
  await dbRemove(`notas/${k}`);
  delete DB.notas[k];
}
// Luego inserta nuevas con notamanual: null
```

### 7.3 notaEfectiva(n) — REGLA ABSOLUTA

```javascript
function notaEfectiva(n) {
  if(n.notamanual !== null && n.notamanual !== undefined && n.notamanual !== '')
    return parseFloat(n.notamanual);
  return parseFloat(n.nota) || 0;
}
```

**TODA operación que use la nota de un alumno DEBE usar `notaEfectiva(n)`, nunca `n.nota`.**

Esto incluye: reportes, promedios, dashboard, PDF, Excel, colores de celda.

**Caso especial crítico:** Si `notamanual = 0`, retorna `0`. Cero es una nota válida. La verificación `!== null && !== undefined && !== ''` acepta el cero correctamente. Si en algún lugar se usa `if(n.notamanual)` el cero sería falsy en JavaScript y se ignoraría — eso es un bug.

### 7.4 Conversión de escala al importar

```javascript
const gradeCol = fields.find(h => /calificaci[oó]n\//i.test(h));
const maxNota  = gradeCol
  ? parseFloat(gradeCol.match(/\/(\d+[,.]\d*)/)[1].replace(',','.')) || 2
  : 2;
const nota = Math.round((rawNota / maxNota) * 10 * 100) / 100;
// Ejemplo: CSV nota=1.67, máximo=2 → nota = (1.67/2)*10 = 8.35
```

---

## 8. Arquitectura del Código

### 8.1 EL problema más importante: ES Modules y window

La app usa `<script type="module">`. **En módulos ES, las funciones tienen scope local — no son globales.** Los atributos HTML `onclick="miFuncion()"` buscan en `window` y no las encuentran → `ReferenceError`.

**Solución obligatoria:** Toda función llamada desde HTML debe exponerse:
```javascript
window.miFuncion = miFuncion;
```

**Diagnóstico:** Si un botón no responde y la consola muestra `ReferenceError: X is not defined`, la causa es que `window.X` no está expuesta. Es el error más frecuente.

### 8.2 Patrón de renderizado

```javascript
function renderAlgo() {
  document.getElementById('panel').innerHTML = '<button id="mi-btn">Click</button>';
  // DESPUÉS del innerHTML, adjuntar eventos:
  document.getElementById('mi-btn').addEventListener('click', function() { ... });
}
```

**Nunca usar onclick inline** en HTML generado dinámicamente.

### 8.3 Event listener stacking — bug frecuente

**Síntoma:** Hay que hacer 4-5 clicks en "Cerrar" para que un modal cierre.
**Causa:** Cada re-render del panel agrega un nuevo listener al mismo elemento DOM.
**Solución:** Flag `_delegated` en el elemento DOM:

```javascript
const panel = document.getElementById('panel-importar');
if(!panel._delegated) {
  panel._delegated = true;
  panel.addEventListener('click', function(e) {
    const btn = e.target.closest('[data-action]');
    if(!btn) return;
    if(btn.dataset.action === 'edit-notas') window.openEditNotas(...);
    if(btn.dataset.action === 'del-notas')  window.delGrpNotas(...);
  });
}
```

### 8.4 DB como espejo local

`DB` es un objeto en memoria que replica los datos de Firebase. **Toda escritura debe actualizar AMBOS:**

```javascript
await dbSet('notas/'+key+'/notamanual', valor);  // Firebase
DB.notas[key].notamanual = valor;                 // espejo local
```

Si solo se actualiza uno, la UI se desincroniza hasta el próximo reload.

### 8.5 Listeners en tiempo real — VERIFICADO

`onValue` está **importado** del SDK de Firebase pero **NO se usa activamente** como listener permanente en la operación normal de la app. La app carga datos con `get()` (lectura única) en `loadAllData()`. No hay sincronización automática entre browsers.

`DB_LISTENERS` existe en el código para limpiar al cambiar de materia, no para listeners continuos.

**Consecuencia:** Si dos profesores editan simultáneamente, sus cambios locales no se sincronizan automáticamente. El último en escribir a Firebase gana (last-write-wins). No hay protección contra concurrencia.

### 8.6 Helpers de base de datos

```javascript
function dbRef(path) { return ref(db, `datos/${CURRENT_MATERIA.id}/${path}`); }
async function dbSet(path, val)    { showSaving(true); await set(dbRef(path), val);    showSaving(false); }
async function dbPush(path, val)   { showSaving(true); const r=await push(dbRef(path),val); showSaving(false); return r.key; }
async function dbRemove(path)      { showSaving(true); await remove(dbRef(path));       showSaving(false); }
async function dbUpdate(path, val) { showSaving(true); await update(dbRef(path), val);  showSaving(false); }
```

**Siempre usar estos wrappers.** Muestran el indicador de guardado y centralizan la lógica.

---

## 9. Variables de Estado Global (verificadas en el código)

```javascript
let CURRENT_USER    = null;
let CURRENT_MATERIA = null;   // { id: "ANALISIS_2026", nombre: "..." }
let USER_MATERIAS   = [];
let DB_LISTENERS    = [];     // para limpiar al cambiar materia
let DB = { config:{}, alumnos:{}, ra:{}, ce:{}, ev:{}, niveles:{}, notas:{}, etiquetas:{} };

let currentTab  = 'inicio';
let alumSearch  = '';
let alumGrupo   = 'todos';

// Importar CSV
let impRA='', impCE='', impEV='', impStep=1, impFile=null, impPreview=[];

// Reportes
let repType='instancia', repEV='', repLevels=[], repNivelMode='any';
let repCellMode = {};  // config de celdas — persistida en localStorage

// Datos temporales para export/PDF
window._lastInstStudents=[], window._lastInstCeIds=[];
window._lastRaceStudents=[], window._lastRaceCeIds=[];
window._lastNivelStudents=[], window._lastNivelCeIds=[];
```

---

## 10. localStorage — Configuración de Celdas (verificado en el código)

Hay tres cosas distintas que no hay que confundir:

| Concepto | Valor |
|---|---|
| Variable JS en memoria | `repCellMode` (objeto) |
| Constante con prefijo | `LS_CM = 'tgb_cm_'` |
| Key real en localStorage | `'tgb_cm_' + CURRENT_MATERIA.id` |

Ejemplo de key real: `'tgb_cm_ANALISIS_2026'`

**`repCellMode` es el nombre de la variable interna en JavaScript. NO es la key de localStorage.** La key de localStorage es `LS_CM + materia.id`.

Funciones: `loadCellMode()` lee de localStorage y puebla `repCellMode`, `saveCellMode()` serializa `repCellMode` y lo escribe en localStorage. Es local por browser, no va a Firebase.

---

## 11. CSS — Clases verificadas en el código

```css
/* Niveles — chips de texto */
.niv-P, .niv-B, .niv-A, .niv-V, .niv-N

/* Notas — pills de color en reportes */
.nota-P, .nota-B, .nota-A, .nota-V, .nota-N, .nota-pill, .nota-cell

/* Etiquetas en celdas de reportes */
.etq-tag          /* contenedor */
.etq-P, .etq-B, .etq-A, .etq-V  /* color por nivel */
```

**Para agregar un nivel nuevo** (ej: code `"E"`), hay que agregar en el código:
1. `.niv-E` en el CSS
2. `.nota-E` en el CSS
3. `.etq-E` en el CSS
4. El color RGB en `generatePDF()` (arrays `nivFill` y `nivTxt`)
5. El nodo en Firebase `/datos/{materiaId}/niveles/E/`

---

## 12. Funciones Clave (verificadas)

### objArr(obj)
```javascript
function objArr(obj) {
  if(!obj) return [];
  return Object.entries(obj).map(([k,v]) => Object.assign({}, v, { _key: k }));
}
```
Convierte objetos Firebase en arrays con `_key`. Usada en prácticamente toda iteración.

### getBestNotaEfectiva(email, raId, ceId)
Filtra notas del alumno para ese RA+CE (todas las instancias), aplica `notaEfectiva()` a cada una, retorna el máximo. Retorna `null` si no hay notas. Usada en el reporte RACE.

### tgbConfirm({ title, msg, okLabel, danger })
Modal de confirmación. Retorna Promise que resuelve `true`/`false`. Usada con `await`.

### loadAllData()
Carga todo `/datos/{materiaId}/` en una sola lectura `get()`. Si no existen niveles, llama a `getDefaultNiveles()` y los persiste en Firebase con `dbSet('niveles', ...)`. Se ejecuta al seleccionar materia.

### sortAlumnos(arr)
```javascript
function sortAlumnos(arr) {
  return [...arr].sort((a,b) => (a.apellido+a.nombre).localeCompare(b.apellido+b.nombre,'es'));
}
```
Ordena por apellido+nombre en español (maneja ñ y tildes).

---

## 13. Verificación de Permisos en la App (código real)

```javascript
USER_MATERIAS = Object.entries(materias)
  .filter(([id]) => {
    const mp = permisos[id] || {};
    return Object.keys(mp).some(k =>
      k.toLowerCase() === email.replace(/\./g, '_')
    );
  })
  .map(([id, dat]) => ({ id, nombre: dat.nombre }));
```

**La app busca el email como KEY, no como valor.** Solo reconoce el formato nuevo:
```
{ "pablo_colombo@gmail_com": true }  ← RECONOCIDO
{ "email0": "pablo.colombo@gmail.com" }  ← IGNORADO (formato viejo)
```

El formato viejo tiene el email como valor. La app itera sobre `.keys()`, no sobre `.values()`, así que el formato viejo no da acceso aunque esté en Firebase.

---

## 14. Reportes

### 14.1 Tres tipos

| Tipo | Muestra |
|---|---|
| Por Instancia | Notas de un EV específico. Genera con `genInstResult()` |
| RACE (Mejor Nota RA/CE) | Mejor `notaEfectiva` por alumno/CE en todas las instancias. Genera con `genRaceResult()` |
| Filtro por Nivel | Alumnos en niveles seleccionados. Genera con `genNivelResult()` |

**Flujo:** Configurar filtros → click "Generar" → función genera tabla HTML + guarda datos en `window._lastXxx` → botones PDF/Excel leen esos datos almacenados.

### 14.2 generatePDF(reportType, reportDetail, students, ceIds)

```javascript
// Callers reales (verificados):
generatePDF('Reporte Por Instancia', repEV+' - '+ev.nombre, students, ceIds);
generatePDF('Mejor Nota por RA/CE', 'Mejor nota considerando todas las instancias', students, ceIds);
generatePDF('Filtro por Nivel', nivNames, students, ceIds);
```

- Usa `willDrawPage` (NO `didDrawPage`) — se ejecuta ANTES de que autoTable dibuje las filas
- `margin: { top: HDR_H+14 }` reserva espacio para el header en todas las páginas
- Nombres truncados a 26 caracteres
- Columna nombre: 46mm, Grp: 9mm, CE: calculado dinámicamente (mín 15mm, máx 30mm)

### 14.3 Export Excel

CSV con BOM UTF-8 (`\uFEFF` al inicio). Sin BOM, tildes y ñ aparecen corruptas en Excel Windows.

---

## 15. Panel "Mostrar en cada celda"

Configura qué aparece en las celdas de reportes por nivel. **Persiste en localStorage** (key: `tgb_cm_` + materiaId). NO va a Firebase.

```javascript
// Estructura de repCellMode:
{
  P: { nota: true,  types: ['Accion'] },
  B: { nota: false, types: ['Estado'] },
  A: { nota: true,  types: [] },
  V: { nota: true,  types: [] }
}
```

**buildCell(nota):** SIEMPRE llamar con `buildCell(notaEfectiva(n))`, nunca `buildCell(n.nota)`.

---

## 16. Niveles de Desempeño

Configurables desde Config → Niveles → ✏️ Editar. Se guardan en Firebase.

**Defaults (se crean automáticamente si no existen en Firebase):**

| Code | Nombre | Desde | Hasta |
|---|---|---|---|
| P | Principiante | 0 | 3.99 |
| B | Básico | 4 | 6.99 |
| A | Autónomo | 7 | 8.99 |
| V | Avanzado | 9 | 10 |

**Clases CSS verificadas:** `.niv-P`, `.niv-B`, `.niv-A`, `.niv-V`, `.niv-N`  
**Colores en PDF:** hardcodeados en `generatePDF()`. Si se agrega code fuera de P/B/A/V, agregar color manualmente ahí.

---

## 17. Funciones Expuestas a window (lista completa verificada)

```javascript
window.signInWithGoogle, window.signOut
window.switchTab, window._selectMateria, window.showMateria
window.renderAlumnos, window.openAlumnoModal, window.saveAlumno, window.delAlumno
window.openImportListaModal, window.handleListaDrop, window.handleListaFile
window.confirmImportLista, window.readListaFile
window.renderImportar, window.parseCSV, window.confirmImport
window.delGrpNotas, window.renderGradesSummary, window.openEditNotas
window.renderReportes, window.renderRepInstancia, window.renderRepRACE, window.renderRepNivel
window.refreshRep, window.renderCfgPanel
window.genInstResult, window.genRaceResult, window.genNivelResult
window.toggleCellNota, window.toggleCellTipo
window.printInstancia, window.printRACE, window.printNivel
window.exportInstancia, window.exportRACE, window.exportNivel
window.buildRepTable
window.renderConfig, window.saveConfig
window.openRAModal, window.saveRA, window.delRA
window.openCEModal, window.saveCE, window.delCE
window.openEVModal, window.saveEV, window.delEV
window.openEtqModal, window.saveEtq, window.delEtq
window.openNivelModal, window.saveNivel
window.delAllNotas
window.renderAyuda, window.renderInicio
window.closeModal
```

---

## 18. Gestión de Permisos (Administración)

### Agregar un profesor

1. Firebase Console → Realtime Database → `/permisos/{materiaId}/`
2. **Key** = email con todos los `.` reemplazados por `_`, **Value** = `true`
   - `pablo.colombo@gmail.com` → `pablo_colombo@gmail_com`
   - `juan.perez@frcu.utn.edu.ar` → `juan_perez@frcu_utn_edu_ar`
3. No hay que tocar el código.

### Agregar una nueva materia

1. `/materias/`: `{ NUEVO_ID: { nombre: "Nombre completo" } }`
2. `/permisos/NUEVO_ID/`: agregar emails de profesores
3. La app la muestra automáticamente.

---

## 19. Bugs Resueltos — Los Más Importantes

| Bug | Síntoma | Causa | Solución |
|---|---|---|---|
| ReferenceError | Botón no responde | Función no expuesta a `window` | `window.fn = fn` |
| Múltiples clicks para cerrar | 4-5 clicks en "Cerrar" | Listeners acumulados en re-renders | Flag `_delegated` |
| Select pierde valor | Select vuelve a `--RA--` | `onchange` inline no funciona en ES modules | Estado en variable + `selected` al renderizar |
| `filtered is not defined` | Reporte no muestra datos | Variable mal nombrada entre funciones | Cada función almacena en su propio `_lastXxx` |
| PDF superpuesto pág 2+ | Header encima de filas | `didDrawPage` ejecuta después de las filas | `willDrawPage` + `margin.top` |
| Función duplicada | `SyntaxError: already declared` | Reemplazo dejó definición vieja | Buscar con `re.finditer(r'function X\b', html)` |
| `notamanual=0` ignorada | Nota cero no tiene prioridad | `if(n.notamanual)` — el 0 es falsy | Comparación explícita `!== null && !== undefined && !== ''` |
| Emails formato viejo | Usuario sin acceso | Formato `{email0:"..."}` — app busca por key | Migrar a `{pablo@gmail_com: true}` |

---

## 20. Paleta de Colores

```css
--navy:   #111111   /* negro principal */
--navy2:  #1c1c1c   /* negro secundario */
--navy3:  #2a2a2a   /* negro terciario */
--gold:   #c9a84c   /* dorado */
--gold2:  #e8d08a   /* dorado claro */
```

**REGLA:** El sistema usa negro + dorado. Nunca agregar azules navy.  
**En PDF:** `navy = [17,17,17]`, `gold = [201,168,76]`

---

## 21. Versioning

Versión en pantalla de carga, un solo lugar: `v2.24`.

| Versión | Cambio |
|---|---|
| v2.00 | Primera versión Firebase estable |
| v2.11 | notamanual + openEditNotas |
| v2.13 | Tab "Importar" renombrado a "Notas" |
| v2.17 | Dashboard Inicio + Ayuda |
| v2.19 | PDF mejorado + niveles configurables |
| v2.22 | Paleta negro/dorado |
| v2.24 | Permisos Firebase por key |

---

## 22. Pendientes Conocidos

- No hay filtro por grupo en PDF/Excel
- No hay log de auditoría de cambios de notas
- No hay protección contra escrituras concurrentes entre profesores
- Ayuda debe actualizarse manualmente al agregar funcionalidades
- Niveles con code fuera de P/B/A/V requieren agregar CSS + color PDF manualmente

---

## 23. Cómo Continuar el Trabajo

1. Compartir este `.md` + el `index.html` de la versión vigente
2. Indicar qué se quiere hacer

**Los 5 puntos que no se pueden olvidar:**
1. Exponer funciones nuevas a `window`
2. Usar `addEventListener`, nunca `onclick` inline en HTML generado
3. Usar `notaEfectiva(n)`, nunca `n.nota`
4. Al reimportar CSV: las notas manuales **SE PIERDEN** (Opción A — decisión de diseño deliberada)
5. Paleta negro + dorado, nunca navy

**Verificar sintaxis antes de subir:**
```bash
python3 -c "
import re
with open('index.html','r',encoding='utf-8',errors='replace') as f: html=f.read()
scripts=re.findall(r'<script type=\"module\">([\s\S]*?)</script>',html)
open('/tmp/check.js','w').write('\n'.join(scripts))
"
node --check /tmp/check.js
```

**Ante cualquier duda sobre un detalle específico:** verificarlo directamente en el `index.html`. No inventar.
