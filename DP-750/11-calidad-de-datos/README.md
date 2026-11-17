[🏠 Índice](../README.md) · [← Anterior](../10-limpieza-transformacion-y-carga/README.md) · [Siguiente →](../12-diseno-de-pipelines/README.md)

> **Estado de estudio:** 🕒 Pendiente · **Unidades oficiales:** 8 · [Microsoft Learn](https://learn.microsoft.com/en-us/training/modules/implement-manage-data-quality-constraints-unity-catalog/)  
> **Versión de apuntes:** 15-11-2026 · Contenido técnico redactado de forma original a partir del itinerario de Microsoft.

# 11. Implementar y gestionar la calidad de datos en Azure Databricks

## 11.1. Por qué la calidad es una responsabilidad del pipeline

Una carga puede terminar correctamente y producir una tabla inutilizable. **Calidad de datos** significa que los registros satisfacen los requisitos de negocio y los contratos técnicos: campos requeridos presentes, tipos correctos, valores en rangos plausibles, claves no duplicadas, coherencia temporal y referencia a entidades válidas. No equivale a simplemente convertir errores a `NULL` o a aceptar cualquier fichero nuevo.

Hay cuatro puntos donde detectar incidencias: **ingesta** (formato y parseo), **Bronze** (observabilidad de crudos), **Silver** (validación y estandarización) y **Gold** (coherencia de agregados y métricas). Una canalización madura registra recuentos de entrada, rechazados, aceptados y motivos, para poder explicar la reducción de filas.

Una buena estrategia define tres posibles decisiones ante un error: **registrar y mantener** si la alerta es tolerable; **descartar o aislar** los registros problemáticos cuando el resto es utilizable; o **detener la ejecución** si la invalidez compromete todo el producto de datos. La elección depende del SLA, no de un comportamiento arbitrario de la librería.

## 11.2. Comprobaciones de nulabilidad, cardinalidad y rango

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

## 11.3. Comprobación de tipos: esquema físico frente a significado

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

## 11.4. Schema enforcement y schema drift

**Schema enforcement** es impedir que datos con forma incompatible entren en una tabla gobernada sin una decisión explícita. **Schema drift** describe cambios en la estructura de las fuentes a lo largo del tiempo: nuevas columnas, columnas ausentes, cambios de tipo, nombres renombrados o nuevas estructuras anidadas. No todos son peligrosos; lo crítico es que el sistema los haga visibles.

Delta Lake proporciona aplicación del esquema en escrituras y mecanismos controlados de evolución. Auto Loader puede inferir o recibir un esquema, conservar metadatos de esquema y gestionar modificaciones conforme a su modo configurado. Entre las estrategias están: fallar en una incompatibilidad, incorporar nuevas columnas de forma compatible o rescatar campos inesperados en un contenedor para inspección.

**Rescued data** no equivale a descartar. El contenido inesperado puede guardarse en una columna como `_rescued_data` para analizarlo posteriormente, conservando trazabilidad. Un cambio de origen debe revisarse antes de convertirlo silenciosamente en una característica del modelo Silver.

## 11.5. Expectations en Lakeflow Spark Declarative Pipelines

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

## 11.6. Monitorización y tratamiento operativo

La calidad debe poder responder: ¿cuántas filas entraron?, ¿cuántas no superaron una comprobación?, ¿qué error creció con respecto a ayer?, ¿se produjo una modificación inesperada de esquema? El grafo del pipeline, las métricas asociadas a expectations y el event log aportan evidencias. Un incremento de errores de parseo tras un despliegue puede indicar un cambio del contrato de origen.

Define umbrales de alerta que no se conviertan en ruido, responsable de la incidencia, tratamiento de registros en cuarentena y procedimiento para recuperar o reprocesar datos. Las reglas de negocio, por ejemplo la prohibición de importes negativos, deberían tener pruebas de regresión.

## 11.7. Repaso razonado

1. **¿Qué diferencia hay entre schema drift y datos erróneos?** El drift modifica la estructura; un dato erróneo puede respetar el esquema y aun así violar el dominio.
2. **¿Cuándo escogerías FAIL en lugar de DROP?** Si continuar genera resultados empresarialmente inseguros o inconsistentes.
3. **¿Por qué se inspecciona `_rescued_data`?** Porque contiene campos no reconocidos o incompatibles que requieren una decisión de modelado.
4. **¿Un cast de texto a decimal garantiza importes válidos?** No: puede fallar, y aun convertido el importe puede estar fuera del rango esperado.

---


## Desarrollo en profundidad: diseñar contratos verificables de calidad

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

---

## 🧪 Guía de laboratorio y comprobación práctica

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


---

### 📚 Fuentes y navegación

- [Contenido oficial de Microsoft Learn](https://learn.microsoft.com/en-us/training/modules/implement-manage-data-quality-constraints-unity-catalog/)
- [Repositorio docente de laboratorios DP-750T00](https://microsoftlearning.github.io/DP-750T00-Implement-Data-Engineering-Solutions-using-Azure-Databricks/)
- [Guía oficial de competencias del examen DP-750](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-750)

[🏠 Índice](../README.md) · [← Anterior](../10-limpieza-transformacion-y-carga/README.md) · [Siguiente →](../12-diseno-de-pipelines/README.md)
