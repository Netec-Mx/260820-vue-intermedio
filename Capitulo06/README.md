# 4 Practica: aplicar optimizaciones y medir mejoras en una SPA real

## Metadatos

| Campo | Valor |
|---|---|
| Duración | 46 minutos |
| Complejidad | Difícil |
| Nivel de Bloom | Aplicar |

## Descripción general

En este laboratorio se establecerá una línea base reproducible de rendimiento para la SPA **Catalogo Pro**, que contiene un catálogo de 2.000 productos, filtros reactivos, vistas de detalle, reportes y administración. Se analizarán los costes de carga y renderizado mediante Lighthouse, Chrome DevTools Performance y un analizador visual de bundles.

Posteriormente se aplicarán optimizaciones de entrega de JavaScript, carga diferida de rutas y activos, memoización de filtros, `v-memo` y virtual scrolling. Finalmente, se repetirá la medición bajo condiciones equivalentes y se documentarán los resultados en `docs/performance-report.md`.

> **Importante:** este laboratorio utiliza el repositorio específico ubicado en `/workspace/vue-intermedio/catalogo-pro`. Aunque el directorio general del curso es `/workspace/vue-intermedio-labs`, para esta práctica debe respetarse la ruta indicada por el alcance del laboratorio.

## Objetivos de aprendizaje

Al completar el laboratorio, podrá:

- [ ] Registrar una línea base de bundle, FCP, LCP, TBT y puntuación Lighthouse en una SPA Vue.
- [ ] Identificar recursos no críticos y trabajo de renderizado costoso usando Network y Performance de Chrome DevTools.
- [ ] Implementar carga perezosa de rutas y división manual de chunks con Vite y Rollup.
- [ ] Reducir el trabajo de la lista de productos mediante virtual scrolling, `v-memo` y memoización de filtros.
- [ ] Comparar y documentar métricas antes y después de la optimización con condiciones de medición equivalentes.

## Prerrequisitos

### Conocimientos requeridos

- Componentes Vue 3, `props`, eventos, `ref`, `computed`, `watch` y ciclo de vida.
- Sintaxis de `<script setup>` y Composition API.
- Configuración básica de rutas con Vue Router.
- Importaciones estáticas y dinámicas de módulos ES.
- Uso básico de Chrome DevTools, especialmente los paneles **Network**, **Performance** y **Lighthouse**.
- Comandos esenciales de Git, Node.js y pnpm.

### Acceso y estado inicial requeridos

- Repositorio inicial disponible en:

  ```text
  /workspace/vue-intermedio/catalogo-pro
  ```

  En Windows 11, use la ruta equivalente configurada por su entorno, por ejemplo:

  ```text
  C:\workspace\vue-intermedio\catalogo-pro
  ```

- Rama única de trabajo: `main`.
- Node.js `22.14.0`.
- pnpm `9.15.4`.
- Google Chrome `132.0.6834.83` o superior.
- Permisos para abrir los puertos `4173` y `5173`.
- No iniciar ningún servicio en el puerto `3000`.

## Entorno del laboratorio

### Requisitos de hardware

| Recurso | Mínimo | Recomendado |
|---|---:|---:|
| Arquitectura | x86_64 o Apple Silicon | x86_64 o Apple Silicon |
| CPU | 2 núcleos a 2.0 GHz | 4 núcleos o más |
| Memoria RAM | 8 GB | 16 GB |
| Espacio libre | 5 GB | 10 GB |
| Resolución | 1366 × 768 | 1920 × 1080 |

### Software principal

| Software | Versión esperada |
|---|---|
| Node.js | 22.14.0 |
| pnpm | 9.15.4 |
| Vue | 3.5.13 |
| Vite | 6.1.0 |
| Vue Router | 4.5.0 |
| Lighthouse | 12.3.0 |
| rollup-plugin-visualizer | 5.14.0 |
| Google Chrome | 132.0.6834.83 o superior |

### Preparación inicial

Abra una terminal y ejecute los siguientes comandos:

```bash
cd /workspace/vue-intermedio/catalogo-pro

git branch --show-current
node --version
pnpm --version

pnpm install
```

La salida esperada debe indicar la rama `main` y versiones compatibles con el laboratorio:

```text
main
v22.14.0
9.15.4
```

Cree el directorio que almacenará los artefactos de medición:

```bash
mkdir -p docs/performance-artifacts
```

En PowerShell de Windows:

```powershell
New-Item -ItemType Directory -Force docs/performance-artifacts
```

> **Condición de comparación:** use el mismo equipo, navegador, resolución, modo de incógnito, perfil de CPU y configuración de red para las mediciones baseline y optimizada. No compare una ejecución con caché caliente contra otra con caché fría.

---

## Paso a paso

### Paso 1. Inspeccionar el estado inicial de la SPA

**Objetivo:** confirmar la estructura del proyecto, las rutas existentes y el estado inicial antes de modificar el código.

**Instrucciones:**

1. Revise los archivos principales del proyecto:

   ```bash
   find src -maxdepth 3 -type f | sort
   ```

   En Windows PowerShell:

   ```powershell
   Get-ChildItem -Path src -Recurse -File | Select-Object -ExpandProperty FullName
   ```

2. Identifique el archivo de configuración de Vite:

   ```bash
   ls vite.config.*
   ```

3. Abra el proyecto en Visual Studio Code:

   ```bash
   code .
   ```

4. Localice y revise:
   - El archivo de rutas, normalmente `src/router/index.js` o `src/router/index.ts`.
   - La vista de catálogo, normalmente `src/views/CatalogView.vue`.
   - La vista de detalle, normalmente `src/views/ProductDetailView.vue`.
   - La vista de reportes, normalmente `src/views/ReportsView.vue`.
   - La vista administrativa, normalmente `src/views/AdminView.vue`.
   - El componente que renderiza cada producto, si existe, por ejemplo `src/components/ProductRow.vue`.

5. Inicie el servidor de desarrollo en el puerto obligatorio `5173`:

   ```bash
   pnpm run dev -- --host 127.0.0.1 --port 5173
   ```

6. Abra la aplicación en Chrome:

   ```text
   http://127.0.0.1:5173
   ```

