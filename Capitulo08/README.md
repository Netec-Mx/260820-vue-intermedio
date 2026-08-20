# 4 Practica final: diseñar y desplegar una aplicación escalable con pipeline automatizado

## Metadatos

| Propiedad | Valor |
|---|---|
| Duración | 43 minutos |
| Complejidad | Difícil |
| Nivel de Bloom | Crear |

## Descripción general

En este laboratorio transformarás la SPA **Catalogo Pro**, validada en los laboratorios anteriores, en una plataforma desplegable y mantenible basada en un monorepo con `pnpm workspaces`. Migrarás la aplicación a `apps/web`, crearás paquetes compartidos para interfaz y configuración, definirás una pipeline de calidad y liberación, y construirás una imagen Docker servida por Nginx.

El resultado incluirá observabilidad mínima en cliente, una imagen versionada, un endpoint de salud, caché para recursos estáticos, documentación de arquitectura y un procedimiento verificable de rollback.

## Objetivos de aprendizaje

Al completar este laboratorio podrás:

- [ ] Reorganizar una SPA Vue en un monorepo con `apps/` y `packages/`, aplicando límites de responsabilidad.
- [ ] Crear configuraciones compartidas de TypeScript, ESLint y Vitest para un workspace de `pnpm`.
- [ ] Implementar una pipeline de GitHub Actions para lint, pruebas, cobertura, build y pruebas E2E con Cypress.
- [ ] Construir y ejecutar una imagen Docker de producción basada en `nginx:1.27.3-alpine`.
- [ ] Registrar errores, duración de navegación y versión desplegada mediante observabilidad cliente.
- [ ] Documentar promoción, verificación post-despliegue, criterios de detención y rollback por etiqueta de imagen.

## Requisitos previos

### Conocimientos requeridos

- Uso básico de Git, ramas, etiquetas y GitHub Actions.
- Manejo de `package.json`, scripts npm, YAML y variables de entorno.
- Conceptos de Dockerfile, imágenes, contenedores, puertos y healthchecks.
- Vue 3, Vite, Vue Router, Pinia, pruebas con Vitest y Cypress.
- Laboratorio 06-00-01 completado, incluyendo `docs/performance-report.md`.
- Laboratorio 07-00-01 completado, incluyendo pruebas unitarias, de componentes y E2E ejecutables.

### Accesos y servicios requeridos

- GitHub CLI autenticado:

  ```bash
  gh auth status
  ```

- Docker Desktop iniciado:

  ```bash
  docker version
  docker info
  ```

- Acceso para crear o actualizar un repositorio GitHub.
- El repositorio debe mantener `main` como única rama de trabajo persistente.

## Entorno del laboratorio

| Componente | Versión objetivo |
|---|---:|
| Node.js | 22.14.0 |
| pnpm | 9.15.4 |
| Vue | 3.5.13 |
| Vite | 6.1.0 |
| Docker Engine | 27.5.1 |
| Docker Desktop | 4.37.2 |
| Nginx | 1.27.3-alpine |
| GitHub CLI | 2.63.2 |
| Google Chrome | 132 o superior |
| Puerto Vite local | 5173 |
| Puerto de contenedor Nginx | 8080 |
| Puerto reservado para API futura | 3000 |

> **Importante:** No inicies servicios en el puerto `3000`. Durante desarrollo, MSW debe continuar interceptando las solicitudes REST y GraphQL desde el navegador.

### Preparación inicial

El repositorio final del laboratorio se ubicará en:

```text
/workspace/vue-intermedio/catalogo-platform
```

En Windows, usa el equivalente:

```text
C:\workspace\vue-intermedio\catalogo-platform
```

Ejecuta las siguientes comprobaciones:

```bash
node --version
npm --version
git --version
docker --version

corepack enable
corepack prepare pnpm@9.15.4 --activate
pnpm --version
```

La salida de `pnpm --version` debe indicar:

```text
9.15.4
```

## Procedimiento paso a paso

### Paso 1. Crear el monorepo y migrar Catalogo Pro

**Objetivo:** Crear la estructura base del monorepo y trasladar la SPA previamente validada a `apps/web`.

**Instrucciones:**

1. Crea el directorio de destino y entra en él:

   ```bash
   mkdir -p /workspace/vue-intermedio/catalogo-platform
   cd /workspace/vue-intermedio/catalogo-platform
   ```

2. Inicializa el repositorio Git solamente si aún no existe:

   ```bash
   git init -b main
   ```

3. Crea la estructura inicial:

   ```bash
   mkdir -p apps packages/ui/src/components packages/config docs .github/workflows
   ```

4. Copia la aplicación validada del laboratorio anterior. Ajusta la ruta de origen si tu repositorio anterior tiene otra ubicación:

   ```bash
   cp -R /workspace/vue-intermedio-labs/projecthub-spa apps/web
   ```

   Si el proyecto anterior se llama de otra forma, copia su contenido a `apps/web`.

5. Elimina artefactos que no deben migrarse:

   ```bash
   rm -rf apps/web/node_modules
   rm -rf apps/web/dist
   rm -f apps/web/package-lock.json
   rm -f apps/web/pnpm-lock.yaml
   ```

6. Verifica que los entregables obligatorios del laboratorio 06 y 07 estén presentes. Si se encontraban en el proyecto anterior, cópialos al nuevo repositorio:

   ```bash
   mkdir -p docs
   cp apps/web/docs/performance-report.md docs/performance-report.md 2>/dev/null || true
   cp apps/web/docs/test-report.md docs/test-report.md 2>/dev/null || true
   ```

