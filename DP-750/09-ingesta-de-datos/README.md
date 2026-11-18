# 09 · Ingerir datos en Unity Catalog

[Índice DP-750](../README.md) · [← Diseñar e implementar el modelado de datos](../08-modelado-de-datos/README.md) · [Limpiar, transformar y cargar datos →](../10-limpieza-transformacion-y-carga/README.md)

**Progreso personal:** Estudiado. [Módulo oficial](https://learn.microsoft.com/en-us/training/modules/ingest-data-into-unity-catalog/).

## Unidades

1. [Introduction](#unidad-01)
2. [Ingest data with Lakeflow Connect](#unidad-02)
3. [Ingest data with notebooks](#unidad-03)
4. [Ingest data with SQL methods](#unidad-04)
5. [Ingest data with CDC feed](#unidad-05)
6. [Ingest data with Spark Structured Streaming](#unidad-06)
7. [Ingest data with Auto Loader](#unidad-07)
8. [Ingest data with Lakeflow pipelines](#unidad-08)
9. [Exercise - Ingest Data into Unity Catalog](#unidad-09)
10. [Module assessment](#unidad-10)
11. [Summary](#unidad-11)

- [Desarrollo aplicado](#desarrollo-aplicado)
- [Laboratorio y comprobaciones](#laboratorio-y-comprobaciones)

## Unidad 01

### Introduction

**Ingerir** significa introducir datos de una fuente al sistema destino preservando su significado y trazabilidad. La transformación puede ocurrir durante o después de la ingesta. La ingesta fiable contempla al menos: conexión, esquema, deduplicación, control de errores, reintentos, observabilidad y posibilidad de reprocesar sin duplicar resultados.

El **batch** procesa conjuntos discretos de datos. El **streaming** procesa entradas conforme están disponibles mediante un sistema continuo o incremental. El **CDC** comunica cambios en registros de un sistema fuente, normalmente inserciones, modificaciones y eliminaciones.

[Unidad oficial](https://learn.microsoft.com/en-us/training/modules/ingest-data-into-unity-catalog/1-introduction)

## Unidad 02

### Ingest data with Lakeflow Connect

Lakeflow Connect proporciona mecanismos gestionados de integración para orígenes compatibles. Frente a desarrollar conectores manuales, reduce trabajo de autenticación, extracción y control operacional, aunque exige configurar accesos a origen/destino y comprobar restricciones del conector, versiones y coste.

Caso típico: sincronizar datos de una base operacional con el Lakehouse y continuar actualizando datos sin repetir la carga completa. Elige un conector administrado cuando cumpla requisitos de datos y seguridad; si el origen no está admitido o exige lógica especial, utiliza una solución programática.

[Unidad oficial](https://learn.microsoft.com/en-us/training/modules/ingest-data-into-unity-catalog/2-ingest-data-lakeflow-connect)

## Unidad 03

### Ingest data with notebooks

Un notebook permite leer CSV, JSON, Parquet u orígenes conectados, transformar y escribir una tabla. Spark usa ejecución distribuida; `spark.read` crea una representación lógica de datos, mientras acciones como `count()` disparan trabajo real.

```python
from pyspark.sql import functions as F

ruta = "/Volumes/estudio/ventas/entradas/pedidos.csv"
df = (spark.read
      .option("header", True)
      .option("inferSchema", True)
      .csv(ruta))

limpio = (df.filter(F.col("id_pedido").isNotNull())
            .dropDuplicates(["id_pedido"]))

limpio.write.format("delta").mode("append").saveAsTable("estudio.ventas.pedidos_raw")
```

Este ejemplo es didáctico: en producción suele preferirse definir esquema explícito, gestionar entradas repetidas y no tratar `dropDuplicates` como solución universal al CDC.

[Unidad oficial](https://learn.microsoft.com/en-us/training/modules/ingest-data-into-unity-catalog/3-ingest-data-notebooks)

## Unidad 04

### Ingest data with SQL methods

`COPY INTO` carga archivos en una tabla Delta y permite evitar reprocesar archivos ya ingeridos en condiciones compatibles. `CREATE TABLE AS SELECT` (CTAS) crea una tabla a partir de los resultados de una consulta. `INSERT INTO ... SELECT` incorpora filas a una tabla existente.

```sql
COPY INTO estudio.ventas.pedidos_raw
FROM '/Volumes/estudio/ventas/entradas/'
FILEFORMAT = CSV
FORMAT_OPTIONS ('header' = 'true');

CREATE TABLE estudio.ventas.totales AS
SELECT id_cliente, SUM(importe) AS total
FROM estudio.ventas.pedidos_raw
GROUP BY id_cliente;
```

El ejemplo presupone esquemas compatibles, formatos y permisos correctos. CTAS es apropiado para construir nuevas tablas, no para mantener por sí solo un flujo CDC de versiones.

[Unidad oficial](https://learn.microsoft.com/en-us/training/modules/ingest-data-into-unity-catalog/4-ingest-data-sql-methods)

## Unidad 05

### Ingest data with CDC feed

Un feed CDC registra operaciones `INSERT`, `UPDATE`, `DELETE` junto con un criterio de orden (timestamp o número de secuencia). Sin orden suficiente pueden producirse resultados incorrectos ante cambios atrasados. Cuando se reconstruye una tabla actual, interesa aplicar el último cambio válido; cuando se necesita histórico, interesan intervalos o versiones.

La API **AUTO CDC** de Lakeflow puede facilitar la aplicación de cambios y la gestión de tipos SCD admitidos. Requiere especificar claves, orden de secuencia y tratamiento de eliminaciones; la sintaxis concreta depende del tipo de pipeline configurado.

**Ejemplo de eventos:** `cliente 10, ciudad A, seq 100` → `cliente 10, ciudad B, seq 102` → `cliente 10 eliminado, seq 103`. Si llega al final un evento con `seq 101`, no debe revertir el estado más reciente.

#### Orden, errores y reconstrucción del destino

Un registro CDC necesita una clave y un criterio de orden reproducible. Dos cambios con el mismo timestamp pueden requerir un desempate mediante un ordinal o identificador de secuencia que conserve el orden del origen. Lakeflow puede expresar una lógica de aplicación mediante AUTO CDC con clave, secuencia y configuración del manejo de eliminaciones. Decide si quieres el último valor o todas las versiones válidas. Si aplicas cortes de retención en el origen, afecta a cuán atrás puedes reconstruir el flujo. Para que la recuperación sea repetible, registra qué snapshot inicial usaste y desde qué posición empezaron los cambios que le siguen.

[Unidad oficial](https://learn.microsoft.com/en-us/training/modules/ingest-data-into-unity-catalog/5-ingest-data-change-data-capture-feed)

## Unidad 06

### Ingest data with Spark Structured Streaming

Structured Streaming usa APIs de Spark para procesar datos incrementales. Una lectura de stream produce un DataFrame de flujo; una escritura con checkpoint conserva progreso y estado necesarios para recuperarse, dependiendo de la fuente y del operador.

```python
stream_df = (spark.readStream
             .format("rate")
             .option("rowsPerSecond", 5)
             .load())

query = (stream_df.writeStream
         .format("delta")
         .option("checkpointLocation", "/Volumes/estudio/ventas/entradas/checkpoint_demo")
         .trigger(availableNow=True)
         .toTable("estudio.ventas.eventos_demo"))
```

Este ejemplo usa una fuente `rate` sintética. En producción, buses como Kafka y Event Hubs necesitan configuración de lectura, autenticación, offsets y tolerancia a fallos. La semántica de «exactamente una vez» requiere considerar tanto la fuente como el destino, no solo activar un checkpoint.

#### Event Hubs y checkpoints

Un flujo de eventos necesita decidir cómo leer cada partición y cómo recuperar el progreso. Para Event Hubs, evalúa su endpoint compatible con Kafka cuando corresponda y configura autenticación mediante el mecanismo admitido. El checkpoint de Structured Streaming persiste el estado operativo, incluidos offsets y estado asociado a la consulta. Cada flujo requiere su ubicación duradera. La primera ejecución toma su posición inicial según la configuración admitida. Las ejecuciones de un flujo recuperado con el mismo checkpoint siguen el progreso guardado. Un cambio de topología, fuente o destino puede invalidar la recuperación desde ese estado y debe probarse con un flujo separado.

[Unidad oficial](https://learn.microsoft.com/en-us/training/modules/ingest-data-into-unity-catalog/6-ingest-data-spark-structured-streaming)

## Unidad 07

### Ingest data with Auto Loader

Auto Loader se basa en `cloudFiles` para detectar archivos nuevos y procesarlos de forma incremental. Suele acompañarse de una ubicación de esquema (`cloudFiles.schemaLocation`) y un checkpoint para retener el progreso. Es útil cuando llegan archivos a ADLS continuamente o en lotes repetidos.

```python
raw = (spark.readStream
       .format("cloudFiles")
       .option("cloudFiles.format", "json")
       .option("cloudFiles.schemaLocation", "/Volumes/estudio/ventas/entradas/esquema")
       .load("/Volumes/estudio/ventas/entradas/json/"))

(raw.writeStream
 .option("checkpointLocation", "/Volumes/estudio/ventas/entradas/checkpoint_json")
 .trigger(availableNow=True)
 .toTable("estudio.ventas.eventos_json"))
```

**Evolución del esquema:** nuevos campos o cambios de tipos pueden necesitar reglas de rescate, reinicio o intervención; no conviene inferir que todos los cambios serán seguros automáticamente. Los checkpoints deben aislarse por flujo y mantenerse persistentes.

[Unidad oficial](https://learn.microsoft.com/en-us/training/modules/ingest-data-into-unity-catalog/7-ingest-data-auto-loader)

## Unidad 08

### Ingest data with Lakeflow pipelines

Un pipeline representa datasets, transformaciones y dependencias. Lakeflow permite diseñar flujos declarativos y enlazar ingestión, validación y publicación. La orquestación necesita dependencias, reintentos, alertas, monitorización y control de ejecución. Conviene distinguir el **pipeline que transforma datos** del **job que organiza varias tareas**.

```text
Origen transaccional → Ingesta Bronze → Validación Silver → Agregado Gold
                             ↓                ↓                  ↓
                        métricas            reglas           consumo BI
```

[Unidad oficial](https://learn.microsoft.com/en-us/training/modules/ingest-data-into-unity-catalog/8-ingest-data-lakeflow-declarative-pipelines)

## Unidad 09

### Exercise - Ingest Data into Unity Catalog

Recibe diez archivos JSON, algunos repetidos. Define una estrategia con Auto Loader, checkpoint y tabla Bronze; construye una comprobación de duplicados y escribe métricas de ingestión. En un segundo paso, plantea qué cambiaría si el origen fuera un feed CDC de clientes con eliminaciones y eventos fuera de orden.

**Preguntas clave:** ¿qué resuelve el checkpoint? ¿En qué se diferencia COPY INTO de CTAS? ¿Para qué sirve `sequence_by` conceptualmente en CDC? ¿En qué caso elegirías Lakeflow Connect en vez de un notebook personalizado?

[Unidad oficial](https://learn.microsoft.com/en-us/training/modules/ingest-data-into-unity-catalog/9-exercise-ingest-data-into-unity-catalog)

## Unidad 10

### Module assessment

Recibes un conjunto de CSV por fichero y necesitas una primera carga con bajo esfuerzo en SQL. Evalúa COPY INTO. Si los ficheros llegan continuamente y aumenta el número de rutas, evalúa Auto Loader con estado y checkpoints separados por flujo. Si necesitas cambios de una tabla fuente, conserva la clave, su tipo de operación y la secuencia del origen. Ninguna garantía de lectura por archivo elimina por sí sola los eventos de negocio duplicados que lleguen en archivos distintos.

[Unidad oficial](https://learn.microsoft.com/en-us/training/modules/ingest-data-into-unity-catalog/10-knowledge-check)

## Unidad 11

### Summary

Antes de poner la ingestión en producción, define el dato que identifica cada entrega y qué puede repetirse tras una caída. La auditoría de cada lote debe incluir entradas detectadas, entradas admitidas, filas persistidas y errores. Si el origen cuenta eventos y el destino cuenta filas, incluye la relación esperada entre ambos. Para eventos de Azure Event Hubs, comprueba el mecanismo de conexión admitido, credenciales, particiones de origen y el seguimiento de offsets antes de calcular paralelismo y latencia.

[Unidad oficial](https://learn.microsoft.com/en-us/training/modules/ingest-data-into-unity-catalog/11-summary)

## Desarrollo aplicado

### Una carga correcta debe poder repetirse

El desafío de la ingestión no es solo introducir datos en una tabla. Debes determinar cómo se comporta el pipeline si la misma carga se ejecuta dos veces, si la conexión falla a mitad de proceso, si llega un archivo duplicado o si el origen cambia de esquema. La **idempotencia** significa que repetir una operación con las mismas entradas no altera indebidamente el resultado final. Es una propiedad esencial cuando hay reintentos automáticos.

### Batch frente a streaming

Una carga batch procesa lotes delimitados, por ejemplo un fichero diario. Streaming procesa datos que llegan progresivamente y suele funcionar en microbatches con estado y checkpoints. No toda actualización frecuente exige streaming: una carga cada quince minutos puede cubrir un SLA sin la complejidad de mantener un flujo continuo. La elección depende de latencia, coste, disponibilidad del origen y gestión de fallos.

### COPY INTO, Auto Loader y Structured Streaming

`COPY INTO` resulta apropiado para cargar archivos desde ubicaciones accesibles mediante SQL y conservar seguimiento de los archivos procesados según la semántica admitida. Auto Loader está diseñado para descubrir progresivamente archivos nuevos, manejar evolución de esquemas y escalar la detección. Structured Streaming proporciona la base de procesamiento incremental y permite escribir con checkpoints. Cuando el origen es una base de datos con cambios, un flujo CDC puede resultar más eficiente que leer tablas completas.

```python
from pyspark.sql import functions as F

raw = (spark.readStream.format("cloudFiles")
    .option("cloudFiles.format", "json")
    .option("cloudFiles.schemaLocation", "/Volumes/estudio/raw/estado/esquemas_ventas")
    .load("/Volumes/estudio/raw/entrada/ventas"))

query = (raw.withColumn("ingestion_ts", F.current_timestamp())
    .writeStream.format("delta")
    .option("checkpointLocation", "/Volumes/estudio/raw/estado/checkpoint_ventas")
    .trigger(availableNow=True)
    .toTable("estudio.bronze.ventas_raw"))
```

El ejemplo presupone volúmenes y tablas creados, privilegios sobre sus rutas y soporte de Auto Loader en el cómputo elegido. `availableNow` procesa los archivos disponibles y finaliza; no deja una consulta continua funcionando indefinidamente. **No reutilices checkpoints entre consultas distintas ni los borres sin entender las consecuencias**, porque puedes volver a procesar archivos o perder el punto de progreso.

### Captura de cambios y orden de eventos

Un origen CDC puede producir inserciones, actualizaciones y eliminaciones. Hay que conservar la clave de negocio, una secuencia de cambios confiable, el tipo de operación y una marca temporal. Si llegan eventos fuera de orden, aplicar simplemente el último evento recibido puede sobrescribir información nueva con una actualización antigua. Definir la ordenación y deduplicación de cambios es tan importante como copiar los bytes.

### Plan de validación de la ingestión

Prueba (1) primera carga, (2) reejecución sin nuevas entradas, (3) llegada de nuevos archivos, (4) fichero con columna extra, (5) error de tipo, (6) eliminación de un dato en el origen, (7) caída a mitad de procesamiento. Para cada caso registra qué debe ocurrir en Bronze y si Silver puede reconstruirse sin acceder de nuevo al origen. El diseño Medallion se beneficia de preservar datos originales y metadatos de procedencia para facilitar auditoría y reprocesos.

## Laboratorio y comprobaciones

**Ejercicio oficial:** [Abrir las instrucciones de MicrosoftLearning](https://microsoftlearning.github.io/DP-750T00-Implement-Data-Engineering-Solutions-using-Azure-Databricks/Instructions/Labs/07-ingest-data-into-unity-catalog.html)  
**Notebook oficial:** [Importar o consultar notebook `.ipynb`](https://raw.githubusercontent.com/MicrosoftLearning/DP-750T00-Implement-Data-Engineering-Solutions-using-Azure-Databricks/refs/heads/main/Allfiles/07-ingest-data-into-unity-catalog.ipynb)  
**Naturaleza de esta sección:** itinerario explicado con redacción propia y comprobaciones de aprendizaje. Las celdas exactas y el material de origen están en el notebook oficial enlazado; no se reproducen íntegramente aquí.


### Caso práctico: Solaris Energy — cuatro modos de ingesta

**Escenario:** instalaciones solares y aerogeneradores de varios países envían CSV de producción y mantenimiento. Se necesita unificar el aterrizaje en Unity Catalog y comparar distintas estrategias de ingestión.

**Desarrollo de la actividad:**

1. Importa el notebook; su configuración crea `solaris_lab` con esquemas `bronze` y `silver`, y el volumen managed `solaris_lab.bronze.raw_files` para almacenar CSV como `solar_readings.csv` y `turbine_events.csv`.
2. **Ingesta batch con DataFrames:** utiliza `spark.read` con un esquema explícito y tipos correctos cuando sea posible; guarda una tabla Delta mediante `DataFrameWriter`. Valida el recuento, el schema y la trazabilidad del archivo original.
3. **SQL incremental:** practica `COPY INTO` desde la ubicación del volumen hasta una tabla Delta. Repite el comando sin incorporar nuevos archivos y razona cómo Databricks registra las cargas de archivo para evitar una recarga no intencionada.
4. **CTAS:** crea una tabla de agregados mediante `CREATE TABLE ... AS SELECT` sobre los datos ya ingeridos. Identifica si el resultado persiste una copia de resultados o es solo una vista.
5. **Auto Loader:** configura `spark.readStream.format('cloudFiles')` y la opción `cloudFiles.format` adecuada; define checkpoint y un destino Delta. Añade un nuevo fichero y comprueba que el siguiente lote de streaming lo incorpora.
6. Como exploración opcional, abre el editor de **Lakeflow Spark Declarative Pipelines** y localiza catálogo, esquema, código fuente y modalidad de ejecución. No confundas el trabajo programado por un Job con una definición declarativa de datasets.

**Puntos de control:** compara las garantías de `COPY INTO` (seguimiento de archivos ingeridos), las de Auto Loader (estado/checkpoint) y las de una escritura batch personalizada. No declares **exactly once** para cualquier código sin identificar sus condiciones de idempotencia.

**SQL de práctica creado para estos apuntes:**

```sql
SELECT site_id, COUNT(*) AS lecturas, AVG(power_kw) AS potencia_media
FROM solaris_lab.bronze.solar_readings
GROUP BY site_id;
```

Ajusta el nombre de la tabla a la definición del notebook oficial antes de ejecutarlo.

### Experimento de reejecución

Prepara la entrada oficial y conserva un recuento antes de iniciar. Ejecuta COPY INTO dos veces sobre el mismo destino con los mismos archivos. Usa `COUNT(*)` antes y después y comprueba que la segunda ejecución normal no suma las filas del mismo fichero. Para Auto Loader utiliza su propio destino y checkpoint. Agrega un archivo nuevo que tenga una fila identificable y verifica que esa fila aparece tras la siguiente ejecución. Si un fichero nuevo contiene eventos repetidos, compara recuentos y claves de negocio. Si el flujo de pruebas queda activo, deténlo al acabar. Conserva el checkpoint para probar recuperación. Su eliminación solo corresponde a un reinicio planificado de un ejercicio descartable.

## Fuentes y navegación

- [Fuentes de la certificación](../FUENTES.md)
- [Preparación y limpieza del entorno](../PREPARACION-LABORATORIOS.md)

[Índice DP-750](../README.md) · [← Diseñar e implementar el modelado de datos](../08-modelado-de-datos/README.md) · [Limpiar, transformar y cargar datos →](../10-limpieza-transformacion-y-carga/README.md)
