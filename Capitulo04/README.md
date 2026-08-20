# 4 Practica: diseñar y aplicar una arquitectura de stores para una aplicación compleja

## Metadatos

| Propiedad | Valor |
|---|---|
| Duración | 46 minutos |
| Complejidad | Difícil |
| Nivel de Bloom | Crear |

## Descripción general

En este laboratorio evolucionará la SPA `projecthub-spa`, creada en laboratorios anteriores, hacia una arquitectura modular de estado global basada en Pinia. Separará los dominios de autenticación, proyectos, tareas, interfaz y estado de operaciones, incorporando persistencia selectiva y trazabilidad durante el desarrollo.

La solución resultante evitará fuentes de verdad duplicadas, mantendrá los tokens de demostración fuera de `localStorage` y preparará la aplicación para la integración posterior con clientes REST, GraphQL, caché y manejo centralizado de errores.

## Objetivos de aprendizaje

Al completar el laboratorio, podrá:

- [ ] Diseñar stores Pinia independientes para autenticación, proyectos, tareas, interfaz y estado de API.
- [ ] Implementar stores de configuración (*setup stores*) con estado, selectores derivados y acciones asíncronas.
- [ ] Persistir de forma selectiva tema, filtros y sesión mínima sin almacenar tokens ni estados transitorios.
- [ ] Crear plugins de Pinia para persistencia y auditoría de acciones y mutaciones.
- [ ] Refactorizar guards de Vue Router y composables para que consulten los stores globales.

## Prerrequisitos

### Conocimientos requeridos

- Haber completado el laboratorio `03-00-01`, incluyendo autenticación simulada, rutas protegidas y carga perezosa.
- Comprender `ref`, `computed`, `watch`, Composition API y `script setup`.
- Diferenciar estado local de componente, estado compartido mediante composables y estado global.
- Conocer el uso básico de `defineStore`, `storeToRefs`, getters y acciones de Pinia.

### Acceso y software requerido

| Recurso | Requisito |
|---|---|
| Directorio de trabajo | `/workspace/vue-intermedio-labs` |
| Equivalente en Windows | `C:\workspace\vue-intermedio-labs` |
| Aplicación | `projecthub-spa` |
| Node.js | 22.14.0 |
| npm | 10.9.2 |
| Vue | 3.5.13 |
| Pinia | 3.0.1 |
| Vue Router | 4.5.0 |
| Navegador | Google Chrome instalado |
| Puerto Vite | `5173` |
| Puerto reservado | `3000`, no usar durante este laboratorio |

> El endpoint REST simulado debe mantenerse en `http://127.0.0.1:5173/api`. MSW intercepta las solicitudes en el navegador; no inicie una API adicional.

## Entorno del laboratorio

1. Abra una terminal y acceda al directorio obligatorio:

   ```bash
   cd /workspace/vue-intermedio-labs/projecthub-spa
   ```

   En Windows PowerShell:

   ```powershell
   cd C:\workspace\vue-intermedio-labs\projecthub-spa
   ```

2. Confirme las versiones principales:

   ```bash
   node --version
   npm --version
   npm list vue pinia vue-router
   ```

3. Si Pinia no está instalado, instale exactamente la versión requerida:

   ```bash
   npm install pinia@3.0.1
   ```

4. Cree los directorios para stores y plugins:

   ```bash
   mkdir -p src/stores src/plugins
   ```

5. Inicie la aplicación con el puerto y host obligatorios:

   ```bash
   npm run dev -- --host 127.0.0.1 --port 5173
   ```

6. Abra la aplicación en Chrome:

   ```text
   http://127.0.0.1:5173
   ```

## Procedimiento paso a paso

### Paso 1. Inventariar el estado y definir límites de dominio

**Objetivo:** identificar qué estado debe migrarse a Pinia y evitar crear una store global monolítica.

**Instrucciones:**

1. Revise los composables, vistas y guards existentes del laboratorio anterior.
2. Clasifique el estado según la siguiente tabla:

   | Dominio | Store | Responsabilidad |
   |---|---|---|
   | Sesión y autorización | `useAuthStore` | Usuario, roles, sesión mínima, renovación y cierre de sesión. |
   | Proyectos | `useProjectsStore` | Entidades de proyectos, carga y selector de proyecto actual. |
   | Tareas | `useTasksStore` | Tareas normalizadas por ID y relación con proyectos. |
   | Interfaz | `useUiStore` | Tema, filtros, notificaciones y estado visual global. |
   | Operaciones HTTP | `useApiStatusStore` | Operaciones activas y errores centralizados. |

