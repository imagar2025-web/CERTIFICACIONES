# 03 · Use Apache Spark in Microsoft Fabric

> **Certificación:** DP-700. **Ruta:** Ingest data with Microsoft Fabric (compartido con Implement a Lakehouse). **Fuente:** [módulo de Microsoft Learn](https://learn.microsoft.com/en-us/training/modules/use-apache-spark-work-files-lakehouse/) · [index.yml y fuentes originales](https://github.com/MicrosoftDocs/learn/tree/main/learn-pr/wwl/use-apache-spark-work-files-lakehouse) · [laboratorio](https://github.com/MicrosoftLearning/mslearn-fabric/blob/main/Instructions/Labs/02-analyze-spark.md).  
> **Estado:** desarrollo de nueve unidades del módulo con laboratorio documentado. **Ejecución real:** no realizada.

## Índice

1. [Introduction](#1-introduction)
2. [Explore Apache Spark in Fabric](#2-explore-apache-spark-in-fabric)
3. [Use Spark code](#3-use-spark-code)
4. [Work with DataFrames](#4-work-with-dataframes)
5. [Use Spark SQL](#5-use-spark-sql)
6. [Visualize data](#6-visualize-data)
7. [Exercise: Analyze data with Apache Spark](#7-exercise-analyze-data-with-apache-spark)
8. [Knowledge check razonado](#8-knowledge-check-razonado)
9. [Summary](#9-summary)
10. [Fuentes y control de calidad](#10-fuentes-y-control-de-calidad)

## 1. Introduction

Apache Spark es un motor de procesamiento distribuido. Permite analizar conjuntos de datos cuya carga de trabajo puede dividirse entre distintos procesos y nodos. Microsoft Fabric incorpora Spark gestionado para procesar información almacenada en OneLake, especialmente mediante Notebooks y Spark Job Definitions. Conviene separar tres conceptos: **almacenamiento** (Lakehouse y sus ficheros/tablas), **cómputo** (pool, sesiones, executors) y **orquestación** (Pipeline, programación, automatización). Spark ejecuta transformaciones; no es en sí mismo un catálogo corporativo, un servicio de visualización de informes ni una garantía automática de calidad.

El módulo oficial cubre Spark pools, sesiones y runtimes; Notebooks y Spark jobs; DataFrames; Spark SQL; visualizaciones; laboratorio, evaluación y síntesis. Aunque Fabric automatiza buena parte de la infraestructura, un ingeniero de datos debe seguir comprendiendo planificación de tareas, paralelismo, particionado, formatos de almacenamiento, transformaciones y costes.

## 2. Explore Apache Spark in Fabric

### 2.1. Arquitectura: driver y executors

Un proceso Spark se coordina desde el **driver**. El driver crea el plan de ejecución de la aplicación, gestiona su distribución y recopila los metadatos necesarios. Los **executors** ejecutan tareas en workers y procesan particiones de los datos. En Fabric, la infraestructura se organiza en Spark pools. Existe un **starter pool** para sesiones rápidas y, cuando está habilitado, se pueden configurar pools personalizados. Los pools no deben confundirse con una tabla del Lakehouse: son recursos de cómputo.

~~~mermaid
flowchart TB
  N["Notebook / Spark Job"] --> D["Driver"]
  D --> P["Spark execution plan"]
  P --> E1["Executor 1"]
  P --> E2["Executor 2"]
  P --> E3["Executor 3"]
  E1 --> O["OneLake / Lakehouse"]
  E2 --> O
  E3 --> O
~~~

**Diagrama propio**: representación conceptual, no topología física literal del servicio. El driver no realiza por sí mismo todas las operaciones sobre cada fila; los executors procesan particiones. Si un DataFrame se reduce con \`collect()\` o \`toPandas()\`, sus datos se trasladan al proceso local que llama al resultado, con el consiguiente riesgo de memoria.

### 2.2. Pools, autoscale y dynamic allocation

El **autoscale** adapta la cantidad de nodos del pool dentro de límites configurados. La **dynamic allocation** ajusta los recursos de ejecución a la carga de trabajo según la configuración disponible. No son sinónimos: uno trata el aprovisionamiento de capacidad del pool y el otro la asignación de executors. La arquitectura debe dimensionarse según datos, paralelismo, distribución de tamaños de particiones, concurrencia y presupuesto de capacidad.

Un pool mayor no garantiza menor tiempo. Si el volumen es muy pequeño, el coste de iniciar Spark o coordinar tareas puede superar las ventajas del paralelismo. En el extremo opuesto, un **skew** de datos puede asignar a un executor una partición enorme y dejar otros inactivos. Antes de aumentar recursos, revisar el plan y el desequilibrio de particiones.

### 2.3. Runtime y Environments

El **runtime** determina versiones compatibles de Apache Spark, Python y bibliotecas de la plataforma. Un **Environment** permite fijar configuraciones y dependencias que utiliza la sesión. Reproducir un Notebook exige más que conservar código: las diferencias entre versiones de librerías pueden alterar comportamiento y resultados. Por ese motivo conviene documentar runtime, versión de dependencias y configuraciones no predeterminadas.

Fabric ofrece opciones para optimizar la ejecución, como **native execution engine** en cargas compatibles y modos de alta concurrencia cuando la funcionalidad está disponible. No se debe habilitar un motor solo por su nombre: hay que comprobar la compatibilidad real con las operaciones y formatos del Notebook, medir rendimientos y evaluar los límites de la capacidad. La concurrencia también obliga a revisar aislamiento y efectos de escrituras simultáneas a la misma tabla.

**Fuente:** [Explore Spark](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/use-apache-spark-work-files-lakehouse/includes/2-spark.md).

## 3. Use Spark code

### 3.1. Notebook frente a Spark Job Definition

Los **Notebooks** son adecuados para exploración interactiva, pruebas, explicación junto al código y depuración. Incluyen celdas de código y Markdown, y permiten utilizar varios lenguajes admitidos. Una **Spark Job Definition** sirve para ejecutar scripts dentro de un proceso más automatizado y repetible; puede referenciar un fichero principal y otros recursos necesarios.

La elección no debería hacerse solo por preferencia de interfaz. Para un prototipo de transformación resulta práctico un Notebook, especialmente si hay que inspeccionar registros. En producción interesa gestionar el código como recurso versionado, parametrizar rutas y tablas, capturar excepciones y asegurarse de que la ejecución no depende del estado accidental de celdas ejecutadas manualmente en orden diferente.

### 3.2. Sesiones y magics

Una sesión Spark aloja el contexto donde se ejecutan instrucciones. En Fabric, una celda puede utilizar \`%%pyspark\`, \`%%spark\` para Scala o \`%%sql\` para expresiones Spark SQL. Estas magics seleccionan el intérprete; no convierten mágicamente una query T-SQL de Warehouse en una query Spark SQL compatible. Los dialectos SQL y las capacidades de escritura no son idénticos.

~~~python
# Celda PySpark en Notebook asociado a un Lakehouse
from pyspark.sql import functions as F

df = spark.createDataFrame(
    [(1, "Madrid", 30.5), (2, "Valencia", 41.2), (3, "Madrid", 12.0)],
    ["id", "ciudad", "importe"]
)
resultado = df.groupBy("ciudad").agg(F.sum("importe").alias("total"))
display(resultado)
~~~

El ejemplo es propio y no requiere una fuente externa. La conversión de una lista pequeña en DataFrame es apropiada para aprender, pero un origen real de gran volumen debería leerse desde almacenamiento distribuido. La operación \`groupBy\` puede necesitar intercambio de datos entre executors (*shuffle*) y, por tanto, debe evaluarse de forma distinta de un simple \`select\`.

**Fuente:** [Spark code](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/use-apache-spark-work-files-lakehouse/includes/3-spark-code.md).

## 4. Work with DataFrames

### 4.1. RDD y DataFrame

Un **RDD** es una colección distribuida de elementos; los **DataFrames** ofrecen una abstracción tabular con esquema y optimizaciones de Spark SQL. El motor puede optimizar expresiones declarativas y evitar operaciones que no son necesarias para el resultado. En ingeniería de datos se prefiere normalmente trabajar con DataFrames y funciones integradas antes que realizar procesamiento fila a fila desde Python, para conservar oportunidades de optimización y distribución.

Un DataFrame se transforma mediante llamadas como \`select\`, \`filter\`, \`withColumn\`, \`groupBy\` o \`join\`. Estas llamadas expresan un plan; en general la evaluación es **lazy** y se materializa cuando una acción solicita resultados o se escribe una salida. Hay diferencias entre acciones, transformaciones y operaciones de visualización; comprenderlas evita medir erróneamente solo el tiempo que tarda en definirse una variable.

### 4.2. Lectura con esquema explícito

Leer un CSV con \`inferSchema\` resulta cómodo, pero puede generar más trabajo y errores de inferencia. Para una ingesta estable, definir el contrato de esquema evita cambios inesperados. Por ejemplo, un identificador aparentemente numérico puede contener ceros iniciales, por lo que debería conservarse como texto. Los importes deben utilizar un tipo apropiado a la precisión que se exige; en el ejemplo se emplea \`DecimalType\` para representar importes con dos decimales.

~~~python
from pyspark.sql.types import (
    StructType, StructField, StringType, DateType, DecimalType, IntegerType
)

schema = StructType([
    StructField("SalesOrderNumber", StringType(), False),
    StructField("OrderDate", DateType(), True),
    StructField("Quantity", IntegerType(), True),
    StructField("UnitPrice", DecimalType(18, 2), True),
])

pedidos = (spark.read.format("csv")
    .option("header", True)
    .option("dateFormat", "yyyy-MM-dd")
    .schema(schema)
    .load("Files/pedidos/pedidos.csv"))

pedidos.printSchema()
~~~

*Código complementario propio, no correspondiente al fichero exacto del laboratorio.* La ruta presupone un fichero de ejemplo creado por el usuario. El esquema debe coincidir con las columnas reales. Un esquema declarado como no nullable en el lector no reemplaza siempre una validación completa de calidad: los registros mal formados y el comportamiento del parser requieren controles adicionales y, si es necesario, una zona de cuarentena.

### 4.3. Transformaciones, agregación y joins

~~~python
from pyspark.sql import functions as F

ventas_validas = (pedidos
    .filter(F.col("Quantity") > 0)
    .filter(F.col("UnitPrice").isNotNull())
    .withColumn("importe", F.col("Quantity") * F.col("UnitPrice"))
    .withColumn("anio", F.year("OrderDate")))

por_anio = ventas_validas.groupBy("anio").agg(
    F.count("*").alias("filas"),
    F.sum("importe").alias("ventas")
)
~~~

La transformación crea un campo importe y reduce las filas según las reglas definidas. Un filtro de cantidad positiva no implica que un pedido cancelado esté eliminado: esa es una regla de negocio adicional que debe existir en el esquema. Al realizar un join, analizar si la cardinalidad es uno-a-uno, muchos-a-uno o muchos-a-muchos. Si una dimensión contiene varias filas para una clave que se suponía única, el join puede inflar los importes sin lanzar excepciones.

### 4.4. Escribir Parquet y Delta

**Parquet** es un formato columnar muy utilizado para intercambio y procesamiento analítico. **Delta Lake** añade un registro transaccional y capacidades como versionado y consistencia en tablas. En Fabric, las tablas del Lakehouse emplean comúnmente Delta. Escribir Parquet en \`Files/\` no tiene el mismo resultado que registrar una tabla Delta en \`Tables/\`.

~~~python
ventas_validas.write.mode("overwrite").parquet("Files/curated/ventas_parquet")
ventas_validas.write.format("delta").mode("overwrite").saveAsTable("ventas_validas")
~~~

La primera instrucción produce ficheros Parquet; la segunda publica una tabla. \`overwrite\` debe utilizarse solo cuando se desea reemplazar el destino, y no significa que el proceso completo sea una transacción entre ambos destinos. Una carga incremental necesita estrategias específicas para conservar lo ya cargado. Evitar particionar excesivamente por campos de alta cardinalidad (por ejemplo, ID único de evento) porque puede producir demasiados ficheros pequeños.

### 4.5. Particionamiento

El particionado agrupa físicamente ficheros según una o varias columnas. Puede disminuir el volumen leído cuando las consultas filtran por dichas columnas y el motor evita particiones innecesarias. No es gratis: demasiadas particiones aumentan metadatos, número de ficheros y sobrecarga administrativa. Debe analizarse el tamaño total, cardinalidad, frecuencia de escritura y patrón de consultas.

~~~python
ventas_validas.write.mode("overwrite") \
    .partitionBy("anio") \
    .parquet("Files/curated/ventas_anuales")
~~~

**Fuente:** [DataFrames](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/use-apache-spark-work-files-lakehouse/includes/4-dataframe.md).

## 5. Use Spark SQL

### 5.1. Catálogo, vistas y tablas

Los DataFrames se pueden exponer como vistas temporales de sesión. Una **temporary view** no es un objeto permanente que sobreviva necesariamente al cierre de la sesión; una tabla persistente sí tiene metadatos y almacenamiento asociados. Esta diferencia es importante en orquestación: si un Pipeline abre una nueva sesión, puede que no exista una vista creada manualmente en otra sesión.

~~~python
ventas_validas.createOrReplaceTempView("vw_ventas_validas")

resumen = spark.sql("""
    SELECT anio, COUNT(*) AS pedidos, SUM(importe) AS ventas
    FROM vw_ventas_validas
    GROUP BY anio
    ORDER BY anio
""")
display(resumen)
~~~

En un Notebook también puede utilizarse:

~~~sql
%%sql
SELECT anio, COUNT(*) AS pedidos, SUM(importe) AS ventas
FROM vw_ventas_validas
GROUP BY anio
ORDER BY anio
~~~

Ambas opciones aprovechan Spark SQL. Un sistema distinto como Fabric Warehouse utiliza T-SQL en su propio endpoint, por lo que no debe asumirse identidad de funciones, sintaxis o compatibilidad con comandos DDL.

### 5.2. Tablas gestionadas y externas

Una **managed table** almacena datos en la ubicación gestionada por el catálogo y su ciclo de vida está relacionado con el objeto de tabla. Una **external table** utiliza una ubicación de datos externa al almacenamiento administrado del propio objeto; borrar su definición no tiene por qué borrar los ficheros subyacentes. Para operaciones destructivas deben comprobarse siempre la clase de tabla, el destino y los permisos efectivos.

### 5.3. Ejecución lógica y optimización

Las expresiones DataFrame y SQL pueden representar operaciones equivalentes, por ejemplo un \`groupBy\` y \`GROUP BY\`. El motor construye y optimiza planes para ejecutarlas. Un ingeniero debe revisar tipos, filtros, joins y particiones, no asumir que SQL siempre es más rápido que PySpark ni viceversa. Resulta más útil comparar planes físicos, volumen de datos, skew, shuffle, paralelismo y número de ficheros.

**Fuente:** [Spark SQL](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/use-apache-spark-work-files-lakehouse/includes/5-spark-sql.md).

## 6. Visualize data

Los Notebooks permiten visualizar tablas y gráficos de forma interactiva. Son adecuados para explorar tendencias o detectar valores atípicos durante desarrollo, pero no sustituyen necesariamente un modelo semántico gobernado ni un informe operativo. Para gráficos más personalizados se pueden utilizar bibliotecas de Python como Matplotlib. La operación \`toPandas()\` convierte el resultado distribuido en un DataFrame local: **solo se debe aplicar a resultados de tamaño controlado**.

~~~python
from matplotlib import pyplot as plt
df_local = por_anio.orderBy("anio").limit(30).toPandas()
plt.bar(df_local["anio"].astype(str), df_local["ventas"].astype(float))
plt.title("Ventas agregadas por año")
plt.xlabel("Año")
plt.ylabel("Ventas")
plt.xticks(rotation=45)
plt.show()
~~~

Esta transformación limita a 30 filas agregadas, no a 30 transacciones. Así se evita la práctica peligrosa de recopilar millones de filas para después agruparlas en Pandas. **Fuente:** [Visualize data](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/use-apache-spark-work-files-lakehouse/includes/6-visualize-data.md).

## 7. Exercise: Analyze data with Apache Spark

**Laboratorio oficial:** [02-analyze-spark.md](https://github.com/MicrosoftLearning/mslearn-fabric/blob/main/Instructions/Labs/02-analyze-spark.md) · [instrucciones navegables](https://microsoftlearning.github.io/mslearn-fabric/Instructions/Labs/02-analyze-spark.html). **Duración oficial aproximada:** 45 minutos. **Estado:** procedimiento consultado y explicado; no ejecutado.

### 7.1. Objetivo y preparación

El ejercicio trabaja con datos históricos de pedidos (2019, 2020 y 2021), que se importan como ficheros de un Lakehouse para exploración, transformación, escritura Parquet, particionado y consulta de tablas con Spark SQL. Se requiere una capacidad Fabric adecuada, permisos de creación en Workspace, acceso al navegador y un Notebook con Lakehouse asociado.

**Datos:** [orders.zip del repositorio MicrosoftLearning/dp-data](https://github.com/MicrosoftLearning/dp-data/raw/main/orders.zip). Antes de iniciar la práctica comprobar que se puede descargar el ZIP y que contiene la carpeta \`orders\` con los tres CSV indicados. No se han verificado aquí sus recuentos concretos.

### 7.2. Crear un Lakehouse y cargar los CSV

Abrir [Fabric](https://app.fabric.microsoft.com/), crear un Workspace en capacidad Trial, Premium o Fabric y añadir un Lakehouse con un nombre único. Descargar y descomprimir \`orders.zip\`, entrar en el Explorador del Lakehouse y utilizar **Files → Upload folder** para cargar \`orders/\`. Verificar visualmente que se muestran \`Files/orders/2019.csv\`, \`2020.csv\` y \`2021.csv\`.

Crear un Notebook desde el Lakehouse e identificar una celda de texto Markdown para describir el objetivo del trabajo. La separación entre explicaciones Markdown y código ejecutable mejora la trazabilidad del laboratorio.

### 7.3. Detectar un encabezado incorrecto

Una de las enseñanzas deliberadas del laboratorio es que el origen no contiene una cabecera convencional. Si se ejecuta un código generado automáticamente con \`header=true\`, Spark interpreta la primera fila como nombres de columnas; una posterior referencia a los nombres esperados puede provocar \`[UNRESOLVED_COLUMN.WITH_SUGGESTION]\`. La corrección oficial consiste en utilizar \`header=false\` y asignar nombres explícitos.

~~~python
df = spark.read.format("csv").option("header", False).load("Files/orders/2019.csv")
display(df.limit(10))
~~~

Después deben renombrarse las columnas \`_c0\`, \`_c1\`, etc., siguiendo la lista de columnas que establece el archivo original del laboratorio. No inventar el esquema completo del CSV ni aplicar casts a columnas sin comprobarlo; revisar su contenido antes de continuar.

### 7.4. Trabajar con los datos y crear Parquet

El laboratorio combina los datos de pedidos y obtiene información de fecha y cliente mediante funciones PySpark. Con el DataFrame transformado, genera una salida Parquet en \`Files/transformed_data/orders\` y vuelve a leerla para demostrar que el resultado puede persistirse y reutilizarse.

~~~python
# Suponiendo que transformed_df se ha construido siguiendo la práctica oficial:
transformed_df.write.mode("overwrite").parquet("Files/transformed_data/orders")
orders_df = spark.read.parquet("Files/transformed_data/orders")
display(orders_df.limit(5))
~~~

**Verificación:** actualizar Files y confirmar que aparece la carpeta Parquet con los archivos de salida. Revisar que el DataFrame reabierto mantiene las columnas esperadas. No se atribuyen aquí cifras de filas ni un resultado exitoso sin ejecutar Spark.

### 7.5. Particionar y consultar

La práctica también guarda ficheros Parquet particionados por **Year** y **Month**, y después lee una ruta con comodín para revisar los datos de 2021.

~~~python
orders_df.write.partitionBy("Year", "Month").mode("overwrite") \
    .parquet("Files/partitioned_data")

orders_2021_df = spark.read.parquet("Files/partitioned_data/Year=2021/Month=*")
display(orders_2021_df.limit(10))
~~~

Hay que distinguir entre seleccionar la ruta de una partición concreta y leer desde la raíz de la tabla particionada: en función del nivel de lectura, algunas columnas de partición pueden no estar disponibles en el resultado. Comprobar el esquema efectivo con \`printSchema()\`.

### 7.6. Crear tabla Delta y utilizar Spark SQL

El laboratorio persiste información en una tabla \`salesorders\` y ejecuta consultas desde Spark. Con Lakehouse schemas, comprobar si el contexto usa el esquema predeterminado correcto y si el nombre debe cualificarse. Una sentencia de ejemplo es:

~~~python
df.write.format("delta").mode("overwrite").saveAsTable("salesorders")
spark.sql("DESCRIBE EXTENDED salesorders").show(truncate=False)
~~~

El ejemplo anterior muestra \`overwrite\` para permitir repetir la comprobación; el material oficial puede utilizar el modo de escritura predeterminado para la creación inicial. **No cambiar silenciosamente el modo durante el laboratorio**: seguir el original para la entrega y tratar esta variante como práctica adicional.

### 7.7. Verificaciones y errores frecuentes

| Comprobación / fallo | Interpretación | Acción |
|---|---|---|
| 2019.csv se lee con cabecera errónea | Primer registro usado como nombres | Desactivar \`header\` y renombrar columnas |
| No encuentra \`Files/orders\` | Ruta o Lakehouse asociado incorrectos | Revisar Files y conexión del Notebook |
| Sale una columna con tipo String | Inferencia o CSV sin esquema | Convertir explícitamente y validar datos |
| Error en \`Year\` o \`Month\` | Columnas no derivadas o mal escritas | Revisar \`withColumn\` y \`printSchema()\` |
| No aparece \`salesorders\` | Tabla no registrada, Notebook sin Lakehouse | Inspeccionar contexto, estado y errores |
| La sesión Spark tarda en arrancar | Inicio del pool | Revisar estado de sesión y capacidad |
| \`toPandas()\` consume memoria | Recopilación al driver | Agregar y limitar antes de convertir |
| Overwrite reemplaza datos previos | Es el comportamiento solicitado | Utilizar rutas aisladas y no sobrescribir producción |

### 7.8. Limpieza y costes

El procedimiento oficial termina eliminando el workspace de laboratorio. Si se ha usado un área creada exclusivamente para esta práctica, ir a **Workspace settings → General → Remove this workspace** y confirmar. Si el workspace contiene otros objetos o pertenece a un equipo, no borrarlo entero: eliminar únicamente los elementos del ejercicio con autorización. Dejar de ejecutar Notebooks no equivale necesariamente a cancelar programaciones, y borrar el workspace no sustituye la administración de una capacidad pagada.

## 8. Knowledge check razonado

Estas cuestiones son **propias**, no el test oficial.

1. **¿Qué coordina un Spark driver?** Planificación y gestión de la ejecución distribuida.
2. **¿Qué hacen los executors?** Ejecutan tareas y procesan particiones de datos.
3. **¿En qué se diferencia autoscale de dynamic allocation?** Aprovisionamiento de nodos frente a asignación dinámica de executors.
4. **¿Por qué es recomendable un Environment?** Para controlar runtime y librerías y favorecer reproducibilidad.
5. **¿Cuándo elegir un Notebook?** Para desarrollo, experimentación o ejecución de código integrable en procesos automatizados.
6. **¿Por qué no conviene inferir todos los esquemas CSV?** Puede ser costoso, ambiguo e inestable ante cambios.
7. **¿Qué hace \`groupBy\`?** Agrupa registros para aplicar agregaciones; puede implicar shuffle.
8. **¿Qué añade Delta a Parquet?** Log transaccional y funciones de gestión de tablas, entre otras capacidades.
9. **¿Por qué puede ser problemático un particionado demasiado granular?** Aumenta el número de ficheros y la sobrecarga de metadatos.
10. **¿Una vista temporal persiste entre sesiones?** No se debe asumir.
11. **¿Por qué limitar \`toPandas()\`?** Porque transfiere resultados a memoria local.
12. **¿Spark SQL es idéntico a T-SQL de Warehouse?** No, tienen dialectos y contextos de ejecución diferentes.

## 9. Summary

Apache Spark en Microsoft Fabric permite combinar procesamiento distribuido y almacenamiento OneLake con DataFrames, SQL y Notebooks. Para el examen es imprescindible comprender **qué se ejecuta en el driver, qué se distribuye, cómo se representan los esquemas, cuándo se materializa una acción y cómo se publican ficheros frente a tablas**. En producción debe añadirse gestión de entornos, control de datos, protección de recursos, estrategias de rerun y monitorización.

## 10. Fuentes y control de calidad

- [Índice del módulo de nueve unidades](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/use-apache-spark-work-files-lakehouse/index.yml).
- [Spark pools y runtime](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/use-apache-spark-work-files-lakehouse/includes/2-spark.md).
- [Notebooks y Spark jobs](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/use-apache-spark-work-files-lakehouse/includes/3-spark-code.md).
- [DataFrames](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/use-apache-spark-work-files-lakehouse/includes/4-dataframe.md).
- [Spark SQL](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/use-apache-spark-work-files-lakehouse/includes/5-spark-sql.md).
- [Visualizaciones](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/use-apache-spark-work-files-lakehouse/includes/6-visualize-data.md).
- [Laboratorio original](https://github.com/MicrosoftLearning/mslearn-fabric/blob/main/Instructions/Labs/02-analyze-spark.md).

**Estado del control:** los conceptos de las nueve unidades, la práctica y la evaluación se desarrollan en este README. Se consultaron las fuentes principales del contenido y el laboratorio. El laboratorio no se ha ejecutado; los resultados y los casos de error se presentan como objetivos y diagnósticos, no como observaciones.