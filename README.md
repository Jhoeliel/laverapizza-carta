# La Vera Pizza — Carta Digital

Carta digital interactiva para **La Vera Pizza**, pizzería delivery-only en Chimbote, Perú. Pensada para compartirse como link único por WhatsApp: el cliente navega el menú, elige tamaño/combinaciones, y termina el pedido directamente en WhatsApp con el mensaje ya armado.

No usa frameworks, build tools, ni backend. Es una aplicación de una sola página (`index.html`) en HTML + CSS + JavaScript vanilla, con imágenes como archivos estáticos separados. Se despliega como sitio estático en Cloudflare Pages, sirviendo directo desde GitHub.

---

## Versiones disponibles (ramas)

Este repositorio mantiene **dos ramas activas en paralelo**, cada una con su propio proyecto y URL en Cloudflare Pages. La rama `main` es la versión estable en producción; `carrito` es una versión experimental que coexiste sin afectar a la primera.

| Rama | URL en producción | Estado | Descripción |
|---|---|---|---|
| `main` | https://laverapizza-carta.pages.dev/ | ✅ Estable, en uso real | Carta de navegación + pedido directo. Cada producto tiene su propio botón "Pedir por WhatsApp" que abre un mensaje individual. |
| `carrito` | https://laverapizza-carta-carrito.pages.dev/ | 🧪 Experimental | Igual a `main`, pero agrega un carrito de compras: se pueden agregar varios productos, ajustar cantidades, y enviar **un solo mensaje de WhatsApp** con todo el pedido y el total. |

**Por qué dos ramas en vez de una sola con feature flag:** se decidió mantenerlas separadas para poder probar el carrito en producción real sin ningún riesgo de romper la carta que ya usan los clientes. Cuando el carrito se valide completamente, la intención es fusionarlo a `main` y retirar la versión sin carrito (o mantener ambas si se decide que tiene sentido ofrecer las dos experiencias).

Cada rama tiene su **propio proyecto de Cloudflare Pages**, ambos apuntando al mismo repositorio de GitHub pero a distinta rama de origen (ver sección de Despliegue).

---

## Arquitectura

### Stack
- **HTML/CSS/JS vanilla**, sin frameworks, sin paso de build, sin dependencias de npm.
- Un único archivo `index.html` por rama contiene todo: estilos en un `<style>` inline, markup, y lógica en un `<script>` al final del `<body>`.
- Las imágenes viven como archivos sueltos en `assets/`, referenciadas por ruta relativa (`assets/nombre.png`).
- Despliegue: **Cloudflare Pages**, conectado directamente a GitHub (auto-deploy en cada push a la rama configurada como producción). No hay paso de build: Cloudflare sirve los archivos tal cual están en el repo.

### Por qué este stack
El sitio nació como una página de una sola pantalla y creció a una experiencia de dos (y luego tres) pantallas dentro del mismo documento, usando transiciones CSS para simular navegación sin recargar la página ni necesitar un router. Dado el tamaño del proyecto y que lo mantiene una sola persona, se priorizó la simplicidad operativa (cero dependencias, cero paso de build, edición directa del HTML) sobre la escalabilidad de un framework.

### Patrón de "pantallas" (SPA sin router)
La interfaz no usa rutas de URL. En su lugar, hay 2 o 3 `<div class="screen">` (lista, detalle, y carrito en la rama `carrito`) que ocupan toda la pantalla (`position: fixed; inset: 0`) y se muestran/ocultan vía clases CSS (`.hidden-right`, `.hidden-left`) que controlan `transform: translateX(...)` y `opacity` con una transición suave. Cambiar de pantalla es JS puro: agregar/quitar esas clases en los `id`s correspondientes (`#screen-lista`, `#screen-detalle`, `#screen-carrito`).

**Advertencia de implementación importante:** el contenedor `.screen` usa `will-change: transform` para optimizar la animación. Esta propiedad crea un nuevo *positioning context* en CSS — el mismo efecto que tendría `transform` o `filter` activos. Esto significa que **cualquier elemento hijo con `position: fixed` dentro de un `.screen` no se ancla a la ventana del navegador, sino al `.screen` mismo**, que tiene scroll propio (`overflow-y: auto`). El síntoma es un botón flotante que en apariencia es `fixed` pero se desplaza junto con el contenido al hacer scroll, como si estuviera anclado a una tarjeta específica de la lista.

