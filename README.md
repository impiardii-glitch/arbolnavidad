# 🎄 Arbrenadal — Navidad AR

Aplicación web de **realidad aumentada** que usa la cámara del móvil o del ordenador,
**detecta ventanas, paredes y árboles** y los decora de Navidad. La decoración queda
**anclada a la escena** (sigue el movimiento de la cámara), con **sombras**, **luces**
que iluminan su entorno y **nieve en 3D** que no atraviesa paredes ni techos.

| Detecta   | Decoración |
|-----------|------------|
| Ventanas  | Cristal escarchado con cristales de hielo, nieve acumulada en el alféizar, luces cálidas en el marco y nieve cayendo al otro lado |
| Árboles   | Luces que iluminan las ramas, bolas que se balancean al mover la cámara, espumillón dorado, nieve en las ramas (exterior) y estrella 3D giratoria |
| Paredes   | Guirnalda de pino en festones con luces, lazos y sombra sobre la pared; nieve acumulada en tejados y muros (exterior) |
| Todo      | Nieve en 3 capas de profundidad (lejos, media y bokeh cercano). En interior solo nieva tras las ventanas |

Las **fotos** se guardan siempre en la **galería de la app** (botón 🖼️, desde donde se
pueden volver a guardar o borrar) y además en el dispositivo: en una carpeta que eliges
una vez (Chrome / Edge de escritorio, botón 📁; la app pide permiso de acceso), con
"Guardar imagen" / "Guardar en Archivos" en iPhone, o en Descargas en Android y otros
navegadores.

Además permite **cambiar de cámara**, **pausar**, **modo noche**,
**luces cálidas**, elegir **detección rápida o precisa** y ver las **máscaras** detectadas.

## Enseñar a la IA (🎓)

Si la IA confunde cosas de tu sitio (una pared de cristal tomada por ventana, un seto
tomado por árbol…), puedes enseñarle:

1. Apunta a la zona y pulsa **🎓**: se congela la imagen.
2. Pinta encima con el dedo: **Ventana**, **Pared**, **Árbol**, **Techo**, **Cielo** o
   **Nada de esto** (para lo que confunde). Bastan unos trazos en cada elemento.
3. Pulsa **Entrenar**. En unos segundos aprende y empieza a usarlo.

Puedes enseñarle varias fotos (se acumulan) y desactivarlo con **Lo aprendido** u
olvidarlo desde el mismo panel. Todo se queda en tu dispositivo.

Por dentro: para cada zona de la imagen se guardan las 150 puntuaciones del modelo (una
por tipo de objeto de ADE20K) y su color, y se entrena un clasificador lineal (regresión
logística multiclase) que traduce esas puntuaciones a tus clases; su resultado se
combina con el del modelo. Se guarda por modelo (rápido / preciso) en IndexedDB.

## Cómo funciona

```
cámara ──► seguimiento de movimiento (cada fotograma) ──► G: mundo → pantalla
   │                                                         │
   └─► IA en Web Worker (SegFormer, ~8 análisis/s) ──► escena ┤──► dibujo 60 FPS
                                                              │    (sombras · adornos · luz · nieve)
       estimación de la luz real (brillo, tono, dirección) ───┘
```

