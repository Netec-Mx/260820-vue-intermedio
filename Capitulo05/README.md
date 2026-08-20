# 4 Practica: implementar clientes REST y GraphQL con manejo de cache y errores

## Metadatos

| Propiedad | Valor |
|---|---|
| Duración | 46 minutos |
| Complejidad | Difícil |
| Nivel de Bloom | Aplicar |

## Descripción general

En este laboratorio extenderás la SPA **ProjectHub** creada en el laboratorio anterior para consumir APIs REST y GraphQL simuladas mediante MSW. Implementarás un cliente Axios con autenticación, tiempo de espera e interceptores de normalización de errores, además de Apollo Client con caché normalizada para proyectos y tareas.

Los stores de Pinia delegarán la comunicación en los clientes de datos, mantendrán los estados de operación centralizados y mostrarán notificaciones coherentes ante errores de red, autorización, validación o indisponibilidad del servicio.

## Objetivos de aprendizaje

Al finalizar el laboratorio podrás:

- [ ] Crear un cliente REST basado en Axios con `baseURL`, `timeout` e interceptores de autenticación y errores.
- [ ] Modelar errores de red como objetos `AppError` consistentes para toda la aplicación.
- [ ] Configurar Apollo Client con `InMemoryCache`, `typePolicies` y consultas GraphQL con variables.
- [ ] Integrar servicios REST y GraphQL en los stores `useProjectsStore` y `useTasksStore`.
- [ ] Verificar que Apollo muestra datos cacheados mientras realiza una actualización en segundo plano.

## Prerrequisitos

### Conocimientos requeridos

- Laboratorio **04-00-01** finalizado, con Pinia configurado, persistencia selectiva y stores modulares activos.
- Conocimiento básico de métodos HTTP, códigos de estado, cabeceras y cuerpos JSON.
- Conocimiento básico de GraphQL: consultas, variables, campos y respuesta `data`.
- Uso básico de Vue 3 con Composition API y `<script setup>`.
- Navegación por Chrome DevTools.

### Acceso y dependencias

Debes disponer de:

- El repositorio de trabajo con una única rama llamada `main`.
- La aplicación existente denominada `projecthub-spa`.
- Node.js `22.14.0` y npm `10.9.2`.
- Google Chrome instalado.
- Docker Desktop no es necesario para este laboratorio.
- Puertos locales `5173` y `4173` disponibles.
- No debes iniciar ningún servicio en el puerto `3000`.

| Herramienta | Versión esperada |
|---|---:|
| Vue | 3.5.13 |
| Vite | 6.1.0 |
| Pinia | 3.0.1 |
| Axios | 1.8.4 |
| Apollo Client | 3.13.8 |
| GraphQL | 16.10.0 |
| MSW | 2.7.3 |

## Entorno del laboratorio

El directorio de trabajo obligatorio es:

```bash
/workspace/vue-intermedio-labs
```

En Windows, utiliza el equivalente:

```powershell
C:\workspace\vue-intermedio-labs
```

### Preparación inicial

1. Abre una terminal.
2. Accede al proyecto.
3. Confirma que trabajas en la rama `main`.
4. Instala las dependencias necesarias si aún no están presentes.

```bash
cd /workspace/vue-intermedio-labs/projecthub-spa
git branch --show-current
npm install
npm install axios@1.8.4 @apollo/client@3.13.8 graphql@16.10.0 msw@2.7.3
```

**Salida esperada**

El comando de rama debe mostrar:

```text
main
```

**Verificación**

Comprueba las dependencias instaladas:

```bash
npm ls axios @apollo/client graphql msw
```

La salida debe contener las versiones compatibles con las indicadas en la tabla anterior.

---

## Paso a paso

### Paso 1. Preparar las variables de entorno y el servicio worker de MSW

**Objetivo:** Configurar las URL constantes de REST y GraphQL y asegurar que MSW intercepte las solicitudes únicamente en desarrollo.

**Instrucciones**

1. Crea o actualiza el archivo `.env.local` en la raíz de `projecthub-spa`.

```dotenv
VITE_API_URL=http://127.0.0.1:5173/api
VITE_GRAPHQL_URL=http://127.0.0.1:5173/graphql
```

2. Genera el archivo de servicio de MSW en la carpeta pública. Ejecuta este comando desde la raíz del proyecto:

```bash
npx msw@2.7.3 init public --save
```

3. Verifica que existe el archivo:

```bash
ls public/mockServiceWorker.js
```

4. Crea o actualiza `src/mocks/browser.js`.

```js
// src/mocks/browser.js
import { setupWorker } from 'msw/browser'
import { handlers } from './handlers'

export const worker = setupWorker(...handlers)
```

5. Ajusta el archivo `src/main.js` o `src/main.ts` para iniciar MSW solo en desarrollo antes de montar Vue.

