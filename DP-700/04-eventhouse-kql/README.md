# 04 · Work with real-time data in an Eventhouse in Microsoft Fabric

> **DP-700 · Ruta Ingest data with Microsoft Fabric / Real-Time Intelligence.** [Módulo oficial](https://learn.microsoft.com/en-us/training/modules/query-data-kql-database-microsoft-fabric/) · [index.yml](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/query-data-kql-database-microsoft-fabric/index.yml) · [Laboratorio complementario oficial de MicrosoftLearning](https://github.com/MicrosoftLearning/mslearn-fabric/blob/main/Instructions/Labs/12-query-data-in-kql-database.md). **Fecha:** noviembre de 2026. **Laboratorio:** procedimiento documentado, no ejecutado.

## Índice

1. [Introducción](#1-introducción)
2. [Get started with KQL queries](#2-get-started-with-kql-queries)
3. [KQL best practices](#3-kql-best-practices)
4. [Advanced features](#4-advanced-features)
5. [Laboratorio oficial complementario](#5-laboratorio-oficial-complementario)
6. [Knowledge check y summary](#6-knowledge-check-y-summary)
7. [Fuentes y alcance](#7-fuentes-y-alcance)

## 1. Introducción

Una plataforma transaccional de negocio puede producir eventos continuos: clics, telemetría industrial, mensajes de aplicación, operaciones de un servicio o posiciones de dispositivos. Cuando el objetivo consiste en detectar patrones sobre los eventos recientes, es posible que un proceso batch que consolide todas las observaciones una vez al día no proporcione la latencia deseada. **Real-Time Intelligence** en Microsoft Fabric aborda necesidades de ingestión, almacenamiento, consultas y análisis sobre eventos frecuentes.

Un **Eventhouse** es un elemento de Fabric que aloja una o varias **KQL databases**. Las bases de datos KQL están optimizadas para explorar series temporales, logs y otros eventos y admiten el lenguaje **Kusto Query Language**, diferente de PySpark y del T-SQL utilizado por Fabric Warehouse. Una KQL database dispone de tablas y puede incluir materialized views, funciones y otros objetos. El Eventhouse no es una sustitución universal del Lakehouse: ambos pueden colaborar, pero tienen modelos de ejecución y patrones de uso diferentes.

La decisión arquitectónica depende de qué preguntas hay que responder. Para registros operativos de telemetría, es habitual filtrar por intervalos recientes, agrupar por identificador y detectar anomalías; KQL es una elección natural. Para construir conjuntos analíticos curados de gran volumen e historificados, puede interesar un Lakehouse Delta. Fabric permite integrar ambas experiencias, incluso con disponibilidad de datos en OneLake en los casos admitidos.

~~~mermaid
flowchart LR
  S["Eventos de aplicaciones"] --> E["Fabric Eventstream"]
  E --> H["Eventhouse"]
  H --> K["KQL Database"]
  K --> Q["Queryset KQL"]
  K --> V["Materialized View"]
  Q --> D["Dashboard / análisis"]
  V --> D
~~~

**Diagrama propio**, simplificado: Eventstream representa una alternativa de ingesta, no la única, ni significa que siempre deba usarse para trabajar con un Eventhouse.

## 2. Get started with KQL queries

### 2.1. Estructura del Eventhouse e ingesta

Al crear un Eventhouse puede generarse una KQL database predeterminada con el mismo nombre. Pueden añadirse otras según responsabilidades, cargas y permisos. Los orígenes admitidos incluyen conectores de nube, archivos, almacenamiento Azure, Amazon S3 y plataformas de streaming mediante las opciones documentadas. Un **database shortcut** puede permitir consultar una base KQL existente sin duplicar todos los datos. La función **OneLake availability** facilita compartir datos compatibles con otras cargas de Fabric cuando se habilita.

**Diferencia conceptual:** un shortcut apunta a datos externos y puede evitar un proceso de copia; una ingesta replica o incorpora eventos a la base KQL. Los efectos en permisos, latencia, actualización y costes no son iguales. Antes de configurar una fuente se debe conocer su esquema, timestamp de evento, zona horaria y posible orden de llegada.

### 2.2. Sintaxis KQL, pipeline operator y sensibilidad a mayúsculas

KQL representa muchas consultas como una tabla seguida de operadores conectados por \`|\`. Cada operador procesa el conjunto producido por el anterior. La sintaxis es expresiva para análisis temporal y agregaciones, pero tiene su propio conjunto de funciones y reglas de nombres. **KQL es case-sensitive**: nombres de tablas, columnas, funciones y valores de cadena deben escribirse con precisión.

~~~kql
Bikestream
| where No_Bikes > 0
| project Street, Neighbourhood, No_Bikes
| take 20
~~~

Este ejemplo supone la tabla \`Bikestream\` del laboratorio oficial. Primero reduce los registros a puntos con bicicletas disponibles; después selecciona las columnas que interesan y por último devuelve una muestra. **Take** no es un mecanismo de ordenación: sin un \`sort\` explícito no se está pidiendo «los 20 eventos más recientes». Tampoco se debe usar la vista de solo 20 filas para demostrar integridad del dataset completo.

### 2.3. Agregación y normalización de valores ausentes

~~~kql
Bikestream
| extend Zona = iff(isempty(Neighbourhood), "Desconocida", Neighbourhood)
| summarize Bicicletas = sum(No_Bikes), Estaciones = count() by Zona
| sort by Bicicletas desc
~~~

La consulta agrupa estaciones según una zona normalizada y suma bicicletas. Antes de comparar el resultado con las métricas de negocio, definir el significado de cada campo: en telemetría un mismo dispositivo puede producir múltiples eventos en poco tiempo; sumar todos los valores de disponibilidad de una ventana puede duplicar el estado de una estación. Para contar el estado actual hacen falta marcas temporales y una estrategia de selección de última observación que el dataset y la consulta soporten. **El ejemplo explica operadores, no diseña por sí solo un indicador listo para producción.**

### 2.4. Comparación KQL / T-SQL

Una KQL database tiene un endpoint de consulta compatible con un subconjunto de T-SQL para ciertos escenarios, pero no ofrece las mismas operaciones SQL que un Warehouse. En el laboratorio se utilizan ejemplos de ambas sintaxis para comparar \`where\`, \`project\`, \`summarize\`, \`sort\`, \`SELECT\`, \`GROUP BY\` y \`ORDER BY\`.

| Necesidad | KQL | T-SQL de consulta |
|---|---|---|
| Seleccionar columnas | \`project\` | \`SELECT\` |
| Filtrar registros | \`where\` | \`WHERE\` |
| Agregar por grupo | \`summarize ... by\` | \`GROUP BY\` |
| Ordenar | \`sort by\` | \`ORDER BY\` |
| Extender cálculo por fila | \`extend\` | Expresión calculada en \`SELECT\` |
| Trabajar con Eventhouse | Nativo y recomendado | Compatibilidad parcial, revisar restricciones |

**Fuente de esta unidad:** [Get started with KQL queries](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/query-data-kql-database-microsoft-fabric/includes/2-get-started-with-kql-queries.md).

## 3. KQL best practices

Las consultas de Eventhouse pueden procesar decenas de millones de eventos. La reducción temprana del trabajo es esencial. Es especialmente conveniente filtrar sobre un intervalo temporal ajustado y seleccionar únicamente las columnas necesarias. Si se retienen todas las filas y columnas para después agregarlas, el cómputo puede crecer innecesariamente.

~~~kql
TaxiTrips
| where pickup_datetime > ago(1d)
| where fare_amount > 0
| project vendor_id, fare_amount, pickup_datetime
| summarize Viajes = count(), ImporteMedio = avg(fare_amount) by vendor_id
| sort by Viajes desc
~~~

*Ejemplo basado en el esquema de demostración oficial; no se ha ejecutado.* La cláusula temporal permite evitar tratar todo el histórico, y el \`project\` reduce las columnas intermedias. Para consultas de producción se debe valorar la selectividad de cada filtro y el comportamiento de los índices y optimizaciones del motor.

### 3.1. Agregaciones y joins

\`summarize\` permite agregar por una o más dimensiones. Si se agrupa por identificadores casi únicos, puede generarse una cardinalidad enorme, por lo que conviene examinar la granularidad real del análisis. Los joins son más costosos que filtrados simples y requieren decidir su semántica y claves. En la guía oficial se recomienda revisar el orden de tablas y comenzar por la más pequeña en algunos patrones de join. No es una regla universal para todos los tamaños y estrategias del motor: se debe consultar la documentación de optimización adecuada y analizar el plan para cargas críticas.

### 3.2. Evitar errores analíticos

Un sistema de eventos puede contener observaciones tardías, duplicadas o fuera de orden. Filtrar por el timestamp de ingesta no es equivalente a filtrar por el timestamp de ocurrencia. Si una orden se genera a las 09:00 pero se incorpora a las 09:10, una consulta «eventos recibidos en los últimos cinco minutos» no responde a la misma pregunta que «eventos ocurridos en los últimos cinco minutos». Definir ambas marcas temporales y su semántica es tan importante como optimizar la sintaxis.

**Fuente:** [KQL best practices](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/query-data-kql-database-microsoft-fabric/includes/3-kql-best-practices.md).

## 4. Advanced features

### 4.1. Materialized views

Una **materialized view** conserva resultados precomputados de una consulta de agregación en vez de calcular siempre todo el histórico. Para fuentes con millones de registros que alimentan dashboards, esto puede reducir el trabajo de consulta. El modelo incorpora una parte ya materializada y los cambios pendientes de materializar que el motor tiene en cuenta cuando se consulta la vista. No confundir una materialized view con una simple función que ejecuta su código completo cada vez.

~~~kql
.create materialized-view EstacionesPorBarrio on table Bikestream
{
    Bikestream
    | summarize Registros = count() by Neighbourhood
}
~~~

*Ejemplo propio ilustrativo.* Solo crear sobre datos de prueba y con permisos suficientes. La vista agregará recuentos de eventos por barrio; si se quiere obtener el número actual de bicicletas por barrio, la definición del agregado tendría que usar una regla temporal diferente. Revisar si la operación escogida es compatible con las restricciones vigentes del servicio y la forma de actualización prevista.

### 4.2. Stored functions

Las **stored functions** permiten encapsular consultas reutilizables y, cuando corresponde, aceptar parámetros. Son útiles para estandarizar una misma lógica en varios análisis. A diferencia de una materialized view, una función no significa que el resultado se haya materializado previamente.

~~~kql
.create-or-alter function puntos_con_minimo(minimo:long)
{
    Bikestream
    | where No_Bikes >= minimo
    | project Street, Neighbourhood, No_Bikes
}
~~~

Después se invoca:

~~~kql
puntos_con_minimo(5)
| take 20
~~~

La función agrupa una condición de filtrado y selección, pero no sustituye permisos del sistema ni equivale a una política de seguridad de filas. Si el conjunto expone información sensible, diseñar controles efectivos de acceso sobre la fuente y sus consumidores.

**Fuente:** [Advanced features](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/query-data-kql-database-microsoft-fabric/includes/4-advanced-features.md).

## 5. Laboratorio oficial complementario

**Fuente:** [Work with data in a Microsoft Fabric eventhouse](https://github.com/MicrosoftLearning/mslearn-fabric/blob/main/Instructions/Labs/12-query-data-in-kql-database.md) · [instrucciones navegables](https://microsoftlearning.github.io/mslearn-fabric/Instructions/Labs/12-query-data-in-kql-database.html). **Duración aproximada publicada:** 25 minutos. El fichero original clasifica esta práctica en la formación DP-603; se añade aquí como **laboratorio oficial complementario pertinente al contenido**, no como laboratorio atribuido falsamente al curso DP-700.

### 5.1. Requisitos y escenario

Se necesita un Workspace en Fabric con capacidad de prueba o de pago, acceso al workload Real-Time Intelligence y permisos para crear Eventhouses. El ejercicio utiliza el dataset de ejemplo **Bike Rental Data** del propio servicio para crear un Eventhouse \`Bike_Database\` y la tabla \`Bikestream\`. No requiere instalar un broker de streaming local ni crear una fuente privada. Si la muestra ya no figura con ese nombre en la interfaz, hay que comprobar cambios en el tutorial oficial.

### 5.2. Crear Eventhouse

Abrir Fabric y el Workspace de pruebas, acceder a la experiencia Real-Time Intelligence, localizar la opción de ejemplo **Bike Rental Data** y crear la muestra. Verificar que el Eventhouse contiene la KQL database \`Bike_Database\`, una tabla \`Bikestream\` y un queryset desde el que ejecutar las consultas. No presupongas que los datos contienen siempre los mismos valores ni recuentos.

### 5.3. Ejecutar consultas

Primero, obtener una muestra:

~~~kql
Bikestream
| take 100
~~~

Después limitar columnas y obtener agregaciones:

~~~kql
Bikestream
| project Street, No_Bikes
| take 10
~~~

~~~kql
Bikestream
| summarize Bicicletas = sum(No_Bikes) by Neighbourhood
| sort by Bicicletas desc
~~~

El ejercicio original introduce también el tratamiento de barrios vacíos con \`case()\`, \`isempty()\` e \`isnull()\`, y filtrados concretos sobre \`Chelsea\`. Si un barrio no aparece no debe interpretarse automáticamente como un error, porque la muestra puede variar. La comprobación correcta es comparar la consulta, el esquema y los resultados disponibles en ese momento.

### 5.4. Comparar con T-SQL

En el queryset, ejecutar una consulta compatible:

~~~sql
SELECT Neighbourhood, SUM(No_Bikes) AS TotalBicicletas
FROM Bikestream
GROUP BY Neighbourhood
ORDER BY TotalBicicletas DESC;
~~~

La finalidad es reconocer el paralelismo entre operadores KQL y las cláusulas SQL, no asumir que el endpoint T-SQL de KQL admite todas las operaciones DDL o DML de Fabric Warehouse. La documentación oficial advierte de limitaciones de escritura y funciones respecto a SQL Server.

### 5.5. Verificación, errores y limpieza

| Problema | Posible causa | Comprobación |
|---|---|---|
| No aparece Eventhouse | Capacidad, permisos o experiencia no habilitada | Revisar Workspace y configuración de tenant |
| \`Bikestream\` no encontrada | Database equivocada o muestra no creada | Verificar Eventhouse, KQL database y tabla |
| Error de columna | Nombre mal escrito o mayúsculas diferentes | Inspeccionar esquema y respetar case sensitivity |
| Cero filas de Chelsea | No existen eventos de ese barrio | Inspeccionar \`distinct Neighbourhood\` |
| KQL ejecuta pero SQL falla | Limitaciones del endpoint compatible | Usar el dialecto admitido y revisar documentación |
| Diferencia entre suma y disponibilidad real | Se agregan snapshots sucesivos | Reconsiderar granularidad y marca temporal |

Recoger capturas de la tabla y del queryset, anotar únicamente los recuentos observados si se ejecuta realmente. Tras la práctica, eliminar el Workspace creado en exclusiva desde **Workspace settings → General → Remove this workspace**. Nunca borrar uno compartido.

## 6. Knowledge check y summary

Cuestiones de elaboración propia, no preguntas reproducidas del examen:

1. **¿Qué contiene un Eventhouse?** Una o varias KQL databases.
2. **¿Qué significa \`|\`?** Encadena operadores de consulta.
3. **¿Por qué es importante respetar mayúsculas y minúsculas?** KQL es case-sensitive.
4. **¿Qué hace \`project\`?** Selecciona o produce columnas para el siguiente operador.
5. **¿Qué hace \`summarize\`?** Agrega registros, opcionalmente agrupados.
6. **¿Qué optimización inicial es habitual?** Filtrar tiempos relevantes y limitar columnas procesadas.
7. **¿Por qué \`take 10\` no significa «últimos 10»?** No especifica orden temporal.
8. **¿Para qué se usa una materialized view?** Para mantener resultados agregados precomputados.
9. **¿En qué se diferencia de una stored function?** La función encapsula lógica reutilizable; no materializa automáticamente resultados.
10. **¿El endpoint T-SQL de Eventhouse es un Warehouse?** No; su compatibilidad y capacidades tienen límites.
11. **¿Qué riesgo presenta sumar snapshots de telemetría?** Duplicar estados en una métrica que pretendía medir el presente.
12. **¿Es el laboratorio Bikestream un laboratorio original de DP-700?** No: se incorpora como práctica complementaria oficial etiquetada DP-603.

**Síntesis:** Eventhouse y KQL son herramientas para analizar eventos y series temporales con baja latencia. Es imprescindible saber escribir filtros, proyecciones, agregaciones, ordenaciones y funciones, además de identificar costes de consulta, semántica temporal, duplicados y permisos.

## 7. Fuentes y alcance

- [Índice oficial de siete unidades](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/query-data-kql-database-microsoft-fabric/index.yml).
- [Get started with KQL](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/query-data-kql-database-microsoft-fabric/includes/2-get-started-with-kql-queries.md).
- [KQL best practices](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/query-data-kql-database-microsoft-fabric/includes/3-kql-best-practices.md).
- [Materialized views and stored functions](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/query-data-kql-database-microsoft-fabric/includes/4-advanced-features.md).
- [Laboratorio complementario oficial](https://github.com/MicrosoftLearning/mslearn-fabric/blob/main/Instructions/Labs/12-query-data-in-kql-database.md).

**Estado:** desarrolladas las competencias de las unidades introductorias, KQL básico, optimización, capacidades avanzadas y síntesis. La práctica complementaria está documentada, no ejecutada. El material de evaluación es propio.