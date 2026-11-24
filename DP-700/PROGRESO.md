# Progreso de documentación · DP-700

**Corte:** 24 de noviembre de 2026. **Alcance total:** 21 módulos distintos en cinco rutas oficiales (26 apariciones). No se considera terminada la certificación hasta completar desarrollo y revisión de todos.

## Matriz de módulos

| Módulo | Tema | Documento | Laboratorio | Estado |
| --- | --- | --- | --- | --- |
| 01 | [Ingest Data with Dataflows Gen2 in Microsoft Fabric](01-dataflows-gen2/README.md) | Publicado | 05-dataflows-gen2 | Teoría y procedimiento documentados; ejercicio NO ejecutado |
| 02 | [Orchestrate processes and data movement with Microsoft Fabric](02-orquestacion-data-factory/README.md) | Publicado | 04-ingest-pipeline | Teoría y procedimiento documentados; ejercicio NO ejecutado |
| 03 | [Use Apache Spark in Microsoft Fabric](03-apache-spark/README.md) | Publicado | 02-analyze-spark | Teoría y procedimiento documentados; ejercicio NO ejecutado |
| 04 | [Work with real-time data in an Eventhouse in Microsoft Fabric](04-eventhouse-kql/README.md) | Publicado | 12-query-data-in-kql-database (complementario) | Teoría y procedimiento documentados; ejercicio NO ejecutado |
| 05 | Introduction to end-to-end analytics using Microsoft Fabric | Pendiente | Pendiente de asociación y revisión | Pendiente |
| 06 | Get started with lakehouses in Microsoft Fabric | Pendiente | Pendiente de asociación y revisión | Pendiente |
| 07 | Work with Delta Lake tables in Microsoft Fabric | Pendiente | Pendiente de asociación y revisión | Pendiente |
| 08 | Organize a Fabric lakehouse using medallion architecture design | Pendiente | Pendiente de asociación y revisión | Pendiente |
| 09 | Get started with Real-Time Intelligence in Microsoft Fabric | Pendiente | Pendiente de asociación y revisión | Pendiente |
| 10 | Use Eventstream in Microsoft Fabric | Pendiente | Pendiente de asociación y revisión | Pendiente |
| 11 | Create Real-Time Dashboards with Microsoft Fabric | Pendiente | Pendiente de asociación y revisión | Pendiente |
| 12 | Use Activator in Microsoft Fabric | Pendiente | Pendiente de asociación y revisión | Pendiente |
| 13 | Get started with data warehouses in Microsoft Fabric | Pendiente | Pendiente de asociación y revisión | Pendiente |
| 14 | Load data into a Microsoft Fabric data warehouse | Pendiente | Pendiente de asociación y revisión | Pendiente |
| 15 | Query a data warehouse in Microsoft Fabric | Pendiente | Pendiente de asociación y revisión | Pendiente |
| 16 | Get started with Copilot in Microsoft Fabric for Data Warehouse | Pendiente | Pendiente de asociación y revisión | Pendiente |
| 17 | Monitor a Microsoft Fabric data warehouse | Pendiente | Pendiente de asociación y revisión | Pendiente |
| 18 | Secure a Microsoft Fabric data warehouse | Pendiente | Pendiente de asociación y revisión | Pendiente |
| 19 | Implement CI/CD in Microsoft Fabric | Pendiente | Pendiente de asociación y revisión | Pendiente |
| 20 | Monitor activities in Microsoft Fabric | Pendiente | Pendiente de asociación y revisión | Pendiente |
| 21 | Secure data access in Microsoft Fabric | Pendiente | Pendiente de asociación y revisión | Pendiente |
 
## Verificaciones completadas

- Rutas oficiales: cinco, contrastadas con Microsoft Learn.
- Módulos diferentes: 21, deduplicados por identidad de módulo.
- Archivo original `index.yml` y unidades principales revisados para módulos 01–04.
- Laboratorios del repositorio MicrosoftLearning leídos y asociados a módulos 01–04, marcando el cuarto como ejercicio **complementario DP-603**.
- Documentos creados en su propio directorio `DP-700/`, sin alterar la documentación DP-750.

## Limitaciones y trabajo pendiente

- **Módulos 05–21:** desarrollar todas sus unidades, consultar includes/README y laboratorios oficiales y añadir pruebas, seguridad, costes, ejemplos y diagramas al nivel aprobado.
- **Módulos 01–04:** antes de declararlos editorialmente definitivos, realizar revisión final contra toda actualización de Microsoft Learn, validar los ejemplos en un tenant y verificar cada actividad de laboratorio.
- **Laboratorios:** no hay ejecuciones reales, capturas ni tiempos medidos; no presentar resultados simulados como observados.
- **Imágenes locales:** pendientes de incorporar cuando haya figuras necesarias y permisos de reutilización; los README actuales emplean diagramas Mermaid.
- **Examen:** mantener la matriz de objetivos de la study guide actualizada, especialmente cuando cambien las competencias evaluadas.

## Regla de avance

No confundir «entrada en el índice» con «manual terminado»: los 17 módulos pendientes están enlazados a la documentación oficial, pero no son archivos de estudio redactados. Una próxima entrega debe incorporar módulos completos de un bloque y actualizar esta matriz.
