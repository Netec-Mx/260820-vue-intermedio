# 4 Practica: crear una estrategia de pruebas para una aplicación Vue

## Metadatos

| Campo | Valor |
|---|---|
| Duración | 46 minutos |
| Complejidad | Difícil |
| Nivel de Bloom | Crear |

## Descripción general

En este laboratorio transformarás la aplicación optimizada **Catalogo Pro** en una aplicación verificable mediante una estrategia de calidad automatizada. Diseñarás una pirámide de pruebas, configurarás Vitest con `jsdom`, crearás pruebas unitarias y de componentes, y automatizarás flujos críticos con Cypress.

El resultado será una base de pruebas determinista y mantenible preparada para ser ejecutada posteriormente en un pipeline de CI/CD.

## Objetivos de aprendizaje

Al finalizar el laboratorio, podrás:

- [ ] Documentar una estrategia de pruebas basada en riesgos, niveles de prueba y criterios de exclusión.
- [ ] Configurar Vitest, `jsdom`, Vue Test Utils y cobertura V8 en una SPA Vue 3.
- [ ] Probar composables, utilidades y cálculos de virtualización de forma aislada.
- [ ] Validar contratos públicos de componentes mediante propiedades, eventos y estados visibles.
- [ ] Automatizar con Cypress los flujos de catálogo, filtros, navegación lazy-loaded y favoritos.

## Requisitos previos

### Conocimientos

Debes conocer o haber practicado previamente:

- Sintaxis básica de Vue 3 con Composition API.
- Uso de `ref`, `computed`, `watch` y propiedades reactivas.
- Navegación con Vue Router y rutas con carga perezosa.
- Principios básicos de aserciones, mocks, stubs y pruebas asíncronas.
- Uso de Git y la rama única `main`.

### Acceso y proyecto base

Debes contar con:

- El laboratorio `06-00-01` finalizado.
- La aplicación optimizada con lista virtualizada, rutas lazy-loaded y reporte de rendimiento.
- Acceso de lectura y escritura al directorio obligatorio:

```bash
/workspace/vue-intermedio-labs/projecthub-spa
```

En Windows, utiliza el equivalente:

```powershell
C:\workspace\vue-intermedio-labs\projecthub-spa
```

- Node.js `22.14.0`.
- npm `10.9.2` o pnpm `9.15.4`.
- Google Chrome instalado para ejecutar Cypress.
- Docker no es necesario en este laboratorio.

> **Importante:** no inicies servicios en el puerto `3000`. Los datos se interceptarán en el navegador mediante MSW durante desarrollo y mediante `cy.intercept()` durante las pruebas E2E.

## Entorno del laboratorio

### Recursos de hardware

| Recurso | Mínimo | Recomendado |
|---|---:|---:|
| CPU | 2 núcleos a 2.0 GHz | 4 núcleos |
| RAM | 8 GB | 16 GB |
| Disco libre | 5 GB | 10 GB |
| Resolución | 1366x768 | 1920x1080 |

### Software principal

| Herramienta | Versión de referencia |
|---|---|
| Node.js | 22.14.0 |
| Vue | 3.5.13 |
| Vite | 6.1.0 |
| Vitest | 3.0.5 |
| jsdom | 26.0.0 |
| Vue Test Utils | 2.4.6 |
| Cypress | 14.1.0 |
| Google Chrome | 133.0.6943.98 o superior |

### Preparación inicial

1. Abre una terminal y entra al proyecto.

```bash
cd /workspace/vue-intermedio-labs/projecthub-spa
```

2. Confirma que trabajas en la única rama permitida.

```bash
git branch --show-current
```

3. El resultado esperado es:

```text
main
```

4. Verifica las versiones instaladas.

```bash
node --version
npm --version
```

5. Instala las dependencias actuales del proyecto.

```bash
npm install
```

6. Crea un punto de control inicial.

```bash
git status
git add .
git commit -m "chore: checkpoint before test strategy"
```

> Si no hay cambios pendientes, Git indicará que no hay nada que confirmar. Continúa sin crear un commit vacío.

---

## Desarrollo paso a paso

### Paso 1. Inspeccionar la arquitectura y definir contratos verificables

**Objetivo:** identificar las unidades críticas de Catalogo Pro y establecer nombres coherentes para archivos, rutas y selectores de prueba.

**Instrucciones:**

1. Inspecciona la estructura principal del código fuente.

```bash
find src -maxdepth 3 -type f | sort
```

2. Localiza o confirma la existencia de estos elementos creados o refactorizados en el laboratorio anterior:

```text
src/components/ProductRow.vue
src/components/ProductFilters.vue
src/components/ProductListVirtualized.vue
src/composables/useProductFilters.ts
src/utils/productCache.ts
src/utils/visibleRange.ts
src/views/CatalogView.vue
src/views/ProductDetailView.vue
src/router/index.ts
```

3. Si los nombres en tu proyecto son diferentes, mantén la implementación existente y adapta únicamente los imports de las pruebas. Para el resto del laboratorio se utilizarán los siguientes contratos:

| Unidad | Contrato que se probará |
|---|---|
| `useProductFilters` | Filtra por texto, categoría y favoritos; permite reiniciar filtros. |
| `productCache` | Guarda resultados por clave y respeta tiempo de vida (`TTL`). |
| `visibleRange` | Calcula los índices visibles de una lista virtualizada. |
| `ProductRow` | Muestra un producto y emite el evento de favorito. |
| `ProductFilters` | Recibe filtros y emite cambios observables. |
| `ProductListVirtualized` | Renderiza únicamente el rango visible y conserva filas con props estables. |
| Catálogo E2E | Carga datos, filtra, abre detalle, conserva filtros y administra favoritos. |

4. Verifica que las rutas lazy-loaded utilicen importaciones dinámicas en `src/router/index.ts`.

```ts
{
  path: '/products/:id',
  name: 'product-detail',
  component: () => import('../views/ProductDetailView.vue'),
}
```

