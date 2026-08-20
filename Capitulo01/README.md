# 4 Practica: diagnóstico y optimización de renders en una SPA

## Metadatos

| Campo | Valor |
|---|---|
| Duración | 24 minutos |
| Complejidad | Media |
| Nivel de Bloom | Aplicar |

## Descripción general

En este laboratorio diagnosticarás renders innecesarios en la SPA **ProjectHub** y aplicarás optimizaciones basadas en el modelo de reactividad de Vue 3. Instrumentarás componentes con contadores de render y marcas de rendimiento, observarás actualizaciones mediante Vue Devtools y Chrome DevTools, y separarás el estado local del estado compartido.

El resultado será una estructura modular con `computed`, `shallowRef`, `v-memo` y un módulo temporal de estado compartido en `src/state/projectHubState.js`. Esta estructura se conservará como punto de partida para los laboratorios posteriores de composables y Pinia.

## Objetivos de aprendizaje

Al completar el laboratorio, podrás:

- [ ] Identificar qué cambios reactivos provocan renders en componentes relacionados y no relacionados.
- [ ] Explicar la diferencia práctica entre `ref`, `reactive`, `computed`, `watch`, `watchEffect` y `shallowRef`.
- [ ] Separar el estado local de filtros del estado compartido de proyectos, tareas, selección y tema.
- [ ] Reemplazar cálculos repetidos en plantilla por propiedades `computed`.
- [ ] Aplicar `shallowRef` y `v-memo` para reducir trabajo de renderizado en una lista filtrable.

## Prerrequisitos

### Conocimientos requeridos

- JavaScript moderno: módulos ES, funciones flecha, destructuring y promesas.
- Vue 3: componentes, props, eventos, `ref`, `reactive`, `computed` y ciclo de vida.
- Fundamentos de reactividad de Vue: proxys, seguimiento de dependencias y efectos de renderizado.
- Uso básico de Chrome DevTools y consola del navegador.

### Acceso requerido

- Repositorio base disponible en:

  ```text
  /workspace/vue-intermedio-labs
  ```

  En Windows:

  ```text
  C:\workspace\vue-intermedio-labs
  ```

- Dependencias instaladas previamente con `npm ci`.
- Extensión Vue Devtools 7.7.2 instalada en Google Chrome.
- Acceso local al puerto `5173`.

## Entorno del laboratorio

### Hardware recomendado

| Recurso | Mínimo |
|---|---|
| CPU | 2 núcleos, 2.0 GHz |
| Memoria RAM | 8 GB |
| Espacio libre | 5 GB |
| Resolución | 1366x768 |
| Red | Internet durante la instalación inicial |

### Software

| Herramienta | Versión de referencia |
|---|---|
| Node.js | 22.14.0 |
| npm | 10.9.2 |
| Vue | 3.5.13 |
| Vite | 6.1.0 |
| Google Chrome | 133.0.6943.98 o superior |
| Vue Devtools | 7.7.2 |
| Sistema operativo | Windows 11, macOS 14 o Ubuntu 24.04 |

### Preparación inicial

1. Abre una terminal en el directorio obligatorio:

   ```bash
   cd /workspace/vue-intermedio-labs
   ```

   En Windows PowerShell:

   ```powershell
   cd C:\workspace\vue-intermedio-labs
   ```

2. Comprueba la rama y el estado del repositorio. El curso utiliza una única rama de trabajo llamada `main`.

   ```bash
   git branch --show-current
   git status
   ```

3. Instala dependencias únicamente si no se instalaron antes:

   ```bash
   npm ci
   ```

4. Inicia el servidor Vite en el puerto fijo del curso:

   ```bash
   npm run dev -- --host 127.0.0.1 --port 5173
   ```

5. Abre la aplicación en Chrome:

   ```text
   http://127.0.0.1:5173
   ```

> No inicies servicios en el puerto `3000`. Este laboratorio no requiere API local; los futuros laboratorios utilizarán MSW para interceptar solicitudes.

---

## Desarrollo paso a paso

### Paso 1. Inspeccionar el estado inicial y preparar el diagnóstico

**Objetivo:** confirmar que la SPA ProjectHub está disponible e identificar los componentes que participan en la pantalla.

**Instrucciones:**

1. Con el servidor Vite en ejecución, abre `http://127.0.0.1:5173`.

