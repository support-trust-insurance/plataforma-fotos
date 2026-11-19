# Sistema de Fotos de Vehículos — Trust Insurance Services

Plataforma web para capturar fotos de vehículos de forma guiada y validarlas automáticamente, como parte del proceso de emisión de pólizas. Está pensada para usarse desde el teléfono: la persona abre un enlace (o escanea un QR) y el sistema la acompaña paso a paso.

## Qué hace
1. La persona deja su nombre, correo y teléfono, y elige el tipo de vehículo.
2. Toma 6 fotos guiadas con la cámara del teléfono: una vuelta al vehículo en sentido horario (frente, lateral derecho, trasera, lateral izquierdo) y luego el VIN y el odómetro. Una silueta sobre la cámara indica cómo encuadrar cada foto y se adapta al tipo de vehículo y a la orientación del teléfono.
3. Cada foto se valida en el navegador antes de aceptarla: resolución, nitidez, exposición, que sea un vehículo en la posición correcta y que todas sean del mismo color. El VIN y el odómetro solo pasan por resolución, nitidez y exposición.
4. Al final se muestra un resumen donde se puede ver cada foto en grande y cambiarla antes de enviar.
5. Las fotos y los datos se envían de forma segura a un servidor propio, que los entrega al sistema de la empresa.

La interfaz está disponible en español e inglés, funciona en vertical y en horizontal, y respeta la preferencia de movimiento reducido del dispositivo.

## Tecnologías
- **Vite** y **JavaScript vanilla**: todo el flujo corre en el navegador, sin frameworks.
- **HTML y CSS** propios: diseño adaptable (mobile-first), animaciones con CSS y siluetas guía dibujadas en SVG.
- **TensorFlow.js (COCO-SSD)**: detección de vehículos en las fotos exteriores.
- **API de cámara del navegador** (`getUserMedia`) y **Canvas**: captura de las fotos y medición de nitidez, exposición y color con JavaScript nativo.
- **Worker serverless** (carpeta `worker/`): recibe las fotos, valida los datos y las entrega al sistema de la empresa.
- **Node.js** con su ejecutor de pruebas integrado, para las pruebas automáticas.

## Estructura
- `src/wizard/` — flujo, cámara, siluetas guía
- `src/validation/` — validadores independientes (cada uno devuelve ok / fail / unverified)
- `src/api/submit.js` — envío al servidor (progreso real y reintentos seguros)
- `src/ui/` — efectos visuales de la interfaz
- `src/i18n.js` — textos en español e inglés
- `worker/` — API del servidor
- `dev/` — servidor simulado para desarrollar sin credenciales
- `tests/` — pruebas automáticas

## Desarrollo
```bash
npm install
npm run dev      # sitio y servidor simulado, sin credenciales
npm test         # pruebas automáticas
npm run build    # versión de producción
```
Modo de diagnóstico: abrir el sitio con `?debug=1` (o tocar 5 veces el logo) muestra bajo cada foto las medidas de calidad y la versión publicada.
