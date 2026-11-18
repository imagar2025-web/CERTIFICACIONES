# 08 · Diseñar e implementar el modelado de datos

[Índice DP-750](../README.md) · [← Gobernar objetos de Unity Catalog](../07-gobernar-objetos-de-unity-catalog/README.md) · [Ingerir datos en Unity Catalog →](../09-ingesta-de-datos/README.md)

**Progreso personal:** Estudiado. [Módulo oficial](https://learn.microsoft.com/en-us/training/modules/design-implement-data-modeling-unity-catalog/).

## Unidades

1. [Introduction](#unidad-01)
2. [Design ingestion logic and data source configuration](#unidad-02)
3. [Choose a data ingestion tool](#unidad-03)
4. [Choose a data table format](#unidad-04)
5. [Design and implement a data partitioning scheme](#unidad-05)
6. [Choose a slowly changing dimension (SCD) type](#unidad-06)
7. [Implement a slowly changing dimension (SCD) type 2](#unidad-07)
8. [Design and implement a temporal (history) table to record changes over time](#unidad-08)
9. [Choose granularity on a column or table based on requirements](#unidad-09)
10. [Choose managed vs external tables](#unidad-10)
11. [Design and implement a clustering strategy](#unidad-11)
12. [Exercise - Design and Implement Data Modeling with Azure Databricks](#unidad-12)
13. [Knowledge check](#unidad-13)
14. [Summary](#unidad-14)

- [Desarrollo aplicado](#desarrollo-aplicado)
- [Laboratorio y comprobaciones](#laboratorio-y-comprobaciones)

## Unidad 01

### Introduction

El modelado de datos define qué representa cada tabla, cuál es su **granularidad**, qué claves utiliza y cómo se relaciona con otras. No se trata únicamente de elegir tipos de columnas. Un mal grano produce duplicaciones en agregados, mientras que una clave inestable dificulta integraciones y actualizaciones.

[Unidad oficial](https://learn.microsoft.com/en-us/training/modules/design-implement-data-modeling-unity-catalog/1-introduction)

## Unidad 02

### Design ingestion logic and data source configuration

Antes de construir un pipeline, caracteriza cada origen: fichero, base de datos, evento o API; disponibilidad batch/streaming; volumen y velocidad; esquema; autenticación; tratamiento de errores; semántica de cambios y reenvíos. También debe definirse el contrato del destino: tabla de llegada inmutable, copia actual o historial versionado.

**Preguntas que evitan fallos:** ¿cómo se detecta una actualización de una fila? ¿pueden llegar eventos duplicados? ¿qué ocurre con datos fuera de orden? ¿el origen permite carga inicial y CDC posterior? ¿existe una fecha fiable de modificación?

[Unidad oficial](https://learn.microsoft.com/en-us/training/modules/design-implement-data-modeling-unity-catalog/2-design-data-ingestion-logic)

## Unidad 03

### Choose a data ingestion tool

- **Lakeflow Connect:** conectores administrados para fuentes soportadas, reduciendo código de integración.
- **Auto Loader:** detección incremental de archivos que aparecen en almacenamiento.
- **COPY INTO:** importación incremental por SQL para archivos compatibles.
- **Structured Streaming:** ingestión continua o microbatch con control de estado y checkpoints.
- **Notebooks / Jobs:** personalización cuando se necesita lógica adicional.

El criterio de elección incluye requisitos de latencia, mantenimiento, disponibilidad del conector, volumen, semántica de errores y control de transformaciones.

[Unidad oficial](https://learn.microsoft.com/en-us/training/modules/design-implement-data-modeling-unity-catalog/3-choose-data-ingestion-tool)

## Unidad 04

### Choose a data table format

**Delta Lake** combina archivos columnares con un registro transaccional que permite operaciones ACID, manejo de esquema y cambios controlados. **Apache Iceberg** es otro formato abierto para tablas analíticas con metadatos de snapshots y evolución de esquemas. No son simplemente «formatos de archivo»: son formatos de tabla con reglas sobre metadatos y mantenimiento.

Su interoperabilidad depende de versiones, motores y características admitidas. Para funciones como `MERGE`, streaming y políticas gestionadas, hay que comprobar compatibilidad real en Databricks antes de elegir un formato por moda.

[Unidad oficial](https://learn.microsoft.com/en-us/training/modules/design-implement-data-modeling-unity-catalog/4-choose-data-table-format)

## Unidad 05

### Design and implement a data partitioning scheme

Particionar separa físicamente datos siguiendo ciertos valores (por ejemplo mes), lo que puede permitir evitar leer particiones irrelevantes. Pero **crear demasiadas particiones pequeñas** genera muchos archivos y metadatos; particionar por un identificador de cliente único suele ser mala elección para un dataset grande y disperso.

**Ejemplo:** consultas filtradas habitualmente por mes pueden beneficiarse de una estrategia de particionado o clustering adecuada; si una tabla tiene pocas filas, añadir particiones puede empeorar su mantenimiento. La cardinalidad, el patrón de filtros y el tamaño real del dataset mandan.

#### Tamaño de particiones y filtros

Si existen cien centros y se escriben entregas pequeñas cada pocos minutos, particionar por centro puede hacer que cada ejecución genere muchos archivos pequeños. Pregunta por el volumen de cada partición, frecuencia de escritura y predicados reales. Un filtro sobre una clave de partición puede reducir directorios leídos. Para tablas nuevas en las que se elige liquid clustering, el diseño concentra filas de valores relacionados en archivos sin introducir esas particiones Hive-style. Compara con consultas representativas y revisa bytes y archivos leídos. Un buen diseño depende de la distribución, de la selectividad y de cuánto cambian los filtros con el tiempo.

[Unidad oficial](https://learn.microsoft.com/en-us/training/modules/design-implement-data-modeling-unity-catalog/5-design-data-partitioning-scheme)

## Unidad 06

### Choose a slowly changing dimension (SCD) type

Las **SCD** describen cómo mantener atributos de dimensiones cuando cambian. Ejemplo: un cliente cambia de ciudad.

| Tipo | Estrategia | ¿Conserva valor antiguo? | Caso típico |
|---|---|---|---|
| SCD 0 | Nunca actualizar atributo | Sí, permanece el original | Fecha de nacimiento validada |
| SCD 1 | Sobrescribir valor actual | No en la tabla dimensional | Corregir un error de escritura |
| SCD 2 | Crear nueva fila versionada | Sí | Analizar evolución histórica |
| SCD 3 | Añadir columnas de valor previo | De forma limitada | Comparar valor anterior y actual |

Para análisis histórico fiable sobre clientes, **SCD Tipo 2** suele ser la respuesta frente a sobrescribir siempre los atributos con Tipo 1.

[Unidad oficial](https://learn.microsoft.com/en-us/training/modules/design-implement-data-modeling-unity-catalog/6-choose-slowly-changing-dimension-type)

## Unidad 07

### Implement a slowly changing dimension (SCD) type 2

SCD2 crea una fila por cada periodo de validez de un miembro de dimensión. Necesita una **clave de negocio** estable (`id_cliente`), un mecanismo para ordenar cambios y campos como `vigente_desde`, `vigente_hasta`, `es_actual`. Puede incorporar una clave sustituta propia de cada versión.

```text
id_cliente | ciudad  | vigente_desde | vigente_hasta | es_actual
101        | Madrid  | 2025-01-01    | 2026-03-14    | false
101        | Toledo  | 2026-03-15    | NULL          | true
```

Procedimiento conceptual:

1. Leer cambios del origen y eliminar duplicados por clave + versión.
2. Comparar columnas que se historizan, no todas indiscriminadamente.
3. Si hay cambio, **cerrar** la versión activa en la fecha efectiva adecuada.
4. **Insertar** una nueva versión con `es_actual=true`.
5. Evitar solapamientos temporales y verificar que exista como máximo una fila actual por clave.

Un `MERGE` simple que solo actualiza valores implementa Tipo 1, **no SCD2**. Para SCD2 se emplean patrones de inserción/cierre o mecanismos gestionados como AUTO CDC cuando encajan con el pipeline.

[Unidad oficial](https://learn.microsoft.com/en-us/training/modules/design-implement-data-modeling-unity-catalog/7-implement-slowly-changing-dimension-type-2)

## Unidad 08

### Design and implement a temporal (history) table to record changes over time

Las tablas temporales registran cambios con un eje temporal. Es útil distinguir **tiempo válido** (cuándo se consideró cierto un atributo en el negocio) y **tiempo de sistema** (cuándo se registró el cambio). En escenarios exigentes se pueden conservar ambos (modelo bitemporal).

```sql
SELECT id_cliente, ciudad
FROM estudio.crm.clientes_historial
WHERE DATE '2026-02-01' >= vigente_desde
  AND (vigente_hasta IS NULL OR DATE '2026-02-01' < vigente_hasta);
```

Este patrón asume intervalos cerrados por inicio y abiertos por fin; es necesario adoptarlo consistentemente. El historial de transacciones Delta facilita consultas de versiones recientes conservadas, pero no reemplaza una dimensión diseñada para duración indefinida.

[Unidad oficial](https://learn.microsoft.com/en-us/training/modules/design-implement-data-modeling-unity-catalog/8-design-temporal-history-table)

## Unidad 09

### Choose granularity on a column or table based on requirements

El **grano** determina qué representa una fila: una línea de pedido, un pedido o un resumen mensual. Una tabla de hechos almacena medidas y claves de dimensión; una dimensión describe entidades y características. Si se mezclan granos sin cuidado, un join puede multiplicar medidas y falsear las sumas.

Ejemplo: `hechos_ventas` con grano **una línea de pedido** puede relacionarse con `dim_cliente`, `dim_producto` y `dim_fecha`. El total de facturación se calcula sumando importes de las líneas y controlando posibles duplicados.

[Unidad oficial](https://learn.microsoft.com/en-us/training/modules/design-implement-data-modeling-unity-catalog/9-choose-granularity)

## Unidad 10

### Choose managed vs external tables

Elegir **managed** significa delegar gestión física y mantenimiento a Unity Catalog, simplificando operaciones. Elegir **external** permite conservar los datos bajo control independiente, integrar fuentes compartidas o satisfacer requisitos específicos de ubicación. En ambos casos los permisos y metadatos se administran desde Unity Catalog si el objeto está registrado allí.

[Unidad oficial](https://learn.microsoft.com/en-us/training/modules/design-implement-data-modeling-unity-catalog/10-choose-managed-vs-unmanaged-tables)

## Unidad 11

### Design and implement a clustering strategy

El **clustering** aproxima físicamente registros que se consultan con filtros relacionados. **Liquid clustering** permite una organización flexible y administrada de los archivos Delta en escenarios compatibles. Debe evaluarse frente a estrategias antiguas de particionado físico y optimización por ordenación. La mejora depende de cardinalidad, volumen y patrón de consultas.

El principal error consiste en optimizar columnas que nunca aparecen en filtros o joins importantes. Primero examina planes de consulta, datos escaneados, file pruning y costes de mantenimiento. El diseño del almacenamiento no reemplaza la buena selección de columnas, uniones y granularidad.

[Unidad oficial](https://learn.microsoft.com/en-us/training/modules/design-implement-data-modeling-unity-catalog/11-design-clustering-strategy)

## Unidad 12

### Exercise - Design and Implement Data Modeling with Azure Databricks

Construye una dimensión de clientes que preserve cambios de ciudad y estado comercial, y unos hechos de pedidos de grano «una línea». Diseña su tratamiento CDC, reglas de no solapamiento, claves y consultas «estado en la fecha del pedido». Razona si un cambio de ciudad debe ser Tipo 1 o Tipo 2 según la necesidad empresarial.

[Unidad oficial](https://learn.microsoft.com/en-us/training/modules/design-implement-data-modeling-unity-catalog/12-exercise-design-implement-data-modeling-unity-catalog)

## Unidad 13

### Knowledge check

Considera una dimensión de clientes que debe reconstruir el segmento válido el día de una compra. El intervalo de validez de un SCD tipo 2 pertenece al negocio. El snapshot por versión de Delta permite consultar el estado físico de la tabla en un commit retenido. CDF expresa cambios de filas por commit. Para auditoría de largo plazo, materializa los cambios necesarios con su propia retención y registra el progreso del proceso consumidor. Un informe temporal debe aclarar si su fecha es la de negocio o la de registro del sistema.

[Unidad oficial](https://learn.microsoft.com/en-us/training/modules/design-implement-data-modeling-unity-catalog/13-knowledge-check)

## Unidad 14

### Summary

Fija primero la granularidad y la clave de negocio. La forma de conservar cambios se decide según las preguntas que tienen que responder los consumidores. Después elige el formato y la organización física según volumen, filtros, latencia y operaciones de mantenimiento. Prueba con intervalos contiguos, dos cambios de la misma clave, eventos fuera de orden y un caso de reintento. La consulta histórica debe seguir devolviendo como máximo una versión por clave y punto temporal.

[Unidad oficial](https://learn.microsoft.com/en-us/training/modules/design-implement-data-modeling-unity-catalog/14-summary)

## Desarrollo aplicado

### Tres conceptos que no deben confundirse

Un modelo **SCD tipo 2** conserva versiones de entidades de negocio; una tabla Delta con **time travel** permite consultar estados anteriores de la tabla que aún estén disponibles; **Change Data Feed** describe cambios realizados desde su activación y dentro de su periodo de disponibilidad. Los tres mecanismos pueden combinarse, pero resuelven problemas diferentes. Si el requisito es reconstruir el perfil del cliente de hace cinco años, un diseño SCD2 con conservación adecuada ofrece semántica de negocio más sólida que confiar únicamente en versiones físicas del almacenamiento.

### Diseñar correctamente intervalos de validez

Supongamos que un cliente cambia de ciudad el 15 de marzo. Una versión representa `[2026-01-01, 2026-03-15)` y la siguiente `[2026-03-15, infinito)`. El corchete inicial indica inclusión y el paréntesis final exclusión. Este convenio evita que dos versiones sean válidas simultáneamente en el instante de cambio. Para reconstruir el perfil a la fecha de una venta, se une la tabla de hechos con la dimensión usando clave de negocio y fecha del evento.

```sql
SELECT v.id_venta, v.fecha, c.ciudad
FROM estudio.gold.ventas v
JOIN estudio.gold.dim_cliente_hist c
  ON v.id_cliente = c.id_cliente
 AND v.fecha >= c.valid_from
 AND (c.valid_to IS NULL OR v.fecha < c.valid_to);
```

Esta consulta solo es correcta si los intervalos no se solapan y todas las ventas encuentran la versión que corresponde. Las cargas tardías plantean un problema adicional: si un cambio del 10 de marzo llega el día 20, hay que decidir si reabrir intervalos históricos y cómo corregir las asociaciones temporales.

### Distinguir clave de negocio y clave de versión

`id_cliente` identifica la entidad, mientras que una clave sustituta identifica una versión concreta. Una tabla de hechos puede guardar la clave sustituta asignada cuando ocurrió el evento. Si se reasignan versiones por un cambio tardío, puede ser necesario recalcular referencias. Establece la granularidad y las reglas de vigencia antes de diseñar las claves.

### Compromisos de rendimiento

Las consultas más frecuentes suelen filtrar por fecha, cliente o periodo. Pero particionar por cliente generaría miles o millones de directorios y archivos pequeños. Liquid clustering puede ayudar cuando las características de tabla y versión lo permiten; `OPTIMIZE`, políticas de mantenimiento y análisis de estadísticas completan el diseño. El objetivo no es ordenar por todas las columnas, sino mejorar poda de datos y reducir coste para consultas reales.

### Comprobar CDF y time travel sin confundirlos

```sql
ALTER TABLE estudio.silver.clientes
SET TBLPROPERTIES (delta.enableChangeDataFeed = true);

DESCRIBE HISTORY estudio.silver.clientes;

SELECT * FROM table_changes('estudio.silver.clientes', 1);
```

El número de versión `1` es ilustrativo: verifica qué versiones existen y desde cuándo se habilitó CDF. También comprueba que tu entorno admite la función y que no se han eliminado los datos necesarios por retención. Para time travel, `SELECT ... VERSION AS OF n` lee el estado de una versión retenida; para una auditoría duradera conserva tus propios datos historizados.

**Ejercicio:** diseña un dataset de cinco clientes con dos cambios de ciudad, un cambio tardío y un evento duplicado. Escribe las reglas de cierre e inserción de versiones y comprueba que la consulta por fecha siempre devuelve una sola versión por cliente. Añade una aserción que detecte intervalos solapados antes de publicar la tabla Gold.

### Caso resuelto · Qué significa «consultar el histórico»

Un cliente cambia de Madrid a Toledo el 1 de marzo. Una venta del 15 de febrero debe asociarse a Madrid aunque se procese en abril. Además, el equipo necesita consultar cómo estaba la tabla antes de una carga y detectar los cambios publicados en esa carga.

| Pregunta | Mecanismo | Dato que se consulta |
|---|---|---|
| ¿Dónde vivía el cliente al hacer la compra? | SCD tipo 2 | Intervalo de validez del perfil |
| ¿Cómo estaba la tabla justo antes de la carga? | Delta time travel | Snapshot de una versión retenida |
| ¿Qué filas cambió la carga? | Change Data Feed (CDF) | Eventos de cambio por commit |

### Prueba de escritorio con SQL

El ejemplo usa una tabla Delta nueva. Sustituye `mi_catalogo.mi_esquema` por un catálogo y un esquema de prácticas donde puedas crear tablas. Ejecuta los bloques en orden una sola vez. El cambio llega ya ordenado por fecha de negocio. El ejemplo se revisó estáticamente y queda pendiente de ejecución en un workspace.

```sql
CREATE TABLE mi_catalogo.mi_esquema.dp750_demo_clientes (
  cliente_id INT,
  ciudad STRING,
  valido_desde DATE,
  valido_hasta DATE
)
USING DELTA
TBLPROPERTIES ('delta.enableChangeDataFeed' = 'true');

INSERT INTO mi_catalogo.mi_esquema.dp750_demo_clientes
VALUES (7, 'Madrid', DATE '2026-01-01', NULL);
```

El `MERGE` recibe dos filas preparadas para el mismo cambio. Una coincide con la versión activa para cerrarla. La otra lleva una clave de unión nula para insertar la nueva versión en el mismo commit.

```sql
MERGE INTO mi_catalogo.mi_esquema.dp750_demo_clientes AS destino
USING (
  SELECT 7 AS clave_merge, 7 AS cliente_id,
         'Toledo' AS ciudad, DATE '2026-03-01' AS fecha_cambio
  UNION ALL
  SELECT CAST(NULL AS INT), 7, 'Toledo', DATE '2026-03-01'
) AS cambio
ON destino.cliente_id = cambio.clave_merge
AND destino.valido_hasta IS NULL
WHEN MATCHED THEN
  UPDATE SET destino.valido_hasta = cambio.fecha_cambio
WHEN NOT MATCHED AND cambio.clave_merge IS NULL THEN
  INSERT (cliente_id, ciudad, valido_desde, valido_hasta)
  VALUES (cambio.cliente_id, cambio.ciudad, cambio.fecha_cambio, NULL);
```

Para un flujo recurrente, deduplica los eventos por su identificador, establece reglas para llegadas tardías y valida que cada cliente conserva como máximo una versión vigente. El SQL anterior simula un único cambio y no implementa esas reglas.

```sql
-- Validez de negocio. El resultado esperado es Madrid.
SELECT ciudad
FROM mi_catalogo.mi_esquema.dp750_demo_clientes
WHERE cliente_id = 7
  AND valido_desde <= DATE '2026-02-15'
  AND (valido_hasta IS NULL OR DATE '2026-02-15' < valido_hasta);

DESCRIBE HISTORY mi_catalogo.mi_esquema.dp750_demo_clientes;
```

En una tabla recién creada con esas tres operaciones, se esperan las versiones 0 (creación), 1 (inserción) y 2 (MERGE). Confirma los números con el historial y ajusta las consultas si difieren.

```sql
-- Snapshot anterior al MERGE. Se espera Madrid como versión abierta.
SELECT *
FROM mi_catalogo.mi_esquema.dp750_demo_clientes VERSION AS OF 1;

-- Cambios del MERGE. Se esperan preimagen y postimagen de Madrid
-- y la inserción de Toledo.
SELECT cliente_id, ciudad, valido_hasta, _change_type, _commit_version
FROM table_changes('mi_catalogo.mi_esquema.dp750_demo_clientes', 2, 2);
```

### Repaso de examen

1. **¿Qué devuelve la consulta de negocio el 1 de marzo?** Toledo. El límite de salida de Madrid es exclusivo y coincide con el inicio de Toledo.
2. **¿Puedo usar `DESCRIBE HISTORY` para leer todas las filas de una versión?** Utiliza `VERSION AS OF` para leer el snapshot. `DESCRIBE HISTORY` sirve para localizar el commit y ver sus metadatos.
3. **¿Cómo conservaría los eventos de CDF para auditoría durante años?** Materializaría el feed en una tabla de auditoría con su política de retención y un proceso incremental que persista el avance.
4. **¿Por qué conviene SCD 2 si la pregunta habla del perfil en el momento de la venta?** La dimensión guarda la vigencia de negocio. Una corrección cargada en abril puede actualizar el perfil aplicable a febrero sin cambiar qué datos estaban físicamente disponibles en el commit de febrero.

- [Historial y time travel de Delta](https://learn.microsoft.com/en-us/azure/databricks/delta/history)
- [Documentación de Change Data Feed](https://learn.microsoft.com/en-us/azure/databricks/tables/features/change-data-feed)


## Laboratorio y comprobaciones

**Ejercicio oficial:** [Abrir las instrucciones de MicrosoftLearning](https://microsoftlearning.github.io/DP-750T00-Implement-Data-Engineering-Solutions-using-Azure-Databricks/Instructions/Labs/06-design-implement-data-modeling-unity-catalog.html)  
**Notebook oficial:** [Importar o consultar notebook `.ipynb`](https://raw.githubusercontent.com/MicrosoftLearning/DP-750T00-Implement-Data-Engineering-Solutions-using-Azure-Databricks/refs/heads/main/Allfiles/06-design-implement-data-modeling-unity-catalog.ipynb)  
**Naturaleza de esta sección:** itinerario explicado con redacción propia y comprobaciones de aprendizaje. Las celdas exactas y el material de origen están en el notebook oficial enlazado; no se reproducen íntegramente aquí.


### Caso práctico: Northbank Financial — modelo histórico y rendimiento

**Escenario:** un banco necesita reconstruir el perfil de un cliente al producirse una transacción, auditar los cambios y consultar versiones anteriores sin recurrir inmediatamente a copias de seguridad.

**Fases reproducibles:**

1. Importa el notebook del caso, con el catálogo `banking_lab`. Prepara una dimensión de clientes y una tabla de hechos de transacciones Delta en Unity Catalog.
2. En la tabla de hechos define **liquid clustering** sobre columnas acordes a los predicados habituales. No presupongas que una combinación de particiones manuales y clustering es siempre la mejor opción.
3. Modela **SCD tipo 2** con clave de negocio del cliente, un identificador de versión, `valid_from`, `valid_to` y una marca de fila vigente. La carga inicial debe dejar una versión activa por cliente.
4. Simula un cambio de ciudad, segmento u otro atributo. Mediante una operación transaccional o flujo controlado, **cierra la versión anterior e inserta otra nueva**. Un `MERGE` simple que solo actualice el registro existente **no conserva SCD 2**.
5. Recupera el perfil válido en una fecha consultando el intervalo de validez; comprueba el caso límite de cierre y apertura de versiones.
6. Habilita **Change Data Feed** en una tabla Delta compatible. Produce al menos un cambio posterior a la activación y consulta `table_changes()` para identificar tipos de cambio y versiones del commit.
7. Inspecciona `DESCRIBE HISTORY`, utiliza `VERSION AS OF` para leer una versión disponible y practica `RESTORE TABLE` **solo sobre datos descartables**.

**Prueba conceptual esencial:** SCD 2 mantiene un historial de negocio de larga duración; time travel conserva versiones físicas mientras el historial Delta y los archivos necesarios estén retenidos; CDF aporta un flujo de cambios y depende de su propia disponibilidad/retención. Son necesidades distintas y pueden convivir.

**Verificación en interfaz:** entra en Catalog Explorer, consulta la ubicación managed de las tablas y confirma la configuración de clustering.

### Evidencias por objetivo

Tras el primer ejercicio abre Catalog Explorer y confirma el tipo managed de las tablas y las columnas de clustering de la tabla de transacciones. En el notebook prepara una consulta de dimensión para el día previo al cambio y otra para el día del cambio. Comprueba que cada consulta devuelve una sola versión y conserva la evidencia de intervalos. Activa CDF antes de la operación que necesitas observar y anota la versión con `DESCRIBE HISTORY`. Después compara el resultado de `table_changes` con la consulta por versión. En el conjunto descartable, si practicas RESTORE, confirma que el historial registra un commit nuevo. Retira los recursos de laboratorio según sus instrucciones y utiliza la práctica independiente del apartado de desarrollo aplicado si quieres repetir la comparación con solo dos perfiles.

## Fuentes y navegación

- [Fuentes de la certificación](../FUENTES.md)
- [Preparación y limpieza del entorno](../PREPARACION-LABORATORIOS.md)

[Índice DP-750](../README.md) · [← Gobernar objetos de Unity Catalog](../07-gobernar-objetos-de-unity-catalog/README.md) · [Ingerir datos en Unity Catalog →](../09-ingesta-de-datos/README.md)