7. Compruebe que:
   - El catálogo muestra aproximadamente 2.000 productos.
   - La búsqueda o filtros actualizan la lista.
   - Las rutas de detalle, reportes y administración son accesibles.
   - No hay errores visibles en la consola del navegador.

**Salida esperada:**

La SPA debe iniciar correctamente y permitir navegar por sus rutas. En la vista de catálogo, el navegador probablemente renderizará un número elevado de nodos de producto al mismo tiempo.

**Verificación:**

En Chrome DevTools, abra **Elements** y observe el árbol DOM del catálogo. Si se renderizan 2.000 filas o tarjetas simultáneamente, el escenario es adecuado para aplicar virtual scrolling.

---

### Paso 2. Generar la compilación baseline y analizar el tamaño del bundle

**Objetivo:** crear una compilación de referencia y registrar los archivos JavaScript y CSS entregados inicialmente.

**Instrucciones:**

1. Detenga el servidor de desarrollo con `Ctrl + C`.

2. Cree la compilación de producción:

   ```bash
   pnpm run build
   ```

3. Revise el tamaño de los archivos generados:

   ```bash
   find dist/assets -type f -printf '%f\t%k KB\n' | sort
   ```

   En macOS, donde `find -printf` puede no estar disponible:

   ```bash
   du -h dist/assets/* | sort -h
   ```

   En PowerShell:

   ```powershell
   Get-ChildItem dist/assets -File |
     Select-Object Name, @{Name="SizeKB"; Expression={[math]::Round($_.Length / 1KB, 2)}} |
     Sort-Object SizeKB
   ```

4. Inicie la previsualización de producción en el puerto `4173`:

   ```bash
   pnpm run preview -- --host localhost --port 4173
   ```

5. Abra una ventana de incógnito de Chrome y acceda a:

   ```text
   http://localhost:4173
   ```

6. Abra Chrome DevTools con `F12` o `Ctrl+Shift+I`.

7. En el panel **Network**:
   - Active **Disable cache**.
   - Seleccione la opción de red **Fast 3G** o el perfil definido por el instructor.
   - Recargue la página con `Ctrl+Shift+R`.
   - Ordene por la columna **Size**.
   - Identifique el JavaScript principal y compruebe si reportes, administración o librerías de visualización están incluidos en la carga inicial.

8. Registre en un archivo temporal los tamaños más relevantes:

   ```bash
   cat > docs/performance-artifacts/baseline-bundle.txt <<'EOF'
   Fecha:
   Commit:
   Archivo JavaScript inicial principal:
   Tamaño transferido:
   Tamaño sin comprimir:
   Archivos JavaScript iniciales:
   Archivos CSS iniciales:
   Observaciones:
   EOF
   ```

**Salida esperada:**

La compilación debe terminar sin errores. Es posible que el bundle inicial incluya código de vistas que no se necesitan en el catálogo, particularmente reportes, administración o dependencias de gráficos.

**Verificación:**

En **Network**, filtre por `JS`. Si al cargar `/` se descarga código asociado a `ReportsView`, `AdminView` o una librería de gráficos antes de visitar esas rutas, existe una oportunidad clara de aplicar code-splitting.

---

### Paso 3. Medir la línea base con Lighthouse

**Objetivo:** registrar métricas cuantitativas reproducibles de la versión no optimizada.

**Instrucciones:**

1. Mantenga el servidor de previsualización ejecutándose en:

   ```text
   http://localhost:4173
   ```

2. Cierre pestañas no relacionadas, extensiones que puedan interferir y aplicaciones intensivas en CPU.

3. Ejecute Lighthouse desde una segunda terminal:

   ```bash
   pnpm exec lighthouse http://localhost:4173 \
     --only-categories=performance \
     --output=html \
     --output=json \
     --output-path=docs/performance-artifacts/lighthouse-baseline \
     --chrome-flags="--headless=new --no-sandbox"
   ```

4. El comando generará, según el sistema, archivos similares a:

   ```text
   docs/performance-artifacts/lighthouse-baseline.report.html
   docs/performance-artifacts/lighthouse-baseline.report.json
   ```

5. Abra el reporte HTML en Chrome:

   ```bash
   xdg-open docs/performance-artifacts/lighthouse-baseline.report.html
   ```

   En macOS:

   ```bash
   open docs/performance-artifacts/lighthouse-baseline.report.html
   ```

   En Windows PowerShell:

   ```powershell
   Start-Process docs/performance-artifacts/lighthouse-baseline.report.html
   ```

6. Registre en `docs/performance-artifacts/baseline-metrics.txt`:
   - Puntuación de rendimiento.
   - FCP.
   - LCP.
   - TBT.
   - CLS.
   - Speed Index.
   - Oportunidades relevantes.
   - Diagnósticos relacionados con JavaScript, imágenes o trabajo del hilo principal.

   Puede crear el archivo con esta plantilla:

   ```bash
   cat > docs/performance-artifacts/baseline-metrics.txt <<'EOF'
   Escenario: carga inicial de http://localhost:4173
   Caché: deshabilitada
   Navegador: Chrome
   Perfil de red:
   Perfil de CPU:

   Performance score:
   FCP:
   LCP:
   TBT:
   CLS:
   Speed Index:

   Oportunidad dominante:
   Diagnóstico dominante:
   Hipótesis técnica:
   EOF
   ```

**Salida esperada:**

Lighthouse generará un informe HTML y otro JSON. Las métricas exactas dependerán del equipo, pero la aplicación baseline debe mostrar oportunidades relacionadas con JavaScript no utilizado, bundle inicial grande, imágenes sin optimización o trabajo excesivo del hilo principal.

**Verificación:**

La línea base queda validada cuando se conservan los dos reportes de Lighthouse y el archivo `baseline-metrics.txt` contiene las métricas de una ejecución identificable.

---

### Paso 4. Identificar renderizados repetidos con Chrome DevTools Performance

**Objetivo:** relacionar el retraso al filtrar productos con tareas largas, cálculo repetido y creación excesiva de DOM.

**Instrucciones:**

1. En Chrome, abra:

   ```text
   http://localhost:4173
   ```

2. Abra DevTools y seleccione **Performance**.

