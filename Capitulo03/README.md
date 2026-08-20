# 4 Practica: implementar flujo de autenticación con rutas protegidas y carga dinámica

## Metadatos

| Campo | Valor |
|---|---|
| Duración | 46 minutos |
| Complejidad | Difícil |
| Nivel de Bloom | Aplicar |

## Descripción general

En este laboratorio ampliarás la SPA **ProjectHub** con una navegación estructurada mediante Vue Router 4. Implementarás autenticación simulada con MSW, rutas anidadas protegidas, autorización por roles y carga perezosa de vistas mediante `import()` dinámico.

El resultado será una aplicación con acceso público a `/login`, un área protegida bajo `/app`, una vista administrativa restringida y una estrategia de recuperación ante errores de carga de chunks.

## Objetivos de aprendizaje

Al completar este laboratorio, podrás:

- [ ] Configurar rutas públicas, protegidas, anidadas y de captura global con Vue Router.
- [ ] Implementar autenticación simulada contra `POST /api/auth/login` usando MSW.
- [ ] Aplicar `meta.requiresAuth`, `meta.roles` y guards globales de navegación.
- [ ] Cargar bajo demanda las vistas Dashboard, Projects, TaskDetail y Admin.
- [ ] Gestionar rutas no encontradas, accesos prohibidos y errores `ChunkLoadError`.

## Prerrequisitos

### Conocimientos requeridos

- Laboratorio `02-00-01` completado, con composables y contexto funcional de ProjectHub.
- Uso básico de Vue 3 con Composition API y `<script setup>`.
- Promesas, `async/await`, `try/catch` y solicitudes HTTP con `fetch`.
- Fundamentos de Vue Router: `RouterLink`, `RouterView`, rutas nombradas y parámetros dinámicos.
- Conocimiento básico de MSW en el navegador.

### Acceso y condiciones técnicas

- Directorio de trabajo obligatorio:
  - Linux/macOS: `/workspace/vue-intermedio-labs`
  - Windows: `C:\workspace\vue-intermedio-labs`
- Aplicación existente con nombre fijo: `projecthub-spa`.
- Rama Git activa: `main`.
- Google Chrome instalado para verificar la SPA.
- Puerto Vite disponible: `5173`.
- No iniciar ningún servicio en el puerto `3000`.

## Entorno del laboratorio

### Software principal

| Herramienta | Versión esperada |
|---|---:|
| Node.js | 22.14.0 |
| npm | 10.9.2 |
| Vue | 3.5.13 |
| Vite | 6.1.0 |
| Vue Router | 4.5.0 |
| MSW | 2.7.3 |
| Google Chrome | 133.0.6943.98 o superior |

### Preparación inicial

1. Abre una terminal.
2. Accede al proyecto existente.
3. Comprueba la rama activa y las dependencias.

```bash
cd /workspace/vue-intermedio-labs/projecthub-spa
git branch --show-current
node --version
npm --version
npm install
```

En Windows PowerShell:

```powershell
cd C:\workspace\vue-intermedio-labs\projecthub-spa
git branch --show-current
node --version
npm --version
npm install
```

La rama mostrada debe ser `main`.

Si Vue Router o MSW no están disponibles en el proyecto, instálalos respetando las versiones del curso:

```bash
npm install vue-router@4.5.0 msw@2.7.3
```

Genera o actualiza el Service Worker de MSW en `public/`:

```bash
npx msw@2.7.3 init public/ --save
```

### Estructura objetivo

Al finalizar, los archivos relevantes deberán tener una estructura similar a esta:

```text
src/
├── App.vue
├── main.js
├── composables/
│   ├── useAsyncState.js
│   └── useAuthSession.js
├── mocks/
│   ├── browser.js
│   └── handlers.js
├── router/
│   └── index.js
└── views/
    ├── AdminView.vue
    ├── ChunkRecoveryView.vue
    ├── DashboardView.vue
    ├── ForbiddenView.vue
    ├── LoginView.vue
    ├── NotFoundView.vue
    ├── ProjectsView.vue
    ├── TaskDetailView.vue
    └── ProjectHubLayout.vue
```

## Procedimiento paso a paso

### Paso 1. Preparar el punto de entrada y el componente raíz

**Objetivo:** Instalar Vue Router en la aplicación y garantizar que MSW se inicialice exclusivamente durante el desarrollo.

**Instrucciones:**

