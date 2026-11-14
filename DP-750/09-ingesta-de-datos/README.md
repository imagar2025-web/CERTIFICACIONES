[🏠 Índice](../README.md) · [← Anterior](../08-modelado-de-datos/README.md) · [Siguiente →](../10-limpieza-transformacion-y-carga/README.md)

> **Estado de estudio:** ✅ Visto · **Unidades oficiales:** 11 · [Microsoft Learn](https://learn.microsoft.com/en-us/training/modules/ingest-data-into-unity-catalog/)  
> **Versión de apuntes:** 14-11-2026 · Contenido técnico redactado de forma original a partir del itinerario de Microsoft.

# 9. Ingerir datos en Unity Catalog

**Referencia Microsoft Learn:** [Ingerir datos en Unity Catalog](https://learn.microsoft.com/en-us/training/modules/ingest-data-into-unity-catalog/) · **11 unidades oficiales**.

## 9.1. Fundamentos de la ingesta

**Ingerir** significa introducir datos de una fuente al sistema destino preservando su significado y trazabilidad. La transformación puede ocurrir durante o después de la ingesta. La ingesta fiable contempla al menos: conexión, esquema, deduplicación, control de errores, reintentos, observabilidad y posibilidad de reprocesar sin duplicar resultados.

El **batch** procesa conjuntos discretos de datos. El **streaming** procesa entradas conforme están disponibles mediante un sistema continuo o incremental. El **CDC** comunica cambios en registros de un sistema fuente, normalmente inserciones, modificaciones y eliminaciones.

## 9.2. Lakeflow Connect

Lakeflow Connect proporciona mecanismos gestionados de integración para orígenes compatibles. Frente a desarrollar conectores manuales, reduce trabajo de autenticación, extracción y control operacional, aunque exige configurar accesos a origen/destino y comprobar restricciones del conector, versiones y coste.

Caso típico: sincronizar datos de una base operacional con el Lakehouse y continuar actualizando datos sin repetir la carga completa. Elige un conector administrado cuando cumpla requisitos de datos y seguridad; si el origen no está admitido o exige lógica especial, utiliza una solución programática.

## 9.3. Ingesta con notebooks y DataFrames

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

## 9.4. Ingesta con SQL: COPY INTO y CTAS

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

## 9.5. Ingesta de cambios con CDC

Un feed CDC registra operaciones `INSERT`, `UPDATE`, `DELETE` junto con un criterio de orden (timestamp o número de secuencia). Sin orden suficiente pueden producirse resultados incorrectos ante cambios atrasados. Cuando se reconstruye una tabla actual, interesa aplicar el último cambio válido; cuando se necesita histórico, interesan intervalos o versiones.

La API **AUTO CDC** de Lakeflow puede facilitar la aplicación de cambios y la gestión de tipos SCD admitidos. Requiere especificar claves, orden de secuencia y tratamiento de eliminaciones; la sintaxis concreta depende del tipo de pipeline configurado.

**Ejemplo de eventos:** `cliente 10, ciudad A, seq 100` → `cliente 10, ciudad B, seq 102` → `cliente 10 eliminado, seq 103`. Si llega al final un evento con `seq 101`, no debe revertir el estado más reciente.

## 9.6. Structured Streaming

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

## 9.7. Auto Loader

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

## 9.8. Lakeflow pipelines y orquestación

Un pipeline representa datasets, transformaciones y dependencias. Lakeflow permite diseñar flujos declarativos y enlazar ingestión, validación y publicación. La orquestación necesita dependencias, reintentos, alertas, monitorización y control de ejecución. Conviene distinguir el **pipeline que transforma datos** del **job que organiza varias tareas**.

```text
Origen transaccional → Ingesta Bronze → Validación Silver → Agregado Gold
                             ↓                ↓                  ↓
                        métricas            reglas           consumo BI
```

## 9.9. Laboratorio y evaluación

Recibe diez archivos JSON, algunos repetidos. Define una estrategia con Auto Loader, checkpoint y tabla Bronze; construye una comprobación de duplicados y escribe métricas de ingestión. En un segundo paso, plantea qué cambiaría si el origen fuera un feed CDC de clientes con eliminaciones y eventos fuera de orden.

**Preguntas clave:** ¿qué resuelve el checkpoint? ¿En qué se diferencia COPY INTO de CTAS? ¿Para qué sirve `sequence_by` conceptualmente en CDC? ¿En qué caso elegirías Lakeflow Connect en vez de un notebook personalizado?

---

---

## 🧪 Guía de laboratorio y comprobación práctica

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


---

### 📚 Fuentes y navegación

- [Contenido oficial de Microsoft Learn](https://learn.microsoft.com/en-us/training/modules/ingest-data-into-unity-catalog/)
- [Repositorio docente de laboratorios DP-750T00](https://microsoftlearning.github.io/DP-750T00-Implement-Data-Engineering-Solutions-using-Azure-Databricks/)
- [Guía oficial de competencias del examen DP-750](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-750)

[🏠 Índice](../README.md) · [← Anterior](../08-modelado-de-datos/README.md) · [Siguiente →](../10-limpieza-transformacion-y-carga/README.md)