2. Identifica las tres zonas funcionales de la SPA:

   - Panel de proyectos.
   - Listado de tareas filtrable.
   - Panel de preferencias visuales.

3. Abre Chrome DevTools con `F12` o `Ctrl+Shift+I`.

4. En la pestaña **Console**, ejecuta:

   ```js
   performance.clearMarks()
   console.clear()
   ```

5. Abre Vue Devtools desde las herramientas de desarrollo de Chrome.

6. En Vue Devtools, localiza el árbol de componentes. Deberías reconocer componentes equivalentes a:

   ```text
   App
   ├── ProjectPanel
   ├── TaskList
   └── PreferencesPanel
   ```

7. Si el repositorio inicial tiene una estructura diferente, identifica los componentes que cumplen esas mismas responsabilidades antes de continuar.

**Salida esperada:**

- La aplicación ProjectHub carga en el navegador.
- Vue Devtools reconoce la aplicación Vue.
- La consola está limpia y lista para registrar actividad de renderizado.

**Verificación:**

En Vue Devtools, selecciona el componente raíz. Comprueba que el estado visible de la aplicación incluye, como mínimo, información de tareas, selección, filtros o preferencias.

---

### Paso 2. Instrumentar renders con contadores y marcas de rendimiento

**Objetivo:** registrar cuándo Vue vuelve a renderizar cada componente sin introducir estado reactivo adicional durante el render.

**Instrucciones:**

1. Crea el directorio de utilidades si todavía no existe:

   ```bash
   mkdir -p src/utils
   ```

2. Crea el archivo `src/utils/renderProbe.js` con el siguiente contenido:

   ```js
   const renderCounts = new Map()

   export function probeRender(componentName) {
     const currentCount = (renderCounts.get(componentName) ?? 0) + 1
     renderCounts.set(componentName, currentCount)

     const markName = `render:${componentName}:${currentCount}`

     performance.mark(markName)
     console.count(`[render] ${componentName}`)

     return ''
   }

   export function getRenderCount(componentName) {
     return renderCounts.get(componentName) ?? 0
   }

   export function resetRenderProbes() {
     renderCounts.clear()
     performance.clearMarks()
   }
   ```

3. Añade la instrumentación temporal a los componentes iniciales de ProjectHub. En cada componente, importa `probeRender`:

   ```js
   import { probeRender } from '../utils/renderProbe'
   ```

   Ajusta la ruta relativa si el componente se encuentra en otra carpeta.

4. Expón la función en el `<script setup>`. Con `<script setup>`, una función importada queda disponible directamente en la plantilla.

5. Dentro de la plantilla de cada componente, agrega una interpolación invisible. Por ejemplo, en `ProjectPanel.vue`:

   ```vue
   <span class="sr-only">{{ probeRender('ProjectPanel') }}</span>
   ```

   En `TaskList.vue`:

   ```vue
   <span class="sr-only">{{ probeRender('TaskList') }}</span>
   ```

   En `PreferencesPanel.vue`:

   ```vue
   <span class="sr-only">{{ probeRender('PreferencesPanel') }}</span>
   ```

6. Si el proyecto no dispone de una clase visual para ocultar contenido, añade esta regla a `src/style.css`:

   ```css
   .sr-only {
     position: absolute;
     width: 1px;
     height: 1px;
     padding: 0;
     margin: -1px;
     overflow: hidden;
     clip: rect(0, 0, 0, 0);
     white-space: nowrap;
     border: 0;
   }
   ```

7. Guarda los archivos y vuelve al navegador. Vite aplicará actualización en caliente.

8. En la consola, realiza interacciones aisladas:

   - Cambia el filtro de tareas.
   - Selecciona una tarea.
   - Cambia el tema visual.
   - Vuelve a cambiar solo una de esas variables.

**Salida esperada:**

La consola muestra mensajes similares a:

```text
[render] ProjectPanel: 1
[render] TaskList: 1
[render] PreferencesPanel: 1
```

Después de una interacción, puede aumentar el contador de más componentes de los esperados en la implementación inicial.

**Verificación:**

Ejecuta en la consola:

```js
performance.getEntriesByType('mark')
  .map((entry) => entry.name)
  .filter((name) => name.startsWith('render:'))
```

Comprueba que existen marcas con nombres como:

```text
render:TaskList:2
render:PreferencesPanel:2
```