1. Crea el directorio del router si todavía no existe:

   ```bash
   mkdir -p src/router src/mocks src/composables src/views
   ```

2. Actualiza `src/App.vue` para que actúe como contenedor raíz de Vue Router.

   ```vue
   <script setup>
   import { RouterView } from 'vue-router'
   </script>

   <template>
     <RouterView />
   </template>
   ```

3. Actualiza `src/main.js`. MSW se cargará dinámicamente solo si Vite se ejecuta en modo desarrollo.

   ```js
   import { createApp } from 'vue'
   import App from './App.vue'
   import router from './router'

   async function bootstrap() {
     if (import.meta.env.DEV) {
       const { worker } = await import('./mocks/browser')

       await worker.start({
         onUnhandledRequest: 'bypass'
       })
     }

     createApp(App)
       .use(router)
       .mount('#app')
   }

   bootstrap()
   ```

4. Verifica que exista el archivo `public/mockServiceWorker.js`.

   ```bash
   ls public/mockServiceWorker.js
   ```

   En Windows PowerShell:

   ```powershell
   Get-Item public\mockServiceWorker.js
   ```

**Resultado esperado:**

- `App.vue` contiene un único `<RouterView />`.
- `main.js` registra Vue Router con `.use(router)`.
- MSW solamente se inicia cuando `import.meta.env.DEV` es verdadero.
- El archivo `public/mockServiceWorker.js` existe.

**Verificación:**

Ejecuta temporalmente la aplicación:

```bash
npm run dev -- --host 127.0.0.1 --port 5173
```

Abre `http://127.0.0.1:5173`. Es normal observar un error de compilación temporal hasta crear `src/router/index.js` en el siguiente paso. Detén el servidor con `Ctrl+C`.

---

### Paso 2. Crear los composables de operación asíncrona y sesión

**Objetivo:** Centralizar el estado de carga, error y autenticación simulada mediante composables reutilizables.

**Instrucciones:**

1. Crea o actualiza `src/composables/useAsyncState.js`.

   ```js
   import { ref } from 'vue'

   export function useAsyncState() {
     const isLoading = ref(false)
     const error = ref(null)

     async function execute(operation) {
       isLoading.value = true
       error.value = null

       try {
         return await operation()
       } catch (currentError) {
         error.value = currentError
         throw currentError
       } finally {
         isLoading.value = false
       }
     }

     return {
       isLoading,
       error,
       execute
     }
   }
   ```

2. Crea `src/composables/useAuthSession.js`.

   Este composable mantiene un estado de sesión único a nivel de módulo durante la ejecución de la SPA. No se persiste todavía en `localStorage`, ya que la persistencia y la migración a Pinia se realizarán en el laboratorio `04-00-01`.

   ```js
   import { computed, readonly, ref } from 'vue'
   import { useAsyncState } from './useAsyncState'

   const API_URL = 'http://127.0.0.1:5173/api'

   const currentUser = ref(null)
   const { isLoading, error, execute } = useAsyncState()

   const isAuthenticated = computed(() => currentUser.value !== null)

   const roles = computed(() => currentUser.value?.roles ?? [])

   async function login(credentials) {
     const response = await execute(async () => {
       const request = await fetch(`${API_URL}/auth/login`, {
         method: 'POST',
         headers: {
           'Content-Type': 'application/json'
         },
         body: JSON.stringify(credentials)
       })

       const payload = await request.json()

       if (!request.ok) {
         throw new Error(payload.message ?? 'No fue posible iniciar sesión.')
       }

       return payload
     })

     currentUser.value = response.user

     return response.user
   }

   function logout() {
     currentUser.value = null
   }

   function hasAnyRole(requiredRoles = []) {
     if (requiredRoles.length === 0) {
       return true
     }

     return requiredRoles.some((role) => roles.value.includes(role))
   }

   export function useAuthSession() {
     return {
       currentUser: readonly(currentUser),
       isAuthenticated,
       roles,
       isLoading,
       error: readonly(error),
       login,
       logout,
       hasAnyRole
     }
   }
   ```

3. Revisa los aspectos importantes de la implementación:

   - `currentUser` está definido fuera de `useAuthSession()`, por lo que todas las vistas y guards comparten la misma sesión.
   - `useAsyncState()` controla `isLoading` y `error`.
   - La autenticación se comunica con el endpoint constante `http://127.0.0.1:5173/api/auth/login`.
   - El método `hasAnyRole()` permite evaluar los roles definidos en los metadatos de rutas.
   - No se usa Pinia en este laboratorio.