3. En la configuración de captura:
   - Active **Screenshots**.
   - Active **Memory**, si está disponible.
   - Configure CPU en **4× slowdown**.
   - Mantenga la caché deshabilitada desde Network.

4. Pulse **Record**.

5. Espere a que cargue el catálogo.

6. En el buscador, escriba una secuencia de varios caracteres, por ejemplo:

   ```text
   monitor
   ```

7. Cambie una vez el ordenamiento o un filtro adicional si la interfaz lo ofrece.

8. Detenga la grabación.

9. Analice la traza:
   - Ubique los eventos de teclado.
   - Identifique tareas superiores a 50 ms.
   - Revise bloques amarillos de JavaScript en el hilo principal.
   - Busque operaciones de estilo, layout o paint extensas.
   - Verifique si se crean o actualizan cientos o miles de nodos DOM.
   - Seleccione una tarea larga y revise la pila de llamadas.

10. Anote los hallazgos en `docs/performance-artifacts/baseline-metrics.txt`. Ejemplo:

   ```text
   Evidencia DevTools:
   - Al escribir en el filtro se observa una tarea de JavaScript de aproximadamente ___ ms.
   - La lista genera o actualiza aproximadamente ___ nodos.
   - La pila apunta a: filtro / ordenamiento / renderizado / componente.
   - Hipótesis: el coste dominante es renderizar todos los productos filtrados.
   ```

11. Si el proyecto posee Vue Devtools, inspeccione el componente de catálogo y confirme si cambios en los filtros provocan actualizaciones de filas que no han cambiado.

**Salida esperada:**

La traza debe evidenciar trabajo del hilo principal asociado a filtrar, ordenar o renderizar la lista. En una lista de 2.000 elementos, el coste de crear y reconciliar todos los nodos suele ser más significativo que el filtrado aislado.

**Verificación:**

La hipótesis debe estar basada en evidencia observable. Una formulación válida es:

> “Cada actualización de filtros recalcula resultados y vuelve a renderizar una lista extensa de productos; el hilo principal contiene tareas largas durante la interacción.”

No concluya simplemente que “Vue es lento”; identifique el componente, operación o dependencia responsable.

---

### Paso 5. Configurar el analizador visual de bundles

**Objetivo:** visualizar la composición del bundle y comprobar posteriormente que las dependencias se separan en chunks razonables.

**Instrucciones:**

1. Instale el analizador como dependencia de desarrollo:

   ```bash
   pnpm add -D rollup-plugin-visualizer@5.14.0
   ```

2. Abra `vite.config.js` o `vite.config.ts`.

3. Importe el plugin y añádalo a la configuración. Adapte la sintaxis al tipo de archivo existente.

   Ejemplo para `vite.config.js`:

   ```js
   import { defineConfig } from 'vite'
   import vue from '@vitejs/plugin-vue'
   import { visualizer } from 'rollup-plugin-visualizer'

   export default defineConfig({
     plugins: [
       vue(),
       visualizer({
         filename: 'docs/performance-artifacts/bundle-baseline.html',
         template: 'treemap',
         gzipSize: true,
         brotliSize: true,
         open: false,
       }),
     ],
   })
   ```

4. Si el archivo ya contiene plugins o configuración, integre `visualizer(...)` sin eliminar la configuración existente.

5. Ejecute una compilación:

   ```bash
   pnpm run build
   ```

6. Abra el reporte:

   ```bash
   xdg-open docs/performance-artifacts/bundle-baseline.html
   ```

   En macOS:

   ```bash
   open docs/performance-artifacts/bundle-baseline.html
   ```

   En Windows PowerShell:

   ```powershell
   Start-Process docs/performance-artifacts/bundle-baseline.html
   ```

7. Inspeccione el treemap y registre:
   - El chunk de entrada más grande.
   - Dependencias de Vue.
   - Vue Router.
   - Librerías de visualización o gráficos.
   - Código de vistas no críticas incluido en la carga inicial.

**Salida esperada:**

Se genera `docs/performance-artifacts/bundle-baseline.html` con un treemap navegable. Antes de la optimización, el chunk inicial puede contener código de reportes o administración.

**Verificación:**

El analizador queda correctamente configurado si el reporte se vuelve a generar al ejecutar `pnpm run build`.

---

### Paso 6. Aplicar carga perezosa a rutas no críticas

**Objetivo:** evitar que las vistas de detalle, reportes y administración formen parte obligatoria del JavaScript inicial del catálogo.

**Instrucciones:**

1. Abra el archivo de rutas, por ejemplo:

   ```text
   src/router/index.js
   ```

2. Identifique importaciones estáticas similares a estas:

   ```js
   import CatalogView from '@/views/CatalogView.vue'
   import ProductDetailView from '@/views/ProductDetailView.vue'
   import ReportsView from '@/views/ReportsView.vue'
   import AdminView from '@/views/AdminView.vue'
   ```

3. Mantenga estática únicamente la vista crítica de catálogo, si es la ruta inicial. Convierta las vistas no críticas a importaciones dinámicas:

   ```js
   import { createRouter, createWebHistory } from 'vue-router'
   import CatalogView from '@/views/CatalogView.vue'

   const router = createRouter({
     history: createWebHistory(),
     routes: [
       {
         path: '/',
         name: 'catalog',
         component: CatalogView,
       },
       {
         path: '/products/:id',
         name: 'product-detail',
         component: () => import('@/views/ProductDetailView.vue'),
       },
       {
         path: '/reports',
         name: 'reports',
         component: () => import('@/views/ReportsView.vue'),
       },
       {
         path: '/admin',
         name: 'admin',
         component: () => import('@/views/AdminView.vue'),
       },
     ],
   })

   export default router
   ```

4. Si existen más rutas no críticas, aplique el mismo criterio: cargue de forma inicial únicamente lo indispensable para la ruta principal.

5. Añada un estado de carga visual para cambios de ruta, si la aplicación no tiene uno. Por ejemplo, en el componente raíz que contiene `RouterView`:

   ```vue
   <template>
     <RouterView v-slot="{ Component }">
       <Suspense>
         <component :is="Component" />

         <template #fallback>
           <p role="status">Cargando vista…</p>
         </template>
       </Suspense>
     </RouterView>
   </template>
   ```

   > Si el proyecto ya utiliza una estrategia de carga o transición de rutas, respétela. No introduzca `Suspense` si genera conflictos con la arquitectura existente.