La solución aplicada (y que debe respetarse en futuros elementos flotantes) es que **cualquier botón o elemento que deba comportarse como un FAB real debe vivir como hermano directo de los `.screen` en el `<body>`, nunca anidado dentro de uno**. Así su `position: fixed` se resuelve correctamente contra el viewport.

### Modelo de datos: objeto `PIZZAS`
Toda la información de productos vive en un único objeto JavaScript `const PIZZAS = {...}` al inicio del `<script>`. Cada clave es un id interno (`carni`, `hawaiana`, `pan_clasico`, etc.) usado en los `onclick="abrirDetalle('id')"` de las tarjetas.

Hay dos formas de producto, distinguidas por la presencia de `tallas`:

**Producto con tallas** (pizzas individuales): tiene un array `tallas: [{ nombre, porciones, precio }, ...]` con 3 tamaños (Pequeña/Mediana/Familiar). El detalle muestra el selector de tamaño y habilita la función de mitad y mitad.

```js
carni: {
  nombre: 'Carnívora', emoji: '🥩', img: 'assets/carnivora.png',
  ingredientes: 'Mozzarella, jamón, tocino, chorizo, cabanossi y bolitas de carne.',
  tallas: [
    { nombre:'Pequeña', porciones:4, precio:29 },
    { nombre:'Mediana', porciones:6, precio:42 },
    { nombre:'Familiar', porciones:8, precio:55 }
  ]
}
```

**Producto de precio único** (`esCombo: true`): no tiene `tallas`, tiene `precio` directo. Dentro de esta categoría hay dos variantes según si trae o no una lista de contenido:

- *Combos reales* (Netflix, La Bestia): incluyen `items: ['...', '...']`, una lista de lo que trae el combo, que se renderiza como checklist en el detalle.
- *Productos simples* (las 5 variantes de Pan al Ajo): **no tienen `items`**, solo `precio`. 

```js
// Combo con lista de contenido
netflix: {
  nombre: 'Combo Netflix', emoji: '🎬', img: 'assets/combo-netflix.jpg',
  ingredientes: 'La noche perfecta para dos.',
  esCombo: true, precio: 45,
  items: ['1 Pizza Mediana mitad y mitad...', '1 Porción de Pan al Ajo', '1 Gaseosa 1.5 L']
}

// Producto simple sin lista de contenido
pan_clasico: {
  nombre: 'Pan al Ajo Clásico', emoji: '🥖', img: 'assets/pan-al-ajo.jpg',
  ingredientes: 'Pan horneado con mantequilla de ajo y hierbas.',
  esCombo: true, precio: 8
}
```

**Bug histórico a tener en cuenta:** la función `abrirDetalle()` originalmente asumía que todo producto con `esCombo: true` tenía `items` y llamaba `p.items.map(...)` sin verificar. Al agregar el Pan al Ajo (sin `items`), esto lanzaba `Cannot read properties of undefined (reading 'map')`, rompiendo el detalle para esos 5 productos. La función ahora valida `if (p.items && p.items.length)` antes de iterar, y si no hay lista, simplemente oculta el bloque de "Incluye". Cualquier producto nuevo de precio único sin checklist debe seguir el patrón de Pan al Ajo (sin `items`), y cualquier combo con lista de contenido debe seguir el patrón de Netflix/Bestia (con `items`).

### Función "Mitad y mitad"
Disponible solo para productos con `tallas` (no para combos ni Pan al Ajo). Al activar el toggle, se muestra una lista de las demás pizzas con `tallas` disponibles como segundo sabor. El precio de la combinación se calcula como el promedio redondeado hacia arriba de ambos sabores en la talla elegida:

```js
precio = Math.ceil((precioSabor1 + precioSabor2) / 2)
```

### Generación del mensaje de WhatsApp
**En `main`:** cada acción de "Pedir" (por talla, por mitad y mitad, o por combo) construye un mensaje individual con `encodeURIComponent()` y lo asigna como `href` de un link `https://wa.me/51974839429?text=...`. No hay estado persistente entre productos; cada visita al detalle sobreescribe el link.