5. Asegúrate de que el listado preserve filtros en la URL, por ejemplo mediante parámetros de consulta:

```text
/products?search=mouse&category=peripherals
```

**Resultado esperado:**

La aplicación tiene una estructura identificable y las funcionalidades críticas están disponibles para ser probadas de forma aislada y de extremo a extremo.

**Verificación:**

Ejecuta la aplicación de desarrollo en el puerto obligatorio:

```bash
npm run dev -- --host 127.0.0.1 --port 5173
```

Abre:

```text
http://127.0.0.1:5173
```

Confirma manualmente que el catálogo, los filtros, los favoritos y el detalle del producto continúan funcionando.

Detén el servidor con `Ctrl+C` antes de continuar.

---

### Paso 2. Crear la estrategia de pruebas basada en riesgos

**Objetivo:** documentar qué se prueba, en qué nivel se prueba, qué riesgos se priorizan y qué escenarios se excluyen por costo o fragilidad.

**Instrucciones:**

1. Crea el directorio de documentación.

```bash
mkdir -p docs
```

2. Crea el archivo `docs/test-strategy.md`.

```bash
touch docs/test-strategy.md
```

3. Agrega el siguiente contenido.

```md
# Estrategia de pruebas de Catalogo Pro

## Propósito

Garantizar que los flujos críticos del catálogo funcionen de forma confiable, con retroalimentación rápida para la lógica de negocio y validación realista de los recorridos principales en navegador.

## Pirámide de pruebas

| Nivel | Proporción objetivo | Responsabilidad |
|---|---:|---|
| Unitarias | 60 % | Lógica de filtros, caché, cálculos de virtualización y composables. |
| Componentes | 30 % | Contratos de props, eventos, estados vacíos y renderizado visible. |
| E2E | 10 % | Flujos críticos completos en Chrome mediante Cypress. |

## Riesgos priorizados

| Riesgo | Impacto | Nivel principal | Motivo |
|---|---|---|---|
| Filtros muestran productos incorrectos | Alto | Unitario y E2E | Afecta directamente la búsqueda y descubrimiento de productos. |
| Caché devuelve resultados vencidos | Medio | Unitario | Es lógica determinista y rápida de validar. |
| Cálculo incorrecto del rango virtual | Alto | Unitario y componente | Puede ocultar productos o degradar rendimiento. |
| Favoritos no cambian de estado | Alto | Componente y E2E | Es una interacción principal del usuario. |
| Ruta de detalle no carga | Alto | E2E | Valida integración de router, lazy loading y vista. |
| Filtros se pierden al volver | Alto | E2E | Afecta continuidad de la experiencia de navegación. |

## Pruebas unitarias

Se prueban de forma aislada:

- `useProductFilters`.
- Utilidad de caché de resultados.
- Cálculo de rango visible de la lista virtualizada.
- Transformaciones puras de productos.
- Reglas de negocio sin DOM, red real ni temporizadores reales.

Las dependencias externas, como reloj, red o almacenamiento remoto, se simulan cuando afectan la determinación del resultado.

## Pruebas de componentes

Se prueban con Vue Test Utils y jsdom:

- Texto visible generado a partir de props.
- Emisión de eventos públicos.
- Estados vacíos y estados de carga visibles.
- Limitación de filas renderizadas en la lista virtualizada.
- Estabilidad de renderizado con datos y props sin cambios relevantes.

No se prueban nombres de métodos internos, estructura exacta de refs ni detalles privados de implementación.

## Pruebas E2E

Se ejecutan con Cypress sobre `vite preview` en el puerto 4173:

1. Carga inicial del catálogo.
2. Aplicación de filtro por texto.
3. Apertura de detalle lazy-loaded.
4. Regreso al catálogo conservando filtros.
5. Marcado y desmarcado de favorito.

Las respuestas HTTP se interceptan con fixtures deterministas. No se depende de una API externa ni del puerto 3000.

## Pruebas excluidas

Se excluyen deliberadamente:

- Comparaciones de píxeles o snapshots visuales extensos.
- Medición de rendimiento exacta dentro de jsdom.
- Pruebas E2E para cada combinación posible de filtros.
- Pruebas de implementación interna de Vue, Pinia o Vue Router.
- Flujos dependientes de servicios externos reales.
- Animaciones CSS y diferencias entre motores de renderizado.

Estas pruebas se excluyen por fragilidad, costo de mantenimiento o bajo valor respecto al riesgo cubierto.

## Criterios de calidad y cobertura

- La suite unitaria y de componentes debe finalizar sin fallos.
- Las pruebas E2E críticas deben pasar en Chrome.
- La cobertura mínima global objetivo es 80 % para líneas, funciones y sentencias.
- Las ramas de lógica crítica deben tener una cobertura mínima de 75 %.
- Toda corrección de defecto debe incluir una prueba de regresión.
- Los selectores E2E deben usar `data-cy`; no se usarán clases CSS ni texto como selector principal.

## Ejecución automatizada

| Momento | Comando | Propósito |
|---|---|---|
| Desarrollo | `npm run test` | Observación de pruebas unitarias y de componentes. |
| Validación local | `npm run test:run` | Ejecución única de Vitest. |
| Cobertura | `npm run test:coverage` | Generación de reporte V8. |
| E2E interactivo | `npm run cypress:open` | Depuración en interfaz Cypress. |
| E2E sin interfaz | `npm run e2e` | Ejecución reproducible sobre Vite preview. |

## Mantenimiento y depuración

- Aplicar el patrón preparar, ejecutar y verificar.
- Cada prueba debe validar un comportamiento observable.
- Usar fixtures para datos E2E repetibles.
- Restablecer mocks, almacenamiento y datos compartidos entre casos.
- Preferir `cy.intercept()` y `cy.wait('@alias')` sobre esperas fijas.
- Investigar primero el error más próximo a la causa: unidad, componente o flujo E2E.
```