**Resultado esperado:**

Dispones de un composable genérico para tareas asíncronas y otro composable específico para sesión simulada.

**Verificación:**

Comprueba que los archivos existen:

```bash
find src/composables -maxdepth 1 -type f
```

Debes observar al menos:

```text
src/composables/useAsyncState.js
src/composables/useAuthSession.js
```

---

### Paso 3. Configurar el endpoint de autenticación con MSW

**Objetivo:** Simular el endpoint REST `POST /api/auth/login` sin iniciar una API real ni utilizar el puerto 3000.

**Instrucciones:**

1. Crea o actualiza `src/mocks/handlers.js`.

   ```js
   import { http, HttpResponse } from 'msw'

   const API_URL = 'http://127.0.0.1:5173/api'

   export const handlers = [
     http.post(`${API_URL}/auth/login`, async ({ request }) => {
       const { email, password } = await request.json()

       if (password !== 'projecthub123') {
         return HttpResponse.json(
           {
             message: 'Credenciales inválidas.'
           },
           {
             status: 401
           }
         )
       }

       if (email === 'admin@projecthub.dev') {
         return HttpResponse.json({
           user: {
             id: 'admin-01',
             name: 'Ada Administradora',
             email,
             roles: ['admin', 'user']
           }
         })
       }

       if (email === 'user@projecthub.dev') {
         return HttpResponse.json({
           user: {
             id: 'user-01',
             name: 'Ursula Usuario',
             email,
             roles: ['user']
           }
         })
       }

       return HttpResponse.json(
         {
           message: 'El usuario no está registrado en el entorno simulado.'
         },
         {
           status: 401
         }
       )
     })
   ]
   ```

2. Crea o actualiza `src/mocks/browser.js`.

   ```js
   import { setupWorker } from 'msw/browser'
   import { handlers } from './handlers'

   export const worker = setupWorker(...handlers)
   ```

3. Conserva las siguientes credenciales de prueba:

   | Tipo de usuario | Correo | Contraseña | Roles |
   |---|---|---|---|
   | Usuario estándar | `user@projecthub.dev` | `projecthub123` | `user` |
   | Administrador | `admin@projecthub.dev` | `projecthub123` | `admin`, `user` |
   | Error esperado | cualquier otro correo o contraseña | cualquiera no válida | Ninguno |

4. No cambies el endpoint para usar el puerto `3000`. MSW interceptará la solicitud en el navegador antes de que alcance una API real.

**Resultado esperado:**

- MSW intercepta `POST http://127.0.0.1:5173/api/auth/login`.
- Los dos usuarios definidos generan respuestas exitosas.
- Una contraseña incorrecta responde con estado HTTP `401`.

**Verificación:**

Inicia la aplicación:

```bash
npm run dev -- --host 127.0.0.1 --port 5173
```

Abre Chrome en `http://127.0.0.1:5173` y revisa la consola de desarrollo. Debe aparecer un mensaje equivalente a:

```text
[MSW] Mocking enabled.
```

Detén el servidor con `Ctrl+C` antes de continuar si deseas editar sin salida de consola adicional.

---

### Paso 4. Definir rutas, metadatos, guards y recuperación de chunks

**Objetivo:** Crear la configuración central de Vue Router con rutas protegidas, autorización por roles, lazy loading, telemetría y manejo de `ChunkLoadError`.

**Instrucciones:**

1. Crea `src/router/index.js`.

