[🏠 Índice](../README.md) · [← Anterior](../07-gobernar-objetos-de-unity-catalog/README.md) · [Siguiente →](../09-ingesta-de-datos/README.md)

> **Estado de estudio:** ✅ Visto · **Unidades oficiales:** 14 · [Microsoft Learn](https://learn.microsoft.com/en-us/training/modules/design-implement-data-modeling-unity-catalog/)  
> **Versión de apuntes:** 14-11-2026 · Contenido técnico redactado de forma original a partir del itinerario de Microsoft.

# 8. Diseñar e implementar el modelado de datos

**Referencia Microsoft Learn:** [Diseñar e implementar el modelado de datos](https://learn.microsoft.com/en-us/training/modules/design-implement-data-modeling-unity-catalog/) · **14 unidades oficiales**.

## 8.1. Modelar antes de procesar

El modelado de datos define qué representa cada tabla, cuál es su **granularidad**, qué claves utiliza y cómo se relaciona con otras. No se trata únicamente de elegir tipos de columnas. Un mal grano produce duplicaciones en agregados, mientras que una clave inestable dificulta integraciones y actualizaciones.

## 8.2. Diseño de ingesta y fuentes

Antes de construir un pipeline, caracteriza cada origen: fichero, base de datos, evento o API; disponibilidad batch/streaming; volumen y velocidad; esquema; autenticación; tratamiento de errores; semántica de cambios y reenvíos. También debe definirse el contrato del destino: tabla de llegada inmutable, copia actual o historial versionado.

**Preguntas que evitan fallos:** ¿cómo se detecta una actualización de una fila? ¿pueden llegar eventos duplicados? ¿qué ocurre con datos fuera de orden? ¿el origen permite carga inicial y CDC posterior? ¿existe una fecha fiable de modificación?

## 8.3. Elegir herramienta de ingesta

- **Lakeflow Connect:** conectores administrados para fuentes soportadas, reduciendo código de integración.
- **Auto Loader:** detección incremental de archivos que aparecen en almacenamiento.
- **COPY INTO:** importación incremental por SQL para archivos compatibles.
- **Structured Streaming:** ingestión continua o microbatch con control de estado y checkpoints.
- **Notebooks / Jobs:** personalización cuando se necesita lógica adicional.

El criterio de elección incluye requisitos de latencia, mantenimiento, disponibilidad del conector, volumen, semántica de errores y control de transformaciones.

## 8.4. Formatos de tabla: Delta e Iceberg

**Delta Lake** combina archivos columnares con un registro transaccional que permite operaciones ACID, manejo de esquema y cambios controlados. **Apache Iceberg** es otro formato abierto para tablas analíticas con metadatos de snapshots y evolución de esquemas. No son simplemente «formatos de archivo»: son formatos de tabla con reglas sobre metadatos y mantenimiento.

Su interoperabilidad depende de versiones, motores y características admitidas. Para funciones como `MERGE`, streaming y políticas gestionadas, hay que comprobar compatibilidad real en Databricks antes de elegir un formato por moda.

## 8.5. Particionado

Particionar separa físicamente datos siguiendo ciertos valores (por ejemplo mes), lo que puede permitir evitar leer particiones irrelevantes. Pero **crear demasiadas particiones pequeñas** genera muchos archivos y metadatos; particionar por un identificador de cliente único suele ser mala elección para un dataset grande y disperso.

**Ejemplo:** consultas filtradas habitualmente por mes pueden beneficiarse de una estrategia de particionado o clustering adecuada; si una tabla tiene pocas filas, añadir particiones puede empeorar su mantenimiento. La cardinalidad, el patrón de filtros y el tamaño real del dataset mandan.

## 8.6. Slowly Changing Dimensions: tipos principales

Las **SCD** describen cómo mantener atributos de dimensiones cuando cambian. Ejemplo: un cliente cambia de ciudad.

| Tipo | Estrategia | ¿Conserva valor antiguo? | Caso típico |
|---|---|---|---|
| SCD 0 | Nunca actualizar atributo | Sí, permanece el original | Fecha de nacimiento validada |
| SCD 1 | Sobrescribir valor actual | No en la tabla dimensional | Corregir un error de escritura |
| SCD 2 | Crear nueva fila versionada | Sí | Analizar evolución histórica |
| SCD 3 | Añadir columnas de valor previo | De forma limitada | Comparar valor anterior y actual |

Para análisis histórico fiable sobre clientes, **SCD Tipo 2** suele ser la respuesta frente a sobrescribir siempre los atributos con Tipo 1.

## 8.7. Implementar SCD Tipo 2

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

## 8.8. Tablas temporales e historial

Las tablas temporales registran cambios con un eje temporal. Es útil distinguir **tiempo válido** (cuándo se consideró cierto un atributo en el negocio) y **tiempo de sistema** (cuándo se registró el cambio). En escenarios exigentes se pueden conservar ambos (modelo bitemporal).

```sql
SELECT id_cliente, ciudad
FROM estudio.crm.clientes_historial
WHERE DATE '2026-02-01' >= vigente_desde
  AND (vigente_hasta IS NULL OR DATE '2026-02-01' < vigente_hasta);
```

Este patrón asume intervalos cerrados por inicio y abiertos por fin; es necesario adoptarlo consistentemente. El historial de transacciones Delta facilita consultas de versiones recientes conservadas, pero no reemplaza una dimensión diseñada para duración indefinida.

## 8.9. Granularidad en hechos y dimensiones

El **grano** determina qué representa una fila: una línea de pedido, un pedido o un resumen mensual. Una tabla de hechos almacena medidas y claves de dimensión; una dimensión describe entidades y características. Si se mezclan granos sin cuidado, un join puede multiplicar medidas y falsear las sumas.

Ejemplo: `hechos_ventas` con grano **una línea de pedido** puede relacionarse con `dim_cliente`, `dim_producto` y `dim_fecha`. El total de facturación se calcula sumando importes de las líneas y controlando posibles duplicados.

## 8.10. Managed frente a external

Elegir **managed** significa delegar gestión física y mantenimiento a Unity Catalog, simplificando operaciones. Elegir **external** permite conservar los datos bajo control independiente, integrar fuentes compartidas o satisfacer requisitos específicos de ubicación. En ambos casos los permisos y metadatos se administran desde Unity Catalog si el objeto está registrado allí.

## 8.11. Clustering y optimización

El **clustering** aproxima físicamente registros que se consultan con filtros relacionados. **Liquid clustering** permite una organización flexible y administrada de los archivos Delta en escenarios compatibles. Debe evaluarse frente a estrategias antiguas de particionado físico y optimización por ordenación. La mejora depende de cardinalidad, volumen y patrón de consultas.

El principal error consiste en optimizar columnas que nunca aparecen en filtros o joins importantes. Primero examina planes de consulta, datos escaneados, file pruning y costes de mantenimiento. El diseño del almacenamiento no reemplaza la buena selección de columnas, uniones y granularidad.

## 8.12. Ejercicio y evaluación

Construye una dimensión de clientes que preserve cambios de ciudad y estado comercial, y unos hechos de pedidos de grano «una línea». Diseña su tratamiento CDC, reglas de no solapamiento, claves y consultas «estado en la fecha del pedido». Razona si un cambio de ciudad debe ser Tipo 1 o Tipo 2 según la necesidad empresarial.

---

---

## 🧪 Guía de laboratorio y comprobación práctica

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


---

### 📚 Fuentes y navegación

- [Contenido oficial de Microsoft Learn](https://learn.microsoft.com/en-us/training/modules/design-implement-data-modeling-unity-catalog/)
- [Repositorio docente de laboratorios DP-750T00](https://microsoftlearning.github.io/DP-750T00-Implement-Data-Engineering-Solutions-using-Azure-Databricks/)
- [Guía oficial de competencias del examen DP-750](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-750)

[🏠 Índice](../README.md) · [← Anterior](../07-gobernar-objetos-de-unity-catalog/README.md) · [Siguiente →](../09-ingesta-de-datos/README.md)