```js
// src/main.js
import { createApp } from 'vue'
import { createPinia } from 'pinia'
import App from './App.vue'
import router from './router'

async function bootstrap() {
  if (import.meta.env.DEV) {
    const { worker } = await import('./mocks/browser')

    await worker.start({
      onUnhandledRequest: 'bypass'
    })
  }

  const app = createApp(App)
  const pinia = createPinia()

  app.use(pinia)
  app.use(router)
  app.mount('#app')
}

bootstrap()
```

> Si el laboratorio anterior ya tiene una función `bootstrap`, conserva su estructura y añade únicamente el bloque de MSW antes de `app.mount()`.

**Salida esperada**

Al iniciar Vite en modo desarrollo, Chrome mostrará en la consola un mensaje similar a:

```text
[MSW] Mocking enabled.
```

**Verificación**

Ejecuta el servidor local:

```bash
npm run dev -- --host 127.0.0.1 --port 5173
```

Abre:

```text
http://127.0.0.1:5173
```

En Chrome DevTools, pestaña **Console**, confirma que MSW está activo y no aparecen errores de carga para `mockServiceWorker.js`.

---

### Paso 2. Definir datos simulados y handlers REST y GraphQL

**Objetivo:** Proporcionar endpoints simulados coherentes para Axios y Apollo Client.

**Instrucciones**

1. Crea el archivo `src/mocks/handlers.js`.
2. Añade datos de proyectos y tareas.
3. Define handlers REST para:
   - `GET /api/projects`
   - `GET /api/projects/:id/tasks`
   - `POST /api/tasks`
4. Define handlers GraphQL para:
   - Consulta de proyectos.
   - Consulta de tareas de un proyecto.
   - Consulta de detalle de tarea.

```js
// src/mocks/handlers.js
import { http, HttpResponse, graphql } from 'msw'

const projects = [
  {
    id: 'p-100',
    name: 'Portal de clientes',
    description: 'SPA para seguimiento de solicitudes de clientes.',
    status: 'active'
  },
  {
    id: 'p-200',
    name: 'Migración de facturación',
    description: 'Actualización del módulo de facturación.',
    status: 'planning'
  },
  {
    id: 'p-300',
    name: 'Observabilidad',
    description: 'Paneles y alertas para servicios internos.',
    status: 'active'
  }
]

let tasks = [
  {
    id: 't-101',
    projectId: 'p-100',
    title: 'Diseñar pantalla de acceso',
    completed: true,
    priority: 'high'
  },
  {
    id: 't-102',
    projectId: 'p-100',
    title: 'Implementar recuperación de contraseña',
    completed: false,
    priority: 'medium'
  },
  {
    id: 't-201',
    projectId: 'p-200',
    title: 'Mapear campos del sistema anterior',
    completed: false,
    priority: 'high'
  },
  {
    id: 't-301',
    projectId: 'p-300',
    title: 'Configurar métricas de rendimiento',
    completed: false,
    priority: 'medium'
  }
]

function requireAuthorization(request) {
  const authorization = request.headers.get('Authorization')

  return authorization?.startsWith('Bearer ')
}

export const handlers = [
  http.get('http://127.0.0.1:5173/api/projects', ({ request }) => {
    if (!requireAuthorization(request)) {
      return HttpResponse.json(
        { message: 'La sesión no es válida o ha expirado.' },
        { status: 401 }
      )
    }

    return HttpResponse.json({ data: projects })
  }),

  http.get(
    'http://127.0.0.1:5173/api/projects/:projectId/tasks',
    ({ params, request }) => {
      if (!requireAuthorization(request)) {
        return HttpResponse.json(
          { message: 'La sesión no es válida o ha expirado.' },
          { status: 401 }
        )
      }

      const project = projects.find((item) => item.id === params.projectId)

      if (!project) {
        return HttpResponse.json(
          { message: 'El proyecto solicitado no existe.' },
          { status: 404 }
        )
      }

      return HttpResponse.json({
        data: tasks.filter((task) => task.projectId === params.projectId)
      })
    }
  ),

  http.post('http://127.0.0.1:5173/api/tasks', async ({ request }) => {
    if (!requireAuthorization(request)) {
      return HttpResponse.json(
        { message: 'La sesión no es válida o ha expirado.' },
        { status: 401 }
      )
    }

    const payload = await request.json()

    if (!payload.title?.trim() || !payload.projectId) {
      return HttpResponse.json(
        {
          message: 'No fue posible validar la tarea.',
          errors: {
            title: !payload.title?.trim()
              ? ['El título es obligatorio.']
              : [],
            projectId: !payload.projectId
              ? ['El proyecto es obligatorio.']
              : []
          }
        },
        { status: 422 }
      )
    }

    const projectExists = projects.some(
      (project) => project.id === payload.projectId
    )

    if (!projectExists) {
      return HttpResponse.json(
        { message: 'El proyecto asociado no existe.' },
        { status: 404 }
      )
    }

    const task = {
      id: `t-${Date.now()}`,
      projectId: payload.projectId,
      title: payload.title.trim(),
      completed: false,
      priority: payload.priority ?? 'medium'
    }

    tasks = [...tasks, task]

    return HttpResponse.json({ data: task }, { status: 201 })
  }),

  graphql.query('Projects', ({ request }) => {
    if (!requireAuthorization(request)) {
      return HttpResponse.json({
        errors: [
          {
            message: 'No autorizado.',
            extensions: { code: 'UNAUTHENTICATED' }
          }
        ]
      })
    }

    return HttpResponse.json({
      data: {
        projects
      }
    })
  }),

  graphql.query('ProjectTasks', ({ variables, request }) => {
    if (!requireAuthorization(request)) {
      return HttpResponse.json({
        errors: [
          {
            message: 'No autorizado.',
            extensions: { code: 'UNAUTHENTICATED' }
          }
        ]
      })
    }

    const project = projects.find((item) => item.id === variables.projectId)

    if (!project) {
      return HttpResponse.json({
        errors: [
          {
            message: 'Proyecto no encontrado.',
            extensions: { code: 'NOT_FOUND' }
          }
        ]
      })
    }

    return HttpResponse.json({
      data: {
        project: {
          ...project,
          tasks: tasks.filter((task) => task.projectId === variables.projectId)
        }
      }
    })
  }),

  graphql.query('TaskDetail', ({ variables, request }) => {
    if (!requireAuthorization(request)) {
      return HttpResponse.json({
        errors: [
          {
            message: 'No autorizado.',
            extensions: { code: 'UNAUTHENTICATED' }
          }
        ]
      })
    }

    const task = tasks.find((item) => item.id === variables.taskId)

    if (!task) {
      return HttpResponse.json({
        errors: [
          {
            message: 'Tarea no encontrada.',
            extensions: { code: 'NOT_FOUND' }
          }
        ]
      })
    }

    return HttpResponse.json({
      data: {
        task
      }
    })
  })
]
```

