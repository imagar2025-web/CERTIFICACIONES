# DP-700 · Microsoft Certified: Fabric Data Engineer Associate

> **Estado del manual:** entrega parcial y trazable. **8 de 21 módulos** con README de estudio publicado; 13 pendientes de desarrollo profundo. **Fecha de revisión:** noviembre de 2026. No se han ejecutado los laboratorios en un tenant de Fabric.

## Qué incluye esta documentación

Manual técnico en español de España, con nombres de servicios y comandos en inglés. Los módulos preparados incluyen arquitectura, ejemplos reproducibles, consideraciones de costes, seguridad, verificaciones y laboratorios explicados. **Los materiales aún no desarrollados aparecen identificados como pendientes; no se incluyen resúmenes vacíos ni se declara terminada la certificación.**

La formación utiliza cinco rutas oficiales con **26 apariciones de módulos, 21 distintos**. Un módulo compartido solo tiene un archivo para evitar divergencias. Los identificadores 01–21 son numeración de esta biblioteca y no números oficiales de Microsoft.

## Rutas de aprendizaje oficiales

### 1. [Ingest data with Microsoft Fabric](https://learn.microsoft.com/en-us/training/paths/ingest-data-with-microsoft-fabric/)

1. [Ingest Data with Dataflows Gen2 in Microsoft Fabric](01-dataflows-gen2/README.md)
2. [Orchestrate processes and data movement with Microsoft Fabric](02-orquestacion-data-factory/README.md)
3. [Use Apache Spark in Microsoft Fabric](03-apache-spark/README.md)
4. [Work with real-time data in an Eventhouse in Microsoft Fabric](04-eventhouse-kql/README.md)

### 2. [Implement a Lakehouse with Microsoft Fabric](https://learn.microsoft.com/en-us/training/paths/implement-lakehouse-microsoft-fabric/)

1. [Introduction to end-to-end analytics using Microsoft Fabric](05-introduccion-microsoft-fabric/README.md)
2. [Get started with lakehouses in Microsoft Fabric](06-lakehouse/README.md)
3. [Use Apache Spark in Microsoft Fabric](03-apache-spark/README.md)
4. [Work with Delta Lake tables in Microsoft Fabric](07-delta-lake/README.md)
5. [Ingest Data with Dataflows Gen2 in Microsoft Fabric](01-dataflows-gen2/README.md)
6. [Orchestrate processes and data movement with Microsoft Fabric](02-orquestacion-data-factory/README.md)
7. [Organize a Fabric lakehouse using medallion architecture design](08-arquitectura-medallion/README.md)

### 3. [Implement Real-Time Intelligence with Microsoft Fabric](https://learn.microsoft.com/en-us/training/paths/explore-real-time-analytics-microsoft-fabric/)

