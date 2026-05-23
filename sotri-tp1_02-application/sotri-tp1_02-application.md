Reporte de Aplicación - SOTRI TP1 (Paso 02)
Este documento detalla los fundamentos operativos del Kernel de FreeRTOS y el comportamiento del planificador de tareas, analizados y validados mediante la depuración en hardware de la plataforma STM32.

¿Cómo FreeRTOS asigna tiempo de procesamiento a cada Tarea en una aplicación? FreeRTOS implementa un esquema de Multitarea Apropiativa por División de Tiempo (Preemptive Time Slicing). El tiempo de ejecución de la CPU se segmenta en fracciones discretas denominadas Ticks del Sistema.

Base de Tiempo Física: Un temporizador de hardware (el periférico SysTick nativo del núcleo ARM Cortex-M) genera una interrupción periódica exacta. La frecuencia de este reloj está determinada en el archivo FreeRTOSConfig.h mediante la macro configTICK_RATE_HZ (típicamente a 1000 Hz, lo que equivale a 1 tick cada 1 ms). Mecanismo de Despacho:Al dispararse la interrupción de hardware, el flujo salta al manejador del puerto (xPortSysTickHandler). Este incrementa el contador del sistema y evalúa si la tarea actual ha agotado su ventana temporal o si una tarea de mayor prioridad requiere el procesador, gatillando un cambio de contexto mediante la interrupción de software PendSV. void HAL_TIM_PeriodElapsedCallback(TIM_HandleTypeDef htim) { / USER CODE BEGIN Callback 0 */

/* USER CODE END Callback 0 / if (htim->Instance == TIM1) { HAL_IncTick(); } / USER CODE BEGIN Callback 1 */ if (htim->Instance == TIM2) { ulHighFrequencyTimerTicks++; }

/* USER CODE END Callback 1 / } #ifdef defaultTask / definition and creation of defaultTask */ osThreadDef(defaultTask, StartDefaultTask, osPriorityNormal, 0, 128); defaultTaskHandle = osThreadCreate(osThread(defaultTask), NULL); #endif

¿Cómo FreeRTOS elige qué Tarea debe ejecutarse en un momento dado?
EVIDENCIA EN TU CÓDIGO: ¿CÓMO FREERTOS ELIGE QUÉ TAREA EJECUTAR?
[REGLA 1 Y 2]: EVALUACIÓN DE ESTADOS Y MAYOR PRIORIDAD NUMÉRICA
Para elegir la tarea, el planificador busca hilos creados y asignados con prioridades específicas. En tu archivo 'main.c' (líneas 105-113) tienes:

#ifdef defaultTask /* definition and creation of defaultTask */ osThreadDef(defaultTask, StartDefaultTask, osPriorityNormal, 0, 128); defaultTaskHandle = osThreadCreate(osThread(defaultTask), NULL); #endif

¿Qué significa esto en el código?

osPriorityNormal: Es el valor numérico de prioridad asignado a tu tarea. FreeRTOS lee este valor y lo indexa en una lista interna de tareas "Ready".
osThreadCreate(): Registra la tarea en el Kernel. A partir de ahí, el planificador sabe que si 'defaultTask' está "Ready", y no hay otra tarea con prioridad mayor a 'osPriorityNormal', ella debe tomar el control.
[REGLA 3]: RESOLUCIÓN DE EMPATES (ROUND-ROBIN) Y MANEJO DEL TICK HOOK
Cuando las tareas empatan en prioridad, FreeRTOS usa el Tick del sistema para conmutarlas. En tu archivo 'freertos.c' (líneas 171-182) encuentras el rastro:

__weak void vApplicationTickHook( void ) { /* This function will be called by each tick interrupt if configUSE_TICK_HOOK is set to 1 in FreeRTOSConfig.h. User code can be added here, but the tick hook is called from an interrupt context... */ }

¿Qué significa esto en el código? El comentario automático del IDE te lo confirma: "called from an interrupt context". Cada vez que ocurre esta interrupción de hardware (el Tick), el planificador se despierta internamente, revisa si hay tareas del mismo nivel empatadas e interrumpe a la tarea actual para darle el paso a la siguiente de su misma prioridad.

EL DETONADOR DEL PLANIFICADOR: LA ENTRADA EN ACCIÓN
Todo este algoritmo de selección empieza a correr de forma matemática justo en la línea 131 de tu 'main.c':

/* Start scheduler */ osKernelStart();

¿Qué pasa al ejecutar esta línea? El flujo secuencial ordinario del microcontrolador se congela. A partir de este punto, el planificador toma el control total del procesador y aplica de forma cíclica las 3 reglas del reporte para decidir si corre tu 'defaultTask' o la tarea 'Idle' del sistema.
¿Cómo la prioridad relativa de cada Tarea afecta el comportamiento del sistema? Impacto de la Prioridad Relativa La asignación de prioridades determina de manera crítica el orden de ejecución y la asignación de recursos de hardware:

Latencia de Respuesta Determinista: Una tarea con alta prioridad interrumpirá inmediatamente a cualquier tarea de menor prioridad en el instante exacto en que pase al estado Ready (por ejemplo, al completarse un retardo o al recibir una señal de interrupción por hardware). Riesgo de Inanición (Starvation): Si una tarea de alta prioridad ejecuta un bucle infinito sin incluir puntos de bloqueo explícitos (como osDelay() o llamadas pendientes a semáforos/colas), retendrá el control total de la CPU de forma permanente. Esto provoca que las tareas de menor prioridad (e incluso las del mismo nivel si no hay ticks intermedios) nunca reciban tiempo de procesamiento, rompiendo la estabilidad de la aplicación.

¿Cuáles son los estados en los que puede encontrarse una Tarea? Ciclo de Vida y Estados de una Tarea En FreeRTOS, un hilo de ejecución transita exclusivamente por los siguientes cuatro estados operativos: Running (En ejecución): La tarea está haciendo uso activo de los registros y la unidad central de proceso (CPU) del microcontrolador. En arquitecturas mononúcleo, solo una tarea ocupa este estado a la vez. Ready (Lista):La tarea está en condiciones óptimas para ejecutarse, pero se encuentra temporalmente en espera porque el planificador está dando servicio a otra tarea de igual o mayor prioridad. Blocked (Bloqueada):La tarea se ha suspendido voluntariamente a la espera de un evento. Esto ocurre cuando invoca un retardo temporal (osDelay()) o cuando se queda escuchando un recurso de sincronización sin datos disponibles (como una cola vacía o un semáforo tomado). Mientras permanece bloqueada, consume 0% de CPU. Suspended (Suspendida): La tarea se retira explícitamente del mapa de ejecución del planificador mediante la función osThreadSuspend(). No volverá a competir por tiempo de procesamiento hasta que otra entidad invoque de manera directa osThreadResume().

¿Cómo implementar Tareas? Implementar tareas Bajo la capa de abstracción CMSIS-OS V1 integrada en el entorno STM32, una tarea se implementa como una función estándar de C que obligatoriamente debe cumplir con dos restricciones de diseño:

Firma de la Función:Debe retornar un tipo de dato void y recibir estrictamente un único argumento de tipo puntero genérico constante (void const * argument).
Estructura de Bucle Infinito: No puede contener una sentencia de retorno (return) ni llegar al cierre de su llave. Debe incorporar un ciclo infinito (for(;;) o while(1)) y albergar por lo menos una función de bloqueo para liberar el procesador periódicamente.
void StartUserTask(void const * argument) { /* Bloque de inicialización local (Se ejecuta una única vez) */

for(;;) { /* Lógica de control de la aplicación */

// Punto de bloqueo obligatorio para permitir el multiplexado de CPU
osDelay(10); 
} }

Paso 03: Modificar las prioridades relativas asignadas a task_btn y task_led, compilar/depurar/observar comportamiento, asentar lo observado en el archivo sotri-tp1_02-application.md y reestablecer las prioridades relativas asignadas originales.

ret = xTaskCreate(task_btn, "Task BTN", (2 * configMINIMAL_STACK_SIZE), NULL, (tskIDLE_PRIORITY + 2ul), // <--- CAMBIADO DE +1ul A +2ul &h_task_btn);

8. Experimento Paso 03: Modificación de Prioridades Relativas
Se procedió a alterar la prioridad base (tskIDLE_PRIORITY + 1ul) de las tareas task_btn y task_led en el archivo app.c para observar el impacto en el planificador.

Caso A: Prioridad del Botón Mayor a la del LED Configuración:** task_btn = 2, task_led = 1. Comportamiento Observado:** El sistema funciona correctamente. Al ser la tarea del botón la de mayor prioridad, el planificador le otorga la CPU inmediatamente cuando ocurre un evento físico (pulsación). Una vez que el botón valida el antirrebote y cambia el estado de la bandera (cediendo la CPU por sus retardos asíncronos), la tarea del LED asume el control y procesa el parpadeo sin interrupciones visibles.

Caso B: Prioridad del LED Mayor a la del Botón Configuración:** task_btn = 1, task_led = 2. Comportamiento Observado:** El sistema sigue siendo funcional gracias a que la tarea del LED posee un diseño no bloqueante (evalúa el tiempo mediante xTaskGetTickCount y retorna). Sin embargo, si la tarea del LED estuviese procesando el parpadeo (estado Ready/Running) justo en el momento en que se pulsa el botón, FreeRTOS no interrumpirá al LED para leer el GPIO, lo que añade una latencia imperceptible pero real a la lectura del hardware.

Conclusión y Restablecimiento El diseño original (ambas con prioridad 1) aprovecha el algoritmo Round-Robin de FreeRTOS, permitiendo un reparto equitativo de la CPU sin riesgo de inanición. Tras la depuración, las prioridades fueron restablecidas a su valor original (tskIDLE_PRIORITY + 1ul) cumpliendo con los lineamientos del TP.

Experimento Paso 04: Instanciación Múltiple y Eliminación Asíncrona de Tareas

Configuración del Experimento Se crearon tres instancias independientes (Task BTN 1, Task BTN 2 y Task BTN 3) mapeadas a la misma función de ejecución task_btn en app.c, compartiendo el mismo nivel de prioridad (tskIDLE_PRIORITY + 1ul). Se modificó la tarea task_led para interceptar y destruir la instancia Task BTN 3 mediante la directiva vTaskDelete(h_task_btn3) tras la primera transición al estado de parpadeo.

Comportamiento Observado e Impacto en la Concurrencia

Conflicto por Estructuras de Datos Globales Compartidas (Condición de Carrera) Al ejecutarse las tres instancias en paralelo bajo la política Round-Robin, se evidenció un comportamiento anómalo en la detección del botón. Dado que la lógica interna en task_btn.c opera modificando una única estructura global de datos estática (task_btn_dta), las tres tareas leen y alteran las variables .state, .event y .tick simultáneamente de forma desordenada.
Efecto: Cuando una de las instancias detecta un flanco de bajada y cambia el estado a ST_BTN_FALLING, la siguiente instancia en tomar la CPU lee ese estado modificado alterando el temporizador de antirrebote (.tick). Esto corrompe la temporización no bloqueante, generando lecturas falsas o ignorando pulsaciones del usuario debido a la falta de reentrancia.
Proceso de Eliminación por el LED Al presionar el botón físico por primera vez y lograr una transición válida, la tarea del botón levantó la bandera global mediante put_event_task_led(EV_LED_BLINK).
Al cambiar de contexto a Task LED, el sistema ejecutó la llamada a vTaskDelete(h_task_btn3).
A través de las trazas del depurador, se constató que el Kernel de FreeRTOS removió inmediatamente el Bloque de Control de Tarea (TCB) de Task BTN 3 de las listas de tareas listas (pxReadyTasksLists), liberando los recursos de su pila (Stack) en la siguiente ejecución de la tarea Idle.
Tras la eliminación, la consola dejó de registrar actividad asociada a la tercera instancia, permaneciendo únicamente las instancias 1 y 2 en ejecución.
Conclusión Técnica
El experimento demuestra que FreeRTOS permite instanciar de forma múltiple una misma función lógica de C. Sin embargo, si dicha función no está diseñada bajo principios de código reentrante (utilizando variables locales dentro del Stack de cada tarea o estructuras independientes pasadas a través del parámetro), se introducen errores críticos de concurrencia debido al acceso compartido de recursos globales no protegidos.
