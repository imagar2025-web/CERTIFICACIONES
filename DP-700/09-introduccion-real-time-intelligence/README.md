# 09 · Get started with Real-Time Intelligence in Microsoft Fabric

> **DP-700 · Implement Real-Time Intelligence with Microsoft Fabric.** [Módulo Microsoft Learn](https://learn.microsoft.com/en-us/training/modules/get-started-kusto-fabric/) · [index.yml y archivos de origen](https://github.com/MicrosoftDocs/learn/tree/main/learn-pr/wwl/get-started-kusto-fabric) · [laboratorio DP-700](https://github.com/MicrosoftLearning/mslearn-fabric/blob/main/Instructions/Labs/07-real-time-Intelligence.md). **Revisión:** noviembre de 2026. **Procedimiento documentado; ejecución no realizada.**

## Índice

1. [Introduction](#1-introduction)
2. [Define real-time analytics](#2-define-real-time-analytics)
3. [Real-Time Intelligence components](#3-real-time-intelligence-components)
4. [Real-Time hub e ingestión](#4-real-time-hub-e-ingestión)
5. [Consultas KQL](#5-consultas-kql)
6. [Visualización y Activator](#6-visualización-y-activator)
7. [Laboratorio oficial](#7-laboratorio-oficial)
8. [Knowledge check y summary](#8-knowledge-check-y-summary)
9. [Fuentes y control](#9-fuentes-y-control)

## 1. Introduction

La analítica *batch* procesa datos almacenados en momentos determinados, por ejemplo un cierre diario de ventas. La analítica **near real time** examina eventos mientras fluyen y busca reducir el intervalo entre el hecho ocurrido y la decisión. No existe latencia verdaderamente cero: las redes, los brokers, la ingesta, el cómputo y la visualización introducen demoras. Diseñar un sistema real-time consiste en definir qué latencia máxima es aceptable para el negocio y qué garantías de consistencia, completitud y coste se necesitan.

Microsoft Fabric agrupa estas capacidades en **Real-Time Intelligence**, combinando fuentes, Eventstreams, Eventhouses, KQL, Real-Time Dashboards y Activator. El examen requiere reconocer el papel de cada componente y cómo se conectan, no tratar todo el sistema de eventos como si fuera un único Pipeline de Data Factory.

## 2. Define real-time analytics

### 2.1. Qué es un evento

Un **evento** representa la ocurrencia de un hecho: una transacción, el cambio de estado de una máquina, un acceso, la posición de un vehículo o una modificación en una base de datos. Puede incluir identificador, timestamp, origen, tipo y propiedades. El timestamp del hecho y el momento de recepción pueden diferir. Una red interrumpida puede entregar eventos tardíos; un reintento puede duplicar un evento; múltiples productores pueden emitir mensajes sin un orden global estable.

Un **stream** es una secuencia de eventos que se transmite a los consumidores. Una aplicación puede publicar muchos eventos por segundo y diversos consumidores pueden necesitar análisis diferentes. No se debe asumir que un stream es una tabla inmutable ni que todos los mensajes llegan exactamente una vez. El proceso receptor debe gestionar claves, reintentos y ventanas temporales según la semántica que ofrezca la fuente.

### 2.2. Diseño por requisitos de latencia

En un dashboard de logística, una actualización cada 30 segundos puede ser suficiente. En alertas de seguridad industrial, el umbral puede ser mucho más estricto y exigir un sistema con propiedades concretas de disponibilidad. Fabric debe evaluarse frente a los objetivos de latencia, volumen, mantenimiento, complejidad y costes de cada solución.

~~~mermaid
flowchart LR
  A["Apps / IoT / CDC"] --> B["Eventstream"]
  B --> C["Eventhouse · KQL DB"]
  C --> D["KQL Queryset"]
  D --> E["Real-Time Dashboard"]
  B --> F["Activator"]
  F --> G["Avisos / acciones"]
~~~

**Diagrama conceptual propio**. Activator también puede alimentarse mediante otras experiencias admitidas. Su presencia en un flujo no prueba que se haya configurado una acción efectiva.

**Fuente:** [Define real-time analytics](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/get-started-kusto-fabric/includes/2-define-real-time-analytics.md).

## 3. Real-Time Intelligence components

### 3.1. Eventstream

Un Eventstream define la canalización para recibir, transformar opcionalmente y enrutar eventos. Se configura mediante un canvas con origen, transformaciones y destinos. Puede ayudar a filtrar, proyectar, agrupar o preparar datos en movimiento antes de enviarlos a un Eventhouse, según las capacidades disponibles para cada tipo de fuente.

### 3.2. Eventhouse y KQL database

El **Eventhouse** aloja KQL databases especializadas en grandes volúmenes de eventos y consultas analíticas temporales. Una KQL database contiene tablas y puede exponer vistas materializadas y funciones. La función de Eventhouse no consiste únicamente en dibujar dashboards: proporciona almacenamiento y motor de consulta para explorar eventos históricos y actuales, sujeto a su configuración de retención.

### 3.3. KQL Queryset

**KQL Queryset** es el entorno de escritura, almacenamiento y ejecución de consultas Kusto. La sintaxis encadena operadores de filtrado, proyección, agregación y ordenación. La **sensibilidad a mayúsculas y minúsculas** importa tanto en identificadores como en funciones y valores de cadenas.

### 3.4. Real-Time Dashboard

El dashboard contiene tiles visuales basados en consultas. Ayuda a observar métricas actuales y comparar períodos, pero la visualización no sustituye el análisis de calidad del origen ni la definición de qué significa «actual». Si los datos llegan tarde, el dashboard puede mostrar una fotografía válida para el dato ingresado y aun así no reflejar el estado real del sistema.

### 3.5. Activator

**Activator** detecta condiciones definidas sobre datos y desencadena acciones como notificaciones o integraciones admitidas. Una regla debe definir el objeto monitoreado, la condición, el ámbito y una acción. Si se activan alertas por cada evento sin control, pueden generarse notificaciones repetidas que dificulten la operación.

**Fuente:** [Componentes](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/get-started-kusto-fabric/includes/3-describe-kusto-databases-tables.md).

## 4. Real-Time hub e ingestión

El **Real-Time hub** facilita descubrir fuentes y eventos disponibles en el entorno de Fabric. En una arquitectura de eventos se pueden realizar dos enfoques generales: ingestar mediante Eventstream con transformaciones y múltiples destinos, o llevar los datos directamente a KQL database utilizando conectores y funciones compatibles.

La elección depende de si se necesitan transformaciones en movimiento, enrutamiento a varios destinos, reuso de conexiones o controles de preprocesamiento. En algunos escenarios conviene conservar la entrada original y transformarla mediante capacidades de la propia KQL database, como las **update policies** admitidas. Debe distinguirse que filtrar antes de almacenar podría impedir reconstruir registros descartados; si hay requisitos de auditoría, conservar una versión de origen sin pérdida cuando proceda.

### 4.1. CDC y eventos de negocio

Un Change Data Capture (CDC) transmite cambios en una fuente transaccional. No es necesariamente igual a un evento semántico de negocio. Un UPDATE de una tabla puede describir la modificación de una columna, mientras un evento «pedido cancelado» expresa una acción empresarial. Antes de diseñar dashboards y alertas, saber qué nivel de significado poseen los eventos disponibles.

**Fuente:** [Real-Time hub y data ingestion](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/get-started-kusto-fabric/includes/3a-define-real-time-hub.md).

## 5. Consultas KQL

KQL presenta una canalización donde el orden de operadores afecta el conjunto intermedio. Para reducir consumo, conviene filtrar por período y seleccionar columnas antes de operaciones que agrupan o unen grandes conjuntos.

~~~kql
Stock
| where Timestamp > ago(30m)
| summarize Events = count() by Symbol
| sort by Events desc
~~~

*Ejemplo propio que presupone una tabla Stock con columnas Timestamp y Symbol; no forma parte de un resultado observado del laboratorio.* El nombre de tabla y las columnas deben ajustarse a la fuente real. Para calcular un precio actual hay que identificar el último evento por acción y no promediar arbitrariamente valores históricos.

Un ejemplo más simple, en una tabla del laboratorio de bicicletas, es:

~~~kql
Bikestream
| where No_Bikes >= 0
| summarize Observaciones = count() by Neighbourhood
| sort by Observaciones desc
~~~

El resultado cuenta observaciones, no necesariamente estaciones únicas. Si se requiere el número de estaciones, definir un identificador estable y una regla de deduplicación. En ejercicios de examen conviene leer atentamente si se pregunta por número de eventos, número de entidades o estado actual.

**Fuente:** [KQL basics](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/get-started-kusto-fabric/includes/4-write-queries-kusto-query-language.md).

## 6. Visualización y Activator

Los **Real-Time Dashboards** permiten configurar conexiones a KQL databases, consultas por tile y opciones de refresco. Son útiles para seguimiento operativo, pero hay que vigilar si cada tile consulta la misma tabla y si los filtros temporales son coherentes. Por ejemplo, no comparar «últimos cinco minutos» de una métrica con «últimas 24 horas» de otra sin avisar al usuario.

Activator añade respuesta automática ante determinadas condiciones. Un caso sería emitir un aviso cuando el precio de un activo supera un umbral durante una ventana definida. La regla debe tratar valores repetidos y recuperación posterior: el umbral puede oscilar alrededor de un límite y activar una cascada de notificaciones. La lógica de disparo, el ámbito y el destinatario deben verificarse antes de habilitar acciones reales.

**Fuentes:** [Visualización](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/get-started-kusto-fabric/includes/4b-visualize-data.md) y [Activator](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/get-started-kusto-fabric/includes/4c-activator.md).

## 7. Laboratorio oficial

**Procedencia:** [Get started with Real-Time Intelligence in Microsoft Fabric](https://github.com/MicrosoftLearning/mslearn-fabric/blob/main/Instructions/Labs/07-real-time-Intelligence.md) · [instrucciones navegables](https://microsoftlearning.github.io/mslearn-fabric/Instructions/Labs/07-real-time-Intelligence.html). **Duración orientativa:** 30 minutos. **Asociado oficialmente a DP-700.** No ha sido ejecutado por este agente.

### 7.1. Objetivo y requisitos

Se construye un escenario de análisis de datos simulados del mercado de valores. Un Eventstream ingiere eventos, un Eventhouse los almacena para consultarlos mediante KQL y un Real-Time Dashboard permite visualizarlos. Después se configura una alerta usando Activator. Es necesario un tenant Fabric con capacidad compatible y permisos para crear estos elementos.

### 7.2. Crear Workspace y Eventstream

Entrar en [Fabric](https://app.fabric.microsoft.com/) y crear un Workspace de pruebas. Crear un Eventstream siguiendo los detalles del origen simulado de cotizaciones que proporciona el laboratorio oficial. Si la opción ha cambiado de ubicación en la interfaz, priorizar la guía oficial actualizada. Verificar en el canvas la presencia de un origen y la llegada de eventos antes de interpretar cualquier gráfico.

**Comprobación:** deberían observarse eventos en la vista previa del stream; no se publican valores concretos ni volumen porque el ejercicio no se ha ejecutado.

### 7.3. Crear Eventhouse y destino

Crear Eventhouse y conectar el Eventstream con la KQL database siguiendo los parámetros indicados en el ejercicio. Revisar la tabla de destino, su esquema, la identidad y el estado del flujo. Una fuente funcionando no demuestra que el destino esté recibiendo datos si la conexión no se ha configurado o la ruta de transformación descarta eventos.

### 7.4. Consultar y visualizar

Abrir KQL Queryset y ejecutar las consultas del tutorial, comenzando con una muestra simple para inspeccionar columnas y tipos. Después generar una agregación temporal o por símbolo según el ejemplo. Crear Real-Time Dashboard con el origen KQL y configurar tiles. Anotar la consulta exacta de cada visualización; sin esa información es difícil justificar sus cifras.

### 7.5. Activator y verificación

Configurar la alerta de Activator según el escenario del laboratorio. Comprobar que la condición se corresponde con el dato emitido y que los destinatarios son de prueba. No habilitar avisos a personas ajenas. Si la regla nunca se activa, revisar el umbral y el stream; si se activa demasiadas veces, revisar filtrado y semántica de disparo.

### 7.6. Errores y limpieza

| Síntoma | Causa posible | Comprobación |
|---|---|---|
| Eventstream sin registros | Origen no emite o no conecta | Revisar source y vista previa |
| Eventhouse vacío | Sink sin configurar o flujo fallido | Revisar destino KQL |
| KQL falla | Columnas, tipos o mayúsculas erróneos | Inspeccionar esquema |
| Dashboard sin datos | Data source o consulta equivocada | Probar query en Queryset |
| Alerta no se activa | Umbral no alcanzado | Revisar eventos y condición |
| Avisos repetidos | Regla demasiado amplia | Restringir ámbito y definir respuesta |
| Consumo elevado | Flujo o refresco sigue activo | Detener recursos y revisar capacidad |

Al terminar, borrar únicamente el Workspace de laboratorio si se ha creado para ese uso exclusivo, o limpiar cada elemento propio. Revisar triggers y streams activos para evitar consumo continuado.

## 8. Knowledge check y summary

Preguntas propias:

1. **¿Por qué hablamos de near real time?** Porque existe latencia de red, ingesta y cómputo.
2. **¿Qué diferencia hay entre evento y stream?** Evento es un registro; stream es la secuencia de eventos.
3. **¿Eventstream almacena necesariamente todo el histórico?** No; su papel principal es ingesta, transformación y enrutamiento.
4. **¿Qué función tiene Eventhouse?** Aloja KQL databases para almacenamiento y consulta de eventos.
5. **¿Qué ofrece KQL Queryset?** Un entorno para consultas Kusto.
6. **¿Qué muestra un Real-Time Dashboard?** Visualizaciones derivadas de consultas a fuentes compatibles.
7. **¿Cuándo actúa Activator?** Cuando una regla configurada detecta la condición pertinente.
8. **¿Por qué diferenciar event time e ingestion time?** Pueden divergir por retrasos.
9. **¿Qué riesgo tienen eventos duplicados?** Contar dos veces un mismo hecho o disparar alertas repetidas.
10. **¿Cómo se diseña un indicador «actual»?** Definiendo una ventana o última observación por entidad.

**Summary:** Real-Time Intelligence vincula ingestión, consulta, visualización y automatización sobre datos en movimiento. La calidad de un sistema depende tanto de la semántica de los eventos y las garantías de ingesta como de las configuraciones visuales.

## 9. Fuentes y control

- [index.yml: diez unidades oficiales](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/get-started-kusto-fabric/index.yml).
- [Real-time analytics](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/get-started-kusto-fabric/includes/2-define-real-time-analytics.md).
- [Componentes RTI](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/get-started-kusto-fabric/includes/3-describe-kusto-databases-tables.md).
- [Ingesta y Real-Time hub](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/get-started-kusto-fabric/includes/3a-define-real-time-hub.md).
- [Visualización](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/get-started-kusto-fabric/includes/4b-visualize-data.md).
- [Activator](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/get-started-kusto-fabric/includes/4c-activator.md).
- [Lab DP-700](https://github.com/MicrosoftLearning/mslearn-fabric/blob/main/Instructions/Labs/07-real-time-Intelligence.md).

**Control:** diez unidades representadas con teoría, práctica, evaluación y síntesis; no existen resultados reales de ejecución.