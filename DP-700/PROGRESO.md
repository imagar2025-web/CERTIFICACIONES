# Progreso de documentación · DP-700

**Corte:** 24 de noviembre de 2026. **Inventario oficial:** cinco rutas, 26 apariciones y 21 módulos distintos.

## Estado resumido

- **21/21 README publicados** en GitHub; existe al menos una versión técnica explicativa y ejercicio asociado de cada módulo.
- **0/21 laboratorios/ejercicios ejecutados** en un tenant real; las instrucciones y verificaciones son documentales.
- **Revisión editorial exhaustiva pendiente.** La muestra Dataflows Gen2 es considerablemente más extensa que los otros módulos. No declaramos que todos los archivos hayan alcanzado la profundidad de esa muestra; requieren ampliación de cada unidad y de los procedimientos de laboratorio, además de revisión de código.
- **DP-750 conservada.** La nueva certificación está aislada bajo `DP-700/` y enlazada desde el README principal.

## Matriz de módulos

| Nº | Módulo | Estado de README | Laboratorio o ejercicio | Ejecución |
| --- | --- | --- | --- | --- |
| 01 | [Ingest Data with Dataflows Gen2 in Microsoft Fabric](01-dataflows-gen2/README.md) | Publicado · muestra amplia | 05-dataflows-gen2 (DP-700) | Documentado, **no ejecutado** |
| 02 | [Orchestrate processes and data movement with Microsoft Fabric](02-orquestacion-data-factory/README.md) | Publicado · borrador técnico | 04-ingest-pipeline (DP-700) | Documentado, **no ejecutado** |
| 03 | [Use Apache Spark in Microsoft Fabric](03-apache-spark/README.md) | Publicado · borrador técnico | 02-analyze-spark (DP-700) | Documentado, **no ejecutado** |
| 04 | [Work with real-time data in an Eventhouse in Microsoft Fabric](04-eventhouse-kql/README.md) | Publicado · borrador técnico | 12-query-data-in-kql-database (complementario DP-603) | Documentado, **no ejecutado** |
| 05 | [Introduction to end-to-end analytics using Microsoft Fabric](05-introduccion-microsoft-fabric/README.md) | Publicado · borrador técnico | Ejercicio propio; no hay lab específico en índice | Documentado, **no ejecutado** |
| 06 | [Get started with lakehouses in Microsoft Fabric](06-lakehouse/README.md) | Publicado · borrador técnico | 01-lakehouse (DP-700) | Documentado, **no ejecutado** |
| 07 | [Work with Delta Lake tables in Microsoft Fabric](07-delta-lake/README.md) | Publicado · borrador técnico | 03-delta-lake (DP-700) | Documentado, **no ejecutado** |
| 08 | [Organize a Fabric lakehouse using medallion architecture design](08-arquitectura-medallion/README.md) | Publicado · borrador técnico | 03b-medallion-lakehouse (DP-700) | Documentado, **no ejecutado** |
| 09 | [Get started with Real-Time Intelligence in Microsoft Fabric](09-introduccion-real-time-intelligence/README.md) | Publicado · borrador técnico | 07-real-time-Intelligence (DP-700) | Documentado, **no ejecutado** |
| 10 | [Use Eventstream in Microsoft Fabric](10-eventstreams/README.md) | Publicado · borrador técnico | 09-real-time-analytics-eventstream (DP-700) | Documentado, **no ejecutado** |
| 11 | [Create Real-Time Dashboards with Microsoft Fabric](11-real-time-dashboards/README.md) | Publicado · borrador técnico | 13-real-time-dashboards (complementario DP-603) | Documentado, **no ejecutado** |
| 12 | [Use Activator in Microsoft Fabric](12-activator/README.md) | Publicado · borrador técnico | 11-data-activator (complementario DP-603) | Documentado, **no ejecutado** |
| 13 | [Get started with data warehouses in Microsoft Fabric](13-introduccion-data-warehouse/README.md) | Publicado · borrador técnico | 06-data-warehouse (DP-700) | Documentado, **no ejecutado** |
| 14 | [Load data into a Microsoft Fabric data warehouse](14-carga-data-warehouse/README.md) | Publicado · borrador técnico | 06a-data-warehouse-load (DP-700) | Documentado, **no ejecutado** |
| 15 | [Query a data warehouse in Microsoft Fabric](15-consultas-data-warehouse/README.md) | Publicado · borrador técnico | 06b-data-warehouse-query (DP-700) | Documentado, **no ejecutado** |
| 16 | [Get started with Copilot in Microsoft Fabric for Data Warehouse](16-copilot-data-warehouse/README.md) | Publicado · borrador técnico | 22c-copilot-fabric-data-warehouse (complementario; verificar asociación de curso) | Documentado, **no ejecutado** |
| 17 | [Monitor a Microsoft Fabric data warehouse](17-monitorizacion-data-warehouse/README.md) | Publicado · borrador técnico | 06c-monitor-data-warehouse (DP-700) | Documentado, **no ejecutado** |
| 18 | [Secure a Microsoft Fabric data warehouse](18-seguridad-data-warehouse/README.md) | Publicado · borrador técnico | 06d-secure-data-warehouse (DP-700) | Documentado, **no ejecutado** |
| 19 | [Implement CI/CD in Microsoft Fabric](19-cicd/README.md) | Publicado · borrador técnico | 21-implement-cicd (DP-700) | Documentado, **no ejecutado** |
| 20 | [Monitor activities in Microsoft Fabric](20-monitorizacion-fabric/README.md) | Publicado · borrador técnico | 18-monitor-hub (complementario; sin DP-700 en metadatos) | Documentado, **no ejecutado** |
| 21 | [Secure data access in Microsoft Fabric](21-seguridad-fabric/README.md) | Publicado · borrador técnico | 19-secure-data-access (DP-700) | Documentado, **no ejecutado** |