2. Agrega el siguiente contenido completo:

   ```js
   import { createRouter, createWebHistory } from 'vue-router'
   import { useAuthSession } from '@/composables/useAuthSession'

   import LoginView from '@/views/LoginView.vue'
   import ForbiddenView from '@/views/ForbiddenView.vue'
   import NotFoundView from '@/views/NotFoundView.vue'
   import ChunkRecoveryView from '@/views/ChunkRecoveryView.vue'
   import ProjectHubLayout from '@/views/ProjectHubLayout.vue'

   const routes = [
     {
       path: '/',
       redirect: {
         name: 'dashboard'
       }
     },
     {
       path: '/login',
       name: 'login',
       component: LoginView,
       meta: {
         public: true
       }
     },
     {
       path: '/forbidden',
       name: 'forbidden',
       component: ForbiddenView,
       meta: {
         public: true
       }
     },
     {
       path: '/chunk-recovery',
       name: 'chunk-recovery',
       component: ChunkRecoveryView,
       meta: {
         public: true
       }
     },
     {
       path: '/app',
       component: ProjectHubLayout,
       meta: {
         requiresAuth: true
       },
       children: [
         {
           path: '',
           redirect: {
             name: 'dashboard'
           }
         },
         {
           path: 'dashboard',
           name: 'dashboard',
           component: () => import('@/views/DashboardView.vue'),
           meta: {
             requiresAuth: true
           }
         },
         {
           path: 'projects',
           name: 'projects',
           component: () => import('@/views/ProjectsView.vue'),
           meta: {
             requiresAuth: true
           }
         },
         {
           path: 'tasks/:taskId',
           name: 'task-detail',
           component: () => import('@/views/TaskDetailView.vue'),
           props: true,
           meta: {
             requiresAuth: true
           }
         }
       ]
     },
     {
       path: '/admin',
       name: 'admin',
       component: () => import('@/views/AdminView.vue'),
       meta: {
         requiresAuth: true,
         roles: ['admin']
       }
     },
     {
       path: '/:pathMatch(.*)*',
       name: 'not-found',
       component: NotFoundView,
       meta: {
         public: true
       }
     }
   ]

   const router = createRouter({
     history: createWebHistory(),
     routes,
     linkActiveClass: 'is-active',
     linkExactActiveClass: 'is-exact-active',
     scrollBehavior(to, from, savedPosition) {
       if (savedPosition) {
         return savedPosition
       }

       if (to.hash) {
         return {
           el: to.hash,
           behavior: 'smooth'
         }
       }

       return {
         top: 0
       }
     }
   })

   router.beforeEach((to) => {
     const auth = useAuthSession()

     const requiresAuth = to.matched.some(
       (routeRecord) => routeRecord.meta.requiresAuth
     )

     const requiredRoles = to.matched.flatMap(
       (routeRecord) => routeRecord.meta.roles ?? []
     )

     if (requiresAuth && !auth.isAuthenticated.value) {
       return {
         name: 'login',
         query: {
           redirect: to.fullPath
         }
       }
     }

     if (
       requiresAuth &&
       requiredRoles.length > 0 &&
       !auth.hasAnyRole(requiredRoles)
     ) {
       return {
         name: 'forbidden',
         query: {
           redirect: to.fullPath
         }
       }
     }

     if (to.name === 'login' && auth.isAuthenticated.value) {
       return {
         name: 'dashboard'
       }
     }

     return true
   })

   router.afterEach((to, from, failure) => {
     const telemetry = {
       event: 'navigation',
       from: from.fullPath,
       to: to.fullPath,
       routeName: String(to.name ?? 'unnamed'),
       status: failure ? 'failed' : 'completed',
       timestamp: new Date().toISOString()
     }

     console.info('[ProjectHub telemetry]', telemetry)
   })

   router.onError((error, to) => {
     const isChunkLoadError =
       error.name === 'ChunkLoadError' ||
       /Loading chunk .* failed|Failed to fetch dynamically imported module/i.test(
         error.message
       )

     if (!isChunkLoadError || to.name === 'chunk-recovery') {
       return
     }

     console.error('[ProjectHub] Error de carga dinámica:', error)

     router.replace({
       name: 'chunk-recovery',
       query: {
         target: to.fullPath
       }
     })
   })

   export default router
   ```

3. Confirma que las rutas protegidas usan `meta.requiresAuth`.

4. Confirma que la ruta `/admin` usa:

   ```js
   meta: {
     requiresAuth: true,
     roles: ['admin']
   }
   ```

5. Observa que las vistas siguientes se cargan mediante importación dinámica:

   ```js
   () => import('@/views/DashboardView.vue')
   () => import('@/views/ProjectsView.vue')
   () => import('@/views/TaskDetailView.vue')
   () => import('@/views/AdminView.vue')
   ```

6. Observa que `/app` es una ruta padre. Por ello, `ProjectHubLayout.vue` deberá contener un `<RouterView />` para renderizar sus rutas hijas.

**Resultado esperado:**

La aplicación dispone de estas rutas:

| URL | Nombre | Acceso |
|---|---|---|
| `/login` | `login` | Público |
| `/forbidden` | `forbidden` | Público |
| `/chunk-recovery` | `chunk-recovery` | Público |
| `/app/dashboard` | `dashboard` | Usuario autenticado |
| `/app/projects` | `projects` | Usuario autenticado |
| `/app/tasks/:taskId` | `task-detail` | Usuario autenticado |
| `/admin` | `admin` | Solo rol `admin` |
| cualquier otra URL | `not-found` | Público |

**Verificación:**

Ejecuta el servidor:

```bash
npm run dev -- --host 127.0.0.1 --port 5173
```

Abre directamente:

```text
http://127.0.0.1:5173/app/dashboard
```

Debes ser redirigido a una URL equivalente a:

```text
http://127.0.0.1:5173/login?redirect=/app/dashboard
```

Aún no habrá una interfaz de login completa; se implementará en el siguiente paso.

---

### Paso 5. Implementar layout y vistas de navegación

**Objetivo:** Crear las vistas públicas, protegidas y de recuperación que consume el router.

**Instrucciones:**

1. Crea `src/views/ProjectHubLayout.vue`.

   ```vue
   <script setup>
   import { RouterLink, RouterView, useRouter } from 'vue-router'
   import { useAuthSession } from '@/composables/useAuthSession'

   const router = useRouter()
   const auth = useAuthSession()

   function closeSession() {
     auth.logout()
     router.push({ name: 'login' })
   }
   </script>

   <template>
     <div class="projecthub-layout">
       <header>
         <div>
           <p>ProjectHub</p>
           <h1>Área de trabajo</h1>
         </div>

         <div>
           <span>
             Sesión: {{ auth.currentUser.value?.name }}
           </span>

           <button type="button" @click="closeSession">
             Cerrar sesión
           </button>
         </div>
       </header>

       <nav aria-label="Navegación principal">
         <RouterLink :to="{ name: 'dashboard' }">
           Dashboard
         </RouterLink>

         <RouterLink :to="{ name: 'projects' }">
           Proyectos
         </RouterLink>

         <RouterLink
           :to="{ name: 'task-detail', params: { taskId: 'task-101' } }"
         >
           Tarea de ejemplo
         </RouterLink>

         <RouterLink
           v-if="auth.roles.value.includes('admin')"
           :to="{ name: 'admin' }"
         >
           Administración
         </RouterLink>
       </nav>

       <main>
         <RouterView />
       </main>
     </div>
   </template>

   <style scoped>
   .projecthub-layout {
     max-width: 960px;
     margin: 0 auto;
     padding: 2rem;
     font-family: Arial, sans-serif;
   }

   header,
   nav {
     display: flex;
     justify-content: space-between;
     gap: 1rem;
     align-items: center;
   }

   nav {
     justify-content: flex-start;
     flex-wrap: wrap;
     margin: 1.5rem 0;
   }

   a {
     color: #334155;
   }

   .is-active {
     color: #1d4ed8;
     font-weight: 700;
   }

   button {
     cursor: pointer;
   }
   </style>
   ```

2. Crea `src/views/LoginView.vue`.

   ```vue
   <script setup>
   import { computed, reactive } from 'vue'
   import { useRoute, useRouter } from 'vue-router'
   import { useAuthSession } from '@/composables/useAuthSession'

   const router = useRouter()
   const route = useRoute()
   const auth = useAuthSession()

   const form = reactive({
     email: 'user@projecthub.dev',
     password: 'projecthub123'
   })

   const redirectTarget = computed(() => {
     const redirect = route.query.redirect

     if (typeof redirect === 'string' && redirect.startsWith('/')) {
       return redirect
     }

     return '/app/dashboard'
   })

   async function submitLogin() {
     try {
       await auth.login({
         email: form.email,
         password: form.password
       })

       await router.replace(redirectTarget.value)
     } catch {
       // El mensaje se muestra mediante auth.error.
     }
   }
   </script>

   <template>
     <main class="login-view">
       <section>
         <h1>Iniciar sesión en ProjectHub</h1>
         <p>Usa una cuenta simulada para acceder al área de trabajo.</p>

         <form @submit.prevent="submitLogin">
           <label for="email">Correo electrónico</label>
           <input
             id="email"
             v-model.trim="form.email"
             type="email"
             autocomplete="email"
             required
           >

           <label for="password">Contraseña</label>
           <input
             id="password"
             v-model="form.password"
             type="password"
             autocomplete="current-password"
             required
           >

           <p v-if="auth.error.value" role="alert">
             {{ auth.error.value.message }}
           </p>

           <button type="submit" :disabled="auth.isLoading.value">
             {{ auth.isLoading.value ? 'Validando…' : 'Iniciar sesión' }}
           </button>
         </form>

         <aside>
           <h2>Credenciales disponibles</h2>
           <ul>
             <li>Usuario: user@projecthub.dev / projecthub123</li>
             <li>Admin: admin@projecthub.dev / projecthub123</li>
           </ul>
         </aside>
       </section>
     </main>
   </template>

   <style scoped>
   .login-view {
     max-width: 420px;
     margin: 4rem auto;
     padding: 2rem;
     font-family: Arial, sans-serif;
   }

   form {
     display: grid;
     gap: 0.75rem;
   }

   input,
   button {
     padding: 0.65rem;
   }

   [role='alert'] {
     color: #b91c1c;
   }
   </style>
   ```