> La función `probeRender()` usa un `Map` normal, no reactivo. Es importante: actualizar un `ref` o un `reactive` desde una plantilla para contar renders crearía nuevas dependencias y podría alterar el diagnóstico.

---

### Paso 3. Diagnosticar dependencias reactivas no deseadas

**Objetivo:** relacionar los renders observados con el grafo de dependencias reactivas de Vue.

**Instrucciones:**

1. Antes de cada prueba, limpia las marcas y la consola:

   ```js
   performance.clearMarks()
   console.clear()
   ```

2. Realiza la primera prueba: modifica solo el texto de filtro de tareas.

3. Observa qué componentes registran un nuevo `[render]`.

4. Realiza la segunda prueba: cambia solo el tema desde preferencias.

5. Realiza la tercera prueba: selecciona una sola tarea.

6. Registra tus resultados en esta tabla:

   | Cambio realizado | Componentes que renderizan en la versión inicial | ¿Es necesario? |
   |---|---|---|
   | Cambiar filtro |  |  |
   | Cambiar tema |  |  |
   | Seleccionar tarea |  |  |

7. En Vue Devtools, selecciona `TaskList` y revisa las propiedades y el estado expuesto. Busca patrones como los siguientes:

   ```js
   const ui = reactive({
     filter: '',
     theme: 'light',
     selectedTaskId: 'task-1'
   })
   ```

   o un objeto grande que se pase completo mediante props:

   ```vue
   <TaskList :ui="ui" :tasks="tasks" />
   <ProjectPanel :ui="ui" :projects="projects" />
   <PreferencesPanel :ui="ui" />
   ```

8. Explica por qué ese patrón amplía las dependencias de renderizado. Aunque un componente solo necesite una propiedad, recibir, leer o derivar datos desde un objeto compartido grande puede mezclar responsabilidades y dificultar el diagnóstico.

9. Usa la siguiente tabla como referencia para interpretar el comportamiento:

   | API | Tipo de seguimiento | Uso apropiado en este laboratorio |
   |---|---|---|
   | `ref()` | Sigue cambios de un valor mediante `.value` | Tema, filtro, identificador de tarea seleccionada |
   | `reactive()` | Proxy profundo para propiedades de un objeto | Objetos pequeños y cohesivos modificados por propiedad |
   | `computed()` | Efecto derivado con caché hasta que cambian sus dependencias | Lista filtrada, contadores y tarea seleccionada |
   | `watch()` | Reacciona explícitamente a una fuente concreta | Sincronizar el tema con `document.documentElement` |
   | `watchEffect()` | Descubre automáticamente las dependencias leídas en su efecto | Diagnóstico o efectos simples; usar con cuidado |
   | `shallowRef()` | Sigue el reemplazo de `.value`, no mutaciones internas profundas | Colecciones externas o inmutables de proyectos y tareas |

10. Recuerda la relación con los fundamentos de reactividad:

    - Un `reactive()` crea un `Proxy`.
    - Una lectura durante el render registra una dependencia.
    - Una escritura notifica a los efectos dependientes.
    - El render del componente es uno de esos efectos.
    - Reducir las lecturas reactivas de un componente reduce su superficie de actualización.

**Salida esperada:**

La versión inicial muestra al menos una actualización innecesaria: por ejemplo, cambiar el tema puede volver a renderizar el listado de tareas, o cambiar el filtro puede actualizar una zona que no depende funcionalmente del filtro.

**Verificación:**

Responde estas preguntas antes de continuar:

1. ¿Qué dato reactivo cambia al escribir en el filtro?
2. ¿Qué componente debería depender de ese dato?
3. ¿Por qué el panel de proyectos no necesita conocer el filtro?
4. ¿Por qué una selección de tarea no debe obligar a renderizar las filas no seleccionadas?

---

### Paso 4. Crear el módulo de estado compartido inicial

**Objetivo:** centralizar únicamente el estado que realmente se comparte y preparar una frontera clara para futuros composables y stores de Pinia.

**Instrucciones:**

1. Crea el directorio de estado:

   ```bash
   mkdir -p src/state
   ```

