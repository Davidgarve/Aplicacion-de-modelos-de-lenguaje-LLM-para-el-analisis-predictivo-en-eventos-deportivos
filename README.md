# Aplicación de modelos de lenguaje LLM para el análisis predictivo en eventos deportivos

Este repositorio contiene el código fuente y la metodología de mi Trabajo de Fin de Grado (TFG) sobre anticipación táctica en el fútbol. La idea principal es comprobar si podemos usar Modelos de Visión-Lenguaje (VLMs) y Modelos de Lenguaje (LLMs) ejecutados en local para predecir a qué lado va a ir un penalti, fijándonos únicamente en la biomecánica y los movimientos del jugador antes y durante el golpeo.

## Contexto y motivación

Normalmente, el análisis de penaltis mediante IA se hace entrenando redes convolucionales o modelos de estimación de pose desde cero. El problema es que estos métodos suelen fallar cuando los vídeos son de retransmisiones de televisión estándar (baja resolución, motion blur, oclusiones). 

El enfoque de este proyecto es distinto: utilizamos modelos fundacionales multimodales (como Qwen-VL) para que actúen como "extractores de características". El modelo analiza el vídeo, razona sobre la postura del jugador y genera un resumen en texto (una memoria visual). Después, un segundo algoritmo lee ese texto para deducir la dirección del tiro.

## Arquitectura del sistema

El proceso es predictivo, es decir, el sistema no mira el vuelo del balón para tomar la decisión. Se divide en tres pasos:

1. **Segmentación temporal:** Recortamos el vídeo en las cuatro fases clave del gesto: carrera de aproximación, planta/apoyo, impacto y seguimiento de la pierna.
2. **Generación de memoria visual:** El VLM mira los frames de cada fase y devuelve un archivo JSON donde describe la orientación de la cadera, el pie de apoyo, el torso, etc.
3. **Inferencia táctica:** Ese JSON pasa por un segundo modelo (un LLM puro o un clasificador tradicional como Random Forest), que traduce las señales biomecánicas del jugador en coordenadas de la portería.

## Estructura del repositorio

El código está dividido en varios cuadernos de Jupyter pensados para ejecutarse en orden. Cada uno documenta una fase del preprocesamiento o una iteración de los experimentos del TFG:

### preprocesamiento.ipynb
Prepara los datos en bruto para que los modelos puedan digerirlos. 
* Agrupa las 9 zonas originales de la portería en un sistema más simple de 3 columnas (Izquierda, Centro, Derecha) para centrarnos en la lateralidad.
* Aplica recortes y rellenos (padding) temporales para obligar a que todos los vídeos duren exactamente 64 frames, asegurando que el instante del impacto nunca se pierda.

### graficas.ipynb
Este cuaderno es el motor visual del proyecto y el que genera las figuras para la memoria del TFG.
* Incluye el código para recortar al jugador usando las bounding boxes sin deformar su anatomía (preservando el aspect ratio mediante márgenes negros).
* Contiene el algoritmo que divide los 64 frames en las 4 fases biomecánicas (carrera, planta, impacto, seguimiento) muestreando solo los fotogramas clave para no saturar al VLM.
* Funciona como entorno de pruebas para calibrar el nivel de padding ideal y generar las tiras secuenciales de depuración visual.

### iteracion1_y_2.ipynb
Primeras pruebas para ver cuánto detalle anatómico entienden los modelos.
* La Iteración 1 intenta sacar microdetalles (ángulos de flexión exactos, rotaciones de tobillo), pero comprobamos que el vídeo se comprime demasiado para que sea fiable.
* La Iteración 2 cambia a un enfoque macro-gestual: le pedimos al modelo que mire cosas más generales como la inclinación de todo el torso o la inercia del cuerpo, lo cual funcionó bastante mejor.

### iteracion_3.ipynb
Pruebas para intentar corregir las alucinaciones espaciales del VLM.
* Procesamos los frames con YOLO-Pose para superponer un esqueleto 2D sobre el jugador. Esto ayuda a la red neuronal a no despistarse con la ropa o el fondo del estadio y a centrarse en los cruces articulares.

### iteracion4_y_5.ipynb
Actualización a modelos más pesados (Qwen-3.5-VL 8B y Llama 3.1 8B).
* Comprobamos si las variables cerradas que no aportan información se pueden eliminar sin perder precisión. 
* Separamos la evaluación del motor de visión y el de texto para ver quién falla realmente: si el VLM no ve bien el gesto o si el LLM se inventa la predicción al leerlo.

### iteracion_6.ipynb
La auditoría final del sistema.
* Convertimos todos los JSON de las memorias visuales en tablas normales (variables categóricas, confianza, visibilidad).
* Entrenamos modelos clásicos de Machine Learning (Regresión Logística y Random Forest) sobre esas tablas para ver el límite real de predicción sin usar LLMs en el último paso.
* Hacemos una prueba binaria: quitamos los tiros al centro (que meten mucho ruido) para aislar lo bien que lee el sistema la diferencia entre un tiro cruzado y uno abierto.

## Tecnologías y dependencias

Todo el pipeline está diseñado para correr en local, así que no hay llamadas a APIs de pago ni problemas de privacidad con los datos.

* `Python 3.x`
* `pandas` y `numpy` para manejar los datos y las tablas.
* `scikit-learn` para el machine learning clásico y validación cruzada.
* `opencv-python` (cv2) para leer, recortar y escalar los vídeos.
* `matplotlib` y `seaborn` para pintar las matrices de confusión y las gráficas.
* `ultralytics` para cargar YOLOv8 y dibujar los esqueletos.

Para los modelos de lenguaje utilizo **LM Studio**. Es muy cómodo porque lo levantas como servidor local, le cargas los modelos en formato GGUF (muy optimizados para VRAM) y le pegas peticiones desde Python como si fuera la API de OpenAI, pero corriendo todo en tu propia máquina.

## Cómo ejecutarlo

1. Clona el repositorio en tu ordenador.
2. Crea un entorno virtual e instala las librerías mencionadas arriba.
3. Descarga e instala LM Studio, carga el modelo que toque para cada iteración (Qwen-VL o Llama) y dale a iniciar servidor en el puerto 1234.
4. Abre el cuaderno `1_preprocesamiento.ipynb` y cambia la ruta de la variable `base_path` para que apunte a tu carpeta donde tengas los vídeos originales y el CSV.
5. Ve ejecutando los cuadernos en orden. Los vídeos recortados, los CSV generados y las gráficas se irán guardando solos en sus carpetas correspondientes.