3. Crea `src/views/DashboardView.vue`.

   ```vue
   <template>
     <section>
       <h2>Dashboard</h2>
       <p>Resumen de actividad, proyectos y tareas asignadas.</p>
     </section>
   </template>
   ```

4. Crea `src/views/ProjectsView.vue`.

   ```vue
   <script setup>
   import { RouterLink } from 'vue-router'

   const projects = [
     { id: 'project-01', name: 'Modernización de ProjectHub' },
     { id: 'project-02', name: 'Automatización de calidad' }
   ]
   </script>

   <template>
     <section>
       <h2>Proyectos</h2>

       <ul>
         <li v-for="project in projects" :key="project.id">
           {{ project.name }}
         </li>
       </ul>

       <p>
         Abre una tarea desde la navegación principal para verificar el parámetro
         dinámico <code>:taskId</code>.
       </p>

       <RouterLink
         :to="{ name: 'task-detail', params: { taskId: 'task-202' } }"
       >
         Abrir tarea task-202
       </RouterLink>
     </section>
   </template>
   ```

5. Crea `src/views/TaskDetailView.vue`. La opción `props: true` en el router entrega `taskId` como prop.

   ```vue
   <script setup>
   defineProps({
     taskId: {
       type: String,
       required: true
     }
   })
   </script>

   <template>
     <section>
       <h2>Detalle de tarea</h2>
       <p>Identificador recibido desde la ruta: <strong>{{ taskId }}</strong></p>
     </section>
   </template>
   ```

6. Crea `src/views/AdminView.vue`.

   ```vue
   <template>
     <section>
       <h1>Administración</h1>
       <p>
         Esta vista se carga bajo demanda y solo está disponible para usuarios
         con el rol <code>admin</code>.
       </p>
     </section>
   </template>
   ```

7. Crea `src/views/ForbiddenView.vue`.

   ```vue
   <script setup>
   import { RouterLink, useRoute } from 'vue-router'

   const route = useRoute()
   </script>

   <template>
     <main>
       <h1>Acceso prohibido</h1>
       <p>No tienes permisos para acceder a este recurso.</p>
       <p v-if="route.query.redirect">
         Recurso solicitado: <code>{{ route.query.redirect }}</code>
       </p>
       <RouterLink :to="{ name: 'dashboard' }">
         Volver al dashboard
       </RouterLink>
     </main>
   </template>
   ```

8. Crea `src/views/NotFoundView.vue`.

   ```vue
   <script setup>
   import { RouterLink } from 'vue-router'
   </script>

   <template>
     <main>
       <h1>Página no encontrada</h1>
       <p>La URL solicitada no coincide con ninguna ruta de ProjectHub.</p>
       <RouterLink :to="{ name: 'dashboard' }">
         Ir al dashboard
       </RouterLink>
     </main>
   </template>
   ```

9. Crea `src/views/ChunkRecoveryView.vue`.

   ```vue
   <script setup>
   import { computed } from 'vue'
   import { useRoute } from 'vue-router'

   const route = useRoute()

   const target = computed(() => {
     const value = route.query.target

     if (typeof value === 'string' && value.startsWith('/')) {
       return value
     }

     return '/app/dashboard'
   })

   function retry() {
     window.location.assign(target.value)
   }
   </script>

   <template>
     <main>
       <h1>No se pudo cargar una parte de la aplicación</h1>
       <p>
         Es posible que haya una versión nueva disponible o que se haya producido
         un error de red al cargar una vista bajo demanda.
       </p>

       <button type="button" @click="retry">
         Recargar e intentar nuevamente
       </button>
     </main>
   </template>
   ```

