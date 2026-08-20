# 4 Practica: refactorizar componentes a Composition API y crear composables

## Metadatos

| Campo | Valor |
|---|---|
| Duración | 30 minutos |
| Complejidad | Media |
| Nivel de Bloom | Crear |

## Descripción general

En este laboratorio transformarás la lógica compartida de ProjectHub desde un módulo de estado centralizado hacia composables cohesionados basados en Composition API. Crearás composables para filtrado, selección, preferencias visuales y estados asíncronos, conservando la reactividad y el comportamiento optimizado del laboratorio anterior.

También implementarás un contexto distribuido con `provide/inject` y un componente renderless que expone datos y acciones mediante un slot con alcance. El resultado será una base reutilizable para la autenticación, las rutas protegidas y los guards del siguiente bloque.

## Objetivos de aprendizaje

Al finalizar el laboratorio podrás:

- [ ] Extraer lógica de interfaz y estado compartido a composables con APIs explícitas.
- [ ] Implementar filtros, ordenamiento y contadores derivados mediante `computed()`.
- [ ] Gestionar selección de tareas y navegación mediante un composable desacoplado.
- [ ] Persistir preferencias visuales de forma controlada usando `watch()` y `onScopeDispose()`.
- [ ] Usar `provide/inject` y un componente renderless con slots con alcance para evitar prop drilling.

## Prerrequisitos

### Conocimientos requeridos

- Haber completado el laboratorio `01-00-01` con la SPA ProjectHub optimizada.
- Comprender el uso de `ref`, `computed`, `watch`, `watchEffect` y `onScopeDispose`.
- Distinguir entre estado local, estado compartido, estado derivado y efectos secundarios.
- Conocer la sintaxis de `<script setup>` en Vue 3.
- Comprender el patrón de slots con alcance (`v-slot` o `#default`).

### Acceso y entorno requeridos

- Acceso local al repositorio de la aplicación `projecthub-spa`.
- Permisos para ejecutar comandos de Node.js y npm.
- Navegador Google Chrome instalado.
- Puerto local `5173` disponible.
- No iniciar servicios en el puerto `3000`; las solicitudes simuladas continúan siendo interceptadas por MSW en el navegador.

## Entorno del laboratorio

### Software esperado

| Herramienta | Versión esperada |
|---|---:|
| Node.js | 22.14.0 |
| npm | 10.9.2 |
| Vue | 3.5.13 |
| Vite | 6.1.0 |
| Vue Router | 4.5.0 |
| Pinia | 3.0.1 |
| Visual Studio Code | 1.97.2 o superior |
| Google Chrome | 132 o superior |

### Preparación inicial

Abre una terminal y sitúate en el directorio obligatorio.

```bash
cd /workspace/vue-intermedio-labs/projecthub-spa
```

En Windows:

```powershell
cd C:\workspace\vue-intermedio-labs\projecthub-spa
```

Comprueba las versiones instaladas:

```bash
node --version
npm --version
```

Instala las dependencias únicamente si todavía no existe `node_modules`:

```bash
npm install
```

Comprueba el estado inicial del repositorio:

```bash
git status
git branch --show-current
```

La salida esperada de la rama es:

```text
main
```

> No crees ramas adicionales. El curso conserva una única rama de trabajo denominada `main`.

---

## Desarrollo paso a paso

### Paso 1. Inspeccionar el estado centralizado existente

**Objetivo:** Identificar las responsabilidades que deben salir de `src/state/projectHubState.js`.

**Instrucciones:**

1. Abre el archivo de estado creado en el laboratorio anterior:

   ```bash
   code src/state/projectHubState.js
   ```

2. Identifica las responsabilidades que actualmente concentra. Normalmente encontrarás una combinación de:
   - Lista de tareas.
   - Búsqueda por texto.
   - Filtros de estado.
   - Ordenamiento.
   - Tarea seleccionada.
   - Preferencias de densidad o tema.
   - Estado de carga, error y éxito.
   - Funciones para cargar o refrescar tareas.

3. Crea la estructura de directorios para los composables y las claves de inyección:

   ```bash
   mkdir -p src/composables src/keys src/components/tasks
   ```

4. Conserva temporalmente `src/state/projectHubState.js` como referencia, pero no lo importes en los archivos nuevos.