**En `carrito`:** en vez de generar el link al instante, cada "Agregar al carrito" empuja un objeto a un array `carrito` en memoria (sin `localStorage`, se pierde al recargar la página — ver sección de Limitaciones). Al abrir la pantalla de carrito, `renderCarrito()` reconstruye el mensaje completo iterando todos los ítems, con cantidades y subtotal por línea, más el total general al final.

---

## Estructura de archivos

```
laverapizza-carta/
├── index.html          # Todo el sitio: HTML + CSS + JS (contenido distinto por rama)
├── assets/
│   ├── header.png           # Banner superior (logo + WhatsApp + teléfono + tagline)
│   ├── carnivora.png
│   ├── hawaiana.png
│   ├── vera-pizza.png
│   ├── tropical.png
│   ├── americana.png
│   ├── pepperoni.png
│   ├── combo-netflix.jpg
│   ├── combo-bestia.jpg
│   └── pan-al-ajo.jpg       # Foto genérica compartida por las 5 variantes de Pan al Ajo
└── README.md            # Este archivo
```

No hay `package.json`, no hay carpeta `src/`, no hay paso de build. Lo que está en el repo es exactamente lo que Cloudflare Pages despliega.

---

## Catálogo actual de productos

| Categoría (`data-cat`) | Productos | Tiene tallas | Tiene mitad y mitad |
|---|---|---|---|
| `favorita` | Carnívora (S/29-55), Hawaiana (S/23-42) | ✅ | ✅ |
| `especial` | Vera Pizza (S/32-61), Tropical (S/29-55) | ✅ | ✅ |
| `clasica` | Americana (S/22-40), Pepperoni (S/27-52) | ✅ | ✅ |
| `combo` | Combo Netflix (S/45), Combo La Bestia (S/67) | ❌ (precio único) | ❌ |
| `pan` | Pan al Ajo: Clásico (S/8), Americano (S/9), Hawaiano (S/9), Chorizo (S/10), Champiñones (S/10) | ❌ (precio único) | ❌ |

Los rangos de precio mostrados son Pequeña–Familiar (4/6/8 porciones respectivamente).

---

## Despliegue (cómo reproducirlo desde cero)

### Requisitos
- Cuenta de GitHub con el repositorio (o un fork).
- Cuenta de Cloudflare con acceso a Pages.
- Ninguna API key ni variable de entorno: el sitio no llama a ningún backend.

### Pasos para conectar Cloudflare Pages a una rama
1. En el dashboard de Cloudflare → **Workers & Pages** → **Create application** → pestaña **Pages** → **Connect to Git**.
2. Autorizar el acceso a GitHub si se solicita.
3. Seleccionar este repositorio.
4. Configuración del proyecto:
   - **Project name:** el que se prefiera (define el subdominio `nombre.pages.dev`).
   - **Production branch:** `main` para la versión estable, o `carrito` para la experimental.
   - **Build command:** dejar vacío (no hay paso de build).
   - **Build output directory:** dejar vacío o `/` (la raíz del repo).
5. **Save and Deploy.**

Cada push posterior a la rama configurada dispara un nuevo deploy automático.

### ⚠️ Problema conocido: desconexión silenciosa de Cloudflare Pages con GitHub
En el desarrollo de este proyecto, el proyecto de Cloudflare Pages se desconectó silenciosamente de GitHub después del primer deploy. Los pushes posteriores se veían reflejados correctamente en GitHub, pero Cloudflare seguía sirviendo el commit original, sin generar nuevos deployments ni mostrar error alguno hasta revisar manualmente la pestaña de **Deployments**, donde aparecía el aviso *"This project is disconnected from your Git account."*

**Si un cambio pusheado a GitHub no se refleja en la URL pública después de un par de minutos:**
1. Verificar en el dashboard de Cloudflare → proyecto → pestaña **Deployments**, qué commit aparece como el desplegado en "Production" y comparar su hash y fecha con el último commit real en GitHub.
2. Si hay un banner de desconexión, o el deployment de Production no avanza con los pushes nuevos, la solución más confiable encontrada fue: **eliminar el proyecto de Cloudflare Pages por completo y crear uno nuevo desde cero** repitiendo los pasos de conexión de arriba. Intentar "reconectar" desde Settings suele redirigir solo a la página genérica de la GitHub App, sin resolver el vínculo roto del proyecto específico.
3. La URL del proyecto puede mantenerse igual al recrearlo con el mismo nombre.

