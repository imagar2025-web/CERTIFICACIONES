# 📘 DP-750 · Guía de estudio de Azure Databricks

> **Apuntes personales extendidos, versión ampliada 15 de noviembre de 2026** · Estructura modular para VS Code y GitHub.

Esta documentación desarrolla en español los **primeros 13 módulos del recorrido formativo para DP-750**, con desarrollos adicionales sobre el funcionamiento interno y escenarios de aplicación. Cada módulo tiene su propio `README.md`, con teoría, ejemplos, comprobaciones y una guía de laboratorio enlazada con el material oficial cuando existe.

**Origen y atribución:** elaboración propia a partir de las páginas de Microsoft Learn y el material público de MicrosoftLearning. No constituye una copia o traducción integral de Microsoft Learn; para el texto exacto, las celdas oficiales y las evaluaciones autenticadas sigue los enlaces proporcionados. Los ejercicios aquí explicados no se han ejecutado en un workspace real.

## 🧭 Menú principal — 13 módulos

| # | Módulo | Estado | Contenido | Unidades |
|---:|---|---|---|---:|
| 01 | [Explorar Azure Databricks](01-explorar-azure-databricks/README.md) | ✅ Visto | Vista del workspace, volumen y notebooks | 8 |
| 02 | [Arquitectura de Azure Databricks](02-arquitectura-de-azure-databricks/README.md) | ✅ Visto | Control plane, cómputo y almacenamiento | 7 |
| 03 | [Integraciones de Azure Databricks](03-integraciones-de-azure-databricks/README.md) | ✅ Visto | Fabric, Power BI, VS Code y otras integraciones | 10 |
| 04 | [Selección y configuración de cómputo](04-seleccion-y-configuracion-de-computo/README.md) | ✅ Visto | Clusters, bibliotecas y costes | 9 |
| 05 | [Crear y organizar objetos en Unity Catalog](05-crear-y-organizar-objetos-en-unity-catalog/README.md) | ✅ Visto | Catálogos, esquemas, tablas y volúmenes | 12 |
| 06 | [Proteger objetos de Unity Catalog](06-proteger-objetos-de-unity-catalog/README.md) | ✅ Visto | Permisos, filtros, máscaras y secretos | 11 |
| 07 | [Gobernar objetos de Unity Catalog](07-gobernar-objetos-de-unity-catalog/README.md) | ✅ Visto | Etiquetas, retención, linaje y auditoría | 10 |
| 08 | [Diseñar e implementar el modelado de datos](08-modelado-de-datos/README.md) | ✅ Visto | SCD 2, CDF, time travel y clustering | 14 |
| 09 | [Ingerir datos en Unity Catalog](09-ingesta-de-datos/README.md) | ✅ Visto | DataFrames, COPY INTO, CTAS y Auto Loader | 11 |
| 10 | [Limpiar, transformar y cargar datos](10-limpieza-transformacion-y-carga/README.md) | ✅ Visto | Calidad, joins, pivot y merge | 11 |
| 11 | [Implementar controles de calidad de datos](11-calidad-de-datos/README.md) | 🕒 Pendiente | Validaciones, schema drift y expectations | 8 |
| 12 | [Diseñar e implementar pipelines de datos](12-diseno-de-pipelines/README.md) | 🕒 Pendiente | Medallion, jobs, dependencias y errores | 10 |
| 13 | [Implementar Lakeflow Jobs](13-implementacion-lakeflow-jobs/README.md) | 🕒 Pendiente | Triggers, cron, alertas y reintentos | 9 |

## 🗂️ Organización por áreas de la certificación

Las páginas oficiales organizan el itinerario en **cuatro rutas de aprendizaje**:

1. **Configuración de Azure Databricks (01–05):** plataforma, arquitectura, integraciones, cómputo y organización de activos.
2. **Seguridad y gobierno (06–07):** permisos, filtrado, máscaras, secretos, linaje, políticas y auditoría.
3. **Preparación y procesamiento (08–11):** modelado, ingesta, limpieza y calidad.
4. **Despliegue y mantenimiento (12–15):** pipelines, Lakeflow Jobs y —en los dos módulos siguientes— ciclo de desarrollo y supervisión.

## 🧪 Material práctico