**Salida esperada**

El proyecto debe compilar sin errores de importación relacionados con `msw`, `http`, `HttpResponse` o `graphql`.

**Verificación**

Con la aplicación abierta, entra en Chrome DevTools:

1. Abre la pestaña **Network**.
2. Activa el filtro **Fetch/XHR**.
3. Navega a una vista que cargue proyectos tras completar los pasos posteriores.
4. Las peticiones deben indicar que son atendidas por el Service Worker, no por un servidor externo.

---

### Paso 3. Crear el contrato centralizado `AppError`

**Objetivo:** Convertir errores técnicos de Axios y Apollo en un contrato seguro y predecible para stores y componentes.

**Instrucciones**

1. Crea la carpeta `src/services` si no existe.
2. Crea `src/services/appError.js`.
3. Implementa la clase y los normalizadores para REST y GraphQL.

```js
// src/services/appError.js
export class AppError extends Error {
  constructor({
    message = 'No fue posible completar la operación.',
    type = 'unknown',
    status = null,
    details = null,
    cause = null
  } = {}) {
    super(message)

    this.name = 'AppError'
    this.type = type
    this.status = status
    this.details = details
    this.cause = cause
  }
}

function typeFromStatus(status) {
  if (status === 401) return 'authentication'
  if (status === 403) return 'authorization'
  if (status === 404) return 'not-found'
  if (status === 422) return 'validation'
  if (status === 500 || status === 502 || status === 503) return 'service'
  return 'http'
}

export function normalizeRestError(error) {
  if (error instanceof AppError) {
    return error
  }

  if (error.code === 'ERR_CANCELED') {
    return new AppError({
      message: 'La solicitud fue cancelada.',
      type: 'cancelled',
      cause: error
    })
  }

  if (error.code === 'ECONNABORTED' || error.code === 'ETIMEDOUT') {
    return new AppError({
      message: 'La solicitud excedió el tiempo de espera.',
      type: 'timeout',
      cause: error
    })
  }

  const status = error.response?.status ?? null
  const payload = error.response?.data

  if (!status) {
    return new AppError({
      message: 'No se pudo conectar con el servicio. Verifica tu conexión.',
      type: 'network',
      cause: error
    })
  }

  return new AppError({
    message:
      payload?.message ??
      `No fue posible completar la solicitud (${status}).`,
    type: typeFromStatus(status),
    status,
    details: payload?.errors ?? payload ?? null,
    cause: error
  })
}

export function normalizeGraphqlError(error) {
  if (error instanceof AppError) {
    return error
  }

  const graphQLError = error.graphQLErrors?.[0]
  const code = graphQLError?.extensions?.code

  if (code === 'UNAUTHENTICATED') {
    return new AppError({
      message: 'La sesión no es válida o ha expirado.',
      type: 'authentication',
      status: 401,
      details: graphQLError,
      cause: error
    })
  }

  if (code === 'NOT_FOUND') {
    return new AppError({
      message: graphQLError.message,
      type: 'not-found',
      status: 404,
      details: graphQLError,
      cause: error
    })
  }

  if (error.networkError) {
    return new AppError({
      message: 'No se pudo conectar con el servicio GraphQL.',
      type: 'network',
      details: error.networkError,
      cause: error
    })
  }

  return new AppError({
    message:
      graphQLError?.message ??
      error.message ??
      'No fue posible completar la consulta GraphQL.',
    type: 'graphql',
    details: graphQLError ?? null,
    cause: error
  })
}
```