5. Crea los archivos que se implementarán durante el laboratorio:

   ```bash
   touch src/composables/useProjectFilters.js
   touch src/composables/useTaskSelection.js
   touch src/composables/useLocalPreferences.js
   touch src/composables/useAsyncState.js
   touch src/keys/projectHub.js
   touch src/components/tasks/TaskCollectionProvider.vue
   ```

**Resultado esperado:**

La estructura relevante del proyecto debe incluir los nuevos archivos:

```text
src/
├── composables/
│   ├── useAsyncState.js
│   ├── useLocalPreferences.js
│   ├── useProjectFilters.js
│   └── useTaskSelection.js
├── components/
│   └── tasks/
│       └── TaskCollectionProvider.vue
├── keys/
│   └── projectHub.js
└── state/
    └── projectHubState.js
```

**Verificación:**

Ejecuta:

```bash
find src/composables src/keys src/components/tasks -type f
```

En Windows PowerShell:

```powershell
Get-ChildItem src/composables, src/keys, src/components/tasks -Recurse -File
```

Confirma que existen los cinco archivos nuevos.

---

### Paso 2. Crear el composable de filtros y ordenamiento

**Objetivo:** Encapsular el filtrado, ordenamiento y conteo de tareas en `useProjectFilters()`.

**Instrucciones:**

1. Abre `src/composables/useProjectFilters.js`.

2. Implementa el composable siguiente. Recibe una referencia reactiva de tareas para que pueda trabajar con listas reemplazables después de una carga asíncrona.

   ```js
   import { computed, ref } from 'vue'

   export function useProjectFilters(tasks) {
     const searchTerm = ref('')
     const statusFilter = ref('all')
     const sortBy = ref('updatedAt-desc')

     const normalizedSearchTerm = computed(() =>
       searchTerm.value.trim().toLocaleLowerCase()
     )

     const filteredTasks = computed(() => {
       const result = tasks.value.filter((task) => {
         const matchesSearch =
           !normalizedSearchTerm.value ||
           task.title.toLocaleLowerCase().includes(normalizedSearchTerm.value)

         const matchesStatus =
           statusFilter.value === 'all' ||
           task.status === statusFilter.value

         return matchesSearch && matchesStatus
       })

       return [...result].sort((firstTask, secondTask) => {
         if (sortBy.value === 'title-asc') {
           return firstTask.title.localeCompare(secondTask.title)
         }

         if (sortBy.value === 'priority-desc') {
           return secondTask.priority - firstTask.priority
         }

         return new Date(secondTask.updatedAt) - new Date(firstTask.updatedAt)
       })
     })

     const totalTasks = computed(() => tasks.value.length)

     const visibleTasks = computed(() => filteredTasks.value.length)

     const completedTasks = computed(() =>
       tasks.value.filter((task) => task.status === 'completed').length
     )

     function clearFilters() {
       searchTerm.value = ''
       statusFilter.value = 'all'
       sortBy.value = 'updatedAt-desc'
     }

     return {
       searchTerm,
       statusFilter,
       sortBy,
       filteredTasks,
       totalTasks,
       visibleTasks,
       completedTasks,
       clearFilters
     }
   }
   ```

3. Revisa que el composable no cargue datos, no acceda al DOM y no persista información. Su única responsabilidad es transformar la colección recibida y mantener el estado de filtros.

4. Observa que `filteredTasks` es un `computed()`, no una lista actualizada manualmente. Esto evita duplicar el estado y conserva la optimización reactiva.

**Resultado esperado:**

El composable ofrece una API explícita:

| Elemento | Responsabilidad |
|---|---|
| `searchTerm` | Texto de búsqueda editable por la interfaz |
| `statusFilter` | Estado de tarea seleccionado |
| `sortBy` | Criterio de ordenamiento |
| `filteredTasks` | Lista derivada y ordenada |
| `totalTasks` | Total de tareas cargadas |
| `visibleTasks` | Cantidad de tareas visibles |
| `completedTasks` | Cantidad global de tareas completadas |
| `clearFilters` | Restablecimiento de filtros |

**Verificación:**

Comprueba que no existen efectos secundarios en el composable:

```bash
grep -nE "fetch|localStorage|window|document" src/composables/useProjectFilters.js
```

El comando no debe devolver resultados.

---

### Paso 3. Crear el composable de selección y navegación

**Objetivo:** Desacoplar la selección de tareas de Vue Router mediante una función de navegación inyectable.

