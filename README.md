# Aplicación de modelos de lenguaje LLM para el análisis predictivo en eventos deportivos

Este repositorio contiene el código y los cuadernos desarrollados para mi Trabajo de Fin de Grado, centrado en el uso de modelos de lenguaje y modelos multimodales para analizar eventos deportivos a partir de vídeo.

El caso de estudio elegido son los lanzamientos de penalti en fútbol. Aunque pueda parecer una acción sencilla, porque el escenario está bastante acotado y el resultado del disparo se puede etiquetar de forma clara, en la práctica es un problema bastante difícil. El gesto ocurre muy rápido, muchas señales aparecen solo durante unos pocos fotogramas, la cámara no siempre ofrece una buena perspectiva y, además, el lanzador intenta no revelar de forma evidente hacia dónde va a tirar.

La idea principal del proyecto no es construir un sistema capaz de predecir penaltis con una precisión alta y listo para usarse en un contexto profesional. El objetivo real es estudiar si una arquitectura basada en modelos de visión-lenguaje puede extraer señales útiles del gesto del jugador y convertirlas en una representación intermedia interpretable. Esa representación se llama en el proyecto **memoria visual**.

En lugar de entrenar una red neuronal que reciba el vídeo y devuelva directamente una clase final, el sistema separa el problema en dos partes. Primero, un modelo de visión-lenguaje observa una selección de fotogramas del penalti y describe lo que ve: postura del cuerpo, orientación del apoyo, trayectoria de la pierna, inclinación del torso, seguimiento posterior al golpeo, etc. Después, un segundo modelo utiliza esa descripción para intentar inferir la dirección del lanzamiento.

Esta separación permite estudiar mejor qué está ocurriendo dentro del sistema. Si la predicción falla, no solo se mira el acierto final, sino también si el modelo visual había generado una memoria útil o si el problema estaba en la fase posterior de interpretación.

## Contexto y motivación

El análisis automático de acciones deportivas suele abordarse con técnicas de visión por computador, redes convolucionales, estimación de pose o modelos entrenados específicamente para reconocer patrones de movimiento. Estos métodos pueden funcionar bien en entornos controlados, pero los vídeos reales de fútbol no suelen ser tan limpios.

En este proyecto se trabaja con vídeos de retransmisiones o clips deportivos donde aparecen varios problemas habituales:

- baja resolución en algunas zonas del cuerpo;
- desenfoque por movimiento;
- cambios de escala entre vídeos;
- cámaras colocadas en perspectivas diagonales;
- oclusiones parciales;
- dificultad para ver detalles pequeños como el pie de apoyo o el punto exacto de contacto con el balón.

En un penalti, estos problemas afectan mucho. Algunas señales biomecánicas que podrían ser útiles para anticipar el disparo son muy sutiles. Por ejemplo, la orientación del pie de apoyo, la apertura de la cadera o el seguimiento de la pierna pueden aportar información, pero no siempre son fáciles de observar en vídeo estándar.

Por eso, el enfoque del trabajo fue evolucionando. Al principio se intentó que el modelo analizara señales muy finas del gesto, pero durante las pruebas se vio que muchas de ellas eran demasiado ambiguas. Más adelante se pasó a estudiar señales más globales, como la postura general del cuerpo, la dirección de la carrera, la trayectoria de la pierna de golpeo o la inercia posterior al disparo.

El interés del proyecto está precisamente en esa evolución: comprobar qué tipo de información puede extraer un VLM, qué señales son demasiado difíciles de observar y hasta qué punto una memoria visual en texto puede servir como base para una predicción posterior.

## Arquitectura del sistema

El sistema sigue un enfoque predictivo. Es decir, no utiliza el vuelo del balón para decidir la dirección del penalti. Solo se analiza la información disponible antes y durante el golpeo.

El pipeline completo se divide en varias fases:

1. **Preprocesamiento temporal**

   Los vídeos originales tienen duraciones distintas. Para poder compararlos y procesarlos de forma homogénea, se normalizan a una longitud común de 64 fotogramas.

   Si un vídeo es más largo, se recorta conservando la parte final para no perder el golpeo. Si es más corto, se añade padding al inicio duplicando el primer fotograma.

2. **Sincronización de anotaciones**

   El dataset incluye anotaciones fotograma a fotograma, como la caja del jugador, el instante exacto del golpeo y la zona real del disparo.

   Como los vídeos se recortan o rellenan durante el preprocesamiento, es necesario remapear los índices para que las bounding boxes y el `kick frame` sigan correspondiendo al fotograma correcto.

3. **Recorte del jugador**

   Se utiliza la bounding box del lanzador para recortar la zona relevante de la imagen. El objetivo es reducir ruido visual y centrar al modelo en el gesto del jugador.

   Aun así, no se recorta de forma excesivamente cerrada, porque es importante conservar algo de contexto: balón, postura general, desplazamiento del cuerpo y orientación del movimiento.

