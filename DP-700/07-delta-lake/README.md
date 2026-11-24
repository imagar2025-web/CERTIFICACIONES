# 07 · Work with Delta Lake tables in Microsoft Fabric

> **Certificación:** DP-700. **Ruta:** Implement a Lakehouse with Microsoft Fabric. [Microsoft Learn](https://learn.microsoft.com/en-us/training/modules/work-delta-lake-tables-fabric/) · [archivos oficiales](https://github.com/MicrosoftDocs/learn/tree/main/learn-pr/wwl/work-delta-lake-tables-fabric) · [laboratorio de MicrosoftLearning](https://github.com/MicrosoftLearning/mslearn-fabric/blob/main/Instructions/Labs/03-delta-lake.md). **Revisión:** noviembre de 2026. **Estado:** teoría y laboratorio documentados; sin ejecución real.

## Índice

1. [Introducción: del archivo a la tabla transaccional](#1-introducción-del-archivo-a-la-tabla-transaccional)
2. [Understand Delta Lake](#2-understand-delta-lake)
3. [Create Delta tables](#3-create-delta-tables)
4. [Optimize Delta tables](#4-optimize-delta-tables)
5. [Work with Delta data](#5-work-with-delta-data)
6. [Use Delta Lake for streaming](#6-use-delta-lake-for-streaming)
7. [Laboratorio oficial](#7-laboratorio-oficial)
8. [Knowledge check y summary](#8-knowledge-check-y-summary)
9. [Fuentes y control](#9-fuentes-y-control)

## 1. Introducción: del archivo a la tabla transaccional

Los archivos Parquet son eficientes para consultas analíticas, pero un directorio de ficheros por sí solo no resuelve todas las necesidades de una tabla actualizable. Un proceso puede escribir nuevos ficheros y fallar antes de completar la carga; otra aplicación puede leer parcialmente una versión en construcción; y una eliminación no debe confundirse con borrar arbitrariamente archivos. **Delta Lake** añade una capa de gestión transaccional sobre archivos para ofrecer semántica de tablas.

En Microsoft Fabric, el formato Delta es la base de las tablas de Lakehouse y se integra con Spark y las capacidades de consulta compatibles. Para DP-700, entender los componentes de Delta no equivale a memorizar el comando \`saveAsTable\`: hay que comprender el transaction log, las operaciones de escritura, las versiones, el mantenimiento de archivos, la retención y el tratamiento de datos en streaming.

## 2. Understand Delta Lake

### 2.1. Organización física y lógica

Una tabla Delta consiste en archivos de datos —habitualmente Parquet— y un registro de transacciones bajo \`_delta_log\`. El log describe acciones y metadatos necesarios para construir una versión coherente de la tabla. Cada commit confirmado define un estado lógico, y los lectores utilizan el log para identificar qué archivos forman parte de la versión solicitada.

~~~mermaid
flowchart LR
  T["Tabla Delta"] --> L["_delta_log / commits"]
  T --> F["Parquet data files"]
  L --> V["Versión N"]
  V --> Q["Lectura consistente"]
  F --> Q
~~~

*Diagrama conceptual de elaboración propia*. No significa que la tabla exista físicamente como un único archivo ni que cada actualización modifique en sitio todos los Parquet previos.

### 2.2. Atomicidad y aislamiento

Delta aporta garantías transaccionales sobre las operaciones admitidas en la tabla. Una escritura confirmada aparece como un nuevo estado, mientras una operación incompleta no debería publicarse como commit válido. Eso protege la coherencia de una tabla individual, pero **no convierte automáticamente una secuencia de actualizaciones de distintas tablas en una transacción global**. En una arquitectura de múltiples capas conviene registrar lotes y garantizar que las versiones publicadas son compatibles entre sí.

### 2.3. Schema enforcement y cambios de esquema

Un esquema identifica tipos y nombres de columnas. Insertar una cadena en una columna decimal o sustituir el tipo de una clave puede ser incompatible con el contrato vigente. No debe habilitarse evolución de esquema de forma indiscriminada: permitir columnas nuevas sin controles puede romper modelos semánticos o permitir que un error tipográfico produzca un campo adicional.

El modelo debe incluir una estrategia de cambios: contrato conocido, revisión de columnas opcionales, migraciones cuando correspondan y validación de consumidores. Las reglas de nulos y unicidad no quedan resueltas por el simple hecho de escribir en Delta; tienen que implementarse mediante los controles compatibles o en el pipeline.

**Fuente:** [Understand Delta Lake](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/work-delta-lake-tables-fabric/includes/2-understand-delta-lake.md).

## 3. Create Delta tables

### 3.1. DataFrame → tabla

El patrón habitual comienza con un DataFrame de Spark, opcionalmente con esquema explícito, y su escritura en Delta:

~~~python
from pyspark.sql.types import (
    StructType, StructField, IntegerType, StringType, DoubleType
)

schema = (StructType()
    .add("ProductID", IntegerType(), True)
    .add("ProductName", StringType(), True)
    .add("Category", StringType(), True)
    .add("ListPrice", DoubleType(), True))

products = (spark.read.format("csv")
    .option("header", True)
    .schema(schema)
    .load("Files/products/products.csv"))

products.write.format("delta").mode("overwrite").saveAsTable("dbo.products_table")
~~~

El CSV corresponde a la estructura utilizada en el laboratorio oficial. En una nueva ejecución, el modo \`overwrite\` reemplaza la tabla y sus datos según la semántica del motor; para seguir literalmente la primera creación del laboratorio, usar su comando original sin cambiar el modo. La distinción importa porque overwrite puede afectar versiones y datos históricos accesibles.

### 3.2. Managed vs external tables

Una **managed table** almacena sus archivos en una ubicación administrada por el catálogo; su ciclo de vida está asociado a la tabla. Una **external table** mantiene metadatos que apuntan a datos ubicados fuera de la ruta gestionada. El efecto de borrar uno u otro objeto puede ser distinto. Antes de eliminar una tabla, determinar qué datos se van a eliminar y si existen consumidores, shortcuts o procesos que dependan de esos ficheros.

### 3.3. Spark SQL y DeltaTable API

Se pueden crear y manipular tablas Delta con Spark SQL y APIs del entorno Spark. Para crear una tabla con un esquema definido se puede utilizar DDL:

~~~sql
CREATE TABLE IF NOT EXISTS dbo.productos_demo (
    ProductID INT,
    ProductName STRING,
    Category STRING,
    ListPrice DOUBLE
)
USING DELTA;
~~~

El ejemplo presupone un Lakehouse con schema \`dbo\`. La estructura de la tabla es lógica: los datos se guardarán en los archivos físicos y el registro de la tabla. También puede emplearse una API de creación de tablas que permita configurar columnas y propiedades. El alumno debe comprender el concepto de catálogo y ubicación, no limitarse a usar botones.

**Fuente:** [Create Delta tables](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/work-delta-lake-tables-fabric/includes/3-create-delta-tables.md).

## 4. Optimize Delta tables

### 4.1. El problema de los archivos pequeños

Los motores distribuidos escriben datos por tareas; muchos lotes pequeños pueden terminar produciendo multitud de ficheros Parquet de tamaño reducido. Abrir y planificar miles de ficheros incrementa trabajo de metadatos y reduce eficiencia de lectura. Por tanto, no es suficiente con ver que la tabla contiene la cantidad correcta de filas: también es importante cómo se distribuye físicamente.

**Optimize Write** busca mejorar el patrón de escritura para reducir la aparición de archivos ineficientes. La operación de mantenimiento **OPTIMIZE** reorganiza archivos cuando resulta apropiado, y **V-Order** puede mejorar disposición de datos para determinados patrones de consulta admitidos en Fabric. Es necesario distinguir qué opción está activa por defecto, qué versiones de runtime la admiten y qué costes de cómputo introduce.

### 4.2. OPTIMIZE frente a VACUUM

**OPTIMIZE** reescribe u organiza archivos para mejorar el rendimiento de ciertas consultas; **VACUUM** gestiona la eliminación de archivos ya no necesarios según las políticas de retención. No son equivalentes. Ejecutar VACUUM sin evaluar el período de retención puede reducir la posibilidad de recuperar versiones históricas mediante time travel y afectar a lectores o procesos atrasados. Nunca usar VACUUM como «limpieza gratuita» de un laboratorio sin comprender sus consecuencias.

### 4.3. Particionado y trade-offs

El particionado físico puede beneficiar lecturas filtradas por columnas con valores repetidos y distribución razonable, como año o mes. Particionar por un identificador único crea carpetas y archivos excesivos. Incluso una partición mensual puede ser inadecuada si el dataset tiene muy pocas filas. La decisión debe estar basada en volumen y consultas, no en aplicar particionado a todas las tablas.

**Fuente:** [Optimize Delta tables](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/work-delta-lake-tables-fabric/includes/3b-optimize-delta-tables.md).

## 5. Work with Delta data

### 5.1. UPDATE, INSERT y DELETE

Delta permite realizar operaciones de modificación de tablas, como UPDATE y DELETE, cuando la superficie y los permisos lo admiten. Por ejemplo, desde una celda de Spark SQL:

~~~sql
%%sql
UPDATE dbo.products_table
SET ListPrice = ListPrice * 0.9
WHERE Category = 'Mountain Bikes';
~~~

La sentencia modifica únicamente las filas de la categoría escogida en el estado actual de la tabla. Es aconsejable verificar primero qué filas selecciona la cláusula WHERE con un SELECT. No ejecutar actualizaciones destructivas en datasets compartidos o productivos sin un plan de prueba y reversión.

### 5.2. Historial y time travel

El **transaction log** conserva versiones de la tabla. Puede consultarse un historial de commits y, mientras se retengan los datos necesarios, consultar estados anteriores.

~~~sql
%%sql
DESCRIBE HISTORY dbo.products_table;
~~~

~~~sql
%%sql
SELECT ProductID, ProductName, ListPrice
FROM dbo.products_table VERSION AS OF 0;
~~~

Una versión inicial \`0\` solo existirá después de la primera escritura; el laboratorio oficial suele utilizarla para comparar el estado original con una actualización posterior. **Time travel no es una solución de historificación de negocio permanente por sí sola.** Las versiones físicas pueden quedar sujetas a retención y VACUUM. Si se necesita conservar el estado histórico de perfiles de clientes durante años, diseñar una estrategia de historificación como SCD Type 2 u otra apropiada, además de cualquier historial transaccional.

### 5.3. Delta API

La API de Delta permite aplicar operaciones sobre tablas desde PySpark, por ejemplo, actualizar valores o gestionar merges con claves. El uso de MERGE no garantiza unicidad si las claves no se validan o si varias filas de origen coinciden con la misma fila de destino. Diseñar una carga incremental requiere normalizar y desduplicar las claves antes de la escritura.

**Fuente:** [Work with Delta data](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/work-delta-lake-tables-fabric/includes/4-work-delta-data.md).

## 6. Use Delta Lake for streaming

### 6.1. Structured Streaming

**Structured Streaming** ofrece una API declarativa para procesar datos a medida que llegan, con una semántica de ejecución configurable. La fuente puede ser una tabla Delta u otra fuente compatible, y el destino puede ser una tabla Delta. La solución mantiene estado de progreso a través de un **checkpoint** cuando se configura, lo que permite avanzar y recuperarse de determinados fallos.

~~~python
stream = spark.readStream.format("delta").table("dbo.eventos_entrada")

query = (stream.writeStream
    .format("delta")
    .option("checkpointLocation", "Files/checkpoints/eventos_salida")
    .outputMode("append")
    .toTable("dbo.eventos_salida"))
~~~

*Ejemplo complementario que presupone que existe \`dbo.eventos_entrada\`.* No ejecutarlo sin adaptar la tabla y la ruta del checkpoint a los permisos y al uso real. Elegir el modo de salida según las transformaciones y el sink: no todas las operaciones soportan todos los modos.

### 6.2. Checkpoints y reintentos

Un checkpoint almacena información esencial para continuar el procesamiento. Cambiarlo arbitrariamente o eliminarlo puede reiniciar el avance lógico y provocar duplicaciones según la fuente y la escritura. Se debe tratar como un componente persistente de la aplicación, con ubicación estable por query y entorno. No reutilizar el mismo checkpoint entre dos consultas lógicamente distintas.

### 6.3. Detener el stream

En un entorno de laboratorio, las consultas persistentes consumen recursos mientras están activas. Una vez realizadas las verificaciones, es importante detener las consultas:

~~~python
query.stop()
~~~

A continuación comprobar que no quedan streams de la práctica activos. Detener el stream no elimina automáticamente todos los archivos, tablas ni checkpoints asociados.

**Fuente:** [Streaming con Delta](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/work-delta-lake-tables-fabric/includes/5-use-delta-lake-streaming-data.md).

## 7. Laboratorio oficial

**Fuente:** [03-delta-lake.md](https://github.com/MicrosoftLearning/mslearn-fabric/blob/main/Instructions/Labs/03-delta-lake.md) · [instrucciones navegables](https://microsoftlearning.github.io/mslearn-fabric/Instructions/Labs/03-delta-lake.html). **Duración oficial aproximada:** 45 minutos. No se ha ejecutado; se documentan operaciones y verificaciones esperadas.

### 7.1. Escenario y requisitos

La práctica crea el Workspace \`dp_workspace\` con capacidad Trial o de pago, añade un Lakehouse \`delta_lakehouse\`, importa un CSV de productos, registra una tabla Delta, modifica precios, inspecciona el historial y trabaja con streaming de eventos de dispositivos simulados. Se necesitan permisos de creación, Spark habilitado y acceso al origen oficial.

**Dataset real publicado:** [products.csv](https://github.com/MicrosoftLearning/dp-data/raw/main/products.csv). Descargarlo y guardarlo localmente, crear la carpeta \`Files/products\` dentro del Lakehouse y subir el CSV. La carpeta debe tener minúsculas según el laboratorio. No se han comprobado recuentos de registros.

### 7.2. Leer el CSV con esquema explícito

Crear Notebook conectado al Lakehouse y definir \`StructType\` con ProductID (Integer), ProductName (String), Category (String) y ListPrice (Double). Leer \`Files/products/products.csv\` con \`header=true\` y comprobar el esquema y la vista previa.

~~~python
from pyspark.sql.types import StructType, IntegerType, StringType, DoubleType

schema = (StructType()
    .add("ProductID", IntegerType(), True)
    .add("ProductName", StringType(), True)
    .add("Category", StringType(), True)
    .add("ListPrice", DoubleType(), True))

df = (spark.read.format("csv").option("header", True)
    .schema(schema)
    .load("Files/products/products.csv"))
display(df.limit(20))
~~~

Si falla, revisar nombre y ruta de los ficheros y la asociación del Lakehouse. Un fallo de esquema puede ser provocado por columna ausente o separador distinto al esperado.

### 7.3. Crear la tabla y verificar

~~~python
df.write.format("delta").saveAsTable("dbo.products_table")
~~~

Actualizar Tables en el explorador; debe mostrarse \`products_table\` con icono Delta. Si el comando falla por existir la tabla, decidir si se está repitiendo la práctica o si debe usarse un nombre nuevo; no sobrescribir indiscriminadamente. Si se utiliza otro schema, adaptar la referencia conforme a la configuración del Lakehouse.

### 7.4. Modificar y consultar versiones

Ejecutar la sentencia UPDATE del apartado 5.1 sobre la categoría Mountain Bikes; después \`DESCRIBE HISTORY dbo.products_table\`. El tutorial explica que una primera creación seguida de una actualización genera versiones que pueden consultarse mediante \`VERSION AS OF\`. Para contrastar el precio original con el actual, emplear el join por ProductID indicado en las instrucciones oficiales.

**Verificación:** deben aparecer operaciones en el historial y una diferencia coherente en el precio actualizado, pero no se atribuyen versiones exactas si se han realizado escrituras adicionales. El número final de versión depende de lo que haya ocurrido realmente en el workspace.

### 7.5. Spark SQL y streaming

El laboratorio realiza análisis con vistas temporales y consultas SQL, luego simula eventos IoT creando ficheros en un directorio de streaming, usa \`readStream\` para consumirlos y escribe en una tabla Delta con checkpoint. El ejemplo oficial emplea un destino similar a \`Tables/dbo/iotdevicedata\` y checkpoint bajo \`Files/delta/checkpoint\`.

Los pasos importantes son separar la fuente del sink, aplicar un esquema al leer datos de streaming, mantener el checkpoint y comprobar que los nuevos archivos aparecen en consultas posteriores. Una vez comprobado, detener la instancia de la consulta \`deltastream.stop()\`, tal como establece el laboratorio. **No afirmar que los eventos se han recibido sin ejecutar el stream.**

### 7.6. Diagnóstico y limpieza

| Síntoma | Causa plausible | Corrección |
|---|---|---|
| No encuentra CSV | Ruta o nombre incorrectos | Revisar \`Files/products/products.csv\` |
| \`Table already exists\` | Segunda ejecución de creación | Usar tabla de prueba nueva o planificar reemplazo |
| UPDATE afecta a cero filas | Condición sin coincidencias | Inspeccionar \`Category\` |
| Time travel no encuentra versión | Número inválido, historial o retención | Revisar \`DESCRIBE HISTORY\` |
| Streaming no recibe archivos | Directorio fuente incorrecto | Revisar escritura y esquema |
| \`checkpointLocation\` en conflicto | Se reutiliza checkpoint de otra query | Aislar ruta por stream |
| Error de escritura en Tables | Lakehouse no conectado o permisos | Revisar asociación y roles |
| Sesión consume recursos | Stream no detenido | \`deltastream.stop()\` y comprobación |

Al terminar, borrar el Workspace si se creó exclusivamente para la práctica; en otro caso, eliminar solo los objetos propios. No ejecutar VACUUM o borrar checkpoints de procesos ajenos. La eliminación de un Workspace no sustituye revisar el consumo de una capacidad dedicada de pago.

## 8. Knowledge check y summary

Preguntas de elaboración propia:

1. **¿Por qué no basta Parquet para mantener una tabla transaccional?** No contiene por sí solo la semántica del log Delta.
2. **¿Qué registra \`_delta_log\`?** Acciones y metadatos que permiten construir versiones coherentes.
3. **¿Qué diferencia hay entre managed y external?** Quién gestiona la ubicación y ciclo de vida de los datos.
4. **¿OPTIMIZE y VACUUM hacen lo mismo?** No; reorganización de ficheros frente a eliminación de archivos elegibles.
5. **¿Qué riesgo tiene VACUUM?** Perder capacidad para consultar versiones históricas según retención.
6. **¿Time travel asegura historificación ilimitada?** No, depende de conservar los datos.
7. **¿Por qué desduplicar antes de MERGE?** Para evitar ambigüedad de claves y resultados incorrectos.
8. **¿Qué permite Structured Streaming?** Procesar fuentes continuas y escribir resultados con progreso controlado.
9. **¿Para qué sirve un checkpoint?** Para registrar el avance necesario para recuperación.
10. **¿Qué debe hacerse al finalizar un streaming de laboratorio?** Detener la query y limpiar recursos apropiados.

**Síntesis:** Delta Lake ofrece tablas transaccionales, versiones y soporte de actualización sobre ficheros. Fabric simplifica su uso, pero no elimina la necesidad de esquemas, permisos, historificación de negocio, particionado racional, mantenimiento y estrategias de reintento.

## 9. Fuentes y control

- [Índice oficial de nueve unidades](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/work-delta-lake-tables-fabric/index.yml).
- [Fundamentos Delta](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/work-delta-lake-tables-fabric/includes/2-understand-delta-lake.md).
- [Creación de tablas](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/work-delta-lake-tables-fabric/includes/3-create-delta-tables.md).
- [Optimización](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/work-delta-lake-tables-fabric/includes/3b-optimize-delta-tables.md).
- [Operaciones y versiones](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/work-delta-lake-tables-fabric/includes/4-work-delta-data.md).
- [Streaming](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/work-delta-lake-tables-fabric/includes/5-use-delta-lake-streaming-data.md).
- [Laboratorio oficial DP-700](https://github.com/MicrosoftLearning/mslearn-fabric/blob/main/Instructions/Labs/03-delta-lake.md).

**Control:** cubiertos fundamentos, creación, optimización, modificación, streaming, laboratorio, evaluación y síntesis. Sin ejecución de laboratorio, capturas ni resultados medidos.