7. Crea el archivo `pnpm-workspace.yaml` en la raíz:

   ```yaml
   packages:
     - "apps/*"
     - "packages/*"
   ```

8. Crea el `package.json` raíz:

   ```json
   {
     "name": "catalogo-platform",
     "version": "1.0.0",
     "private": true,
     "packageManager": "pnpm@9.15.4",
     "scripts": {
       "dev": "pnpm --filter @catalogo/web dev",
       "lint": "pnpm -r --if-present lint",
       "test": "pnpm -r --if-present test",
       "test:coverage": "pnpm -r --if-present test:coverage",
       "build": "pnpm --filter @catalogo/web build",
       "test:e2e": "pnpm --filter @catalogo/web test:e2e",
       "check": "pnpm lint && pnpm test:coverage && pnpm build",
       "docker:build": "docker build -t catalogo-platform:local .",
       "docker:run": "docker run --rm -p 8080:80 --name catalogo-platform catalogo-platform:local"
     }
   }
   ```

9. Actualiza `apps/web/package.json`. Conserva las dependencias existentes de Vue, Pinia, Router, Axios, Apollo, MSW, Vitest y Cypress del laboratorio anterior, pero ajusta el nombre y los scripts principales:

   ```json
   {
     "name": "@catalogo/web",
     "version": "1.0.0",
     "private": true,
     "type": "module",
     "scripts": {
       "dev": "vite",
       "build": "vite build",
       "preview": "vite preview",
       "lint": "eslint .",
       "test": "vitest run",
       "test:coverage": "vitest run --coverage",
       "test:e2e": "start-server-and-test \"vite preview --host 127.0.0.1 --port 4173\" http://127.0.0.1:4173 \"cypress run\""
     },
     "dependencies": {
       "@catalogo/ui": "workspace:*",
       "vue": "^3.5.13",
       "vue-router": "^4.5.0",
       "pinia": "^3.0.1"
     }
   }
   ```

10. Instala `start-server-and-test` como dependencia de desarrollo de la aplicación:

    ```bash
    pnpm --filter @catalogo/web add -D start-server-and-test@2.0.8
    ```

11. Revisa que MSW siga inicializándose exclusivamente en desarrollo. En `apps/web/src/main.ts`, la lógica debe conservar una condición equivalente a esta:

    ```ts
    async function enableMocking() {
      if (!import.meta.env.DEV) return

      const { worker } = await import('./mocks/browser')
      return worker.start({
        onUnhandledRequest: 'bypass'
      })
    }

    enableMocking().then(() => {
      createApp(App).use(router).use(pinia).mount('#app')
    })
    ```

**Salida esperada:**

La estructura inicial debe ser similar a:

```text
catalogo-platform/
├── .github/
│   └── workflows/
├── apps/
│   └── web/
├── docs/
│   ├── performance-report.md
│   └── test-report.md
├── packages/
│   ├── config/
│   └── ui/
├── package.json
└── pnpm-workspace.yaml
```

**Verificación:**

```bash
find . -maxdepth 3 -type d | sort
test -f docs/performance-report.md && echo "Informe de rendimiento encontrado"
```

Debes confirmar que `docs/performance-report.md` existe antes de continuar. Este informe es un control obligatorio para la liberación.

---

### Paso 2. Crear paquetes compartidos y establecer fronteras modulares

**Objetivo:** Crear un paquete de interfaz reutilizable y un paquete de configuraciones compartidas, evitando dependencias inversas desde paquetes hacia la aplicación.

**Instrucciones:**

1. Crea `packages/ui/package.json`:

   ```json
   {
     "name": "@catalogo/ui",
     "version": "1.0.0",
     "private": true,
     "type": "module",
     "exports": {
       ".": "./src/index.ts"
     },
     "peerDependencies": {
       "vue": "^3.5.13"
     },
     "devDependencies": {
       "vue": "^3.5.13"
     }
   }
   ```

2. Crea un componente visual reutilizable en `packages/ui/src/components/AppStatusBadge.vue`:

   ```vue
   <script setup lang="ts">
   import { computed } from 'vue'

   type Status = 'active' | 'inactive' | 'pending'

   const props = defineProps<{
     status: Status
   }>()

   const label = computed(() => {
     const labels: Record<Status, string> = {
       active: 'Activo',
       inactive: 'Inactivo',
       pending: 'Pendiente'
     }

     return labels[props.status]
   })
   </script>

   <template>
     <span class="app-status-badge" :data-status="status">
       {{ label }}
     </span>
   </template>

   <style scoped>
   .app-status-badge {
     display: inline-flex;
     border-radius: 999px;
     font-size: 0.75rem;
     font-weight: 700;
     padding: 0.25rem 0.6rem;
   }

   .app-status-badge[data-status="active"] {
     background: #dcfce7;
     color: #166534;
   }

   .app-status-badge[data-status="inactive"] {
     background: #fee2e2;
     color: #991b1b;
   }

   .app-status-badge[data-status="pending"] {
     background: #fef3c7;
     color: #92400e;
   }
   </style>
   ```

3. Crea la API pública del paquete en `packages/ui/src/index.ts`:

   ```ts
   export { default as AppStatusBadge } from './components/AppStatusBadge.vue'
   ```

