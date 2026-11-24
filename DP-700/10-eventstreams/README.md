# 10 · Use Eventstream in Microsoft Fabric

> **DP-700 · Real-Time Intelligence.** [Módulo Microsoft Learn](https://learn.microsoft.com/en-us/training/modules/explore-event-streams-microsoft-fabric/) · [fuentes oficiales](https://github.com/MicrosoftDocs/learn/tree/main/learn-pr/wwl/explore-event-streams-microsoft-fabric) · [laboratorio DP-700](https://github.com/MicrosoftLearning/mslearn-fabric/blob/main/Instructions/Labs/09-real-time-analytics-eventstream.md). **Revisión:** noviembre de 2026. **Estado:** contenido técnico y práctica documentados; ejecución no realizada.

## Índice

1. [Introduction](#1-introduction)
2. [Understand Eventstream components](#2-understand-eventstream-components)
3. [Configure sources and destinations](#3-configure-sources-and-destinations)
4. [Transform and route event data](#4-transform-and-route-event-data)
5. [Laboratorio oficial](#5-laboratorio-oficial)
6. [Knowledge check y summary](#6-knowledge-check-y-summary)
7. [Fuentes y control](#7-fuentes-y-control)

## 1. Introduction

Un sistema analítico batch espera a que los datos estén disponibles para iniciar un proceso periódico. Una arquitectura **event-driven** puede reaccionar a mensajes conforme llegan desde aplicaciones, dispositivos o cambios en bases de datos. El procesamiento de eventos requiere una cadena de responsabilidades: recibir los mensajes, autenticar los orígenes, interpretar el esquema, filtrar o transformar cuando sea necesario y enrutar el resultado hacia uno o varios consumidores.

**Eventstream** es la capacidad de Microsoft Fabric Real-Time Intelligence que permite realizar ese flujo de forma visual sin administrar manualmente toda la infraestructura de streaming. No es idéntico a un Data Pipeline de Data Factory: este último organiza tareas con actividades de control y copia; Eventstream se orienta al movimiento continuo de eventos y sus transformaciones y destinos compatibles.

### 1.1. Modelo de eventos

Un evento debería contener, cuando proceda, identificador, tipo, instante de ocurrencia, entidad afectada, origen y propiedades. Un flujo de telemetría de bicicletas compartidas puede emitir la cantidad de bicicletas y de puntos de anclaje libres en una estación concreta. La llegada de dos mensajes sobre una estación no implica que existan dos estaciones: pueden ser dos observaciones sucesivas del mismo objeto.

El diseño debe establecer reglas para retrasos, duplicados, mensajes fuera de orden y cambios de esquema. Si la fuente envía JSON sin contrato, un campo numérico puede comenzar a aparecer como cadena o un valor nulo puede faltar por completo. La interfaz visual simplifica la conexión, pero no corrige por sí sola estas inconsistencias.

## 2. Understand Eventstream components

### 2.1. Sources

Una **source** es el punto desde el cual Eventstream recibe eventos. La fuente puede ser un servicio Microsoft o un proveedor compatible externo; existen conectores para determinadas plataformas de mensajería, servicios de Azure y cambios de datos. La configuración depende del conector: endpoint, nombre del recurso, autenticación, red y políticas de acceso.

La identidad utilizada debe tener permisos mínimos para la lectura necesaria. Evitar utilizar cuentas personales con permisos excesivos en soluciones automatizadas. Para redes privadas o fuentes de otra nube, comprobar los requisitos concretos de conectividad y seguridad.

### 2.2. Transformations

Las **transformations** procesan datos en movimiento: filtrar, cambiar tipos, derivar columnas, agregar o redirigir eventos. Filtrar temprano puede evitar enviar datos irrelevantes al destino, pero también significa descartar información que podría ser necesaria para auditoría. Si la fuente original debe conservarse, diseñar una rama de enrutamiento bruto adicional u otro mecanismo de retención.

### 2.3. Destinations

Una **destination** es el receptor de los eventos. Las capacidades dependen del tipo de salida elegido. Un Eventhouse permite almacenar y consultar eventos mediante KQL, mientras otros destinos admitidos pueden dirigirse a distintos workloads o componentes de automatización. La forma del mensaje debe coincidir con el esquema y los requisitos del sink; un destino conectado pero mal mapeado puede recibir datos inválidos o no recibir ninguno.

~~~mermaid
flowchart LR
  S["IoT / CDC / Sample events"] --> E["Eventstream"]
  E --> F{"Validación y ruta"}
  F -->|Todos| R["Raw sink"]
  F -->|Válidos| K["Eventhouse KQL"]
  F -->|Alerta| A["Activator"]
~~~

*Diagrama propio.* Los destinos y las transformaciones concretos requieren configuraciones compatibles con el servicio.

### 2.4. Canvas y observabilidad

El **canvas** representa el grafo de procesamiento. Es útil para documentar cómo se relacionan los nodos, pero una conexión visual no basta para demostrar que llegan mensajes. Hay que comprobar estado del origen, vista previa de datos, errores de transformación, destino real y que las consultas devuelven registros nuevos.

**Fuente:** [Eventstream components](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/explore-event-streams-microsoft-fabric/includes/2-eventstream-components.md).

## 3. Configure sources and destinations

### 3.1. Escoger una fuente

El módulo oficial menciona orígenes de Microsoft como Event Hubs, IoT Hub, Service Bus y fuentes CDC admitidas, además de otras plataformas. Cada tipo tiene semántica propia. Por ejemplo, CDC representa cambios de estado de una base de datos y puede incluir operaciones de inserción, actualización o borrado; un stream de mensajes de telemetría puede representar snapshots independientes. Ignorar esta diferencia conduce a métricas incorrectas.

Para un sistema de producción, establecer si cada evento incluye un identificador único, qué políticas de reintento aplica el productor, cómo se reconoce un mensaje inválido y si existe posibilidad de recuperar el histórico de la fuente.

### 3.2. Configurar el destino KQL

Al conectar Eventstream a un Eventhouse, definir la KQL database y la tabla objetivo. Revisar el contrato de columnas y tipos. Si la fuente tiene timestamps ISO 8601, comprobar cómo se interpretan, especialmente cuando contienen zona horaria. Un campo incorrectamente interpretado como texto limita los filtros temporales y puede impedir las optimizaciones esperadas.

### 3.3. Resiliencia

En streaming conviene diferenciar disponibilidad de fuente, recepción de eventos y persistencia final. Un Eventstream activo no demuestra que todos los mensajes estén almacenados en el sink; puede existir una transformación con filtro, un error de esquema o un destino desconectado. El control operativo debe incluir monitorización del flujo y comprobaciones sobre los datos en destino.

El coste depende de la infraestructura y de los recursos consumidos por el flujo y el almacenamiento. Una muestra de laboratorio debe detenerse al terminar; una aplicación productiva necesita dimensionar capacidad y retención.

**Fuente:** [Configure eventstreams](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/explore-event-streams-microsoft-fabric/includes/3-setup-eventstreams.md).

## 4. Transform and route event data

### 4.1. Filtrado y calidad

Supongamos mensajes de estaciones de bicicleta que contienen \`StationId\`, \`Timestamp\`, \`Bikes\` y \`FreeDocks\`. La primera regla puede descartar observaciones con IDs vacíos, y otra puede enviar a una ruta de incidencia cantidades negativas. Sin embargo, **descartar negativos sin registrar el motivo** ocultaría un error de captura. Una estrategia de calidad mejor conserva una señal de error o una copia en cuarentena, cuando el escenario lo requiera.

### 4.2. Enriquecimiento

Se pueden añadir valores derivados o normalizados. Por ejemplo, convertir la temperatura de Fahrenheit a Celsius, calcular el porcentaje de ocupación o seleccionar únicamente campos útiles. La fórmula de ocupación \`Bikes / (Bikes + FreeDocks)\` debe tratar divisiones por cero y estaciones fuera de servicio. Si se trabaja con ventanas temporales, diferenciar un promedio de eventos de un promedio por estación, que no siempre representa la misma métrica.

### 4.3. Enrutamiento a varias salidas

Un mismo stream puede alimentar una tabla de eventos y una condición operativa. La tabla sirve para consultas históricas y análisis, mientras un sistema de alertas permite reaccionar. La transformación debería ser específica para cada destino: no tiene por qué almacenarse solo el subconjunto que activa una alerta, ni una alerta debe dispararse por cada actualización rutinaria.

### 4.4. Contratos y cambios de esquema

Cuando aparece una nueva versión de la aplicación emisora, puede añadirse una columna o cambiarse el significado de un campo. La solución debe registrar versión del evento y garantizar compatibilidad. Un destino que interpreta \`Precio\` como importe con IVA incluido cuando la fuente lo considera sin IVA devuelve datos aparentemente válidos pero semánticamente incorrectos. Las pruebas de contrato son fundamentales en streaming.

**Fuente:** [Transform and route data](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/explore-event-streams-microsoft-fabric/includes/4-route-event-data-to-destinations.md).

## 5. Laboratorio oficial

**Fuente:** [Ingest real-time data with Eventstream in Microsoft Fabric](https://github.com/MicrosoftLearning/mslearn-fabric/blob/main/Instructions/Labs/09-real-time-analytics-eventstream.md) · [versión navegable](https://microsoftlearning.github.io/mslearn-fabric/Instructions/Labs/09-real-time-analytics-eventstream.html). **Asociación:** DP-700 y DP-603 en el catálogo oficial. **Duración aproximada:** 30 minutos. **Sin ejecución real.**

### 5.1. Objetivo y preparación

El laboratorio utiliza un origen de ejemplo que emite datos de estaciones de bicicletas de alquiler. Se crea Eventhouse y Eventstream, se configura la fuente y un destino KQL, se consultan eventos, se realiza una transformación y se analizan los resultados. También se añade un destino Activator. Requiere un Workspace con capacidad compatible y permisos para crear los elementos.

### 5.2. Crear Workspace y Eventhouse

Iniciar sesión en Fabric y crear un Workspace de práctica. A continuación crear un Eventhouse y comprobar que contiene una KQL database. Registrar el nombre del recurso y no mezclarlo con otro Eventhouse del tenant. Si no aparece la opción, revisar la capacidad y permisos.

### 5.3. Crear Eventstream

Añadir Eventstream y seleccionar el origen de muestra de datos de bicicletas utilizado en el tutorial. Verificar la vista previa de eventos y revisar columnas y tipos. Sin eventos entrantes no tiene sentido interpretar un destino vacío como un error de la consulta KQL.

### 5.4. Añadir destino y consultar

Configurar el destino Eventhouse seleccionando la KQL database y la tabla indicada en la guía. Una vez conectado, ejecutar desde Queryset una consulta de inspección sobre la tabla creada. El nombre exacto depende de la configuración del ejercicio: no se inventan nombres de tablas ni filas obtenidas.

### 5.5. Transformación y segunda consulta

Añadir los nodos de transformación indicados por el tutorial, como agregación u operaciones de preparación, y dirigirlos al destino correspondiente. Comparar datos brutos y transformados para comprobar qué ha cambiado. Una agregación puede devolver menos registros porque reúne observaciones, pero eso no demuestra automáticamente pérdida o calidad; hay que revisar el grupo, la ventana y el criterio.

### 5.6. Activator

Añadir la salida Activator que plantea el ejercicio. Comprobar que el objeto o evento correcto llega a esta experiencia antes de crear reglas que puedan enviar avisos. En un laboratorio, cualquier acción de notificación debe dirigirse al usuario de prueba o a destinatarios autorizados.

### 5.7. Errores y limpieza

| Síntoma | Causa probable | Comprobación |
|---|---|---|
| No se observan eventos | Fuente desconectada | Revisar sample source y preview |
| Destination sin datos | Mapping, permisos o conexión | Verificar Eventhouse y tabla |
| Error de transformación | Tipo incompatible o nombre de campo | Inspeccionar esquema de entrada |
| Métrica agregada inesperada | Ventana o granularidad equivocada | Revisar regla de agregación |
| Activator no detecta eventos | Salida incorrecta | Revisar ruta y configuración |
| Consumo sostenido | Flujo activo después del laboratorio | Detener o eliminar los elementos de prueba |

Al terminar, eliminar el Workspace propio o los elementos creados según el procedimiento oficial, y comprobar que no permanecen fuentes de eventos de laboratorio activas. No borrar recursos compartidos.

## 6. Knowledge check y summary

Preguntas originales:

1. **¿Cuál es el papel de Eventstream?** Recoger, transformar opcionalmente y distribuir eventos.
2. **¿Eventstream y Pipeline son idénticos?** No, tienen distintos modelos de procesamiento.
3. **¿Qué es un destino?** Un receptor de datos procesados o reenviados.
4. **¿Un preview correcto garantiza persistencia?** No, hay que verificar el destino.
5. **¿Por qué es importante event time?** Puede diferir de la hora de recepción.
6. **¿Qué riesgo tiene filtrar eventos demasiado pronto?** Perder datos necesarios para auditoría o reanálisis.
7. **¿Qué significa CDC?** Captura de cambios en una fuente compatible.
8. **¿Por qué hay que verificar mappings?** Para evitar incompatibilidades de esquema.
9. **¿Qué aporta Activator?** Detección y respuesta ante condiciones configuradas.
10. **¿Qué hacer tras finalizar?** Detener el flujo de prueba y gestionar los recursos.

**Summary:** Eventstream integra productores y consumidores mediante flujos continuos. El trabajo de ingeniería consiste en garantizar contrato de eventos, permisos, rutas, datos correctos, observabilidad y costes, no solo conectar nodos en el canvas.

## 7. Fuentes y control

- [index.yml: siete unidades](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/explore-event-streams-microsoft-fabric/index.yml).
- [Componentes](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/explore-event-streams-microsoft-fabric/includes/2-eventstream-components.md).
- [Configuración](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/explore-event-streams-microsoft-fabric/includes/3-setup-eventstreams.md).
- [Transformaciones y rutas](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/explore-event-streams-microsoft-fabric/includes/4-route-event-data-to-destinations.md).
- [Laboratorio original asociado a DP-700](https://github.com/MicrosoftLearning/mslearn-fabric/blob/main/Instructions/Labs/09-real-time-analytics-eventstream.md).

**Control:** teoría, laboratorio y evaluación documentados; no hay datos de ejecución ni capturas.