# 15 · Query a data warehouse in Microsoft Fabric

> **DP-700 · Warehouse.** [Módulo Microsoft Learn](https://learn.microsoft.com/en-us/training/modules/query-data-warehouse-microsoft-fabric/) · [origen MicrosoftDocs (\`wwl-data-ai\`)](https://github.com/MicrosoftDocs/learn/tree/main/learn-pr/wwl-data-ai/query-data-warehouse-microsoft-fabric) · [laboratorio oficial DP-700](https://github.com/MicrosoftLearning/mslearn-fabric/blob/main/Instructions/Labs/06b-data-warehouse-query.md). **Revisión:** noviembre de 2026. Sin ejecución real.

## Índice

1. [Introducción](#1-introducción)
2. [T-SQL y consultas analíticas](#2-t-sql-y-consultas-analíticas)
3. [SQL query editor](#3-sql-query-editor)
4. [Visual query editor](#4-visual-query-editor)
5. [SSMS y clientes externos](#5-ssms-y-clientes-externos)
6. [Laboratorio oficial](#6-laboratorio-oficial)
7. [Knowledge check y summary](#7-knowledge-check-y-summary)
8. [Fuentes](#8-fuentes)

## 1. Introducción

Consultar un Warehouse exige conocer tanto el lenguaje SQL como el significado de sus tablas. Una consulta puede ser sintácticamente correcta y producir una cifra incorrecta si une dimensiones en una cardinalidad inadecuada, mezcla monedas o no distingue ventas de devoluciones. El módulo explica cómo extraer y agregar datos T-SQL y cómo utilizar las herramientas de consulta de Fabric: editor SQL, editor visual y SQL Server Management Studio (SSMS).

El objetivo de un Data Engineer no es únicamente escribir un SELECT: debe producir resultados **reproducibles**, **explicables** y coherentes con el modelo dimensional. Una tabla de hechos puede contener millones de líneas de venta; la consulta debe seleccionar claves, métricas, condiciones y nivel de agregación que respondan a la pregunta empresarial.

## 2. T-SQL y consultas analíticas

### 2.1. JOIN y agregación por dimensiones

En un modelo estrella, la tabla de hechos contiene referencias a las dimensiones. Para obtener ventas por categoría, puede unirse FactSales con DimProduct y agregar importes.

~~~sql
SELECT p.Category,
       SUM(f.Quantity * f.UnitPrice) AS Revenue,
       COUNT(*) AS SalesLines
FROM dbo.FactSales AS f
INNER JOIN dbo.DimProduct AS p
    ON f.ProductKey = p.ProductKey
GROUP BY p.Category
ORDER BY Revenue DESC;
~~~

*Ejemplo propio sobre tablas hipotéticas.* No se ha ejecutado. Es imprescindible saber si el precio incluye impuestos y si una fila representa una línea de venta. Si el join multiplica filas porque ProductKey no es único en la dimensión, el importe quedará inflado. Para depurar, comparar número de filas antes y después del join y comprobar claves duplicadas en las dimensiones.

### 2.2. Star schema frente a snowflake schema

El **star schema** conecta hechos directamente con dimensiones relativamente desnormalizadas. El **snowflake schema** puede normalizar algunas dimensiones y dividir jerarquías en tablas. Esto añade joins y puede afectar a la complejidad de consultas. Ninguno es automáticamente correcto para todos los contextos: las dimensiones conformadas y el contrato de consumo importan más que el nombre de la forma.

### 2.3. Ranking y funciones ventana

Las **window functions** calculan valores sobre grupos lógicos sin reducir necesariamente el número de filas como hace GROUP BY.

~~~sql
WITH TotalByProduct AS (
    SELECT f.ProductKey, SUM(f.Quantity * f.UnitPrice) AS Revenue
    FROM dbo.FactSales AS f
    GROUP BY f.ProductKey
)
SELECT ProductKey,
       Revenue,
       ROW_NUMBER() OVER (ORDER BY Revenue DESC, ProductKey) AS Position
FROM TotalByProduct;
~~~

El ejemplo obtiene un ranking reproducible usando ProductKey como criterio adicional para empates. Si se quieren posiciones compartidas, se puede estudiar RANK o DENSE_RANK; la elección depende del requisito de negocio.

### 2.4. Recuentos aproximados y exactos

La documentación oficial también trata la obtención de **approximate counts**. Una aproximación puede reducir coste cuando se trabaja con conjuntos masivos y tolerancia a error, pero no equivale a una cuenta exacta para facturación o cumplimiento. Leer el contrato de la función específica y utilizar un recuento exacto cuando el resultado sea normativamente relevante.

**Fuente:** [Query data](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl-data-ai/query-data-warehouse-microsoft-fabric/includes/2-query-data.md).

## 3. SQL query editor

El editor SQL integrado conecta con el Warehouse desde el portal de Fabric. Permite escribir scripts T-SQL, ejecutarlos, revisar resultados y almacenar consultas. Las ayudas de IntelliSense y validación mejoran productividad, pero no sustituyen la revisión de permisos o la semántica del dato.

### 3.1. Del SELECT a una view

Es útil encapsular una consulta estable en una vista para que varios consumidores utilicen la misma lógica.

~~~sql
CREATE VIEW dbo.vw_SalesByProduct AS
SELECT f.ProductKey,
       SUM(f.Quantity * f.UnitPrice) AS Revenue
FROM dbo.FactSales AS f
GROUP BY f.ProductKey;
~~~

*Ejemplo propio.* Deben existir las tablas y los permisos para crear vistas. No suponer que toda vista materializa resultados ni que crearla restringe automáticamente el acceso directo a las tablas subyacentes.

### 3.2. Resultados y rendimiento

Guardar las consultas con nombre, propósito y filtros documentados. Cuando un conjunto es grande, limitar la exploración inicial y no usar \`SELECT *\` indiscriminadamente. Para revisar rendimiento, analizar duración de consulta y fuentes de carga del Warehouse, no atribuir toda latencia a T-SQL.

### 3.3. Copilot en el editor

El editor puede integrar funciones Copilot según disponibilidad y configuración del tenant. Las respuestas de IA son sugerencias, no garantía de SQL seguro o correcto. Un código generado debe comprobarse ante el esquema efectivo y contra las políticas de acceso y calidad.

**Fuente:** [SQL query editor](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl-data-ai/query-data-warehouse-microsoft-fabric/includes/3-use-sql-query-editor.md).

## 4. Visual query editor

El **Visual query editor** permite incorporar tablas al canvas y configurar uniones, campos y filtros sin escribir T-SQL manualmente. La interfaz puede generar la consulta SQL correspondiente y ayuda a usuarios familiarizados con modelado visual. Sin embargo, elegir columnas gráficamente no evita los riesgos de un join muchos-a-muchos o de un filtro incorrecto.

### 4.1. Cómo validar el SQL generado

Tras crear una consulta visual, revisar la sentencia generada: tipos de join, condición ON, filtros WHERE, GROUP BY y ordenación. Comparar un conjunto pequeño y conocido de datos con una consulta SQL manual permite detectar discrepancias. Si la consulta es lenta, no suponer que el editor visual o el SQL son diferentes motores: lo relevante es el plan que ejecuta Warehouse.

**Fuente:** [Visual query editor](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl-data-ai/query-data-warehouse-microsoft-fabric/includes/4-explore-visual-query-editor.md).

## 5. SSMS y clientes externos

**SQL Server Management Studio** puede conectarse a un Warehouse Fabric utilizando el endpoint y la autenticación admitida, incluyendo Microsoft Entra ID. La conexión exige conocer la dirección del SQL endpoint real, la identidad autorizada y el catálogo correspondiente. No emplear credenciales personales incrustadas en scripts distribuidos ni confundir el endpoint de un Lakehouse con el de un Warehouse.

### 5.1. Procedimiento conceptual

Desde el Workspace, abrir las propiedades del Warehouse, copiar su cadena/endpoint de conexión SQL y configurar en SSMS el tipo de autenticación Microsoft Entra apropiado. Iniciar sesión con una identidad autorizada y comprobar que se puede consultar una tabla o vista. Para clientes de equipo, documentar los mínimos permisos y controlar quién puede ejecutar DDL o DML. La disponibilidad de un endpoint no implica acceso universal.

**Fuente:** [Client tools](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl-data-ai/query-data-warehouse-microsoft-fabric/includes/5-use-client-tools.md).

## 6. Laboratorio oficial

**Fuente:** [Query a data warehouse in Microsoft Fabric](https://github.com/MicrosoftLearning/mslearn-fabric/blob/main/Instructions/Labs/06b-data-warehouse-query.md). Duración orientativa oficial: 30 minutos. Asociado a DP-700. Se ha consultado la estructura de la práctica, no se ha ejecutado.

### 6.1. Objetivo y requisitos

El ejercicio crea Workspace y un Warehouse con datos de ejemplo. Después realiza consultas, valida coherencia identificando anomalías y crea una vista para un escenario de informe. Se requieren capacidad Fabric compatible y permisos de creación y lectura de los elementos.

### 6.2. Crear entorno y cargar muestra

Entrar en [Fabric](https://app.fabric.microsoft.com/) y crear Workspace separado. Crear el Warehouse de muestra siguiendo la guía original y verificar la existencia de tablas con datos. Anotar nombres reales antes de copiar cualquier consulta. No publicar una cifra o recuento atribuido a la muestra sin haberla consultado.

### 6.3. Consultas, joins y agregados

Ejecutar los SELECT y JOIN de la práctica. Revisar que cada filtro hace referencia a una columna real y que los importes agregados se interpretan según la definición de negocio. Utilizar una consulta independiente para buscar claves sin correspondencia o filas que incumplan el esquema previsto.

### 6.4. Verificar data consistency

La práctica incluye una sección de detección y tratamiento de anomalías. La calidad no se limita a comprobar que el motor no devuelve excepciones. Deben revisarse valores nulos, claves duplicadas y registros que alteran los agregados. Anotar hallazgos y su impacto antes de modificar los datos.

### 6.5. Guardar como vista

Crear una vista desde una consulta verificada y comprobar que devuelve el resultado esperado. Confirmar su nombre y permisos para que pueda localizarse desde el catálogo. Una vista puede facilitar el consumo, pero debe mantenerse cuando evolucionen tablas subyacentes.

### 6.6. Diagnóstico y limpieza

| Fallo | Diagnóstico |
|---|---|
| Error de autorización | Revisar roles, permisos de ítem y SQL |
| Tabla no encontrada | Verificar nombre cualificado y Warehouse conectado |
| Agregados duplicados | Comprobar claves de join y granularidad |
| Vista falla tras cambio de esquema | Revisar dependencias y columnas |
| SSMS no conecta | Revisar endpoint, red y Entra authentication |
| Editor visual produce filas distintas | Comparar SQL generado, joins y filtros |

Borrar exclusivamente el Workspace de práctica o los elementos propios y revisar capacidad y conexiones.

## 7. Knowledge check y summary

Preguntas propias: ¿Qué papel tiene el JOIN de una dimensión? ¿Por qué es importante la granularidad? ¿Qué diferencia hay entre GROUP BY y una window function? ¿Por qué una vista no materializa necesariamente? ¿Cómo se interpreta un approximate count? ¿Qué debe revisarse en una consulta visual? ¿Cómo se autentica SSMS? ¿Qué implica el mínimo privilegio?

**Summary:** consultar Warehouse exige dominar T-SQL y el modelo dimensional, además de saber usar interfaces SQL y visuales y clientes externos de forma segura.

## 8. Fuentes

- [Index oficial, ocho unidades](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl-data-ai/query-data-warehouse-microsoft-fabric/index.yml).
- [T-SQL analítico](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl-data-ai/query-data-warehouse-microsoft-fabric/includes/2-query-data.md).
- [SQL editor](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl-data-ai/query-data-warehouse-microsoft-fabric/includes/3-use-sql-query-editor.md).
- [Visual query editor](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl-data-ai/query-data-warehouse-microsoft-fabric/includes/4-explore-visual-query-editor.md).
- [SSMS](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl-data-ai/query-data-warehouse-microsoft-fabric/includes/5-use-client-tools.md).
- [Laboratorio DP-700](https://github.com/MicrosoftLearning/mslearn-fabric/blob/main/Instructions/Labs/06b-data-warehouse-query.md).

**Control:** explicación de las ocho unidades, laboratorio y síntesis desarrollados; no se ha realizado una ejecución real.