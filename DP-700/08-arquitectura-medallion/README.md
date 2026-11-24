# 08 · Organize a Fabric lakehouse using medallion architecture design

> **Certificación:** DP-700. [Módulo oficial](https://learn.microsoft.com/en-us/training/modules/describe-medallion-architecture/) · [fuentes Markdown](https://github.com/MicrosoftDocs/learn/tree/main/learn-pr/wwl/describe-medallion-architecture) · [laboratorio oficial](https://github.com/MicrosoftLearning/mslearn-fabric/blob/main/Instructions/Labs/03b-medallion-lakehouse.md). **Revisión:** noviembre de 2026. **Laboratorio documentado, no ejecutado**.

## Índice

1. [Introducción](#1-introducción)
2. [Describe medallion architecture](#2-describe-medallion-architecture)
3. [Implement medallion in Fabric](#3-implement-medallion-in-fabric)
4. [Query and report](#4-query-and-report)
5. [Secure and govern](#5-secure-and-govern)
6. [Laboratorio oficial](#6-laboratorio-oficial)
7. [Knowledge check y summary](#7-knowledge-check-y-summary)
8. [Fuentes y control](#8-fuentes-y-control)

## 1. Introducción

Los sistemas operacionales organizan sus datos para registrar transacciones, no necesariamente para análisis. Un CSV recibido de una fuente puede contener columnas de tipos inconsistentes, filas duplicadas, valores que representan operaciones anuladas y un modelo que no se corresponde con las preguntas del negocio. Publicar ese mismo archivo directamente para analistas traslada la responsabilidad de limpieza a cada consumidor y multiplica las inconsistencias.

La **arquitectura medallion** propone refinar progresivamente la información en tres capas: **Bronze**, **Silver** y **Gold**. Es un patrón de organización y procesamiento, no un componente activable con un botón. En Fabric se puede implementar con Lakehouses, ficheros, tablas Delta y actividades de ingesta/transformación. La distribución física y la organización de permisos deben ajustarse al tamaño de la organización y los requisitos de gobierno.

## 2. Describe medallion architecture

### 2.1. Bronze: fidelidad al origen

La capa **Bronze** recibe datos en una forma lo más fiel posible al origen. El objetivo es conservar la evidencia de qué llegó, cuándo llegó y de qué sistema procedía, para que un cambio en la lógica de transformación no obligue necesariamente a volver a solicitar el histórico al proveedor. Se puede almacenar un CSV original en \`Files/bronze\` o normalizar físicamente el formato manteniendo el carácter de dato bruto, según el contrato.

No debe confundirse **bruto** con **sin gobernanza**. Bronze requiere permisos, retención y trazabilidad; puede contener datos sensibles o defectuosos. Un diseño robusto añade metadatos técnicos como identificador de lote, fecha de ingesta, nombre de fuente y checksum si corresponde. Si hay duplicados originales, pueden conservarse para auditoría, mientras la eliminación o resolución semántica de esos duplicados se implementa en Silver.

### 2.2. Silver: calidad y conformidad

La capa **Silver** aplica reglas repetibles de depuración: tipado, normalización de fechas, deduplicación, validación de claves, manejo de valores ausentes y unificación de entidades procedentes de varios orígenes. Silver es una base confiable para múltiples productos de datos, pero no todos los datos Silver deben resumirse. Es frecuente conservar el detalle suficiente para que posteriores transformaciones puedan crear distintas vistas Gold sin perder la trazabilidad.

Una regla de calidad debe incluir qué hacer con los registros inválidos. Descartar filas silenciosamente altera resultados y dificulta auditoría. Una estrategia más rigurosa separa registros válidos y una **quarantine** con motivo de rechazo, fecha y lote de procedencia. También se puede bloquear el pipeline cuando el porcentaje de errores supere un umbral aprobado.

### 2.3. Gold: consumo específico

La capa **Gold** modela información para preguntas concretas: hechos y dimensiones, agregaciones, indicadores, tablas para entrenamiento de IA o conjuntos de datos listos para un informe. La definición de Gold depende del producto de datos. Para ventas, una tabla de hechos puede representar líneas de pedido con claves a cliente, producto y fecha; un dashboard puede necesitar agregaciones diarias por región.

No es recomendable construir una tabla Gold agregada a un nivel que impida responder preguntas futuras necesarias. Conviene definir la granularidad antes de diseñar medidas y dimensiones.

~~~mermaid
flowchart LR
  R["CSV y sistemas fuente"] --> B["Bronze · copia fiel y metadatos"]
  B --> S["Silver · esquema, calidad, claves"]
  S --> G["Gold · hechos, dimensiones, KPIs"]
  G --> BI["Power BI / SQL / Apps"]
  B --> Q["Quarantine o errores de ingesta"]
  S --> Q
~~~

**Diagrama de elaboración propia.** La cuarentena es un patrón complementario, no un requisito para todos los diseños de Microsoft Learn.

### 2.4. Adaptar el patrón al negocio

No todos los proyectos requieren tres Workspaces y tres Lakehouses. Una demostración puede utilizar un Lakehouse con una carpeta Bronze y varias tablas Silver/Gold; una plataforma empresarial puede separar cada capa en distintos Workspaces para controlar permisos y costes. La decisión se basa en dominios, criticidad de datos, necesidad de aislamiento y complejidad operativa.

**Fuente:** [Describe medallion architecture](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/describe-medallion-architecture/includes/2-describe-medallion-architecture.md).

## 3. Implement medallion in Fabric

### 3.1. Diseño físico

Se pueden emplear carpetas dentro de un Lakehouse, esquemas o elementos separados, según las capacidades del entorno. En todos los casos debe establecerse una convención inequívoca. Si Bronze se guarda como CSV y Silver y Gold como tablas Delta, el procedimiento de carga y los permisos difieren. Si las tres capas se guardan en tablas Delta, se puede aprovechar versionado y determinadas operaciones transaccionales en cada capa, sin que ello implique una transacción global entre ellas.

La política de retención de Bronze debería considerar requisitos de auditoría, privacidad, coste y disponibilidad de fuentes. Conservar todo el histórico indiscriminadamente no siempre es apropiado. En Silver, identificar cambios de esquema y mantener versiones o migraciones cuando sean necesarias. En Gold, registrar la lógica de transformación y el significado de cada tabla.

### 3.2. Ingesta Bronze

Se puede utilizar Copy Data para traer un CSV a \`Files/bronze\`, o Dataflow Gen2 para conectar a un sistema origen. El patrón correcto depende del nivel de transformación aceptado en la capa bruta. Guardar solo una tabla muy transformada en Bronze limita la reconstrucción de reglas nuevas. A la inversa, duplicar grandes fuentes sin política de retención aumenta el coste.

### 3.3. Transformación Silver

Un Notebook permite definir un esquema y crear una tabla Silver a partir de ficheros Bronze. El siguiente ejemplo es **propio** y utiliza un esquema de pedidos compatible con el laboratorio oficial.

~~~python
from pyspark.sql import functions as F
from pyspark.sql.types import (
    StructType, StructField, StringType, IntegerType, DateType, DecimalType
)

schema = StructType([
    StructField("SalesOrderNumber", StringType()),
    StructField("SalesOrderLineNumber", IntegerType()),
    StructField("OrderDate", DateType()),
    StructField("CustomerName", StringType()),
    StructField("Email", StringType()),
    StructField("Item", StringType()),
    StructField("Quantity", IntegerType()),
    StructField("UnitPrice", DecimalType(18, 2)),
    StructField("Tax", DecimalType(18, 2)),
])

raw = (spark.read.format("csv")
       .option("header", False)
       .schema(schema)
       .load("Files/bronze/*.csv"))

silver = (raw
    .filter(F.col("SalesOrderNumber").isNotNull())
    .filter(F.col("Quantity") > 0)
    .dropDuplicates(["SalesOrderNumber", "SalesOrderLineNumber"])
    .withColumn("Year", F.year("OrderDate"))
    .withColumn("Month", F.month("OrderDate")))

silver.write.format("delta").mode("overwrite").saveAsTable("dbo.sales_silver")
~~~

Este código es una **ampliación conceptual**; no se debe dar por idéntico a los pasos del laboratorio oficial. Las columnas y el orden deben cotejarse con los ficheros reales antes de ejecutar. La clave compuesta de pedido y línea se ha elegido como hipótesis de unicidad; si el sistema permite cambios de línea, el diseño necesita una política de versionado.

### 3.4. Transformación Gold

A partir de datos Silver, pueden derivarse dimensiones y hechos. Una tabla Gold de ventas por mes puede elaborarse mediante:

~~~python
gold = spark.sql("""
    SELECT YEAR(OrderDate) AS Anio,
           MONTH(OrderDate) AS Mes,
           Item,
           SUM(Quantity) AS Unidades,
           SUM(Quantity * UnitPrice + Tax) AS Importe
    FROM dbo.sales_silver
    GROUP BY YEAR(OrderDate), MONTH(OrderDate), Item
""")

gold.write.format("delta").mode("overwrite").saveAsTable("dbo.sales_monthly_gold")
~~~

La interpretación del impuesto y la fórmula de importe dependen del contrato. Si \`Tax\` es un porcentaje y no un importe monetario, esta fórmula sería incorrecta: el ingeniero debe verificarlo. Esta es precisamente la diferencia entre «código que funciona» y «datos de negocio correctos».

### 3.5. Idempotencia y automatización

Una arquitectura medallion fiable no puede presuponer que un proceso se ejecutará siempre una sola vez. Bronze necesita control de archivos o lotes recibidos; Silver necesita reglas estables y un método de upsert o sustitución adecuado; Gold debe publicarse cuando sus entradas estén completas. El Pipeline debe orquestar las fases con dependencias y controles de calidad. Si Silver falla, no debería actualizarse Gold con datos del lote incompleto.

**Fuente:** [Implement medallion](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/describe-medallion-architecture/includes/3-implement-medallion-archecture-fabric.md).

## 4. Query and report

Una vez curada la capa Gold, debe existir un mecanismo de consulta para los consumidores. El **SQL analytics endpoint** del Lakehouse permite consultar tablas Delta compatibles; Power BI puede utilizar un modelo semántico para traducir estas tablas en medidas, relaciones y definiciones comprensibles por negocio.

El diseño de un **star schema** separa medidas cuantitativas en tablas de hechos y atributos descriptivos en dimensiones. Una dimensión de fecha, otra de producto y una de cliente pueden asociarse con una tabla de hechos de ventas. Es importante fijar la granularidad: una fila por línea de pedido no debe confundirse con una fila por pedido. La suma incorrecta de una medida en una relación muchos-a-muchos puede duplicar importes sin que el SQL genere errores.

**Modelado y rendimiento:** los consumidores necesitan columnas, filtros y medidas de negocio estables. Evitar construir cálculos enormes en cada informe si pueden resolverse con tablas Gold adecuadamente particionadas y mantenidas. También hay que considerar privacidad: una tabla Gold de agregaciones puede ser menos sensible que la Bronze original, pero hay que verificar que no se pueda reconstruir información personal a través de claves o combinaciones de dimensiones.

**Fuente:** [Query and report](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/describe-medallion-architecture/includes/4-query-report-data.md).

## 5. Secure and govern

### 5.1. Principio de mínimo privilegio

Una arquitectura por capas ofrece oportunidades de aislamiento. Los procesos de ingesta pueden escribir Bronze; los procesos de calidad pueden leer Bronze y escribir Silver; los consumidores de negocio pueden recibir acceso de lectura a Gold. Las identidades técnicas no deberían disponer de permisos administrativos generales salvo que realmente lo necesiten.

Fabric dispone de permisos de Workspace e ítem, además de controles de acceso a datos en los componentes que los soportan. No asumir que crear tres carpetas dentro del mismo Lakehouse crea tres fronteras de seguridad equivalentes. La política debe corresponder con las capacidades concretas de seguridad de OneLake, Lakehouse y el mecanismo de consulta.

### 5.2. Cambio, linaje y Git

El versionado de Notebooks, definiciones de Pipeline y otros artefactos facilita revisar cambios y regresar a configuraciones anteriores. Una commit de Git no guarda necesariamente el estado completo de los datos ni las versiones de las tablas; guarda los artefactos admitidos por la integración. La trazabilidad debe combinar versión de código, lote de entrada y versión de tabla publicada.

Un cambio en la Silver puede obligar a regenerar Gold y revisar modelos semánticos. La política de despliegue debe considerar dependencias, compatibilidad de esquemas, permisos y reversión.

**Fuente:** [Secure and govern](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/describe-medallion-architecture/includes/5-secure-govern.md).

## 6. Laboratorio oficial

**Fuente:** [03b-medallion-lakehouse.md](https://github.com/MicrosoftLearning/mslearn-fabric/blob/main/Instructions/Labs/03b-medallion-lakehouse.md) · [laboratorio navegable](https://microsoftlearning.github.io/mslearn-fabric/Instructions/Labs/03b-medallion-lakehouse.html). **Duración orientativa oficial:** 45 minutos. **Estado:** procedimiento consultado; sin ejecución real.

### 6.1. Escenario y preparación

Se implementa una arquitectura Bronze/Silver/Gold dentro de un Lakehouse **Sales**, utilizando los ficheros de pedidos correspondientes a 2019, 2020 y 2021. El laboratorio crea un Notebook para transformar Bronze a Silver, consulta la tabla desde SQL y después genera el modelo Gold con dimensiones y hechos. La creación de un modelo semántico es opcional. Es necesario Workspace con capacidad compatible y permisos para crear Lakehouse y Notebooks.

**Dataset oficial:** [orders.zip](https://github.com/MicrosoftLearning/dp-data/blob/main/orders.zip). Descargar, descomprimir y comprobar que contiene los tres CSV anuales. No se han obtenido ni verificado recuentos exactos aquí.

### 6.2. Bronze

Crear el Workspace y el Lakehouse \`Sales\` según las instrucciones. En \`Files\`, crear carpeta \`bronze\`. Subir \`2019.csv\`, \`2020.csv\` y \`2021.csv\` a esa carpeta, conservando los nombres originales. Verificar que aparecen en Lakehouse Explorer. En este punto los datos siguen siendo archivos; no existe todavía una tabla Silver.

### 6.3. Silver

Crear el Notebook **Transform data for Silver**. La práctica original define un \`StructType\` con columnas como SalesOrderNumber, SalesOrderLineNumber, OrderDate, CustomerName, Email, Item, Quantity, UnitPrice y Tax, y transforma los archivos brutos en una tabla Delta Silver.

En el desarrollo hay que explicar qué validaciones se hacen realmente y cuáles no. Definir un esquema y materializar Delta no garantiza por sí solo que los registros sean correctos. Añadir como verificación complementaria \`printSchema()\`, revisión de nulos en claves y recuentos de origen y destino. Los resultados exactos deben anotarse durante una ejecución real, no inferirse.

### 6.4. SQL endpoint

Abrir el SQL analytics endpoint del Lakehouse e inspeccionar la tabla Silver. Comprobar que se puede consultar mediante SELECT. Si no aparece, revisar el esquema, refrescar la vista y confirmar que la escritura se realizó correctamente. El endpoint SQL es una vía de consulta, no un sustituto universal de las operaciones Spark necesarias para las transformaciones.

### 6.5. Gold y modelo dimensional

La práctica desarrolla una capa Gold de hechos y dimensiones con los nombres:

- \`dimdate_gold\`
- \`dimcustomer_gold\`
- \`dimproduct_gold\`
- \`factsales_gold\`

Estas tablas no deben tratarse como cuatro agregaciones sin relación. Se pretende construir un esquema analítico en el que la tabla de hechos conserva las medidas de ventas y claves hacia fecha, producto y cliente. Comprobar claves únicas en las dimensiones, claves presentes en la tabla de hechos y ausencia de filas huérfanas.

**Verificación esperada:** después de ejecutar el Notebook correspondiente, las cuatro tablas aparecen en el explorador y pueden consultarse. No afirmar que el esquema o las relaciones del modelo semántico se han creado si no se ha realizado la parte opcional.

### 6.6. Modelo semántico opcional

Si se realiza la ampliación oficial, crear un modelo semántico a partir de las tablas Gold y definir relaciones apropiadas. Una relación entre hechos y dimensión debe construirse con la clave correcta y cardinalidad adecuada; no basta con unir columnas que se llaman parecido. Comprobar dirección de filtro y evitar relaciones ambiguas.

### 6.7. Diagnóstico y limpieza

| Problema | Posible origen | Comprobación o solución |
|---|---|---|
| CSV no localizable | Carpeta \`Files/bronze\` incorrecta | Revisar Files y nombres |
| Error de esquema | CSV sin cabecera / tipos inesperados | Aplicar esquema definido por la práctica |
| No se crea Silver | Fallo Spark, schema o permisos | Revisar Notebook y Lakehouse asociado |
| SQL no muestra Silver | Nombre/schema equivocado | Refrescar endpoint e inspeccionar catálogo |
| Gold contiene duplicados | Claves o joins mal diseñados | Revisar cardinalidad y claves naturales |
| Tablas de dimensiones vacías | Dependencia Silver incompleta | Comprobar filas y filtros de transformación |
| Relaciones incorrectas en Power BI | Cardinalidad o clave incompatibles | Revisar modelo estrella |
| Spark sigue ejecutándose | Sesiones o jobs del laboratorio | Detener operaciones activas y limpiar |

Al terminar, eliminar únicamente el Workspace creado para el laboratorio, si es exclusivo. Si contiene datos ajenos o trabajo compartido, eliminar solo los recursos propios con autorización. Revisar también la capacidad y las programaciones.

## 7. Knowledge check y summary

Cuestiones de elaboración propia:

1. **¿Cuál es el objetivo de Bronze?** Conservar el dato recibido y su trazabilidad.
2. **¿Qué se garantiza en Silver?** Aplicación de reglas de calidad y conformidad según el contrato definido.
3. **¿Para qué se diseña Gold?** Para productos de datos y necesidades concretas de consumo.
4. **¿Tres capas requieren tres Workspaces?** No, es una decisión arquitectónica de aislamiento.
5. **¿Qué es una quarantine?** Destino para registros que incumplen reglas definidas.
6. **¿Por qué identificar lote y fuente?** Para auditar, reprocesar y diagnosticar cargas.
7. **¿Qué relación existe entre tabla de hechos y dimensiones?** Medidas vinculadas a atributos descriptivos mediante claves.
8. **¿Gold debe publicar el resultado de una Silver fallida?** No debería, salvo un diseño explícito de datos parciales.
9. **¿Una carpeta Bronze constituye seguridad suficiente?** No; se requieren permisos efectivos.
10. **¿Git permite recuperar automáticamente un histórico de datos?** No, versiona artefactos admitidos, no reemplaza retención de los datos.
11. **¿La práctica produce un modelo semántico automáticamente?** Hay una parte opcional para crearlo.
12. **¿Qué riesgo tiene desduplicar con una clave equivocada?** Perder eventos legítimos o mantener duplicados.

**Resumen:** el patrón medallion transforma datos de origen en conjuntos confiables y listos para consumo. No es solo una convención de carpetas: define contratos, gobernanza, control de calidad, procesos de publicación y seguridad.

## 8. Fuentes y control

- [Índice oficial de ocho unidades](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/describe-medallion-architecture/index.yml).
- [Fundamentos Bronze/Silver/Gold](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/describe-medallion-architecture/includes/2-describe-medallion-architecture.md).
- [Implementación](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/describe-medallion-architecture/includes/3-implement-medallion-archecture-fabric.md).
- [Consumo](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/describe-medallion-architecture/includes/4-query-report-data.md).
- [Seguridad y gobernanza](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/describe-medallion-architecture/includes/5-secure-govern.md).
- [Laboratorio oficial DP-700](https://github.com/MicrosoftLearning/mslearn-fabric/blob/main/Instructions/Labs/03b-medallion-lakehouse.md).

**Control:** cubiertos fundamentos, arquitectura, implementación, consumo, seguridad, laboratorio, evaluación y síntesis; procedimientos técnicos documentados, **no ejecutados**.