**Instrucciones:**

1. Abre `src/composables/useTaskSelection.js`.

2. Implementa el composable:

   ```js
   import { computed, ref } from 'vue'

   export function useTaskSelection({ navigate } = {}) {
     const selectedTaskId = ref(null)

     const hasSelection = computed(() => selectedTaskId.value !== null)

     function isSelected(taskId) {
       return selectedTaskId.value === taskId
     }

     async function selectTask(task) {
       selectedTaskId.value = task.id

       if (navigate) {
         await navigate(task.id)
       }
     }

     function clearSelection() {
       selectedTaskId.value = null
     }

     return {
       selectedTaskId,
       hasSelection,
       isSelected,
       selectTask,
       clearSelection
     }
   }
   ```

3. Observa que el composable no importa `useRouter()` ni `useRoute()`. Esta decisión mantiene la lógica reutilizable:
   - Un componente puede navegar mediante Vue Router.
   - Una prueba unitaria puede entregar una función simulada.
   - Otro consumidor puede seleccionar tareas sin cambiar la URL.

4. El contrato del argumento `navigate` es el siguiente:

   ```js
   async function navigate(taskId) {
     // Navegación o sincronización externa.
   }
   ```

5. La selección se actualiza antes de navegar. De este modo, la interfaz refleja inmediatamente la intención del usuario, incluso si la navegación tarda una fracción de segundo.

**Resultado esperado:**

`useTaskSelection()` mantiene el estado de selección y delega la navegación a una dependencia externa en lugar de acoplarse al router.

**Verificación:**

Ejecuta:

```bash
grep -n "vue-router" src/composables/useTaskSelection.js
```

No debe aparecer ninguna importación ni referencia a Vue Router.

---

### Paso 4. Crear el composable de preferencias locales

**Objetivo:** Persistir preferencias visuales de forma controlada y limpiar el observador al destruir el ámbito reactivo.

**Instrucciones:**

1. Abre `src/composables/useLocalPreferences.js`.

2. Implementa el composable:

   ```js
   import { onScopeDispose, ref, watch } from 'vue'

   const DEFAULT_PREFERENCES = {
     compactMode: false,
     theme: 'light'
   }

   function readPreferences(storageKey) {
     try {
       const storedValue = window.localStorage.getItem(storageKey)

       if (!storedValue) {
         return { ...DEFAULT_PREFERENCES }
       }

       return {
         ...DEFAULT_PREFERENCES,
         ...JSON.parse(storedValue)
       }
     } catch {
       return { ...DEFAULT_PREFERENCES }
     }
   }

   export function useLocalPreferences(storageKey = 'projecthub:preferences') {
     const preferences = ref(readPreferences(storageKey))

     const stopPersistence = watch(
       preferences,
       (value) => {
         try {
           window.localStorage.setItem(storageKey, JSON.stringify(value))
         } catch {
           // La interfaz sigue funcionando si el almacenamiento no está disponible.
         }
       },
       { deep: true }
     )

     function toggleCompactMode() {
       preferences.value.compactMode = !preferences.value.compactMode
     }

     function setTheme(theme) {
       preferences.value.theme = theme === 'dark' ? 'dark' : 'light'
     }

     function resetPreferences() {
       preferences.value = { ...DEFAULT_PREFERENCES }
     }

     onScopeDispose(() => {
       stopPersistence()
     })

     return {
       preferences,
       toggleCompactMode,
       setTheme,
       resetPreferences
     }
   }
   ```

3. Este composable usa `ref()` para permitir reemplazar completamente el objeto de preferencias durante `resetPreferences()`.

4. El efecto secundario de `localStorage` se realiza dentro de `watch()`, no en un `computed()`. Las propiedades calculadas deben permanecer libres de efectos externos.

5. `onScopeDispose()` registra la limpieza en el ámbito del componente que invoque el composable. Aunque Vue detiene automáticamente los observadores creados dentro de `setup()`, registrar la liberación expresa la intención y permite reutilizar el composable en ámbitos de efecto avanzados.

**Resultado esperado:**

Las preferencias quedan almacenadas bajo la clave:

```text
projecthub:preferences
```

El valor almacenado tendrá una forma similar a:

```json
{
  "compactMode": true,
  "theme": "dark"
}
```

**Verificación:**

Cuando la aplicación esté en ejecución, abre Chrome DevTools:

