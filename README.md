# Sistema de Fotos de Vehículos — Trust Insurance Services

Captura guiada y validación automática de fotos de vehículos para emisión de pólizas.

## Qué hace
1. La persona abre un enlace (o QR), deja nombre, correo, teléfono y elige su tipo de vehículo.
2. Toma 6 fotos guiadas, en una vuelta al vehículo en sentido horario (frente, lateral derecho, trasera, lateral izquierdo) y luego VIN y odómetro. Cada una se valida **en el navegador**: resolución, nitidez, exposición, que sea un vehículo en la posición correcta, que todas sean del mismo color. El VIN y el odómetro solo pasan por resolución, nitidez y exposición: no se leen con OCR.
3. Las fotos y los datos se entregan **automáticamente al CRM** (GoHighLevel).

## Stack
- Vite + JavaScript vanilla (todo el wizard corre en el navegador)
- Validación: TensorFlow.js (COCO-SSD) y Tesseract.js; nitidez, exposición y color con JavaScript nativo
- Hosting: Cloudflare Workers con assets (`wrangler.jsonc`): sirve el sitio y atiende `/api/*`
- Servidor (`worker/`): único lugar que conoce el token del CRM

## Estructura
- `src/wizard/` — flujo, cámara, siluetas guía
- `src/validation/` — validadores independientes (cada uno devuelve ok / fail / unverified)
- `src/api/submit.js` — envío al servidor (progreso real y reintentos seguros)
- `worker/` — API: `api.js` (rutas), `ghl.js` (cliente del CRM), `index.js` (entrada)
- `dev/mock-ghl.js` — CRM simulado para probar sin credenciales
- `scripts/check-ghl.mjs` — verifica la conexión con el CRM real
- `tests/` — pruebas automáticas (`npm test`)

## Desarrollo
```bash
npm install
npm run dev      # sitio + /api/* con un CRM simulado (no necesita credenciales)
npm test         # pruebas automáticas
```
Modo de diagnóstico: abrir el sitio con `?debug=1` (o tocar 5 veces el logo) muestra bajo cada foto las medidas de calidad y la versión publicada.

## Conexión con el CRM (GoHighLevel)

### Qué se guarda en el CRM
Por cada persona que termina el wizard:
- **Contacto:** el del enlace (`?contact=ID`) o, si no hay, se crea/actualiza por correo y teléfono.
- **Fotos:** las 6 en el campo personalizado **«Fotos»** (tipo archivo) del contacto: se ven en su ficha. No se usa la biblioteca de medios.
- **Solo las fotos**, por ahora. Opcional (apagado por defecto): `GHL_ADD_NOTE=true` escribe además una nota con los enlaces, el tipo de vehículo, la ubicación del VIN, qué fotos quedaron sin verificar y las lecturas del OCR; `GHL_TAGS=etiqueta1,etiqueta2` pone etiquetas al contacto (útil para automatizaciones, p. ej. la Fase 2).

### Enlace con el contacto
Si Trust envía el enlace desde un workflow del CRM, puede incluir el ID del contacto:
`https://TU-SITIO.workers.dev/?contact={{contact.id}}`
Así las fotos quedan en ese contacto sin crear uno nuevo. Sin el parámetro, se usa el correo/teléfono que escribe la persona.

### Configuración (una sola vez)
1. **Crear el token.** En la subcuenta de Trust: *Ajustes → Integraciones privadas → Crear*. Permisos (scopes): `contacts.readonly`, `contacts.write` y los de campos personalizados (`locations/customFields.readonly` y `.write`, solo si hay que crear o revisar el campo «Fotos»). Copia el token (solo se muestra una vez).
2. **Ubicación.** El `Location ID` es el ID de la subcuenta (aparece en la URL del CRM: `.../location/<ID>/...`).
3. **Probar la conexión** en tu computador: copia `.env.example` a `.dev.vars`, completa `GHL_TOKEN` y `GHL_LOCATION_ID`, y ejecuta:
   ```bash
   npm run check:ghl
   ```
   Crea un contacto de prueba, sube 6 fotos al campo «Fotos», comprueba que la ficha las muestra, y borra el contacto. Cada paso dice si funcionó y, si no, la causa probable.
4. **Producción.** En Cloudflare → *Workers & Pages → sistema-de-fotos-tru → Settings → Variables and Secrets*, crea como **Secret**: `GHL_TOKEN` y `GHL_LOCATION_ID`. Los secretos se conservan en cada despliegue. El ID del campo «Fotos» ya viene en el código (`worker/api.js`). Opcionales: `GHL_PHOTOS_FIELD_ID`, `GHL_API_VERSION`, `GHL_ADD_NOTE`, `GHL_TAGS`.
5. Despliega (cada `push` a `main` lo hace) y haz una prueba real con tu teléfono.

### Probar el sitio y el servidor juntos con el CRM real
```bash
npm run cf:dev   # construye el sitio y levanta el Worker local con los valores de .dev.vars
```

### El campo «Fotos»
Es un campo personalizado del contacto (tipo archivo, hasta 6 archivos) que ya existe en el CRM de Trust. **Importante:** el CRM reemplaza los archivos del campo en cada subida en lugar de sumarlos; por eso las 6 fotos se envían juntas en una sola llamada, y repetir un envío nunca deja fotos duplicadas (si la misma persona vuelve a enviar, las fotos nuevas reemplazan a las anteriores; los enlaces de la nota anterior se conservan).

### Las variables de Cloudflare y los despliegues
**Créalas como _Secret_**, no como _Text_. `wrangler.jsonc` no define `vars` y tiene `keep_vars: true` para que un despliegue nunca borre lo creado en el panel; aun así, un Secret es lo correcto para un token. Si el sitio responde «La conexión con el CRM no está configurada» (error 500), el error en pantalla y en *Observability → Logs* dice qué variable falta (solo el nombre).

### CPU del Worker (plan gratuito)
El plan gratuito de Cloudflare permite ~10 ms de CPU por petición. Abrir las 6 fotos en el servidor y volver a armarlas gastaba justo eso y podía superarlo (error 500). Por eso el navegador arma el cuerpo con los archivos ya nombrados como exige el CRM y el Worker lo **reenvía sin abrirlo** (`handleFinish` en `worker/api.js`). A cambio, las validaciones de cada foto (JPEG, tamaño, calidad) viven solo en el navegador.

### Seguridad
- El token **nunca** llega al navegador: solo vive en el Worker.
- El servidor valida todo lo que recibe (datos y que cada foto sea JPEG y de tamaño razonable) y solo acepta peticiones del propio sitio.
- Si el CRM falla, al navegador solo llega qué paso falló; el detalle queda en los registros del Worker.
- Límite práctico: cualquiera con el enlace puede enviar fotos. Si hace falta más control, se puede agregar un código firmado al enlace.

## Despliegue
Cloudflare está conectado a este repositorio: cada `push` a `main` ejecuta `npm run build` y `npx wrangler deploy`.