**Salida esperada**

La aplicación dispondrá de un único formato de error con estas propiedades principales:

```js
{
  name: 'AppError',
  message: '...',
  type: 'validation',
  status: 422,
  details: {}
}
```

**Verificación**

En la consola de Chrome, después de provocar un error de validación en una tarea, el error gestionado debe cumplir:

```js
error.name === 'AppError'
error.type === 'validation'
error.status === 422
```

---

### Paso 4. Implementar el cliente REST con Axios e interceptores

**Objetivo:** Crear `restClient.js` con autenticación simulada, tiempo de espera y normalización de errores.

**Instrucciones**

1. Crea el archivo `src/services/restClient.js`.
2. Configura Axios con `baseURL`, `timeout` y la cabecera `Accept`.
3. Añade un interceptor de solicitud que lea el token de autenticación persistido.
4. Añade un interceptor de respuesta que convierta todos los fallos en `AppError`.

```js
// src/services/restClient.js
import axios from 'axios'
import { normalizeRestError } from './appError'

export const restClient = axios.create({
  baseURL: import.meta.env.VITE_API_URL,
  timeout: 8_000,
  headers: {
    Accept: 'application/json'
  }
})

restClient.interceptors.request.use((config) => {
  const token = localStorage.getItem('access_token')

  if (token) {
    config.headers.Authorization = `Bearer ${token}`
  }

  return config
})

restClient.interceptors.response.use(
  (response) => response,
  (error) => Promise.reject(normalizeRestError(error))
)
```

5. Crea `src/services/projectsRestApi.js`.

```js
// src/services/projectsRestApi.js
import { restClient } from './restClient'

export async function getProjects({ signal } = {}) {
  const response = await restClient.get('/projects', { signal })

  return response.data.data
}

export async function getProjectTasks(projectId, { signal } = {}) {
  const response = await restClient.get(`/projects/${projectId}/tasks`, {
    signal
  })

  return response.data.data
}
```

6. Crea `src/services/tasksRestApi.js`.

```js
// src/services/tasksRestApi.js
import { restClient } from './restClient'

export async function createTask(payload) {
  const response = await restClient.post('/tasks', payload, {
    headers: {
      'Content-Type': 'application/json'
    }
  })

  return response.data.data
}
```

> No añadas `Content-Type: application/json` a la configuración global de Axios. Una carga futura basada en `FormData` requerirá que el navegador genere automáticamente la cabecera multipart correcta.

**Salida esperada**

Todas las peticiones REST incluirán una cabecera como:

```text
Authorization: Bearer <token>
```

cuando exista `access_token` en `localStorage`.

**Verificación**

1. Inicia sesión mediante el flujo simulado creado en laboratorios anteriores.
2. Abre Chrome DevTools > **Application** > **Local Storage**.
3. Comprueba que existe `access_token`.
4. En **Network**, selecciona una petición a `/api/projects`.
5. En **Request Headers**, verifica la cabecera `Authorization`.

---

### Paso 5. Configurar Apollo Client y la caché normalizada

**Objetivo:** Configurar Apollo Client para utilizar GraphQL, autenticación y caché basada en identidad de proyectos y tareas.

**Instrucciones**

1. Crea el archivo `src/services/graphqlClient.js`.
2. Configura un enlace HTTP hacia `VITE_GRAPHQL_URL`.
3. Añade un enlace de contexto que incluya el token.
4. Configura `InMemoryCache` con políticas para `Project` y `Task`.

```js
// src/services/graphqlClient.js
import {
  ApolloClient,
  HttpLink,
  InMemoryCache,
  from
} from '@apollo/client/core'
import { setContext } from '@apollo/client/link/context'

const httpLink = new HttpLink({
  uri: import.meta.env.VITE_GRAPHQL_URL
})

const authLink = setContext((_, { headers }) => {
  const token = localStorage.getItem('access_token')

  return {
    headers: {
      ...headers,
      Authorization: token ? `Bearer ${token}` : ''
    }
  }
})

export const graphqlClient = new ApolloClient({
  link: from([authLink, httpLink]),
  cache: new InMemoryCache({
    typePolicies: {
      Project: {
        keyFields: ['id']
      },
      Task: {
        keyFields: ['id']
      },
      Query: {
        fields: {
          projects: {
            merge(_, incoming) {
              return incoming
            }
          }
        }
      }
    }
  }),
  connectToDevTools: import.meta.env.DEV
})
```

5. Crea `src/services/projectsGraphqlApi.js`.

```js
// src/services/projectsGraphqlApi.js
import { gql } from '@apollo/client/core'
import { graphqlClient } from './graphqlClient'
import { normalizeGraphqlError } from './appError'

export const PROJECTS_QUERY = gql`
  query Projects {
    projects {
      id
      name
      description
      status
    }
  }