4. Crea `packages/config/package.json`:

   ```json
   {
     "name": "@catalogo/config",
     "version": "1.0.0",
     "private": true,
     "type": "module",
     "exports": {
       "./eslint": "./eslint.config.js",
       "./tsconfig": "./tsconfig.base.json",
       "./vitest": "./vitest.config.ts"
     }
   }
   ```

5. Crea `packages/config/tsconfig.base.json`:

   ```json
   {
     "compilerOptions": {
       "target": "ES2020",
       "module": "ESNext",
       "moduleResolution": "Bundler",
       "strict": true,
       "isolatedModules": true,
       "esModuleInterop": true,
       "skipLibCheck": true,
       "resolveJsonModule": true,
       "noEmit": true
     }
   }
   ```

6. Crea `packages/config/eslint.config.js`:

   ```js
   import js from '@eslint/js'
   import pluginVue from 'eslint-plugin-vue'

   export default [
     js.configs.recommended,
     ...pluginVue.configs['flat/recommended'],
     {
       files: ['**/*.{ts,vue}'],
       rules: {
         'vue/multi-word-component-names': 'off'
       }
     },
     {
       ignores: ['dist/**', 'coverage/**', 'node_modules/**']
     }
   ]
   ```

7. Crea `packages/config/vitest.config.ts`:

   ```ts
   import { defineConfig } from 'vitest/config'

   export default defineConfig({
     test: {
       environment: 'jsdom',
       coverage: {
         provider: 'v8',
         reporter: ['text', 'json-summary', 'html'],
         reportsDirectory: './coverage'
       }
     }
   })
   ```

8. Asegura que el paquete web pueda resolver el paquete interno. En `apps/web/vite.config.ts`, agrega o conserva un alias hacia la API pública:

   ```ts
   import { fileURLToPath, URL } from 'node:url'
   import { defineConfig } from 'vite'
   import vue from '@vitejs/plugin-vue'

   export default defineConfig({
     plugins: [vue()],
     resolve: {
       alias: {
         '@': fileURLToPath(new URL('./src', import.meta.url)),
         '@catalogo/ui': fileURLToPath(
           new URL('../../packages/ui/src/index.ts', import.meta.url)
         )
       }
     }
   })
   ```

9. Usa el componente compartido desde una vista o componente existente de Catalogo Pro:

   ```vue
   <script setup lang="ts">
   import { AppStatusBadge } from '@catalogo/ui'
   </script>

   <template>
     <section>
       <h2>Estado del catálogo</h2>
       <AppStatusBadge status="active" />
     </section>
   </template>
   ```

**Salida esperada:**

El paquete `@catalogo/ui` expone componentes exclusivamente mediante `src/index.ts`. La aplicación `@catalogo/web` consume el paquete sin importar rutas internas de `packages/ui`.

**Verificación:**

```bash
pnpm install
pnpm --filter @catalogo/web build
```

La compilación debe finalizar con una salida similar a:

```text
✓ built in ...s
```

---

### Paso 3. Documentar arquitectura, convenciones y ownership

**Objetivo:** Dejar explícitas las reglas de organización, dependencia, responsabilidad y flujo de trabajo para evitar que el monorepo pierda modularidad.

**Instrucciones:**