**Resultado esperado:**

Existe una estrategia versionable que explica la distribución de pruebas, los riesgos cubiertos y las exclusiones deliberadas.

**Verificación:**

```bash
cat docs/test-strategy.md
```

Confirma que aparecen las secciones `Pirámide de pruebas`, `Riesgos priorizados`, `Pruebas excluidas` y `Criterios de calidad y cobertura`.

---

### Paso 3. Instalar y configurar Vitest, jsdom y cobertura V8

**Objetivo:** preparar un entorno de pruebas rápido para lógica y componentes Vue.

**Instrucciones:**

1. Instala las dependencias de desarrollo.

```bash
npm install -D vitest@3.0.5 jsdom@26.0.0 @vue/test-utils@2.4.6 @vitest/coverage-v8@3.0.5 cypress@14.1.0 start-server-and-test
```

2. Abre `vite.config.ts` y conserva la configuración existente de Vue, alias y plugins. Añade la sección `test`.

```ts
import { fileURLToPath, URL } from 'node:url'
import { defineConfig } from 'vitest/config'
import vue from '@vitejs/plugin-vue'

export default defineConfig({
  plugins: [vue()],
  resolve: {
    alias: {
      '@': fileURLToPath(new URL('./src', import.meta.url)),
    },
  },
  test: {
    globals: true,
    environment: 'jsdom',
    include: ['src/**/*.{test,spec}.{js,ts}'],
    setupFiles: ['./src/test/setupTests.ts'],
    clearMocks: true,
    coverage: {
      provider: 'v8',
      reporter: ['text', 'html', 'json-summary'],
      reportsDirectory: './coverage',
      include: [
        'src/composables/**/*.ts',
        'src/utils/**/*.ts',
        'src/components/**/*.vue',
      ],
      exclude: [
        'src/main.ts',
        'src/mocks/**',
        'src/**/*.test.ts',
        'src/test/**',
      ],
      thresholds: {
        lines: 80,
        functions: 80,
        statements: 80,
        branches: 75,
      },
    },
  },
})
```

3. Crea el archivo de preparación para pruebas.

```bash
mkdir -p src/test
touch src/test/setupTests.ts
```

4. Añade el desmontaje automático de componentes y un mock mínimo para `matchMedia`.

```ts
import { afterEach } from 'vitest'
import { enableAutoUnmount } from '@vue/test-utils'

enableAutoUnmount(afterEach)

Object.defineProperty(window, 'matchMedia', {
  writable: true,
  value: (query: string) => ({
    matches: false,
    media: query,
    onchange: null,
    addListener: () => undefined,
    removeListener: () => undefined,
    addEventListener: () => undefined,
    removeEventListener: () => undefined,
    dispatchEvent: () => false,
  }),
})
```

5. Si el proyecto utiliza TypeScript, agrega los tipos de Vitest en el archivo TypeScript que incluya `src`. Normalmente será `tsconfig.app.json`.

```json
{
  "compilerOptions": {
    "types": ["vitest/globals", "node"]
  }
}
```

6. Actualiza la sección `scripts` de `package.json`.

```json
{
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview",
    "test": "vitest",
    "test:run": "vitest run",
    "test:coverage": "vitest run --coverage",
    "cypress:open": "cypress open",
    "cypress:run": "cypress run --browser chrome",
    "e2e": "start-server-and-test \"npm run build && npm run preview -- --host 127.0.0.1 --port 4173\" http://127.0.0.1:4173 \"npm run cypress:run\""
  }
}
```

**Resultado esperado:**

Vitest puede ejecutar pruebas en entorno `jsdom`, desmontar componentes entre casos y generar reportes de cobertura V8.

**Verificación:**

Ejecuta Vitest aunque todavía no existan pruebas.

```bash
npm run test:run
```

El resultado debe finalizar sin errores de configuración. Es aceptable que indique que no encontró archivos de prueba antes de crear la primera suite.

---

### Paso 4. Crear pruebas unitarias para filtros, caché y rango virtual

**Objetivo:** validar la lógica crítica de Catalogo Pro sin montar una aplicación completa.

**Instrucciones:**

1. Asegúrate de que el composable de filtros tenga una interfaz equivalente a la siguiente. Adapta nombres, pero conserva el comportamiento.

```ts
// src/composables/useProductFilters.ts
import { computed, ref } from 'vue'

export type Product = {
  id: string
  name: string
  category: string
  isFavorite: boolean
}

export function useProductFilters(products: () => Product[]) {
  const search = ref('')
  const category = ref('all')
  const favoritesOnly = ref(false)

  const filteredProducts = computed(() => {
    const normalizedSearch = search.value.trim().toLowerCase()

    return products().filter((product) => {
      const matchesSearch =
        !normalizedSearch ||
        product.name.toLowerCase().includes(normalizedSearch)

      const matchesCategory =
        category.value === 'all' || product.category === category.value

      const matchesFavorite =
        !favoritesOnly.value || product.isFavorite

      return matchesSearch && matchesCategory && matchesFavorite
    })
  })

  function resetFilters() {
    search.value = ''
    category.value = 'all'
    favoritesOnly.value = false
  }

  return {
    search,
    category,
    favoritesOnly,
    filteredProducts,
    resetFilters,
  }
}
```

2. Crea `src/composables/useProductFilters.test.ts`.