`

export const PROJECT_TASKS_QUERY = gql`
  query ProjectTasks($projectId: ID!) {
    project(id: $projectId) {
      id
      name
      description
      status
      tasks {
        id
        projectId
        title
        completed
        priority
      }
    }
  }
`

export async function getProjectsFromGraphql() {
  try {
    const result = await graphqlClient.query({
      query: PROJECTS_QUERY,
      fetchPolicy: 'cache-and-network',
      errorPolicy: 'none'
    })

    return result.data.projects
  } catch (error) {
    throw normalizeGraphqlError(error)
  }
}

export async function getProjectTasksFromGraphql(projectId) {
  try {
    const result = await graphqlClient.query({
      query: PROJECT_TASKS_QUERY,
      variables: { projectId },
      fetchPolicy: 'cache-first',
      errorPolicy: 'none'
    })

    return result.data.project
  } catch (error) {
    throw normalizeGraphqlError(error)
  }
}
```

**Salida esperada**

Apollo identificará de forma única los objetos en caché con referencias equivalentes a:

```text
Project:{"id":"p-100"}
Task:{"id":"t-101"}
```

**Verificación**

Instala la extensión **Apollo Client Devtools** en Chrome si está disponible en tu entorno. Después:

1. Recarga la aplicación.
2. Abre DevTools.
3. Accede a la pestaña de Apollo.
4. Revisa que la caché contiene entidades `Project` y `Task` después de navegar por las vistas correspondientes.

---

### Paso 6. Integrar los clientes en los stores de Pinia

**Objetivo:** Delegar las solicitudes de red en servicios especializados y mantener los estados de interfaz centralizados.

**Instrucciones**

1. Identifica los nombres reales de los stores creados en el laboratorio 04-00-01.
2. En esta guía se asume que existen:
   - `useApiStatusStore`
   - `useUiStore`
   - `useProjectsStore`
   - `useTasksStore`
3. Si tus métodos tienen nombres distintos, conserva su comportamiento y adapta las llamadas mostradas.

El store de estado de API debe poder registrar operaciones. Debe exponer, como mínimo, métodos equivalentes a:

```js
apiStatus.start('projects:list')
apiStatus.finish('projects:list')
```

El store de interfaz debe poder mostrar una notificación. Debe exponer un método equivalente a:

```js
ui.notify({
  type: 'error',
  message: 'Mensaje visible para el usuario.'
})
```

4. Actualiza `src/stores/projects.js` o el archivo equivalente.

```js
// src/stores/projects.js
import { defineStore } from 'pinia'
import { ref } from 'vue'
import { getProjects } from '../services/projectsRestApi'
import {
  getProjectsFromGraphql,
  getProjectTasksFromGraphql
} from '../services/projectsGraphqlApi'
import { useApiStatusStore } from './apiStatus'
import { useUiStore } from './ui'

export const useProjectsStore = defineStore('projects', () => {
  const projects = ref([])
  const selectedProject = ref(null)

  function reportError(error) {
    const ui = useUiStore()

    if (error.type !== 'cancelled') {
      ui.notify({
        type: 'error',
        message: error.message
      })
    }
  }

  async function loadProjectsRest({ signal } = {}) {
    const apiStatus = useApiStatusStore()
    const operation = 'projects:list:rest'

    apiStatus.start(operation)

    try {
      projects.value = await getProjects({ signal })
    } catch (error) {
      reportError(error)
      throw error
    } finally {
      apiStatus.finish(operation)
    }
  }

  async function loadProjectsPanel() {
    const apiStatus = useApiStatusStore()
    const operation = 'projects:list:graphql'

    apiStatus.start(operation)

    try {
      projects.value = await getProjectsFromGraphql()
    } catch (error) {
      reportError(error)
      throw error
    } finally {
      apiStatus.finish(operation)
    }
  }

  async function loadProjectWithTasks(projectId) {
    const apiStatus = useApiStatusStore()
    const operation = `projects:${projectId}:graphql`

    apiStatus.start(operation)

    try {
      selectedProject.value = await getProjectTasksFromGraphql(projectId)

      return selectedProject.value
    } catch (error) {
      reportError(error)
      throw error
    } finally {
      apiStatus.finish(operation)
    }
  }

  return {
    projects,
    selectedProject,
    loadProjectsRest,
    loadProjectsPanel,
    loadProjectWithTasks
  }
})
```

5. Actualiza `src/stores/tasks.js` o el archivo equivalente.