1. Presiona `F12`.
2. Abre **Application**.
3. Selecciona **Local Storage**.
4. Abre `http://127.0.0.1:5173`.
5. Confirma que aparece la clave `projecthub:preferences` después de cambiar una preferencia.

---

### Paso 5. Crear el composable para estados asíncronos

**Objetivo:** Centralizar los estados de carga, éxito, error y cancelación de operaciones asíncronas.

**Instrucciones:**

1. Abre `src/composables/useAsyncState.js`.

2. Implementa el composable:

   ```js
   import { computed, onScopeDispose, ref } from 'vue'

   export function useAsyncState() {
     const status = ref('idle')
     const data = ref(null)
     const error = ref(null)
     let controller = null

     const isIdle = computed(() => status.value === 'idle')
     const isLoading = computed(() => status.value === 'loading')
     const isSuccess = computed(() => status.value === 'success')
     const isError = computed(() => status.value === 'error')

     async function execute(operation) {
       controller?.abort()
       controller = new AbortController()

       status.value = 'loading'
       error.value = null

       try {
         const result = await operation(controller.signal)

         data.value = result
         status.value = 'success'

         return result
       } catch (caughtError) {
         if (caughtError.name === 'AbortError') {
           status.value = 'idle'
           return null
         }

         error.value = caughtError
         status.value = 'error'

         throw caughtError
       }
     }

     function reset() {
       controller?.abort()
       controller = null
       status.value = 'idle'
       data.value = null
       error.value = null
     }

     onScopeDispose(() => {
       controller?.abort()
     })

     return {
       status,
       data,
       error,
       isIdle,
       isLoading,
       isSuccess,
       isError,
       execute,
       reset
     }
   }
   ```

3. Comprende el flujo implementado:
   - `execute()` cancela una operación anterior antes de iniciar otra.
   - Cada operación recibe un `AbortSignal`.
   - Un error de cancelación no se presenta como error de interfaz.
   - Otros errores se guardan y se propagan para que el componente pueda decidir cómo mostrarlos.
   - `onScopeDispose()` cancela solicitudes pendientes al desmontar la vista.

4. Si tu servicio existente usa `fetch`, asegúrate de recibir y reenviar la señal:

   ```js
   export async function fetchProjectTasks({ signal } = {}) {
     const response = await fetch('/api/tasks', { signal })

     if (!response.ok) {
       throw new Error('Unable to load tasks')
     }

     return response.json()
   }
   ```

**Resultado esperado:**

El composable ofrece indicadores reactivos para representar la interfaz:

```text
idle → loading → success
idle → loading → error
loading → idle (cuando se cancela)
```

**Verificación:**

Comprueba que existe limpieza de solicitudes pendientes:

```bash
grep -n "onScopeDispose" src/composables/useAsyncState.js
```

La salida debe mostrar la importación y el bloque de cancelación.

---

### Paso 6. Definir las claves de inyección y el contexto compartido

**Objetivo:** Crear símbolos de inyección seguros para distribuir el contexto de ProjectHub y una sesión simulada sin prop drilling.

**Instrucciones:**

1. Abre `src/keys/projectHub.js`.

2. Implementa las claves y el helper de consumo:

   ```js
   import { inject } from 'vue'

   export const projectHubKey = Symbol('projectHub')
   export const projectHubSessionKey = Symbol('projectHubSession')

   export function useProjectHubContext() {
     const context = inject(projectHubKey, null)

     if (!context) {
       throw new Error(
         'useProjectHubContext debe utilizarse dentro de ProjectHubView.'
       )
     }

     return context
   }
   ```

3. Usa `Symbol()` en vez de una cadena. Esto reduce el riesgo de colisiones con otros contextos inyectados.

4. El helper `useProjectHubContext()` evita errores silenciosos. Si un componente consumidor se monta fuera del proveedor, el error indica inmediatamente la causa arquitectónica.

5. La clave `projectHubSessionKey` prepara el contexto de sesión simulada que será ampliado y consumido por rutas y guards en el laboratorio `03-00-01`.

**Resultado esperado:**

El archivo expone dos claves únicas y una función segura para consumir el contexto de ProjectHub.

**Verificación:**

Ejecuta:

```bash
cat src/keys/projectHub.js
```

Confirma que las claves se crean con `Symbol('projectHub')` y `Symbol('projectHubSession')`.

