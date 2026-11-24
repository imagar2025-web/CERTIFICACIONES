# 06 · Get started with lakehouses in Microsoft Fabric

> **DP-700 · Implement a Lakehouse with Microsoft Fabric.** [Módulo Microsoft Learn](https://learn.microsoft.com/en-us/training/modules/get-started-lakehouses/) · [Fuentes \`index.yml\` e \`includes\`](https://github.com/MicrosoftDocs/learn/tree/main/learn-pr/wwl/get-started-lakehouses) · [Laboratorio oficial](https://github.com/MicrosoftLearning/mslearn-fabric/blob/main/Instructions/Labs/01-lakehouse.md). **Fecha de revisión:** noviembre de 2026. **Laboratorio:** documentado, no ejecutado.

## Índice

1. [Introducción y fundamentos](#1-introducción-y-fundamentos)
2. [Understand the Fabric lakehouse](#2-understand-the-fabric-lakehouse)
3. [Work with a lakehouse](#3-work-with-a-lakehouse)
4. [Explore lakehouse data](#4-explore-lakehouse-data)
5. [Laboratorio oficial](#5-laboratorio-oficial)
6. [Knowledge check](#6-knowledge-check)
7. [Summary, fuentes y control](#7-summary-fuentes-y-control)

## 1. Introducción y fundamentos

Un **data lake** permite conservar archivos heterogéneos a escala y suele aceptar esquemas flexibles. Un **data warehouse** organiza información analítica con modelos tabulares y mecanismos de consulta optimizados para usuarios de SQL y BI. El concepto **Lakehouse** combina características de ambos: almacenamiento abierto de archivos y tablas con un catálogo accesible desde motores de análisis. No significa que todas las operaciones SQL de una base de datos relacional estén necesariamente disponibles.

En Fabric, el Lakehouse se apoya en **OneLake** y organiza información mediante dos áreas diferenciadas: **Files** y **Tables**. Files permite alojar ficheros de entrada e intermedios en diferentes formatos. Tables representa tablas registradas en el catálogo del Lakehouse, generalmente con formato **Delta Lake**. Esta diferencia condiciona las herramientas de consulta, el tratamiento de esquemas y las garantías de operación.

La propuesta del módulo es conectar tres ciclos: crear el elemento de almacenamiento, ingerir y transformar datos, y consumirlos con Spark o el SQL analytics endpoint. Si se diseñan por separado, el ingeniero puede conservar material bruto y publicar tablas depuradas en vez de sobrescribir los CSV originales.

## 2. Understand the Fabric lakehouse

### 2.1. Diseño y arquitectura

El almacenamiento del Lakehouse vive sobre OneLake. Un elemento Lakehouse incluye metadatos y una estructura de datos para organizar tablas y archivos. La parte **Files** se utiliza para entradas, ficheros binarios y salidas que todavía no están representadas como tablas. La parte **Tables** expone tablas Delta y su esquema a los motores compatibles.

~~~mermaid
flowchart TB
  S["CSV, JSON, Parquet, APIs"] --> I["Pipeline / Dataflow / Spark"]
  I --> F["OneLake · Lakehouse / Files"]
  F --> P["Notebook PySpark · limpieza"]
  P --> T["Lakehouse / Tables · Delta"]
  T --> Q["Spark SQL"]
  T --> E["SQL analytics endpoint"]
  E --> B["Power BI / consumidores SQL"]
~~~

La escritura de un fichero Parquet en Files no crea automáticamente una tabla Delta. Por el contrario, \`saveAsTable\` o la creación de una tabla compatible registra el objeto y sus datos. Es esencial entender la independencia parcial entre almacenamiento físico y catálogo lógico.

### 2.2. Delta Lake y consistencia

Delta Lake se basa en archivos de datos y un **transaction log**, que registra versiones y acciones que permiten la gestión de tablas. Las operaciones sobre una tabla Delta pueden proporcionar atomicidad dentro del alcance transaccional del formato. Esto no convierte todo el Pipeline que la alimenta en una transacción global; dos tablas actualizadas consecutivamente pueden encontrarse temporalmente en versiones distintas.

En lugar de representar toda la información con CSV, una tabla Delta permite procesos de actualización, borrado y versionado apropiados para escenarios analíticos. Los consumidores deben seguir conociendo la granularidad del dato. Una tabla de pedidos que contiene una fila por línea de pedido no equivale a otra que contiene una fila por pedido, aunque ambas se llamen «ventas».

### 2.3. Seguridad y control de acceso

Fabric administra acceso a través de Workspaces, permisos de elementos y mecanismos de seguridad adicionales según el objeto y las capacidades disponibles. No basta con ocultar una columna de un informe para proteger información personal en el Lakehouse de origen. Si un usuario tiene permisos directos sobre el almacenamiento, puede tener una vía de consulta distinta de la del modelo semántico. Los equipos deben diseñar políticas de lectura de datos y separación de capas, idealmente aplicando mínimo privilegio.

**Fuente:** [Fabric lakehouse](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/get-started-lakehouses/includes/2-fabric-lakehouse.md).

## 3. Work with a lakehouse

### 3.1. Crear y explorar

En un Workspace con capacidad y permisos adecuados, seleccionar **New item → Lakehouse**, elegir nombre único y revisar las opciones de schemas. Abrir Lakehouse Explorer y distinguir Files de Tables. La creación del Lakehouse no importa fuentes externas por sí misma; hay que configurar una de las experiencias de ingesta.

### 3.2. Ingesta de datos

Hay varias rutas: subir archivos locales, copiar desde sistemas externos mediante Pipeline o Copy Job, transformar con Dataflow Gen2 o utilizar Notebooks Spark. La elección depende de la frecuencia de llegada y de la necesidad de tratamiento. Una carga desde CSV sin transformación puede situarse primero en Files; una transformación posterior lee esa ruta y genera una tabla Delta.

Un ejemplo ilustrativo de lectura desde Files es:

~~~python
from pyspark.sql.types import StructType, StructField, StringType, IntegerType

schema = StructType([
    StructField("producto", StringType(), True),
    StructField("unidades", IntegerType(), True)
])

df = (spark.read.format("csv")
    .option("header", True)
    .schema(schema)
    .load("Files/entradas/inventario.csv"))

display(df.limit(20))
~~~

*Ejemplo propio*. La ruta exige que antes se haya cargado un CSV con esas columnas. En una carga real también hay que comprobar delimitador, codificación, nulos, fechas y registros corruptos.

### 3.3. Shortcuts

Un **shortcut** permite exponer datos almacenados en otras ubicaciones compatibles, evitándose en algunos casos el paso de copiar. Esta capacidad es útil para reducir duplicación de datos y conectar dominios, pero debe valorarse su dependencia de la disponibilidad y la identidad de acceso del origen. El shortcut no corrige errores del dataset ni convierte una fuente de lectura en una tabla transaccional por arte de magia.

### 3.4. Transformaciones

Spark puede crear un DataFrame, convertir columnas y guardarlo como tabla Delta:

~~~python
from pyspark.sql import functions as F

depurado = (df
    .filter(F.col("producto").isNotNull())
    .filter(F.col("unidades") >= 0)
    .dropDuplicates(["producto"]))

depurado.write.format("delta").mode("overwrite").saveAsTable("inventario_curado")
~~~

En este ejemplo se presupone una única fila final por producto. Esa elección de clave no sería correcta si el negocio mantiene inventario por almacén o fecha. Para datasets incrementales debe estudiarse MERGE y control de lotes: overwrite elimina el contenido anterior según el alcance de escritura, y append puede duplicar registros.

**Fuente:** [Work with a lakehouse](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/get-started-lakehouses/includes/3-work-lakehouse.md).

## 4. Explore lakehouse data

### 4.1. SQL analytics endpoint

El **SQL analytics endpoint** ofrece una experiencia T-SQL de consulta sobre las tablas compatibles del Lakehouse, especialmente valiosa para herramientas y consumidores que no necesitan abrir Spark. No debe confundirse con un Fabric Warehouse ni con un endpoint de escritura SQL general sobre las mismas tablas. El motor y las funciones permitidas dependen del servicio.

~~~sql
SELECT producto, SUM(unidades) AS unidades_totales
FROM dbo.inventario_curado
GROUP BY producto
ORDER BY unidades_totales DESC;
~~~

Este ejemplo presupone un Lakehouse con schema \`dbo\` y tabla existente. Si el Lakehouse usa otra organización de schemas, adaptar la referencia. Verificar los permisos del endpoint además de los del elemento.

### 4.2. Lakehouse Query Explorer

El explorador permite consultar y trabajar con los datos de las tablas desde la interfaz. Es útil para una validación rápida después de una carga, pero para verificar calidad hay que examinar recuentos, valores nulos, claves y registros problemáticos. Una vista previa correcta no demuestra que la ingestión haya procesado todos los ficheros ni que la transformación sea idempotente.

### 4.3. Spark notebooks

Con Spark puede leerse la tabla del catálogo, aplicar transformaciones y guardar nuevos resultados. Una diferencia esencial es que Spark SQL y T-SQL del endpoint no son idénticos: tienen motores, funciones y capacidades de escritura diferentes. Elegir Spark cuando interesa programación distribuida, transformaciones complejas o operaciones de ingeniería sobre Delta; utilizar el endpoint SQL cuando la necesidad es consulta compatible y consumo.

~~~python
productos = spark.table("inventario_curado")
productos.groupBy("producto").sum("unidades").show()
~~~

### 4.4. Power BI y modelos semánticos

La capa de consumo puede utilizar Power BI para construir un modelo semántico que define relaciones, medidas y seguridad de presentación. Un Lakehouse no crea automáticamente el modelo de negocio completo; hay que diseñar dimensiones, hechos y cálculos. La semántica correcta de un indicador depende de la granularidad y calidad de las tablas publicadas.

**Fuente:** [Explore data in a lakehouse](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/get-started-lakehouses/includes/4-explore-data-lakehouse.md).

## 5. Laboratorio oficial

**Fuente de la práctica:** [01-lakehouse.md de MicrosoftLearning](https://github.com/MicrosoftLearning/mslearn-fabric/blob/main/Instructions/Labs/01-lakehouse.md) · [versión navegable](https://microsoftlearning.github.io/mslearn-fabric/Instructions/Labs/01-lakehouse.html). **Estado:** documento y objetivos contrastados, ejecución no realizada. Se distingue el recorrido oficial de las recomendaciones pedagógicas adicionales.

### 5.1. Objetivo

Aprender a crear un Lakehouse, subir archivos, convertir datos de ficheros en tablas y consultarlas. Se utiliza un Workspace de prueba con una capacidad Fabric adecuada. Las fuentes y nombres de archivo concretos deben tomarse del ejercicio enlazado; no se inventan datasets ni se atribuyen resultados a una ejecución no efectuada.

### 5.2. Preparación

Iniciar sesión en [Fabric](https://app.fabric.microsoft.com/), crear un Workspace aislado asignando Trial, Premium o Fabric cuando proceda, y crear un Lakehouse. Anotar nombre y permisos, y revisar que se muestran las secciones Files y Tables.

**Verificación:** Workspace y Lakehouse aparecen en el catálogo de elementos. El Lakehouse no debe confundirse con un Warehouse ni con una tabla cargada.

### 5.3. Ingesta inicial

Seguir el origen de datos indicado por el laboratorio oficial, cargar el archivo en la zona **Files** y comprobar su presencia. La carga de un CSV puede dejar únicamente un archivo; utilizar la acción de carga a tabla o el flujo del laboratorio para materializarlo en **Tables** con el esquema deseado.

Antes de guardar, inspeccionar los tipos reconocidos. Un identificador, un código postal o una clave de cliente pueden necesitar String aunque su representación contenga dígitos. En fechas, revisar formato regional. El número de filas que devuelve la muestra debe anotarse solo después de ejecutar el procedimiento.

### 5.4. Consulta SQL y Spark

Una vez creada la tabla, abrir la experiencia SQL y consultar sus filas. En paralelo, desde Notebook asociado al Lakehouse, ejecutar:

~~~python
# Reemplazar nombre_tabla por el nombre REAL obtenido en la práctica
df = spark.table("nombre_tabla")
df.printSchema()
display(df.limit(10))
~~~

Este ejemplo es **complementario** y debe adaptarse. La tabla puede tener schema \`dbo\` u otro. No copiar el nombre literal \`nombre_tabla\` como si existiera en el servicio.

### 5.5. Diagnóstico y verificación

| Fallo | Explicación | Corrección |
|---|---|---|
| No aparece Lakehouse | Restricción de capacidad o permiso | Revisar Workspace y tenant |
| Archivo no aparece en Files | Carga fallida o ruta equivocada | Revisar Explorer y actualizar |
| Tabla no aparece en Tables | El archivo no se materializó como tabla | Ejecutar el paso oficial de carga a tabla |
| Columnas mal tipadas | Inferencia de esquema incorrecta | Configurar tipos explícitos |
| Spark no localiza tabla | Lakehouse no asociado o nombre/schema equivocado | Revisar conexión y catálogo |
| Endpoint SQL no encuentra objeto | Sincronización o referencia incorrecta | Refrescar y comprobar esquema/tabla |
| Filas duplicadas | Reintento con append sobre el mismo origen | Diseñar una carga idempotente |

**Verificaciones mínimas:** existe el archivo de origen; la tabla destino aparece en Tables; el esquema corresponde al esperado; las consultas SQL y Spark devuelven filas coherentes. No proporcionar un número de registros no observado.

### 5.6. Limpieza

Al finalizar, borrar solo los elementos creados o el Workspace de práctica si es exclusivo. La eliminación puede ser irreversible. Revisar que no queden tareas programadas que vuelvan a crear datos y que la capacidad pagada se administre independientemente.

## 6. Knowledge check

Preguntas originales con su respuesta conceptual:

1. **¿Qué diferencia existe entre Files y Tables?** Files almacena archivos; Tables expone tablas gestionadas en el catálogo.
2. **¿Qué función desempeña OneLake?** Almacenamiento unificado para Fabric.
3. **¿Un CSV en Files equivale a una tabla Delta?** No.
4. **¿Qué añade el log de Delta?** Gestión transaccional y versionado.
5. **¿Para qué sirve un shortcut?** Referenciar determinados datos externos compatibles sin duplicarlos.
6. **¿El SQL analytics endpoint se utiliza igual que un Warehouse?** No; hay diferencias de capacidades y escritura.
7. **¿Qué aporta Spark?** Procesamiento programable distribuido.
8. **¿El modelo semántico existe completo tras crear Lakehouse?** No, debe diseñarse.
9. **¿Qué riesgo tiene overwrite?** Reemplazar información no destinada a eliminarse.
10. **¿Qué riesgo tiene append?** Duplicación al reintentar el mismo lote.

## 7. Summary, fuentes y control

Un Lakehouse aporta almacenamiento flexible y tablas Delta bajo OneLake y puede consultarse desde Spark o el endpoint SQL compatible. Elegir bien la zona de entrada, el contrato de esquema, las claves y el mecanismo de carga es fundamental. La seguridad se diseña en el almacenamiento, catálogo y consumo; no se delega exclusivamente al informe.

**Fuente del catálogo de unidades:** [index.yml](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/get-started-lakehouses/index.yml) — siete unidades: Introduction, Fabric lakehouse, Work with a lakehouse, Explore data, Exercise, Knowledge check y Summary. Las cuatro unidades centrales se contrastaron con los \`includes\`; el laboratorio se enlaza al repositorio oficial y se expone de forma razonada. **No ejecutado.**