```js
// src/stores/tasks.js
import { defineStore } from 'pinia'
import { ref } from 'vue'
import { getProjectTasks } from '../services/projectsRestApi'
import { createTask } from '../services/tasksRestApi'
import { useApiStatusStore } from './apiStatus'
import { useUiStore } from './ui'

export const useTasksStore = defineStore('tasks', () => {
  const tasksByProject = ref({})

  function reportError(error) {
    const ui = useUiStore()

    if (error.type !== 'cancelled') {
      ui.notify({
        type: 'error',
        message: error.message
      })
    }
  }

  async function loadTasksForProject(projectId, { signal } = {}) {
    const apiStatus = useApiStatusStore()
    const operation = `tasks:list:${projectId}`

    apiStatus.start(operation)

    try {
      const tasks = await getProjectTasks(projectId, { signal })
      tasksByProject.value[projectId] = tasks

      return tasks
    } catch (error) {
      reportError(error)
      throw error
    } finally {
      apiStatus.finish(operation)
    }
  }

  async function addTask(payload) {
    const apiStatus = useApiStatusStore()
    const operation = 'tasks:create'

    apiStatus.start(operation)

    try {
      const task = await createTask(payload)
      const currentTasks = tasksByProject.value[task.projectId] ?? []

      tasksByProject.value[task.projectId] = [...currentTasks, task]

      return task
    } catch (error) {
      reportError(error)
      throw error
    } finally {
      apiStatus.finish(operation)
    }
  }

  return {
    tasksByProject,
    loadTasksForProject,
    addTask
  }
})
```

**Salida esperada**

Los componentes no construirán URLs ni usarán directamente Axios o Apollo. En su lugar, invocarán acciones de Pinia como:

```js
projectsStore.loadProjectsPanel()
tasksStore.loadTasksForProject(projectId)
tasksStore.addTask(payload)
```

**Verificación**

Busca usos directos de Axios en los componentes:

```bash
grep -R "axios\|restClient\|graphqlClient" src/components src/views
```

La salida no debe mostrar clientes HTTP utilizados directamente dentro de componentes o vistas, salvo importaciones deliberadas para diagnóstico.

---

### Paso 7. Consumir los stores desde las vistas y mostrar estados de carga

**Objetivo:** Mostrar proyectos y tareas sin acoplar las vistas a detalles de REST o GraphQL.

**Instrucciones**

1. Actualiza la vista del panel de proyectos, por ejemplo `src/views/ProjectsView.vue`.
2. Usa la acción GraphQL con `cache-and-network`.
3. Muestra una indicación de carga y un botón de actualización.

```vue
<!-- src/views/ProjectsView.vue -->
<script setup>
import { computed, onMounted } from 'vue'
import { useProjectsStore } from '../stores/projects'
import { useApiStatusStore } from '../stores/apiStatus'

const projectsStore = useProjectsStore()
const apiStatus = useApiStatusStore()

const loading = computed(() =>
  apiStatus.isLoading('projects:list:graphql')
)

onMounted(() => {
  projectsStore.loadProjectsPanel()
})
</script>

<template>
  <section>
    <header class="toolbar">
      <div>
        <h1>Proyectos</h1>
        <p>Consulta GraphQL con caché normalizada.</p>
      </div>

      <button :disabled="loading" @click="projectsStore.loadProjectsPanel">
        Actualizar
      </button>
    </header>

    <p v-if="loading" role="status">
      Cargando proyectos…
    </p>

    <p v-else-if="projectsStore.projects.length === 0">
      No hay proyectos disponibles.
    </p>

    <ul v-else class="project-list">
      <li
        v-for="project in projectsStore.projects"
        :key="project.id"
      >
        <RouterLink
          :to="{ name: 'project-detail', params: { id: project.id } }"
        >
          <strong>{{ project.name }}</strong>
          <span>{{ project.status }}</span>
        </RouterLink>
      </li>
    </ul>
  </section>
</template>
```

4. Actualiza la vista de detalle de proyecto, por ejemplo `src/views/ProjectDetailView.vue`.

```vue
<!-- src/views/ProjectDetailView.vue -->
<script setup>
import { computed, watch } from 'vue'
import { useRoute } from 'vue-router'
import { useProjectsStore } from '../stores/projects'
import { useApiStatusStore } from '../stores/apiStatus'

const route = useRoute()
const projectsStore = useProjectsStore()
const apiStatus = useApiStatusStore()

const projectId = computed(() => String(route.params.id))

const loading = computed(() =>
  apiStatus.isLoading(`projects:${projectId.value}:graphql`)
)

watch(
  projectId,
  async (id) => {
    await projectsStore.loadProjectWithTasks(id)
  },
  { immediate: true }
)
</script>

<template>
  <section>
    <p v-if="loading" role="status">
      Cargando detalle del proyecto…
    </p>

    <template v-else-if="projectsStore.selectedProject">
      <h1>{{ projectsStore.selectedProject.name }}</h1>
      <p>{{ projectsStore.selectedProject.description }}</p>

      <h2>Tareas</h2>

      <p v-if="projectsStore.selectedProject.tasks.length === 0">
        Este proyecto no tiene tareas.
      </p>

      <ul v-else>
        <li
          v-for="task in projectsStore.selectedProject.tasks"
          :key="task.id"
        >
          <input
            :checked="task.completed"
            type="checkbox"
            disabled
          >
          {{ task.title }} — prioridad: {{ task.priority }}
        </li>
      </ul>
    </template>
  </section>
</template>
```

