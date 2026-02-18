## Explicación codigo en C

Este código implementa la captura de señales biomédicas (Parte B de la guía) utilizando un microcontrolador STM32.

### 1

Se definen variables globales para el manejo de la información capturada:

- `uint32_t datos`: Almacena la conversión actual del ADC asistida por DMA.
- `uint8_t datosenvio[50]`: Buffer de transmisión que acumula 25 muestras (cada una de 2 bytes) antes de enviarlas por USB.
- `int contador`: Índice que rastrea el llenado del buffer para asegurar que siempre se envíen paquetes completos de 50 bytes.

### 2

En la función `main`, se inicializan los módulos críticos:

- `MX_ADC1_Init()`: Configura el ADC1 para trabajar con disparador externo del Timer 3 (`ADC_EXTERNALTRIGCONV_T3_TRGO`), permitiendo un muestreo preciso.
- `MX_TIM3_Init()`: Establece la frecuencia de muestreo. El Timer 3 actúa como el "reloj" que le indica al ADC cuándo tomar una muestra.
- `MX_USB_DEVICE_Init()`: Prepara la interfaz USB CDC (Puerto COM Virtual) para enviar los datos a la computadora.

### 3
(`HAL_ADC_ConvCpltCallback`)

Esta es la parte más importante del algoritmo, ya que es donde se gestiona la interrupción por conversión completa:

#### 4
Como el ADC entrega valores de 12 bits y el protocolo USB transmite datos en bytes (8 bits), cada muestra se divide en dos:

```c
datosenvio[contador++] = datos[0] & 0xFF;
datosenvio[contador++] = (datos[0] >> 8) & 0xFF;

- El primer byte guarda los 8 bits menos significativos.
- El segundo byte guarda los bits restantes desplazados.

### Transmisión por USB: 

Cuando el contador llega a 50 bytes (25 muestras):
CDC_Transmit_FS(datosenvio, 50);
Se envía el paquete completo a Python para su posterior análisis estadístico (media, desviación estándar, etc.).

## Paso a paso del codigo: 

1. TIM3 genera un evento de actualización.
2. ADC1 realiza la conversión de la señal analógica.
3. DMA mueve automáticamente el resultado a memoria.
4. La interrupción procesa la muestra.
5. Al completar 25 muestras, se envía el paquete por USB.
