[🏠 Índice](../README.md) · [← Anterior](../04-seleccion-y-configuracion-de-computo/README.md) · [Siguiente →](../06-proteger-objetos-de-unity-catalog/README.md)

> **Estado de estudio:** ✅ Visto · **Unidades oficiales:** 12 · [Microsoft Learn](https://learn.microsoft.com/en-us/training/modules/create-and-organize-objects-in-unity-catalog/)  
> **Versión de apuntes:** 15-11-2026 · Contenido técnico redactado de forma original a partir del itinerario de Microsoft.

# 5. Crear y organizar objetos en Unity Catalog

**Referencia Microsoft Learn:** [Crear y organizar objetos en Unity Catalog](https://learn.microsoft.com/en-us/training/modules/create-and-organize-objects-in-unity-catalog/) · **12 unidades oficiales**.

## 5.1. Diseño del espacio de nombres

Unity Catalog organiza objetos mediante `catalog.schema.object`. Un catálogo puede representar un área de negocio (`finanzas`), un entorno (`dev`, `prod`) o una categoría regulatoria. Cada esquema agrupa objetos relacionados, por ejemplo `prod.crm.clientes` y `prod.crm.interacciones`.

Una convención sencilla y consistente ayuda a evitar errores: minúsculas, nombres descriptivos, separar dominios por catálogo o esquema y documentar abreviaturas. Los identificadores son parte de los contratos de datos; renombrarlos sin evaluar consumidores puede romper informes o procesos.

## 5.2. Crear catálogos

Los catálogos requieren permisos apropiados; pueden tener propietario, comentarios, reglas de acceso, asignación a workspaces y ubicaciones de almacenamiento administrado.

```sql
CREATE CATALOG IF NOT EXISTS estudio;
COMMENT ON CATALOG estudio IS 'Entorno didáctico de ingeniería de datos';
```

En un proyecto productivo, planifica previamente dónde residirán físicamente las tablas managed y quién administra el catálogo. Es recomendable aislar los datos críticos de los esquemas experimentales.

## 5.3. Crear esquemas

```sql
CREATE SCHEMA IF NOT EXISTS estudio.ventas
COMMENT 'Datos y productos analíticos del dominio comercial';

USE CATALOG estudio;
USE SCHEMA ventas;
```

El esquema permite aplicar privilegios sobre conjuntos de objetos y, si procede, una ubicación administrada más específica. `USE CATALOG` y `USE SCHEMA` establecen el contexto de nombres, pero no conceden permisos por sí mismos.

## 5.4. Tablas administradas, externas y vistas

Las tablas **managed** son preferibles cuando Databricks debe administrar los archivos y su ciclo de vida. Las **external** son útiles para datos existentes, compartidos o gobernados también desde fuera. Las **views** almacenan una consulta lógica y normalmente no duplican físicamente los datos de resultado; las vistas materializadas conservan resultados precalculados bajo su mecanismo de actualización.

```sql
CREATE TABLE IF NOT EXISTS estudio.ventas.pedidos (
  id_pedido BIGINT,
  id_cliente BIGINT,
  importe DECIMAL(12,2),
  fecha DATE
) USING DELTA;

CREATE OR REPLACE VIEW estudio.ventas.pedidos_validos AS
SELECT * FROM estudio.ventas.pedidos
WHERE importe >= 0 AND id_cliente IS NOT NULL;
```

La vista puede servir como contrato de consumo, aunque para seguridad sensible hay que configurar privilegios, identidad de ejecución y políticas apropiadas, no solo ocultar columnas en un `SELECT`.

## 5.5. Volúmenes

Un volumen almacena y gobierna **archivos que no necesitan representarse como filas de una tabla**. Puede ser managed o external. Su ubicación lógica se representa como `/Volumes/<catalogo>/<esquema>/<volumen>/archivo`.

```sql
CREATE VOLUME IF NOT EXISTS estudio.ventas.entradas;
```

Casos de uso: PDFs, CSV sin ingerir, imágenes, modelos auxiliares y archivos temporales gobernados. Para datos que ya van a consultarse regularmente con SQL, lo habitual es registrar una tabla.

## 5.6. DDL: administrar definiciones

El lenguaje de definición de datos (DDL) comprende `CREATE`, `ALTER`, `DROP` y otras operaciones sobre objetos. Afecta a **estructura y metadatos**, mientras que operaciones como `INSERT`, `UPDATE` y `DELETE` cambian filas. En Unity Catalog también se registran funciones y otros objetos gobernables; la disponibilidad de procedimientos almacenados depende de las capacidades de la plataforma y del motor.

```sql
ALTER TABLE estudio.ventas.pedidos
ADD COLUMN canal STRING;

DESCRIBE TABLE EXTENDED estudio.ventas.pedidos;
```

Antes de modificar una tabla productiva, considera contratos de esquema, compatibilidad de consumidores, histórico y posibilidad de reversión.

## 5.7. Catálogos foráneos y federation

Un **foreign catalog** representa objetos de un origen externo integrado mediante funciones de federación. Permite consultar datos fuera del almacenamiento administrado de Databricks sin tener que cargarlos siempre primero. Esto beneficia exploración o integración, pero no elimina latencia, coste de red, límites del origen ni restricciones del conector.

La conexión necesita identidad y permisos en ambos extremos. Su utilidad se valora frente a copiar los datos a Delta y ejecutar consultas locales: si se necesitan transformaciones intensivas repetidas, la ingesta puede rendir mejor.

## 5.8. AI/BI Genie: instrucciones y semántica

Genie traduce preguntas de negocio a análisis basados en datos autorizados. Configurar un espacio útil exige seleccionar tablas relevantes, proporcionar definiciones consistentes, describir unidades de medida, establecer reglas de negocio y revisar consultas de ejemplo. Una frase ambigua como «ventas del mes» necesita precisar si significa facturación bruta, neta, pedidos cobrados o pedidos emitidos.

Las instrucciones orientan la interpretación, pero **no sustituyen** los privilegios de Unity Catalog ni garantizan exactitud de cualquier respuesta generada.

## 5.9. Ejercicio y repaso

Diseña `estudio.ventas` con dos tablas (`clientes`, `pedidos`), una vista de pedidos válidos y un volumen de archivos de entrada. Define propietarios, comentarios y permisos mínimos. Explica por qué crearías una tabla external solo cuando el ciclo de vida del dato se administre fuera del sistema.

**Preguntas:** ¿cuáles son los tres niveles de nombres? ¿Qué diferencia una view de una materialized view? ¿Cuándo conviene un volumen? ¿Para qué sirve un foreign catalog?

---

---


## Desarrollo en profundidad: diseñar la jerarquía de gobierno

### El identificador de tres niveles no es una convención estética

En Unity Catalog una tabla se identifica mediante `catalogo.esquema.objeto`. Este espacio de nombres permite separar áreas, equipos y entornos. Un catálogo suele representar una frontera de organización o gobierno; dentro de él, los esquemas agrupan objetos relacionados. La elección exacta es organizativa: una empresa puede usar catálogos por entorno y dominio, o una estrategia diferente conforme a políticas y necesidades de uso compartido.

Por ejemplo, `produccion.finanzas.movimientos` comunica que el objeto pertenece a un catálogo productivo y un esquema financiero. En cambio, una tabla temporal de laboratorio debería ubicarse en un catálogo de desarrollo para evitar que un borrado accidental afecte a datos reales. Los nombres deben ser estables, comprensibles y resistentes a reestructuraciones frecuentes.

### Tablas, vistas, vistas materializadas y volúmenes

Una tabla almacena registros persistentes; una vista expresa una consulta lógica cuya resolución puede depender de las tablas originales; una vista materializada conserva resultados administrados que deben actualizarse según las capacidades del servicio; un volumen gestiona archivos que no se consultan necesariamente como filas y columnas. La elección responde al modo de consumo: no crees una tabla para guardar un PDF ni un volumen para sustituir un modelo analítico.

```sql
CREATE SCHEMA IF NOT EXISTS estudio.comercial;

CREATE TABLE IF NOT EXISTS estudio.comercial.clientes (
  id_cliente BIGINT,
  nombre STRING,
  alta DATE
) USING DELTA;

CREATE OR REPLACE VIEW estudio.comercial.clientes_recientes AS
SELECT id_cliente, nombre
FROM estudio.comercial.clientes
WHERE alta >= DATE '2026-01-01';
```

Antes de ejecutar las sentencias debes disponer de permisos adecuados sobre el catálogo y el esquema. La tabla del ejemplo no contiene claves primarias físicas que impidan duplicados automáticamente: la integridad se gestiona con reglas y controles compatibles con el motor.

### Managed y external: quién controla el ciclo de vida

Una tabla administrada simplifica la responsabilidad sobre ubicación y mantenimiento; una tabla externa referencia datos ubicados fuera de ese control directo. En ambos casos puede existir gobierno mediante Unity Catalog, pero eliminar un objeto de metadatos no tiene necesariamente el mismo impacto físico. Antes de usar `DROP TABLE`, consulta tipo de tabla, ubicación y política de retención. Cuando intervienen ubicaciones externas, credenciales de almacenamiento y grants, la autorización debe mantenerse coherente entre nube y metadatos.

### Convenciones, descubrimiento y contratos

Documenta propietario funcional, frecuencia de actualización, clasificación de datos, grano, fuente y consumidores. Un catálogo que permite localizar tablas pero no explica qué significan sus columnas sigue siendo difícil de usar. Las etiquetas y comentarios ayudan a la búsqueda, pero no sustituyen permisos, máscaras y validación.

**Caso de diseño:** tienes departamentos de operaciones y finanzas que necesitan compartir una dimensión de calendario sin compartir los movimientos bancarios. Propón un catálogo común para activos de referencia y catálogos o esquemas específicos para datos sensibles. Explica qué identidades necesitan `USE CATALOG`, `USE SCHEMA` y `SELECT`, y cómo un nuevo analista descubriría los datos autorizados sin ver los objetos restringidos.

---

## 🧪 Guía de laboratorio y comprobación práctica

**Ejercicio oficial:** [Abrir las instrucciones de MicrosoftLearning](https://microsoftlearning.github.io/DP-750T00-Implement-Data-Engineering-Solutions-using-Azure-Databricks/Instructions/Labs/03-create-and-organize-objects-in-unity-catalog.html)  
**Notebook oficial:** [Importar o consultar notebook `.ipynb`](https://raw.githubusercontent.com/MicrosoftLearning/DP-750T00-Implement-Data-Engineering-Solutions-using-Azure-Databricks/refs/heads/main/Allfiles/03-create-and-organize-objects-in-unity-catalog.ipynb)  
**Naturaleza de esta sección:** itinerario explicado con redacción propia y comprobaciones de aprendizaje. Las celdas exactas y el material de origen están en el notebook oficial enlazado; no se reproducen íntegramente aquí.


### Caso práctico: Lakeside University — modelar el espacio de nombres

**Escenario:** una universidad gestiona alumnos, asignaturas y matrículas. El objetivo es implementar una estructura gobernada por Unity Catalog que siga la arquitectura Medallion.

**Guía de trabajo:**

1. Comprueba que tienes **CREATE CATALOG** y un metastore funcional. Importa el notebook de Microsoft e identifica las tablas y funciones que se van a crear antes de ejecutarlo.
2. Crea un catálogo de desarrollo llamado `edu_dev`. Dentro prepara los esquemas `bronze`, `silver` y `gold`; explica por qué separar los datos por grado de refinamiento.
3. En la capa de trabajo estructurado diseña tablas managed para estudiantes, asignaturas y matrículas. Define tipos, comentarios descriptivos y claves lógicas. **Las restricciones PK/FK de Databricks son informativas y no sustituyen comprobaciones de integridad del ETL.**
4. Construye una vista normal para consultar alumnos y matrículas, y una vista materializada orientada a estadísticas de departamentos. Diferencia entre almacenamiento de resultados y ejecución de una consulta virtual.
5. Registra un **managed volume** para archivos sin formato tabular y deposita un CSV de prueba. Distingue el fichero físico de su representación como tabla.
6. Crea una función SQL escalar reutilizable que clasifique calificaciones mediante rangos. Añade una columna con `ALTER TABLE` y documenta una columna o tabla mediante comentarios y tags cuando existan permisos.
7. **Actividad opcional de UI:** crea un espacio AI/BI Genie con tablas `edu_dev.silver.students`, `courses`, `enrollments`, la vista de matrículas y la vista de estadísticas Gold. Pregunta por el departamento con mejor promedio y revisa el SQL generado antes de aceptar su conclusión.

**Validaciones:** `SHOW CATALOGS`, `SHOW SCHEMAS IN edu_dev`, `DESCRIBE TABLE`, `SHOW TABLES`, consulta de la vista y prueba de la función. Comprueba la diferencia entre privilegios de creación y de consulta.

**Precaución:** una limpieza con `DROP CATALOG ... CASCADE` elimina objetos; ejecútala solo sobre un catálogo de laboratorio que hayas identificado inequívocamente.


---

### 📚 Fuentes y navegación

- [Contenido oficial de Microsoft Learn](https://learn.microsoft.com/en-us/training/modules/create-and-organize-objects-in-unity-catalog/)
- [Repositorio docente de laboratorios DP-750T00](https://microsoftlearning.github.io/DP-750T00-Implement-Data-Engineering-Solutions-using-Azure-Databricks/)
- [Guía oficial de competencias del examen DP-750](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-750)

[🏠 Índice](../README.md) · [← Anterior](../04-seleccion-y-configuracion-de-computo/README.md) · [Siguiente →](../06-proteger-objetos-de-unity-catalog/README.md)