4. **Preservación de la relación de aspecto**

   Los recortes del jugador se adaptan a un formato cuadrado sin deformar la imagen. Para ello se mantiene la proporción original y se añaden márgenes negros cuando hace falta.

   Esto es importante porque deformar el cuerpo del jugador podría alterar visualmente la inclinación del torso, la posición de la pierna o la orientación del apoyo.

5. **Segmentación temporal por fases**

   Cada penalti se divide en cuatro fases del gesto:

   - carrera de aproximación;
   - planta o apoyo previo al golpeo;
   - impacto con el balón;
   - seguimiento e inercia posterior.

   En lugar de enviar los 64 fotogramas completos al VLM, se seleccionan fotogramas representativos de cada fase. Así se reduce el coste computacional y se evita saturar al modelo con información repetida.

6. **Generación de memoria visual**

   El modelo de visión-lenguaje analiza los fotogramas seleccionados y genera una memoria visual del lanzamiento.

   Esta memoria puede tener formato abierto, con una descripción más libre del gesto, o formato cerrado, usando variables categóricas predefinidas.

7. **Inferencia táctica**

   La memoria visual pasa a un segundo módulo. En las primeras iteraciones se utilizó un LLM para interpretar el texto y predecir la dirección del disparo. En la fase final también se probaron clasificadores clásicos, como Regresión Logística y Random Forest, para estudiar cuánta señal útil contenía realmente la memoria visual.

8. **Mapeo a la clase final**

   En varias pruebas la inferencia se hace primero en un marco relativo al lanzador:

   - `CRUZADO`
   - `CENTRAL`
   - `ABIERTO`

   Después, esa predicción se transforma a la clase final del dataset teniendo en cuenta si el jugador es diestro o zurdo.

## Estructura del repositorio

El código está organizado en cuadernos de Jupyter. Cada cuaderno corresponde a una parte del preprocesamiento o a una iteración concreta del proyecto.

La idea es que puedan ejecutarse en orden, aunque varios cuadernos también sirven como entorno de pruebas para generar gráficas, revisar recortes o comprobar resultados intermedios.

### preprocesamiento.ipynb

Este cuaderno prepara los datos originales para que puedan ser utilizados por el resto del sistema.

Incluye:

- lectura de vídeos y anotaciones originales;
- agrupación de las 9 zonas originales de la portería en 3 zonas horizontales;
- normalización temporal de todos los vídeos a 64 fotogramas;
- aplicación de recorte o padding según la duración de cada vídeo;
- sincronización entre los nuevos índices de vídeo y las anotaciones originales;
- generación de archivos intermedios para las siguientes fases.

La reducción de 9 zonas a 3 se hace porque el proyecto se centra en la lateralidad del disparo. Predecir también la altura del balón habría requerido detalles visuales mucho más finos, como el punto exacto de contacto con el balón, que no siempre son fiables en los vídeos disponibles.

### graficas.ipynb

Este cuaderno se utilizó principalmente para generar figuras, revisar visualmente el preprocesamiento y preparar material para la memoria del TFG.

Incluye:

- visualización de la distribución de longitudes de los vídeos;
- análisis del contexto conservado al usar ventanas de 64 fotogramas;
- pruebas con distintos niveles de padding espacial;
- comparación entre redimensionamiento directo y preservación de aspect ratio;
- generación de tiras de fotogramas por fase;
- revisión visual de la segmentación del gesto;
- creación de algunas matrices de confusión y gráficas usadas en la memoria.

También sirvió para ajustar decisiones prácticas del pipeline, como el margen alrededor del jugador o el número de fotogramas seleccionados por fase.

### iteracion1_y_2.ipynb

Este cuaderno recoge las dos primeras iteraciones experimentales del sistema.

#### Iteración 1

La primera iteración siguió un enfoque micro-biomecánico. Se intentó que el VLM analizara señales bastante finas del gesto, como:

- orientación del pie de apoyo;
- posición relativa del apoyo respecto al balón;
- apertura de cadera;
- inclinación del tronco;
- orientación de pelvis y torso;
- zona aparente de contacto entre bota y balón;
- dirección inicial del seguimiento.

Se probaron dos tipos de memoria visual:

- una memoria abierta, donde el modelo describía libremente el gesto;
- una memoria cerrada, donde el modelo debía escoger entre etiquetas predefinidas.

Los resultados fueron bajos. El principal aprendizaje de esta fase fue que muchas señales micro-biomecánicas eran demasiado difíciles de observar con fiabilidad en los vídeos disponibles.

#### Iteración 2

