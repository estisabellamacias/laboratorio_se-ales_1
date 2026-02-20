# **Análisis estadístico de señales**

## *Contexto teorico* 
El análisis de señales biomédicas permite extraer información fisiológica relevante mediante la identificación de puntos fiduciales, como el pico "R" en un electrocardiograma. Sin embargo, una alternativa robusta es el cálculo de características no fiduciales, las cuales analizan la señal como un todo en una ventana de tiempo sin depender de puntos específicos. Estas características se representan mediante magnitudes estadísticas fundamentales: la media (valor promedio), la desviación estándar y varianza (dispersión), la asimetría (inclinación de la distribución) y la curtosis (pronunciamiento del pico de la distribución). El estudio de estos parámetros es crucial, ya que pueden variar debido a condiciones patofisiológicas (como hipertensión o diabetes) o por la contaminación de la señal con ruidos y artefactos.

## *Objetivos* 
***Objetivo General:*** Caracterizar una señal biomédica en función de sus parámetros estadísticos.

***Objetivos especificos:***
1. Identificar las principales magnitudes estadísticas que describen el comportamiento de una señal fisiológicas.
2. Utilizar entornos de programación (como Python) y funciones aritméticas para calcular estos parámetros de forma manual y predefinida.
3. Plantear hipótesis desde la fisiología que logren explicar los valores estadísticos obtenidos en las mediciones.
4. Evaluar la calidad de la señal mediante el cálculo de la relación señal-ruido (SNR) tras contaminarla con diferentes tipos de ruido.

## *Desarrollo de la pratica*

***1.Calculo estadisticos descriptivos y SNR:*** 

**1. Sincronización y Carga de Datos (Partes A y B)**

El código establece una ventana de observación de 8 segundos para asegurar que la comparación entre la señal sintética y la real sea válida.

*- PhysioNet (Parte A):* Utiliza la librería `wfdb` para leer registros profesionales (`.hea` y `.dat`). El script busca automáticamente el canal de ECG y calcula cuántas muestras equivalen a 8 segundos basándose en la frecuencia de muestreo (`fs`) original del archivo.

*- STM32 (Parte B):* Importa los datos capturados por el hardware desde un archivo `.txt` usando `np.loadtxt`. Para esta señal, se asume una fs de 100 Hz, lo que resulta en un bloque de 800 muestras para el análisis.

**2. Normalización Unitaria**

Para que las señales sean comparables morfológicamente, sin importar si vienen de una base de datos o de un ADC de 12 bits, el código aplica un proceso de normalización:

*- Centrado:* Resta la media de la señal para eliminar el offset o desplazamiento de continua, situando la línea base en `0.0`.

*- Escalado:* Divide toda la señal por su valor máximo absoluto, forzando a que los datos oscilen estrictamente en el rango de `[-1, 1]`.

**3. Momentos Estadísticos: Manual vs. Funciones**

Cumpliendo con el requerimiento de programar las fórmulas "desde cero", el código define la función `calcular_estadisticos_manual`:

- Media (`μ`): Calcula el promedio o valor central de la señal. `(media = sum(datos) / n )`

- Desviación Estándar (`σ`): Mide la dispersión de los valores. El código utiliza la corrección de sesgo (`n−1`) para obtener la desviación muestral exacta. `(desv = math.sqrt(sum_d2 / (n - 1)))`

- Asimetría (Skewness) y Curtosis: Implementa el tercer y cuarto momento central para determinar hacia dónde se inclina la distribución y qué tan pronunciado es su pico. `((sum_d3 / n) / (desv**3))` -  `(kurt = (sum_d4 / n) / (desv**4))`

- Validación: El script utiliza `numpy` y `scipy.stats` para verificar que los cálculos manuales sean idénticos a los de las funciones predefinidas de Python. 

Para las funciones espcificas de Phyton se utilizaron las siguientes:

- Media (`μ`): `np.mean(data)`
  
- Desviación Estándar (`σ`): Para que este cálculo coincida con la fórmula manual que usa n−1, es necesario configurar un parámetro adicional en `Numpy. np.std(data, ddof=1)`

- Asimetría (Skewness): `stats.skew(data, bias=False)`. Al poner `bias=False`, la función ajusta el cálculo para que sea estadísticamente exacto para una muestra, lo que permite compararlo directamente con tu lógica manual.

- Curtosis: `stats.kurtosis(data, fisher=False, bias=False)`
  
**4. Histogramas**

El código genera histogramas con 30 contenedores (`bins`) para visualizar cómo se distribuyen las amplitudes de la señal:

- Método Manual: Clasifica cada dato en su "caja" correspondiente mediante un ciclo `for`.