6. Compile el proyecto:

   ```bash
   pnpm run build
   ```

7. Revise `dist/assets`. Deben existir archivos separados asociados a vistas no críticas.

**Salida esperada:**

La compilación debe generar chunks independientes para detalle, reportes y administración. La ruta inicial no debe descargar esos chunks hasta que el usuario navegue a la ruta correspondiente.

**Verificación:**

1. Inicie la previsualización:

   ```bash
   pnpm run preview -- --host localhost --port 4173
   ```

2. Abra DevTools > **Network** y filtre por `JS`.

3. Recargue `/` con caché deshabilitada.

4. Confirme que el chunk de reportes no se descarga inicialmente.

5. Navegue a:

   ```text
   http://localhost:4173/reports
   ```

6. Confirme que el chunk de reportes se solicita al navegar por primera vez.

---

### Paso 7. Separar dependencias mediante `manualChunks`

**Objetivo:** mejorar la organización de la caché y evitar que dependencias de distinta frecuencia de cambio queden agrupadas sin criterio.

**Instrucciones:**

1. Abra `vite.config.js` o `vite.config.ts`.

2. Añada la configuración `build.rollupOptions.output.manualChunks`.

3. Use la siguiente configuración como base. Ajuste los nombres de paquetes de visualización según las dependencias reales del proyecto:

   ```js
   import { defineConfig } from 'vite'
   import vue from '@vitejs/plugin-vue'
   import { visualizer } from 'rollup-plugin-visualizer'

   export default defineConfig({
     plugins: [
       vue(),
       visualizer({
         filename: 'docs/performance-artifacts/bundle-optimized.html',
         template: 'treemap',
         gzipSize: true,
         brotliSize: true,
         open: false,
       }),
     ],

     build: {
       rollupOptions: {
         output: {
           manualChunks(id) {
             if (!id.includes('node_modules')) {
               return
             }

             if (id.includes('/vue/')) {
               return 'vue'
             }

             if (id.includes('vue-router')) {
               return 'router'
             }

             if (
               id.includes('chart.js') ||
               id.includes('apexcharts') ||
               id.includes('echarts') ||
               id.includes('d3')
             ) {
               return 'visualization'
             }

             return 'vendor'
           },
         },
       },
     },
   })
   ```

4. Si las dependencias de gráficos son locales o simuladas y no existen en `node_modules`, no fuerce el bloque `visualization`; las rutas dinámicas ya separarán `ReportsView` y sus importaciones transitivas.

5. Ejecute una nueva compilación:

   ```bash
   pnpm run build
   ```

6. Revise el directorio de salida:

   ```bash
   ls -lh dist/assets
   ```

7. Abra el nuevo análisis visual:

   ```bash
   xdg-open docs/performance-artifacts/bundle-optimized.html
   ```

**Salida esperada:**

La salida debe contener chunks separados con nombres similares a:

```text
vue-<hash>.js
router-<hash>.js
vendor-<hash>.js
visualization-<hash>.js
ReportsView-<hash>.js
AdminView-<hash>.js
ProductDetailView-<hash>.js
```

Los nombres exactos dependen de Vite, Rollup y la configuración del proyecto.

**Verificación:**

En el treemap optimizado, compruebe que:

- Vue no está mezclado arbitrariamente con código de reportes.
- Vue Router está separado.
- Las librerías de gráficos, si existen, no forman parte del chunk crítico de catálogo.
- Las vistas no críticas aparecen como chunks cargables bajo demanda.

> **Criterio técnico:** no se trata de maximizar el número de chunks. La división debe responder a límites funcionales, rutas de navegación y posibilidades reales de reutilización de caché.

---

### Paso 8. Optimizar imágenes con carga diferida y dimensiones explícitas

**Objetivo:** reducir solicitudes de imágenes fuera de la ventana visible y prevenir cambios de diseño acumulados.

**Instrucciones:**

1. Localice las imágenes de producto en el componente de fila o tarjeta. Un patrón inicial habitual es:

   ```vue
   <img :src="product.imageUrl" :alt="product.name" />
   ```

2. Añada carga diferida, ancho, alto y decodificación asíncrona:

   ```vue
   <img
     :src="product.imageUrl"
     :alt="product.name"
     width="96"
     height="96"
     loading="lazy"
     decoding="async"
   />
   ```

3. Mantenga dimensiones acordes con el diseño real. Si las tarjetas usan imágenes de 160 × 120 píxeles, use estas dimensiones:

   ```vue
   <img
     :src="product.imageUrl"
     :alt="product.name"
     width="160"
     height="120"
     loading="lazy"
     decoding="async"
   />
   ```

4. No aplique `loading="lazy"` a una imagen que sea el contenido LCP de la pantalla inicial, como un hero principal. En el catálogo, las imágenes de productos fuera del viewport sí son candidatas apropiadas.

5. Si se usan estilos CSS responsivos, preserve la relación de aspecto:

   ```css
   .product-image {
     width: 96px;
     height: 96px;
     object-fit: cover;
   }
   ```

6. Guarde los cambios y pruebe la vista de catálogo.

**Salida esperada:**

Las imágenes de productos que no estén visibles inicialmente deben descargarse conforme se acercan al viewport. La interfaz no debe presentar saltos visuales por falta de dimensiones reservadas.

**Verificación:**

En DevTools > **Network**:

1. Recargue el catálogo con caché deshabilitada.
2. Observe las solicitudes de imágenes.
3. Compruebe que no se descargan todas las imágenes de productos de inmediato.
4. Desplácese por la lista y confirme que aparecen solicitudes adicionales de imágenes según se necesiten.

---

### Paso 9. Implementar memoización de filtros y ordenamiento

**Objetivo:** evitar recalcular el mismo resultado filtrado y ordenado cuando los criterios no han cambiado.

**Instrucciones:**

1. Abra la vista de catálogo, por ejemplo:

   ```text
   src/views/CatalogView.vue
   ```

