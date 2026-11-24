# 14 · Load data into a Microsoft Fabric data warehouse

> **DP-700.** [Microsoft Learn](https://learn.microsoft.com/en-us/training/modules/load-data-into-microsoft-fabric-data-warehouse/) · [Fuentes originales \`wwl-data-ai\`](https://github.com/MicrosoftDocs/learn/tree/main/learn-pr/wwl-data-ai/load-data-into-microsoft-fabric-data-warehouse) · [Laboratorio asociado a DP-700](https://github.com/MicrosoftLearning/mslearn-fabric/blob/main/Instructions/Labs/06a-data-warehouse-load.md). **Revisión:** noviembre de 2026. Documentación sin ejecución del laboratorio.

## Índice

1. [Introducción](#1-introducción)
2. [Load strategies, staging y SCD](#2-load-strategies-staging-y-scd)
3. [Load using Pipelines and Copy jobs](#3-load-using-pipelines-and-copy-jobs)
4. [Load using T-SQL](#4-load-using-t-sql)
5. [Load using Dataflow Gen2](#5-load-using-dataflow-gen2)
6. [Laboratorio oficial](#6-laboratorio-oficial)
7. [Knowledge check y summary](#7-knowledge-check-y-summary)
8. [Fuentes](#8-fuentes)

## 1. Introducción

Una ingesta no termina cuando un fichero se copia. En un Warehouse, **cargar datos** implica garantizar que los registros llegan a tablas con claves y tipos correctos, que las dimensiones y los hechos son coherentes y que una segunda ejecución no reproduce los mismos datos. La elección del mecanismo de carga responde a las características del origen, el volumen, la frecuencia, la necesidad de transformación y el modelo de datos.

Fabric permite utilizar Pipelines, Copy jobs, Dataflow Gen2 y T-SQL. Es preferible comprender los contratos de entrada y salida antes de seleccionar la herramienta. En un sistema con ventas diarias, puede convenir realizar una copia incremental a una staging table, comparar el lote con lo previamente registrado y después actualizar las tablas dimensionales. El error más común consiste en unir directamente el CSV con las tablas finales sin identificar claves ni operaciones actualizadas.

## 2. Load strategies, staging y SCD

### 2.1. Full load frente a incremental load

Una **full load** reemplaza o reconstruye todo el conjunto de destino. Es sencilla para dimensiones pequeñas o cargas iniciales, pero puede ser ineficiente y disruptiva para volúmenes grandes. Una **incremental load** procesa registros nuevos o modificados respecto a un punto de control. Reduce transferencias, pero exige gestionar borrados, cambios tardíos y reintentos.

| Enfoque | Ventajas | Problemas que deben resolverse |
| --- | --- | --- |
| Full | Lógica simple, reconstruye el conjunto | Coste, ventana de carga, consistencia durante reemplazos |
| Incremental por timestamp | Procesa cambios recientes | Marcas de tiempo no fiables, cambios atrasados |
| CDC | Captura operaciones sobre origen compatible | Orden, borrados, schema evolution, lag |
| Comparación por hash/clave | Detecta variaciones de registros | Coste de comparación, claves de negocio |
| Upsert / MERGE | Inserta y actualiza según coincidencias | Duplicidad del origen y semántica de matched |

### 2.2. Staging y validación

Una **staging table** funciona como zona intermedia. Se cargan allí los datos con identificación del lote, se valida el esquema, se eliminan duplicados y se comprueba la integridad antes de modificar tablas finales. Esta etapa ofrece una oportunidad de rollback lógico: si la entrada es incorrecta, puede rechazarse el lote sin exponerlo en Gold. La estrategia de transacciones del Warehouse y de reprocesamiento debe diseñarse explícitamente; el pipeline completo no necesariamente es atómico.

### 2.3. Business keys, surrogate keys y SCD

Una **business key** identifica una entidad en un sistema de origen. Una **surrogate key** identifica técnicamente una fila de dimensión en el Warehouse. Ambas son especialmente importantes en dimensiones historificadas. En un **Slowly Changing Dimension Type 1**, un cambio sobrescribe el atributo anterior. En **Type 2**, se conserva la fila histórica y se inserta una nueva versión con vigencia propia. No confundir Type 2 con time travel de Delta: la primera modela explícitamente el historial de negocio, mientras el segundo consulta estados físicos que pueden estar sujetos a retención.

Ejemplo conceptual:

| CustomerKey | CustomerCode | Region | ValidFrom | ValidTo | IsCurrent |
| ---: | --- | --- | --- | --- | --- |
| 101 | C500 | Norte | 2025-01-01 | 2026-05-01 | false |
| 226 | C500 | Sur | 2026-05-01 | null | true |

Una venta de marzo de 2026 debe asociarse a la versión correcta si se pretende analizar la región histórica. Una consulta que une únicamente por \`CustomerCode\` puede duplicar la venta al relacionarla con ambas filas.

### 2.4. Orden de carga de dimensiones y hechos

Primero deben resolverse claves de dimensión; después se cargan hechos con las referencias correspondientes. Si llega una transacción cuyo cliente aún no está disponible, definir qué hacer con la referencia desconocida: rechazar, poner en cuarentena, utilizar una fila «Unknown» o esperar una carga posterior. Es una decisión de negocio que no puede inferirse de la herramienta.

**Fuente:** [Load strategies](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl-data-ai/load-data-into-microsoft-fabric-data-warehouse/includes/2-explore-data-load-strategies.md).

## 3. Load using Pipelines and Copy jobs

**Copy job** ofrece un recorrido simplificado de movimiento entre fuentes y destino, y puede cubrir escenarios de copia incremental cuando la funcionalidad elegida lo admite. **Copy Data** en un Pipeline integra la copia dentro de una secuencia de actividades, como validación de fuente, carga staging, ejecución de un procedimiento SQL y comunicación del resultado. No deben usarse como sinónimos absolutos.

~~~mermaid
flowchart LR
  S["CSV / SQL / Lakehouse"] --> C["Copy job o Copy Data"]
  C --> ST["Warehouse staging"]
  ST --> V{"Validaciones"}
  V -->|OK| D["Cargar dimensiones"]
  D --> F["Cargar hechos"]
  V -->|Error| Q["Rechazar y registrar lote"]
~~~

**Diagrama propio.** Los bloques de validación no aparecen automáticamente al configurar Copy; deben implementarse, por ejemplo mediante T-SQL u otras actividades compatibles. Monitorizar el ID de ejecución y el número de filas cargadas es esencial para diagnosticar reintentos.

La orquestación puede programarse y supervisarse. Si el Pipeline usa credenciales externas, mantenerlas en conexiones gestionadas y no incrustarlas en scripts o parámetros visibles. Si un lote falla en mitad del procesamiento, registrar qué parte ya fue confirmada antes de reintentar.

**Fuente:** [Load using pipeline](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl-data-ai/load-data-into-microsoft-fabric-data-warehouse/includes/3-load-data-using-data-pipeline.md).

## 4. Load using T-SQL

### 4.1. COPY INTO

El comando \`COPY INTO\` permite ingerir archivos desde ubicaciones admitidas en tablas Warehouse. Hay que revisar el formato, ubicación, tratamiento de cabeceras, identidad, acceso al almacenamiento y comportamiento de filas rechazadas. No debe confundirse con \`COPY\` de una tabla SQL a otra.

~~~sql
COPY INTO dbo.SalesOrders
FROM 'https://fabrictutorialdata.dfs.core.windows.net/sampledata/WideWorldImportersDW/csv/sample/fact_sale/'
WITH (FILE_TYPE = 'CSV', FIRSTROW = 2);
~~~

*Ejemplo de sintaxis basado en documentación oficial, requiere tabla destino compatible y fuente accesible.* El archivo puede tener más columnas que la tabla ejemplo; por tanto, antes de ejecutar, cotejar esquema y contrato real. \`FIRSTROW = 2\` salta una primera fila, no resuelve cualquier formato CSV arbitrario.

### 4.2. CTAS e INSERT SELECT

**CREATE TABLE AS SELECT (CTAS)** permite crear una tabla a partir de una consulta, útil para materializaciones y transformaciones. **INSERT ... SELECT** inserta filas de la salida de una consulta en una tabla preexistente. Se debe revisar el esquema de columnas, conversiones, permisos y si la carga es idempotente. Una inserción repetida puede duplicar hechos aunque cada transacción sea válida.

~~~sql
INSERT INTO dbo.StageProduct (ProductCode, ProductName)
SELECT ProductCode, ProductName
FROM dbo.ImportProduct;
~~~

*Ejemplo propio, presupone las tablas y columnas.* Al cargar un histórico, no se debería confiar solo en un INSERT sin definir cómo identificar cambios o registros ya procesados.

### 4.3. Consultas entre recursos

El material oficial describe opciones para utilizar datos de otros Warehouse y Lakehouses compatibles. Una consulta entre bases o recursos requiere comprobar nombres cualificados, permisos de identidad y capacidades efectivas. En sistemas corporativos, la disponibilidad de OneLake no equivale a autorización indiscriminada.

**Fuente:** [Load using T-SQL](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl-data-ai/load-data-into-microsoft-fabric-data-warehouse/includes/4-load-data-using-tsql.md).

## 5. Load using Dataflow Gen2

Dataflow Gen2 es una alternativa low-code cuando conviene preparar la entrada con **Power Query Online**. El diseñador permite conectar fuentes, tipar columnas, eliminar nulos, unir consultas, aplicar reglas y configurar un destino Warehouse. Debe estudiarse la diferencia entre **Append**, **Replace** y las posibilidades de refresh incremental en el destino elegido.

Un Dataflow no sustituye automáticamente un modelo de cambios de dimensión. Si una dimensión necesita SCD Type 2, cargar únicamente un resultado de Power Query sobre una tabla sin gestionar vigencias históricas no aporta las propiedades requeridas. Puede utilizarse Dataflow para staging y una transformación posterior SQL para el manejo de versiones.

### 5.1. Query folding

Cuando el origen lo permite, **query folding** delega transformaciones en el sistema origen. Esto puede disminuir tráfico y trabajo en Fabric. Sin embargo, no todos los conectores ni operaciones lo permiten. Los pasos deben ordenarse y revisarse, en especial filtros y selección de columnas. La optimización debe ser comprobada, no asumida por haber utilizado una herramienta visual.

**Fuente:** [Load using Dataflow](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl-data-ai/load-data-into-microsoft-fabric-data-warehouse/includes/5-load-data-using-dataflow.md).

## 6. Laboratorio oficial

**Fuente:** [Load data into a warehouse using T-SQL](https://github.com/MicrosoftLearning/mslearn-fabric/blob/main/Instructions/Labs/06a-data-warehouse-load.md). Duración aproximada publicada: 30 minutos. Asociado al temario DP-700. **Procedimiento revisado; no ejecutado.**

### 6.1. Objetivos y arquitectura

La práctica crea un Workspace, un Lakehouse con ficheros de entrada, una tabla de origen, un Warehouse con hechos y dimensiones, y un procedimiento almacenado que utiliza consultas entre bases para cargar el Warehouse. Finalmente ejecuta consultas analíticas para verificar el resultado.

**Importante:** seguir los nombres concretos y las URL del archivo oficial enlazado. No se han reproducido datasets completos ni capturas de ejecución.

### 6.2. Preparar Workspace y Lakehouse

Crear Workspace de prueba con capacidad adecuada en [Fabric](https://app.fabric.microsoft.com/). Añadir Lakehouse, subir los archivos oficiales y verificar Files. Materializar las tablas de origen según las instrucciones. Un CSV visible en Files no garantiza que exista un objeto tabular disponible para la consulta del Warehouse.

### 6.3. Crear Warehouse y modelo

Crear el Warehouse en el mismo entorno autorizado. Ejecutar los DDL proporcionados para tablas de dimensión y hechos y cualquier vista de apoyo. Revisar las claves antes de ejecutar las cargas: una tabla de hechos necesita referencias a dimensiones y cantidades monetarias con tipos adecuados.

### 6.4. Crear y ejecutar el procedimiento

El laboratorio utiliza un stored procedure para trasladar datos desde Lakehouse a Warehouse mediante consultas compatibles entre bases. Se debe leer el código SQL para identificar cada \`INSERT\`, \`SELECT\`, relación y transformación. No tratar el procedimiento como una caja negra. Verificar el esquema de destino, permisos y orden de carga.

### 6.5. Consultas y verificaciones

Ejecutar las consultas de análisis publicadas en el ejercicio y comprobar que las tablas contienen filas y que las agregaciones mantienen sentido. Como comprobación complementaria, comparar recuentos de origen y destino y buscar claves huérfanas o duplicadas. No consignar cifras hasta ejecutar realmente el laboratorio.

### 6.6. Fallos frecuentes y limpieza

| Síntoma | Posible causa | Acción |
|---|---|---|
| Warehouse no detecta Lakehouse | Ruta, referencia o permisos | Revisar nombre cualificado y acceso |
| SQL falla al crear tablas | Objeto existente o tipo no admitido | Comprobar catálogo y compatibilidad |
| Filas duplicadas al repetir | INSERT sin control incremental | Identificar lotes y diseñar idempotencia |
| Fallo de procedimiento | Permisos o tablas faltantes | Revisar mensaje, DDL y orden de carga |
| Hechos sin dimensiones | Orden de carga incorrecto | Cargar dimensiones antes |
| Consulta sin resultados | Filtros, claves o carga vacía | Validar staging y destino |

Tras las verificaciones, eliminar únicamente objetos de práctica o Workspace exclusivo; revisar costes de capacidad y programaciones.

## 7. Knowledge check y summary

Preguntas propias: ¿Cuándo conviene full load? ¿Qué significa un watermark y cómo puede fallar? ¿Por qué se usa staging? ¿Qué diferencia hay entre business y surrogate key? ¿Qué necesita una dimensión SCD Type 2? ¿En qué se distinguen Copy job y Copy Data? ¿Qué hace COPY INTO? ¿Para qué se utiliza CTAS? ¿Qué peligro presenta INSERT SELECT repetido? ¿Por qué Dataflow Gen2 no historifica automáticamente una dimensión?

**Summary:** cargar un Warehouse implica ingesta, estrategia de cambios, control de lotes, dimensiones y hechos consistentes. La herramienta debe seleccionarse por la semántica y las garantías, no solo por facilidad de configuración.

## 8. Fuentes

- [Índice de ocho unidades (\`wwl-data-ai\`)](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl-data-ai/load-data-into-microsoft-fabric-data-warehouse/index.yml).
- [Load strategies](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl-data-ai/load-data-into-microsoft-fabric-data-warehouse/includes/2-explore-data-load-strategies.md).
- [Pipelines y copy](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl-data-ai/load-data-into-microsoft-fabric-data-warehouse/includes/3-load-data-using-data-pipeline.md).
- [T-SQL](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl-data-ai/load-data-into-microsoft-fabric-data-warehouse/includes/4-load-data-using-tsql.md).
- [Dataflow Gen2](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl-data-ai/load-data-into-microsoft-fabric-data-warehouse/includes/5-load-data-using-dataflow.md).
- [Laboratorio DP-700](https://github.com/MicrosoftLearning/mslearn-fabric/blob/main/Instructions/Labs/06a-data-warehouse-load.md).

**Control:** teoría, práctica, evaluación y síntesis redactados; ningún resultado o ejecución del servicio verificado.