- [Preparación del entorno y precauciones de costes](PREPARACION-LABORATORIOS.md)
- [Colección oficial MicrosoftLearning DP-750T00](https://microsoftlearning.github.io/DP-750T00-Implement-Data-Engineering-Solutions-using-Azure-Databricks/)
- [Glosario, mapa mental y preguntas de repaso](GLOSARIO-Y-REPASO.md)
- [Archivos fuente y metodología](FUENTES-Y-METODOLOGIA.md)

Cada guía práctica indica **qué parte se hace en la interfaz** y **qué parte continúa en el notebook oficial**. Para reproducir exactamente los datasets, código, ejercicios incompletos o pasos de interfaz de la versión publicada por Microsoft, utiliza el enlace de cada módulo.

## ✅ Progreso personal

- [x] Módulos 01–10: contenidos que ya has visto, incluidos y desarrollados en esta versión.
- [ ] Módulo 11: calidad de datos y manejo de schema drift.
- [ ] Módulo 12: diseño e implementación de pipelines.
- [ ] Módulo 13: implementación y operación de Lakeflow Jobs.

## 🔜 Ampliaciones posteriores (fuera de esta entrega)

El itinerario completo **continúa después del 13**. No he sustituido esos módulos por resúmenes incompletos ni los he marcado como estudiados:

14. [Implement development lifecycle processes in Azure Databricks](https://learn.microsoft.com/en-us/training/modules/implement-development-lifecycle-processes-in-azure-databricks/) — Git, pruebas, Databricks Asset Bundles y despliegue.
15. [Monitor, troubleshoot and optimize workloads in Azure Databricks](https://learn.microsoft.com/en-us/training/modules/monitor-troubleshoot-optimize-workloads-azure-databricks/) — Spark UI, skew, shuffle, Delta y monitorización.

## 📂 Estructura de carpetas

```text
DP-750/
├── README.md                      ← menú principal
├── PREPARACION-LABORATORIOS.md
├── GLOSARIO-Y-REPASO.md
├── 01-explorar-azure-databricks/
│   └── README.md
├── 02-arquitectura-de-azure-databricks/
│   └── README.md
├── 03-integraciones-de-azure-databricks/
│   └── README.md
├── 04-seleccion-y-configuracion-de-computo/
│   └── README.md
├── 05-crear-y-organizar-objetos-en-unity-catalog/
│   └── README.md
├── 06-proteger-objetos-de-unity-catalog/
│   └── README.md
├── 07-gobernar-objetos-de-unity-catalog/
│   └── README.md
├── 08-modelado-de-datos/
│   └── README.md
├── 09-ingesta-de-datos/
│   └── README.md
├── 10-limpieza-transformacion-y-carga/
│   └── README.md
├── 11-calidad-de-datos/
│   └── README.md
├── 12-diseno-de-pipelines/
│   └── README.md
├── 13-implementacion-lakeflow-jobs/
│   └── README.md
```

## 📖 Uso de los archivos fuente

Los materiales de Microsoft Learn se alojan en [MicrosoftDocs/learn, directorio `learn-pr/wwl-databricks`](https://github.com/MicrosoftDocs/learn/tree/main/learn-pr/wwl-databricks). En su estructura, los archivos `.yml` describen las unidades y los archivos `includes/*.md` contienen gran parte de la explicación técnica y las referencias a imágenes. Este proyecto es una **elaboración propia en español**: no incorpora ni redistribuye el texto completo de Microsoft. Cada módulo conserva el enlace a la página original; para verificar instrucciones visuales exactas, consulta sus archivos fuente y la página Learn.

Las prácticas incorporadas son **guías de ejecución y comprobación**; los notebooks enlazados son los materiales canónicos de Microsoft. No se han ejecutado dentro de un workspace real ni se certifica que todas las rutas o parámetros de ejemplo estén disponibles sin cambios en cada suscripción.

## 📚 Fuentes oficiales

- [Guía oficial del examen DP-750](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-750)
- [Ruta: configurar Azure Databricks](https://learn.microsoft.com/en-us/training/paths/azure-databricks-data-engineer-set-up-configure-environment/)
- [Ruta: asegurar y gobernar Unity Catalog](https://learn.microsoft.com/en-us/training/paths/azure-databricks-data-engineer-secure-govern-unity-catalog/)
- [Ruta: preparar y procesar datos](https://learn.microsoft.com/en-us/training/paths/azure-databricks-data-engineer-prepare-process-data/)
- [Ruta: desplegar y mantener pipelines](https://learn.microsoft.com/en-us/training/paths/azure-databricks-data-engineer-deploy-maintain-data-pipelines-workloads/)
- [Laboratorios oficiales DP-750T00](https://microsoftlearning.github.io/DP-750T00-Implement-Data-Engineering-Solutions-using-Azure-Databricks/)

---

> **Mantenimiento:** como Databricks evoluciona con rapidez, comprueba compatibilidad de Runtime, permisos y cambios de interfaz antes de ejecutar un laboratorio o realizar el examen.