3. Mantenga como estado local de componente elementos tales como:
   - El contenido temporal de un campo de formulario.
   - La apertura de un menú usado por una sola vista.
   - La validación local de un formulario antes de enviarlo.

4. No conserve en composables una copia reactiva del usuario, proyectos o filtros globales. Los composables restantes deben actuar como adaptadores de presentación y consultar stores.

**Resultado esperado:**

Existe una decisión clara sobre la fuente de verdad de cada tipo de dato. No se crea una única store llamada `app` o `global`.

**Verificación:**

Compruebe que puede responder estas preguntas:

- ¿Dónde vive el tema? En `useUiStore`.
- ¿Dónde vive el usuario autenticado? En `useAuthStore`.
- ¿Dónde se registra un error de carga? En `useApiStatusStore`.
- ¿Dónde se resuelve una tarea por identificador? En `useTasksStore`.

---

### Paso 2. Registrar Pinia y los plugins globales

**Objetivo:** crear una instancia de Pinia y registrar los plugins antes de montar la aplicación.

**Instrucciones:**

1. Cree el archivo `src/plugins/piniaPersistence.js`:

   ```js
   function selectPaths(source, paths) {
     return paths.reduce((result, path) => {
       if (Object.prototype.hasOwnProperty.call(source, path)) {
         result[path] = source[path]
       }

       return result
     }, {})
   }

   export function piniaPersistence({ store, options }) {
     const persist = options.persist

     if (!persist) {
       return
     }

     const config = persist === true ? {} : persist
     const key = config.key ?? `projecthub:${store.$id}`
     const paths = config.paths ?? Object.keys(store.$state)

     try {
       const savedState = localStorage.getItem(key)

       if (savedState) {
         store.$patch(JSON.parse(savedState))
       }
     } catch (error) {
       console.warn(`[persistencia] No se pudo restaurar ${store.$id}`, error)
       localStorage.removeItem(key)
     }

     store.$subscribe(
       (_mutation, state) => {
         try {
           const stateToPersist = selectPaths(state, paths)
           localStorage.setItem(key, JSON.stringify(stateToPersist))
         } catch (error) {
           console.warn(`[persistencia] No se pudo guardar ${store.$id}`, error)
         }
       },
       { detached: true },
     )
   }
   ```

2. Cree el archivo `src/plugins/piniaAudit.js`:

   ```js
   export function piniaAudit({ store }) {
     store.$onAction(({ name, args, after, onError }) => {
       console.info(`[pinia:action] ${store.$id}.${name}`, { args })

       after((result) => {
         console.info(`[pinia:success] ${store.$id}.${name}`, { result })
       })

       onError((error) => {
         console.error(`[pinia:error] ${store.$id}.${name}`, error)
       })
     })

     store.$subscribe((mutation) => {
       console.info(`[pinia:mutation] ${store.$id}`, {
         type: mutation.type,
         payload: mutation.payload,
       })
     })
   }
   ```

3. Actualice `src/main.js` o `src/main.ts`:

   ```js
   import { createApp } from 'vue'
   import { createPinia } from 'pinia'
   import App from './App.vue'
   import router from './router'
   import { piniaPersistence } from './plugins/piniaPersistence'
   import { piniaAudit } from './plugins/piniaAudit'

   const app = createApp(App)
   const pinia = createPinia()

   pinia.use(piniaPersistence)

   if (import.meta.env.DEV) {
     pinia.use(piniaAudit)
   }

   app.use(pinia)
   app.use(router)
   app.mount('#app')
   ```

4. Verifique que MSW continúa inicializándose exclusivamente en desarrollo, según la configuración heredada:

   ```js
   if (import.meta.env.DEV) {
     const { worker } = await import('./mocks/browser')
     await worker.start()
   }
   ```

**Resultado esperado:**

Pinia queda disponible para componentes, guards y stores. El plugin de auditoría solo escribe mensajes en desarrollo.

**Verificación:**

Abra DevTools de Chrome, pestaña **Console**. No debe observar errores relacionados con `getActivePinia()` al navegar por la aplicación.