1. Crea `docs/architecture.md`:

   ```md
   # Arquitectura de Catalogo Platform

   ## Estructura

   - `apps/web`: SPA desplegable Catalogo Pro.
   - `packages/ui`: componentes visuales reutilizables y sin lógica de transporte.
   - `packages/config`: configuraciones compartidas de TypeScript, ESLint y Vitest.
   - `docs`: decisiones, reportes de rendimiento, calidad y runbooks.
   - `.github/workflows`: automatización de integración continua y releases.

   ## Convenciones de nombres

   - Paquetes: `@catalogo/<responsabilidad>` en minúsculas y con guiones.
   - Componentes Vue: PascalCase, por ejemplo `CatalogFilters.vue`.
   - Composables: prefijo `use`, por ejemplo `useCatalogSearch.ts`.
   - Stores Pinia: sufijo `Store`, por ejemplo `useCatalogStore`.
   - Pruebas unitarias: `*.spec.ts`.
   - Pruebas de componentes: `*.component.spec.ts`.
   - Pruebas E2E: `cypress/e2e/*.cy.ts`.

   ## Límites de responsabilidad

   - Las aplicaciones pueden depender de paquetes compartidos.
   - Los paquetes nunca importan desde `apps/`.
   - `packages/ui` no realiza solicitudes HTTP, no accede a Router y no conoce stores de Pinia.
   - La API pública de cada paquete se expone desde `src/index.ts`.
   - No se permiten importaciones hacia rutas internas de otro paquete.

   ## Ownership

   | Área | Responsable |
   |---|---|
   | `apps/web/src/features` | Equipo de producto web |
   | `apps/web/src/mocks` | Equipo de calidad frontend |
   | `packages/ui` | Equipo de plataforma frontend |
   | `packages/config` | Equipo de plataforma frontend |
   | `.github/workflows` y Docker | Equipo de plataforma y release |
   | `docs/` | Autor del cambio y revisor técnico |

   ## Flujo Git

   - La única rama persistente de trabajo es `main`.
   - Los cambios se validan mediante pull requests efímeros cuando el repositorio remoto lo requiera.
   - Los pull requests se integran mediante squash y se eliminan después de la integración.
   - Toda publicación debe partir de un commit verificable de `main`.
   - No se publica una imagen si fallan lint, pruebas, cobertura, build, Cypress o los controles documentados de rendimiento.
   ```

2. Comprueba que no existan importaciones desde paquetes hacia la aplicación:

   ```bash
   grep -R "from ['\"]@catalogo/web" packages || true
   grep -R "from ['\"].*apps/web" packages || true
   ```

3. Añade el primer commit estructural:

   ```bash
   git add .
   git commit -m "chore: migrate catalogo pro to pnpm monorepo"
   ```

**Salida esperada:**

El archivo `docs/architecture.md` documenta estructura, convenciones, límites, ownership y flujo de ramas.

**Verificación:**

```bash
git status
git log --oneline -1
```

La salida debe indicar un árbol de trabajo limpio y un commit con el mensaje de migración.

---

### Paso 4. Implementar observabilidad cliente y versionado de despliegue

**Objetivo:** Registrar errores no controlados, duración de navegación y versión desplegada en consola estructurada y `localStorage`.

**Instrucciones:**

1. Crea `apps/web/src/shared/observability/clientObservability.ts`:

   ```ts
   import type { Router } from 'vue-router'

   const STORAGE_KEY = 'catalogo-platform:observability'
   const MAX_EVENTS = 50

   type ObservabilityEvent = {
     type: 'navigation' | 'error' | 'unhandledrejection'
     timestamp: string
     version: string
     payload: Record<string, unknown>
   }

   const version = import.meta.env.VITE_APP_VERSION || 'development'

   function persist(event: ObservabilityEvent) {
     const current = JSON.parse(
       localStorage.getItem(STORAGE_KEY) ?? '[]'
     ) as ObservabilityEvent[]

     const next = [...current, event].slice(-MAX_EVENTS)
     localStorage.setItem(STORAGE_KEY, JSON.stringify(next))
   }

   function report(
     type: ObservabilityEvent['type'],
     payload: Record<string, unknown>
   ) {
     const event: ObservabilityEvent = {
       type,
       timestamp: new Date().toISOString(),
       version,
       payload
     }

     console.info('[catalogo-observability]', event)
     persist(event)
   }

   export function installClientObservability(router: Router) {
     let navigationStartedAt = performance.now()

     router.beforeEach(() => {
       navigationStartedAt = performance.now()
       return true
     })

     router.afterEach((to, from, failure) => {
       report('navigation', {
         from: from.fullPath,
         to: to.fullPath,
         durationMs: Math.round(performance.now() - navigationStartedAt),
         status: failure ? 'failed' : 'completed'
       })
     })

     window.addEventListener('error', (event) => {
       report('error', {
         message: event.message,
         source: event.filename,
         line: event.lineno,
         column: event.colno
       })
     })

     window.addEventListener('unhandledrejection', (event) => {
       report('unhandledrejection', {
         reason: String(event.reason)
       })
     })

     report('navigation', {
       event: 'application_started',
       version
     })
   }
   ```

2. Integra el módulo en `apps/web/src/main.ts` antes de montar la aplicación:

   ```ts
   import { createApp } from 'vue'
   import { createPinia } from 'pinia'
   import App from './App.vue'
   import router from './router'
   import { installClientObservability } from './shared/observability/clientObservability'

   async function enableMocking() {
     if (!import.meta.env.DEV) return

     const { worker } = await import('./mocks/browser')
     return worker.start({
       onUnhandledRequest: 'bypass'
     })
   }

   enableMocking().then(() => {
     const app = createApp(App)
     const pinia = createPinia()

     installClientObservability(router)

     app.use(pinia)
     app.use(router)
     app.mount('#app')
   })
   ```

3. Crea `apps/web/.env.production`:

   ```env
   VITE_APP_VERSION=1.0.0
   ```

4. Añade una prueba mínima para el módulo, si tu configuración de Vitest ya soporta `jsdom`. Crea `apps/web/src/shared/observability/clientObservability.spec.ts`:

   ```ts
   import { beforeEach, describe, expect, it } from 'vitest'
   import { createMemoryHistory, createRouter } from 'vue-router'
   import { installClientObservability } from './clientObservability'

   describe('installClientObservability', () => {
     beforeEach(() => {
       localStorage.clear()
     })

     it('registra eventos de navegación en localStorage', async () => {
       const router = createRouter({
         history: createMemoryHistory(),
         routes: [
           { path: '/', component: { template: '<div />' } },
           { path: '/catalogo', component: { template: '<div />' } }
         ]
       })

       installClientObservability(router)
       await router.push('/catalogo')

       const events = JSON.parse(
         localStorage.getItem('catalogo-platform:observability') ?? '[]'
       )

       expect(events.length).toBeGreaterThan(0)
       expect(events.some((event: { type: string }) => event.type === 'navigation')).toBe(true)
     })
   })
   ```

5. Ejecuta la aplicación en el puerto obligatorio:

   ```bash
   pnpm --filter @catalogo/web dev -- --host 127.0.0.1 --port 5173
   ```

6. Abre `http://127.0.0.1:5173`, navega entre dos rutas y revisa la consola del navegador.

**Salida esperada:**

La consola del navegador debe mostrar eventos estructurados similares a:

```text
[catalogo-observability] {
  type: "navigation",
  version: "development",
  payload: {
    from: "/",
    to: "/catalogo",
    durationMs: 12,
    status: "completed"
  }
}
```

**Verificación:**

En la consola de Chrome, ejecuta:

```js
JSON.parse(localStorage.getItem('catalogo-platform:observability'))
```

Debes observar una lista de eventos con `timestamp`, `version`, `type` y `payload`.

---

### Paso 5. Crear la imagen Docker de producción con Nginx, healthcheck y caché

**Objetivo:** Empaquetar la SPA compilada en una imagen Nginx con fallback SPA, endpoint `/health`, cabeceras de caché y metadatos de versión.

**Instrucciones:**

1. Crea `nginx.conf` en la raíz del monorepo:

   ```nginx
   server {
     listen 80;
     server_name _;
     root /usr/share/nginx/html;
     index index.html;

     add_header X-Content-Type-Options "nosniff" always;
     add_header X-Frame-Options "SAMEORIGIN" always;
     add_header Referrer-Policy "strict-origin-when-cross-origin" always;

     location = /health {
       access_log off;
       default_type text/plain;
       add_header Cache-Control "no-store";
       return 200 "ok\n";
     }

     location ~* \.(?:js|css|woff2?|png|jpg|jpeg|gif|svg|ico)$ {
       try_files $uri =404;
       expires 1y;
       add_header Cache-Control "public, max-age=31536000, immutable";
     }

     location / {
       try_files $uri $uri/ /index.html;
       add_header Cache-Control "no-cache";
     }
   }
   ```

2. Crea `Dockerfile` en la raíz:

   ```dockerfile
   FROM node:22.14.0-alpine AS build

   WORKDIR /workspace

   ARG APP_VERSION=development
   ARG GIT_SHA=unknown

   ENV VITE_APP_VERSION=${APP_VERSION}
   ENV VITE_GIT_SHA=${GIT_SHA}

   RUN corepack enable && corepack prepare pnpm@9.15.4 --activate

   COPY package.json pnpm-lock.yaml pnpm-workspace.yaml ./
   COPY apps/web/package.json apps/web/package.json
   COPY packages/ui/package.json packages/ui/package.json
   COPY packages/config/package.json packages/config/package.json

   RUN pnpm install --frozen-lockfile

   COPY . .

   RUN pnpm --filter @catalogo/web build

   FROM nginx:1.27.3-alpine AS production

   ARG APP_VERSION=development
   ARG GIT_SHA=unknown

   LABEL org.opencontainers.image.title="catalogo-platform"
   LABEL org.opencontainers.image.version="${APP_VERSION}"
   LABEL org.opencontainers.image.revision="${GIT_SHA}"

   COPY nginx.conf /etc/nginx/conf.d/default.conf
   COPY --from=build /workspace/apps/web/dist /usr/share/nginx/html

   EXPOSE 80

   HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
     CMD wget -q -O - http://127.0.0.1/health || exit 1
   ```

3. Crea `.dockerignore`:

   ```dockerignore
   **/node_modules
   **/dist
   **/coverage
   .git
   .github
   cypress/videos
   cypress/screenshots
   npm-debug.log
   ```

4. Obtén la versión y SHA actuales:

   ```bash
   APP_VERSION=$(node -p "require('./apps/web/package.json').version")
   GIT_SHA=$(git rev-parse --short=12 HEAD)

   echo "$APP_VERSION"
   echo "$GIT_SHA"
   ```

5. Construye la imagen:

   ```bash
   docker build \
     --build-arg APP_VERSION="$APP_VERSION" \
     --build-arg GIT_SHA="$GIT_SHA" \
     -t "catalogo-platform:${APP_VERSION}" \
     -t "catalogo-platform:${GIT_SHA}" \
     .
   ```

6. Ejecuta el contenedor:

   ```bash
   docker run -d \
     --name catalogo-platform \
     -p 8080:80 \
     "catalogo-platform:${APP_VERSION}"
   ```

7. Espera el healthcheck y consulta su estado:

   ```bash
   docker inspect \
     --format='{{.State.Health.Status}}' \
     catalogo-platform
   ```

**Salida esperada:**

El estado de salud debe ser:

```text
healthy
```

**Verificación:**

Ejecuta:

```bash
curl -i http://127.0.0.1:8080/health
curl -I http://127.0.0.1:8080/assets/
curl -I http://127.0.0.1:8080/
docker images catalogo-platform
```

Verifica los siguientes criterios:

- `/health` responde `200 OK` con contenido `ok`.
- La ruta principal responde la SPA.
- La imagen posee al menos una etiqueta semántica y una etiqueta basada en SHA.
- Una ruta Vue, por ejemplo `/catalogo` o `/login`, responde `200` gracias al fallback SPA:

  ```bash
  curl -I http://127.0.0.1:8080/catalogo
  ```

---

### Paso 6. Crear la pipeline de integración continua

**Objetivo:** Automatizar instalación, lint, pruebas unitarias, cobertura, build y Cypress para `push` y `pull_request` sobre `main`.

**Instrucciones:**

1. Crea `.github/workflows/ci.yml`:

   ```yaml
   name: CI

   on:
     push:
       branches:
         - main
     pull_request:
       branches:
         - main

   concurrency:
     group: ci-${{ github.workflow }}-${{ github.ref }}
     cancel-in-progress: true

   jobs:
     quality:
       name: Calidad, cobertura y build
       runs-on: ubuntu-latest

       steps:
         - name: Obtener código
           uses: actions/checkout@v4

         - name: Configurar Node.js
           uses: actions/setup-node@v4
           with:
             node-version: 22.14.0
             cache: pnpm
             cache-dependency-path: pnpm-lock.yaml

         - name: Configurar pnpm
           uses: pnpm/action-setup@v4
           with:
             version: 9.15.4
             run_install: false

         - name: Instalar dependencias
           run: pnpm install --frozen-lockfile

         - name: Validar lint
           run: pnpm lint

         - name: Ejecutar pruebas unitarias y de componentes
           run: pnpm test

         - name: Generar cobertura
           run: pnpm test:coverage

         - name: Compilar SPA
           run: pnpm build

         - name: Publicar resumen de cobertura
           if: always()
           uses: actions/upload-artifact@v4
           with:
             name: coverage-report
             path: apps/web/coverage
             if-no-files-found: warn

         - name: Publicar build
           if: success()
           uses: actions/upload-artifact@v4
           with:
             name: web-dist
             path: apps/web/dist
             if-no-files-found: error

     e2e:
       name: Pruebas E2E Cypress
       runs-on: ubuntu-latest
       needs: quality

       steps:
         - name: Obtener código
           uses: actions/checkout@v4

         - name: Configurar Node.js
           uses: actions/setup-node@v4
           with:
             node-version: 22.14.0
             cache: pnpm
             cache-dependency-path: pnpm-lock.yaml

         - name: Configurar pnpm
           uses: pnpm/action-setup@v4
           with:
             version: 9.15.4
             run_install: false

         - name: Instalar dependencias
           run: pnpm install --frozen-lockfile

         - name: Compilar SPA para Cypress
           run: pnpm build

         - name: Ejecutar Cypress
           run: pnpm test:e2e

         - name: Publicar evidencias Cypress
           if: failure()
           uses: actions/upload-artifact@v4
           with:
             name: cypress-artifacts
             path: |
               apps/web/cypress/screenshots
               apps/web/cypress/videos
             if-no-files-found: ignore
   ```

2. Verifica que las pruebas E2E existan dentro de `apps/web/cypress/e2e/` o ajusta la ubicación configurada en `apps/web/cypress.config.ts`.

3. Ejecuta localmente el conjunto de controles previos a CI:

   ```bash
   pnpm lint
   pnpm test
   pnpm test:coverage
   pnpm build
   pnpm test:e2e
   ```

4. Añade y confirma los cambios:

   ```bash
   git add .
   git commit -m "ci: add quality and e2e workflow"
   ```

5. Si el repositorio remoto aún no existe, créalo con GitHub CLI:

   ```bash
   gh repo create catalogo-platform --private --source=. --remote=origin --push
   ```

   Si ya existe, configura el remoto y publica `main`:

   ```bash
   git remote add origin https://github.com/TU_USUARIO/catalogo-platform.git
   git push -u origin main
   ```

**Salida esperada:**

GitHub Actions debe iniciar dos jobs:

```text
Calidad, cobertura y build
Pruebas E2E Cypress
```

**Verificación:**

```bash
gh run list --limit 5
gh run watch
```

La ejecución más reciente debe finalizar con estado:

```text
completed success
```

---

### Paso 7. Crear la pipeline de release y el runbook de despliegue

**Objetivo:** Construir una imagen versionada con el SHA Git exacto y la versión semántica de `apps/web/package.json`, además de documentar promoción y rollback.

**Instrucciones:**

1. Crea `.github/workflows/release.yml`:

   ```yaml
   name: Release

   on:
     push:
       tags:
         - "v*.*.*"
     workflow_dispatch:

   permissions:
     contents: read
     packages: write

   jobs:
     release:
       name: Construir y publicar imagen Docker
       runs-on: ubuntu-latest

       steps:
         - name: Obtener código
           uses: actions/checkout@v4
           with:
             fetch-depth: 0

         - name: Configurar Node.js
           uses: actions/setup-node@v4
           with:
             node-version: 22.14.0

         - name: Configurar pnpm
           uses: pnpm/action-setup@v4
           with:
             version: 9.15.4
             run_install: false

         - name: Instalar dependencias
           run: pnpm install --frozen-lockfile

         - name: Ejecutar controles de release
           run: |
             pnpm lint
             pnpm test:coverage
             pnpm build

         - name: Calcular metadatos de imagen
           id: metadata
           shell: bash
           run: |
             VERSION=$(node -p "require('./apps/web/package.json').version")
             GIT_SHA=$(git rev-parse --short=12 HEAD)
             IMAGE="ghcr.io/${GITHUB_REPOSITORY_OWNER,,}/catalogo-platform"

             echo "version=${VERSION}" >> "$GITHUB_OUTPUT"
             echo "git_sha=${GIT_SHA}" >> "$GITHUB_OUTPUT"
             echo "image=${IMAGE}" >> "$GITHUB_OUTPUT"

         - name: Iniciar sesión en GitHub Container Registry
           uses: docker/login-action@v3
           with:
             registry: ghcr.io
             username: ${{ github.actor }}
             password: ${{ secrets.GITHUB_TOKEN }}

         - name: Configurar Buildx
           uses: docker/setup-buildx-action@v3

         - name: Construir y publicar imagen
           uses: docker/build-push-action@v6
           with:
             context: .
             push: true
             build-args: |
               APP_VERSION=${{ steps.metadata.outputs.version }}
               GIT_SHA=${{ steps.metadata.outputs.git_sha }}
             tags: |
               ${{ steps.metadata.outputs.image }}:${{ steps.metadata.outputs.version }}
               ${{ steps.metadata.outputs.image }}:${{ steps.metadata.outputs.git_sha }}
             labels: |
               org.opencontainers.image.version=${{ steps.metadata.outputs.version }}
               org.opencontainers.image.revision=${{ steps.metadata.outputs.git_sha }}
   ```

2. Crea `docs/deployment-runbook.md`:

   ```md
   # Runbook de despliegue de Catalogo Platform

   ## Condiciones para promover una release

   1. El commit debe existir en `main`.
   2. La pipeline CI debe estar en estado exitoso.
   3. Deben existir y estar revisados:
      - `docs/performance-report.md`
      - pruebas unitarias y de componentes
      - pruebas E2E Cypress
   4. No deben existir incidencias críticas abiertas de navegación, autenticación simulada, carga perezosa o manejo de errores.
   5. La versión de `apps/web/package.json` debe cumplir Semantic Versioning.

   ## Promoción

   1. Actualizar la versión en `apps/web/package.json`.
   2. Ejecutar localmente:
      ```bash
      pnpm lint
      pnpm test:coverage
      pnpm build
      pnpm test:e2e
      ```
   3. Confirmar el cambio:
      ```bash
      git add apps/web/package.json pnpm-lock.yaml
      git commit -m "chore(release): vX.Y.Z"
      git push origin main
      ```
   4. Crear y publicar la etiqueta:
      ```bash
      git tag -a vX.Y.Z -m "Release vX.Y.Z"
      git push origin vX.Y.Z
      ```
   5. Verificar la ejecución del workflow `Release`.
   6. Registrar la etiqueta de imagen publicada:
      - `ghcr.io/<organizacion>/catalogo-platform:X.Y.Z`
      - `ghcr.io/<organizacion>/catalogo-platform:<git-sha>`

   ## Verificación post-despliegue

   1. Comprobar salud:
      ```bash
      curl -i https://DOMINIO/health
      ```
   2. Confirmar respuesta `200` y cuerpo `ok`.
   3. Abrir la SPA y verificar:
      - carga de ruta inicial;
      - navegación directa a una ruta protegida;
      - fallback SPA tras recargar una ruta interna;
      - ausencia de errores no controlados en consola;
      - presencia de eventos `catalogo-observability`.
   4. Revisar cabeceras de assets hash:
      ```bash
      curl -I https://DOMINIO/assets/ARCHIVO-HASH.js
      ```
   5. Confirmar `Cache-Control: public, max-age=31536000, immutable`.

   ## Criterios para detener una publicación

   Detener o revertir la publicación si ocurre cualquiera de estas condiciones:

   - `/health` no devuelve `200`.
   - La SPA no carga o falla el fallback de rutas.
   - Se detectan errores no controlados repetidos en observabilidad.
   - Fallan pruebas E2E críticas de autenticación, navegación o catálogo.
   - El informe de rendimiento no cumple el presupuesto acordado.
   - La versión desplegada no corresponde con el tag Git o el SHA esperado.

   ## Rollback por imagen etiquetada

   1. Identificar la última imagen estable:
      ```bash
      docker images ghcr.io/<organizacion>/catalogo-platform
      ```
   2. Detener la versión defectuosa:
      ```bash
      docker stop catalogo-platform
      docker rm catalogo-platform
      ```
   3. Descargar y ejecutar la imagen estable anterior:
      ```bash
      docker pull ghcr.io/<organizacion>/catalogo-platform:X.Y.Z-ANTERIOR

      docker run -d \
        --name catalogo-platform \
        --restart unless-stopped \
        -p 8080:80 \
        ghcr.io/<organizacion>/catalogo-platform:X.Y.Z-ANTERIOR
      ```
   4. Validar:
      ```bash
      curl -i http://127.0.0.1:8080/health
      docker inspect --format='{{.State.Health.Status}}' catalogo-platform
      ```
   5. Registrar la causa, la versión revertida, la versión restaurada y la evidencia de verificación.
   ```

3. Crea una etiqueta semántica de prueba. Usa una versión que coincida con `apps/web/package.json`:

   ```bash
   APP_VERSION=$(node -p "require('./apps/web/package.json').version")

   git add .
   git commit -m "release: add docker workflow and deployment runbook"
   git tag -a "v${APP_VERSION}" -m "Release v${APP_VERSION}"
   git push origin main --tags
   ```

4. Consulta la ejecución de release:

   ```bash
   gh run list --workflow release.yml --limit 3
   ```

**Salida esperada:**

La pipeline de release publica dos etiquetas para la misma imagen:

```text
ghcr.io/<organizacion>/catalogo-platform:1.0.0
ghcr.io/<organizacion>/catalogo-platform:<sha-git-de-12-caracteres>
```

**Verificación:**

Comprueba que el workflow esté finalizado correctamente:

```bash
gh run watch
```

Si tienes permisos sobre GitHub Container Registry, lista paquetes publicados desde la interfaz de GitHub o con:

```bash
gh api user/packages?package_type=container
```

## Validación y pruebas

Ejecuta esta secuencia final desde la raíz del monorepo:

```bash
pnpm install --frozen-lockfile
pnpm lint
pnpm test
pnpm test:coverage
pnpm build
pnpm test:e2e
```

Después, valida la imagen local:

```bash
APP_VERSION=$(node -p "require('./apps/web/package.json').version")
GIT_SHA=$(git rev-parse --short=12 HEAD)