5. Asegúrate de que existe una ruta con nombre `project-detail`.

```js
{
  path: '/projects/:id',
  name: 'project-detail',
  component: () => import('../views/ProjectDetailView.vue'),
  meta: { requiresAuth: true }
}
```

**Salida esperada**

La vista de proyectos debe mostrar los tres proyectos simulados. Al seleccionar uno, la aplicación debe cargar el detalle y sus tareas.

**Verificación**

1. Inicia sesión.
2. Accede al panel de proyectos.
3. Selecciona **Portal de clientes**.
4. Deben mostrarse las tareas:
   - Diseñar pantalla de acceso.
   - Implementar recuperación de contraseña.
5. Navega a otro proyecto y vuelve al primero.

---

### Paso 8. Implementar un formulario REST para crear tareas y tratar errores de validación

**Objetivo:** Consumir `POST /api/tasks` mediante Axios y mostrar errores normalizados.

**Instrucciones**

1. Crea `src/components/TaskCreateForm.vue`.
2. El formulario debe enviar la tarea al store, no al cliente REST directamente.
3. Para errores de validación, muestra los detalles del campo si existen.

```vue
<!-- src/components/TaskCreateForm.vue -->
<script setup>
import { computed, ref } from 'vue'
import { useTasksStore } from '../stores/tasks'
import { useApiStatusStore } from '../stores/apiStatus'

const props = defineProps({
  projectId: {
    type: String,
    required: true
  }
})

const emit = defineEmits(['created'])

const tasksStore = useTasksStore()
const apiStatus = useApiStatusStore()

const title = ref('')
const priority = ref('medium')
const fieldErrors = ref({})

const creating = computed(() => apiStatus.isLoading('tasks:create'))

async function submit() {
  fieldErrors.value = {}

  try {
    const task = await tasksStore.addTask({
      projectId: props.projectId,
      title: title.value,
      priority: priority.value
    })

    title.value = ''
    priority.value = 'medium'
    emit('created', task)
  } catch (error) {
    if (error.type === 'validation') {
      fieldErrors.value = error.details ?? {}
    }
  }
}
</script>

<template>
  <form @submit.prevent="submit">
    <h3>Nueva tarea</h3>

    <label for="task-title">Título</label>
    <input
      id="task-title"
      v-model="title"
      :aria-invalid="Boolean(fieldErrors.title?.length)"
      :disabled="creating"
      type="text"
    >
    <small v-if="fieldErrors.title?.length" role="alert">
      {{ fieldErrors.title[0] }}
    </small>

    <label for="task-priority">Prioridad</label>
    <select
      id="task-priority"
      v-model="priority"
      :disabled="creating"
    >
      <option value="low">Baja</option>
      <option value="medium">Media</option>
      <option value="high">Alta</option>
    </select>

    <button :disabled="creating" type="submit">
      {{ creating ? 'Creando…' : 'Crear tarea' }}
    </button>
  </form>
</template>
```

4. Importa el formulario en `ProjectDetailView.vue`.

```vue
<script setup>
import TaskCreateForm from '../components/TaskCreateForm.vue'
// Conserva las demás importaciones existentes.
</script>
```

5. Añade el componente dentro del bloque que muestra el proyecto.

```vue
<TaskCreateForm :project-id="projectsStore.selectedProject.id" />
```

**Salida esperada**

Al enviar una tarea válida, MSW debe responder con estado HTTP `201` y la nueva tarea debe incorporarse al store de tareas.

Al enviar el formulario vacío, debe aparecer:

```text
El título es obligatorio.
```

**Verificación**

En Chrome DevTools > **Network**:

1. Envía una tarea válida.
2. Selecciona la petición `POST /api/tasks`.
3. Confirma:
   - Método: `POST`
   - Estado: `201`
   - Cabecera `Authorization`
   - Cuerpo JSON con `projectId`, `title` y `priority`.

Después, envía el formulario sin título y confirma que la petición devuelve estado `422`.

---

## Validación y pruebas

Realiza la siguiente validación funcional completa.

### 1. Validación de compilación

Detén y reinicia Vite si modificaste `.env.local`:

```bash
npm run dev -- --host 127.0.0.1 --port 5173
```

La terminal no debe mostrar errores de compilación.

### 2. Validación de autenticación REST

1. Inicia sesión mediante el flujo simulado existente.
2. Navega al panel de proyectos.
3. En DevTools > **Network**, selecciona `GET /api/projects`.
4. Confirma que la cabecera `Authorization` contiene el token.

Para validar el caso no autorizado:

1. Abre DevTools > **Application** > **Local Storage**.
2. Elimina `access_token`.
3. Recarga la página protegida.
4. Ejecuta una operación REST, como crear una tarea.

Debe aparecer una notificación equivalente a:

```text
La sesión no es válida o ha expirado.
```

### 3. Validación de errores de formulario

1. Accede a un proyecto.
2. Deja vacío el campo de título.
3. Selecciona **Crear tarea**.

