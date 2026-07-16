# Encuesta Scanner AI

Digitalización automática de la "Encuesta salud integral" (4 páginas fotografiables) usando visión por computadora con **posiciones fijas** (sin IA generativa) y envío directo a Google Forms.

## 1. Instalar y correr en local

```bash
npm install
npm run dev
```

Abre `http://localhost:5173`.

La carpeta `reference-images/` incluye las 4 fotos que se usaron para modelar la estructura de preguntas (`src/types/survey.ts`) y que puedes reutilizar directamente en el paso de calibración.

> **Nota sobre la página 4**: la foto de referencia disponible para esta página empieza en la pregunta 5 del bloque "paz" (¿Te sientes en paz contigo mismo?). Si en la hoja física ese bloque tiene preguntas 1 a 4 antes de la 5, agrégalas en `src/types/survey.ts` (dentro de `page4.fields`, usando `likertRow('paz', N, ...)`) antes de calibrar esa página, o el orden de calibración no coincidirá con la hoja real.

## 2. Calibrar la plantilla (una sola vez)

Antes de escanear encuestas reales:

1. Ve a la pestaña **Calibración**.
2. Para cada una de las 4 páginas, sube la foto de referencia (una hoja llena de ejemplo, bien encuadrada).
3. Haz clic en el centro de cada casilla/círculo en el orden que la app te indica. Para los campos manuscritos (edad, código) haz clic en la esquina superior-izquierda y luego inferior-derecha del recuadro donde se escribe.
4. Presiona **Guardar** en cada página.

Estas coordenadas (relativas, no en píxeles absolutos) quedan guardadas en `localStorage` del navegador y son las que usa el pipeline de escaneo real. **Si cambias de navegador o borras datos del sitio, tendrás que recalibrar.**

## 3. Configurar Google Forms

1. Ve a **Google Forms**, pega la URL de tu formulario y presiona **Detectar campos**.
2. La app intenta leer el HTML público del formulario para extraer los `entry.xxxxx` automáticamente.
   - **Esto normalmente fallará por CORS** (Google no expone cabeceras CORS en esa ruta para peticiones hechas desde JS de otro origen). Es una restricción del navegador/Google, no un bug de la app.
   - Cuando falla, la app te muestra un script de **Google Apps Script** listo para copiar: despliégalo como aplicación web y pega su URL en el campo correspondiente. Eso sí puede leer los `entry.xxxxx` de forma 100% confiable porque corre del lado de Google, no en el navegador del usuario.
3. Mapea cada pregunta de la encuesta con la pregunta correspondiente del Google Forms.

## 4. Escanear

1. Ve a **Escanear**, sube las 4 fotos en orden.
2. Presiona **Procesar 4 páginas**. Verás una barra de progreso real por etapa (detección de hoja, corrección, OCR, detección de marcas, validación).
3. En **Revisión**, corrige cualquier campo marcado en rojo/amarillo (baja confianza o ambiguo) y presiona **Enviar a Google Forms**.

## Limitaciones reales (léelas antes de usar en producción)

- **OCR de escritura a mano** (edad, código de encuesta): Tesseract.js es razonablemente bueno con dígitos manuscritos claros, pero no es perfecto. Por eso la app siempre muestra el campo para edición manual con su nivel de confianza — no confíes ciegamente en el número leído automáticamente si el badge está en amarillo/rojo.
- **Confirmación de envío a Google Forms**: el envío usa `fetch` con `mode: 'no-cors'` porque es la única forma de hacer un POST real desde el navegador sin backend propio. Esto significa que la app **no puede leer la respuesta HTTP real de Google** (por diseño de CORS) — "Enviado correctamente" indica que la petición de red se realizó sin errores, no una confirmación criptográfica de que Google la procesó. Si necesitas una confirmación 100% verificable, la alternativa es enviar a través del mismo Apps Script (`doPost`) en vez de a `formResponse` directamente.
- **Detección de hoja**: funciona con contornos rectangulares claros (buen contraste hoja/fondo). Con fondos muy parecidos en color a la hoja, o hojas muy arrugadas, la confianza baja y conviene repetir la foto.
- **Plantilla fija**: si el diseño impreso de la encuesta cambia (aunque sea el interlineado), hay que recalibrar esa página.

## Despliegue en Vercel

```bash
npm run build
```

Sube el repositorio a GitHub y conéctalo en [vercel.com/new](https://vercel.com/new), o usa la CLI:

```bash
npm i -g vercel
vercel --prod
```

No se necesitan variables de entorno: todo corre en el navegador del usuario (OpenCV.js y Tesseract.js vía WASM) y los datos (calibración, códigos enviados, config de Google Forms) se guardan en `localStorage`, por dispositivo/navegador.

## Arquitectura

```
src/
  types/survey.ts          Estructura de las 4 páginas y sus preguntas (fijo, real)
  types/calibration.ts     Tipos de la plantilla de coordenadas calibradas
  lib/opencv/               Detección de hoja, perspectiva, mejora de imagen
  lib/ocr/                  Wrapper de Tesseract.js
  lib/markDetection.ts      Medición de densidad de tinta por posición fija
  lib/scanPipeline.ts       Orquestador con progreso real por etapas
  lib/googleForms/          Extracción de entry.xxxxx + envío real
  lib/storage.ts            LocalStorage: calibración, config, duplicados
  pages/                    Escanear, Calibración, Revisión, Google Forms
  components/                UI reutilizable
```