1. [Get started with Real-Time Intelligence in Microsoft Fabric](https://learn.microsoft.com/en-us/training/modules/get-started-kusto-fabric/) **(pendiente de desarrollo)**
2. [Use Eventstream in Microsoft Fabric](https://learn.microsoft.com/en-us/training/modules/explore-event-streams-microsoft-fabric/) **(pendiente de desarrollo)**
3. [Work with real-time data in an Eventhouse in Microsoft Fabric](04-eventhouse-kql/README.md)
4. [Create Real-Time Dashboards with Microsoft Fabric](https://learn.microsoft.com/en-us/training/modules/create-real-time-dashboards-microsoft-fabric/) **(pendiente de desarrollo)**
5. [Use Activator in Microsoft Fabric](https://learn.microsoft.com/en-us/training/modules/use-fabric-activator/) **(pendiente de desarrollo)**

### 4. [Implement a data warehouse with Microsoft Fabric](https://learn.microsoft.com/en-us/training/paths/work-with-data-warehouses-using-microsoft-fabric/)

1. [Introduction to end-to-end analytics using Microsoft Fabric](https://learn.microsoft.com/en-us/training/modules/introduction-end-analytics-use-microsoft-fabric/) **(pendiente de desarrollo)**
2. [Get started with data warehouses in Microsoft Fabric](https://learn.microsoft.com/en-us/training/modules/get-started-data-warehouse/) **(pendiente de desarrollo)**
3. [Load data into a Microsoft Fabric data warehouse](https://learn.microsoft.com/en-us/training/modules/load-data-into-microsoft-fabric-data-warehouse/) **(pendiente de desarrollo)**
4. [Query a data warehouse in Microsoft Fabric](https://learn.microsoft.com/en-us/training/modules/query-data-warehouse-microsoft-fabric/) **(pendiente de desarrollo)**
5. [Get started with Copilot in Microsoft Fabric for Data Warehouse](https://learn.microsoft.com/en-us/training/modules/get-started-copilot-fabric-data-warehouse/) **(pendiente de desarrollo)**
6. [Monitor a Microsoft Fabric data warehouse](https://learn.microsoft.com/en-us/training/modules/monitor-fabric-data-warehouse/) **(pendiente de desarrollo)**
7. [Secure a Microsoft Fabric data warehouse](https://learn.microsoft.com/en-us/training/modules/secure-data-warehouse-in-microsoft-fabric/) **(pendiente de desarrollo)**

### 5. [Manage a Microsoft Fabric environment](https://learn.microsoft.com/en-us/training/paths/manage-microsoft-fabric-environment/)

1. [Implement CI/CD in Microsoft Fabric](https://learn.microsoft.com/en-us/training/modules/implement-cicd-in-fabric/) **(pendiente de desarrollo)**
2. [Monitor activities in Microsoft Fabric](https://learn.microsoft.com/en-us/training/modules/monitor-fabric-items/) **(pendiente de desarrollo)**
3. [Secure data access in Microsoft Fabric](https://learn.microsoft.com/en-us/training/modules/secure-data-access-in-fabric/) **(pendiente de desarrollo)**

## Control de progreso

| Estado | Recuento | Observación |
| --- | ---: | --- |
| README de estudio publicados | 8 | Dataflows Gen2, Data Factory Pipelines, Apache Spark, Eventhouse KQL, fundamentos Fabric, Lakehouse, Delta Lake y Medallion |
| Módulos pendientes de redacción exhaustiva | 13 | Se conservan enlaces al contenido oficial hasta desarrollarlos |
| Laboratorios documentados en los módulos preparados | 7 | Seis asociados a DP-700, uno complementario de DP-603 y un ejercicio propio introductorio |
| Laboratorios ejecutados por este agente | 0 | No hay resultados medidos ni capturas de ejecución |

Para seguimiento por unidad y laboratorio, consultar [PROGRESO.md](PROGRESO.md).

## Archivos generales

- [GLOSARIO.md](GLOSARIO.md): términos del ecosistema de ingeniería de datos en Fabric.
- [FUENTES.md](FUENTES.md): URLs oficiales de certificación, rutas, material fuente y laboratorios.
- [PROGRESO.md](PROGRESO.md): control de cobertura, estado por módulo, limitaciones y trabajos pendientes.

## Criterios editoriales

1. Un README por módulo con todas sus unidades, en lugar de documentos separados por unidad.
2. Priorizar explicación extensa y documentación original Microsoft Learn y MicrosoftLearning.
3. Distinguir código de elaboración propia del contenido y los pasos oficiales.
4. No inventar resultados de ejercicios ni asumir compatibilidad de opciones de distintas capacidades de Fabric.
5. Publicar módulos adicionales solo cuando su desarrollo técnico y laboratorio hayan sido revisados.
6. Actualizar el índice y el registro de progreso al incorporar un módulo.

## Referencias de examen

- [Certificación DP-700](https://learn.microsoft.com/en-us/credentials/certifications/fabric-data-engineer-associate/)
- [Guía del examen](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-700)
- [Curso DP-700T00-A](https://learn.microsoft.com/en-us/training/courses/dp-700t00)
- [Repositorio de documentación MicrosoftDocs/learn](https://github.com/MicrosoftDocs/learn)
- [Laboratorios MicrosoftLearning/mslearn-fabric](https://github.com/MicrosoftLearning/mslearn-fabric)