---

### Paso 7. Proveer el contexto desde `ProjectHubView.vue`

**Objetivo:** Componer los nuevos composables en la vista principal y distribuirlos mediante `provide()`.

**Instrucciones:**

1. Abre la vista principal. Según la estructura del laboratorio anterior, normalmente será:

   ```bash
   code src/views/ProjectHubView.vue
   ```

2. Elimina la importación de `projectHubState.js`.

3. Adapta el bloque `<script setup>` para crear el contexto. Conserva la función o servicio de carga que ya estuviera implementado en el laboratorio anterior. El siguiente ejemplo supone que existe `fetchProjectTasks()` en `src/services/projectService.js`.

   ```vue
   <script setup>
   import { onMounted, provide, readonly, ref } from 'vue'
   import { useRoute, useRouter } from 'vue-router'
   import TaskCollectionProvider from '@/components/tasks/TaskCollectionProvider.vue'
   import { useAsyncState } from '@/composables/useAsyncState'
   import { useLocalPreferences } from '@/composables/useLocalPreferences'
   import { useProjectFilters } from '@/composables/useProjectFilters'
   import { useTaskSelection } from '@/composables/useTaskSelection'
   import {
     projectHubKey,
     projectHubSessionKey
   } from '@/keys/projectHub'
   import { fetchProjectTasks } from '@/services/projectService'

   const router = useRouter()
   const route = useRoute()

   const tasks = ref([])

   const session = ref({
     user: {
       id: 'demo-user',
       name: 'Ada Lovelace',
       role: 'member'
     },
     isAuthenticated: true
   })

   const filters = useProjectFilters(tasks)
   const preferences = useLocalPreferences()
   const asyncState = useAsyncState()

   const selection = useTaskSelection({
     navigate: (taskId) =>
       router.push({
         query: {
           ...route.query,
           task: taskId
         }
       })
   })

   async function reloadTasks() {
     return asyncState.execute(async (signal) => {
       const loadedTasks = await fetchProjectTasks({ signal })
       tasks.value = loadedTasks
       return loadedTasks
     })
   }

   const projectHubContext = {
     tasks: readonly(tasks),
     filters,
     preferences,
     asyncState,
     selection,
     reloadTasks
   }

   provide(projectHubKey, projectHubContext)
   provide(projectHubSessionKey, readonly(session))

   onMounted(() => {
     reloadTasks().catch(() => {
       // El mensaje se representa mediante asyncState.error.
     })
   })
   </script>
   ```

4. Si tu servicio anterior se llama de otra manera, conserva su importación y adapta solamente su llamada para aceptar `{ signal }`.

5. No conviertas `tasks` en un objeto global mutable. Se entrega como `readonly(tasks)` para que los consumidores puedan leer las tareas, pero no reemplazarlas directamente. La vista proveedora mantiene la responsabilidad de cargar y actualizar la colección.

6. Agrega una estructura básica de plantilla que use el proveedor renderless. El componente se implementará en el siguiente paso:

   ```vue
   <template>
     <section
       class="project-hub"
       :class="{
         'project-hub--compact': preferences.preferences.value.compactMode
       }"
     >
       <header class="project-hub__header">
         <div>
           <h1>ProjectHub</h1>
           <p v-if="session.user">
             Sesión simulada: {{ session.user.name }}
           </p>
         </div>

         <button
           type="button"
           @click="preferences.toggleCompactMode"
         >
           Cambiar densidad
         </button>
       </header>

       <p v-if="asyncState.isLoading" role="status">
         Cargando tareas…
       </p>

       <p v-else-if="asyncState.isError" role="alert">
         No fue posible cargar las tareas:
         {{ asyncState.error?.message }}
       </p>

       <TaskCollectionProvider v-else>
         <template #default="collection">
           <!-- El contenido se completará en el paso siguiente. -->
           <p aria-live="polite">
             Mostrando {{ collection.visibleTasks }} de
             {{ collection.totalTasks }} tareas.
           </p>
         </template>
       </TaskCollectionProvider>
     </section>
   </template>
   ```

**Resultado esperado:**

`ProjectHubView.vue` se convierte en el punto de composición de la funcionalidad:

```text
ProjectHubView
├── tasks
├── useProjectFilters(tasks)
├── useTaskSelection(navigate)
├── useLocalPreferences()
├── useAsyncState()
├── provide(projectHubKey, contexto)
└── provide(projectHubSessionKey, sesión simulada)
```