2. Identifique el `computed` que filtra y ordena productos. Una versión no optimizada puede tener esta estructura:

   ```js
   const filteredProducts = computed(() => {
     const term = search.value.toLowerCase()

     return products.value
       .filter((product) => product.name.toLowerCase().includes(term))
       .sort((a, b) => a.name.localeCompare(b.name))
   })
   ```

3. Declare una caché fuera del `computed`. La clave debe derivarse de todos los filtros y criterios de ordenación que afectan el resultado:

   ```js
   const filterCache = new Map()
   ```

4. Cree una clave estable. Ajuste los nombres de referencias a los que use el proyecto:

   ```js
   const filterKey = computed(() => {
     return JSON.stringify({
       search: search.value.trim().toLocaleLowerCase(),
       category: selectedCategory.value,
       status: selectedStatus.value,
       sort: sortBy.value,
       direction: sortDirection.value,
     })
   })
   ```

5. Reemplace el `computed` de resultados por una versión memoizada:

   ```js
   const filteredProducts = computed(() => {
     const key = filterKey.value
     const cached = filterCache.get(key)

     if (cached) {
       return cached
     }

     performance.mark('catalog-filter-start')

     const normalizedSearch = search.value.trim().toLocaleLowerCase()

     const result = products.value
       .filter((product) => {
         const matchesSearch =
           !normalizedSearch ||
           product.name.toLocaleLowerCase().includes(normalizedSearch)

         const matchesCategory =
           !selectedCategory.value ||
           product.category === selectedCategory.value

         const matchesStatus =
           !selectedStatus.value ||
           product.status === selectedStatus.value

         return matchesSearch && matchesCategory && matchesStatus
       })
       .toSorted((a, b) => {
         const multiplier = sortDirection.value === 'desc' ? -1 : 1

         if (sortBy.value === 'price') {
           return (a.price - b.price) * multiplier
         }

         return a.name.localeCompare(b.name) * multiplier
       })

     performance.mark('catalog-filter-end')
     performance.measure(
       'catalog-filter-duration',
       'catalog-filter-start',
       'catalog-filter-end',
     )

     filterCache.set(key, result)

     return result
   })
   ```

6. Si el navegador objetivo no soporta `toSorted`, use una copia antes de ordenar:

   ```js
   const result = [...products.value]
     .filter(/* criterio */)
     .sort(/* ordenamiento */)
   ```

7. Limpie la caché cuando cambie la fuente de productos. Esto evita conservar resultados obsoletos si se recargan los datos:

   ```js
   watch(
     () => products.value,
     () => {
       filterCache.clear()
     },
   )
   ```

8. Si la lista de productos es completamente estática durante la sesión, esta limpieza solo se ejecutará si cambia la referencia del arreglo.

9. Para evitar crecimiento ilimitado de memoria si hay muchas combinaciones de filtros, limite el tamaño de la caché:

   ```js
   const MAX_FILTER_CACHE_ENTRIES = 30

   function setCachedFilterResult(key, value) {
     if (filterCache.size >= MAX_FILTER_CACHE_ENTRIES) {
       const oldestKey = filterCache.keys().next().value
       filterCache.delete(oldestKey)
     }

     filterCache.set(key, value)
   }
   ```

10. Sustituya `filterCache.set(key, result)` por:

   ```js
   setCachedFilterResult(key, result)
   ```

**Salida esperada:**

El resultado filtrado se calcula la primera vez que se utiliza una combinación de filtros y se reutiliza cuando se vuelve a esa misma combinación.

**Verificación:**

En la consola de Chrome, ejecute:

```js
performance.getEntriesByName('catalog-filter-duration').slice(-5)
```

Debe observar mediciones para cálculos no almacenados en caché. Al volver a una combinación ya usada, no debe agregarse una nueva medida de filtrado, porque se devuelve el arreglo memoizado.

> **Nota:** la memoización reduce cálculos repetidos, pero no elimina por sí sola el coste de renderizar 2.000 nodos. El siguiente paso aborda ese coste mediante virtual scrolling.

---

### Paso 10. Implementar virtual scrolling para el catálogo

**Objetivo:** renderizar solo las filas visibles y una pequeña zona de reserva, en lugar de renderizar los 2.000 productos simultáneamente.

**Instrucciones:**

1. Cree el componente `src/components/VirtualProductList.vue`.

2. Implemente una ventana virtual con altura fija por fila. El ejemplo siguiente usa filas de `112` píxeles y un contenedor de `560` píxeles de alto:

   ```vue
   <script setup>
   import { computed, ref } from 'vue'
   import ProductRow from './ProductRow.vue'

   const props = defineProps({
     products: {
       type: Array,
       required: true,
     },
   })

   const ROW_HEIGHT = 112
   const VIEWPORT_HEIGHT = 560
   const OVERSCAN = 4

   const scrollTop = ref(0)

   const totalHeight = computed(() => {
     return props.products.length * ROW_HEIGHT
   })

   const visibleStart = computed(() => {
     return Math.max(
       0,
       Math.floor(scrollTop.value / ROW_HEIGHT) - OVERSCAN,
     )
   })

   const visibleCount = computed(() => {
     return Math.ceil(VIEWPORT_HEIGHT / ROW_HEIGHT) + OVERSCAN * 2
   })

   const visibleEnd = computed(() => {
     return Math.min(
       props.products.length,
       visibleStart.value + visibleCount.value,
     )
   })

   const visibleProducts = computed(() => {
     return props.products.slice(visibleStart.value, visibleEnd.value)
   })

   const offsetY = computed(() => {
     return visibleStart.value * ROW_HEIGHT
   })

   function handleScroll(event) {
     scrollTop.value = event.currentTarget.scrollTop
   }
   </script>

   <template>
     <section
       class="virtual-list"
       aria-label="Resultados del catálogo"
       :style="{ height: `${VIEWPORT_HEIGHT}px` }"
       @scroll="handleScroll"
     >
       <div
         class="virtual-list__spacer"
         :style="{ height: `${totalHeight}px` }"
       >
         <div
           class="virtual-list__content"
           :style="{ transform: `translateY(${offsetY}px)` }"
         >
           <ProductRow
             v-for="product in visibleProducts"
             :key="product.id"
             :product="product"
             :style="{ height: `${ROW_HEIGHT}px` }"
           />
         </div>
       </div>
     </section>
   </template>

   <style scoped>
   .virtual-list {
     overflow-y: auto;
     position: relative;
     contain: strict;
   }

   .virtual-list__spacer {
     position: relative;
   }

   .virtual-list__content {
     left: 0;
     position: absolute;
     right: 0;
     top: 0;
     will-change: transform;
   }
   </style>
   ```

