[🏠 Índice](../README.md) · [← Anterior](../06-proteger-objetos-de-unity-catalog/README.md) · [Siguiente →](../08-modelado-de-datos/README.md)

> **Estado de estudio:** ✅ Visto · **Unidades oficiales:** 10 · [Microsoft Learn](https://learn.microsoft.com/en-us/training/modules/govern-unity-catalog-objects/)  
> **Versión de apuntes:** 15-11-2026 · Contenido técnico redactado de forma original a partir del itinerario de Microsoft.

# 7. Gobernar objetos de Unity Catalog

**Referencia Microsoft Learn:** [Gobernar objetos de Unity Catalog](https://learn.microsoft.com/en-us/training/modules/govern-unity-catalog-objects/) · **10 unidades oficiales**.

## 7.1. Gobierno: más que permisos

Gobernar datos es mantenerlos **descubribles, comprensibles, seguros, auditables y reutilizables**. Una tabla puede ser accesible técnicamente y aun así tener escaso valor si nadie sabe qué significa cada columna, de dónde viene un importe o quién debe aprobar cambios en su esquema.

## 7.2. Definiciones y metadatos

Documentar significa explicar granularidad, campos, unidades de medida, propietario funcional, frescura y reglas de calidad. Los comentarios de catálogo, esquema, tabla y columna se convierten en metadatos accesibles para analistas y herramientas automatizadas.

```sql
COMMENT ON TABLE estudio.ventas.pedidos IS
'Tabla transaccional de pedidos; una fila por pedido';

ALTER TABLE estudio.ventas.pedidos
ALTER COLUMN importe COMMENT 'Importe del pedido en EUR';
```

Una definición como «ingresos» sin concretar devoluciones, impuestos o descuentos puede dar lugar a informes incoherentes incluso con un pipeline perfecto.

## 7.3. ABAC: Attribute-Based Access Control

El **RBAC** concede derechos según roles o grupos. El **ABAC** toma decisiones usando atributos de datos y políticas, por ejemplo etiquetas de clasificación. Las **governed tags** contribuyen a describir de modo normalizado información como sensibilidad y dominio; las políticas ABAC compatibles pueden aplicar controles de fila o máscara en función de esas etiquetas y reglas.

Ejemplo: una columna con una etiqueta de dato personal debe tratarse con una política de protección; la etiqueta por sí sola **no equivale** a una prohibición de lectura. Hay que configurar la política y verificar su alcance.

## 7.4. Retención de datos

La retención cumple objetivos diferentes: cumplimiento, recuperación, auditoría y coste. En tablas Delta, el historial de versiones está relacionado con el registro de transacciones y la conservación de archivos. **Time travel no es un archivo histórico permanente garantizado.** Las operaciones de mantenimiento y la política de retención pueden eliminar archivos antiguos necesarios para reconstruir versiones.

```sql
DESCRIBE HISTORY estudio.ventas.pedidos;

-- Consulta de versión histórica solo si se conservan sus archivos:
SELECT * FROM estudio.ventas.pedidos VERSION AS OF 3;
```

Para analizar décadas de cambios de perfiles o estados, conviene diseñar una tabla histórica explícita (por ejemplo SCD Tipo 2) además de usar time travel operacional. Ajustar cualquier retención exige comprobar requisitos de streaming, operaciones concurrentes, gobierno y recuperación.

## 7.5. Linaje de datos

El **lineage** describe dependencias: tablas de origen, procesos transformadores y objetos de destino. A nivel de tabla indica qué datasets alimentan a otros; a nivel de columna ayuda a explicar de qué campos resulta una métrica. Resulta útil para análisis de impacto (¿qué informes se rompen si cambio esta columna?), auditorías y depuración.

```text
crm.clientes ──┐
               ├─→ silver.clientes_limpios ─→ gold.clientes_activos → informe
web.eventos ──┘
```

El linaje registrado depende de los mecanismos y operaciones reconocidos por la plataforma: no debe asumirse que toda transformación en cualquier herramienta externa se captura automáticamente.

## 7.6. Audit logging

Los registros de auditoría ayudan a responder **quién** accedió, **qué** operación realizó, **cuándo**, desde **dónde** y con qué resultado. Se utilizan en respuesta a incidentes y controles de cumplimiento. Las tablas de sistema y los mecanismos de entrega de logs ofrecen información según configuración, disponibilidad y retención del entorno.

Una auditoría eficaz también exige conservar registros de forma segura, limitar su acceso y evitar almacenar secretos en los propios logs.

## 7.7. Compartir datos: OpenSharing / Delta Sharing

El intercambio seguro de datos permite proporcionar conjuntos de datos a destinatarios fuera de un mismo workspace sin exportaciones manuales constantes. **Delta Sharing** se utiliza para compartir datos de manera controlada, con objetos como shares, recipients y permisos. En el temario se introduce la denominación **OpenSharing** para la estrategia de intercambio abierto.

Elegir una estrategia requiere conocer el destinatario, credenciales, revocación, formatos admitidos, posibles restricciones de filtros y residencia de los datos. Compartir una tabla no debe interpretarse como conceder acceso al metastore entero.

## 7.8. Laboratorio y repaso

Documenta una tabla con su propietario, granularidad y clasificación. Diseña una política que proteja atributos personales, un procedimiento para revisar linaje antes de renombrar una columna y una retención adecuada para auditoría. Explica por qué las versiones Delta antiguas no son sustituto automático de una tabla histórica de clientes.

---

---


## Desarrollo en profundidad: gobernar el ciclo de vida del dato

### Gobierno no es únicamente permisos

El gobierno de datos incluye saber qué existe, quién es responsable, dónde se utiliza, cuál es su sensibilidad, de dónde procede y qué políticas determinan su retención. Una tabla perfectamente protegida puede seguir siendo un problema si nadie conoce su linaje o su definición de negocio. Unity Catalog aporta un punto de control común para muchos de estos aspectos, pero la calidad del gobierno depende también de los procesos de la organización.

### Linaje y análisis de impacto

El linaje registra relaciones entre entradas, transformaciones y salidas cuando los mecanismos del servicio pueden capturarlas. Si una columna `importe_total` en Gold procede de `importe` y `descuento` en Silver, el linaje ayuda a responder qué informes podrían verse afectados por cambiar una de las columnas de origen. No equivale a documentar toda la lógica funcional y no todos los procesos externos necesariamente quedan representados.

**Ejemplo:** un equipo necesita cambiar el tipo de dato de `id_cliente` de texto a entero. Antes del cambio consulta consumidores, pipelines y vistas dependientes. Puede haber joins que asumen ceros a la izquierda y se romperían al convertir a entero. El análisis de impacto debe combinar linaje con pruebas y revisión de contratos.

### Etiquetas, clasificación y propiedad

Las etiquetas ayudan a clasificar activos por sensibilidad, dominio, criticidad o necesidades de tratamiento. Un diseño mantenible define un vocabulario común en lugar de permitir diez variantes de `confidencial`. La propiedad identifica quién debe aprobar cambios, responder ante incidencias y mantener documentación. En objetos compartidos conviene diferenciar propietario técnico de propietario funcional.

### Retención frente a recuperación

Con Delta Lake puedes consultar versiones antiguas dentro del periodo de retención disponible, pero la existencia de un historial transaccional no garantiza conservación indefinida. Políticas de limpieza física, requisitos regulatorios y recuperación de desastres deben considerarse por separado. Para SCD2 o auditoría a largo plazo se necesita un diseño explícito, no depender solo del *time travel*.

### Auditoría y uso compartido

Los registros de auditoría sirven para investigar operaciones y cambios relevantes, sujetos a la configuración y disponibilidad del entorno. Delta Sharing permite escenarios de intercambio gobernado: antes de compartir, identifica el destinatario, los objetos permitidos, la duración del acuerdo y el procedimiento de revocación. La facilidad técnica de publicar datos no sustituye el contrato legal ni la minimización de datos.

**Práctica de gobierno:** define cinco campos de documentación para una tabla Gold (`owner`, `clasificación`, `frecuencia`, `grano`, `SLA`) y describe cómo detectarías consumidores afectados por cambiar su esquema. Elabora una política de retención distinta para logs operativos, transacciones y datos agregados. Explica qué registrarías cuando una cuenta de servicio modifica permisos o publica una compartición externa.

---

## 🧪 Guía de laboratorio y comprobación práctica

**Ejercicio oficial:** [Abrir las instrucciones de MicrosoftLearning](https://microsoftlearning.github.io/DP-750T00-Implement-Data-Engineering-Solutions-using-Azure-Databricks/Instructions/Labs/05-govern-unity-catalog-objects.html)  
**Notebook oficial:** [Importar o consultar notebook `.ipynb`](https://raw.githubusercontent.com/MicrosoftLearning/DP-750T00-Implement-Data-Engineering-Solutions-using-Azure-Databricks/refs/heads/main/Allfiles/05-govern-unity-catalog-objects.ipynb)  
**Naturaleza de esta sección:** itinerario explicado con redacción propia y comprobaciones de aprendizaje. Las celdas exactas y el material de origen están en el notebook oficial enlazado; no se reproducen íntegramente aquí.


### Caso práctico: AutoSphere AG — políticas, linaje y auditoría

**Contexto:** una plataforma automovilística recibe telemetría, matrículas y operaciones de taller. Es necesario clasificar información sensible, controlar retenciones y averiguar qué procesos producen y consumen cada conjunto de datos.

**Recorrido de laboratorio:**

1. Importa el notebook y crea el catálogo y las tablas del caso. La guía usa `automotive_catalog.governance_lab` y activos como `vehicle_telemetry` y `service_records`.
2. Añade comentarios de tabla y columna orientados al negocio. Registra tags de clasificación para campos que contienen datos personales; distingue tags documentales de políticas ABAC que realmente apliquen reglas.
3. Examina las propiedades de **retención** en Delta Lake. Estudia por separado la conservación del **log de transacciones** y la de **archivos eliminados**. Antes de cualquier `VACUUM`, comprueba el periodo de recuperación y los lectores concurrentes: reducir retención de forma agresiva puede impedir time travel o dañar consumidores rezagados.
4. Realiza una prueba controlada de `VACUUM` en datos descartables e inspecciona el historial de la tabla. Investiga cómo funciona predictive optimization para managed tables elegibles.
5. En **Catalog Explorer**, abre **Lineage** de `vehicle_telemetry` y navega a dependencias de tablas y, si aparecen, de columnas de `service_records`.
6. Examina las tablas del sistema de linaje y de auditoría que tu workspace tenga habilitadas. Formula consultas temporales para determinar qué identidad accedió a un recurso y qué operación se registró.

**Verificación:** diferencia `DESCRIBE HISTORY` (historial de transacciones Delta) de auditoría (eventos sobre operaciones de usuario) y de linaje (dependencias entre activos). Los tres mecanismos proporcionan señales diferentes; ninguno reemplaza a los demás.

**Advertencia:** los registros y las tablas de sistema pueden necesitar activación o privilegios específicos y presentar latencia. No interpretes una ausencia de filas como prueba de que ningún usuario accedió al dato.


---

### 📚 Fuentes y navegación

- [Contenido oficial de Microsoft Learn](https://learn.microsoft.com/en-us/training/modules/govern-unity-catalog-objects/)
- [Repositorio docente de laboratorios DP-750T00](https://microsoftlearning.github.io/DP-750T00-Implement-Data-Engineering-Solutions-using-Azure-Databricks/)
- [Guía oficial de competencias del examen DP-750](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-750)

[🏠 Índice](../README.md) · [← Anterior](../06-proteger-objetos-de-unity-catalog/README.md) · [Siguiente →](../08-modelado-de-datos/README.md)