La segunda iteración cambió el enfoque. En lugar de insistir en detalles muy pequeños, se pasó a señales macro-gestuales más visibles.

Se analizaron aspectos como:

- orientación general del cuerpo;
- dirección de llegada al balón;
- inclinación del torso;
- trayectoria de la pierna de golpeo;
- dirección del seguimiento;
- giro aparente del cuerpo tras el disparo.

Esta iteración funcionó mejor que la primera. No convirtió el sistema en un predictor robusto, pero sí mostró que las señales globales del gesto eran más útiles que las micro-biomecánicas para este tipo de vídeo.

### iteracion_3.ipynb

La tercera iteración incorporó YOLO-Pose para superponer un esqueleto 2D sobre el jugador.

La idea era comprobar si una representación de pose podía ayudar al VLM a localizar mejor las relaciones anatómicas importantes: hombros, caderas, rodillas, tobillos y orientación general del cuerpo.

El pipeline fue:

1. recortar al jugador;
2. aplicar YOLO-Pose;
3. generar imágenes con el esqueleto superpuesto;
4. enviar esas imágenes al VLM;
5. crear una memoria visual basada en la pose;
6. pasar esa memoria al LLM para la predicción final.

El resultado fue intermedio. La pose ayudaba en algunos casos a ordenar visualmente el cuerpo, pero no resolvía el problema principal. En acciones rápidas, con perspectiva diagonal y cruces de piernas, la pose 2D también podía ser ambigua.

Esta iteración sirvió para comprobar que el fallo no estaba solo en localizar el cuerpo del jugador, sino en interpretar correctamente señales muy rápidas y a veces poco informativas.

### iteracion4_y_5.ipynb

Este cuaderno recoge las pruebas con modelos más recientes y una revisión de los enfoques anteriores.

#### Iteración 4

En la cuarta iteración se volvió al enfoque micro-biomecánico, pero usando modelos más actualizados.

El objetivo era comprobar si los malos resultados de la Iteración 1 se debían al enfoque utilizado o a que el modelo visual inicial no tenía suficiente capacidad.

Se probaron de nuevo:

- prompt abierto;
- prompt cerrado;
- prompt cerrado con poda de variables poco informativas.

La versión podada eliminaba variables que aparecían casi siempre con la misma etiqueta. La idea era quitar atributos que no aportaban diversidad real a la memoria visual.

La mejora fue limitada. Esto reforzó una conclusión importante del proyecto: el problema no era únicamente el modelo utilizado, sino la dificultad real de observar señales biomecánicas finas en vídeos de este tipo.

#### Iteración 5

La quinta iteración recuperó el enfoque macro-gestual, esta vez con modelos más recientes.

Se mantuvo el VLM actualizado y se compararon distintas estrategias de inferencia textual. También se probaron diferentes formulaciones del prompt abierto, dando más importancia a señales globales del movimiento.

Esta fue una de las líneas más prometedoras del proyecto. El sistema seguía teniendo problemas, sobre todo con los lanzamientos centrales, pero las memorias visuales generadas parecían contener más información útil que en las variantes micro-biomecánicas.

### iteracion_6.ipynb

La sexta iteración fue la fase de diagnóstico final del sistema.

En lugar de limitarse a probar otro prompt, se analizó la memoria visual como objeto de estudio. Para ello, las memorias generadas por el VLM se convirtieron en una tabla de atributos.

Cada variable de la memoria se transformó en columnas como:

- valor categórico;
- confianza;
- visibilidad;
- fase del gesto a la que pertenece.

Después se entrenaron modelos clásicos de machine learning sobre esa tabla:

- Regresión Logística;
- Random Forest.

El objetivo era comprobar si la memoria visual contenía señal predictiva antes de pasar por el LLM.

También se hizo una evaluación binaria eliminando los tiros al centro. Esta prueba no sustituye al problema original de tres clases, pero ayuda a estudiar si el sistema distingue mejor entre disparos laterales, es decir, entre abierto y cruzado.

Esta iteración permitió separar mejor dos tipos de error:

- errores de observación del VLM;
- errores de interpretación del LLM.

La conclusión principal fue que la memoria visual no estaba vacía. En varios casos contenía información útil, especialmente para diferenciar direcciones laterales. Aun así, esa señal no era lo bastante fuerte ni estable como para obtener un predictor robusto en el problema completo.

## Tecnologías y dependencias

El proyecto está diseñado para ejecutarse en local. No depende de APIs de pago ni de servicios externos para procesar los vídeos o consultar los modelos.

Las principales librerías utilizadas son:

- `Python 3.x`
- `pandas`
- `numpy`
- `opencv-python`
- `matplotlib`
- `seaborn`
- `scikit-learn`
- `ultralytics`
- `json`
- `os`
- `glob`

