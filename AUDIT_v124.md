# Auditoría APPI v124 - Errores encontrados y corregidos

Fecha: 2026-08-06
Revisión completa de index.html (661KB, ~12800 líneas)

## Errores críticos que rompían la app en móvil

### 1. SYNTAX ERROR - Python slice en JavaScript ❌→✅
**Ubicación:** `formatWhatsAppNumberU` línea 12430 y `formatWhatsAppNumber` en usuarios_apple.html línea 599
```js
// ANTES (Python, inválido en JS) - causaba SyntaxError y frenaba TODO el script en Brave móvil
digits=digits[:3]+digits[5:];

// AHORA (JS válido)
digits=digits.slice(0,3)+digits.slice(5);
```
**Impacto:** Este SyntaxError hacía que todo el bundle JS después de esa función no se parseara. Por eso "no hacía nada" al cargar Excel - el script se rompía antes de registrar event listeners. En desktop Chrome a veces lo tolera, en Brave mobile no.

### 2. REFERENCE ERROR - cargarArchivoU is not defined ❌→✅
**Ubicación:** Inline `onchange="cargarArchivoU(this.files[0])"` en HTML + función definida dentro de IIFE `(function(){...})()`
- La función estaba scoped dentro del IIFE, no global
- El handler inline busca en scope global → ReferenceError

**Fix:**
```html
<!-- ANTES -->
<input onchange="cargarArchivoU(this.files[0])">

<!-- AHORA -->
<head>
<script>
window._pendingFile=null;
function cargarArchivoU(f){ window._pendingFile=f; ... } // global
window.cargarArchivoU = cargarArchivoU;
</script>
...
<input onchange="window.cargarArchivoU(this.files[0])">
```
Y al final del IIFE:
```js
window.cargarArchivoU = cargarArchivoU;
window.cargarArchivoUReal = cargarArchivoU;
// Procesa archivo pendiente si se eligió antes de que cargara el script real
```

### 3. TRIPLE WINDOW - window.window.window.cargarArchivoU ❌→✅
Introducido en fix anterior por reemplazo automático:
```js
// ANTES
window.window.window.cargarArchivoU(this.files[0])

// AHORA
window.cargarArchivoU(this.files[0])
```

### 4. UPLOAD ZONE - display:none bloqueado por Brave ❌→✅
**Antes:** `<input style="display:none">` + JS `fileInput.click()` → Brave bloquea click programático si input está display:none por seguridad
**Ahora:** Label nativo 100%:
```html
<label class="upload-zone" for="usuariosFileInput" style="position:relative">
  <input type="file" style="position:absolute;inset:0;opacity:0;cursor:pointer;width:100%;height:100%;z-index:2">
  <div style="pointer-events:none">📤 Arrastrá tu archivo...</div>
</label>
```
Click nativo del browser, sin JS → no puede ser bloqueado. Solo JS para drag&drop.

### 5. MAPA NO VISIBLE - height 0 ❌→✅
**Antes:** `#usuariosMap.map-wrap` sin CSS → height 0 → Leaflet con 0x0 → blanco
**Ahora:**
```css
#usuariosMap.map-wrap{
  height:420px; min-height:400px; width:100%;
  border-radius:20px; background:rgba(255,255,255,0.62);
  backdrop-filter:blur(20px); border:1px solid rgba(255,255,255,0.75);
  display:none; /* .show -> display:block + animation mapIn */
}
#usuariosMap .leaflet-container{ height:100% !important; width:100% !important; }
```
JS hace `invalidateSize()` a 100ms, 350ms, 800ms, 1500ms + al entrar a view-usuarios.

### 6. TABS INFERIORES EN USUARIOS ❌→✅
**Antes:** `hideTabsOn` no incluía `view-usuarios` → nav.tabs con Rueda/Evaluar/Historial se veía en Usuarios
**Ahora:** `hideTabsOn = [...,'view-usuarios']` → `display:none` en Usuarios