**Resultado esperado:**

- La vista de login permite iniciar sesión.
- `ProjectHubLayout` contiene el segundo `<RouterView />` necesario para las rutas hijas de `/app`.
- El enlace administrativo solo se muestra a usuarios con rol `admin`.
- `TaskDetailView` recibe el parámetro `taskId` como prop.
- Existe una vista específica para recuperación de carga dinámica.

**Verificación:**

Inicia el servidor:

```bash
npm run dev -- --host 127.0.0.1 --port 5173
```

Accede a:

```text
http://127.0.0.1:5173/login
```

Inicia sesión con:

```text
user@projecthub.dev
projecthub123
```

Debes terminar en:

```text
http://127.0.0.1:5173/app/dashboard
```

---

### Paso 6. Verificar redirecciones, roles, carga perezosa y telemetría

**Objetivo:** Validar el comportamiento funcional completo del flujo de navegación.

**Instrucciones:**

1. Con sesión cerrada, abre directamente:

   ```text
   http://127.0.0.1:5173/app/projects
   ```

2. Inicia sesión como usuario estándar:

   ```text
   user@projecthub.dev
   projecthub123
   ```

3. Comprueba que se conserva el destino original y que terminas en `/app/projects`.

4. Abre:

   ```text
   http://127.0.0.1:5173/app/tasks/task-202
   ```

5. Confirma que la página muestra `task-202`.

6. Como usuario estándar, abre:

   ```text
   http://127.0.0.1:5173/admin
   ```

7. Comprueba que eres redirigido a `/forbidden`.

8. Cierra sesión desde el layout.

9. Inicia sesión como administrador:

   ```text
   admin@projecthub.dev
   projecthub123
   ```

10. Abre `/admin` y confirma que se muestra la vista administrativa.

11. Abre las herramientas de desarrollo de Chrome, pestaña **Network**, y activa el filtro **JS**. Navega entre Dashboard, Proyectos, Detalle de tarea y Administración.

12. Observa solicitudes de archivos JavaScript generados para las vistas cargadas dinámicamente. Los nombres exactos de los chunks dependen de Vite, pero deben cargarse al navegar por primera vez a cada vista lazy.

13. Revisa la consola de Chrome. Debes observar mensajes similares a:

   ```text
   [ProjectHub telemetry] {
     event: "navigation",
     from: "/app/dashboard",
     to: "/app/projects",
     routeName: "projects",
     status: "completed"
   }
   ```

**Resultado esperado:**

- Los visitantes no autenticados son enviados a `/login`.
- Después del login se restaura la ruta solicitada originalmente.
- El rol `user` no puede acceder a `/admin`.
- El rol `admin` sí puede acceder a `/admin`.
- Las vistas protegidas configuradas con `import()` se descargan de forma diferida.
- Cada navegación genera telemetría en consola.

**Verificación:**

Ejecuta estas comprobaciones manuales:

| Caso | Acción | Resultado esperado |
|---|---|---|
| Ruta pública | Abrir `/login` sin sesión | Se visualiza el formulario de login |
| Ruta protegida | Abrir `/app/dashboard` sin sesión | Redirección a `/login?redirect=/app/dashboard` |
| Redirección posterior | Autenticarse después del caso anterior | Retorno a `/app/dashboard` |
| Parámetro dinámico | Abrir `/app/tasks/task-202` | Se muestra `task-202` |
| Usuario sin permisos | Autenticarse como `user` y abrir `/admin` | Redirección a `/forbidden` |
| Administrador | Autenticarse como `admin` y abrir `/admin` | Se carga `AdminView` |
| Ruta inválida | Abrir `/ruta-inexistente` | Se muestra `NotFoundView` |
| Error de chunk | Simular fallo de importación dinámica | Se muestra `ChunkRecoveryView` |

Para una comprobación rápida de calidad estática, ejecuta el script disponible en tu proyecto:

```bash
npm run lint
```

Si el proyecto no incluye el script `lint`, inspecciona la compilación ejecutando:

```bash
npm run build
```

La compilación debe finalizar sin errores de importación, rutas no resueltas ni errores de sintaxis.