---

### Paso 3. Crear la store de estado de API

**Objetivo:** centralizar operaciones en curso y errores para que las stores de dominio no dupliquen lógica transversal.

**Instrucciones:**

1. Cree `src/stores/apiStatus.js`:

   ```js
   import { computed, ref } from 'vue'
   import { defineStore } from 'pinia'

   export const useApiStatusStore = defineStore('apiStatus', () => {
     const pendingOperations = ref([])
     const errors = ref([])

     const isLoading = computed(() => pendingOperations.value.length > 0)

     function start(operation) {
       if (!pendingOperations.value.includes(operation)) {
         pendingOperations.value.push(operation)
       }
     }

     function finish(operation) {
       pendingOperations.value = pendingOperations.value.filter(
         (item) => item !== operation,
       )
     }

     function reportError({ operation, message, cause = null }) {
       errors.value.unshift({
         id: crypto.randomUUID(),
         operation,
         message,
         cause,
         createdAt: new Date().toISOString(),
       })
     }

     function clearError(errorId) {
       errors.value = errors.value.filter((error) => error.id !== errorId)
     }

     function clearErrors() {
       errors.value = []
     }

     return {
       pendingOperations,
       errors,
       isLoading,
       start,
       finish,
       reportError,
       clearError,
       clearErrors,
     }
   })
   ```

2. No configure persistencia para esta store. Las operaciones y errores son transitorios.

3. Mantenga los nombres de operación consistentes, por ejemplo:
   - `auth.signIn`
   - `auth.refreshSession`
   - `projects.loadAll`
   - `tasks.loadByProject`

**Resultado esperado:**

La aplicación dispone de una fuente única para indicar carga global y registrar fallos de operaciones.

**Verificación:**

En Vue DevTools, tras ejecutar una acción asíncrona, confirme que `pendingOperations` vuelve a ser un arreglo vacío cuando finaliza la operación.

---

### Paso 4. Implementar la store de interfaz con persistencia selectiva

**Objetivo:** administrar preferencias y notificaciones globales sin persistir datos temporales.

**Instrucciones:**

1. Cree `src/stores/ui.js`:

   ```js
   import { computed, ref } from 'vue'
   import { defineStore } from 'pinia'

   export const useUiStore = defineStore(
     'ui',
     () => {
       const theme = ref('light')
       const projectFilters = ref({
         query: '',
         status: 'all',
         ownerId: 'all',
       })
       const notifications = ref([])

       const hasNotifications = computed(
         () => notifications.value.length > 0,
       )

       function setTheme(nextTheme) {
         theme.value = nextTheme === 'dark' ? 'dark' : 'light'
         document.documentElement.dataset.theme = theme.value
       }

       function setProjectFilters(filters) {
         projectFilters.value = {
           ...projectFilters.value,
           ...filters,
         }
       }

       function resetProjectFilters() {
         projectFilters.value = {
           query: '',
           status: 'all',
           ownerId: 'all',
         }
       }

       function notify({ message, type = 'info' }) {
         notifications.value.unshift({
           id: crypto.randomUUID(),
           message,
           type,
           createdAt: new Date().toISOString(),
         })
       }

       function dismissNotification(notificationId) {
         notifications.value = notifications.value.filter(
           (notification) => notification.id !== notificationId,
         )
       }

       return {
         theme,
         projectFilters,
         notifications,
         hasNotifications,
         setTheme,
         setProjectFilters,
         resetProjectFilters,
         notify,
         dismissNotification,
       }
     },
     {
       persist: {
         key: 'projecthub:ui',
         paths: ['theme', 'projectFilters'],
       },
     },
   )
   ```

2. En `App.vue`, aplique el tema restaurado cuando se cargue la aplicación:

   ```vue
   <script setup>
   import { onMounted } from 'vue'
   import { useUiStore } from '@/stores/ui'

   const uiStore = useUiStore()

   onMounted(() => {
     document.documentElement.dataset.theme = uiStore.theme
   })
   </script>

   <template>
     <RouterView />
   </template>
   ```

3. No incluya `notifications` en `paths`. Las notificaciones deben desaparecer al recargar la página.

**Resultado esperado:**

El tema y filtros sobreviven a una recarga; las notificaciones no.

**Verificación:**

En Chrome DevTools, ejecute:

```js
JSON.parse(localStorage.getItem('projecthub:ui'))
```

La salida debe incluir `theme` y `projectFilters`, pero no `notifications`.

---

### Paso 5. Implementar la store de autenticación

**Objetivo:** centralizar sesión, roles, renovación simulada y efectos secundarios de autenticación.

**Instrucciones:**

1. Cree `src/stores/auth.js`:

   ```js
   import { computed, ref } from 'vue'
   import { defineStore } from 'pinia'
   import { useApiStatusStore } from './apiStatus'
   import { useUiStore } from './ui'

   export const useAuthStore = defineStore(
     'auth',
     () => {
       const user = ref(null)
       const roles = ref([])
       const sessionExpiresAt = ref(null)

       const isAuthenticated = computed(() => Boolean(user.value))
       const isAdmin = computed(() => roles.value.includes('admin'))

       function setSession(session) {
         user.value = {
           id: session.user.id,
           name: session.user.name,
           email: session.user.email,
         }
         roles.value = session.roles ?? []
         sessionExpiresAt.value = session.expiresAt ?? null
       }

       async function signIn(credentials) {
         const apiStatusStore = useApiStatusStore()
         const uiStore = useUiStore()

         apiStatusStore.start('auth.signIn')

         try {
           const response = await fetch('/api/session', {
             method: 'POST',
             headers: {
               'Content-Type': 'application/json',
             },
             body: JSON.stringify(credentials),
           })

           if (!response.ok) {
             throw new Error('Las credenciales no son válidas.')
           }

           const session = await response.json()
           setSession(session)

           uiStore.notify({
             type: 'success',
             message: `Bienvenido, ${user.value.name}.`,
           })
         } catch (error) {
           apiStatusStore.reportError({
             operation: 'auth.signIn',
             message: error.message,
             cause: error,
           })
           throw error
         } finally {
           apiStatusStore.finish('auth.signIn')
         }
       }

       async function refreshSession() {
         if (!isAuthenticated.value) {
           return
         }

         const apiStatusStore = useApiStatusStore()
         apiStatusStore.start('auth.refreshSession')

         try {
           sessionExpiresAt.value = new Date(
             Date.now() + 30 * 60 * 1000,
           ).toISOString()
         } finally {
           apiStatusStore.finish('auth.refreshSession')
         }
       }

       function signOut() {
         user.value = null
         roles.value = []
         sessionExpiresAt.value = null

         useUiStore().notify({
           type: 'info',
           message: 'La sesión se cerró correctamente.',
         })
       }

       function hasRole(role) {
         return roles.value.includes(role)
       }

       return {
         user,
         roles,
         sessionExpiresAt,
         isAuthenticated,
         isAdmin,
         setSession,
         signIn,
         refreshSession,
         signOut,
         hasRole,
       }
     },
     {
       persist: {
         key: 'projecthub:auth',
         paths: ['user', 'roles', 'sessionExpiresAt'],
       },
     },
   )
   ```

2. Confirme que la respuesta simulada de MSW para `POST /api/session` tiene una estructura compatible:

   ```json
   {
     "user": {
       "id": "u-001",
       "name": "Ada Lovelace",
       "email": "ada@example.test"
     },
     "roles": ["member"],
     "expiresAt": "2026-08-20T12:00:00.000Z"
   }
   ```

3. No persista tokens de demostración, cabeceras `Authorization`, estados `loading` ni mensajes de error.

**Resultado esperado:**

La sesión mínima se conserva tras recargar, pero el almacenamiento local no contiene secretos de autenticación.

**Verificación:**

Ejecute en la consola del navegador:

```js
JSON.parse(localStorage.getItem('projecthub:auth'))
```

Confirme que solo aparecen `user`, `roles` y `sessionExpiresAt`.

---

### Paso 6. Crear stores normalizadas de proyectos y tareas

**Objetivo:** gestionar entidades de dominio de forma escalable, evitando copias de tareas dentro de cada proyecto.

**Instrucciones:**