```ts
import { describe, expect, it } from 'vitest'
import { useProductFilters } from './useProductFilters'

const products = [
  {
    id: 'p-1',
    name: 'Mouse inalámbrico',
    category: 'peripherals',
    isFavorite: true,
  },
  {
    id: 'p-2',
    name: 'Teclado mecánico',
    category: 'peripherals',
    isFavorite: false,
  },
  {
    id: 'p-3',
    name: 'Monitor 4K',
    category: 'displays',
    isFavorite: true,
  },
]

describe('useProductFilters', () => {
  it('filtra productos por texto sin distinguir mayúsculas', () => {
    const { search, filteredProducts } = useProductFilters(() => products)

    search.value = 'MOUSE'

    expect(filteredProducts.value).toEqual([products[0]])
  })

  it('combina categoría y favoritos', () => {
    const { category, favoritesOnly, filteredProducts } = useProductFilters(
      () => products,
    )

    category.value = 'peripherals'
    favoritesOnly.value = true

    expect(filteredProducts.value).toEqual([products[0]])
  })

  it('restablece todos los filtros al estado inicial', () => {
    const { search, category, favoritesOnly, resetFilters } =
      useProductFilters(() => products)

    search.value = 'monitor'
    category.value = 'displays'
    favoritesOnly.value = true

    resetFilters()

    expect(search.value).toBe('')
    expect(category.value).toBe('all')
    expect(favoritesOnly.value).toBe(false)
  })
})
```

3. Implementa o adapta una utilidad de caché con TTL.

```ts
// src/utils/productCache.ts
type CacheEntry<T> = {
  value: T
  expiresAt: number
}

const cache = new Map<string, CacheEntry<unknown>>()

export function setCachedResult<T>(
  key: string,
  value: T,
  ttlMs: number,
): void {
  cache.set(key, {
    value,
    expiresAt: Date.now() + ttlMs,
  })
}

export function getCachedResult<T>(key: string): T | null {
  const entry = cache.get(key) as CacheEntry<T> | undefined

  if (!entry) {
    return null
  }

  if (Date.now() > entry.expiresAt) {
    cache.delete(key)
    return null
  }

  return entry.value
}

export function clearProductCache(): void {
  cache.clear()
}
```

4. Crea la prueba de caché.

```ts
// src/utils/productCache.test.ts
import { afterEach, describe, expect, it, vi } from 'vitest'
import {
  clearProductCache,
  getCachedResult,
  setCachedResult,
} from './productCache'

describe('productCache', () => {
  afterEach(() => {
    clearProductCache()
    vi.useRealTimers()
  })

  it('recupera un resultado almacenado antes de que expire', () => {
    setCachedResult('products:all', [{ id: 'p-1' }], 1_000)

    expect(getCachedResult('products:all')).toEqual([{ id: 'p-1' }])
  })

  it('elimina y devuelve null cuando el resultado expiró', () => {
    vi.useFakeTimers()
    setCachedResult('products:all', [{ id: 'p-1' }], 1_000)

    vi.advanceTimersByTime(1_001)

    expect(getCachedResult('products:all')).toBeNull()
  })

  it('devuelve null para una clave inexistente', () => {
    expect(getCachedResult('unknown')).toBeNull()
  })
})
```

5. Crea o adapta el cálculo de rango visible.

```ts
// src/utils/visibleRange.ts
export type VisibleRange = {
  start: number
  end: number
}

export function calculateVisibleRange(
  scrollTop: number,
  viewportHeight: number,
  itemHeight: number,
  totalItems: number,
  overscan = 2,
): VisibleRange {
  if (totalItems === 0) {
    return { start: 0, end: 0 }
  }

  const firstVisibleIndex = Math.floor(scrollTop / itemHeight)
  const visibleItems = Math.ceil(viewportHeight / itemHeight)

  const start = Math.max(0, firstVisibleIndex - overscan)
  const end = Math.min(
    totalItems,
    firstVisibleIndex + visibleItems + overscan,
  )

  return { start, end }
}
```

6. Crea su prueba.

```ts
// src/utils/visibleRange.test.ts
import { describe, expect, it } from 'vitest'
import { calculateVisibleRange } from './visibleRange'

describe('calculateVisibleRange', () => {
  it('calcula el rango visible inicial con overscan', () => {
    const range = calculateVisibleRange(0, 300, 50, 100, 2)

    expect(range).toEqual({
      start: 0,
      end: 8,
    })
  })

  it('calcula un rango intermedio sin salir de los límites', () => {
    const range = calculateVisibleRange(500, 300, 50, 100, 2)

    expect(range).toEqual({
      start: 8,
      end: 18,
    })
  })

  it('limita el rango al total de elementos', () => {
    const range = calculateVisibleRange(4_900, 300, 50, 100, 2)

    expect(range).toEqual({
      start: 96,
      end: 100,
    })
  })

  it('devuelve un rango vacío cuando no hay productos', () => {
    expect(calculateVisibleRange(0, 300, 50, 0)).toEqual({
      start: 0,
      end: 0,
    })
  })
})
```

**Resultado esperado:**

Las reglas de filtrado, expiración de caché y cálculo de virtualización se validan sin depender de red, navegador real ni interfaz completa.

**Verificación:**

```bash
npm run test:run
```

Debes observar suites verdes equivalentes a:

```text
✓ src/composables/useProductFilters.test.ts
✓ src/utils/productCache.test.ts
✓ src/utils/visibleRange.test.ts
```

---

### Paso 5. Preparar componentes con contratos y selectores `data-cy`

**Objetivo:** hacer que los componentes críticos sean accesibles para pruebas de componentes y flujos E2E sin depender de clases CSS.

**Instrucciones:**

1. Actualiza `ProductRow.vue` para exponer información observable y emitir el evento de favorito.

```vue
<script setup lang="ts">
type Product = {
  id: string
  name: string
  category: string
  price: number
  isFavorite: boolean
}

defineProps<{
  product: Product
}>()

const emit = defineEmits<{
  toggleFavorite: [id: string]
}>()
</script>

<template>
  <article class="product-row" :data-cy="`product-row-${product.id}`">
    <RouterLink
      :to="{ name: 'product-detail', params: { id: product.id } }"
      :data-cy="`product-detail-link-${product.id}`"
    >
      {{ product.name }}
    </RouterLink>

    <span data-cy="product-category">{{ product.category }}</span>

    <button
      type="button"
      :aria-pressed="product.isFavorite"
      :data-cy="`favorite-button-${product.id}`"
      @click="emit('toggleFavorite', product.id)"
    >
      {{ product.isFavorite ? 'Quitar de favoritos' : 'Añadir a favoritos' }}
    </button>
  </article>
</template>
```