## Validación y pruebas

Realiza la siguiente secuencia final sin reiniciar el navegador entre pasos:

1. Abre `http://127.0.0.1:5173/app/dashboard` sin sesión.
2. Verifica la redirección a login.
3. Inicia sesión como `user@projecthub.dev`.
4. Navega a Proyectos y a una tarea.
5. Intenta acceder manualmente a `/admin`.
6. Comprueba la pantalla de acceso prohibido.
7. Cierra sesión.
8. Inicia sesión como `admin@projecthub.dev`.
9. Accede a `/admin`.
10. Visita una ruta inexistente, por ejemplo:

    ```text
    http://127.0.0.1:5173/no-existe
    ```

11. Abre la consola y confirma que las navegaciones producen eventos `[ProjectHub telemetry]`.

Para comprobar manualmente el flujo de recuperación de chunks durante desarrollo, puedes modificar temporalmente una importación dinámica a una ruta inexistente:

```js
component: () => import('@/views/ArchivoInexistente.vue')
```

Navega a esa ruta, verifica que el router dirige a `ChunkRecoveryView` y restaura inmediatamente la importación correcta. No conserves esta modificación en el código final.

## Solución de problemas

### 1. La solicitud de login intenta llegar a una API real o aparece `Failed to fetch`

**Síntomas:** Al enviar el formulario de login se muestra un error de red, no aparece `[MSW] Mocking enabled.` en consola o la pestaña Network indica que la solicitud no fue interceptada.

**Causa:** El worker de MSW no se generó en `public/`, `src/mocks/browser.js` no se carga desde `main.js`, o la URL del handler no coincide exactamente con la URL usada por `useAuthSession.js`.

**Solución:**

```bash
npx msw@2.7.3 init public/ --save
```

Confirma además que:

- Existe `public/mockServiceWorker.js`.
- `main.js` contiene `await worker.start(...)` dentro de `if (import.meta.env.DEV)`.
- `handlers.js` intercepta `http://127.0.0.1:5173/api/auth/login`.
- `useAuthSession.js` usa el mismo origen y ruta.
- Vite se ejecuta exactamente con `--host 127.0.0.1 --port 5173`.

### 2. La URL cambia a `/app/dashboard`, pero no se muestra la vista hija

**Síntomas:** La navegación funciona y la URL es correcta, pero el área principal permanece vacía o solo se ve la cabecera del layout.

**Causa:** `ProjectHubLayout.vue` no contiene un `<RouterView />`, o se importó `RouterView` incorrectamente.

**Solución:**

Verifica que `ProjectHubLayout.vue` tenga esta importación:

```js
import { RouterLink, RouterView, useRouter } from 'vue-router'
```

También debe contener este punto de renderizado dentro de su plantilla:

```vue
<main>
  <RouterView />
</main>
```

Las rutas hijas de `/app` solo pueden mostrarse dentro del `<RouterView />` del componente padre.

## Limpieza

1. Detén el servidor de desarrollo con `Ctrl+C`.
2. Verifica que no queden procesos adicionales usando el puerto `5173`.
3. No inicies ni dejes servicios ejecutándose en el puerto `3000`.
4. Revisa los cambios pendientes antes de continuar con el siguiente laboratorio:

   ```bash
   git status
   ```

5. Mantén la rama de trabajo `main`; no crees ramas adicionales para esta tanda de laboratorios.
6. Conserva `public/mockServiceWorker.js`, los handlers de MSW, los composables y la configuración del router. Serán la base para la migración a Pinia del laboratorio `04-00-01`.

## Resumen

En este laboratorio implementaste una arquitectura de navegación avanzada para ProjectHub. Configuraste rutas públicas, rutas protegidas y rutas anidadas con `ProjectHubLayout` y `<RouterView />`. También aplicaste `meta.requiresAuth`, `meta.roles`, guards globales `beforeEach`, telemetría con `afterEach` y lazy loading mediante importaciones dinámicas.

La autenticación se simuló con MSW a través del endpoint `POST /api/auth/login`, mientras que `useAsyncState.js` centralizó los estados de carga y error. Finalmente, añadiste una ruta comodín para recursos no encontrados y una pantalla de recuperación para errores de carga de chunks. En el siguiente bloque, el estado de sesión y las definiciones relacionadas se migrarán a stores Pinia con persistencia controlada y efectos secundarios centralizados.
