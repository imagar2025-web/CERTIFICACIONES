[🏠 Índice](../README.md) · [← Anterior](../02-arquitectura-de-azure-databricks/README.md) · [Siguiente →](../04-seleccion-y-configuracion-de-computo/README.md)

> **Estado de estudio:** ✅ Visto · **Unidades oficiales:** 10 · [Microsoft Learn](https://learn.microsoft.com/en-us/training/modules/understand-azure-databricks-integrations/)  
> **Versión de apuntes:** 14-11-2026 · Contenido técnico redactado de forma original a partir del itinerario de Microsoft.

# 3. Integraciones de Azure Databricks

**Referencia Microsoft Learn:** [Integraciones de Azure Databricks](https://learn.microsoft.com/en-us/training/modules/understand-azure-databricks-integrations/) · **10 unidades oficiales**.

## 3.1. Introducción: la plataforma no vive aislada

Databricks transforma y gobierna datos; otras herramientas permiten consumirlos, automatizar acciones, producir informes o construir agentes de IA. Diseñar una integración exige responder cuatro preguntas: ¿los datos se **copian o consultan**?, ¿quién ejecuta las consultas?, ¿qué identidad se utiliza? y ¿dónde se aplican realmente los permisos?

## 3.2. Microsoft Fabric y OneLake

Microsoft Fabric agrupa servicios de ingeniería, análisis y BI en una experiencia unificada. Databricks se especializa en procesamiento, Lakehouse y cargas de IA. Se pueden complementar de dos formas principales:

1. **Fabric accede a tablas Delta registradas en Unity Catalog** mediante catálogos reflejados y accesos directos (*shortcuts*) a los datos de origen. Al no copiar necesariamente los archivos, se reducen duplicidades; sin embargo, es imprescindible comprender el modelo de identidad y los permisos efectivos en Fabric.
2. **Databricks escribe resultados en OneLake**, usando rutas compatibles con ABFS, para que Fabric los analice o visualice. Se considera quién posee permisos de escritura, la región, la modalidad de autenticación y los costes de transferencia.

```text
Fuentes → Databricks (limpieza + transformación) → OneLake / Fabric → Power BI
                        └─────── Unity Catalog (metadatos y gobierno)
```

**Advertencia de seguridad:** al exponer datos de Unity Catalog a otra plataforma, no debes asumir que filtros de filas, máscaras de columnas o la identidad del usuario original se transmiten intactos. La unidad oficial señala restricciones relevantes de la opción de catálogo reflejado. Una integración no debe configurarse como si fuera siempre un DirectQuery con seguridad extremo a extremo.

## 3.3. Power BI: Import y DirectQuery

Power BI se conecta al endpoint de un **SQL warehouse** u otro cómputo compatible. En modo **Import**, incorpora una copia de datos en el modelo semántico que necesita refrescos periódicos: permite análisis rápidos, pero crea otra copia que debe protegerse. En **DirectQuery**, manda consultas al origen cuando el usuario interactúa; mantiene datos más cercanos al estado actual, pero afecta a latencia, concurrencia y costes del warehouse.

| Característica | Import | DirectQuery |
|---|---|---|
| Dónde se almacenan los datos del modelo | En Power BI | Principalmente en Databricks |
| Actualización | Refresco | Consultas en uso |
| Rendimiento | Suele ser rápido para modelo optimizado | Depende de Databricks y del diseño |
| Gobierno | Revisar reglas de la copia | Revisar autenticación y permisos delegados |

La conexión incluye hostname del servidor, ruta HTTP y autenticación; un SQL warehouse adecuado y datos bien modelados suelen ser preferibles a exponer tablas de ingestión crudas.

## 3.4. VS Code para desarrollar localmente

La extensión de Databricks para VS Code permite trabajar con un proyecto local, autenticarse contra un workspace y desplegar o ejecutar código en recursos remotos. Este esquema separa la **experiencia de edición local** de la **ejecución sobre un entorno distribuido**. No todas las funciones del notebook web se trasladan exactamente al editor local: conviene comprobar la configuración del proyecto y del cómputo.

Flujo recomendado: abrir el repositorio Git → configurar perfil de Databricks → instalar dependencias en un entorno reproducible → asociar el proyecto al workspace → ejecutar pruebas y trabajos → subir cambios de código a Git, evitando incluir tokens.

## 3.5. Power Platform

La Power Platform incluye herramientas como Power Apps y Power Automate. Los datos preparados en Databricks pueden alimentar aplicaciones de negocio y procesos automatizados mediante conectores, APIs o capas de acceso compatibles. Antes de conectar una aplicación, es importante analizar qué operaciones están permitidas, si se pueden escribir datos, cómo se autentica cada aplicación y cómo se limita el acceso a los usuarios finales.

**Ejemplo de arquitectura:** Databricks consolida métricas de clientes; una aplicación empresarial muestra sus indicadores y lanza un flujo de aprobación. Es preferible entregar una tabla o vista diseñada para ese consumo y no una tabla raw con datos sensibles innecesarios.

## 3.6. Copilot Studio

Copilot Studio permite definir agentes conversacionales y acciones. Puede aprovechar información que reside en Databricks a través de mecanismos admitidos por la integración. Un agente no debería recibir acceso indiscriminado a todo un catálogo: hay que diseñar identidades, controles de acceso, límites de consulta, calidad de las respuestas y supervisión de uso.

Diferencia importante: **un agente que consulta datos** no equivale a **un usuario humano con permisos heredados automáticamente**. El resultado depende del mecanismo concreto de conexión y de la identidad que ejecute la consulta.

## 3.7. Microsoft Purview

**Unity Catalog** gobierna activos dentro del ecosistema Databricks: permisos, metadatos, linaje y auditoría. **Microsoft Purview** ayuda a descubrir, catalogar y gobernar activos en un paisaje de datos más amplio de Microsoft y otras fuentes. Integrarlos puede ampliar la visibilidad corporativa, pero no reemplaza la configuración de permisos en Databricks.

Una organización puede usar Unity Catalog como punto de aplicación de las reglas de acceso y Purview como un plano transversal de descubrimiento y análisis de activos. **Catalogar** no significa **autorizar**.

## 3.8. Microsoft Foundry y agentes

Microsoft Foundry aporta herramientas para construir soluciones y agentes de IA. Databricks puede servir como origen analítico y semántico mediante funciones admitidas, entre ellas espacios de **AI/BI Genie**. Genie permite formular preguntas de negocio en lenguaje natural sobre conjuntos de datos definidos; su calidad depende de la descripción de tablas, las relaciones, las instrucciones y los permisos.

**Diseño de referencia:** un agente recibe una pregunta → invoca una herramienta autorizada que consulta un espacio Genie → obtiene resultados fundamentados → explica la respuesta con los límites de los datos disponibles. Aun existiendo IA, se requieren controles contra consultas no deseadas y divulgación de información sensible.

## 3.9. Evaluación y síntesis

- ¿Qué herramienta presentaría un panel empresarial? **Power BI**.
- ¿Qué facilita desarrollar código local y ejecutarlo remotamente? **Extensión Databricks para VS Code**.
- ¿Qué diferencia a Purview de Unity Catalog? **Gobierno transversal frente a administración y control de acceso de activos Databricks**.
- ¿Qué debe revisarse antes de conectar Fabric? **Identidad, seguridad efectiva, formatos admitidos y posible transferencia de datos**.

---

---

## 🧪 Guía de laboratorio y comprobación práctica

**Este módulo no presenta un laboratorio independiente en la colección DP-750T00 consultada.** Para consolidar la teoría, realiza una actividad de arquitectura propia: dibuja los componentes e integraciones descritos y justifica dónde se ejecuta el trabajo, cómo acceden las identidades y qué datos permanecen en cada servicio. [Índice de laboratorios oficiales](https://microsoftlearning.github.io/DP-750T00-Implement-Data-Engineering-Solutions-using-Azure-Databricks/)


---

### 📚 Fuentes y navegación

- [Contenido oficial de Microsoft Learn](https://learn.microsoft.com/en-us/training/modules/understand-azure-databricks-integrations/)
- [Repositorio docente de laboratorios DP-750T00](https://microsoftlearning.github.io/DP-750T00-Implement-Data-Engineering-Solutions-using-Azure-Databricks/)
- [Guía oficial de competencias del examen DP-750](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-750)

[🏠 Índice](../README.md) · [← Anterior](../02-arquitectura-de-azure-databricks/README.md) · [Siguiente →](../04-seleccion-y-configuracion-de-computo/README.md)