**Verificación:**

Busca referencias al estado anterior:

```bash
grep -R "projectHubState" -n src
```

La única coincidencia permitida durante la transición puede ser el propio archivo antiguo. No debe existir una importación activa desde componentes o vistas.

---

### Paso 8. Construir el componente renderless `TaskCollectionProvider`

**Objetivo:** Reutilizar la lógica de colección sin imponer una estructura visual mediante un slot con alcance.

**Instrucciones:**

1. Abre `src/components/tasks/TaskCollectionProvider.vue`.

2. Implementa el componente renderless:

   ```vue
   <script setup>
   import { useProjectHubContext } from '@/keys/projectHub'

   const {
     filters,
     selection
   } = useProjectHubContext()
   </script>

   <template>
     <slot
       :tasks="filters.filteredTasks"
       :total-tasks="filters.totalTasks"
       :visible-tasks="filters.visibleTasks"
       :completed-tasks="filters.completedTasks"
       :search-term="filters.searchTerm"
       :status-filter="filters.statusFilter"
       :sort-by="filters.sortBy"
       :clear-filters="filters.clearFilters"
       :is-selected="selection.isSelected"
       :select-task="selection.selectTask"
       :clear-selection="selection.clearSelection"
     />
   </template>
   ```

3. El componente no contiene etiquetas visuales propias, estilos ni una lista `<ul>`. Su única salida es un `slot`.

4. Los atributos en `kebab-case`, como `:visible-tasks`, se exponen al consumidor como `visibleTasks`. Vue normaliza los nombres de atributos de slots.

5. Este patrón permite reutilizar la misma lógica con representaciones diferentes:
   - Lista de tarjetas.
   - Tabla accesible.
   - Tablero Kanban.
   - Lista virtualizada.
   - Vista móvil compacta.

6. Vuelve a `src/views/ProjectHubView.vue` y reemplaza el contenido del slot por una interfaz funcional:

   ```vue
   <TaskCollectionProvider v-else>
     <template #default="collection">
       <div class="project-hub__filters">
         <label>
           Buscar tarea
           <input
             :value="collection.searchTerm"
             type="search"
             placeholder="Buscar por título"
             @input="collection.searchTerm = $event.target.value"
           >
         </label>

         <label>
           Estado
           <select v-model="collection.statusFilter">
             <option value="all">Todos</option>
             <option value="pending">Pendientes</option>
             <option value="in-progress">En progreso</option>
             <option value="completed">Completadas</option>
           </select>
         </label>

         <label>
           Ordenar por
           <select v-model="collection.sortBy">
             <option value="updatedAt-desc">Actualización reciente</option>
             <option value="title-asc">Título</option>
             <option value="priority-desc">Prioridad</option>
           </select>
         </label>

         <button type="button" @click="collection.clearFilters">
           Limpiar filtros
         </button>
       </div>

       <p aria-live="polite">
         Mostrando {{ collection.visibleTasks }} de
         {{ collection.totalTasks }} tareas.
         Completadas: {{ collection.completedTasks }}.
       </p>

       <ul v-if="collection.tasks.length" class="task-list">
         <li
           v-for="task in collection.tasks"
           :key="task.id"
           :class="{ 'task-list__item--selected': collection.isSelected(task.id) }"
         >
           <button
             type="button"
             class="task-list__item"
             @click="collection.selectTask(task)"
           >
             <strong>{{ task.title }}</strong>
             <span>{{ task.status }}</span>
             <span>Prioridad: {{ task.priority }}</span>
           </button>
         </li>
       </ul>

       <p v-else>
         No hay tareas que coincidan con los filtros seleccionados.
       </p>
     </template>
   </TaskCollectionProvider>
   ```

7. En una expresión asignable de plantilla, Vue desempaqueta automáticamente los `ref` expuestos por el slot. Por ello, `v-model="collection.statusFilter"` actualiza correctamente el `ref` recibido.

**Resultado esperado:**

La vista conserva la responsabilidad de presentación, mientras que `TaskCollectionProvider` concentra el contrato de datos y acciones de la colección.

**Verificación:**

Comprueba que el proveedor no renderiza una estructura visual propia:

```bash
grep -nE "<ul|<li|<button|<section|<div" src/components/tasks/TaskCollectionProvider.vue
```

