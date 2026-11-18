# 11 · Gestionar restricciones de calidad

[Índice DP-750](../README.md) · [← Limpiar, transformar y cargar datos](../10-limpieza-transformacion-y-carga/README.md) · [Diseñar e implementar pipelines →](../12-diseno-de-pipelines/README.md)

**Progreso personal:** Pendiente de estudio. [Módulo oficial](https://learn.microsoft.com/en-us/training/modules/implement-manage-data-quality-constraints-unity-catalog/).

## Unidades

1. [Introduction](#unidad-01)
2. [Implement validation checks](#unidad-02)
3. [Implement data type checks](#unidad-03)
4. [Detect and manage schema drift](#unidad-04)
5. [Manage data quality with pipeline expectations](#unidad-05)
6. [Exercise - Implement and Manage Data Quality Constraints with Azure Databricks](#unidad-06)
7. [Module assessment](#unidad-07)
8. [Summary](#unidad-08)

- [Desarrollo aplicado](#desarrollo-aplicado)
- [Laboratorio y comprobaciones](#laboratorio-y-comprobaciones)

## Unidad 01

### Introduction

Una carga puede terminar correctamente y producir una tabla inutilizable. **Calidad de datos** significa que los registros satisfacen los requisitos de negocio y los contratos técnicos: campos requeridos presentes, tipos correctos, valores en rangos plausibles, claves no duplicadas, coherencia temporal y referencia a entidades válidas. No equivale a simplemente convertir errores a `NULL` o a aceptar cualquier fichero nuevo.

Hay cuatro puntos donde detectar incidencias: **ingesta** (formato y parseo), **Bronze** (observabilidad de crudos), **Silver** (validación y estandarización) y **Gold** (coherencia de agregados y métricas). Una canalización madura registra recuentos de entrada, rechazados, aceptados y motivos, para poder explicar la reducción de filas.

Una buena estrategia define tres posibles decisiones ante un error: **registrar y mantener** si la alerta es tolerable; **descartar o aislar** los registros problemáticos cuando el resto es utilizable; o **detener la ejecución** si la invalidez compromete todo el producto de datos. La elección depende del SLA, no de un comportamiento arbitrario de la librería.

[Unidad oficial](https://learn.microsoft.com/en-us/training/modules/implement-manage-data-quality-constraints-unity-catalog/1-introduction)

## Unidad 02

### Implement validation checks

**Nulabilidad.** Un `NULL` representa ausencia de valor, pero puede ser normal en un campo opcional. En los identificadores de negocio, importes o fechas de carga puede impedir una operación esencial. La definición del modelo debe indicar qué columnas admiten nulos y cómo se controla ese requisito.

**Cardinalidad y unicidad.** La cardinalidad indica el número de valores distintos. Por ejemplo, un identificador de reclamación debería tener alta cardinalidad y no repetirse en una tabla de estado actual; un campo de estado puede tener solo cuatro valores. `COUNT(DISTINCT ...)` no siempre detecta todas las causas: compara nulos y duplicados por clave compuesta cuando corresponda.

**Rangos y dominios.** Un importe de indemnización negativo puede ser válido para un ajuste contable, pero no para una reclamación original. La regla debe estar vinculada al contexto. Una fecha posterior al momento de ingreso puede indicar un error del origen o una previsión legítima.

```sql
SELECT
  COUNT(*) AS total,
  SUM(CASE WHEN claim_id IS NULL THEN 1 ELSE 0 END) AS sin_identificador,
  COUNT(DISTINCT claim_id) AS identificadores_distintos,
  SUM(CASE WHEN claim_amount < 0 THEN 1 ELSE 0 END) AS importes_negativos
FROM insurance_lab.bronze.claims_raw;
```

*Consulta didáctica:* si `claim_amount` llega como texto, convierte primero con una función tolerante como `try_cast(claim_amount AS DECIMAL(12,2))` y verifica los fallos, sin suponer que la tabla original ya tiene un campo numérico.

[Unidad oficial](https://learn.microsoft.com/en-us/training/modules/implement-manage-data-quality-constraints-unity-catalog/2-implement-validation-checks)

## Unidad 03

### Implement data type checks

El tipo `STRING` acepta casi cualquier representación, pero no garantiza que un campo denominado fecha o precio pueda operarse correctamente. `DECIMAL(12,2)` representa importes decimales con precisión fija, mientras que `DOUBLE` favorece cálculos numéricos aproximados. Para timestamps, define zona horaria y formato de origen, especialmente si provienen de países diferentes.

En PySpark, `withColumn` reemplaza o crea una columna y `cast` intenta representar el valor en el tipo indicado. En entornos con modo ANSI estricto, una conversión inválida puede generar una excepción, por lo que **`try_cast` en SQL** o un método de validación tolerante resulta más apropiado para auditar errores de origen. No construyas una regla de calidad suponiendo que todo cast fallido produce silenciosamente nulos.

```sql
SELECT
  claim_id,
  try_cast(claim_date AS DATE) AS fecha_limpia,
  try_cast(claim_amount AS DECIMAL(12,2)) AS importe_limpio
FROM insurance_lab.bronze.claims_raw;
```

El resultado es un conjunto de columnas tipadas, pero no una corrección automática de los datos: hay que tratar los valores nulos derivados de errores de parseo y distinguirlos de los nulos originales.

[Unidad oficial](https://learn.microsoft.com/en-us/training/modules/implement-manage-data-quality-constraints-unity-catalog/3-implement-data-type-checks)

## Unidad 04

### Detect and manage schema drift

**Schema enforcement** es impedir que datos con forma incompatible entren en una tabla gobernada sin una decisión explícita. **Schema drift** describe cambios en la estructura de las fuentes a lo largo del tiempo: nuevas columnas, columnas ausentes, cambios de tipo, nombres renombrados o nuevas estructuras anidadas. No todos son peligrosos; lo crítico es que el sistema los haga visibles.

Delta Lake proporciona aplicación del esquema en escrituras y mecanismos controlados de evolución. Auto Loader puede inferir o recibir un esquema, conservar metadatos de esquema y gestionar modificaciones conforme a su modo configurado. Entre las estrategias están: fallar en una incompatibilidad, incorporar nuevas columnas de forma compatible o rescatar campos inesperados en un contenedor para inspección.

**Rescued data** no equivale a descartar. El contenido inesperado puede guardarse en una columna como `_rescued_data` para analizarlo posteriormente, conservando trazabilidad. Un cambio de origen debe revisarse antes de convertirlo silenciosamente en una característica del modelo Silver.

#### Schema drift y cuarentena

Añadir una columna nueva es un cambio de contrato distinto de cambiar el tipo de una columna existente. Auto Loader permite gestionar información rescatada para inspeccionar valores o campos que la lectura no pudo encajar. Guarda el contenido rescatado junto con identificación del fichero y fecha de ingesta. Con ese punto de partida el responsable puede decidir cómo evolucionar Silver. Define el comportamiento ante cambios compatibles y no compatibles y prueba qué ocurre después de reintentar la carga con el mismo checkpoint.

Una expectativa declarativa con política de descarte elimina filas del resultado del dataset. Para conservar sus datos en cuarentena, diseña una rama explícita que seleccione las filas inválidas y las persista con el motivo correspondiente. Conserva también las métricas de volumen para reconciliar flujo válido e inválido. Si faltan filas, revisa primero la política aplicada en cada dataset y comprueba que los recuentos se comparan en el mismo punto del pipeline.

[Unidad oficial](https://learn.microsoft.com/en-us/training/modules/implement-manage-data-quality-constraints-unity-catalog/4-detect-manage-schema-drift)

## Unidad 05

### Manage data quality with pipeline expectations

Una **expectation** asocia un nombre a una condición verificable sobre los registros. En Lakeflow pueden definirse reglas en Python y escoger una acción en caso de incumplimiento. La sintaxis habitual utiliza decoradores del paquete `pyspark.pipelines` importado como `dp` cuando el runtime lo admite.

| Comportamiento | Mecanismo conceptual | Resultado |
|---|---|---|
| Conservar y registrar | `@dp.expect` | La fila continúa; la violación se contabiliza |
| Rechazar fila | `@dp.expect_or_drop` | La fila incumplidora se excluye del resultado |
| Bloquear actualización | `@dp.expect_or_fail` | La actualización falla ante una violación |

```python
from pyspark import pipelines as dp

@dp.table(name="claims_validated")
@dp.expect_or_drop("id_obligatorio", "claim_id IS NOT NULL")
@dp.expect("estado_reconocido", "status IN ('OPEN','PENDING','CLOSED')")
def claims_validated():
    return spark.read.table("insurance_lab.bronze.claims_raw")
```

Este fragmento es **ilustrativo**. No reemplaza el pipeline completo oficial, que contiene reglas de tipado y más expectativas. Debes revisar el soporte para funciones y tipos de entrada de tu versión de Databricks.

#### Decidir qué pasa con las filas inválidas

Una regla que informa sin filtrar es útil durante el arranque de un pipeline, cuando todavía estás midiendo cuántos productores incumplen el contrato. La política de descarte se ajusta a datasets de consumo cuando existe una regla explícita para excluir esos registros y no es posible repararlos a tiempo. Para una condición que invalida información contable que debe publicarse completa, configura el fallo y prepara una ruta de reparación. Los resultados de la prueba deben mostrar qué filas llegaron al dataset y qué impacto tiene cada violación en la actualización. En un fallo, conserva evidencia de la ejecución y mide fallos y rechazos en fuentes de observabilidad apropiadas.

```sql
CREATE OR REFRESH STREAMING TABLE silver_reclamaciones (
  CONSTRAINT id_presente EXPECT (reclamacion_id IS NOT NULL)
    ON VIOLATION DROP ROW
)
AS SELECT reclamacion_id, importe
FROM STREAM bronze_reclamaciones;
```

Este ejemplo complementario presupone una streaming table Bronze en el namespace del pipeline. Introduce una fila con identificador y otra sin él. La válida debe estar en Silver. La inválida debe quedar fuera. Comprueba que la métrica de calidad refleja el descarte y que la fila original sigue disponible en Bronze para investigación.

[Unidad oficial](https://learn.microsoft.com/en-us/training/modules/implement-manage-data-quality-constraints-unity-catalog/5-manage-data-quality-pipeline-expectations)

## Unidad 06

### Exercise - Implement and Manage Data Quality Constraints with Azure Databricks

La actividad de esta unidad se desarrolla en [Laboratorio y comprobaciones](#laboratorio-y-comprobaciones), con requisitos, secuencia de trabajo y criterios para revisar el resultado.

[Unidad oficial](https://learn.microsoft.com/en-us/training/modules/implement-manage-data-quality-constraints-unity-catalog/6-exercise-implement-manage-data-quality-constraints-unity-catalog)

## Unidad 07

### Module assessment

1. **¿Qué diferencia hay entre schema drift y datos erróneos?** El drift modifica la estructura; un dato erróneo puede respetar el esquema y aun así violar el dominio.
2. **¿Cuándo escogerías FAIL en lugar de DROP?** Si continuar genera resultados empresarialmente inseguros o inconsistentes.
3. **¿Por qué se inspecciona `_rescued_data`?** Porque contiene campos no reconocidos o incompatibles que requieren una decisión de modelado.
4. **¿Un cast de texto a decimal garantiza importes válidos?** No: puede fallar, y aun convertido el importe puede estar fuera del rango esperado.

[Unidad oficial](https://learn.microsoft.com/en-us/training/modules/implement-manage-data-quality-constraints-unity-catalog/7-knowledge-check)

## Unidad 08

### Summary

Las reglas de calidad deben especificar condición, punto de aplicación y reacción. Valida esquema antes de interpretar significado. Aísla datos recuperables para inspección y distingue el volumen rechazado de un problema operativo que exige parar la publicación. En un pipeline, archiva métricas por ejecución y fija umbrales con el responsable de los datos. Para comprobar una expectativa aporta casos que se sabe que pasan y casos que incumplen específicamente una regla.

[Unidad oficial](https://learn.microsoft.com/en-us/training/modules/implement-manage-data-quality-constraints-unity-catalog/8-summary)

## Desarrollo aplicado

### Calidad es una propiedad medible

Un dato puede ser sintácticamente válido pero incorrecto para el negocio. Por ejemplo, una columna `importe` de tipo `DOUBLE` permite `-100.0`, aunque un pedido confirmado quizá nunca debería tener importe negativo. Conviene separar controles estructurales (columnas, tipos, claves), controles de contenido (rangos, valores permitidos), consistencia entre entidades y frescura (cuánto tarda en llegar la información).

Cada regla debe tener nombre, severidad, propietario y acción cuando se incumple. La respuesta no siempre es eliminar la fila: para datos auditables puede ser mejor enviarla a una zona de cuarentena con motivo de rechazo y metadatos de origen.

### Schema enforcement y schema evolution

**Enforcement** evita aceptar estructuras incompatibles con el esquema del destino. **Evolution** permite incorporar cambios compatibles cuando está configurado explícitamente. Si una fuente cambia `precio` de número a texto, dejar que todo se convierta automáticamente puede destruir garantías analíticas. Si agrega una columna opcional, tal vez quieras aceptarla y registrarla. El **schema drift** debe tratarse como un acontecimiento observable, no como una sorpresa silenciosa.

```python
from pyspark.sql import functions as F
pedidos = spark.table("estudio.bronze.pedidos")
validos = pedidos.filter(
    F.col("id_pedido").isNotNull() &
    F.col("importe").isNotNull() &
    (F.col("importe") >= 0)
)
rechazados = pedidos.exceptAll(validos)
```

`exceptAll` ilustra cómo conservar duplicados en una diferencia de datasets, pero no es el mecanismo más eficiente ni siempre el más conveniente para cuarentena. En proyectos reales, añade una columna `motivo_rechazo` y separa datos por condición para poder explicar por qué se descartó cada fila.

### Expectations en Lakeflow Declarative Pipelines

Las expectativas expresan condiciones de calidad asociadas al procesamiento. Según la sintaxis y las capacidades del pipeline pueden registrarse incumplimientos, descartar filas o hacer fallar una actualización. La selección tiene consecuencias: descartar un dato incorrecto es cómodo para una tabla de consumo, pero puede ocultar pérdida de registros; detener la actualización protege la integridad a costa de disponibilidad. Para datos financieros, un rechazo explícito y rastreable suele ser preferible a una eliminación silenciosa.

### Métricas y umbrales

Mide tanto el número absoluto de errores como su porcentaje. Diez filas incorrectas en una ingestión de veinte es alarmante; diez en cien millones puede requerir otro tratamiento. Una regla de unicidad sobre `id_pedido` puede ser equivocada si la tabla tiene grano «una línea de pedido»; en ese caso la clave sería `(id_pedido, num_linea)`.

**Laboratorio de calidad:** construye diez filas con dos importes negativos, un identificador nulo, una fecha inválida y un registro duplicado. Define el grano y clasifica las incidencias. Carga válidos y rechazados en tablas distintas, añade métricas por motivo y simula un cambio de esquema en el origen. En cada paso explica si el proceso continúa, omite registros o se detiene; la decisión debe obedecer a una política documentada.

## Laboratorio y comprobaciones

**Ejercicio oficial:** [Abrir las instrucciones de MicrosoftLearning](https://microsoftlearning.github.io/DP-750T00-Implement-Data-Engineering-Solutions-using-Azure-Databricks/Instructions/Labs/09-implement-manage-data-quality-constraints-unity-catalog.html)  
**Notebook oficial:** [Importar o consultar notebook `.ipynb`](https://raw.githubusercontent.com/MicrosoftLearning/DP-750T00-Implement-Data-Engineering-Solutions-using-Azure-Databricks/refs/heads/main/Allfiles/09-implement-manage-data-quality-constraints-unity-catalog.ipynb)  
**Naturaleza de esta sección:** itinerario explicado con redacción propia y comprobaciones de aprendizaje. Las celdas exactas y el material de origen están en el notebook oficial enlazado; no se reproducen íntegramente aquí.


### Caso práctico: ClearCover Insurance — calidad declarativa en Lakeflow

**Escenario oficial:** una aseguradora recibe solicitudes con identificadores faltantes, importes negativos, fechas que no se pueden convertir y nuevas columnas inesperadas. El trabajo separa la preparación del entorno (notebook) de la definición de un pipeline Python.

**Ejecución recomendada, por fases:**

1. Importa y ejecuta el notebook de preparación. Debe dejar el catálogo `insurance_lab`, esquemas `bronze`, `silver` y `gold`, un volumen de ficheros y una tabla `insurance_lab.bronze.claims_raw` con registros de ensayo.
2. Inspecciona las columnas `claim_id`, `customer_id`, `claim_date`, `claim_amount`, `status` y `coverage_amount`. Identifica nulos, textos no convertibles, números negativos y estados fuera del vocabulario admitido.
3. Importa **también** el archivo oficial del pipeline Python: [09-implement-manage-data-quality-constraints-unity-catalog.py](https://raw.githubusercontent.com/MicrosoftLearning/DP-750T00-Implement-Data-Engineering-Solutions-using-Azure-Databricks/refs/heads/main/Allfiles/09-implement-manage-data-quality-constraints.py). En Jobs & Pipelines crea una canalización ETL con destino `insurance_lab.silver`, modo triggered y cómputo compatible.
4. En la definición de `claims_validated`, combina expectativas: **DROP** para identificadores ausentes, **WARN** para estados no permitidos y **FAIL** para reglas cuyo incumplimiento deba bloquear el pipeline. Anota por qué el equipo escoge cada política.
5. Convierte `claim_date` a `DATE` e `claim_amount` a `DECIMAL(12,2)` con PySpark. Utiliza `try_cast` donde necesites una conversión tolerante en SQL; no asumas que todo cast inválido devuelve NULL: el comportamiento varía según el modo ANSI y la función empleada.
6. Define las comprobaciones posteriores a la conversión: fechas válidas, importes presentes y no negativos. Inspecciona qué registros habrán sido descartados.
7. En una función de lectura adicional, prueba **Auto Loader** con `cloudFiles`, lectura CSV, `schemaEvolutionMode = rescue` y columna `_rescued_data` para capturar campos no reconocidos. Asegura una ruta de esquema/checkpoint que no se confunda con los datos de entrada.
8. Ejecuta el pipeline y consulta el grafo. Comprueba los datasets de reclamaciones validadas, datos rescatados y el resumen de la capa Gold. Revisa las métricas por expectativa y el event log cuando un paso falle.

**Criterios de aceptación:** ninguna reclamación Silver tiene identificadores obligatorios nulos; las fechas e importes presentan los tipos esperados; la capa de rescate conserva información de columnas nuevas; las métricas explican qué datos se descartaron, conservaron o provocaron el fallo.

**Atención:** un `expect_or_fail` sobre un registro de prueba inválido puede interrumpir intencionadamente la ejecución. Es un resultado de aprendizaje que debes diagnosticar, no silenciar cambiando a WARN sin comprender el requisito.

### Prueba complementaria con datos sintéticos

En una tabla de pruebas, el siguiente control sirve para diseñar las filas de entrada antes de trasladar reglas al pipeline oficial. Ejecuta primero el SQL y comprueba que cada fila falla por la causa prevista.

```sql
WITH entradas(id, importe_txt) AS (
  SELECT * FROM VALUES (1, '15.40'), (2, '-3.10'), (3, 'N/D')
),
tipadas AS (
  SELECT id, importe_txt, try_cast(importe_txt AS DECIMAL(12,2)) AS importe
  FROM entradas
)
SELECT id, importe_txt, importe,
  CASE
    WHEN importe IS NULL THEN 'importe_invalido'
    WHEN importe < 0 THEN 'importe_negativo'
    ELSE 'valido'
  END AS resultado
FROM tipadas
ORDER BY id;
```

El esperado es una fila válida y dos rechazables por reglas distintas. `TRY_CAST` convierte los valores incompatibles en nulos en este ejemplo. Decide si un valor ausente se trata igual que una cadena malformada. Si tienen implicaciones diferentes, expresa las reglas en orden y separa las métricas por motivo.

## Fuentes y navegación

- [Fuentes de la certificación](../FUENTES.md)
- [Preparación y limpieza del entorno](../PREPARACION-LABORATORIOS.md)

[Índice DP-750](../README.md) · [← Limpiar, transformar y cargar datos](../10-limpieza-transformacion-y-carga/README.md) · [Diseñar e implementar pipelines →](../12-diseno-de-pipelines/README.md)