Resultado esperado:

- La respuesta de red es `422`.
- Se muestra un mensaje junto al campo de título.
- La aplicación no se bloquea.
- El error técnico no se muestra directamente al usuario.

### 4. Validación de caché Apollo

1. Inicia sesión nuevamente para restaurar el token.
2. Abre la vista de proyectos.
3. Navega al detalle de `p-100`.
4. Navega a otro proyecto.
5. Regresa a `p-100`.

Resultado esperado:

- Apollo conserva entidades `Project` y `Task` usando sus IDs.
- La segunda visita puede mostrar información existente de caché antes de recibir la respuesta de red.
- En DevTools > **Network** se observará una nueva consulta GraphQL, ya que el panel de proyectos usa `fetchPolicy: 'cache-and-network'`.
- En Apollo DevTools, la caché debe mostrar entidades normalizadas, no copias independientes de los mismos objetos.

### 5. Validación de separación de responsabilidades

Comprueba que:

- Los componentes no contienen URLs de API.
- Los componentes no añaden cabeceras `Authorization`.
- Los stores invocan funciones de servicios.
- Los servicios REST y GraphQL gestionan detalles de transporte.
- Los errores se convierten a `AppError`.
- El store de UI es responsable de las notificaciones.
- El store de estado de API es responsable de los indicadores de operación.

---

## Resolución de problemas

### Problema 1: MSW no intercepta las solicitudes y aparece `404` o `ERR_CONNECTION_REFUSED`

**Síntomas**

- Las solicitudes a `/api/projects` o `/graphql` fallan.
- La consola no muestra `[MSW] Mocking enabled`.
- En Network no aparece actividad asociada al Service Worker.

**Causa**

El archivo `public/mockServiceWorker.js` no fue generado, MSW se inició después de montar la aplicación o el navegador conserva un service worker anterior.

**Solución**

1. Genera nuevamente el archivo del worker:

   ```bash
   npx msw@2.7.3 init public --save
   ```

2. Comprueba que `src/main.js` ejecuta `await worker.start()` antes de `app.mount('#app')`.
3. En Chrome DevTools > **Application** > **Service Workers**, pulsa **Unregister** para eliminar workers obsoletos.
4. Recarga la página usando una recarga completa:

   ```text
   Ctrl+Shift+R
   ```

5. Confirma que Vite se ejecuta exactamente en:

   ```text
   http://127.0.0.1:5173
   ```

### Problema 2: Apollo devuelve “No autorizado” aunque el inicio de sesión parece correcto

**Síntomas**

- Las consultas GraphQL muestran una notificación de sesión expirada.
- REST funciona o el usuario parece autenticado visualmente.
- En Network, la petición a `/graphql` no contiene una cabecera `Authorization` válida.

**Causa**

El flujo de autenticación del laboratorio anterior guarda el token con una clave distinta de `access_token`, o Apollo Client fue creado antes de que el token se persistiera.

**Solución**

1. Revisa Chrome DevTools > **Application** > **Local Storage**.
2. Identifica la clave real donde el store de autenticación persiste el token.
3. Ajusta ambos clientes para leer la misma clave:

   ```js
   const token = localStorage.getItem('access_token')
   ```

4. Cierra sesión e inicia sesión otra vez.
5. Recarga la página y revisa que la cabecera de la consulta GraphQL sea:

   ```text
   Authorization: Bearer <token>
   ```

---

## Limpieza

1. Detén el servidor de Vite con:

   ```bash
   Ctrl+C
   ```

2. No elimines:
   - `public/mockServiceWorker.js`
   - `.env.local`
   - `src/mocks`
   - `src/services`
   - Los cambios en los stores de Pinia.

3. Revisa los archivos modificados:

   ```bash
   git status
   ```

4. Si tu flujo del curso requiere conservar el avance local, registra los cambios en la única rama permitida:

   ```bash
   git add .
   git commit -m "feat: add REST and GraphQL clients with cache and errors"
   ```

5. Confirma que continúas en la rama requerida:

   ```bash
   git branch --show-current
   ```

La salida debe ser:

```text
main
```

## Resumen

En este laboratorio implementaste una capa de datos escalable para ProjectHub:

- Axios consume endpoints REST con autenticación, tiempo de espera e interceptores.
- Los fallos REST y GraphQL se normalizan mediante el contrato `AppError`.
- Apollo Client consume GraphQL con autenticación y caché normalizada por `id`.
- Los stores de Pinia centralizan las operaciones, actualizan el estado de la aplicación y notifican errores mediante el store de UI.
- Las vistas se enfocan en la presentación, sin conocer URLs, cabeceras ni detalles de transporte.
- MSW permite desarrollar y validar la SPA sin iniciar una API real ni ocupar el puerto `3000`.

Como siguiente paso, esta arquitectura puede extenderse con mutaciones GraphQL, invalidación selectiva de caché, reintentos controlados, telemetría de errores y pruebas automatizadas de servicios y stores.