El comando no debe producir resultados.

---

### Paso 9. Retirar el módulo de estado anterior y comprobar la integración

**Objetivo:** Finalizar la migración sin mantener dos fuentes de verdad para el mismo estado.

**Instrucciones:**

1. Ejecuta una búsqueda global de importaciones del módulo antiguo:

   ```bash
   grep -R "projectHubState" -n src --exclude=projectHubState.js
   ```

2. Si aparecen resultados, reemplaza cada importación por el consumo del contexto:

   ```js
   import { useProjectHubContext } from '@/keys/projectHub'

   const { tasks, filters, selection } = useProjectHubContext()
   ```

3. Elimina el archivo de estado centralizado cuando no tenga consumidores:

   ```bash
   rm src/state/projectHubState.js
   ```

   En Windows PowerShell:

   ```powershell
   Remove-Item src/state/projectHubState.js
   ```

4. Si el directorio `src/state` queda vacío, puedes eliminarlo:

   ```bash
   rmdir src/state
   ```

5. Ejecuta el linter y corrige errores de imports no utilizados, nombres incorrectos o problemas de formato:

   ```bash
   npm run lint
   ```

6. Genera la compilación de producción:

   ```bash
   npm run build
   ```

**Resultado esperado:**

- No existe `src/state/projectHubState.js`.
- No hay imports activos hacia el módulo eliminado.
- La aplicación compila correctamente.
- El estado se distribuye desde `ProjectHubView.vue` mediante contexto y composables.

**Verificación:**

Ejecuta:

```bash
test ! -f src/state/projectHubState.js && echo "Estado centralizado eliminado"
npm run build
```

La compilación debe terminar sin errores.

---

## Validación y pruebas

### Validación manual de la aplicación

Inicia el servidor Vite con el puerto y host obligatorios:

```bash
npm run dev -- --host 127.0.0.1 --port 5173
```

Abre:

```text
http://127.0.0.1:5173
```

Realiza las siguientes comprobaciones:

| Prueba | Acción | Resultado esperado |
|---|---|---|
| Carga inicial | Abre la vista ProjectHub | Se muestran las tareas interceptadas por MSW o por el servicio simulado existente. |
| Estado de carga | Recarga la página con DevTools abierto | Se muestra temporalmente “Cargando tareas…”. |
| Búsqueda | Escribe una palabra incluida en un título | La lista se reduce sin recargar la página. |
| Filtro por estado | Selecciona “Completadas” | Solo aparecen tareas con `status: "completed"`. |
| Ordenamiento | Selecciona “Prioridad” | Las tareas se reordenan de mayor a menor prioridad. |
| Restablecimiento | Pulsa “Limpiar filtros” | Se restablecen búsqueda, estado y ordenamiento. |
| Selección | Pulsa una tarea | La tarea obtiene la clase visual de selección y la URL contiene `?task=<id>`. |
| Preferencias | Pulsa “Cambiar densidad” | La clase `project-hub--compact` cambia y la preferencia se guarda en Local Storage. |
| Persistencia | Recarga la página después de cambiar densidad | La preferencia visual se conserva. |
| Error de carga | Simula temporalmente un error en MSW | Se muestra un mensaje con `role="alert"`. |

### Validación estática

Ejecuta:

```bash
npm run lint
npm run build
```

Ambos comandos deben finalizar correctamente.

### Validación de arquitectura

Ejecuta las siguientes comprobaciones:

```bash
grep -R "projectHubState" -n src || true
grep -R "provide(projectHubKey" -n src/views
grep -R "useProjectHubContext" -n src/components
```

Criterios esperados:

- No existen referencias al módulo de estado eliminado.
- `ProjectHubView.vue` provee el contexto mediante `provide(projectHubKey, ...)`.
- `TaskCollectionProvider.vue` consume el contexto mediante `useProjectHubContext()`.

### Checklist de entrega

- [ ] Existen los cuatro composables solicitados en `src/composables`.
- [ ] `useProjectFilters()` usa `computed()` para tareas filtradas y contadores.
- [ ] `useTaskSelection()` no importa Vue Router directamente.
- [ ] `useLocalPreferences()` persiste preferencias y usa `onScopeDispose()`.
- [ ] `useAsyncState()` cancela operaciones pendientes con `AbortController`.
- [ ] Existe `src/keys/projectHub.js` con símbolos de inyección.
- [ ] `ProjectHubView.vue` provee contexto de proyecto y sesión simulada.
- [ ] `TaskCollectionProvider.vue` es renderless y usa un slot con alcance.
- [ ] Se eliminó `src/state/projectHubState.js`.
- [ ] `npm run lint` y `npm run build` finalizan correctamente.

