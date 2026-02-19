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
***Parte A:***  Selección de la señal de Physionet, cómo se importó a Python y la comparación entre el cálculo manual de estadísticos y el uso de funciones predefinidas. 

***Parte B:***

*1. Captura de datos con la STM32 y el generador de señales:*

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

*2. Recoleccion datos de la STM32*

Este script implementa una Interfaz Gráfica de Usuario (GUI) que permite la visualización en tiempo real y la captura de señales biomédicas provenientes del microcontrolador STM32.

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

***Parte C:*** Definición de SNR y cómo afectó cada tipo de ruido (gaussiano, impulso y artefacto) a la señal biomédica.

## **Diagramas de flujo**

***1. PYTHON IMPRIMIR GRAFICAS Y HACER CALCULOS ESTADISTICOS***

***2. CAPTURA DE DATOS STM32:***

***3. PYTHON RECOLECCION DATOS STM32***

## **Análisis de Resultados y Preguntas de Discusión**

Determine el alcance y las posibles limitaciones de emplear parámetros
estadísticos para detectar patologías en seres humanos.
Determine el alcance y las posibles limitaciones de emplear parámetros
estadísticos para evaluar la calidad de una señal biomédica. 

- ¿Los valores estadísticos calculados sobre la señal sintética son
exactamente iguales a los obtenidos a partir de la señal real? ¿Por qué?
- ¿Afecta el tipo de ruido el valor de la SNR calculado? ¿Cuáles podrían ser
las razones? 

## **Conclusiones**

