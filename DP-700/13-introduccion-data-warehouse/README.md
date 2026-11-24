# 13 · Get started with data warehouses in Microsoft Fabric

> **DP-700 · Implement a data warehouse.** [Módulo oficial](https://learn.microsoft.com/en-us/training/modules/get-started-data-warehouse/) · [\`index.yml\` y \`includes\`](https://github.com/MicrosoftDocs/learn/tree/main/learn-pr/wwl/get-started-data-warehouse) · [Laboratorio MicrosoftLearning](https://github.com/MicrosoftLearning/mslearn-fabric/blob/main/Instructions/Labs/06-data-warehouse.md). **Revisión:** noviembre de 2026. Laboratorio documentado, no ejecutado.

## Índice

1. [Introducción](#1-introducción)
2. [Understand a data warehouse](#2-understand-a-data-warehouse)
3. [Warehouse in Microsoft Fabric](#3-warehouse-in-microsoft-fabric)
4. [Query and transform data](#4-query-and-transform-data)
5. [Model data](#5-model-data)
6. [Security and monitoring](#6-security-and-monitoring)
7. [Laboratorio oficial](#7-laboratorio-oficial)
8. [Knowledge check y summary](#8-knowledge-check-y-summary)
9. [Fuentes y control](#9-fuentes-y-control)

## 1. Introducción

Una base de datos operacional optimiza el registro y modificación de transacciones del día a día; una solución de **data warehousing** organiza datos integrados de varias fuentes para análisis históricos, comparación temporal y elaboración de informes. La diferencia no está simplemente en que el warehouse contenga «muchos datos», sino en la naturaleza de sus consultas, la estructura de sus modelos y las estrategias de actualización. En un esquema transaccional normalizado, consultar millones de ventas y sus dimensiones puede requerir numerosas uniones. Un modelo dimensional bien diseñado procura ofrecer un contrato sencillo y estable para usuarios analíticos.

**Fabric Warehouse** proporciona almacenamiento analítico relacional gestionado y capacidades T-SQL, integrado con OneLake. Debe diferenciarse del **SQL analytics endpoint de Lakehouse**: ambos admiten consultas SQL sobre determinados datos, pero Warehouse ofrece operaciones de creación y modificación de objetos y datos con un alcance distinto. La selección depende de cómo se pretende escribir, modelar y consultar la información, de las necesidades de transacciones y de las características concretas del motor.

## 2. Understand a data warehouse

### 2.1. Del proceso de ingesta al consumo

Un warehouse comprende ingestión de fuentes, staging, transformación, modelo y consultas. Una carga puede importar registros desde un sistema CRM y combinarse con ventas transaccionales, pero los identificadores de clientes de ambos sistemas pueden no coincidir. Por eso la transformación requiere claves de negocio, correspondencias y reglas explícitas de conformidad. Sin una dimensión de cliente homogénea, un informe puede asignar dos clientes distintos a la misma entidad o combinar clientes diferentes por error.

~~~mermaid
flowchart LR
  A["Operacional / CSV / APIs"] --> B["Ingesta / staging"]
  B --> C["Calidad y claves"]
  C --> D["Hechos y dimensiones"]
  D --> E["Consultas T-SQL / vistas"]
  E --> F["Modelos semánticos / BI"]
~~~

Diagrama de elaboración propia. **Staging** es una zona intermedia para validar fuentes y aplicar transformaciones; no siempre se expone a los consumidores. En producción puede servir para deduplicar, identificar un lote y probar integridad antes de actualizar tablas de destino.

### 2.2. Diseño dimensional

Una **fact table** almacena eventos medibles, como líneas de venta, y claves a las dimensiones. Las **dimension tables** aportan contexto: producto, fecha, cliente, tienda o región. El esquema en estrella conecta una tabla de hechos central con sus dimensiones. Las claves deben reflejar la granularidad: por ejemplo, una fila de \`FactSales\` por línea de ticket puede contener \`Quantity\`, \`NetAmount\` y referencias a DateKey, ProductKey y CustomerKey.

Una **business key** procede de la aplicación y representa una entidad dentro de su dominio; una **surrogate key** es una clave técnica gestionada en el warehouse para identificar filas de dimensión, especialmente relevante cuando se registran cambios históricos. No sustituir una por otra sin comprender el modelo de historificación.

### 2.3. Granularidad y relaciones

Antes de diseñar una medida de ventas, especificar si cada fila representa un pedido, línea de pedido o agregado diario. Si se unen hechos con dimensiones que contienen varias filas para una clave natural sin filtrar la versión aplicable, el resultado puede multiplicar importes. Deben validarse claves únicas, relaciones y reglas de vigencia. Los modelos dimensionales no evitan automáticamente errores de negocio; hacen más explícitos los contratos.

**Fuente:** [Understand a data warehouse](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/get-started-data-warehouse/includes/2-understand-data-warehouse.md).

## 3. Warehouse in Microsoft Fabric

### 3.1. Arquitectura y capacidades

Warehouse es un elemento del Workspace, con datos en formatos abiertos sobre OneLake y motor relacional T-SQL gestionado por Fabric. La documentación oficial describe soporte de transacciones ACID y operaciones T-SQL DDL/DML admitidas, incluidas creación de tablas, vistas, modificaciones y operaciones \`MERGE\`. No debe inferirse que soporta sin restricciones cualquier característica histórica de SQL Server: consultar la matriz de compatibilidad para índices, constraints, procedimientos o tipos específicos.

Se pueden crear tablas y cargar datos desde recursos externos, usar staging, crear vistas y efectuar transformaciones SQL. La estrategia de ingesta debe preservar procedencia y evitar cargas parciales. Una tabla de staging permite recibir un lote, comprobar su esquema y realizar las modificaciones definitivas en un orden controlado.

~~~sql
CREATE TABLE dbo.DimProduct (
    ProductKey INT NOT NULL,
    ProductCode VARCHAR(30) NOT NULL,
    ProductName VARCHAR(200) NOT NULL
);
~~~

Este ejemplo es propio y necesita Warehouse adecuado y permisos de DDL. No se incluyen restricciones de unicidad por suposiciones: antes de añadirlas, comprobar qué constraints y su enforcement admite Fabric Warehouse en la versión consultada.

### 3.2. Warehouse vs Lakehouse SQL analytics endpoint

| Característica | Warehouse | SQL analytics endpoint del Lakehouse |
| --- | --- | --- |
| Enfoque | Almacén analítico relacional | Consulta SQL de tablas compatibles del Lakehouse |
| Escritura T-SQL | DDL y DML soportados por Warehouse | No equivale a motor de escritura T-SQL general |
| Datos | Almacenamiento gestionado integrado con OneLake | Tablas Lakehouse/Delta |
| Integración típica | ETL/ELT SQL y modelos dimensionales | Explorar y consultar resultados producidos en Lakehouse |
| Herramientas | Editor SQL, clientes compatibles, BI | Editor SQL y clientes compatibles de consulta |

### 3.3. Table clones

Las **table clones** permiten reutilizar una representación de una tabla según las capacidades del servicio para pruebas y análisis. Hay que entender la semántica de almacenamiento y aislamiento antes de asumir que el clon equivale a una copia independiente completa o que heredará todas las políticas de retención. Un clon de pruebas nunca sustituye una política de backup o recuperación de producción.

**Fuente:** [Warehouse in Fabric](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/get-started-data-warehouse/includes/3-understand-data-warehouse-fabric.md).

## 4. Query and transform data

El editor SQL de Fabric permite escribir consultas T-SQL, inspeccionar resultados y gestionar scripts. El **Visual query editor** ofrece una alternativa no-code para seleccionar y relacionar datos; su uso no garantiza que se hayan elegido uniones o filtros óptimos. Deben comprobarse las consultas generadas igual que se revisa código escrito a mano.

~~~sql
SELECT p.ProductName,
       SUM(f.Quantity) AS Units,
       SUM(f.NetAmount) AS Revenue
FROM dbo.FactSales AS f
JOIN dbo.DimProduct AS p ON p.ProductKey = f.ProductKey
GROUP BY p.ProductName
ORDER BY Revenue DESC;
~~~

Este ejemplo presupone un esquema estrella creado previamente. Una consulta que agrupa \`ProductName\` podría combinar productos distintos con el mismo nombre; es más seguro incluir \`ProductKey\` cuando se desea una métrica por identidad de producto.

Las **views** encapsulan consultas y pueden ofrecer contratos de consumo estables, mientras los **stored procedures** pueden organizar operaciones T-SQL repetibles. No confundir una vista ordinaria con datos materializados: consultar una vista suele implicar ejecutar su definición, y la optimización depende del motor.

**Fuente:** [Query and transform data](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/get-started-data-warehouse/includes/4-query-transform-data.md).

## 5. Model data

Un warehouse con docenas de tablas técnicas no es automáticamente comprensible para un equipo de negocio. El modelo debe documentar hechos, dimensiones, relaciones, medidas y valores desconocidos. En un esquema estrella, la dimensión de fecha permite comparar períodos incluso si hay días sin ventas, mientras las dimensiones conformadas favorecen análisis cruzados entre distintos hechos.

Para un modelo semántico de Power BI, diseñar relaciones y medidas compatibles con la granularidad de los hechos. Es importante controlar el número de vías de filtro entre tablas y evitar relaciones ambiguas o muchos-a-muchos innecesarias. Una medida calculada incorrectamente puede producir cifras plausibles pero erróneas, especialmente cuando se mezclan niveles de agregación.

La documentación oficial aborda vistas, medidas y modelos semánticos para presentar la información a los consumidores. **Fuente:** [Model data](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/get-started-data-warehouse/includes/5-model-data.md).

## 6. Security and monitoring

### 6.1. Seguridad por capas

El acceso a un Warehouse combina roles de Workspace, permisos de elemento y permisos SQL granulares. Una identidad con permiso para entrar al Workspace no necesita tener automáticamente privilegios para modificar todas las tablas. Diseñar el mínimo privilegio; por ejemplo, los analistas pueden utilizar vistas de solo lectura mientras las identidades de ETL escriben en staging y las tablas publicadas.

### 6.2. Monitorización

Las **Dynamic Management Views (DMVs)** ofrecen información de sesiones, consultas y ejecución según la documentación de Fabric, mientras **Query insights** permite examinar patrones e historial de consultas. La monitorización debe buscar duraciones excesivas, fallos y consumo inesperado, diferenciando operaciones de ingestión y consultas analíticas.

Un informe de rendimiento no debe atribuir un bloqueo o una consulta lenta sin revisar los datos de ejecución correspondientes. La capacidad compartida puede influir en la experiencia y requiere correlacionar métricas.

**Fuente:** [Security and monitoring](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/get-started-data-warehouse/includes/6-security-monitor.md).

## 7. Laboratorio oficial

**Fuente:** [Analyze data in a data warehouse](https://github.com/MicrosoftLearning/mslearn-fabric/blob/main/Instructions/Labs/06-data-warehouse.md). Duración oficial aproximada: **30 minutos**. Ejercicio catalogado para DP-700. **Sin ejecución real**.

### 7.1. Objetivo

Crear Workspace, Warehouse y tablas, insertar datos de muestra, realizar consultas con joins y agregaciones, construir una vista, utilizar visual query y, opcionalmente, modelar relaciones. La práctica permite distinguir entre preparar datos y exponer un modelo para consumidores.

### 7.2. Workspace y Warehouse

Entrar en [Fabric](https://app.fabric.microsoft.com/), crear un Workspace de prueba con capacidad compatible y añadir un Warehouse. Comprobar el nombre y esquema del Warehouse antes de iniciar SQL. No reutilizar una tabla productiva para un ejercicio que inserta datos de prueba.

### 7.3. Crear y cargar tablas

Ejecutar los scripts de creación e inserción del laboratorio oficial para las tablas de hechos y dimensiones. Revisar que los tipos se corresponden con las columnas e identificar la clave primaria lógica de cada dimensión y la granularidad del hecho. No añadir columnas o cambiar nombres del dataset original sin modificar también las consultas posteriores.

**Verificación:** consultar recuentos por tabla y examinar que las claves relacionadas tienen correspondencias. Los recuentos exactos solo se consignarán tras una ejecución; no se inventan.

### 7.4. Consultar y crear una view

Usar el editor SQL y ejecutar los joins y agregaciones del tutorial. Después crear una vista para reutilizar una lógica de consulta. Revisar qué columnas y filtros expone. Si se utiliza la vista como mecanismo de seguridad, no asumir que por sí misma protege el acceso directo a las tablas originales: revisar permisos de cada identidad.

### 7.5. Visual query y modelo opcional

Abrir Visual query editor, reproducir una consulta sencilla y comparar sus resultados con T-SQL. En la parte opcional, crear o modificar el modelo de datos, revisar cardinalidades y relaciones. No declarar completada esta parte si no se ejecuta.

### 7.6. Diagnóstico y limpieza

| Incidencia | Comprobación |
|---|---|
| No hay opción Warehouse | Capacidad y permisos |
| \`Invalid object name\` | Schema, tabla y Warehouse conectado |
| Join multiplica filas | Clave no única en dimensión |
| La vista devuelve datos antiguos | Revisar consulta, tablas y momento de carga |
| Editor visual no permite una transformación | Revisar las capacidades y escribir T-SQL si procede |
| Consulta lenta | Query insights, filtros y tamaño de joins |

Al terminar, eliminar el Workspace exclusivo o los objetos de prueba, nunca recursos compartidos. Revisar conexiones y costes de capacidad.

## 8. Knowledge check y summary

Preguntas propias: ¿Qué es la granularidad de una fact table? ¿Por qué se utilizan dimensiones? ¿Qué diferencia hay entre clave de negocio y surrogate key? ¿Warehouse y SQL endpoint del Lakehouse admiten la misma escritura? ¿Qué función tiene staging? ¿Cómo identificar una duplicación causada por joins? ¿Por qué una vista no sustituye la seguridad SQL? ¿Qué información proporcionan DMVs?

**Resumen técnico:** el Warehouse es una solución relacional analítica gestionada sobre Fabric; su calidad depende del modelo, las cargas, los permisos y la monitorización. La consulta SQL es solo la última etapa de una arquitectura de datos coherente.

## 9. Fuentes y control

- [\`index.yml\`, nueve unidades](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/get-started-data-warehouse/index.yml).
- [Diseño de warehouse](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/get-started-data-warehouse/includes/2-understand-data-warehouse.md).
- [Warehouse Fabric](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/get-started-data-warehouse/includes/3-understand-data-warehouse-fabric.md).
- [Consultas](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/get-started-data-warehouse/includes/4-query-transform-data.md).
- [Modelado](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/get-started-data-warehouse/includes/5-model-data.md).
- [Seguridad y monitorización](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/get-started-data-warehouse/includes/6-security-monitor.md).
- [Laboratorio DP-700](https://github.com/MicrosoftLearning/mslearn-fabric/blob/main/Instructions/Labs/06-data-warehouse.md).

**Cobertura:** unidades conceptuales, ejercicio, evaluación y síntesis desarrollados. Queda pendiente comprobar todos los pasos en un tenant real y contrastar cualquier actualización del producto.