3. Abra la vista de catálogo.

4. Reemplace el `v-for` que renderiza todos los productos por el componente virtual:

   ```vue
   <script setup>
   import VirtualProductList from '@/components/VirtualProductList.vue'

   // filteredProducts ya existe en la vista.
   </script>

   <template>
     <section>
       <!-- Controles de búsqueda, filtros y ordenamiento. -->

       <p aria-live="polite">
         {{ filteredProducts.length }} productos encontrados
       </p>

       <VirtualProductList :products="filteredProducts" />
     </section>
   </template>
   ```

5. Elimine o comente el `v-for` anterior para evitar renderizar dos listas simultáneamente.

6. Asegúrese de que `ProductRow` mantiene una altura fija compatible con `ROW_HEIGHT`. Ejemplo:

   ```css
   .product-row {
     box-sizing: border-box;
     min-height: 112px;
   }
   ```

7. Pruebe los siguientes escenarios:
   - Lista completa de 2.000 productos.
   - Búsqueda con pocos resultados.
   - Búsqueda sin resultados.
   - Desplazamiento rápido con rueda del ratón.
   - Navegación desde una fila hacia el detalle del producto.

**Salida esperada:**

El DOM debe contener únicamente un conjunto reducido de filas visibles, normalmente entre 10 y 25 según el tamaño del viewport y el valor de `OVERSCAN`, aunque existan 2.000 productos en memoria.

**Verificación:**

En DevTools > **Elements**, cuente aproximadamente las instancias de `.product-row`. No deben existir 2.000 nodos simultáneos.

En la interfaz, el scrollbar debe conservar una altura proporcional a la cantidad total de resultados, permitiendo desplazarse hasta el final de la lista.

> **Limitación conocida:** un virtual scroller de altura fija requiere que cada fila mantenga una altura estable. Si el contenido puede crecer por descripciones muy largas o cambios de idioma, deberá usar una estrategia de altura dinámica o truncar el contenido de la fila.

---

### Paso 11. Aplicar `v-memo` a las filas de producto

**Objetivo:** impedir actualizaciones innecesarias de filas visibles cuando sus propiedades relevantes no han cambiado.

**Instrucciones:**

1. Abra el componente de fila, por ejemplo:

   ```text
   src/components/ProductRow.vue
   ```

2. Asegúrese de que el componente recibe datos específicos y estables mediante props:

   ```vue
   <script setup>
   defineProps({
     product: {
       type: Object,
       required: true,
     },
   })
   </script>
   ```

3. Aplique `v-memo` al elemento raíz repetido. Incluya únicamente dependencias que deban provocar una actualización visual:

   ```vue
   <template>
     <article
       v-memo="[
         product.id,
         product.name,
         product.price,
         product.stock,
         product.status,
         product.imageUrl
       ]"
       class="product-row"
     >
       <img
         class="product-image"
         :src="product.imageUrl"
         :alt="product.name"
         width="96"
         height="96"
         loading="lazy"
         decoding="async"
       />

       <div class="product-row__content">
         <h2>{{ product.name }}</h2>
         <p>{{ product.description }}</p>
         <strong>{{ product.price }} €</strong>
         <span>Stock: {{ product.stock }}</span>
       </div>
     </article>
   </template>
   ```

4. Si una fila tiene estado propio visible, como selección, favoritos o edición, incluya ese valor en el arreglo de `v-memo`. Ejemplo:

   ```vue
   v-memo="[product.id, product.name, product.price, isSelected]"
   ```

5. No incluya objetos completos, arreglos mutables o referencias que cambien en cada render, porque invalidarían la memoización con frecuencia:

   ```vue
   <!-- Evitar -->
   <article v-memo="[product]">
   ```

6. Guarde los cambios y pruebe filtros, ordenamiento y desplazamiento.

**Salida esperada:**

Las filas que permanecen visibles y cuyos datos no cambian no deben actualizarse innecesariamente ante cambios reactivos ajenos a ellas.

**Verificación:**

1. Abra Vue Devtools si está disponible.
2. Cambie un filtro o escriba en la búsqueda.
3. Observe las actualizaciones de componentes.
4. Las filas que no pertenecen a la ventana visible o no cambian no deben experimentar renderizados innecesarios.

> **Uso apropiado de `v-memo`:** es una optimización dirigida. No debe aplicarse de forma indiscriminada a todos los componentes, porque puede ocultar actualizaciones que sí deberían mostrarse si el arreglo de dependencias está incompleto.

---

### Paso 12. Repetir las mediciones con la SPA optimizada

**Objetivo:** medir el impacto de los cambios bajo condiciones equivalentes a la línea base.

**Instrucciones:**

1. Ejecute la compilación final:

   ```bash
   pnpm run build
   ```

2. Confirme que se genera el reporte visual optimizado:

   ```text
   docs/performance-artifacts/bundle-optimized.html
   ```

3. Inicie la versión de producción:

   ```bash
   pnpm run preview -- --host localhost --port 4173
   ```

4. Abra una ventana de incógnito nueva.

5. Acceda a:

   ```text
   http://localhost:4173
   ```

6. En DevTools > **Network**:
   - Active **Disable cache**.
   - Use el mismo perfil de red empleado en baseline.
   - Recargue la página.
   - Verifique que los chunks de reportes, administración y detalle no se cargan inicialmente.

7. Ejecute Lighthouse con el mismo comando, pero cambie el nombre de salida:

   ```bash
   pnpm exec lighthouse http://localhost:4173 \
     --only-categories=performance \
     --output=html \
     --output=json \
     --output-path=docs/performance-artifacts/lighthouse-optimized \
     --chrome-flags="--headless=new --no-sandbox"
   ```