- **Detección**: [SegFormer entrenado en ADE20K](https://huggingface.co/Xenova/segformer-b0-finetuned-ade-512-512)
  con [Transformers.js](https://huggingface.co/docs/transformers.js) dentro del navegador
  (WebGPU o WASM). No se envía ninguna imagen a ningún servidor. Se usan las
  **probabilidades** de cada clase (no solo la clase ganadora), se fusionan en el tiempo
  compensando el movimiento de la cámara, y las ventanas se ajustan a **cuadriláteros nítidos**.
  La imagen entra con su **proporción real** (sin aplastarla) y se alterna con su versión
  en **espejo**. Cuadros, puertas, cortinas o muebles altos cuentan como parte de la pared,
  los paneles de una misma ventana se unen, y los bordes de ventanas y paredes se
  **ajustan a las líneas reales** de la imagen (marcos, unión pared-techo, cornisas).
- **Rendimiento**: modelo en **fp16** con WebGPU (cuantizado q8 en CPU), preprocesado
  propio y **resolución adaptativa** (320–640 px) para mantener ~170 ms por análisis;
  con el seguimiento de movimiento la animación va a 60 FPS aunque el modelo sea lento.
- **Seguimiento (RA)**: flujo óptico por bloques contra un fotograma clave
  (traslación global + bloques refinados con Lucas-Kanade + ajuste robusto de
  traslación, giro y zoom). Toda la decoración vive en coordenadas de "mundo", así que
  se mueve con la escena entre un análisis y el siguiente.
- **Simulación 3D**: sombras suaves proyectadas en dirección opuesta a la luz detectada,
  exposición y tono de los adornos ajustados a la escena, luz emitida sumada aparte
  (las bombillas brillan más cuanto más oscuro está), péndulo físico en las bolas,
  paralaje y desenfoque en la nieve cercana.

## Archivos

```
index.html          Estructura de la página
style.css           Estilos (adaptados a móvil)
js/main.js          Bucle principal: cámara, seguimiento, IA, luz y composición de capas
js/worker.js        IA de segmentación (Web Worker)
js/tracker.js       Seguimiento del movimiento de la cámara
js/scene.js         Fusión de análisis y geometría: ventanas, árboles, borde de paredes
js/decor.js         Dibujo de la decoración (ventanas, árboles, paredes)
js/teach.js         Modo «Enseñar»: pintar, etiquetas y persistencia de lo aprendido
js/store.js         Almacén persistente (IndexedDB)
js/photos.js        Guardado de fotos (galería interna + carpeta con permiso, Guardar como, Fotos/Archivos en iPhone o descarga)
js/snow.js          Nieve en 3 capas con oclusión y colisiones
js/sprites.js       Sprites pre-renderizados (bombillas, bolas, estrella, guirnalda…)
js/geom.js          Utilidades geométricas
js/config.js        Parámetros ajustables
.github/workflows/pages.yml   Publicación automática en GitHub Pages
```

## Probarlo en local

La cámara solo funciona en **https** o en **localhost**. Desde la carpeta del proyecto:

```bash
python3 -m http.server 8000
```

y abre <http://localhost:8000>. La primera vez descarga el modelo (~15 MB en modo
rápido, ~110 MB en modo preciso); después queda en caché.

## Publicarlo en GitHub Pages

1. En GitHub: **Settings → Pages → Build and deployment → Source: GitHub Actions**.
2. Haz push a la rama `main` (o lanza el workflow a mano desde la pestaña **Actions**).
3. La web quedará en `https://<usuario>.github.io/arbrenadal/`.

## Consejos de uso

- Mueve la cámara con suavidad: el seguimiento necesita algo de textura (ladrillos,
  muebles, ramas…). Sobre una pared lisa y uniforme cuesta más.
- Si una ventana o pared no se detecta bien, prueba **Detección: Precisa**.
- **Noche** oscurece la imagen para que las luces luzcan más.
- Si la luz es muy baja (anochecer, interior oscuro), la detección automática baja los
  umbrales para no perder objetos; si es muy alta (sol directo), los sube para evitar
  falsos positivos.

## Mejorar la detección de objetos

La detección usa **SegFormer** (Transformers.js) entrenado en ADE20K, un modelo genérico
de segmentación semántica. No está especializado en ventanas, paredes y árboles navideños,
por lo que puede fallar en ciertos sitios.

### Opción 1: "Enseñar" a la IA (rápido, sin servidor)

Si la IA confunde elementos de tu sitio (una pared de cristal tomada por ventana, un seto
tomado por árbol…), usa el botón **🎓 Enseñar**:

1. Apunta a la zona y pulsa **🎓**: se congela la imagen.
2. Pinta encima con el dedo: **Ventana**, **Pared**, **Árbol**, **Techo**, **Cielo** o
   **Nada de esto** (para lo que confunde). Bastan unos trazos en cada elemento.
3. Pulsa **Entrenar**. La IA aprende un clasificador lineal que corrige las predicciones
   del modelo base en ese sitio.
4. Puedes enseñarle varias fotos (se acumulan) y desactivarlo con **Lo aprendido** u
   olvidarlo desde el mismo panel. Todo se queda en tu dispositivo (IndexedDB).

**Límite**: el adapter solo re-pesifica las probabilidades del modelo base. Si el modelo
base no detecta en absoluto una clase (por ejemplo, no distingue una ventana de un espejo),
el adapter no puede hacer milagros.

### Opción 2: Reentrenar el modelo con tus propias fotos (mejor detección general)

Para una detección realmente mejor en todo tipo de escenas navideñas, se necesita
**fine-tuning del modelo SegFormer** con fotos etiquetadas de ventanas, paredes y árboles.

#### Pasos:

1. **Recolectar fotos**: Haz 50–200 fotos de ventanas, paredes y árboles en distintos
   sitios, ángulos y condiciones de luz (día, noche, interior, exterior).

2. **Etiquetar**: Usa una herramienta de segmentación (por ejemplo, [CVAT](https://cvat.ai/),
   [LabelMe](https://labelme.cloud/) o [Roboflow](https://roboflow.com/)) para pintar
   máscaras de cada clase: ventana, pared, árbol, techo, cielo.

3. **Entrenar**: Fine-tunea SegFormer-b0 o b2 en ADE20K con tus datos. Necesitas una GPU
   (Colab o local). El resultado es un modelo en formato PyTorch/Hugging Face.

4. **Convertir a ONNX**: Usa `optimum` o `transformers.js` CLI para convertir el modelo
   a ONNX cuantizado (q8 para CPU, fp16 para WebGPU).

5. **Reemplazar en `config.js`**:
   ```js
   export const MODELS = {
     fast:  "tu-usuario/segformer-b0-finetuned-navidad",  // tu modelo rápido
     precise: "tu-usuario/segformer-b2-finetuned-navidad", // tu modelo preciso
   };
   ```

6. **Ajustar grupos en `config.js`**: Si tus clases etiquetadas tienen nombres distintos
   a los de ADE20K, actualiza las expresiones regulares de `GROUPS`.

7. **Ajustar umbrales en `config.js`**: Los umbrales por defecto están calibrados para
   ADE20K. Con un modelo propio, prueba valores entre 0.3 y 0.6.

### Ajustes (`js/config.js`)

| Parámetro               | Efecto |
|-------------------------|--------|
| `thresholds`            | Umbral base de probabilidad por clase |
| `minThresholds` / `maxThresholds` | Límites para los umbrales adaptativos según luz |
| `blend`                 | Peso de cada análisis nuevo (más bajo = más estable, más alto = reacciona antes) |
| `adapterWeight`         | Cuánto pesa lo aprendido vs el modelo base (0 = solo modelo, 1 = solo adapter) |
| `adapterConfidence`     | Confianza mínima del adapter para aplicarlo (0.55 por defecto) |
| `minTreeArea`, `minWindowArea` | Tamaño mínimo (fracción de imagen) para decorar |
| `minTreeHeightRatio`    | Altura mínima del árbol respecto a la imagen (0.18 = 18 %) |
| `minWindowRect`         | No usado directamente, forma controlada por relación de aspecto |
| `snow`                  | Número de copos por capa |
