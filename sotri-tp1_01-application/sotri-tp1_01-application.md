Reporte de Aplicación - SOTRI TP1
Información del Proyecto
Workspace: sotri_workspace
Proyecto: sotri-tp1_01-application
Target MCU: STM32F103RBTx
Archivos Entregados
Core/Src/main.c - Hilo principal y lógica de inicialización.
Core/Src/stm32f1xx_it.c - Manejadores de interrupciones de hardware.
Core/Src/freertos.c - Definición de tareas, semáforos y colas del RTOS.
Core/Inc/FreeRTOSConfig.h - Configuración de tiempos, prioridades y features del kernel.
Core/Startup/startup_stm32f103rbtx.s - Vector de interrupciones y script de inicio en lenguaje ensamblador.
Respuesta de la Plataforma
Reporte de Aplicación - SOTRI TP1
1. Análisis de Estructura de Archivos
A partir de los fuentes cargados al espacio de trabajo sotri_workspace\sotri-tp1_01-application, se identifican los siguientes componentes críticos del firmware:

main.c: Inicializa el hardware base del microcontrolador (reloj, periféricos GPIO, USART2 y TIM2) usando las capas HAL de ST. Define el punto de entrada y lanza el planificador de FreeRTOS mediante la llamada a osKernelStart().
freertos.c: Centraliza la infraestructura operativa del kernel del sistema operativo de tiempo real. Gestiona la memoria estática para la tarea Idle y declara las funciones Hook del sistema (Idle, Tick y Stack Overflow).
2. Configuración y Módulos de Hardware Inicializados
El archivo main.c configura los periféricos de soporte del sistema con los siguientes parámetros:

UART (USART2)
Velocidad (BaudRate): 115200 bps.
Estructura de Trama: 8 bits de datos, 1 bit de parada, sin paridad.
Modo: Transmisión y Recepción activas (TX/RX), sin control de flujo por hardware.
Temporizador de Estadísticas (TIM2)
Propósito: Actúa como la base de tiempo de alta frecuencia requerida para calcular estadísticas de consumo de CPU (configGENERATE_RUN_TIME_STATS).
Manejo de Interrupción: Al dispararse el desbordamiento de TIM2, se ejecuta la rutina de callback HAL_TIM_PeriodElapsedCallback(), incrementando la variable global volátil ulHighFrequencyTimerTicks.
Inicializaciones Externas de la Aplicación
Se detecta el soporte para Semihosting mediante la verificación condicional de LOGGER_CONFIG_USE_SEMIHOSTING que invoca a initialise_monitor_handles().
Se ejecuta de forma temprana app_init() para preparar las tareas de la lógica de negocio antes de iniciar el scheduler.
3. Arquitectura del Kernel FreeRTOS y CMSIS-OS
Tareas Definidas (Threads)
El sistema configura e intenta instanciar una tarea inicializada por defecto:

Nombre: defaultTask
Función de Entrada: StartDefaultTask
Prioridad: osPriorityNormal
Tamaño de Stack: 128 palabras (Words).
Nota de Control: La creación de este hilo se encuentra protegida bajo la macro #ifdef _defaultTask_. Su rutina principal ejecuta un bucle infinito conteniendo un retardo no bloqueante de 1 milisegundo (osDelay(1)).

Gestión de Memoria y Hooks del Kernel
Asignación de Memoria: Implementa vApplicationGetIdleTaskMemory() para proveer almacenamiento estático (StaticTask_t) al stack de la tarea Idle (xIdleStack) garantizando robustez frente a fallos de asignación dinámica.
Seguridad y Monitoreo: Contiene las firmas base listas para depuración avanzada: vApplicationIdleHook(), vApplicationTickHook(), y vApplicationStackOverflowHook().