1. Cree `src/stores/projects.js`:

   ```js
   import { computed, ref } from 'vue'
   import { defineStore } from 'pinia'
   import { useApiStatusStore } from './apiStatus'
   import { useUiStore } from './ui'

   export const useProjectsStore = defineStore('projects', () => {
     const byId = ref({})
     const allIds = ref([])
     const selectedProjectId = ref(null)

     const projects = computed(() =>
       allIds.value.map((id) => byId.value[id]).filter(Boolean),
     )

     const selectedProject = computed(
       () => byId.value[selectedProjectId.value] ?? null,
     )

     function upsertMany(items) {
       items.forEach((project) => {
         byId.value[project.id] = project

         if (!allIds.value.includes(project.id)) {
           allIds.value.push(project.id)
         }
       })
     }

     function selectProject(projectId) {
       selectedProjectId.value = projectId
     }

     async function loadAll() {
       const apiStatusStore = useApiStatusStore()
       const uiStore = useUiStore()

       apiStatusStore.start('projects.loadAll')

       try {
         const response = await fetch('/api/projects')

         if (!response.ok) {
           throw new Error('No se pudieron cargar los proyectos.')
         }

         upsertMany(await response.json())
       } catch (error) {
         apiStatusStore.reportError({
           operation: 'projects.loadAll',
           message: error.message,
           cause: error,
         })

         uiStore.notify({
           type: 'error',
           message: 'No fue posible cargar los proyectos.',
         })
       } finally {
         apiStatusStore.finish('projects.loadAll')
       }
     }

     return {
       byId,
       allIds,
       selectedProjectId,
       projects,
       selectedProject,
       upsertMany,
       selectProject,
       loadAll,
     }
   })
   ```

2. Cree `src/stores/tasks.js`:

   ```js
   import { computed, ref } from 'vue'
   import { defineStore } from 'pinia'
   import { useApiStatusStore } from './apiStatus'

   export const useTasksStore = defineStore('tasks', () => {
     const byId = ref({})
     const idsByProjectId = ref({})

     const totalTasks = computed(() => Object.keys(byId.value).length)

     function upsertMany(tasks) {
       tasks.forEach((task) => {
         byId.value[task.id] = task

         const projectTaskIds = idsByProjectId.value[task.projectId] ?? []

         if (!projectTaskIds.includes(task.id)) {
           idsByProjectId.value[task.projectId] = [
             ...projectTaskIds,
             task.id,
           ]
         }
       })
     }

     function tasksForProject(projectId) {
       return (idsByProjectId.value[projectId] ?? [])
         .map((taskId) => byId.value[taskId])
         .filter(Boolean)
     }

     async function loadByProject(projectId) {
       const apiStatusStore = useApiStatusStore()
       apiStatusStore.start('tasks.loadByProject')

       try {
         const response = await fetch(`/api/projects/${projectId}/tasks`)

         if (!response.ok) {
           throw new Error('No se pudieron cargar las tareas.')
         }

         upsertMany(await response.json())
       } catch (error) {
         apiStatusStore.reportError({
           operation: 'tasks.loadByProject',
           message: error.message,
           cause: error,
         })
       } finally {
         apiStatusStore.finish('tasks.loadByProject')
       }
     }

     return {
       byId,
       idsByProjectId,
       totalTasks,
       upsertMany,
       tasksForProject,
       loadByProject,
     }
   })
   ```

3. Mantenga proyectos y tareas sin persistencia. Los datos de dominio deben recuperarse o invalidarse de forma controlada en sesiones futuras.

**Resultado esperado:**

Los proyectos se almacenan por ID y las tareas se relacionan con proyectos mediante `idsByProjectId`.

**Verificación:**

Después de cargar un proyecto, inspeccione `tasksStore` en Vue DevTools. Debe observar una estructura similar a:

```js
{
  byId: {
    "task-1": { id: "task-1", projectId: "project-1", title: "..." }
  },
  idsByProjectId: {
    "project-1": ["task-1"]
  }
}
```

---

### Paso 7. Refactorizar vistas y composables consumidores

**Objetivo:** consumir estado reactivo de Pinia sin perder reactividad al desestructurar.

**Instrucciones:**