## Resolución de problemas

### Problema 1. Error: `useProjectHubContext debe utilizarse dentro de ProjectHubView`

**Síntoma:** La consola del navegador muestra el error personalizado al cargar `TaskCollectionProvider` o un componente hijo.

**Causa:** El componente que llama a `useProjectHubContext()` no está debajo de la instancia de `ProjectHubView.vue` que ejecuta `provide(projectHubKey, projectHubContext)`. También puede ocurrir si se usa accidentalmente otra clave `Symbol()` creada en un archivo diferente.

**Solución:**

1. Confirma que `TaskCollectionProvider` está anidado dentro de `ProjectHubView.vue`.
2. Confirma que proveedor y consumidor importan exactamente la misma clave:

   ```js
   import { projectHubKey, useProjectHubContext } from '@/keys/projectHub'
   ```

3. No declares un nuevo `Symbol('projectHub')` en el componente consumidor.
4. Verifica que `provide(projectHubKey, projectHubContext)` se ejecuta de forma síncrona dentro de `<script setup>` y no después de un `await`.

### Problema 2. La preferencia cambia, pero desaparece al recargar la página

**Síntoma:** El modo compacto o el tema cambia durante la sesión actual, pero vuelve al valor inicial después de recargar.

**Causa:** El navegador puede tener Local Storage deshabilitado, la aplicación puede estar usando una clave diferente a `projecthub:preferences`, o se está mutando una copia no reactiva de las preferencias.

**Solución:**

1. Abre Chrome DevTools > **Application** > **Local Storage** y verifica si existe la clave `projecthub:preferences`.
2. Comprueba que la llamada usa el mismo nombre de clave:

   ```js
   const preferences = useLocalPreferences('projecthub:preferences')
   ```

3. Actualiza las preferencias mediante las acciones del composable:

   ```js
   preferences.toggleCompactMode()
   preferences.setTheme('dark')
   ```

4. No desestructures `preferences.value` en variables normales, ya que perderían el vínculo reactivo.
5. Si el navegador bloquea el almacenamiento, usa una ventana normal en lugar de una sesión con políticas restrictivas o revisa la configuración de privacidad.

## Limpieza

1. Detén el servidor de desarrollo con:

   ```text
   Ctrl + C
   ```

2. Comprueba los cambios realizados:

   ```bash
   git status
   ```

3. Añade los archivos modificados al área de preparación:

   ```bash
   git add src
   ```

4. Crea un commit en la rama `main`:

   ```bash
   git commit -m "refactor project hub to composition composables"
   ```

5. Confirma el historial reciente:

   ```bash
   git log --oneline -3
   ```

No elimines `node_modules`, la configuración de MSW ni los artefactos necesarios del proyecto. El siguiente laboratorio utilizará esta arquitectura para incorporar rutas, autenticación simulada y guards.

## Resumen

En este laboratorio refactorizaste ProjectHub desde un módulo de estado concentrado hacia una arquitectura de composición reutilizable. `useProjectFilters()` concentra el estado derivado de búsqueda y ordenamiento; `useTaskSelection()` gestiona selección con navegación desacoplada; `useLocalPreferences()` controla persistencia visual; y `useAsyncState()` estandariza los ciclos de carga y error.

Además, `ProjectHubView.vue` actúa ahora como proveedor de contexto, mientras `TaskCollectionProvider.vue` aplica un patrón renderless para separar lógica de colección y presentación. Esta estructura reduce prop drilling, evita duplicar estado derivado y deja preparada la aplicación para introducir autenticación, navegación protegida y guards en el laboratorio `03-00-01`.

### Recursos opcionales

- [Vue: Composition API](https://vuejs.org/guide/extras/composition-api-faq.html)
- [Vue: Provide / Inject](https://vuejs.org/guide/components/provide-inject.html)
- [Vue: Slots con alcance](https://vuejs.org/guide/components/slots.html#scoped-slots)
- [Vue: Reactividad y observadores](https://vuejs.org/guide/essentials/watchers.html)
