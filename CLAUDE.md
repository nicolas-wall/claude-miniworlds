# Proyecto WebAR — contexto del repositorio

> Archivo de contexto para Claude Code. Contiene las decisiones ya tomadas,
> sus fundamentos, y lo que está explícitamente descartado.
> **No revisar decisiones marcadas como CERRADAS sin fundamento nuevo.**

---

## 1. Objetivo

Una imagen impresa que, escaneada con el teléfono, abre un pequeño mundo en
realidad aumentada — efecto portal, se ve "hacia adentro" de la imagen.

**Sin descarga de app.** Todo corre en el navegador.

La identidad de la experiencia es **la imagen**, no el lugar. La misma imagen
puede estar en una revista, en un afiche en la calle, en un flyer. No hay
vínculo con coordenadas geográficas.

---

## 2. Restricciones duras (no negociables, definen la arquitectura)

| Restricción | Consecuencia |
|---|---|
| **iOS Safari no soporta WebXR `immersive-ar`** | Queda descartado todo el camino A-Frame + WebXR. Obligatorio tracking por visión computacional sobre `getUserMedia` + canvas. |
| **La cámara exige contexto seguro** | HTTPS o `localhost`. `file://` y HTTP por IP no funcionan. |
| **Navegadores embebidos** (Instagram, Facebook, TikTok, WeChat) | En iOS no tienen permiso de cámara. Hace falta detección de user-agent y una pantalla que fuerce abrir en el navegador del sistema. Es el canal por el que va a llegar buena parte del tráfico. |
| **Presupuesto de peso: < 5 MB por escena** | Se abre con datos móviles, parado en la calle. glTF con compresión Draco + texturas KTX2. Cargar el `.glb` recién en `targetFound`, nunca al inicio. |

---

## 3. Stack

- **MindAR (image tracking) + Three.js** — open source, corre en iOS y Android.
- Hosting estático con TLS: Vercel o Netlify.
- Assets: `.glb` con Draco, texturas KTX2.
- Compilación de targets: compilador oficial de MindAR
  (`hiukim.github.io/mind-ar-js-doc/tools/compile`). Genera el `.mind`.

**Alternativa evaluada y descartada por ahora:** 8th Wall / Niantic Studio.
Mejor tracking y SLAM, pero ~USD 100+/mes y lock-in total del código.
Reconsiderar solo si MindAR no alcanza el piso de calidad en Fase 0.

---

## 4. Decisiones cerradas

### 4.1 El QR nunca es el target de tracking — CERRADA
Un QR tiene baja densidad de features y patrones repetidos: el tracker lo
pierde constantemente. **Dos funciones separadas:**
- QR / NFC / URL corta → resuelve a qué página ir.
- Imagen de tracking → textura rica y asimétrica, es otra cosa.

### 4.2 Escala relativa, no absoluta — CERRADA
MindAR **normaliza el ancho del target a 1 unidad**. Todo lo que se modele
dentro del anchor está en unidades-de-imagen, no en metros.

- Imagen impresa a 10 cm → el mundo mide 10 cm.
- Imagen impresa a 2 m → el mundo mide 2 m.

**La escala ya es automática y proporcional. No hay cálculo que hacer y no hace
falta conocer el tamaño de impresión en runtime.** Para un efecto portal es
exactamente el comportamiento deseado: el mundo llena el marco de la imagen,
siempre, en cualquier soporte. Se autora una vez y funciona en revista y en
afiche.

**Escala absoluta y "misma imagen en cualquier soporte" son incompatibles.**
Si se quisiera un personaje de 1,70 m real, el factor sería
`1.70 / anchoImpresoEnMetros`: en una revista de 15 cm da 11,3× (el personaje
sale disparado fuera del marco), en un afiche de 2 m da 0,85×. La misma escena
se convierte en dos cosas distintas.

Si en algún momento se necesita escala absoluta, la visión **no puede inferir**
el tamaño físico. Única salida: variantes de URL (`/w/patio?cm=15` en revista,
`?cm=200` en afiche) con la misma imagen y distinto QR. Beneficio lateral:
atribución de campaña gratis.

### 4.3 Los mundos son datos, no código — CERRADA
Un JSON define qué mundos existen. El motor AR no sabe cuántos hay.
Hardcodear la primera experiencia obliga a reescribir todo en el mundo 4.

### 4.4 Un solo bundle `.mind` con múltiples imágenes — CERRADA
MindAR compila varias imágenes en un único `.mind` y devuelve `targetIndex`
al detectar. Una sola URL para toda la campaña.

**Límite conocido:** el costo de detección crece con la cantidad de targets.
Arriba de ~8-10 imágenes el matching se degrada en gama media. La solución
entonces **no es optimizar sino particionar** el `.mind` por campaña o zona.
Por eso `bundleUrl` es propiedad de un grupo, no constante global.

---

## 5. Esquema de datos (v3)

```json
{
  "schemaVersion": 3,
  "bundleUrl": "/targets/campana-v1.mind",
  "worlds": [
    {
      "id": "patio-01",
      "targetIndex": 0,
      "title": "El patio de atrás",
      "modelUrl": "/scenes/patio-v3.glb",
      "sizeBytes": 3900000,
      "scaleMode": "relative",
      "minPrintCm": 12,
      "status": "live",
      "updatedAt": "2026-07-28T12:00:00Z"
    }
  ]
}
```

**Justificación de cada campo:**