1. Actualice una vista de proyectos, por ejemplo `src/views/ProjectsView.vue`:

   ```vue
   <script setup>
   import { computed, onMounted } from 'vue'
   import { storeToRefs } from 'pinia'
   import { useProjectsStore } from '@/stores/projects'
   import { useUiStore } from '@/stores/ui'

   const projectsStore = useProjectsStore()
   const uiStore = useUiStore()

   const { projects } = storeToRefs(projectsStore)
   const { projectFilters } = storeToRefs(uiStore)

   const filteredProjects = computed(() => {
     const query = projectFilters.value.query.trim().toLowerCase()

     return projects.value.filter((project) => {
       const matchesQuery = project.name.toLowerCase().includes(query)
       const matchesStatus =
         projectFilters.value.status === 'all' ||
         project.status === projectFilters.value.status

       return matchesQuery && matchesStatus
     })
   })

   onMounted(() => {
     projectsStore.loadAll()
   })
   </script>

   <template>
     <section>
       <h1>Proyectos</h1>

       <label>
         Buscar proyecto
         <input
           :value="projectFilters.query"
           type="search"
           @input="uiStore.setProjectFilters({ query: $event.target.value })"
         >
       </label>

       <p v-if="filteredProjects.length === 0">
         No hay proyectos que coincidan con el filtro.
       </p>

       <ul v-else>
         <li v-for="project in filteredProjects" :key="project.id">
           {{ project.name }} — {{ project.status }}
         </li>
       </ul>
     </section>
   </template>
   ```

2. Refactorice los composables antiguos para que no creen estado global propio. Por ejemplo, cree o actualice `src/composables/useProjectFilters.js`:

   ```js
   import { storeToRefs } from 'pinia'
   import { useUiStore } from '@/stores/ui'

   export function useProjectFilters() {
     const uiStore = useUiStore()
     const { projectFilters } = storeToRefs(uiStore)

     return {
       projectFilters,
       setProjectFilters: uiStore.setProjectFilters,
       resetProjectFilters: uiStore.resetProjectFilters,
     }
   }
   ```

3. No use este patrón:

   ```js
   const { projects } = useProjectsStore()
   ```

4. Use `storeToRefs` para estado y getters, y extraiga acciones directamente:

   ```js
   const projectsStore = useProjectsStore()
   const { projects, selectedProject } = storeToRefs(projectsStore)
   const { loadAll, selectProject } = projectsStore
   ```

**Resultado esperado:**

Los componentes se actualizan cuando cambia el estado global y los composables se convierten en adaptadores pequeños de presentación.

**Verificación:**

Escriba un texto en el filtro de proyectos, recargue la página y confirme que el filtro se restaura desde `localStorage`.

---

### Paso 8. Migrar los guards de Vue Router

**Objetivo:** hacer que la protección de rutas consulte `useAuthStore` en lugar de composables o estado local.

**Instrucciones:**

1. Abra `src/router/index.js`.
2. Compruebe que las rutas protegidas mantienen metadatos explícitos:

   ```js
   {
     path: '/projects',
     name: 'projects',
     component: () => import('@/views/ProjectsView.vue'),
     meta: {
       requiresAuth: true,
     },
   }
   ```

3. Importe la store de autenticación y actualice el guard:

   ```js
   import { createRouter, createWebHistory } from 'vue-router'
   import { useAuthStore } from '@/stores/auth'

   const router = createRouter({
     history: createWebHistory(),
     routes: [
       // Rutas existentes.
     ],
   })

   router.beforeEach((to) => {
     const authStore = useAuthStore()

     if (to.meta.requiresAuth && !authStore.isAuthenticated) {
       return {
         name: 'login',
         query: {
           redirect: to.fullPath,
         },
       }
     }

     if (to.name === 'login' && authStore.isAuthenticated) {
       return {
         name: 'projects',
       }
     }

     return true
   })

   export default router
   ```

4. En la vista de inicio de sesión, conserve el redireccionamiento original:

   ```js
   import { useRoute, useRouter } from 'vue-router'
   import { useAuthStore } from '@/stores/auth'

   const route = useRoute()
   const router = useRouter()
   const authStore = useAuthStore()

   async function submit(credentials) {
     await authStore.signIn(credentials)

     await router.push(
       typeof route.query.redirect === 'string'
         ? route.query.redirect
         : { name: 'projects' },
     )
   }
   ```

**Resultado esperado:**

Una ruta protegida redirige al login cuando no hay sesión. Tras iniciar sesión, la ruta solicitada se recupera correctamente.

**Verificación:**

1. Abra una ventana de incógnito.
2. Visite directamente:

   ```text
   http://127.0.0.1:5173/projects
   ```

3. Confirme la redirección al login.
4. Inicie sesión con las credenciales admitidas por MSW.
5. Confirme que vuelve a `/projects`.

