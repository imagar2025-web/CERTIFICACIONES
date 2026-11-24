# 16 · Get started with Copilot in Microsoft Fabric for Data Warehouse

> **DP-700 · Warehouse.** [Microsoft Learn](https://learn.microsoft.com/en-us/training/modules/get-started-copilot-fabric-data-warehouse/) · [fuentes del módulo](https://github.com/MicrosoftDocs/learn/tree/main/learn-pr/wwl/get-started-copilot-fabric-data-warehouse) · [laboratorio oficial complementario](https://github.com/MicrosoftLearning/mslearn-fabric/blob/main/Instructions/Labs/22c-copilot-fabric-data-warehouse.md). **Revisión:** noviembre de 2026. **Práctica documentada, no ejecutada.**

## Índice

1. [Introducción](#1-introducción)
2. [Code completions](#2-code-completions)
3. [Chat](#3-chat)
4. [Quick actions](#4-quick-actions)
5. [Best practices](#5-best-practices)
6. [Laboratorio](#6-laboratorio)
7. [Knowledge check, summary y fuentes](#7-knowledge-check-summary-y-fuentes)

## 1. Introducción

Un asistente de IA integrado en un Warehouse permite acelerar tareas habituales de análisis y desarrollo SQL: completar fragmentos de código, explicar consultas, proponer transformaciones y sugerir correcciones. En Microsoft Fabric, **Copilot para Data Warehouse** ofrece estas experiencias cuando el tenant, la capacidad, la licencia y las políticas de la organización lo permiten. La disponibilidad de un icono de Copilot no debe darse por sentada en todos los entornos.

La distinción fundamental para un Data Engineer es la que existe entre **generar SQL plausible** y **generar SQL correcto para el contrato de datos real**. Un modelo puede proponer una columna que no existe, un tipo de join que duplica resultados o un DELETE sin el filtro adecuado. Copilot debe tratarse como un colaborador de generación y explicación, no como una autoridad que sustituye pruebas, revisiones de permisos o control de cambios.

## 2. Code completions

Las **code completions** proponen continuaciones mientras se escribe T-SQL. Pueden ayudar con SELECT, joins, funciones agregadas y referencias a objetos. Son especialmente útiles para patrones repetitivos, pero un texto sugerido sigue sujeto a validación del usuario.

Por ejemplo, ante una consulta de ventas y dimensiones, la herramienta puede sugerir un JOIN entre \`FactSales\` y \`DimProduct\`. Antes de aceptarlo se debe revisar que ambas columnas corresponden a la misma clave lógica, que los tipos sean compatibles y que la dimensión no tenga duplicados para esa clave. Una sugerencia sintáctica no demuestra cardinalidad.

### 2.1. Validación mínima

Comprobar que la consulta no altera datos cuando solo se quería analizar. Limitar el número de filas en exploración y comparar agregados con un conjunto de control. Si el código usa DML, revisar el WHERE, el entorno y la transacción. Los comentarios generados pueden ser incorrectos aunque el SQL compile.

**Fuente:** [Code completions](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/get-started-copilot-fabric-data-warehouse/includes/code-completions.md) (comprobar la estructura actual del repositorio si se reorganizan unidades).

## 3. Chat

La experiencia de **Chat** permite preguntar por la estructura de una query, solicitar explicaciones o pedir una propuesta de SQL. Es conveniente escribir prompts con contexto específico: nombre de la tabla de hechos, columna del importe, granularidad, filtros temporales y métrica deseada. Un prompt vago como «dame las mejores ventas» es ambiguo: puede significar mayor facturación, mayor cantidad de unidades o mayor margen.

### 3.1. Ejemplo de prompt propio

> Proponme una consulta T-SQL de lectura para obtener ingresos por trimestre desde dbo.FactSales y dbo.DimDate. La tabla de hechos contiene una fila por línea de pedido; utiliza el campo NetAmount y muestra año, trimestre y SUM(NetAmount). No modifiques datos. Explica la condición del join y señala las suposiciones que deban comprobarse.

La respuesta debe verificarse contra tablas reales. No compartir contraseñas ni identificadores secretos en los prompts. Cuando el entorno contiene información de carácter personal, aplicar las políticas del tenant sobre uso de IA y exposición de metadatos.

### 3.2. Explicar y depurar

El chat también puede ayudar a comprender un error de tipos, un alias inexistente o una consulta que parece lenta. No obstante, la causa de un cuello de botella debe fundamentarse en medidas de ejecución y no en una suposición de la IA. Si una consulta devuelve valores incorrectos, contrastar el dataset con consultas de control y no solo pedir a Copilot que «arregle la respuesta».

**Fuente:** [Chat](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/get-started-copilot-fabric-data-warehouse/includes/chat.md).

## 4. Quick actions

Las **quick actions** ofrecen opciones contextuales para tareas habituales del editor, como explicar, corregir o mejorar SQL según las capacidades vigentes. Deben entenderse como accesos directos a tareas de asistencia y no como un proceso de aprobación de código.

Una acción que propone simplificar una consulta puede cambiar implícitamente filtros, joins o redondeos. Para evaluar la sugerencia, comparar resultado antes/después sobre un conjunto conocido y revisar la lógica. Si se modifica una vista consumida por varios equipos, documentar el cambio y realizar pruebas de regresión. El ciclo de vida del SQL sigue perteneciendo al equipo de ingeniería.

**Fuente:** [Quick actions](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/get-started-copilot-fabric-data-warehouse/includes/quick-actions.md).

## 5. Best practices

### 5.1. Preguntas verificables

Los mejores prompts describen esquema, objetivo, restricciones, formato esperado y qué acciones están prohibidas. Para desarrollar un ETL, incluir si la carga debe ser incremental, si hay claves naturales o surrogate keys, qué columna representa la fecha de cambio y qué comportamiento se espera ante duplicados.

### 5.2. Seguridad y privacidad

No solicitar a un asistente que exponga registros sensibles o que genere procedimientos ejecutables con privilegios innecesarios. Validar que los usuarios conservan permisos correctos al ejecutar SQL. Una herramienta de IA no concede autorizaciones adicionales por el mero hecho de proponer la consulta; el acceso depende de las credenciales y controles del Warehouse.

### 5.3. Costes y disponibilidad

Copilot está sujeto a requisitos de servicio, capacidad y habilitación. Consultar la documentación oficial antes de diseñar un procedimiento que dependa de él; no convertir una función opcional en una dependencia obligatoria de producción. Si Copilot no está habilitado, el flujo de trabajo debe seguir siendo ejecutable con editor SQL normal.

### 5.4. Revisión de rendimiento

Una consulta generada puede funcionar con diez filas de ejemplo y degradarse a escala. Analizar joins, proyecciones, filtros y ejecución real mediante métricas de Warehouse. La asistencia de IA no sustituye la capacidad de interpretar Query insights o DMVs.

**Fuente:** [Best practices](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/get-started-copilot-fabric-data-warehouse/includes/best-practices.md).

## 6. Laboratorio

**Fuente oficial MicrosoftLearning:** [Use Copilot in Microsoft Fabric data warehouse](https://github.com/MicrosoftLearning/mslearn-fabric/blob/main/Instructions/Labs/22c-copilot-fabric-data-warehouse.md). La práctica se añade aquí como complemento; verificar su asociación de curso en los metadatos actuales. **Duración orientativa:** 30 minutos. **Sin ejecución.**

### 6.1. Escenario

Se trabaja con un Warehouse de ventas minoristas. El ejercicio crea tablas, inserta datos, consulta hechos y dimensiones y utiliza Copilot para generar, refinar o corregir SQL. Su propósito es entender la interacción entre asistente y motor relacional, no delegar sin comprobación la creación de objetos de producción.

### 6.2. Requisitos y preparación

Se necesita un Workspace con capacidad compatible y Copilot habilitado por la organización, además de permisos sobre el Warehouse. Abrir Fabric, crear el Workspace y Warehouse según los pasos oficiales y ejecutar los scripts de datos de ejemplo.

### 6.3. Generación y explicación de consultas

Probar sugerencias para un SELECT que una ventas y productos, revisar el SQL generado y ejecutarlo. Pedir una explicación de una consulta y contrastar esa explicación con la lógica real. Introducir de forma controlada un error de sintaxis en un script de laboratorio y examinar la sugerencia de corrección; no hacerlo en producción.

### 6.4. Comprobaciones

Anotar el prompt utilizado, el SQL producido y las modificaciones manuales. Comparar recuentos, agregados y filtros con una consulta SQL de referencia. Si Copilot sugiere un campo que no existe, corregir el prompt y documentar el error como limitación de la generación, no como error de datos.

### 6.5. Diagnóstico y limpieza

| Problema | Diagnóstico |
|---|---|
| Copilot no aparece | Tenant, capacidad, permisos o política |
| Sugiere columna inexistente | Esquema mal inferido; aportar contexto |
| Query devuelve cifras incoherentes | Revisar joins y granularidad |
| SQL propuesto es destructivo | No ejecutarlo; exigir lectura o autorización |
| Ejecución falla pese a sintaxis válida | Permisos o características no soportadas |
| La respuesta parece correcta | Comparar con validaciones independientes |

Eliminar solo el Workspace de laboratorio o los objetos propios. Revisar el consumo de capacidad y no asumir que no hay costes por usar datos de muestra.

## 7. Knowledge check, summary y fuentes

Cuestiones propias: ¿Para qué sirven las code completions? ¿Qué información debe contener un prompt SQL? ¿Cómo se controla una query potencialmente destructiva? ¿Por qué se necesitan pruebas de cardinalidad? ¿Qué diferencia hay entre sugerencia y validación? ¿Qué restricciones de tenant pueden impedir Copilot?

**Summary:** las capacidades de IA pueden acelerar desarrollo SQL, pero exigen revisión humana, seguridad de información, pruebas de resultados y control de los cambios.

Fuentes: [index.yml de ocho unidades](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/get-started-copilot-fabric-data-warehouse/index.yml) · [módulo de Microsoft Learn](https://learn.microsoft.com/en-us/training/modules/get-started-copilot-fabric-data-warehouse/) · [laboratorio oficial](https://github.com/MicrosoftLearning/mslearn-fabric/blob/main/Instructions/Labs/22c-copilot-fabric-data-warehouse.md). **Estado:** documentación conceptual y guía de práctica; no ejecutadas.