2. Crea el archivo `src/state/projectHubState.js`:

   ```js
   import { computed, ref, shallowRef, watch } from 'vue'

   const initialProjects = Object.freeze([
     Object.freeze({
       id: 'project-alpha',
       name: 'Portal de clientes',
       owner: 'Ana Torres',
       status: 'En curso'
     }),
     Object.freeze({
       id: 'project-beta',
       name: 'Automatización QA',
       owner: 'Luis Mora',
       status: 'Planificación'
     })
   ])

   const initialTasks = Object.freeze([
     Object.freeze({
       id: 'task-1',
       projectId: 'project-alpha',
       title: 'Diseñar panel de métricas',
       status: 'Pendiente',
       priority: 'Alta'
     }),
     Object.freeze({
       id: 'task-2',
       projectId: 'project-alpha',
       title: 'Revisar accesibilidad',
       status: 'En curso',
       priority: 'Media'
     }),
     Object.freeze({
       id: 'task-3',
       projectId: 'project-beta',
       title: 'Definir casos E2E',
       status: 'Pendiente',
       priority: 'Alta'
     }),
     Object.freeze({
       id: 'task-4',
       projectId: 'project-beta',
       title: 'Preparar pipeline CI',
       status: 'Completada',
       priority: 'Media'
     })
   ])

   // Las colecciones se reemplazan como unidades inmutables.
   // Vue no convierte profundamente cada tarea o proyecto en un proxy.
   export const projects = shallowRef(initialProjects)
   export const tasks = shallowRef(initialTasks)

   // Estado realmente compartido entre zonas de la interfaz.
   export const selectedTaskId = ref('task-1')
   export const theme = ref('light')

   export const selectedTask = computed(() => {
     return tasks.value.find((task) => task.id === selectedTaskId.value) ?? null
   })

   export const totalCompletedTasks = computed(() => {
     return tasks.value.filter((task) => task.status === 'Completada').length
   })

   export function selectTask(taskId) {
     selectedTaskId.value = taskId
   }

   export function toggleTaskStatus(taskId) {
     tasks.value = tasks.value.map((task) => {
       if (task.id !== taskId) return task

       return Object.freeze({
         ...task,
         status: task.status === 'Completada' ? 'Pendiente' : 'Completada'
       })
     })
   }

   export function setTheme(nextTheme) {
     theme.value = nextTheme
   }

   // watch es apropiado porque la fuente y el efecto están explícitamente definidos.
   watch(
     theme,
     (nextTheme) => {
       document.documentElement.dataset.theme = nextTheme
     },
     { immediate: true }
   )
   ```

3. Analiza por qué `projects` y `tasks` usan `shallowRef`:

   - La colección es un conjunto de datos tratado como inmutable.
   - No se necesita observar cada propiedad interna de cada tarea.
   - Las actualizaciones reemplazan el arreglo completo.
   - Vue solo debe reaccionar a `tasks.value = nuevoArreglo`.

4. Analiza esta diferencia importante:

   ```js
   // No desencadena actualización con shallowRef:
   tasks.value[0].status = 'Completada'

   // Sí desencadena actualización:
   tasks.value = tasks.value.map((task) => ({
     ...task,
     status: task.id === 'task-1' ? 'Completada' : task.status
   }))
   ```

5. No desestructures directamente propiedades de un objeto `reactive()` si necesitas preservar su conexión reactiva. En este laboratorio se exportan `ref`, `shallowRef` y `computed` individuales, evitando ese problema.

**Salida esperada:**

Existe un módulo `src/state/projectHubState.js` que expone estado compartido, selectores derivados y acciones de modificación controladas.

**Verificación:**

Comprueba que el archivo contiene estas exportaciones:

```js
projects
tasks
selectedTaskId
theme
selectedTask
totalCompletedTasks
selectTask
toggleTaskStatus
setTheme
```

---

### Paso 5. Separar responsabilidades en componentes optimizados

**Objetivo:** evitar que filtros locales, selección y preferencias visuales se propaguen como un único objeto reactivo amplio.

**Instrucciones:**

1. Crea el directorio de componentes:

   ```bash
   mkdir -p src/components
   ```

2. Crea o reemplaza `src/components/ProjectPanel.vue`:

   ```vue
   <script setup>
   import { computed } from 'vue'
   import { projects, totalCompletedTasks } from '../state/projectHubState'
   import { probeRender } from '../utils/renderProbe'

   const activeProjects = computed(() => {
     return projects.value.filter((project) => project.status === 'En curso').length
   })
   </script>

   <template>
     <section class="panel">
       <span class="sr-only">{{ probeRender('ProjectPanel') }}</span>

       <h2>Proyectos</h2>
       <p class="summary">
         {{ activeProjects }} activos · {{ totalCompletedTasks }} tareas completadas
       </p>

       <ul class="project-list">
         <li v-for="project in projects" :key="project.id">
           <strong>{{ project.name }}</strong>
           <span>{{ project.owner }} · {{ project.status }}</span>
         </li>
       </ul>
     </section>
   </template>
   ```