- Método Automatizado: Utiliza `plt.hist` para validar la distribución visual, permitiendo comparar la señal de PhysioNet frente a la de la STM32.

**5. Análisis de SNR y Contaminación (Parte C)**

El código evalúa la robustez de la señal mediante la Relación Señal-Ruido (SNR), calculada en decibelios (dB). Para ello, contamina la señal de la STM32 con tres tipos de interferencias clínicas:

*- Ruido Gaussiano:* Simula ruido térmico electrónico con un promedio de 0 y una desviación de 0.08.

*- Ruido Impulso:* Modela interferencias súbitas (como fallos de electrodos) inyectando picos aleatorios de gran amplitud (±1.2) que sobresalen de la señal normalizada.

*- Ruido de Artefacto:* Simula la deriva de línea base causada por la respiración, sumando ondas senoidales de baja frecuencia (0.1 y 0.4 Hz).

**6.Tablas comparativas**

El codigo finaliza imprimiendo tablas comparativas en la consola que muestran los resultados de ambos métodos (manual vs. función) y el impacto de cada ruido en el SNR. 


***2. Captura de datos con la STM32 y el generador de señales: (Parte B)***

Este código implementa la captura de señales biomédicas utilizando un microcontrolador en este caso la STM32.

**Estructura de Datos**

Se definen variables globales para el manejo de la información capturada:

- `uint32_t datos`: Almacena la conversión actual del ADC asistida por DMA.
- `uint8_t datosenvio[50]`: Buffer de transmisión que acumula 25 muestras (cada una dividida en 2 bytes, sumando 50 bytes) antes de enviarlas por USB.
- `int contador`: Índice que rastrea el llenado del buffer para asegurar que siempre se envíen paquetes completos de 50 bytes.

**Configuración de Periféricos (HAL)**

En la función `main`, se inicializan los módulos críticos:

- `MX_ADC1_Init()`: Configura el ADC1 para trabajar con disparador externo del Timer 3 (`ADC_EXTERNALTRIGCONV_T3_TRGO`), permitiendo un muestreo preciso.
- `MX_TIM3_Init()`: Establece la frecuencia de muestreo. El Timer 3 actúa como el "reloj" que le indica al ADC cuándo tomar una muestra.
- `MX_USB_DEVICE_Init()`: Prepara la interfaz USB CDC (Puerto COM Virtual) para enviar los datos a la computadora.

**Lógica de Captura y Procesamiento**
(`HAL_ADC_ConvCpltCallback`)

Esta es la parte más importante del algoritmo, donde se gestiona la interrupción por conversión completa:

**• División de Bytes (Byte Splitting)**

Como el ADC entrega valores de 12 bits y el protocolo USB transmite datos en bytes (8 bits), cada muestra se divide en dos:

```c
datosenvio[contador++] = datos[0] & 0xFF;
datosenvio[contador++] = (datos[0] >> 8) & 0xFF;
```
- El primer byte guarda los 8 bits menos significativos.
- El segundo byte guarda los bits restantes desplazados.

**• Transmisión por USB:** Cuando el contador llega a 50 bytes (25 muestras), se dispara la función `CDC_Transmit_FS(datosenvio, 50)`; enviando el paquete a la computadora para su posterior análisis estadístico (media, desviación estándar, etc.) en Python. 

***3. Recoleccion datos de la STM32 (Parte B)***

Este codigo implementa una Interfaz Gráfica de Usuario (GUI) que permite la visualización en tiempo real y la captura de señales biomédicas provenientes del microcontrolador STM32.

Es la herramienta principal utilizada para generar los archivos de datos necesarios para el análisis estadístico posterior.

**Estructura**

La aplicación está desarrollada sobre librerías especializadas en alto rendimiento y procesamiento de datos:

• `PyQt5`: Proporciona la estructura de la interfaz gráfica y la gestión de eventos del usuario (botones, selección de puerto, temporizadores, etc.).

• `pyqtgraph`: Utilizada para el renderizado de la señal en tiempo real.  Se elige por su alta eficiencia en la visualización de grandes volúmenes de datos con bajo consumo de CPU.

• `pyserial`: Gestiona la comunicación serial a través del puerto USB (Virtual COM Port) con la STM32, configurada a una velocidad de **115200 baudios**.

• `numpy`: Permite la manipulación eficiente de arreglos de datos y la reconstrucción de las muestras digitales provenientes del ADC.

**Componentes del Código**

• *Gestión de Conexión – `toggle_connection()`*

- Escanea dinámicamente los puertos disponibles.
- Permite seleccionar el puerto COM correspondiente a la STM32.
- Al establecer conexión, inicia un `QTimer`.
- El temporizador ejecuta la lectura de datos cada 10 milisegundos, garantizando una visualización fluida y continua.