## Cobertura de fuentes y control técnico

Se localizaron los 21 módulos oficiales y sus unidades mediante Microsoft Learn y los índices `index.yml`, incluyendo el directorio alternativo `learn-pr/wwl-data-ai` para determinados módulos de Data Warehouse y administración. Se contrastaron archivos fuente `includes/*.md` de contenidos centrales, y las instrucciones de los laboratorios relevantes en el repositorio MicrosoftLearning.

**No es una auditoría exhaustiva ni un certificado de ejecución:** antes de considerar definitivo un módulo se debe cotejar toda unidad y paso del laboratorio con el contenido vigente, comprobar compatibilidad de código con un Fabric tenant, ampliar explicaciones que hayan quedado abreviadas y verificar sus vínculos internos y figuras necesarias.

## Prioridades de revisión

1. **Ampliación editorial de módulos 02–21:** extender las unidades breves hasta el detalle de la muestra aprobada, especialmente los procedimientos de laboratorio, problemas frecuentes y seguridad/costes.
2. **Verificación en entorno real:** ejecutar laboratorios, documentar tiempos, permisos, capturas y resultados observados, sin alterar recursos productivos.
3. **Revisión técnica:** contrastar API y sintaxis de T-SQL, PySpark, M y KQL que hayan evolucionado, revisar dependencias de esquemas y permisos.
4. **Contenido visual:** añadir imágenes locales cuando aporten valor y sea posible distribuirlas respetando licencias; los diagramas Mermaid propios ya funcionan como apoyo conceptual.
5. **Mantenimiento de examen:** comprobar cambios en la guía oficial de DP-700 y actualizar objetivos, referencias y temario.

## Criterio de finalización

**README publicado** significa que existe un documento con desarrollo técnico inicial; **módulo aprobado** exigirá cobertura exhaustiva de unidades, procedimiento de laboratorio completo, códigos revisados y las fuentes verificadas. En esta entrega **ningún módulo se declara validado mediante ejecución** y la biblioteca global sigue **en revisión editorial**.