3. Crea o reemplaza `src/components/TaskList.vue`:

   ```vue
   <script setup>
   import { computed, ref } from 'vue'
   import {
     selectedTaskId,
     selectTask,
     tasks,
     toggleTaskStatus
   } from '../state/projectHubState'
   import { probeRender } from '../utils/renderProbe'

   // El filtro pertenece solamente a este componente.
   const filterText = ref('')

   // computed evita ejecutar filter() repetidamente en cada interpolación.
   const filteredTasks = computed(() => {
     const normalizedFilter = filterText.value.trim().toLocaleLowerCase()

     if (!normalizedFilter) return tasks.value

     return tasks.value.filter((task) => {
       return task.title.toLocaleLowerCase().includes(normalizedFilter)
     })
   })

   const pendingTasks = computed(() => {
     return filteredTasks.value.filter((task) => task.status !== 'Completada').length
   })
   </script>

   <template>
     <section class="panel">
       <span class="sr-only">{{ probeRender('TaskList') }}</span>

       <div class="panel-header">
         <div>
           <h2>Tareas</h2>
           <p class="summary">{{ pendingTasks }} pendientes visibles</p>
         </div>

         <label>
           Filtrar
           <input
             v-model="filterText"
             type="search"
             placeholder="Buscar tarea"
           >
         </label>
       </div>

       <ul class="task-list">
         <li
           v-for="task in filteredTasks"
           :key="task.id"
           v-memo="[task.id === selectedTaskId, task.status]"
           :class="{ selected: task.id === selectedTaskId }"
         >
           <button class="task-main" type="button" @click="selectTask(task.id)">
             <strong>{{ task.title }}</strong>
             <span>{{ task.priority }} · {{ task.status }}</span>
           </button>

           <button
             class="task-action"
             type="button"
             @click.stop="toggleTaskStatus(task.id)"
           >
             {{ task.status === 'Completada' ? 'Reabrir' : 'Completar' }}
           </button>
         </li>
       </ul>

       <p v-if="filteredTasks.length === 0" class="empty-state">
         No hay tareas que coincidan con el filtro.
       </p>
     </section>
   </template>
   ```

4. Crea o reemplaza `src/components/PreferencesPanel.vue`:

   ```vue
   <script setup>
   import { setTheme, theme } from '../state/projectHubState'
   import { probeRender } from '../utils/renderProbe'
   </script>

   <template>
     <section class="panel">
       <span class="sr-only">{{ probeRender('PreferencesPanel') }}</span>

       <h2>Preferencias</h2>

       <label>
         Tema visual
         <select :value="theme" @change="setTheme($event.target.value)">
           <option value="light">Claro</option>
           <option value="dark">Oscuro</option>
         </select>
       </label>
     </section>
   </template>
   ```

5. Crea o reemplaza `src/App.vue`:

   ```vue
   <script setup>
   import ProjectPanel from './components/ProjectPanel.vue'
   import TaskList from './components/TaskList.vue'
   import PreferencesPanel from './components/PreferencesPanel.vue'
   import { theme } from './state/projectHubState'
   import { probeRender } from './utils/renderProbe'
   </script>

   <template>
     <main class="app-shell" :class="`theme-${theme}`">
       <span class="sr-only">{{ probeRender('App') }}</span>

       <header class="app-header">
         <p class="eyebrow">ProjectHub</p>
         <h1>Panel de trabajo</h1>
         <p>Diagnóstico de reactividad y optimización de renders.</p>
       </header>

       <div class="dashboard-grid">
         <ProjectPanel />
         <TaskList />
         <PreferencesPanel />
       </div>
     </main>
   </template>
   ```

6. Observa las fronteras de estado finales:

   | Dato | Ubicación | Motivo |
   |---|---|---|
   | `filterText` | `TaskList.vue` | Solo afecta el listado de tareas |
   | `projects` | `projectHubState.js` | Datos compartidos y base para futuros stores |
   | `tasks` | `projectHubState.js` | Datos compartidos entre lista, selección y métricas |
   | `selectedTaskId` | `projectHubState.js` | Selección potencialmente consumible por otras zonas |
   | `theme` | `projectHubState.js` | Preferencia visual global |
   | `filteredTasks` | `TaskList.vue` | Estado derivado local, no debe persistirse ni compartirse |