docker build \
  --build-arg APP_VERSION="$APP_VERSION" \
  --build-arg GIT_SHA="$GIT_SHA" \
  -t "catalogo-platform:${APP_VERSION}" \
  .

docker rm -f catalogo-platform 2>/dev/null || true

docker run -d \
  --name catalogo-platform \
  -p 8080:80 \
  "catalogo-platform:${APP_VERSION}"

sleep 12

docker inspect --format='{{.State.Health.Status}}' catalogo-platform
curl -fsS http://127.0.0.1:8080/health
curl -I http://127.0.0.1:8080/
curl -I http://127.0.0.1:8080/catalogo
```

### Lista de comprobación final

- [ ] El repositorio está ubicado en `/workspace/vue-intermedio/catalogo-platform`.
- [ ] Existe `pnpm-workspace.yaml` con `apps/*` y `packages/*`.
- [ ] La SPA reside en `apps/web`.
- [ ] Existe el paquete `@catalogo/ui` con API pública en `src/index.ts`.
- [ ] Existe el paquete `@catalogo/config`.
- [ ] `docs/architecture.md` define ownership, convenciones y límites de dependencia.
- [ ] `docs/performance-report.md` está presente.
- [ ] Las pruebas del laboratorio 07 siguen ejecutándose.
- [ ] `pnpm lint`, `pnpm test:coverage`, `pnpm build` y `pnpm test:e2e` finalizan correctamente.
- [ ] `.github/workflows/ci.yml` se ejecuta para `push` y `pull_request` sobre `main`.
- [ ] `.github/workflows/release.yml` publica etiquetas de versión y SHA.
- [ ] La imagen usa `nginx:1.27.3-alpine`.
- [ ] `/health` responde `200 OK`.
- [ ] La SPA funciona tras acceder directamente a una ruta interna.
- [ ] Los assets hash reciben caché inmutable.
- [ ] La observabilidad registra navegación, errores no controlados y versión.
- [ ] `docs/deployment-runbook.md` documenta promoción, detención y rollback.

## Solución de problemas

### Problema 1: Cypress falla en CI con error de conexión a `http://127.0.0.1:4173`

**Síntomas:**

```text
Error: Timed out waiting for: http://127.0.0.1:4173
```

o Cypress indica que no puede acceder a la aplicación.

**Causa:**

La SPA no se compiló antes de ejecutar `vite preview`, el script `test:e2e` usa un puerto distinto, o la configuración de Cypress apunta a otra URL base.

**Solución:**

1. Comprueba que el build se genera correctamente:

   ```bash
   pnpm build
   ls -la apps/web/dist
   ```

2. Verifica que `apps/web/package.json` use el puerto `4173` en `test:e2e`:

   ```json
   "test:e2e": "start-server-and-test \"vite preview --host 127.0.0.1 --port 4173\" http://127.0.0.1:4173 \"cypress run\""
   ```

3. En `apps/web/cypress.config.ts`, confirma que la URL base coincida:

   ```ts
   e2e: {
     baseUrl: 'http://127.0.0.1:4173'
   }
   ```

4. Repite la prueba local:

   ```bash
   pnpm --filter @catalogo/web test:e2e
   ```

### Problema 2: El contenedor responde en `/`, pero una ruta Vue como `/catalogo` devuelve `404`

**Síntomas:**

```bash
curl -I http://127.0.0.1:8080/catalogo
```

produce:

```text
HTTP/1.1 404 Not Found
```

**Causa:**

La configuración Nginx no incluye fallback SPA o el archivo `nginx.conf` no fue copiado a `/etc/nginx/conf.d/default.conf` durante la construcción de la imagen.

**Solución:**

1. Verifica que `nginx.conf` contenga:

   ```nginx
   location / {
     try_files $uri $uri/ /index.html;
   }
   ```

2. Comprueba que el Dockerfile copie la configuración:

   ```dockerfile
   COPY nginx.conf /etc/nginx/conf.d/default.conf
   ```

3. Reconstruye la imagen sin caché:

   ```bash
   docker build --no-cache -t catalogo-platform:local .
   ```

4. Recrea el contenedor:

   ```bash
   docker rm -f catalogo-platform
   docker run -d --name catalogo-platform -p 8080:80 catalogo-platform:local
   ```

5. Valida nuevamente:

   ```bash
   curl -I http://127.0.0.1:8080/catalogo
   ```

## Limpieza

Detén y elimina el contenedor local:

```bash
docker rm -f catalogo-platform
```

Elimina las imágenes locales creadas durante el laboratorio, si ya no las necesitas:

```bash
docker images catalogo-platform
docker rmi catalogo-platform:local 2>/dev/null || true
```

Para conservar el repositorio y liberar espacio de dependencias, elimina únicamente los directorios `node_modules`:

```bash
rm -rf node_modules apps/web/node_modules packages/ui/node_modules packages/config/node_modules
```

No elimines los siguientes artefactos, ya que forman parte del resultado del laboratorio:

```text
docs/performance-report.md
docs/architecture.md
docs/deployment-runbook.md
.github/workflows/ci.yml
.github/workflows/release.yml
Dockerfile
nginx.conf
pnpm-lock.yaml
```

## Resumen

Has convertido Catalogo Pro en una plataforma Vue escalable con un monorepo `pnpm`, una aplicación desplegable en `apps/web`, paquetes internos con fronteras explícitas y documentación de ownership. También implementaste controles de calidad automatizados, observabilidad cliente, una imagen Docker versionada con Nginx, healthcheck, caché de assets y fallback SPA.

La plataforma resultante permite promover releases con etiquetas semánticas y SHA Git, verificar el despliegue mediante `/health`, diagnosticar errores desde el navegador y volver de forma controlada a una imagen estable anterior cuando se incumplen los criterios de calidad o disponibilidad.
