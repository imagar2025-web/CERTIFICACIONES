# 17 · Monitor a Microsoft Fabric data warehouse

> **DP-700 · Warehouse.** [Microsoft Learn](https://learn.microsoft.com/en-us/training/modules/monitor-fabric-data-warehouse/) · [fuentes](https://github.com/MicrosoftDocs/learn/tree/main/learn-pr/wwl/monitor-fabric-data-warehouse) · [laboratorio DP-700](https://github.com/MicrosoftLearning/mslearn-fabric/blob/main/Instructions/Labs/06c-monitor-data-warehouse.md). **Revisión:** noviembre de 2026. Documentación sin ejecución del laboratorio.

## Índice

1. [Introducción](#1-introducción)
2. [Capacity metrics](#2-capacity-metrics)
3. [Dynamic Management Views](#3-dynamic-management-views)
4. [Query insights](#4-query-insights)
5. [Laboratorio oficial](#5-laboratorio-oficial)
6. [Knowledge check y summary](#6-knowledge-check-y-summary)
7. [Fuentes](#7-fuentes)

## 1. Introducción

Un Warehouse puede devolver consultas correctas y, aun así, incumplir los objetivos operativos: una consulta que tarda treinta minutos, una carga que compite con otros procesos o una capacidad saturada pueden afectar a todo el ecosistema de datos. **Monitorizar** no consiste solamente en comprobar que las ejecuciones terminan en verde, sino en medir latencia, consumo de recursos, volumen de datos, errores y tendencias.

Microsoft Fabric ofrece mecanismos complementarios. La **Capacity Metrics app** muestra consumo de recursos compartidos; las **Dynamic Management Views (DMVs)** exponen el estado de conexiones, sesiones y solicitudes; y **Query insights** proporciona información histórica para analizar consultas ejecutadas. Es importante comprender la escala temporal y el propósito de cada herramienta: un instante de alta actividad no explica por sí solo todo el consumo de un mes.

## 2. Capacity metrics

### 2.1. Capacity Units

Las **Capacity Units (CUs)** representan recursos consumidos por cargas de Fabric, incluida la ejecución de consultas, cargas y otras operaciones. La facturación y el comportamiento concreto de capacidad dependen del SKU y las opciones contratadas. El equipo necesita observar el consumo para distinguir una carga optimizable de una capacidad mal dimensionada.

La **Microsoft Fabric Capacity Metrics app** permite analizar la utilización por cargas y períodos. El administrador debe revisar no solo cuánto consume un Warehouse, sino si se producen picos coincidentes con otros trabajos. Si varios Pipelines inician cargas Spark y SQL a la misma hora, una degradación de rendimiento puede ser producto de contención de recursos en lugar de una regresión en la query.

### 2.2. Capacidad frente a tamaño de datos

Una tabla más grande no necesariamente consume más CUs en cada consulta: importa cuánto se lee, filtra, une y procesa. Un SELECT con un filtro selectivo puede costar menos que un agregado que escanea todo el histórico. Para optimizar hay que evaluar datos escaneados, cardinalidad, patrón de joins y frecuencia. Una query pequeña ejecutada miles de veces puede sumar más consumo que otra ocasional pero pesada.

### 2.3. Observabilidad útil

Registrar al menos hora, duración y contexto de operaciones relevantes. Las métricas de infraestructura deben correlacionarse con despliegues y cambios de workload, evitando culpar a un único informe cuando las mismas horas incluyen varios procesos de ingesta.

**Fuente:** [Capacity metrics](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/monitor-fabric-data-warehouse/includes/02-capacity-metrics.md).

## 3. Dynamic Management Views

Las **DMVs** de Fabric Warehouse proporcionan una vista del estado del sistema. Las principales vistas indicadas por el módulo son:

| DMV | Qué permite investigar |
|---|---|
| \`sys.dm_exec_connections\` | Conexiones establecidas hacia el Warehouse |
| \`sys.dm_exec_sessions\` | Sesiones autenticadas |
| \`sys.dm_exec_requests\` | Solicitudes actualmente en ejecución |

Estas vistas son especialmente útiles cuando un proceso se ralentiza o necesita inspeccionarse en el instante de la incidencia. No se deben confundir con Query insights, que conserva información histórica y agregada de consultas terminadas.

~~~sql
SELECT *
FROM sys.dm_exec_requests;
~~~

La consulta es un **ejemplo propio mínimo** de inspección; la visibilidad puede depender de permisos. En producción conviene proyectar solo columnas necesarias y registrar muestras periódicas autorizadas para diagnóstico. Las DMVs no deben tratarse como una tabla de negocio persistente ni como un registro eterno: su estado refleja la actividad que la vista expone en el momento de la consulta.

### 3.1. Diagnóstico mediante correlación

Si existe una consulta lenta, identificar su sesión y solicitud, revisar si se mantiene activa y comparar con otras peticiones. Si la consulta ya terminó, es preferible utilizar Query insights. Si ninguna consulta parece lenta pero la aplicación se percibe degradada, examinar conexiones, patrón de concurrencia y capacidad.

**Fuente:** [DMVs](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/monitor-fabric-data-warehouse/includes/03-dynamic-management-views.md).

## 4. Query insights

**Query insights** ayuda a examinar patrones históricos de ejecución, en particular consultas frecuentes, consultas de larga duración e historial de requests. El módulo oficial identifica vistas con nombres:

- \`queryinsights.exec_requests_history\`
- \`queryinsights.long_running_queries\`
- \`queryinsights.frequently_run_queries\`

~~~sql
SELECT TOP 20 *
FROM queryinsights.long_running_queries;
~~~

Esta consulta es ilustrativa; consultar columnas concretas de las vistas antes de escribir paneles de observabilidad. La interpretación de «long running» depende de la definición del sistema y de su ventana de observación.

### 4.1. Diagnóstico de regresiones

Una query que normalmente dura dos segundos y pasa a tardar quince puede haber sufrido un cambio de datos, un nuevo join, un cambio de distribución o competencia con otras cargas. La metodología debe comparar consultas equivalentes, en intervalos y condiciones similares, y revisar si ha cambiado el plan o el volumen procesado. Sin una línea base resulta difícil distinguir anomalías de variabilidad normal.

### 4.2. Optimización y coste

Las mejoras pueden incluir seleccionar solo las columnas necesarias, filtrar antes de agregaciones costosas, evitar joins que multiplican filas o precomputar resultados apropiados. No todas las consultas más breves en caracteres son más eficientes. El objetivo real es disminuir trabajo, duración o coste sin alterar el resultado correcto.

**Fuente:** [Query insights](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/monitor-fabric-data-warehouse/includes/04-query-insights.md).

## 5. Laboratorio oficial

**Fuente:** [Monitor a data warehouse in Microsoft Fabric](https://github.com/MicrosoftLearning/mslearn-fabric/blob/main/Instructions/Labs/06c-monitor-data-warehouse.md). **Duración aproximada:** 30 minutos. Asociado a DP-700. **No ejecutado**.

### 5.1. Preparación y objetivo

El laboratorio crea un Workspace y un Warehouse de muestra, inspecciona DMVs y utiliza Query insights para examinar consultas. Requiere permisos de creación y lectura del entorno con capacidad Fabric compatible.

### 5.2. Crear el entorno

Abrir [Fabric](https://app.fabric.microsoft.com/), crear Workspace aislado y Warehouse con las tablas del ejercicio. Ejecutar consultas de prueba que permitan generar actividad. Registrar la hora y el propósito de cada consulta si se desea relacionarlas con los resultados de monitorización.

### 5.3. Explorar DMVs

Consultar las vistas \`sys.dm_exec_connections\`, \`sys.dm_exec_sessions\` y \`sys.dm_exec_requests\` siguiendo el SQL proporcionado. Comparar cuáles muestran conexiones, cuáles sesiones y cuáles peticiones. No esperar encontrar una solicitud que ya ha finalizado en una vista de solicitudes activas.

### 5.4. Examinar Query insights

Abrir las vistas \`queryinsights\` del tutorial y revisar el historial, las consultas que tardan más y las más repetidas. Comparar la duración con los tiempos medidos y diferenciar una ejecución individual de agregaciones de múltiples ejecuciones. No se ha ejecutado el ejercicio aquí, por lo que no se atribuyen valores exactos.

### 5.5. Errores, verificaciones y limpieza

| Síntoma | Posible causa | Comprobación |
|---|---|---|
| DMVs sin filas esperadas | Sin actividad actual | Ejecutar consulta autorizada y volver a consultar |
| Query insights no incluye ejecución reciente | Latencia o ventana de disponibilidad | Revisar cuándo terminó y período histórico |
| Error de permisos | Identidad sin acceso requerido | Revisar rol y permisos |
| Coste inesperado | Muchas ejecuciones o cargas concurrentes | Capacity Metrics app y logs |
| Query lenta solo algunas veces | Contención y variabilidad | Correlacionar hora, carga y datos |

Al terminar, borrar solo el Workspace de laboratorio o los elementos propios; la capacidad dedicada se gestiona por separado.

## 6. Knowledge check y summary

Preguntas propias: ¿Qué mide una Capacity Unit? ¿En qué se diferencia una sesión de una request? ¿Qué DMV expone solicitudes activas? ¿Qué función tiene Query insights? ¿Por qué no se debe comparar consultas con distinto volumen? ¿Cómo identificar una consulta ejecutada demasiadas veces? ¿Qué información conservar para diagnosticar una regresión?

**Summary:** la monitorización de un Warehouse une capacidad, estado actual y análisis histórico de consultas. La optimización debe basarse en métricas y preservar la corrección funcional.

## 7. Fuentes

- [Índice del módulo, siete unidades](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/monitor-fabric-data-warehouse/index.yml).
- [Capacity Metrics](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/monitor-fabric-data-warehouse/includes/02-capacity-metrics.md).
- [DMVs](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/monitor-fabric-data-warehouse/includes/03-dynamic-management-views.md).
- [Query insights](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/monitor-fabric-data-warehouse/includes/04-query-insights.md).
- [Laboratorio oficial](https://github.com/MicrosoftLearning/mslearn-fabric/blob/main/Instructions/Labs/06c-monitor-data-warehouse.md).

**Control:** documentación técnica, práctica y evaluación redactadas; no ejecutadas.