**Salida esperada:**

La aplicación queda dividida en tres componentes funcionales. El filtro ya no se almacena en un objeto global de interfaz y cada componente consume solamente el estado que necesita.

**Verificación:**

En `TaskList.vue`, confirma que `filterText` no se pasa como prop a `App.vue`, `ProjectPanel.vue` ni `PreferencesPanel.vue`.

---

### Paso 6. Aplicar estilos mínimos y comprobar la interfaz

**Objetivo:** asegurar una visualización clara de los estados de selección, tema y lista de tareas.

**Instrucciones:**

1. Reemplaza o complementa `src/style.css` con el siguiente contenido:

   ```css
   :root {
     font-family: Inter, system-ui, sans-serif;
     color: #172033;
     background: #eef2f8;
   }

   :root[data-theme='dark'] {
     color: #e8edf8;
     background: #121826;
   }

   * {
     box-sizing: border-box;
   }

   body {
     margin: 0;
     min-width: 320px;
   }

   button,
   input,
   select {
     font: inherit;
   }

   .app-shell {
     min-height: 100vh;
     padding: 2rem;
     background: #eef2f8;
   }

   .theme-dark {
     background: #121826;
     color: #e8edf8;
   }

   .app-header,
   .dashboard-grid {
     max-width: 1100px;
     margin: 0 auto;
   }

   .app-header {
     margin-bottom: 1.5rem;
   }

   .app-header h1,
   .panel h2 {
     margin: 0;
   }

   .eyebrow,
   .summary {
     color: #667085;
   }

   .theme-dark .summary,
   .theme-dark .eyebrow {
     color: #aab6cf;
   }

   .dashboard-grid {
     display: grid;
     grid-template-columns: repeat(auto-fit, minmax(260px, 1fr));
     gap: 1rem;
     align-items: start;
   }

   .panel {
     padding: 1rem;
     border: 1px solid #d0d7e5;
     border-radius: 12px;
     background: #ffffff;
     box-shadow: 0 4px 16px rgb(20 31 56 / 8%);
   }

   .theme-dark .panel {
     border-color: #32405c;
     background: #1b2538;
   }

   .panel-header {
     display: flex;
     gap: 1rem;
     justify-content: space-between;
     align-items: start;
   }

   .panel-header label,
   .panel > label {
     display: grid;
     gap: 0.35rem;
   }

   input,
   select,
   button {
     border: 1px solid #aeb9cc;
     border-radius: 6px;
     padding: 0.5rem 0.65rem;
   }

   .project-list,
   .task-list {
     display: grid;
     gap: 0.65rem;
     padding: 0;
     list-style: none;
   }

   .project-list li,
   .task-list li {
     display: flex;
     gap: 0.75rem;
     justify-content: space-between;
     padding: 0.75rem;
     border: 1px solid #dbe2ef;
     border-radius: 8px;
   }

   .project-list span,
   .task-main span {
     display: block;
     margin-top: 0.2rem;
     color: #667085;
     font-size: 0.875rem;
   }

   .task-list li.selected {
     border-color: #356ae6;
     box-shadow: 0 0 0 2px rgb(53 106 230 / 16%);
   }

   .task-main {
     flex: 1;
     padding: 0;
     border: 0;
     color: inherit;
     text-align: left;
     background: transparent;
     cursor: pointer;
   }

   .task-action {
     align-self: center;
     cursor: pointer;
   }

   .empty-state {
     color: #667085;
   }

   .sr-only {
     position: absolute;
     width: 1px;
     height: 1px;
     padding: 0;
     margin: -1px;
     overflow: hidden;
     clip: rect(0, 0, 0, 0);
     white-space: nowrap;
     border: 0;
   }
   ```

2. Guarda todos los archivos.

3. En el navegador, comprueba que puedes:

   - Escribir texto en el filtro.
   - Seleccionar una tarea.
   - Completar y reabrir una tarea.
   - Alternar entre tema claro y oscuro.

**Salida esperada:**

ProjectHub presenta tres paneles funcionales, filas de tarea seleccionables y un cambio visible de tema.

**Verificación:**