## Validación y pruebas

Ejecute las siguientes comprobaciones antes de considerar terminado el laboratorio.

1. Compruebe la calidad estática y el empaquetado:

   ```bash
   npm run lint
   npm run build
   ```

2. Inicie nuevamente el servidor:

   ```bash
   npm run dev -- --host 127.0.0.1 --port 5173
   ```

3. Valide manualmente la matriz de comportamiento:

   | Caso | Resultado esperado |
   |---|---|
   | Usuario anónimo abre `/projects` | Router redirige a login. |
   | Inicio de sesión válido | `authStore.isAuthenticated` pasa a `true`. |
   | Recarga tras iniciar sesión | Se restaura usuario, roles y expiración mínima. |
   | Revisión de `localStorage` | No existen tokens, errores ni operaciones pendientes. |
   | Cambio de tema | El tema se mantiene tras recargar. |
   | Aplicación de filtro | El filtro se mantiene tras recargar. |
   | Carga de proyectos | Se registran acciones `projects.loadAll` en consola de desarrollo. |
   | Error simulado de API | Se registra en `apiStatusStore.errors` y aparece una notificación. |
   | Cierre de sesión | Se elimina la sesión y una ruta protegida vuelve a redirigir. |

4. Inspeccione la consola de desarrollo. Debe encontrar mensajes similares a:

   ```text
   [pinia:action] auth.signIn
   [pinia:success] auth.signIn
   [pinia:mutation] auth
   [pinia:action] projects.loadAll
   ```

5. Confirme que el plugin de auditoría no forma parte del comportamiento de producción:

   ```bash
   npm run build
   npm run preview -- --host 127.0.0.1 --port 4173
   ```

   Abra `http://127.0.0.1:4173` y verifique que la aplicación compila y navega correctamente. Detenga la previsualización al terminar.

## Solución de problemas

### Problema 1: `getActivePinia()` o “no active Pinia” al ejecutar un guard

**Síntoma:** al navegar a una ruta protegida aparece un error similar a `getActivePinia() was called but there was no active Pinia`.

**Causa:** el router o un guard intenta usar `useAuthStore()` antes de que la instancia de Pinia haya sido registrada en la aplicación.

**Solución:** confirme el orden en `src/main.js`:

```js
app.use(pinia)
app.use(router)
app.mount('#app')
```

También confirme que no está ejecutando `useAuthStore()` en el nivel superior de `src/router/index.js`. Debe invocarse dentro de `router.beforeEach()`.

### Problema 2: el tema o los filtros no se restauran, o aparece un error al leer `localStorage`

**Síntoma:** tras recargar la página, el tema vuelve al valor inicial; la consola muestra un error de `JSON.parse`; o la aplicación conserva datos antiguos incompatibles.

**Causa:** existe una entrada corrupta o con una estructura previa en `localStorage`.

**Solución:** elimine exclusivamente las claves de persistencia de este laboratorio y recargue:

```js
localStorage.removeItem('projecthub:ui')
localStorage.removeItem('projecthub:auth')
location.reload()
```

Después, vuelva a cambiar el tema, aplicar un filtro e iniciar sesión para que los plugins generen datos con la estructura actual.

## Limpieza

1. Detenga el servidor Vite con:

   ```bash
   Ctrl+C
   ```

2. Si inició el servidor de previsualización, deténgalo también con:

   ```bash
   Ctrl+C
   ```

3. No elimine el proyecto, las stores ni los plugins. Serán la base para el laboratorio `05-00-01`.

4. Si necesita restablecer el estado local para una nueva demostración, ejecute en la consola del navegador:

   ```js
   localStorage.removeItem('projecthub:ui')
   localStorage.removeItem('projecthub:auth')
   ```

## Resumen

En este laboratorio implementó una arquitectura Pinia modular para `projecthub-spa`. La autenticación, proyectos, tareas, interfaz y estado de API ahora tienen responsabilidades independientes y una fuente de verdad explícita.

También creó persistencia selectiva para preferencias y sesión mínima, excluyendo tokens y estados transitorios, y añadió auditoría de acciones y mutaciones en desarrollo. Finalmente, los guards de Vue Router y los composables de presentación fueron adaptados para depender de stores Pinia en lugar de mantener copias locales del estado global.