8. Ejecute una segunda captura de Chrome DevTools Performance:
   - CPU a `4× slowdown`.
   - Misma secuencia de búsqueda, por ejemplo `monitor`.
   - Mismo filtro u ordenamiento que en baseline.
   - Misma ventana de incógnito y caché deshabilitada.

9. Registre las métricas en:

   ```bash
   cat > docs/performance-artifacts/optimized-metrics.txt <<'EOF'
   Escenario: carga inicial de http://localhost:4173
   Caché: deshabilitada
   Navegador: Chrome
   Perfil de red:
   Perfil de CPU:

   Performance score:
   FCP:
   LCP:
   TBT:
   CLS:
   Speed Index:

   Tamaño del JS inicial:
   Chunks bajo demanda:
   Resultado de la interacción de filtrado:
   Observaciones:
   EOF
   ```

10. Compare ambos reportes visuales del bundle:
    - `bundle-baseline.html`
    - `bundle-optimized.html`

**Salida esperada:**

La versión optimizada debe mostrar, en términos generales:

- Menor JavaScript crítico para la ruta inicial.
- Chunks separados para rutas no críticas.
- Menos nodos de productos en el DOM.
- Menor trabajo de renderizado al interactuar con filtros.
- Menos solicitudes iniciales de imágenes fuera de la ventana visible.
- Mejora o mantenimiento de FCP, LCP, TBT y puntuación Lighthouse.

**Verificación:**

La optimización se considera válida si las métricas no empeoran de manera significativa y la aplicación conserva sus funcionalidades:

- El catálogo carga y filtra correctamente.
- La navegación a detalle funciona.
- Reportes y administración cargan al acceder a sus rutas.
- Las imágenes aparecen al desplazarse.
- No hay errores en consola.
- El puerto `3000` permanece sin servicios iniciados.

---

### Paso 13. Documentar la comparación de rendimiento

**Objetivo:** conservar evidencia técnica de la optimización y dejar una entrada clara para el laboratorio posterior.

**Instrucciones:**

1. Cree o actualice el archivo:

   ```text
   docs/performance-report.md
   ```

2. Use la siguiente plantilla. Reemplace los valores entre corchetes por métricas reales obtenidas en su equipo:

   ```markdown
   # Informe de rendimiento de Catalogo Pro

   ## Contexto de medición

   - Fecha: [fecha]
   - Commit evaluado: [hash de git]
   - URL medida: `http://localhost:4173`
   - Navegador: [versión de Chrome]
   - Caché: deshabilitada
   - Perfil de red: [perfil utilizado]
   - Perfil de CPU: [por ejemplo, 4× slowdown]
   - Escenario de interacción: cargar catálogo y buscar `monitor`.

   ## Métricas Lighthouse

   | Métrica | Baseline | Optimizada | Variación |
   |---|---:|---:|---:|
   | Performance score | [valor] | [valor] | [variación] |
   | FCP | [valor] | [valor] | [variación] |
   | LCP | [valor] | [valor] | [variación] |
   | TBT | [valor] | [valor] | [variación] |
   | CLS | [valor] | [valor] | [variación] |
   | Speed Index | [valor] | [valor] | [variación] |

   ## Tamaño y composición de bundles

   | Indicador | Baseline | Optimizada |
   |---|---:|---:|
   | JavaScript inicial principal | [valor] | [valor] |
   | Número de chunks iniciales | [valor] | [valor] |
   | Chunk de reportes descargado en `/` | Sí / No | Sí / No |
   | Chunk de administración descargado en `/` | Sí / No | Sí / No |
   | Dependencias de visualización en carga inicial | Sí / No | Sí / No |

   ## Hallazgos baseline

   1. [Hallazgo basado en Network.]
   2. [Hallazgo basado en Performance.]
   3. [Hallazgo basado en Lighthouse o visualizer.]

   ## Cambios aplicados

   1. Se transformaron las rutas de detalle, reportes y administración en importaciones dinámicas.
   2. Se configuraron `manualChunks` para separar Vue, Vue Router, dependencias de visualización y vendor.
   3. Se añadieron `loading="lazy"`, dimensiones explícitas y `decoding="async"` a imágenes de productos.
   4. Se implementó memoización de filtros mediante una clave derivada de filtros y ordenamiento.
   5. Se reemplazó el renderizado completo por virtual scrolling de altura fija.
   6. Se aplicó `v-memo` a las filas de producto con dependencias visuales explícitas.

   ## Resultado de la interacción

   - Baseline: [describir tareas largas, retraso percibido y renders.]
   - Optimizada: [describir reducción de nodos y comportamiento.]
   - Conclusión: [indicar qué coste dominante se redujo y qué limitaciones permanecen.]

   ## Artefactos

   - `docs/performance-artifacts/lighthouse-baseline.report.html`
   - `docs/performance-artifacts/lighthouse-optimized.report.html`
   - `docs/performance-artifacts/bundle-baseline.html`
   - `docs/performance-artifacts/bundle-optimized.html`
   - `docs/performance-artifacts/baseline-metrics.txt`
   - `docs/performance-artifacts/optimized-metrics.txt`
   ```

3. Obtenga el hash del commit actual:

   ```bash
   git rev-parse --short HEAD
   ```

4. Revise el estado de Git:

   ```bash
   git status
   ```

5. Compruebe que solo se han añadido artefactos y modificaciones necesarias para la optimización.

6. Cree un commit en la rama `main`:

   ```bash
   git add src vite.config.* package.json pnpm-lock.yaml docs
   git commit -m "perf: optimize catalog rendering and route loading"
   ```

**Salida esperada:**

Debe existir un informe `docs/performance-report.md` con métricas baseline y optimizadas, descripción de los cambios, evidencia y conclusión técnica.

**Verificación:**

Ejecute:

```bash
git status
git log -1 --oneline
```

El repositorio debe contener el commit de optimización. La SPA optimizada y documentada será la entrada obligatoria del laboratorio `07-00-01`.

---

## Validación y pruebas

Ejecute las siguientes comprobaciones finales antes de considerar completado el laboratorio.

### Validación de compilación

```bash
pnpm run build
```

**Resultado esperado:** la compilación termina sin errores y genera los chunks separados.

### Validación funcional manual

Con el servidor de previsualización activo:

```bash
pnpm run preview -- --host localhost --port 4173
```

Compruebe manualmente:

| Escenario | Resultado esperado |
|---|---|
| Carga de `/` | El catálogo aparece sin errores. |
| Búsqueda de productos | Los resultados se actualizan y la interfaz mantiene capacidad de respuesta. |
| Filtro u ordenamiento | La cantidad visible y la posición del scroll se comportan de forma coherente. |
| Desplazamiento rápido | Se renderizan filas visibles sin crear 2.000 elementos DOM. |
| Navegación a detalle | El chunk se descarga al entrar y la ruta funciona. |
| Navegación a reportes | El chunk de reportes se descarga bajo demanda. |
| Navegación a administración | El chunk de administración se descarga bajo demanda. |
| Carga de imágenes | Las imágenes fuera del viewport se solicitan progresivamente. |
| Consola de Chrome | No hay errores de JavaScript ni advertencias críticas. |

### Validación de red

En DevTools > **Network**, con caché deshabilitada:

1. Recargue `http://localhost:4173`.
2. Filtre por `JS`.
3. Confirme que las vistas no críticas no se descargan inicialmente.
4. Navegue a `/reports`.
5. Confirme que el chunk correspondiente se descarga en ese momento.
6. Verifique que no hay solicitudes hacia el puerto `3000`.