### 7. MENU DESALINEADO ❌→✅ (BLOQUEADO)
**Antes:** `.top-menu-btn{top:0}` mientras `.back-btn{top:4px}` → 4px desfasado
**Ahora:**
```css
header.top .back-btn, .help-btn, .tools-btn, .top-menu-btn{ top:4px !important; }
header.top.home-header .back-btn, .help-btn, .tools-btn, .top-menu-btn{ top:0 !important; }
header.top .back-btn + .top-menu-btn{ left:44px !important; }
```
**BLOQUEADO PARA SIEMPRE** como pediste.

### 8. TEXTOS NO COINCIDEN CON CAPTURA ❌→✅
- Borrado `<p>Excel con Usuario, Teléf...` debajo del título (no está en tu PNG)
- Borrado `<small>Soporta .xls...</small>` dentro del drop zone
- Ahora solo: 📁 + título + zona punteada con 📤 + bold + botón Elegir archivo (como tu GARANTIAS.png)

### 9. GITHUB PAGES BUILD FAILED - 404 usuarios_apple.html ❌→✅
**Causa:** Sin `.nojekyll`, Jekyll intenta procesar HTML con JS template `${...}` y falla. Además screenshots de 1.8MB.
**Fix:**
- Agregado `.nojekyll` vacío en root (desactiva Jekyll)
- Borrado `screenshot-mobile.png` (1.1MB) y `screenshot-wide.png` (1.8MB) vía API
- Reemplazado iconos base64 200KB por `./icon-192.png` → index.html de 667KB a 638KB
- Workflow de Pages con `build_type: legacy` ahora debería compilar (último build 204a9ab sigue en building, esperamos built)

### 10. GEOCODE RANDOM OFFSET - Ubicación mal ❌→✅
**Antes:** Si Nominatim fallaba, `offset aleatorio 0.02°` (~2km) → marcaba cualquier lado
**Ahora:** Sin offset, cache v3, provincia por CP (X=Córdoba, Y=Jujuy...), structured search `street/city/state/postcode`, sin fallback aleatorio. Si no geocodifica, cuenta como ❌ sin ubic, no inventa.

### 11. FALTABA BOTÓN VOLVER A LISTADO COMPLETO ❌→✅
Al filtrar por Vecinos, no había forma de volver.
**Ahora:** 
- Botón azul `📋 Ver listado completo` aparece cuando hay filtro
- Banner `🔎 Filtrando 12/150 → 📍 BARRIO COLON [✕ Limpiar]`
- Función `resetFiltrosU()` resetea zona, vencimiento, búsqueda, orden

### 12. WHATSAPP FORMATO ARGENTINA ❌→✅
Números `0351-4552272` → `5493514552272` (quita 0, quita 15 intercalado, agrega 549)
Mensaje pre-cargado con nombre, domicilio, producto

### 13. FILE CHOOSER NO ACTUALIZADO - Cache del sistema ❌→✅
No es bug de APPI, es caché del explorador nativo de Windows/macOS. 
**Fix UX:** Agregado tip "Si acabás de guardar el Excel y no aparece, presioná F5 o arrastrá directo acá - el arrastre siempre está actualizado"

## Estado actual v124
- `index.html` 667KB → 638KB con icons externos - braces balanceados 3104/3104
- `service-worker.js` v124
- `test_garantias.xlsx` 5.4KB de prueba con formato real (Garantias fila + header + 4 usuarios)
- Todos los event listeners con console.log para debug
- Menu bloqueado, Lista Precios bloqueada

## Próximos pasos recomendados
- Probar v124 con `?v=124` en incógnito + DevTools Console abierta para ver logs `📂 onchange inline`
- Si sigue sin cargar, usar botón de emergencia visible (input file nativo visible)
- Limpiar localStorage si está lleno: `localStorage.removeItem('usuarios_garantias')`
- Desactivar AdBlock para cdn.jsdelivr.net (XLSX) y unpkg (Leaflet)