Al completar una tarea, el contador de tareas pendientes visibles disminuye cuando corresponda. Esto confirma que `tasks.value` se reemplaza y que los `computed` dependientes se recalculan.

---

### Paso 7. Verificar el efecto de `computed`, `shallowRef` y `v-memo`

**Objetivo:** medir el comportamiento optimizado y explicar qué optimización corresponde a cada necesidad.

**Instrucciones:**

1. Abre la consola y limpia las marcas:

   ```js
   performance.clearMarks()
   console.clear()
   ```

2. Cambia el tema visual de claro a oscuro.

3. Observa los mensajes de render. En la arquitectura optimizada:

   - `App` puede renderizar porque consume `theme`.
   - `PreferencesPanel` puede renderizar porque consume `theme`.
   - `TaskList` no debería necesitar renderizar por el cambio de tema.
   - `ProjectPanel` no debería necesitar renderizar por el cambio de tema.

4. Limpia nuevamente consola y marcas:

   ```js
   performance.clearMarks()
   console.clear()
   ```

5. Escribe un filtro en el campo de tareas.

6. Comprueba que el cambio afecta a `TaskList`, pero no requiere renderizar `PreferencesPanel`.

7. Limpia nuevamente consola y marcas.

8. Selecciona una tarea diferente.

9. En la lista, observa que solo la fila antes seleccionada y la nueva fila seleccionada cambian visualmente. `v-memo` permite a Vue reutilizar subárboles de filas cuyas dependencias declaradas no han cambiado.

10. Completa una tarea y verifica que:

    - Se reemplaza el arreglo almacenado en `tasks`.
    - Se actualiza la fila afectada.
    - Se actualizan los `computed` que dependen de `tasks`.
    - El panel de proyectos puede actualizar su métrica global de tareas completadas, porque esa métrica sí es una dependencia real.

11. En la consola, inspecciona las marcas recientes:

    ```js
    performance.getEntriesByType('mark')
      .filter((entry) => entry.name.startsWith('render:'))
      .map((entry) => entry.name)
    ```

12. Resume el resultado esperado:

   | Técnica | Problema que resuelve | Evidencia observable |
   |---|---|---|
   | Separación de estado | Dependencias entre zonas no relacionadas | Filtro no actualiza preferencias |
   | `computed` | Repetir filtros y conteos en cada render | Lista y contador derivan de una fuente clara |
   | `shallowRef` | Reactividad profunda innecesaria en colecciones inmutables | La actualización ocurre al reemplazar el arreglo |
   | `v-memo` | Recrear subárboles de filas estables | Las filas no afectadas conservan su subárbol virtual |
   | `watch` | Sincronizar una fuente concreta con un efecto externo | `data-theme` cambia en `<html>` |

**Salida esperada:**

Los contadores de render demuestran una reducción de actualizaciones en zonas no relacionadas. La funcionalidad de ProjectHub se mantiene.

**Verificación:**

En la consola, ejecuta:

```js
document.documentElement.dataset.theme
```

El valor debe coincidir con el tema seleccionado, por ejemplo:

```text
dark
```

Esto verifica que el `watch(theme, ...)` se ejecuta como efecto secundario controlado.

---

## Validación y pruebas

Completa las siguientes pruebas manuales. Reinicia los contadores antes de cada una con:

```js
performance.clearMarks()
console.clear()
```

| Prueba | Acción | Resultado funcional esperado | Resultado de render esperado |
|---|---|---|---|
| Filtro local | Escribir `pipeline` | Se muestra “Preparar pipeline CI” | Se actualiza `TaskList`; preferencias no requieren render |
| Filtro vacío | Borrar el filtro | Se muestran todas las tareas | Se actualiza `TaskList` |
| Selección | Seleccionar una tarea distinta | La nueva fila queda resaltada | Cambia `TaskList`; filas estables se benefician de `v-memo` |
| Cambio de estado | Pulsar “Completar” | La tarea cambia a `Completada` | Se actualiza la lista y métricas dependientes |
| Tema | Seleccionar “Oscuro” | Cambia el estilo global | Se actualizan `App` y preferencias; la lista no depende del tema |
| Tema inverso | Volver a “Claro” | Se restaura el estilo claro | Mismo patrón de dependencias que en la prueba anterior |

Ejecuta también una validación de compilación:

```bash
npm run build
```

El resultado esperado incluye una salida similar a:

```text
✓ built in ...
```

Finalmente, revisa los cambios pendientes:

```bash
git status
```

