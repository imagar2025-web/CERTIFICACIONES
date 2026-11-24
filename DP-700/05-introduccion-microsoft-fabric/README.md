# 05 · Introduction to end-to-end analytics using Microsoft Fabric

> **Certificación:** DP-700 · Microsoft Fabric Data Engineer Associate. **Ruta:** Implement a Lakehouse with Microsoft Fabric (también aparece en Warehouse). [Módulo Microsoft Learn](https://learn.microsoft.com/en-us/training/modules/introduction-end-analytics-use-microsoft-fabric/) · [Fuentes originales](https://github.com/MicrosoftDocs/learn/tree/main/learn-pr/wwl/introduction-end-analytics-use-microsoft-fabric). **Fecha:** noviembre de 2026. **Estado:** teoría desarrollada; el índice oficial no incluye un laboratorio independiente.

## Índice

1. [Introduction: por qué una plataforma de análisis unificada](#1-introduction-por-qué-una-plataforma-de-análisis-unificada)
2. [Explore analytics with Microsoft Fabric](#2-explore-analytics-with-microsoft-fabric)
3. [Understand roles in a data team](#3-understand-roles-in-a-data-team)
4. [Use Microsoft Fabric](#4-use-microsoft-fabric)
5. [Caso práctico complementario y verificación](#5-caso-práctico-complementario-y-verificación)
6. [Knowledge check](#6-knowledge-check)
7. [Summary y fuentes](#7-summary-y-fuentes)

## 1. Introduction: por qué una plataforma de análisis unificada

Las plataformas analíticas tradicionales suelen integrar múltiples servicios: una herramienta de ETL, un data lake, otro motor para Spark, un almacén relacional, herramientas de BI y un servicio de seguridad. Cada componente puede resolver correctamente su problema, pero la suma introduce costes de integración, transferencia de datos, aprovisionamiento de credenciales y trazabilidad. Si el dato se copia repetidamente entre plataformas, es difícil determinar qué versión contiene la verdad operativa más reciente y en qué punto debe aplicarse una regla de seguridad.

**Microsoft Fabric** propone una plataforma SaaS que reúne diversos workloads de datos y análisis alrededor de una experiencia de colaboración y almacenamiento común. SaaS significa que Microsoft administra buena parte de la infraestructura subyacente; no significa que los ingenieros estén libres de diseñar la seguridad, los pipelines, el modelo de datos, la gobernanza o el control de costes.

El principio de arquitectura más importante es separar el **plano de almacenamiento** (OneLake y sus objetos) de los **motores de análisis y transformación** (Data Factory, Spark, Warehouse, Real-Time Intelligence y experiencias relacionadas). Un mismo dato puede participar en diferentes casos de uso sin que cada equipo tenga que implantar su propio sistema desconectado. Sin embargo, las capacidades exactas de reutilización dependen de qué formato, servicio, identidad y permisos se utilicen.

### 1.1. Los tres problemas de una solución end-to-end

En cualquier solución analítica se deben contestar tres preguntas de ingeniería. **De dónde proceden los datos:** API, bases de datos, logs, ficheros, otros tenants o almacenamiento externo. **Cómo se convierten en información fiable:** normalización de tipos, depuración, detección de duplicados, reglas de negocio, trazabilidad y prueba de calidad. **Cómo se consumen y gobiernan:** tablas y modelos semánticos, consultas SQL, dashboards, aplicaciones y controles de acceso.

En Fabric cada respuesta puede necesitar varios workloads. La integración entre ellos reduce fricciones, pero no elimina la responsabilidad de definir contratos de datos. Una tabla Gold sin descripción de granularidad puede producir cálculos incorrectos aunque se ejecute en una plataforma muy avanzada.

## 2. Explore analytics with Microsoft Fabric

### 2.1. OneLake

**OneLake** es la capa de almacenamiento unificada de Fabric. Ofrece un punto de referencia común para los datos y está integrada con distintos motores de la plataforma. Su planteamiento evita que cada workload deba mantener copias independientes por obligación arquitectónica. No debe confundirse OneLake con un Lakehouse concreto: OneLake es la capacidad de almacenamiento a nivel de plataforma; un Lakehouse es un elemento analítico que organiza ficheros y tablas sobre ese almacenamiento.

Uno de los mecanismos de integración son los **shortcuts**, que permiten presentar ubicaciones o datos externos sin realizar necesariamente una copia física. No hay que interpretar shortcut como garantía de ausencia de coste, velocidad instantánea, compatibilidad universal o bypass de permisos. Debe comprobarse el origen y la semántica de autorización, la frescura y el rendimiento.

### 2.2. Workspaces

Un **Workspace** agrupa los elementos que desarrolla o consume un equipo. En él se pueden crear Lakehouses, Notebooks, Pipelines, Warehouses, Eventhouses y otros objetos en función de capacidades y permisos. Los Workspaces proporcionan un límite práctico para organizar responsabilidades, entornos, colaboración y asignación de capacidad. No constituyen automáticamente una frontera completa de seguridad de datos: los permisos de elemento, datos y roles deben configurarse de acuerdo con la sensibilidad real.

A efectos de ingeniería, conviene diseñar nombres coherentes, por ejemplo \`ws-ventas-dev\` y \`ws-ventas-prod\`, y mantener separadas las fuentes de prueba y los datos productivos. Si distintos equipos necesitan acceder solo a un subconjunto de columnas o filas, no basta con conceder acceso indiscriminado a un Workspace de producción.

### 2.3. Administración y gobernanza

El **tenant** y las capacidades determinan controles de administración y recursos disponibles. El administrador puede habilitar experiencias, asignar permisos o restringir configuraciones. La existencia de un menú en una captura de Microsoft Learn no garantiza que esté habilitado para todas las licencias. El principio del mínimo privilegio debe aplicarse a identidades humanas y a procesos automatizados. Junto a ello se necesita visibilidad del consumo y de los elementos que permanecen ejecutándose.

~~~mermaid
flowchart TB
    subgraph S["Microsoft Fabric SaaS"]
      O["OneLake · almacenamiento"]
      W["Workspaces · organización y roles"]
      F["Data Factory · ingesta/orquestación"]
      SP["Spark · procesamiento"]
      WH["Warehouse · analítica SQL"]
      RT["Real-Time Intelligence · eventos"]
      BI["Power BI · consumo"]
    end
    W --> F
    W --> SP
    W --> WH
    W --> RT
    F --> O
    SP --> O
    WH --> O
    RT -.integración compatible.-> O
    O --> BI
~~~

*Diagrama conceptual propio.* No representa que los motores puedan leer indiscriminadamente todos los formatos y objetos. Cada motor tiene conectores, permisos y límites concretos. **Fuente:** [Explore analytics](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/introduction-end-analytics-use-microsoft-fabric/includes/2-explore-analytics-fabric.md).

## 3. Understand roles in a data team

### 3.1. Responsabilidades diferenciadas

Un **data engineer** construye ingestas, transformaciones, estructuras de almacenamiento y controles de calidad. Un **data analyst** interpreta los datos y modela indicadores; un **data scientist** explora patrones y desarrolla modelos de aprendizaje; un **BI developer** transforma criterios de negocio en modelos y artefactos de consumo; un **platform administrator** gestiona recursos, configuración y controles organizativos. Estos nombres no implican que las organizaciones empleen equipos completamente separados.

La clave es la colaboración sobre contratos estables. Si el ingeniero modifica una columna \`precio_unitario\` por \`pvp\`, un modelo semántico puede romperse. El equipo debería registrar cambios en los esquemas, validar dependencias y hacer los despliegues de manera coordinada. Fabric facilita que los participantes trabajen en un entorno común, pero la definición del propietario, el responsable de datos y los criterios de aprobación sigue siendo una responsabilidad organizativa.

### 3.2. Evitar silos de datos

Los silos aparecen cuando cada departamento mantiene su propio repositorio y reglas. Una arquitectura compartida permite publicar un conjunto Silver validado y diferentes tablas o modelos Gold orientados a consumidores concretos. El dato puede conservar una procedencia común, mientras cada equipo define métricas de negocio que deben documentarse y aprobarse.

Un buen contrato de datos debe incluir nombre, significado, granularidad, claves, tipos, reglas de nulos, frecuencia de actualización, propietario, nivel de confidencialidad y compatibilidad de versiones. Sin ese contrato, la plataforma ofrece integración técnica pero no consistencia semántica.

**Fuente:** [Data team](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/introduction-end-analytics-use-microsoft-fabric/includes/3-data-team.md).

## 4. Use Microsoft Fabric

### 4.1. Habilitación, capacidad y permisos

Para utilizar Fabric puede ser necesario que la organización haya habilitado el servicio y asignado una capacidad compatible. Los administradores pertinentes, como Fabric administrator o roles administrativos de Microsoft asociados, gestionan configuraciones del tenant. El usuario final no debe suponer que puede habilitar características corporativas sin autorización.

El procedimiento habitual comienza en [app.fabric.microsoft.com](https://app.fabric.microsoft.com/), creando un Workspace y eligiendo un modo de licencia adecuado. Después se crean elementos del workload correspondiente. Antes de iniciar pruebas, comprobar si la capacidad de evaluación tiene fecha de expiración y revisar qué recursos o programaciones podrían generar consumo.

### 4.2. Descubrir datos y crear elementos

El **OneLake catalog** ayuda a localizar elementos y datos que están publicados y que la identidad está autorizada a ver. El descubrimiento no implica que se tenga permiso de lectura sobre cualquier tabla. Desde un Workspace, los profesionales pueden crear objetos según su rol: un Notebook para Spark, un Lakehouse para tablas y ficheros, un Dataflow Gen2 para preparación o un Pipeline para orquestación.

Es recomendable elegir el workload por la función requerida, no por familiaridad con un menú. Un ingeniero que solo necesita copiar un archivo puede usar una opción de copia; otro que necesita transformar datos masivos con código puede usar Spark; otro que precisa consultas SQL transaccionales compatibles debe revisar las capacidades específicas de Warehouse o SQL database en Fabric.

### 4.3. Copilot, data agents y Fabric IQ

El módulo de Microsoft Learn incorpora capacidades de IA, incluyendo **Copilot**, **data agents** y **Fabric IQ**. Estas herramientas ayudan a descubrir, consultar o razonar sobre datos cuando están habilitadas y configuradas apropiadamente. Conviene distinguir sus funciones: Copilot ofrece asistencia dentro de experiencias concretas; los agentes de datos orientan preguntas sobre fuentes y conocimiento; y Fabric IQ agrupa capacidades semánticas e inteligencia de negocio según sus características vigentes.

Las respuestas generadas mediante IA no sustituyen la validación. Si se genera SQL, M o PySpark, revisar permisos, fuentes, columnas, rendimiento y semántica. No se deben introducir secretos, credenciales o datos sensibles en prompts fuera de las políticas corporativas. La disponibilidad y facturación de estas opciones depende de capacidad, tenant y configuración.

**Fuente:** [Use Fabric](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/introduction-end-analytics-use-microsoft-fabric/includes/4-use-fabric.md).

## 5. Caso práctico complementario y verificación

**Esta unidad oficial no contiene un laboratorio independiente en su index.yml.** El siguiente ejercicio es una ampliación original para practicar el reconocimiento de elementos, sin inventar un laboratorio MicrosoftLearning correspondiente.

### Escenario

Una compañía desea crear un prototipo de análisis de ventas en Fabric con un Workspace aislado. El objetivo no es construir todavía una arquitectura productiva sino reconocer los objetos y comprender su relación.

1. Verificar que la identidad dispone de acceso a Fabric y una capacidad adecuada. Registrar el tipo de capacidad y los límites de la evaluación.
2. Crear un Workspace exclusivo de prácticas, sin incluir datos sensibles.
3. Crear un Lakehouse con nombre \`lh_ventas_demo\` y examinar la separación entre \`Files\` y \`Tables\`.
4. Crear un Notebook conectado a ese Lakehouse y ejecutar un DataFrame de dos filas sin acceder a datos externos.
5. Guardar una tabla Delta de prueba y consultar su registro en el explorador del Lakehouse.

~~~python
df = spark.createDataFrame(
    [(1, "Producto A", 15.0), (2, "Producto B", 24.0)],
    ["id_producto", "nombre", "precio"]
)
df.write.format("delta").mode("overwrite").saveAsTable("demo_productos")
display(spark.sql("SELECT * FROM demo_productos"))
~~~

**Resultados esperados:** un Workspace con los elementos creados, una tabla \`demo_productos\` consultable y conocimiento de qué acciones precisaron Spark. No hay resultados de ejecución reales registrados para este ejercicio. Si no se permite iniciar Spark, revisar asignación de capacidad y restricciones del administrador. Si el nombre de la tabla no se resuelve, revisar el Lakehouse asociado y el esquema predeterminado.

**Limpieza:** al finalizar, eliminar los elementos propios o, solo si el Workspace es de uso exclusivo, eliminar el Workspace entero. Revisar que no queden programaciones; borrar el Workspace no es una operación para desactivar por completo una capacidad de pago.

## 6. Knowledge check

Preguntas originales de repaso, no transcripciones de Microsoft:

1. **¿Por qué Fabric se describe como SaaS?** Porque entrega los workloads como servicio gestionado, sin exigir administrar toda la infraestructura subyacente.
2. **¿OneLake y Lakehouse significan lo mismo?** No: uno es almacenamiento unificado, otro un objeto analítico.
3. **¿Qué agrupa un Workspace?** Elementos, colaboradores y controles de organización.
4. **¿Qué problema resuelve un shortcut?** Integrar determinadas ubicaciones de datos sin replicarlos obligatoriamente.
5. **¿Quién debe aprobar el acceso a un dataset sensible?** El gobierno de datos y las identidades autorizadas, con las políticas corporativas.
6. **¿Por qué diferentes equipos deben acordar contratos de datos?** Para controlar esquemas y reglas de negocio que alimentan múltiples consumidores.
7. **¿Copilot sustituye la revisión de SQL generado?** No, el código y los datos deben validarse.
8. **¿Todo tenant ofrece todas las capacidades?** No, dependen de configuración, permisos y capacidad.
9. **¿Qué diferencia hay entre un pipeline y un modelo semántico?** El primero ejecuta tareas, el segundo define consumo y semántica analítica.
10. **¿Qué registrarías antes de realizar el laboratorio?** Capacidad, nombre de Workspace, permisos, pasos y posibles costes.

## 7. Summary y fuentes

Fabric unifica cargas de datos y colaboración sobre OneLake y Workspaces. La selección del motor depende del caso, no de una equivalencia indiscriminada entre servicios. Una arquitectura sólida necesita contratos, calidad, seguridad, trazabilidad y control del coste aun cuando la infraestructura sea SaaS.

**Unidades cubiertas:** Introduction, Explore analytics, Data team, Use Fabric, Knowledge check y Summary. Se consultó el [index.yml oficial](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/introduction-end-analytics-use-microsoft-fabric/index.yml) y las tres unidades centrales, conservando sus enlaces específicos. El ejercicio es **propio**, no oficial y no ejecutado.