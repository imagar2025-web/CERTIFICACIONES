# 20 · Monitor activities in Microsoft Fabric

> **DP-700 · Manage Microsoft Fabric.** [Módulo oficial](https://learn.microsoft.com/en-us/training/modules/monitor-fabric-items/) · [fuentes](https://github.com/MicrosoftDocs/learn/tree/main/learn-pr/wwl/monitor-fabric-items) · [laboratorio oficial complementario](https://github.com/MicrosoftLearning/mslearn-fabric/blob/main/Instructions/Labs/18-monitor-hub.md). **Revisión:** noviembre de 2026. Laboratorio sin ejecutar.

## Índice

1. [Introducción](#1-introducción)
2. [Understand monitoring](#2-understand-monitoring)
3. [Monitoring hub](#3-monitoring-hub)
4. [Take action with Activator](#4-take-action-with-activator)
5. [Laboratorio complementario](#5-laboratorio-complementario)
6. [Knowledge check y summary](#6-knowledge-check-y-summary)
7. [Fuentes y control](#7-fuentes-y-control)

## 1. Introducción

Una solución de Data Engineering puede fallar en distintas etapas: un archivo no llega, un Pipeline pierde acceso al origen, un Notebook devuelve un error Spark o un Dataflow transforma una columna de forma inesperada. La monitorización de Fabric ayuda a conocer el **estado operativo** de esos procesos, investigar fallos y responder ante condiciones relevantes. No debe confundirse con el control de calidad: un run exitoso puede producir datos incorrectos.

Microsoft Fabric centraliza información de ejecución en el **Monitoring hub**, siempre dentro de los permisos efectivos de la identidad. El módulo también describe el uso de **Activator** para actuar automáticamente ante ciertos eventos de trabajos. Para convertir estos mecanismos en una estrategia productiva hay que definir indicadores, objetivos, umbrales, responsables y acciones de recuperación.

## 2. Understand monitoring

### 2.1. Qué observar

En una solución analítica son relevantes al menos cuatro grupos de indicadores: **disponibilidad de fuentes** (¿llega el fichero?), **estado de ejecuciones** (¿completó la actividad?), **rendimiento** (¿cuánto tardó y cuánto consumió?) y **calidad/frescura** (¿los datos son correctos y están actualizados?). Estos grupos necesitan herramientas distintas y deben correlacionarse mediante identificadores de lote y run.

Una ingesta diaria puede cumplir el estado técnico de éxito, pero cargar un CSV del día anterior. Sin una comprobación de frescura, la operación parece sana y el negocio utiliza información desactualizada. Conviene registrar campos como \`batch_id\`, \`source_updated_at\`, \`ingested_at\`, \`row_count\`, \`status\` y el identificador de ejecución de Pipeline.

### 2.2. Baseline y alertas útiles

Una **baseline** indica comportamiento normal: tiempos de ejecución, cantidades de registros y horarios habituales. Sin ella es fácil confundir un pico esperado con una anomalía. Un buen umbral de alerta evita notificar cada desviación irrelevante y distingue entre warnings y fallos que afectan a consumidores.

Por ejemplo, un Pipeline que normalmente termina en seis minutos podría necesitar investigación cuando supera los veinte. Sin embargo, la duración no basta: si el volumen de datos se multiplicó por cinco, el incremento puede ser coherente con la carga. Analizar simultáneamente tiempos, recursos y datos.

### 2.3. Riesgos de observabilidad

Los logs y trazas pueden contener nombres de tablas, rutas y mensajes de error con información confidencial. Limitar quién puede consultarlos y evitar incluir datos personales o credenciales en parámetros y mensajes de depuración. La monitorización también tiene costes operativos y no debe almacenar información indefinidamente sin políticas de retención.

**Fuente:** [Understand monitoring](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/monitor-fabric-items/includes/2-understand-monitoring.md).

## 3. Monitoring hub

### 3.1. Visión centralizada

El **Monitoring hub** presenta actividades de distintos elementos para las que el usuario está autorizado: pipelines, notebooks, dataflows y otras cargas compatibles. Permite examinar estado, fecha, duración e historial según el tipo de actividad. La visibilidad depende de permisos; no ver un run no demuestra que no exista.

### 3.2. Filtrar y diagnosticar

En un entorno con muchas actividades, conviene filtrar por estado, tipo, Workspace e intervalo temporal. Para investigar un fallo, comenzar por la actividad concreta y su mensaje de error; después revisar dependencias y las entradas y salidas pertinentes. Si la carga de un Pipeline contiene una actividad de Notebook, el estado del Pipeline no sustituye la revisión de la ejecución Spark.

Un error de autenticación del origen no se arregla aumentando recursos Spark. Un error de falta de memoria en un Notebook no se investiga como si fuera una URL HTTP mal escrita. La monitorización útil relaciona el síntoma con el componente responsable.

### 3.3. Historial y retención

El historial permite comparar ejecuciones, pero las ventanas de retención dependen del sistema y su configuración. Para auditoría de largo plazo puede ser necesario recopilar registros de ejecución en un sistema propio autorizado. No asumir que el Monitoring hub conserva indefinidamente todos los datos operativos.

~~~mermaid
flowchart LR
  P["Pipeline / Dataflow / Notebook"] --> H["Monitoring hub"]
  H --> I{"Estado / duración"}
  I -->|OK| V["Validación de datos"]
  I -->|Fallo| D["Diagnóstico por actividad"]
  D --> R["Corrección y reintento controlado"]
  V --> Q{"¿Calidad correcta?"}
  Q -->|Sí| F["Publicar / confirmar"]
  Q -->|No| R
~~~

*Diagrama propio.* Validación de datos y reintento controlado requieren lógica adicional del equipo.

**Fuente:** [Monitoring hub](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/monitor-fabric-items/includes/3-use-monitor-hub.md).

## 4. Take action with Activator

Activator puede observar eventos de trabajos de Fabric mediante las integraciones disponibles, incluida la entrada de eventos del **Real-Time hub**, y activar notificaciones o acciones. Una regla puede detectar que un Pipeline falla y comunicarlo por correo o Teams, o desencadenar un proceso de respuesta cuando la característica y los permisos lo permiten.

### 4.1. Alertas de fallo

Una alerta debe incluir nombre del recurso, Workspace, momento e identificador de ejecución, sin exponer secretos. Si falla una actividad crítica, la notificación debería dirigirse al equipo propietario. No habilitar acciones de recuperación que reintenten indefinidamente una carga defectuosa: pueden aumentar costes y duplicar datos.

### 4.2. Idempotencia de la respuesta

Si un run produce múltiples eventos de fallo, una regla sin control puede generar varias incidencias. Conviene deduplicar por \`run_id\` y condición, o aplicar una política de correlación. Cuando se reanuda una ejecución, revisar qué operaciones ya persistieron resultados. Una reparación automática no debe volver a insertar datos sin estrategia de idempotencia.

### 4.3. Activator frente a monitoring hub

El Monitoring hub permite **observar e investigar**; Activator permite **detectar condiciones y actuar**. Un dashboard no sustituye una regla, y una regla no sustituye el historial de diagnóstico. Son herramientas complementarias.

**Fuente:** [Take action with Activator](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/monitor-fabric-items/includes/4-take-action-with-activator.md).

## 5. Laboratorio complementario

**Fuente:** [Monitor Fabric activity in the monitoring hub](https://github.com/MicrosoftLearning/mslearn-fabric/blob/main/Instructions/Labs/18-monitor-hub.md). **Duración aproximada:** 30 minutos. El laboratorio es oficial de MicrosoftLearning y se utiliza como complemento del módulo; comprobar los metadatos de cursos para la clasificación exacta. **No ejecutado**.

### 5.1. Objetivo y requisitos

Crear un Workspace, Lakehouse, Dataflow Gen2 y Notebook Spark y observar su actividad desde Monitoring hub. Después explorar el historial de un elemento y personalizar la vista del hub. Requiere capacidad Fabric compatible y permisos de creación y acceso a los elementos.

### 5.2. Preparar los recursos

Entrar en [Fabric](https://app.fabric.microsoft.com/), crear Workspace de laboratorio y añadir un Lakehouse. Crear un Dataflow Gen2 de prueba conforme al ejercicio y ejecutarlo, anotando su nombre. Crear también un Notebook, asociarlo al Lakehouse y ejecutar las celdas del tutorial. No presupongas que ambos tipos de actividad generan las mismas métricas o campos de historial.

### 5.3. Monitorizar Dataflow y Notebook

Abrir el Monitoring hub y localizar las ejecuciones. Filtrar por tipo, hora o estado. Inspeccionar el detalle de un run, incluyendo inicio, fin y resultado. Si una ejecución no aparece, verificar permisos, tiempo de ejecución y tipo de actividad. No inventar duraciones o recuentos de filas.

### 5.4. Consultar historial

Desde el elemento, examinar sus runs históricos y compararlos con la vista centralizada. Esta comparación ayuda a entender la relación entre monitorización individual y global. Un estado de éxito del Notebook no implica que la tabla destino contenga registros correctos: comprobar también los datos cuando proceda.

### 5.5. Personalizar la vista

Aplicar filtros y opciones de personalización para mostrar únicamente el subconjunto de actividades de interés. Registrar qué configuración permite aislar fallos de Notebooks frente a Dataflows.

### 5.6. Errores y limpieza

| Problema | Comprobación |
|---|---|
| El run no aparece | Permisos, filtros y fecha |
| No puede abrir detalles | Rol e identidad |
| Notebook en ejecución prolongada | Estado de sesión Spark y duración |
| Dataflow falla | Conexión, tipos o destino |
| Los resultados de la vista difieren | Comparar intervalos y filtros |
| Alerta repetida | Correlación e idempotencia |

Eliminar el Workspace de prueba o sus elementos, detener actividades y revisar capacidad.

## 6. Knowledge check y summary

Preguntas propias: ¿Qué diferencia hay entre Monitoring hub y calidad del dato? ¿Para qué sirve una baseline? ¿Por qué un run exitoso puede producir datos obsoletos? ¿Qué identifica el run ID? ¿Cómo se filtran actividades? ¿Qué función tiene Activator? ¿Qué riesgos presenta una reparación automática sin idempotencia?

**Summary:** la monitorización efectiva combina visibilidad centralizada, diagnóstico detallado, datos de auditoría y alertas accionables.

## 7. Fuentes y control

- [index.yml, siete unidades](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/monitor-fabric-items/index.yml).
- [Fundamentos](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/monitor-fabric-items/includes/2-understand-monitoring.md).
- [Monitoring hub](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/monitor-fabric-items/includes/3-use-monitor-hub.md).
- [Activator](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/monitor-fabric-items/includes/4-take-action-with-activator.md).
- [Laboratorio oficial complementario](https://github.com/MicrosoftLearning/mslearn-fabric/blob/main/Instructions/Labs/18-monitor-hub.md).

**Control:** teoría, práctica y evaluación desarrolladas; no ejecutadas.