- `id` — **inmutable, nunca se reusa.** Es la clave de analytics histórico.
- `targetIndex` — índice dentro del bundle `.mind`. Lo devuelve MindAR al detectar.
- `sizeBytes` — permite advertir *"3,9 MB, ¿continuar?"* antes de quemarle datos al usuario.
- `minPrintCm` — límite duro de impresión, no sugerencia (ver 6.3).
- `status` (`draft` / `live` / `retired`) — apagar un mundo sin deploy.
- `updatedAt` — cache-busting. Sin esto el CDN sirve el `.glb` viejo y no se entiende por qué no actualiza.
- `schemaVersion` — presente desde el día uno para permitir migración.

**Descartado del schema:** `placements` (dónde está físicamente cada copia).
Al no estar la experiencia atada a un lugar, la ubicación no es dato del
sistema. Se reemplaza por `?src=revista-gente-ago` en la URL: analytics de
canal sin tocar el schema.

**Evolución prevista:** JSON estático en el repo (Fase 0-1) → Postgres/Supabase
con endpoint `/api/worlds` (Fase 2, cuando haga falta editar sin deploy).
El schema no cambia; cambia el transporte.

---

## 6. Restricciones de impresión (son técnicas, no estéticas)

### 6.1 Sustrato
**Papel ilustración brillante es el peor caso posible.** El reflejo especular
borra features enteras bajo luz cenital. Pedir **barniz mate**. Sin presupuesto
para mate: diseñar con medios tonos oscuros y evitar grandes áreas blancas —
el blanco brilloso es el que más rebota.

### 6.2 Curvatura
MindAR asume homografía planar. Una página cerca del lomo se deforma y el
tracking salta. **La imagen va al centro de página o a doble página completa,
nunca pegada al pliegue.**

### 6.3 Tamaño mínimo
Abajo de ~10-12 cm de ancho el detalle de features cae bajo la resolución útil
de la cámara a distancia de lectura. **Media página es el piso.**

### 6.4 Rango de tracking
Aproximadamente **1× a 4× el ancho físico** de la imagen. Un afiche de 2 m no
se trackea desde 15 m: la cámara no resuelve el detalle. La distancia máxima
real es más corta de lo que la gente asume.

### 6.5 Distancia visual entre imágenes de la misma familia
Si se hace una serie con identidad visual compartida, **el tracker las
confunde**. El matching es por features locales: dos piezas del mismo estilo
con distinto texto son casi idénticas para el algoritmo. Cada imagen necesita
distinta composición y distinta distribución de contraste.
**Verificar el score del compilador de MindAR antes de mandar a imprenta.**

---

## 7. Estado actual — Fase 0

Existe `fase0.html`: banco de pruebas. Un target, un cubo girando, escala
relativa (cubo = 25% del ancho del target), marco de referencia ámbar.

**No es el primer commit del producto. Es un instrumento de medición.**
Deliberadamente no lee el JSON, no resuelve `targetIndex` dinámico y no tiene
portal.

### Qué mide
| Métrica | Para qué |
|---|---|
| Tiempo hasta enganche | Latencia de detección percibida |
| FPS mediana | Si el dispositivo aguanta contenido más pesado |
| Cantidad de pérdidas | Proxy de jitter / estabilidad |
| % de sesión con tracking | Calidad real del target en condiciones de calle |

Doble toque en el HUD cierra la sesión y muestra el informe.

### Diagnóstico visual
El **marco ámbar** debe quedar pegado a los bordes de la imagen impresa. Si
flota, respira o se desfasa en ángulo rasante, la homografía es débil y no
tiene sentido pasar a Fase 1 con esa imagen.

### Criterios de aceptación (gama media, a pleno sol, no en escritorio)
- FPS mediana ≥ 25
- Enganche < 2 s
- < 3 pérdidas por minuto sostenido

**Si un target da 40% de hold al sol, el problema es la imagen, no el código.**

### Setup
1. Compilar el `.mind` en el compilador de MindAR. **Anotar el score: abajo de
   ~3 estrellas no mandar a imprenta.**
2. Ajustar `TARGET_ASPECT` en `CONFIG` con el ancho/alto real de la imagen.
3. Servir por HTTPS. Para probar sin deploy: `npx serve` + `npx localtunnel`,
   o push a Vercel.

---

## 8. Roadmap

**Fase 1 — Portal.** Efecto de "ver hacia adentro" mediante stencil buffer en
Three.js. Acá aparece el problema serio: **orden de renderizado y stencil**.
Es la parte no trivial del proyecto. Entra también el `worlds.json` con
`targetIndex` dinámico y carga diferida del `.glb` en `targetFound`.

**Fase 2 — Multi-mundo y contenido.** Varios targets en un bundle, audio,
interacción. Migración del JSON a base de datos si hace falta editar sin deploy.

**Fase 3 — Escala de campaña.** Partición de bundles, analytics por `?src=`,
panel de administración.

---

## 9. Riesgos abiertos

- **Tracking en exteriores:** reflejo, lluvia, sombras duras y ángulos rasantes
  degradan el reconocimiento. Se mide en Fase 0, no se supone.
- **Navegadores embebidos:** ya hay detección, pero es un punto de fuga real de
  usuarios. Medir cuántos llegan por ahí.
- **Copia de la imagen:** cualquiera la fotocopia y el mundo aparece en su
  cocina. **Con este modelo es imposible de prevenir.** Está asumido como
  aceptable — la mitigación (validación por geo) contradice todo el diseño.
- **Vandalismo / desgaste en vía pública:** si el afiche se raya, ese punto
  muere. Mitigación: código de fallback tipeable, si se decide implementarlo.

---

## 10. Decisión pendiente

Ninguna bloqueante. El siguiente paso es **correr Fase 0 con un target real,
impreso, al sol**, y traer los cuatro números antes de escribir el portal.
