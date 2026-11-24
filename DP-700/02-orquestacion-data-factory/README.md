# 02 · Orchestrate processes and data movement with Microsoft Fabric

> **Certificación:** DP-700 · Data Engineering Solutions Using Microsoft Fabric. **Fuente:** [módulo oficial](https://learn.microsoft.com/en-us/training/modules/use-data-factory-pipelines-fabric/) y [archivos originales del módulo](https://github.com/MicrosoftDocs/learn/tree/main/learn-pr/wwl/use-data-factory-pipelines-fabric). **Revisión:** noviembre de 2026. **Laboratorio:** documentado, sin ejecución real.

## Índice

1. [Introducción: orquestación frente a transformación](#1-introducción-orquestación-frente-a-transformación)
2. [Understand Fabric pipelines](#2-understand-fabric-pipelines)
3. [Copy Data y movimiento de datos](#3-copy-data-y-movimiento-de-datos)
4. [Plantillas y reutilización](#4-plantillas-y-reutilización)
5. [Ejecución, monitorización y recuperación](#5-ejecución-monitorización-y-recuperación)
6. [Laboratorio oficial desarrollado](#6-laboratorio-oficial-desarrollado)
7. [Knowledge check y síntesis](#7-knowledge-check-y-síntesis)
8. [Fuentes y alcance](#8-fuentes-y-alcance)

## 1. Introducción: orquestación frente a transformación

La orquestación de datos responde a una pregunta distinta de la transformación: no «¿qué valores resultan de aplicar esta regla?» sino «¿qué procesos deben ejecutarse, en qué orden, bajo qué condiciones y con qué garantías de recuperación?». Una organización podría ingerir ficheros CSV, normalizarlos mediante PySpark, cargar datos en Delta Lake y ejecutar controles de calidad. Esas cuatro tareas tienen dependencias y puntos de fallo. La solución necesita observabilidad y repetibilidad, no solo cuatro scripts independientes.

Microsoft Fabric incorpora capacidades de Data Factory para definir **Data Pipelines** visuales. Un Pipeline es una colección de actividades y relaciones de control. Una actividad puede mover datos, procesarlos o decidir qué actividad ejecutar después. Ejecutar una actividad no significa que el Pipeline encapsule una transacción atómica: si una carga escribe filas y una etapa posterior falla, las filas pueden permanecer en la tabla. El diseño del reintento debe considerar los efectos persistidos.

El módulo oficial persigue cuatro objetivos prácticos: describir las capacidades de los pipelines, configurar Copy Data, crear pipelines desde plantillas y ejecutarlos y supervisarlos. Además introduce el laboratorio que une copia HTTP, OneLake y un Notebook de transformación. Los ejemplos de arquitectura, idempotencia y diagnóstico de este documento amplían lo enseñado en Microsoft Learn.

## 2. Understand Fabric pipelines

### 2.1. Actividades y dependencias

Una **activity** es la unidad operativa que se configura y monitoriza. Puede representar Copy Data, Dataflow Gen2, Notebook, ejecución SQL o una estructura de control. Un Pipeline puede combinar actividades que mueven datos, actividades que los transforman y actividades de control (condiciones, bucles, variables y parámetros). Cada actividad tiene propiedades propias y produce información de ejecución útil para diagnóstico. Las conexiones entre actividades expresan dependencias de éxito, fallo, finalización o salto, no simplemente un orden decorativo en el canvas.

~~~mermaid
flowchart LR
    A["Comprobar origen"] --> B{"¿Archivo disponible?"}
    B -->|Sí| C["Copy Data"]
    B -->|No| X["Registrar ausencia"]
    C -->|On success| D["Notebook: validar y transformar"]
    C -->|On failure| E["Registrar error"]
    D -->|On success| F["Publicar tabla Delta"]
    D -->|On failure| E
~~~

**Ejemplo propio.** El estado global debe interpretarse atendiendo a las ramas y actividades efectivamente ejecutadas. Es importante distinguir «continuar cuando termine» (On completion) de «continuar cuando haya terminado correctamente» (On success). En ingestas críticas, desencadenar la transformación después de una copia fallida puede exponer datos antiguos o parciales.

### 2.2. Parámetros frente a variables

Un **parameter** define una entrada para una ejecución del Pipeline: permite suministrar nombre de fichero, carpeta de llegada, fecha del lote o identificador del entorno sin tener que duplicar el Pipeline. El parámetro se configura por ejecución y debe tratarse como una entrada del proceso. Una **variable** almacena un valor que las actividades pueden actualizar durante esa ejecución. No conviene trasladar secretos como texto en parámetros corrientes: las credenciales pertenecen a las conexiones y a los mecanismos de identidad y seguridad del servicio.

Imaginemos un Pipeline «Ingest Sales Data» con un parámetro \`file_name\` y otro \`load_date\`. Cada día el programador invoca la misma definición con valores diferentes. La transformación puede incluir \`load_date\` en el destino para rastrear la procedencia de las filas. Esta trazabilidad facilita separar un error de contenido del fichero de un problema en el servicio. El valor recibido debe validarse: no asumir que toda cadena pasada como ruta representa una ubicación confiable o permitida.

### 2.3. Runs y observabilidad

Cada ejecución genera un **pipeline run** identificable. El run tiene estado, hora, duración y registros de actividad. Cuando una actividad falla, se debe conservar el nombre de la actividad, identificador de run, información de entrada y salida apropiada, y mensaje de error. Para datos sensibles, limitar lo que aparece en logs: las entradas y salidas pueden contener nombres de recursos o valores que no deben exponerse indiscriminadamente.

La línea temporal permite localizar cuellos de botella. Por ejemplo, una ejecución de 25 minutos puede dedicar 2 minutos a Copy Data, 21 a arrancar y ejecutar Spark y 2 a validaciones. Optimizar la lectura CSV cuando el tiempo dominante es Spark sería una actuación poco eficaz. Antes de modificar el rendimiento, identificar la fase lenta y reproducirla con un conjunto comparable de datos.

### 2.4. Idempotencia, reintentos y calidad

La **idempotencia** es una propiedad del proceso por la cual repetirlo con la misma entrada deja el destino en el mismo estado lógico que ejecutarlo una sola vez. Es una meta de diseño, no una función automática de todo Pipeline. Un Notebook que hace \`append\` sobre una tabla Delta puede duplicar los registros si se ejecuta dos veces con el mismo archivo. Para cargas productivas conviene elegir entre overwrite controlado, MERGE con clave estable, registro de lotes procesados o una estrategia equivalente.

Un Pipeline técnicamente correcto tampoco demuestra la calidad del dato. La validación puede comprobar cantidad de registros, unicidad de claves, nulos en campos obligatorios, rangos numéricos, esquema y fecha de actualización. Cuando estas condiciones fallan, debe evitarse la publicación en Gold o marcar claramente la información como no certificada.

**Fuente directa de esta unidad:** [Understand Fabric pipelines](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/use-data-factory-pipelines-fabric/includes/2-understand-fabric-pipeline.md).

## 3. Copy Data y movimiento de datos

### 3.1. Copy job y Copy Data Activity

El origen puede ser una base de datos, almacenamiento de objetos o un endpoint HTTP. La actividad **Copy Data** realiza movimiento desde una fuente compatible hasta un destino compatible y se integra con otros pasos de un Pipeline. Se diferencia de un **Copy job**, diseñado como una experiencia simplificada y autónoma de movimiento. Un **Dataflow Gen2** es apropiado cuando la necesidad principal consiste en aplicar transformaciones Power Query M. Si se necesitan cálculos complejos distribuidos, un Notebook Spark puede asumir una etapa posterior.

| Capacidad | Copy job | Copy Data activity | Dataflow Gen2 | Notebook |
|---|---|---|---|---|
| Objetivo dominante | Copiar datos | Copiar dentro de un proceso orquestado | Transformar con Power Query | Transformar con código |
| Herramienta de trabajo | Experiencia de copia | Pipeline de Data Factory | Editor Power Query Online | PySpark / Spark SQL |
| Dependencias entre actividades | No es su foco | Sí, por la definición del Pipeline | Mediante Pipeline externo | Mediante Pipeline externo |
| Reglas de transformación | Según opciones de copia | Limitadas frente a una transformación completa | Transformaciones low-code | Transformaciones programables |
| Comprobaciones necesarias | Esquema, lotes, volumen | Esquema, destino, volumen | Tipos, lógica, destino | Lógica, rendimiento, calidad |

### 3.2. Configurar fuente y destino correctamente

La configuración de una actividad Copy Data se puede explicar como dos contratos. El contrato de **source** define dónde están los datos, cómo autenticarse y cómo interpretarlos: por ejemplo, HTTP con formato DelimitedText, separador coma, primera fila como encabezado y codificación adecuada. El contrato de **destination** especifica el recurso de Fabric y la ruta o tabla que recibirá los datos. Para un Lakehouse se debe distinguir entre \`Files/\`, donde puede conservarse el fichero bruto, y \`Tables/\`, donde se publican tablas del Lakehouse. Una copia hacia un CSV en \`Files/new_data/\` no crea automáticamente una tabla Delta consultable como relación.

En ficheros delimitados, indicar \`First row as header\` incorrectamente provoca que se pierda un registro real o que los nombres de columna sean interpretados como datos. También puede ocurrir un desajuste de \`Column delimiter\` (coma frente a punto y coma), codificación, comillas o escape. No basta con que Copy Data termine: debe comprobarse el archivo resultante y su interpretación en la siguiente fase.

### 3.3. Estrategia Bronze → Silver → Gold

Un escenario reproducible consiste en copiar los datos originales a \`Files/bronze/\`, utilizar Spark para limpiar y normalizar los tipos en tablas Silver y publicar agregaciones Gold. La capa Bronze permite auditar el dato de entrada y, según la política de retención, reprocesar sin acceder otra vez a un origen externo. La capa Silver concentra calidad y semántica común; Gold ofrece conjuntos listos para métricas de negocio.

~~~mermaid
flowchart TB
  S["HTTP CSV / BD externa"] --> C["Copy Data"]
  C --> B["OneLake · Bronze / Files"]
  B --> N["Notebook PySpark"]
  N --> V["Delta Lake · Silver"]
  V --> A["SQL / Spark · Gold"]
  A --> P["Modelo semántico y consumidores"]
~~~

El diagrama es una extensión propia del ejercicio. No presenta las tres capas como servicios separados: pueden ubicarse en Lakehouses y Workspaces distintos o compartir el mismo almacenamiento según la estrategia de seguridad, dominios y responsabilidades.

**Fuente directa:** [Copy Data](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/use-data-factory-pipelines-fabric/includes/3-copy-data.md).

## 4. Plantillas y reutilización

Microsoft Fabric ofrece plantillas predefinidas para acelerar la creación de determinados pipelines. Una plantilla incorpora un patrón de actividades y conexiones entre ellas que después debe adaptarse al entorno. También pueden exportarse configuraciones personalizadas como plantilla \`.zip\` e importarse en otro Pipeline. Esto ayuda a estandarizar la forma de ingerir datos, pero no exime de revisar que los conectores, credenciales, recursos y rutas apuntan al entorno correcto.

En sistemas con entornos dev/test/prod, la reutilización necesita un plan de despliegue. Copiar visualmente un Pipeline a producción y editar sus conexiones a mano introduce riesgos de configuración. Es preferible gobernar las definiciones y los parámetros de entorno con las capacidades de CI/CD apropiadas y mantener las credenciales fuera de archivos de configuración compartidos. Las plantillas no son equivalentes a un pipeline de CI/CD: una plantilla ayuda a recrear un diseño, mientras que CI/CD gestiona versiones, validaciones y despliegues.

**Fuente:** [Pipeline templates](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/use-data-factory-pipelines-fabric/includes/4-pipeline-templates.md).

## 5. Ejecución, monitorización y recuperación

El proceso operativo mínimo incluye validar, guardar, ejecutar y analizar. Una **ejecución on demand** se inicia manualmente; una **programada** se configura con frecuencia y zona horaria; un **event trigger** responde a un evento admitido por Fabric. No todos los triggers son adecuados para cualquier fuente: la elección debe corresponder con el mecanismo real de llegada y con el requisito de latencia.

En el **Monitoring hub** se puede revisar el historial y acceder a información de actividades, además de visualizaciones temporales como Gantt. El histórico ayuda a responder por qué una ejecución fue más lenta y si varias ejecuciones se solaparon. Antes de reejecutar desde una actividad fallida, comprobar qué estados previos ya fueron persistidos: reintentar solo el Notebook puede ser correcto o puede sumar por segunda vez datos del mismo lote.

Para registrar errores productivos, una tabla de auditoría propia puede almacenar \`run_id\`, \`batch_id\`, \`source_path\`, \`expected_rows\`, \`written_rows\`, \`status\` y \`completed_at\`. No es una tabla automática del Pipeline: es un diseño complementario que el equipo debe implementar y proteger.

**Fuente:** [Run and monitor pipelines](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/use-data-factory-pipelines-fabric/includes/5-run-monitor-pipelines.md).

## 6. Laboratorio oficial desarrollado

**Procedencia:** [Ingest data with a pipeline in Microsoft Fabric](https://github.com/MicrosoftLearning/mslearn-fabric/blob/main/Instructions/Labs/04-ingest-pipeline.md) · [instrucciones navegables](https://microsoftlearning.github.io/mslearn-fabric/Instructions/Labs/04-ingest-pipeline.html). **Duración orientativa oficial:** 45 minutos. **Estado:** instrucciones revisadas; no se ha ejecutado el ejercicio, por lo que las verificaciones descritas son esperadas.

### 6.1. Escenario y objetivos

El laboratorio combina un origen CSV accesible por HTTPS, una actividad Copy Data, un Lakehouse, un Notebook PySpark y un Pipeline. Se copia \`sales.csv\` a la carpeta \`Files/new_data/\`; el Notebook deriva columnas de fecha y nombres y escribe una tabla Delta; finalmente se añade Notebook al Pipeline para convertir pasos aislados en una secuencia reutilizable.

**Fuente de datos:** [sales.csv de MicrosoftLearning](https://raw.githubusercontent.com/MicrosoftLearning/dp-data/main/sales.csv). La conexión HTTP del laboratorio es anónima. El usuario necesita una cuenta Microsoft Fabric con capacidad Trial, Premium o Fabric adecuada y permisos para crear elementos en un workspace.

### 6.2. Preparar Workspace y Lakehouse

Entrar en [Fabric](https://app.fabric.microsoft.com/), abrir Workspaces y crear un área separada para el laboratorio. Seleccionar un modo con Fabric capacity; el nombre puede ser \`lab_dp700_pipelines\`. Después crear un Lakehouse con nombre único, manteniendo Lakehouse schemas según indique la experiencia actual, y preparar en \`Files\` una carpeta \`new_data\`. Si las opciones no aparecen, revisar la capacidad asignada, permisos del workspace y configuración administrativa.

**Verificación:** el Lakehouse aparece como elemento del workspace y el árbol Files permite seleccionar la nueva carpeta. No se presupone que crear el Lakehouse haya creado por sí solo una tabla de ventas.

### 6.3. Configurar Copy Data

Crear el Pipeline **Ingest Sales Data**. Agregar la actividad **Copy Data**, elegir una conexión HTTP y configurar \`https://raw.githubusercontent.com/MicrosoftLearning/dp-data/main/sales.csv\`, autenticación Anonymous y sin gateway. En el formato de entrada seleccionar **DelimitedText**, coma como delimitador y **First row as header** habilitado. Confirmar la conexión y examinar la vista previa cuando sea posible.

En el destino seleccionar el Lakehouse, **Root folder = Files**, **Directory = new_data**, **File name = sales.csv**. Guardar y ejecutar el Pipeline. Revisar **Output** para confirmar el estado de Copy Data. Abrir el Lakehouse y verificar la existencia de \`Files/new_data/sales.csv\`. El éxito de una copia no demuestra aún que se hayan creado tablas Delta.

### 6.4. Transformar mediante Notebook

Crear un Notebook **Load Sales** asociado al Lakehouse. Preparar la variable \`table_name\` en una **parameter cell** para que una actividad Notebook pueda proporcionar un nombre alternativo de tabla sin modificar el cuerpo de la transformación. A continuación ejecutar el ejemplo didáctico, adaptado con cast explícito de importes y fechas, y comparar sus resultados con el procedimiento original.

~~~python
from pyspark.sql import functions as F

table_name = "dbo.sales"  # Parameter cell en Fabric

df = (spark.read.format("csv")
    .option("header", True)
    .load("Files/new_data/sales.csv"))

sales = (df
    .withColumn("OrderDate", F.to_date("OrderDate"))
    .withColumn("Year", F.year("OrderDate"))
    .withColumn("Month", F.month("OrderDate"))
    .withColumn("FirstName", F.split(F.col("CustomerName"), " ").getItem(0))
    .withColumn("LastName", F.split(F.col("CustomerName"), " ").getItem(1)))

selected = sales.select(
    "SalesOrderNumber", "SalesOrderLineNumber", "OrderDate",
    "Year", "Month", "FirstName", "LastName",
    "EmailAddress", "Item", "Quantity", "UnitPrice", "TaxAmount"
)
selected.write.format("delta").mode("append").saveAsTable(table_name)
~~~

**Distinción importante:** este código es una adaptación pedagógica propia y puede requerir ajustar los nombres de columna o el esquema a la versión efectiva del CSV. El original utiliza también un \`append\` a Delta. No ejecutar repetidamente con el mismo lote sin medir los duplicados. En Lakehouses habilitados con schemas, \`dbo.sales\` identifica el esquema y la tabla; comprobar que el Lakehouse y el contexto de ejecución admiten esa referencia.

### 6.5. Encadenar actividades

Volver al Pipeline y agregar la actividad **Delete data** con nombre **Delete old files**, seleccionando \`Files/new_data\` y comodín \`*.csv\`. Enlazarla antes de Copy Data como propone el laboratorio. Agregar una actividad **Notebook** que seleccione \`Load Sales\` e introducir el parámetro \`table_name = dbo.new_sales\` (tipo String) para que se cree una tabla diferente al ejecutar el Notebook desde el Pipeline.

El material oficial utiliza relaciones **On completion** para algunos pasos; analizar críticamente sus implicaciones: completar una eliminación o una copia con fallo y continuar puede ser aceptable para un ejercicio, pero en una ingesta empresarial normalmente interesa que la fase de transformación solo empiece con **On success** de la copia.

Guardar, validar y ejecutar la secuencia. Abrir el Lakehouse y actualizar **Tables**. **Resultado esperado:** \`dbo.new_sales\` contiene los pedidos transformados. No se asignan recuentos concretos porque no se ha ejecutado el laboratorio contra el servicio ni validado el fichero en este entorno.

### 6.6. Diagnóstico técnico

| Síntoma | Causa probable | Acción |
|---|---|---|
| HTTP connection failed | URL no accesible, credenciales o política de red | Comprobar URL y Anonymous |
| Una columna aparece desplazada | Delimitador o encabezado incorrectos | Revisar opciones DelimitedText |
| Notebook no encuentra \`Files/new_data\` | Lakehouse no asociado o ruta errónea | Asociar Lakehouse y revisar Explorer |
| \`Spark SQL queries are only possible in the context of a lakehouse\` | Asociación de Lakehouse inconsistente | Seguir la corrección del laboratorio: quitar y volver a añadir el Lakehouse |
| No aparece \`dbo.new_sales\` | Nombre de parámetro erróneo o ejecución fallida | Revisar parameter cell y activity Output |
| Se duplican ventas | Se ha vuelto a ejecutar el Notebook con Append | Diseñar carga idempotente por lote |
| El Pipeline prosigue tras un fallo | Dependencia On completion | Revisar On success y ruta de fallo |
| Se dispara el coste de capacidad | Ejecutar repetidamente Spark o Dataflow | Revisar historial, frecuencia y capacidad |

### 6.7. Limpieza

Una vez recogidas las comprobaciones y las capturas que necesites para estudiar, eliminar únicamente el workspace aislado de pruebas desde **Workspace settings → General → Remove this workspace**. Este borrado es destructivo. Si existen más elementos compartidos, no borrar el workspace; eliminar únicamente los objetos del laboratorio. Revisar también programaciones y consumo de capacidad. No afirmar que se han liberado costes de capacidad dedicada solo por borrar las tablas.

## 7. Knowledge check y síntesis

Las preguntas siguientes son **de elaboración propia**, no reproducciones de las preguntas del examen:

1. **¿Qué encapsula un Pipeline?** Actividades y sus reglas de orquestación.
2. **¿En qué se diferencia un Pipeline de un Dataflow?** El primero coordina la ejecución; el segundo aplica transformaciones Power Query y puede escribir destinos.
3. **¿Qué significa On completion?** Que una actividad posterior se activa al finalizar la anterior, aunque no haya concluido con éxito.
4. **¿Por qué copiar a Files no es lo mismo que cargar una tabla?** Porque un archivo bruto todavía no se ha materializado ni registrado como tabla Delta.
5. **¿Para qué sirven los parámetros?** Para reutilizar una definición con entradas particulares en distintas ejecuciones.
6. **¿Qué información investigar primero ante un fallo?** Run ID, activity ID/nombre, estado, mensaje y entrada/salida aplicables.
7. **¿Qué riesgo tiene Append en un retry?** Insertar duplicados si no hay control de lotes.
8. **¿Cuándo es útil una plantilla?** Para reproducir un patrón de actividades, revisando conexiones y parámetros al importarlo.
9. **¿Cómo asegurar que un Notebook procese el destino correcto?** Verificar Lakehouse asociado y parámetro de tabla.
10. **¿Por qué no basta con que todos los pasos estén en verde?** Porque la validez semántica del dato requiere pruebas de calidad independientes.

**Resumen técnico:** seleccionar Copy Data cuando se necesite mover información, Notebook cuando se requiera transformación programable y Pipeline cuando haya que coordinar, programar y supervisar ambas tareas. El diseño productivo necesita control de acceso, datos de auditoría, validación y una estrategia de rerun.

## 8. Fuentes y alcance

- [Índice oficial (8 unidades)](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/use-data-factory-pipelines-fabric/index.yml): Introduction, Understand, Copy Data, Templates, Run and monitor, Exercise, Knowledge check y Summary.
- [Understand pipelines](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/use-data-factory-pipelines-fabric/includes/2-understand-fabric-pipeline.md).
- [Copy Data](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/use-data-factory-pipelines-fabric/includes/3-copy-data.md).
- [Templates](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/use-data-factory-pipelines-fabric/includes/4-pipeline-templates.md).
- [Monitor](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/use-data-factory-pipelines-fabric/includes/5-run-monitor-pipelines.md).
- [Laboratorio oficial](https://github.com/MicrosoftLearning/mslearn-fabric/blob/main/Instructions/Labs/04-ingest-pipeline.md), que fue revisado hasta su limpieza de recursos.

**Estado:** se explican las ocho unidades del módulo por sus competencias, incluida práctica y evaluación propia. El laboratorio está documentado, **no ejecutado**. El ejemplo de código adapta el ejercicio y debe comprobarse con el esquema del dataset.