2. Añade selectores estables en `ProductFilters.vue`.

```vue
<template>
  <form data-cy="product-filters" @submit.prevent>
    <input
      :value="search"
      type="search"
      data-cy="filter-search"
      placeholder="Buscar productos"
      @input="$emit('update:search', ($event.target as HTMLInputElement).value)"
    >

    <select
      :value="category"
      data-cy="filter-category"
      @change="$emit('update:category', ($event.target as HTMLSelectElement).value)"
    >
      <option value="all">Todas las categorías</option>
      <option value="peripherals">Periféricos</option>
      <option value="displays">Pantallas</option>
    </select>

    <label>
      <input
        :checked="favoritesOnly"
        type="checkbox"
        data-cy="filter-favorites"
        @change="$emit('update:favoritesOnly', ($event.target as HTMLInputElement).checked)"
      >
      Solo favoritos
    </label>

    <button type="button" data-cy="reset-filters" @click="$emit('reset')">
      Limpiar filtros
    </button>
  </form>
</template>
```

3. En `ProductListVirtualized.vue`, conserva la limitación de renderizado y aplica `v-memo` sobre cada fila cuando los datos estables no cambien.

```vue
<template>
  <section
    ref="container"
    data-cy="virtual-product-list"
    class="virtual-product-list"
    @scroll="onScroll"
  >
    <div :style="{ height: `${totalHeight}px` }">
      <div :style="{ transform: `translateY(${offsetTop}px)` }">
        <ProductRow
          v-for="product in visibleProducts"
          :key="product.id"
          v-memo="[product.id, product.isFavorite]"
          :product="product"
          @toggle-favorite="$emit('toggleFavorite', $event)"
        />
      </div>
    </div>

    <p v-if="products.length === 0" data-cy="empty-products">
      No se encontraron productos.
    </p>
  </section>
</template>
```

4. Agrega selectores a la vista de catálogo y detalle.

```vue
<!-- CatalogView.vue -->
<template>
  <main data-cy="catalog-page">
    <h1>Catálogo de productos</h1>
    <!-- ProductFilters y ProductListVirtualized -->
  </main>
</template>
```

```vue
<!-- ProductDetailView.vue -->
<template>
  <main data-cy="product-detail-page">
    <RouterLink :to="{ name: 'catalog', query: $route.query }" data-cy="back-to-catalog">
      Volver al catálogo
    </RouterLink>

    <h1 data-cy="product-detail-name">{{ product.name }}</h1>
  </main>
</template>
```

> El enlace de regreso debe conservar `query: $route.query`, o una estrategia equivalente, para preservar filtros tras volver al catálogo.

**Resultado esperado:**

La interfaz expone selectores semánticos y estables que representan contratos de usuario, no detalles visuales internos.

**Verificación:**

Inicia la aplicación:

```bash
npm run dev -- --host 127.0.0.1 --port 5173
```

En Chrome, inspecciona un producto y confirma la existencia de atributos similares a:

```html
<button data-cy="favorite-button-p-1">
```

Detén el servidor con `Ctrl+C`.

---

### Paso 6. Crear pruebas de componentes con Vue Test Utils

**Objetivo:** comprobar propiedades, eventos, estados vacíos y comportamiento observable del listado virtualizado.

**Instrucciones:**

1. Crea `src/components/ProductRow.test.ts`.

```ts
import { mount, RouterLinkStub } from '@vue/test-utils'
import { describe, expect, it } from 'vitest'
import ProductRow from './ProductRow.vue'

const product = {
  id: 'p-1',
  name: 'Mouse inalámbrico',
  category: 'peripherals',
  price: 49.99,
  isFavorite: false,
}

describe('ProductRow', () => {
  it('muestra el nombre y la categoría recibidos', () => {
    const wrapper = mount(ProductRow, {
      props: { product },
      global: {
        stubs: {
          RouterLink: RouterLinkStub,
        },
      },
    })

    expect(wrapper.get('[data-cy="product-row-p-1"]').text()).toContain(
      'Mouse inalámbrico',
    )
    expect(wrapper.get('[data-cy="product-category"]').text()).toBe(
      'peripherals',
    )
  })

  it('emite el identificador al cambiar favorito', async () => {
    const wrapper = mount(ProductRow, {
      props: { product },
      global: {
        stubs: {
          RouterLink: RouterLinkStub,
        },
      },
    })

    await wrapper.get('[data-cy="favorite-button-p-1"]').trigger('click')

    expect(wrapper.emitted('toggleFavorite')).toEqual([['p-1']])
  })

  it('comunica el estado favorito mediante aria-pressed', () => {
    const wrapper = mount(ProductRow, {
      props: {
        product: {
          ...product,
          isFavorite: true,
        },
      },
      global: {
        stubs: {
          RouterLink: RouterLinkStub,
        },
      },
    })

    expect(
      wrapper.get('[data-cy="favorite-button-p-1"]').attributes('aria-pressed'),
    ).toBe('true')
  })
})
```

2. Crea `src/components/ProductFilters.test.ts`.