### Validación de rendimiento

Compruebe que el informe incluye:

- Lighthouse baseline y optimizado.
- Tamaño de bundles baseline y optimizado.
- Explicación de la causa dominante identificada.
- Captura o evidencia de Performance para la interacción de filtrado.
- Conclusión basada en medidas comparables, no en una única puntuación aislada.

---

## Solución de problemas

### Problema 1: Lighthouse no puede conectarse a `http://localhost:4173`

**Síntoma:** el comando de Lighthouse muestra mensajes como `ERR_CONNECTION_REFUSED`, `Unable to connect to Chrome` o no genera los reportes esperados.

**Causa:** el servidor de previsualización no está activo, el puerto `4173` está ocupado por otro proceso o Chrome está bloqueado por una instancia previa de Lighthouse.

**Solución:**

1. Compruebe si la SPA responde:

   ```bash
   curl -I http://localhost:4173
   ```

2. Si no responde, inicie la previsualización:

   ```bash
   pnpm run preview -- --host localhost --port 4173
   ```

3. Si el puerto está ocupado, identifique el proceso:

   ```bash
   lsof -i :4173
   ```

   En Windows PowerShell:

   ```powershell
   netstat -ano | Select-String ":4173"
   ```

4. Finalice únicamente el proceso que ocupa el puerto o reutilice el servidor correcto.

5. Cierre instancias de Chrome controladas por automatización y vuelva a ejecutar Lighthouse.

---

### Problema 2: El virtual scroller muestra espacios vacíos, saltos o productos superpuestos

**Síntoma:** al desplazarse, algunas filas se solapan, hay espacios vacíos o la posición de la lista parece incorrecta.

**Causa:** la altura real de `ProductRow` no coincide con la constante `ROW_HEIGHT`, o el contenido de una fila cambia dinámicamente y rompe el supuesto de altura fija.

**Solución:**

1. Inspeccione una fila en DevTools > **Elements** y mida su altura real.
2. Ajuste `ROW_HEIGHT` para que coincida exactamente con la altura renderizada, incluyendo bordes y padding:

   ```js
   const ROW_HEIGHT = 112
   ```

3. Aplique `box-sizing: border-box` a la fila:

   ```css
   .product-row {
     box-sizing: border-box;
     height: 112px;
     overflow: hidden;
   }
   ```

4. Limite textos que puedan alterar la altura, por ejemplo con truncamiento CSS.
5. Si el diseño requiere filas de altura variable, no mantenga esta implementación fija; adopte una biblioteca de virtualización compatible con tamaños dinámicos en una iteración posterior.

---

## Limpieza

1. Detenga el servidor de previsualización o desarrollo con `Ctrl + C`.

2. No elimine los siguientes elementos, ya que son evidencia y entrada para el siguiente laboratorio:

   ```text
   docs/performance-report.md
   docs/performance-artifacts/
   vite.config.js
   vite.config.ts
   src/components/VirtualProductList.vue
   ```

3. Revise el estado final del repositorio:

   ```bash
   git status
   ```

4. Si aún no creó el commit final, hágalo:

   ```bash
   git add src vite.config.* package.json pnpm-lock.yaml docs
   git commit -m "perf: optimize catalog rendering and route loading"
   ```

5. Confirme que continúa trabajando únicamente en la rama `main`:

   ```bash
   git branch --show-current
   ```

---

## Resumen

En este laboratorio se aplicó un proceso de optimización basado en evidencia:

1. Se estableció una línea base mediante compilación de producción, Lighthouse, Network, Performance y análisis de bundles.
2. Se identificó la carga inicial de rutas no críticas y el coste de renderizar una lista extensa.
3. Se aplicó code-splitting con `dynamic import()` y separación de dependencias con `manualChunks`.
4. Se redujo el coste visual usando imágenes diferidas, dimensiones explícitas, memoización de filtros, virtual scrolling y `v-memo`.
5. Se repitieron las mediciones en condiciones equivalentes y se documentó el impacto en `docs/performance-report.md`.

La optimización resultante no debe evaluarse únicamente por una puntuación Lighthouse. La evidencia principal debe demostrar que la ruta crítica entrega menos JavaScript no esencial, que la lista crea menos nodos DOM y que la interacción de filtrado reduce el trabajo innecesario del hilo principal.

### Recursos opcionales

- [Vue: Optimización de rendimiento](https://vuejs.org/guide/best-practices/performance.html)
- [Vue: `v-memo`](https://vuejs.org/api/built-in-directives.html#v-memo)
- [Vite: Build Options](https://vite.dev/config/build-options)
- [Chrome DevTools: Performance](https://developer.chrome.com/docs/devtools/performance)
- [Lighthouse](https://developer.chrome.com/docs/lighthouse/overview)
- [rollup-plugin-visualizer](https://github.com/btd/rollup-plugin-visualizer)