### Verificación de sintaxis antes de cada push
Dado que no hay build ni linter automático, cualquier edición del `<script>` debe validarse manualmente antes de subir, para evitar que un error de sintaxis (como una llave `}` faltante) rompa **todo** el JavaScript de la página silenciosamente. Ejemplo de verificación rápida con Node:

```bash
# Extraer el contenido del <script> a un archivo aparte y validarlo
python3 -c "
html = open('index.html', encoding='utf-8').read()
script = html[html.find('<script>')+8 : html.find('</script>')]
open('/tmp/check.js', 'w', encoding='utf-8').write(script)
"
node --check /tmp/check.js
```

Si el comando no imprime nada y termina con código de salida 0, la sintaxis es válida.

---

## Personalización rápida

- **Cambiar el número de WhatsApp:** buscar y reemplazar `51974839429` (todas las ocurrencias) por el nuevo número con código de país, sin signos ni espacios.
- **Agregar un producto con tallas:** copiar el patrón de cualquier pizza existente en el objeto `PIZZAS`, agregar su tarjeta correspondiente en `<div class="lista">` con `data-cat` apropiado y `onclick="abrirDetalle('nuevo_id')"`.
- **Agregar un producto de precio único:** copiar el patrón de Pan al Ajo (sin `items`) o de los combos (con `items`), según corresponda.
- **Cambiar/agregar fotos:** subir el archivo a `assets/` y referenciarlo por ruta relativa (`img: 'assets/archivo.jpg'`) tanto en el objeto `PIZZAS` como en el `<img src="...">` de la tarjeta correspondiente en la lista.
- **Nuevas categorías de filtro:** agregar un botón en `.filtros` con `onclick="filtrar('nueva_cat', this)"`, y usar ese mismo string en el `data-cat` de las tarjetas correspondientes.

---

## Limitaciones conocidas

- **Sin persistencia:** el carrito de la rama `carrito` vive en una variable JS en memoria. Si el cliente recarga la página o cierra la pestaña antes de enviar el pedido por WhatsApp, el carrito se pierde por completo. No usa `localStorage` ni ningún backend.
- **Sin inventario ni disponibilidad en tiempo real:** todos los precios y productos son estáticos, definidos directamente en el código. No hay forma de marcar un producto como "agotado" sin editar el HTML.
- **Sin analítica integrada:** no hay tracking de qué productos se ven más, cuántos carritos se abandonan, etc. (Nota: existe un proyecto relacionado, "La Vera Pizza Insights", que analiza conversaciones de WhatsApp para ese propósito, pero es independiente de esta carta digital).
- **Una sola foto para las 5 variantes de Pan al Ajo:** por ahora comparten `assets/pan-al-ajo.jpg`; si se consiguen fotos individuales, basta con crear los archivos nuevos y actualizar el campo `img` de cada entrada en `PIZZAS`.
- **Sin botón de WhatsApp visible dentro del detalle en la rama `carrito` además del de "Agregar":** una vez en el detalle de un producto, solo se puede agregar al carrito; para enviar el pedido hay que ir a la pantalla de carrito. Esto es intencional (evita mensajes duplicados/parciales) pero vale la pena tenerlo presente al testear con usuarios reales.

---

## Historial de decisiones relevantes

- El header pasó por tres iteraciones: degradado tipo CSS inspirado en un sello de seguridad ("precinto"), luego una versión con franjas naranja/negro replicando esa estética, hasta llegar a la versión actual y definitiva: la imagen oficial de marca (`assets/header.png`) usada directamente, sin reconstrucción en CSS.
- Las imágenes se migraron de base64 embebido directamente en el HTML (~3MB de archivo) a archivos estáticos en `assets/` (HTML resultante de ~36-50KB), por motivos de velocidad de carga y de edición.
- El toggle de "Mitad y mitad" se rediseñó de un control discreto tipo switch a una card grande con badge llamativo y animación pulsante, después de que pruebas de campo con clientes reales mostraran que la función pasaba desapercibida en su diseño original.