```ts
import { mount } from '@vue/test-utils'
import { describe, expect, it } from 'vitest'
import ProductFilters from './ProductFilters.vue'

describe('ProductFilters', () => {
  function createWrapper() {
    return mount(ProductFilters, {
      props: {
        search: '',
        category: 'all',
        favoritesOnly: false,
      },
    })
  }

  it('emite el nuevo texto de búsqueda', async () => {
    const wrapper = createWrapper()

    await wrapper.get('[data-cy="filter-search"]').setValue('mouse')

    expect(wrapper.emitted('update:search')).toEqual([['mouse']])
  })

  it('emite la categoría seleccionada', async () => {
    const wrapper = createWrapper()

    await wrapper
      .get('[data-cy="filter-category"]')
      .setValue('peripherals')

    expect(wrapper.emitted('update:category')).toEqual([['peripherals']])
  })

  it('emite el estado de favoritos y permite reiniciar', async () => {
    const wrapper = createWrapper()

    await wrapper.get('[data-cy="filter-favorites"]').setValue(true)
    await wrapper.get('[data-cy="reset-filters"]').trigger('click')

    expect(wrapper.emitted('update:favoritesOnly')).toEqual([[true]])
    expect(wrapper.emitted('reset')).toHaveLength(1)
  })
})
```

3. Crea `src/components/ProductListVirtualized.test.ts`. Este ejemplo asume que el componente acepta `products`, `itemHeight` y `viewportHeight`. Adapta los nombres de props si tu implementación del laboratorio anterior es distinta.

```ts
import { mount } from '@vue/test-utils'
import { describe, expect, it, vi } from 'vitest'
import ProductListVirtualized from './ProductListVirtualized.vue'

const products = Array.from({ length: 100 }, (_, index) => ({
  id: `p-${index + 1}`,
  name: `Producto ${index + 1}`,
  category: 'peripherals',
  price: index + 10,
  isFavorite: false,
}))

describe('ProductListVirtualized', () => {
  it('muestra solo las filas necesarias para el viewport inicial', () => {
    const wrapper = mount(ProductListVirtualized, {
      props: {
        products,
        itemHeight: 50,
        viewportHeight: 200,
      },
      global: {
        stubs: {
          ProductRow: {
            props: ['product'],
            template: '<article data-cy="virtual-row">{{ product.name }}</article>',
          },
        },
      },
    })

    const rows = wrapper.findAll('[data-cy="virtual-row"]')

    expect(rows.length).toBeLessThan(products.length)
    expect(rows.length).toBeGreaterThan(0)
  })

  it('muestra un estado vacío cuando no recibe productos', () => {
    const wrapper = mount(ProductListVirtualized, {
      props: {
        products: [],
        itemHeight: 50,
        viewportHeight: 200,
      },
    })

    expect(wrapper.get('[data-cy="empty-products"]').text()).toBe(
      'No se encontraron productos.',
    )
  })

  it('emite el identificador de la fila al alternar favorito', async () => {
    const wrapper = mount(ProductListVirtualized, {
      props: {
        products: products.slice(0, 5),
        itemHeight: 50,
        viewportHeight: 200,
      },
      global: {
        stubs: {
          ProductRow: {
            props: ['product'],
            emits: ['toggleFavorite'],
            template: `
              <button
                data-cy="virtual-favorite"
                @click="$emit('toggleFavorite', product.id)"
              >
                Favorite
              </button>
            `,
          },
        },
      },
    })

    await wrapper.get('[data-cy="virtual-favorite"]').trigger('click')

    expect(wrapper.emitted('toggleFavorite')).toEqual([['p-1']])
  })

  it('no vuelve a renderizar una fila memorizada ante props externas estables', async () => {
    const rowRenderSpy = vi.fn()

    const wrapper = mount(ProductListVirtualized, {
      props: {
        products: products.slice(0, 5),
        itemHeight: 50,
        viewportHeight: 200,
      },
      global: {
        stubs: {
          ProductRow: {
            props: ['product'],
            setup() {
              rowRenderSpy()
              return {}
            },
            template: '<article data-cy="virtual-row">row</article>',
          },
        },
      },
    })

    const initialCalls = rowRenderSpy.mock.calls.length

    await wrapper.setProps({
      products: products.slice(0, 5),
    })

    expect(rowRenderSpy.mock.calls.length).toBeLessThanOrEqual(initialCalls + 5)
  })
})
```

4. Interpreta la última prueba correctamente: `v-memo` no debe probarse observando una variable interna de Vue. La prueba comprueba que un cambio de contenedor con productos equivalentes no provoque un crecimiento descontrolado de renderizados de filas visibles.

**Resultado esperado:**

Las pruebas de componentes validan contratos públicos: contenido visible, eventos emitidos, accesibilidad y reducción del renderizado.

**Verificación:**

```bash
npm run test:run
```

El resultado debe incluir seis o más suites verdes, según los archivos creados.

---

### Paso 7. Configurar Cypress y datos deterministas

**Objetivo:** preparar Cypress para probar la aplicación compilada mediante `vite preview` en el puerto `4173`.

**Instrucciones:**

1. Inicializa la estructura de Cypress sin abrir la interfaz.

```bash
mkdir -p cypress/e2e cypress/fixtures cypress/support
```

2. Crea `cypress.config.ts`.

```ts
import { defineConfig } from 'cypress'

export default defineConfig({
  e2e: {
    baseUrl: 'http://127.0.0.1:4173',
    specPattern: 'cypress/e2e/**/*.cy.{js,ts}',
    supportFile: 'cypress/support/e2e.ts',
    video: false,
    screenshotOnRunFailure: true,
    viewportWidth: 1366,
    viewportHeight: 768,
  },
})
```

3. Crea `cypress/support/e2e.ts`.

```ts
beforeEach(() => {
  cy.clearLocalStorage()
})
```

4. Crea un fixture determinista en `cypress/fixtures/products.json`.

```json
[
  {
    "id": "p-1",
    "name": "Mouse inalámbrico",
    "category": "peripherals",
    "price": 49.99,
    "isFavorite": false
  },
  {
    "id": "p-2",
    "name": "Teclado mecánico",
    "category": "peripherals",
    "price": 89.99,
    "isFavorite": false
  },
  {
    "id": "p-3",
    "name": "Monitor 4K",
    "category": "displays",
    "price": 399.99,
    "isFavorite": true
  }
]
```

5. Confirma la URL usada por el repositorio para recuperar productos. Para una API REST simulada, debe apuntar al endpoint constante:

```text
http://127.0.0.1:5173/api
```

6. En las pruebas E2E interceptarás rutas con un patrón equivalente a:

```text
**/api/products*
```

> Si Catalogo Pro usa GraphQL, reemplaza la interceptación REST por `**/graphql` y responde según `req.body.operationName`. No uses una API real para estas pruebas.

**Resultado esperado:**

Cypress puede localizar los archivos de prueba y ejecutarlos contra una aplicación compilada en un puerto independiente del servidor de desarrollo.

**Verificación:**

```bash
npx cypress verify
```

Debes obtener un mensaje equivalente a:

```text
Verified Cypress!
```

---

### Paso 8. Automatizar los flujos E2E críticos

**Objetivo:** validar los recorridos de negocio principales desde la perspectiva de un usuario real.

**Instrucciones:**

1. Crea `cypress/e2e/catalog.cy.ts`.

```ts
describe('Catálogo de productos', () => {
  beforeEach(() => {
    cy.fixture('products.json').then((products) => {
      cy.intercept('GET', '**/api/products*', {
        statusCode: 200,
        body: products,
      }).as('getProducts')

      cy.intercept('GET', '**/api/products/p-*', (request) => {
        const productId = request.url.split('/').pop()
        const product = products.find(
          (item: { id: string }) => item.id === productId,
        )

        request.reply({
          statusCode: product ? 200 : 404,
          body: product,
        })
      }).as('getProduct')
    })

    cy.visit('/products')
    cy.wait('@getProducts')
    cy.get('[data-cy="catalog-page"]').should('be.visible')
  })

  it('carga el catálogo y aplica un filtro por texto', () => {
    cy.get('[data-cy="product-row-p-1"]').should('contain.text', 'Mouse inalámbrico')
    cy.get('[data-cy="product-row-p-2"]').should('contain.text', 'Teclado mecánico')

    cy.get('[data-cy="filter-search"]').type('mouse')

    cy.get('[data-cy="product-row-p-1"]').should('be.visible')
    cy.get('[data-cy="product-row-p-2"]').should('not.exist')
    cy.location('search').should('contain', 'search=mouse')
  })

  it('abre el detalle lazy-loaded y vuelve preservando el filtro', () => {
    cy.get('[data-cy="filter-search"]').type('mouse')

    cy.get('[data-cy="product-detail-link-p-1"]').click()

    cy.location('pathname').should('eq', '/products/p-1')
    cy.get('[data-cy="product-detail-page"]').should('be.visible')
    cy.get('[data-cy="product-detail-name"]').should(
      'contain.text',
      'Mouse inalámbrico',
    )

    cy.get('[data-cy="back-to-catalog"]').click()

    cy.location('pathname').should('eq', '/products')
    cy.get('[data-cy="filter-search"]').should('have.value', 'mouse')
    cy.get('[data-cy="product-row-p-1"]').should('be.visible')
    cy.get('[data-cy="product-row-p-2"]').should('not.exist')
  })

  it('marca y desmarca un producto favorito', () => {
    const favoriteButton = '[data-cy="favorite-button-p-1"]'

    cy.get(favoriteButton)
      .should('have.attr', 'aria-pressed', 'false')
      .click()
      .should('have.attr', 'aria-pressed', 'true')

    cy.get(favoriteButton)
      .click()
      .should('have.attr', 'aria-pressed', 'false')
  })
})
```

2. Ajusta la ruta inicial si tu catálogo no usa `/products`. Por ejemplo, si la ruta de catálogo es `/`, sustituye:

```ts
cy.visit('/products')
```

por:

```ts
cy.visit('/')
```

3. Confirma que el nombre de ruta utilizado en `ProductRow.vue` corresponde a la ruta real del detalle.

4. Ejecuta las pruebas E2E sin interfaz.

```bash
npm run e2e
```

5. Para depurar visualmente una prueba, ejecuta:

```bash
npm run cypress:open
```

En la interfaz de Cypress, selecciona **E2E Testing**, el navegador Chrome y el archivo `catalog.cy.ts`.

**Resultado esperado:**

Cypress construye la aplicación, inicia `vite preview` en `http://127.0.0.1:4173`, intercepta respuestas deterministas y ejecuta los tres flujos críticos sin usar esperas fijas.

**Verificación:**

La salida debe mostrar un resultado equivalente a:

```text
3 passing
```

Además, confirma que no existen comandos como los siguientes en la suite E2E:

```ts
cy.wait(1000)
cy.wait(2000)
```

Las esperas deben basarse en estado observable, alias HTTP, elementos visibles, valores de campos o cambios de URL.

---

### Paso 9. Generar cobertura y consolidar la validación local

**Objetivo:** validar que las suites están verdes, revisar cobertura y preparar el repositorio para CI/CD.

**Instrucciones:**

1. Ejecuta las pruebas unitarias y de componentes con cobertura.

```bash
npm run test:coverage
```

2. Revisa el resumen mostrado en la terminal.

3. Abre el reporte HTML local.

```bash
open coverage/index.html
```

En Ubuntu, si `open` no está disponible:

```bash
xdg-open coverage/index.html
```

En Windows PowerShell:

```powershell
start coverage\index.html
```

4. Identifica archivos con cobertura menor al objetivo. Prioriza primero:

```text
src/composables/useProductFilters.ts
src/utils/productCache.ts
src/utils/visibleRange.ts
src/components/ProductRow.vue
src/components/ProductFilters.vue
src/components/ProductListVirtualized.vue
```

5. Ejecuta la validación completa.

```bash
npm run test:run && npm run e2e
```

6. Revisa los cambios generados.

```bash
git status
```

7. Confirma los artefactos relevantes.

```bash
git add docs/test-strategy.md \
  vite.config.ts \
  package.json \
  package-lock.json \
  tsconfig.app.json \
  src \
  cypress \
  cypress.config.ts

git commit -m "test: add quality strategy and automated suites"
```

