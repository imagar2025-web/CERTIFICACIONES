# Glosario técnico · DP-700

Términos técnicos en inglés, con explicación original en español.

| Término | Definición |
| --- | --- |
| **OneLake** | Capa unificada de almacenamiento de Microsoft Fabric, accesible por distintos workloads compatibles. |
| **Workspace** | Contenedor de colaboración y administración de elementos de Fabric, asociado a permisos y una capacidad. |
| **Fabric capacity** | Conjunto de recursos de cómputo asignados a workloads; su consumo y facturación requieren seguimiento. |
| **Lakehouse** | Modelo analítico que combina archivos de data lake con tablas y motores de consulta. |
| **Warehouse** | Servicio de almacenamiento y consulta analítica relacional de Fabric, con endpoint SQL y cargas T-SQL. |
| **Dataflow Gen2** | Elemento de Data Factory que permite extracción y transformación low-code mediante Power Query Online y escritura a destinos. |
| **Power Query M** | Lenguaje declarativo utilizado para definir pasos de transformación de Power Query. |
| **Query folding** | Capacidad de delegar operaciones de una consulta a la fuente para reducir movimiento y procesamiento local. |
| **Staging** | Materialización intermedia que permite organizar o acelerar etapas de transformación. |
| **Copy Data activity** | Actividad de Pipeline dedicada al movimiento de datos desde un origen hasta un destino compatibles. |
| **Copy job** | Experiencia independiente para copiar datos con flujo de configuración simplificado. |
| **Pipeline** | Conjunto de actividades y reglas que ordenan y supervisan procesos de datos. |
| **Activity** | Unidad ejecutable de un Pipeline, sea movimiento, transformación o control. |
| **Run ID** | Identificador de una ejecución concreta, útil para observar y diagnosticar fallos. |
| **Idempotencia** | Propiedad deseada por la que repetir una carga con la misma entrada no cambia su resultado lógico final. |
| **ETL** | Extract, Transform, Load: transformar antes de cargar en el destino analítico. |
| **ELT** | Extract, Load, Transform: cargar primero y transformar posteriormente sobre un almacén elegido. |
| **DataFrame** | Estructura tabular distribuida de Spark con esquema y transformaciones optimizables. |
| **Driver** | Proceso Spark que coordina la planificación y ejecución de trabajos. |
| **Executor** | Proceso Spark que ejecuta tareas sobre particiones de datos. |
| **Spark pool** | Conjunto de recursos de cómputo utilizados para iniciar y ejecutar sesiones Spark en Fabric. |
| **Runtime** | Combinación de versiones de Spark, Python y otros componentes compatibles. |
| **Environment** | Configuración reutilizable de runtime, dependencias y propiedades de ejecución en Fabric. |
| **Shuffle** | Redistribución de datos entre particiones/executors para joins o agregaciones que la requieren. |
| **Parquet** | Formato columnar optimizado para lectura analítica. |
| **Delta Lake** | Capa de tablas transaccionales sobre archivos, con log de operaciones y versionado. |
| **Time travel** | Consulta de versiones históricas disponibles de una tabla, condicionada por las políticas de retención. |
| **Bronze / Silver / Gold** | Capas de una arquitectura medallion: dato de origen, dato validado y dato listo para consumo. |
| **Eventhouse** | Elemento de Fabric que contiene una o varias bases de datos KQL orientadas a eventos. |
| **KQL** | Kusto Query Language, lenguaje de consulta por operadores para logs, telemetría y series temporales. |
| **Queryset** | Espacio de consultas guardadas o editables sobre bases KQL. |
| **Materialized view** | Vista con resultados agregados precomputados y gestionados por el sistema. |
| **Stored function** | Función persistente y reutilizable que encapsula lógica de consultas. |
| **Eventstream** | Funcionalidad para recoger, transformar y distribuir eventos hacia destinos admitidos. |
| **CI/CD** | Prácticas de integración y entrega continuas para controlar cambios y despliegues. |
| **Deployment pipeline** | Mecanismo de promoción de elementos entre etapas de desarrollo y producción cuando el servicio lo permite. |

Para detalles, configuración y ejemplos, consultar los [módulos publicados](README.md) y [fuentes oficiales](FUENTES.md).
