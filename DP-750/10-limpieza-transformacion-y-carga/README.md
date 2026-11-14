[🏠 Índice](../README.md) · [← Anterior](../09-ingesta-de-datos/README.md) · [Siguiente →](../11-calidad-de-datos/README.md)

> **Estado de estudio:** ✅ Visto · **Unidades oficiales:** 11 · [Microsoft Learn](https://learn.microsoft.com/en-us/training/modules/cleanse-transform-load-data-into-unity-catalog/)  
> **Versión de apuntes:** 14-11-2026 · Contenido técnico redactado de forma original a partir del itinerario de Microsoft.

# 10. Limpiar, transformar y cargar datos

**Referencia Microsoft Learn:** [Limpiar, transformar y cargar datos](https://learn.microsoft.com/en-us/training/modules/cleanse-transform-load-data-into-unity-catalog/) · **11 unidades oficiales**.

## 10.1. De datos crudos a datos confiables

La calidad no es una operación única. Primero se exploran anomalías, se definen reglas, se aplican transformaciones y se miden sus resultados. Una plataforma fiable conserva suficiente trazabilidad para explicar por qué una fila fue rechazada o modificada.

**Dimensiones frecuentes:** completitud, validez, unicidad, consistencia, actualidad y exactitud. No todas son verificables automáticamente; por ejemplo, un email puede cumplir una expresión regular y seguir siendo falso.

## 10.2. Perfilado de datos

El **data profiling** calcula frecuencias, rangos, proporción de nulos, valores distintos, distribuciones y posibles anomalías antes de decidir las transformaciones. Es un diagnóstico, no una corrección.

```sql
SELECT
  COUNT(*) AS filas,
  COUNT(DISTINCT id_pedido) AS pedidos_distintos,
  SUM(CASE WHEN id_cliente IS NULL THEN 1 ELSE 0 END) AS sin_cliente,
  MIN(importe) AS importe_minimo,
  MAX(importe) AS importe_maximo
FROM estudio.ventas.pedidos_raw;
```

Interpretación: si el número de pedidos distintos es menor que el total, puede haber duplicados; pero también podrían existir varias líneas por pedido. Antes de borrar registros hay que comprobar el grano esperado.

## 10.3. Elegir tipos de datos

Un tipo correcto mejora almacenamiento, integridad y comportamiento de los cálculos. **STRING** para texto e identificadores que no se calculan; **BIGINT/INT** para enteros; **DECIMAL(p,s)** para cantidades exactas como dinero; **DOUBLE** para magnitudes aproximadas; **DATE** y **TIMESTAMP** para fechas/instantes; **BOOLEAN** para estados lógicos.

No utilices `FLOAT`/`DOUBLE` indiscriminadamente en importes: la representación binaria puede dar lugar a pequeñas diferencias. Una fecha no debería permanecer como texto si se pretende filtrar y comparar temporalmente. La conversión requiere estudiar formatos y valores inválidos.

```sql
SELECT
  TRY_CAST(id_cliente AS BIGINT) AS id_cliente,
  TRY_CAST(importe AS DECIMAL(12,2)) AS importe,
  TRY_CAST(fecha AS DATE) AS fecha
FROM estudio.ventas.pedidos_raw;
```

`TRY_CAST` puede devolver `NULL` en conversiones que no sean válidas en vez de abortar como una conversión estricta, según las reglas del tipo y el motor.

## 10.4. Resolver nulos y duplicados

**NULL** significa valor desconocido o ausente; no es lo mismo que cero ni que cadena vacía. SQL utiliza lógica de tres valores: una comparación con `NULL` normalmente no devuelve `TRUE` ni `FALSE`, sino `UNKNOWN`. Para detectarlo se utiliza `IS NULL` / `IS NOT NULL`, no `= NULL`.

```sql
SELECT * FROM estudio.ventas.pedidos_raw
WHERE id_cliente IS NOT NULL;

SELECT COALESCE(canal, 'sin_especificar') AS canal
FROM estudio.ventas.pedidos_raw;
```

La deduplicación necesita una clave de negocio y una regla sobre qué fila conservar. Si dos registros comparten ID, conservar el más reciente requiere una fecha o secuencia fiable. `DISTINCT` elimina filas idénticas de la selección; no resuelve por sí solo duplicados semánticos con diferentes timestamps.

```sql
WITH ordenados AS (
  SELECT *,
    ROW_NUMBER() OVER (
      PARTITION BY id_pedido ORDER BY fecha_actualizacion DESC
    ) AS rn
  FROM estudio.ventas.pedidos_raw
)
SELECT * EXCEPT (rn) FROM ordenados WHERE rn = 1;
```

Esquema didáctico: se supone que existe `fecha_actualizacion`; cuando hay empates, añadir un desempate determinista.

## 10.5. Filtros, grupos y agregaciones

`WHERE` filtra filas **antes** del agrupamiento; `HAVING` filtra **grupos** después de agregar. `GROUP BY` determina los grupos y funciones como `SUM`, `COUNT`, `AVG`, `MIN`, `MAX` calculan estadísticas.

```sql
SELECT id_cliente, COUNT(*) AS num_pedidos, SUM(importe) AS gasto
FROM estudio.ventas.pedidos
WHERE importe > 0
GROUP BY id_cliente
HAVING COUNT(*) >= 3;
```

Esta consulta cuenta pedidos con importe positivo, calcula su gasto y muestra clientes con al menos tres. Los filtros aplicados antes de agregar cambian la población analizada y, por tanto, el significado de las métricas.

## 10.6. Joins y operadores de conjuntos

| Operación | Resultado |
|---|---|
| `INNER JOIN` | Filas con coincidencia en ambos lados |
| `LEFT JOIN` | Todas las del lado izquierdo, coincidan o no |
| `FULL OUTER JOIN` | Filas de ambos lados, con nulos donde no hay coincidencia |
| `LEFT ANTI JOIN` | Filas del lado izquierdo sin coincidencia |
| `UNION` | Combina resultados compatibles eliminando duplicados |
| `UNION ALL` | Combina manteniendo duplicados |
| `INTERSECT` | Filas presentes en ambos conjuntos |
| `EXCEPT` | Filas del primer conjunto que no aparecen en el segundo |

```sql
SELECT p.id_pedido, c.nombre, p.importe
FROM estudio.ventas.pedidos p
LEFT JOIN estudio.ventas.clientes c
  ON p.id_cliente = c.id_cliente;
```

Un `JOIN` puede multiplicar filas cuando las claves no son únicas. Antes de agregar después del join hay que confirmar cardinalidades y grano. Los operadores de conjuntos requieren columnas compatibles y el mismo número de posiciones.

## 10.7. Denormalizar, pivotar y despivotar

**Denormalización** combina atributos para facilitar consultas y reducir uniones, a cambio de posible redundancia y mantenimiento. **PIVOT** transforma valores de filas en columnas agregadas; **UNPIVOT** transforma columnas en filas, útil para normalizar archivos anchos.

```sql
SELECT * FROM (
  SELECT region, canal, importe FROM estudio.ventas.pedidos
)
PIVOT (SUM(importe) FOR canal IN ('web', 'tienda'));
```

Un pivot requiere conocer las categorías que se materializan como columnas o generar la consulta dinámicamente. No debe confundirse con una operación que agregue «automáticamente» todos los posibles valores futuros.

## 10.8. INSERT, APPEND, OVERWRITE y MERGE

- **`INSERT INTO` / append:** incorpora nuevos registros sin reemplazar los anteriores; correcto cuando llegan nuevos datos y no existen colisiones de clave.
- **Overwrite:** sustituye el conjunto de datos especificado; apropiado para cargas completas controladas, pero peligroso si se aplica a un destino equivocado.
- **`MERGE INTO`:** realiza cambios condicionales según coincidencia en una clave. Resulta habitual en upsert (actualizar existentes, insertar nuevos) y CDC.

```sql
MERGE INTO estudio.ventas.clientes AS destino
USING estudio.ventas.clientes_nuevos AS fuente
ON destino.id_cliente = fuente.id_cliente
WHEN MATCHED THEN UPDATE SET
  destino.nombre = fuente.nombre,
  destino.ciudad = fuente.ciudad
WHEN NOT MATCHED THEN INSERT (id_cliente, nombre, ciudad)
VALUES (fuente.id_cliente, fuente.nombre, fuente.ciudad);
```

Este `MERGE` ilustra un **SCD Tipo 1** sobre el estado actual. No preserva automáticamente versiones antiguas. Además, una fuente con varias filas por clave puede crear ambigüedades o fallos: resuelve antes la unicidad de la clave de emparejamiento.

## 10.9. Laboratorio final y evaluación

Diseña una tubería con estas etapas: CSV bruto → perfilado → tipado explícito → detección de registros inválidos → deduplicación por clave y fecha → join con clientes → agregación por canal → `MERGE` hacia la tabla de consumo. Conserva un dataset de rechazos que explique la causa de cada exclusión.

**Comprobaciones que debes poder explicar:** ¿por qué `WHERE campo = NULL` no funciona? ¿cuándo usar `DECIMAL`? ¿cómo detectar un join que duplica importes? ¿qué diferencia hay entre `INSERT` y `MERGE`? ¿cómo afecta `HAVING` a un grupo?

---

---

## 🧪 Guía de laboratorio y comprobación práctica

**Ejercicio oficial:** [Abrir las instrucciones de MicrosoftLearning](https://microsoftlearning.github.io/DP-750T00-Implement-Data-Engineering-Solutions-using-Azure-Databricks/Instructions/Labs/08-cleanse-transform-load-data-into-unity-catalog.html)  
**Notebook oficial:** [Importar o consultar notebook `.ipynb`](https://raw.githubusercontent.com/MicrosoftLearning/DP-750T00-Implement-Data-Engineering-Solutions-using-Azure-Databricks/refs/heads/main/Allfiles/08-cleanse-transform-load-data-into-unity-catalog.ipynb)  
**Naturaleza de esta sección:** itinerario explicado con redacción propia y comprobaciones de aprendizaje. Las celdas exactas y el material de origen están en el notebook oficial enlazado; no se reproducen íntegramente aquí.


### Caso práctico: Pristine Properties — convertir un conjunto irregular en analítica confiable

**Problema:** datos de viviendas de varias delegaciones llegan duplicados, con importes inconsistentes, nulos y estadísticas organizadas en columnas demasiado anchas.

**Fases recogidas del laboratorio de Microsoft:**

1. Importa el notebook y genera el catálogo `realestate_lab`. En la capa Bronze localiza `realestate_lab.bronze.listings` y revisa el esquema inicial.
2. Realiza un **perfilado**: recuentos por columna, porcentaje de nulos, cardinalidad por identificador y distribución de precios. Una comprobación útil es comparar `COUNT(*)` y `COUNT(DISTINCT listing_id)` para detectar sospechas de duplicados.
3. Cambia los tipos cuando corresponda: importes monetarios con `DECIMAL(p,s)` en lugar de `FLOAT`, fechas y timestamps correctamente parseados y claves identificadoras con tipos adecuados. Documenta los valores que no se convierten.
4. Define una política de duplicados: por ejemplo, conservar la revisión más reciente de cada vivienda según la fecha de actualización; no borres filas únicamente porque se repita alguna característica comercial.
5. Define la semántica de los nulos: elimina registros irrecuperables para el uso previsto e imputa otros valores únicamente cuando exista una regla defendible. Guarda conteos de filas antes y después de cada limpieza.
6. Une el conjunto depurado con tablas de agentes y transacciones. Compara INNER JOIN con LEFT JOIN; identifica cuántas viviendas quedan fuera cuando falta su agente.
7. Transforma estadísticas mediante **PIVOT** y **UNPIVOT**. Comprueba la cardinalidad, que los periodos son comparables y que no se mezclan medidas incompatibles.
8. **Trabajo de UI:** en Catalog Explorer → tabla `listings` → sección de calidad, configura un perfil de tipo **Snapshot**, si la característica y tus permisos lo permiten. Revisa recuentos de nulos, valores distintos y distribuciones. La interfaz puede cambiar con nuevas versiones.

**Entrega recomendada:** tabla Silver de viviendas depuradas, consulta Gold de precios medios por ciudad y una bitácora con recuentos de registros eliminados o transformados. Esta última propuesta de entrega es complementaria, no un requisito literal del laboratorio.


---

### 📚 Fuentes y navegación

- [Contenido oficial de Microsoft Learn](https://learn.microsoft.com/en-us/training/modules/cleanse-transform-load-data-into-unity-catalog/)
- [Repositorio docente de laboratorios DP-750T00](https://microsoftlearning.github.io/DP-750T00-Implement-Data-Engineering-Solutions-using-Azure-Databricks/)
- [Guía oficial de competencias del examen DP-750](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-750)

[🏠 Índice](../README.md) · [← Anterior](../09-ingesta-de-datos/README.md) · [Siguiente →](../11-calidad-de-datos/README.md)