Debes ver archivos equivalentes a los siguientes:

```text
src/App.vue
src/components/PreferencesPanel.vue
src/components/ProjectPanel.vue
src/components/TaskList.vue
src/state/projectHubState.js
src/style.css
src/utils/renderProbe.js
```

---

## Resolución de problemas

### Problema 1: `v-memo` no parece reducir actualizaciones o la lista no refleja un cambio

**Síntoma:** al completar una tarea, la interfaz no cambia, o todas las filas parecen reconstruirse aunque las tareas no hayan cambiado.

**Causa:** con `shallowRef`, Vue solo observa el reemplazo de `tasks.value`. Una mutación profunda como `tasks.value[0].status = 'Completada'` no notifica automáticamente una actualización. Además, `v-memo` solo puede reutilizar una fila si las expresiones de su arreglo de dependencias permanecen iguales.

**Solución:** actualiza la colección de forma inmutable mediante reemplazo, como hace `toggleTaskStatus()`:

```js
tasks.value = tasks.value.map((task) => {
  if (task.id !== taskId) return task

  return {
    ...task,
    status: 'Completada'
  }
})
```

Mantén una clave estable en el `v-for` y dependencias relevantes en `v-memo`:

```vue
<li
  v-for="task in filteredTasks"
  :key="task.id"
  v-memo="[task.id === selectedTaskId, task.status]"
>
```

### Problema 2: Vue Devtools no detecta la aplicación o no aparecen contadores en consola

**Síntoma:** Vue Devtools muestra “No Vue.js app detected”, o no aparecen mensajes `[render] ...` al interactuar con la interfaz.

**Causa:** la aplicación puede haberse abierto en una URL distinta al servidor Vite requerido, la extensión Vue Devtools puede estar deshabilitada, o `probeRender()` no se invocó desde la plantilla del componente.

**Solución:** confirma que estás usando exactamente la URL local del laboratorio:

```text
http://127.0.0.1:5173
```

Verifica que el servidor se inició con:

```bash
npm run dev -- --host 127.0.0.1 --port 5173
```

Después, revisa que cada componente importe la utilidad y contenga una llamada en su plantilla:

```vue
<span class="sr-only">{{ probeRender('TaskList') }}</span>
```

Recarga la página sin caché con `Ctrl+Shift+R` o `Cmd+Shift+R` y vuelve a abrir Vue Devtools.

---

## Limpieza

1. Detén el servidor Vite en la terminal con:

   ```text
   Ctrl+C
   ```

2. Opcionalmente, elimina las marcas de rendimiento de la sesión actual en Chrome DevTools:

   ```js
   performance.clearMarks()
   ```

3. Conserva los archivos creados. Son la entrada requerida para el siguiente laboratorio, donde el módulo temporal `src/state/projectHubState.js` evolucionará hacia composables y/o un store de Pinia.

4. Verifica el estado final del repositorio:

   ```bash
   git status
   ```

5. No cambies de rama ni crees ramas adicionales; el curso continúa utilizando `main`.

## Resumen

En este laboratorio aplicaste el modelo de reactividad de Vue para diagnosticar renders y reducir actualizaciones innecesarias. Instrumentaste componentes con `performance.mark()` y `console.count()`, observaste dependencias con Vue Devtools y separaste el estado local del estado compartido.

La solución final usa:

- `ref` para valores independientes como tema, filtro y selección.
- `computed` para datos derivados y cacheados.
- `watch` para centralizar el efecto secundario de sincronización del tema.
- `shallowRef` para colecciones tratadas como datos inmutables.
- `v-memo` para reutilizar filas estables de una lista.
- `src/state/projectHubState.js` como frontera temporal de estado compartido.

Como siguiente paso, esta estructura permitirá extraer lógica reutilizable mediante composables y migrar el estado global a una arquitectura modular con Pinia.

### Recursos opcionales

- [Vue: fundamentos de reactividad](https://es.vuejs.org/guide/essentials/reactivity-fundamentals.html)
- [Vue: reactividad en profundidad](https://vuejs.org/guide/extras/reactivity-in-depth.html)
- [Vue: propiedades calculadas](https://es.vuejs.org/guide/essentials/computed.html)
- [Vue: optimización de rendimiento](https://vuejs.org/guide/best-practices/performance.html)
- [Chrome DevTools: Performance](https://developer.chrome.com/docs/devtools/performance)