> Si usas pnpm, confirma `pnpm-lock.yaml` en lugar de `package-lock.json`.

**Resultado esperado:**

El repositorio contiene estrategia documentada, configuración de Vitest, cobertura, pruebas unitarias, pruebas de componentes, fixtures y pruebas E2E reproducibles.

**Verificación:**

Ejecuta:

```bash
git log --oneline -2
```

Debes observar un commit con un mensaje equivalente a:

```text
test: add quality strategy and automated suites
```

---

## Validación y pruebas

Completa esta lista antes de considerar finalizado el laboratorio:

| Validación | Comando o evidencia | Resultado esperado |
|---|---|---|
| Rama correcta | `git branch --show-current` | `main` |
| Estrategia documentada | `cat docs/test-strategy.md` | Pirámide, riesgos, exclusiones y mantenimiento. |
| Vitest configurado | `npm run test:run` | Suites unitarias y de componentes verdes. |
| Cobertura V8 | `npm run test:coverage` | Reporte en `coverage/` y umbrales cumplidos. |
| Prueba de filtros | Suite `useProductFilters.test.ts` | Filtra y reinicia correctamente. |
| Prueba de caché | Suite `productCache.test.ts` | Respeta TTL y claves inexistentes. |
| Prueba de virtualización | Suite `visibleRange.test.ts` | Limita rangos correctamente. |
| Pruebas de componentes | Suites de `ProductRow`, `ProductFilters` y lista virtualizada | Props, eventos y estado vacío correctos. |
| Cypress | `npm run e2e` | Tres flujos E2E aprobados. |
| Sin esperas fijas | Revisión de `cypress/e2e/catalog.cy.ts` | No usa `cy.wait(numero)`. |
| Puerto reservado | Revisión de procesos | No hay servicio iniciado en el puerto 3000. |

Para comprobar los puertos activos en sistemas Unix:

```bash
lsof -i :4173 -i :5173 -i :3000
```

Para Windows PowerShell:

```powershell
netstat -ano | findstr ":4173 :5173 :3000"
```

## Solución de problemas

### 1. Vitest muestra `ReferenceError: document is not defined` o falla al montar componentes Vue

**Síntoma:** al ejecutar `npm run test:run`, las pruebas de componentes fallan al acceder a `document`, `window` o métodos de Vue Test Utils.

**Causa:** Vitest está usando el entorno predeterminado de Node en lugar de `jsdom`, o el archivo de configuración no está siendo interpretado por Vitest.

**Solución:**

1. Confirma que `vite.config.ts` importa desde `vitest/config`.

```ts
import { defineConfig } from 'vitest/config'
```

2. Confirma que existe esta configuración:

```ts
test: {
  environment: 'jsdom',
  setupFiles: ['./src/test/setupTests.ts'],
}
```

3. Verifica que `jsdom` esté instalado.

```bash
npm ls jsdom
```

4. Reinstala dependencias si es necesario.

```bash
rm -rf node_modules package-lock.json
npm install
```

5. Ejecuta nuevamente:

```bash
npm run test:run
```

### 2. Cypress falla esperando `@getProducts` o intenta consultar una API real

**Síntoma:** Cypress muestra un error de timeout en `cy.wait('@getProducts')`, recibe una respuesta `404` o intenta conectarse a un servicio externo.

**Causa:** el patrón de `cy.intercept()` no coincide con la URL real usada por la aplicación, o la aplicación consulta GraphQL en lugar de REST.

**Solución:**

1. Inicia la aplicación en modo desarrollo.

```bash
npm run dev -- --host 127.0.0.1 --port 5173
```

2. Abre las herramientas de desarrollo del navegador y revisa la pestaña **Network** al cargar el catálogo.

3. Ajusta el patrón de Cypress a la URL observada. Para REST:

```ts
cy.intercept('GET', '**/api/products*', { fixture: 'products.json' }).as(
  'getProducts',
)
```

4. Si usa GraphQL, intercepta el endpoint correspondiente:

```ts
cy.intercept('POST', '**/graphql', {
  statusCode: 200,
  body: {
    data: {
      products: [],
    },
  },
}).as('getProducts')
```

5. Detén el servidor de desarrollo y vuelve a ejecutar la suite oficial:

```bash
npm run e2e
```

## Limpieza

1. Detén cualquier proceso de Vite o Cypress que permanezca activo con `Ctrl+C`.

2. No elimines las dependencias, fixtures, reportes ni configuración de pruebas: serán necesarios en el laboratorio `08-00-01`.

3. Si deseas eliminar únicamente artefactos regenerables de cobertura:

```bash
rm -rf coverage
```

4. Verifica el estado final del repositorio.

```bash
git status
```

El resultado esperado es:

```text
nothing to commit, working tree clean
```

5. Confirma que no se dejó ningún servicio en el puerto reservado `3000`.

```bash
lsof -i :3000
```

La salida debe estar vacía.

## Resumen

En este laboratorio creaste una estrategia de calidad escalable para Catalogo Pro. Documentaste riesgos y niveles de prueba, configuraste Vitest con `jsdom` y cobertura V8, y construiste pruebas unitarias para lógica de filtros, caché y virtualización.

También validaste contratos de componentes con Vue Test Utils y automatizaste con Cypress los flujos críticos de catálogo, filtrado, detalle lazy-loaded, preservación de filtros y favoritos. Esta base será utilizada en el siguiente laboratorio para integrar la aplicación en un monorepo y ejecutar estas validaciones dentro de un pipeline CI/CD.

### Recursos opcionales

- [Documentación de Vitest](https://vitest.dev/)
- [Vue Test Utils](https://test-utils.vuejs.org/)
- [Documentación de Cypress](https://docs.cypress.io/)
- [Cobertura V8 en Vitest](https://vitest.dev/guide/coverage.html)
- [Guía de pruebas de Vue](https://vuejs.org/guide/scaling-up/testing.html)