Para la parte de modelos de lenguaje se utilizó **LM Studio**. La ventaja de LM Studio es que permite levantar un servidor local compatible con llamadas tipo API de OpenAI, pero ejecutando modelos descargados en la propia máquina.

Durante el proyecto se trabajó con modelos locales y cuantizados, entre ellos:

- Qwen-VL / Qwen2.5-VL;
- Qwen-3.5-VL;
- Qwen3.5 como LLM textual;
- Llama 3.1;
- DeepSeek;
- Gemma.

La elección de modelos estuvo condicionada por el hardware disponible. El sistema se desarrolló en un entorno local, con una GPU de consumo, por lo que fue necesario trabajar con modelos compactos o cuantizados.

## Cómo ejecutarlo

1. Clona el repositorio:

   ```bash
   git clone <url-del-repositorio>
   cd <nombre-del-repositorio>
   ```

2. Crea un entorno virtual:

   ```bash
   python -m venv .venv
   ```

3. Activa el entorno virtual.

   En Windows:

   ```bash
   .venv\Scripts\activate
   ```

   En Linux o macOS:

   ```bash
   source .venv/bin/activate
   ```

4. Instala las dependencias necesarias:

   ```bash
   pip install pandas numpy opencv-python matplotlib seaborn scikit-learn ultralytics
   ```

5. Descarga e instala LM Studio.

6. Carga en LM Studio el modelo correspondiente a la iteración que quieras ejecutar.

7. Inicia el servidor local de LM Studio. En los cuadernos se asume normalmente el puerto:

   ```text
   http://localhost:1234
   ```

8. Abre el cuaderno de preprocesamiento y modifica la variable `base_path` para que apunte a la carpeta donde tengas los vídeos y el CSV de anotaciones.

9. Ejecuta primero:

   ```text
   preprocesamiento.ipynb
   ```

10. Después puedes ejecutar el resto de cuadernos según la parte del proyecto que quieras reproducir:

   ```text
   graficas.ipynb
   iteracion1_y_2.ipynb
   iteracion_3.ipynb
   iteracion4_y_5.ipynb
   iteracion_6.ipynb
   ```

Los archivos generados, como vídeos recortados, memorias visuales, CSV intermedios, matrices de confusión y gráficas, se guardan en las carpetas configuradas dentro de cada cuaderno.

## Resultados generales

Los resultados del proyecto no permiten considerar el sistema como un predictor fiable de la dirección de un penalti. El problema es difícil, especialmente por la clase central, que introduce mucha ambigüedad. En varias iteraciones el sistema tendía a confundir los tiros al centro con una de las dos zonas laterales.

Aun así, los experimentos sí muestran algo interesante: las memorias visuales generadas por el VLM no son simplemente ruido. En algunas configuraciones contienen señales aprovechables, sobre todo cuando el problema se reduce a distinguir entre direcciones laterales.

Las principales conclusiones prácticas fueron:

- las señales micro-biomecánicas son difíciles de observar con fiabilidad en vídeos reales;
- las señales macro-gestuales funcionan mejor que los detalles anatómicos finos;
- la estimación de pose ayuda en algunos casos, pero no elimina la ambigüedad del gesto;
- separar observación visual e inferencia textual facilita el análisis de errores;
- los modelos locales pueden usarse para experimentar con este tipo de arquitectura, aunque tienen limitaciones claras;
- la memoria visual es útil como herramienta de diagnóstico, incluso cuando la predicción final no es robusta.

## Limitaciones

El sistema no debe entenderse como un predictor robusto de penaltis. Los resultados muestran que el problema es difícil, especialmente por la ambigüedad de los lanzamientos centrales y por las limitaciones visuales de los vídeos disponibles.

Durante el desarrollo se observaron varias limitaciones importantes:

- las señales micro-biomecánicas son difíciles de observar con fiabilidad en vídeos reales;
- la perspectiva de cámara afecta a la interpretación del gesto;
- algunas señales aparecen durante muy pocos fotogramas;
- la clase central introduce mucha ambigüedad;
- el VLM puede sobreinterpretar detalles poco visibles;
- el LLM no siempre aprovecha bien la información contenida en la memoria visual.

Aun así, las pruebas muestran que la memoria visual generada por el VLM no está vacía. En varias configuraciones contiene información útil, sobre todo cuando se analiza la diferencia entre direcciones laterales.

## Estado del proyecto

Este repositorio recoge el desarrollo experimental del TFG. No es una librería cerrada ni una herramienta final de predicción deportiva.

Su valor principal está en el pipeline desarrollado, en las iteraciones realizadas y en el análisis de las limitaciones de una arquitectura basada en VLM + LLM aplicada a una tarea deportiva rápida, ambigua y difícil de interpretar.