*• Procesamiento de Paquetes – `update_data()`*

  - Sincronización: El programa espera hasta que haya al menos 50 bytes disponibles en el buffer de entrada.  
Esto asegura que se procesen paquetes completos equivalentes a 25 muestras.

  - Reconstrucción de Datos: Se utiliza `np.frombuffer()` para convertir los bytes binarios nuevamente en valores enteros de 12 bits `uint16`, recuperando así los datos originales del ADC.

• *Visualización Dinámica:* La gráfica muestra constantemente las últimas 1000 muestras recibidas, permitiendo monitorear la estabilidad y morfología de la señal biomédica capturada

• *Grabación de Datos `(toggle_recording)`:* Permite exportar la señal en tiempo real a un archivo .txt. Cada muestra se guarda en una línea nueva, facilitando su importación posterior para los cálculos de media, desviación y otros estadísticos requeridos en la guía


## **Imagenes**

Las siguientes imagen presentan los resultaos obtenidos:

**Señal de PhysioNet comparada con la señal de la STM32**

<img src="Graficas y tablas/Señal physionet y stm32.jpeg" width="600">

<em>Grafica 1. Comparacion de señales PhysioNet y STM32.</em>

**Histograma manual e histograma funciones Python (Señal PhysioNet)**

<img src="Graficas y tablas/histograma physionet.jpeg" width="600">

<em>Grafica 2. Comparacion de histogramas de la señal de PhysioNet.</em>

**Histograma manual e histograma funciones Python (Señal STM32)**

<img src="Graficas y tablas/Histograma stm.jpeg" width="600">

<em>Grafica 3.Comparacion de histogramas de la señal de la STM32 .</em>

**Señal con ruido Gaussiano**

<img src="Graficas y tablas/señal con ruido Gaussiano.jpeg" width="600">

<em>Grafica 4. Señal con ruido Gaussiano.</em>

**Señal con ruido impulso**

<img src="Graficas y tablas/señal con ruido Gaussiano.jpeg" width="600">

<em>Grafica 5. Señal con ruido Gaussiano.</em>

**Señal con ruido artefacto de movimiento**

<img src="Graficas y tablas/señal con ruido Gaussiano.jpeg" width="600">

<em>Grafica 6. Señal con ruido Gaussiano.</em>

**Tabla de reporte del SNR de los diferentes ruidos**

<img src="Graficas y tablas/Reporte SNR.jpeg" width="600">

<em>Tabla1. Tabla con el reporte del SNR .</em>

**Tabla comparativa calculos estadisticos**

<img src="Graficas y tablas/Tabla comparativa.jpeg" width="600">

<em>Tabla2. Tabla comparativa resultados calculos estadisticos.</em>


## **Diagramas de flujo**

Los siguientes diagramas representan la lógica algorítmica implementada en cada módulo del sistema:

***1. PYTHON IMPRIMIR GRAFICAS - HACER CALCULOS ESTADISTICOS - SNR***

<img src="diagramas/Diagrama PYTHON IMPRIMIR GRAFICAS Y HACER CALCULOS ESTADISTICOS.png" width="600">

<em>Figura 1. Flujo algorítmico del análisis estadístico aplicado a las señales adquiridas, incluyendo normalización, estimación de parámetros descriptivos y evaluación de contaminación por ruido.</em>

***2. CAPTURA DE DATOS STM32:***

<img src="diagramas/Diagrama CAPTURA DE DATOS STM32.png" width="600">

<em>Figura 2. Diagrama de flujo del firmware implementado en la STM32.</em>

***3. PYTHON RECOLECCION DATOS STM32***

<img src="diagramas/Diagrama PYTHON RECOLECCION DATOS STM32.png" width="600">

<em>Figura 3. Flujo funcional de la interfaz gráfica encargada de la recepción de datos de la STM32</em>


## **Análisis de Resultados y Preguntas de Discusión**

**1. Determine el alcance y las posibles limitaciones de emplear parámetros estadísticos para detectar patologías en seres humanos.**

*Alcance:* El uso de parámetros estadísticos (media, desviación estándar, asimetría, curtosis, varianza, entre otros) permite:

- Cuantificar objetivamente características globales de una señal biomédica.

- Identificar variabilidad asociada a posibles alteraciones fisiológicas.

- Facilitar la comparación entre pacientes o entre estados fisiológicos (reposo vs. estrés, sano vs. patológico).

*Limitaciones:* Sin embargo, existen limitaciones importantes:

- Los parámetros estadísticos pierden información temporal (no consideran la morfología específica de ondas como P, QRS o T en ECG).

- No distinguen entre patología real y ruido si no hay preprocesamiento adecuado.

- Aunque sirven de apoyo, no sustituyen análisis clínico ni diagnóstico médico.

En conclusión, los parámetros estadísticos son útiles como indicadores preliminares, pero no son suficientes por sí solos para diagnosticar patologías.

**2. Determine el alcance y las posibles limitaciones de emplear parámetros estadísticos para evaluar la calidad de una señal biomédica.**

*Alcance:* En la evaluación de calidad de la señal, los parámetros estadísticos son más útiles porque:

- Permiten medir nivel de ruido (aumento de varianza).

- Detectan deriva de línea base (cambio en la media).

- Identifican presencia de picos anómalos (aumento de curtosis).

En este caso son herramientas muy efectivas para determinar si la señal es apta para análisis posterior.

*Limitaciones:* 
- No identifican el tipo específico de ruido sin análisis adicional.

- Pueden dar valores similares en señales con morfologías distintas.

- No evalúan directamente la fidelidad fisiológica del contenido.

**3. ¿Los valores estadísticos calculados sobre la señal sintética son exactamente iguales a los obtenidos a partir de la señal real? ¿Por qué?**

No, no son iguales. Aunque ambas representan el mismo fenómeno (ECG), sus propiedades estadísticas difieren por las siguientes razones técnicas:

- Pureza vs. Ruido Ambiental: La señal de la STM32, al estar en condiciones ideales, presenta una línea base perfectamente plana y picos definidos sin fluctuaciones térmicas. Esto resulta en una Media prácticamente en cero y una Desviación Estándar que solo mide la energía de los complejos QRS. En cambio, la de PhysioNet contiene ruido electromagnético y biológico real, lo que "ensucia" la distribución y modifica la varianza.

- Morfología y Momentos Superiores (Asimetría y Curtosis):

*Asimetría (Skewness):* La señal de PhysioNet tiene una asimetría menor (2.78) comparada con la del STM32 (4.16). Esto indica que en la señal ideal del STM32, los picos R son mucho más prominentes y limpios respecto al resto de la señal, mientras que en la real, el ruido "rellena" los valles y suaviza esa inclinación hacia los valores positivos.

*Curtosis:* La curtosis de la STM32 es más alta (22.23) que la de PhysioNet (17.75). Una curtosis elevada (leptocúrtica) significa que la señal tiene picos muy agudos y extremos. La señal ideal del STM32 tiene ondas R muy finas y "puntiagudas", mientras que en la señal real, el ruido ensancha la base de la señal, reduciendo su "picudez" estadística.

- Cuantización y Muestreo: La señal de la STM32 está limitada por la resolución de su ADC (10 o 12 bits) y una frecuencia de 100 Hz, mientras que PhysioNet usa equipos médicos de alta gama con mayor tasa de muestreo, capturando detalles finos que la estadística detecta como variaciones de alta frecuencia.
  
**4. ¿Afecta el tipo de ruido el valor de la SNR calculado? ¿Cuáles podrían ser las razones?** 

Sí, el tipo de ruido afecta significativamente el SNR porque cada uno ataca la señal con una energía y una distribución temporal distinta. La relación Señal-Ruido (SNR) es inversamente proporcional a la potencia del ruido: si el ruido tiene mucha amplitud, el SNR cae.

*Análisis por tipo de ruido según los resultados:*
- Ruido Gaussiano (SNR: 6.50 dB): Este ruido es de baja amplitud (desviación de 0.08) y se distribuye uniformemente por toda la señal. Aunque "borronea" la señal, su potencia total es pequeña comparada con la potencia de los picos del ECG. Es el ruido típico de los componentes electrónicos del STM32.

- Ruido de Impulso (SNR: 2.06 dB): A diferencia del anterior, este ruido no es constante, sino que son "golpes" de gran magnitud (picos de 1.2). Aunque ocurren pocas veces, cada impacto tiene muchísima energía. Al promediar la potencia del ruido en la fórmula del SNR, estos pocos picos pesan tanto que degradan la relación mucho más que el ruido gaussiano constante. 

- Artefacto de Movimiento (SNR: -4.84 dB): Es el único con SNR negativo, lo que significa que el ruido tiene más potencia que la propia señal de ECG. Al ser una onda senoidal de baja frecuencia y gran amplitud (0.4), desplaza toda la línea base hacia arriba y abajo. En nuestro caso (Ing.biomédica), esto es crítico porque la respiración o movimiento del paciente oculta por completo la morfología de la señal, haciendo que la energía del error sea mucho mas grande.

