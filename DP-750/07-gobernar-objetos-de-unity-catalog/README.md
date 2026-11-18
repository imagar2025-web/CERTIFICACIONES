# 07 · Gobernar objetos de Unity Catalog

[Índice DP-750](../README.md) · [← Proteger objetos de Unity Catalog](../06-proteger-objetos-de-unity-catalog/README.md) · [Diseñar e implementar el modelado de datos →](../08-modelado-de-datos/README.md)

**Progreso personal:** Estudiado. [Módulo oficial](https://learn.microsoft.com/en-us/training/modules/govern-unity-catalog-objects/).

## Unidades

1. [Introduction](#unidad-01)
2. [Create and preserve table definitions](#unidad-02)
3. [Configure ABAC with tags and policies](#unidad-03)
4. [Apply data retention policies](#unidad-04)
5. [Set up and manage data lineage](#unidad-05)
6. [Configure audit logging](#unidad-06)
7. [Design secure OpenSharing strategy](#unidad-07)
8. [Exercise - Govern Unity Catalog Objects](#unidad-08)
9. [Module assessment](#unidad-09)
10. [Summary](#unidad-10)

- [Desarrollo aplicado](#desarrollo-aplicado)
- [Laboratorio y comprobaciones](#laboratorio-y-comprobaciones)

## Unidad 01

### Introduction

Gobernar datos es mantenerlos **descubribles, comprensibles, seguros, auditables y reutilizables**. Una tabla puede ser accesible técnicamente y aun así tener escaso valor si nadie sabe qué significa cada columna, de dónde viene un importe o quién debe aprobar cambios en su esquema.

[Unidad oficial](https://learn.microsoft.com/en-us/training/modules/govern-unity-catalog-objects/1-introduction)

## Unidad 02

### Create and preserve table definitions

Documentar significa explicar granularidad, campos, unidades de medida, propietario funcional, frescura y reglas de calidad. Los comentarios de catálogo, esquema, tabla y columna se convierten en metadatos accesibles para analistas y herramientas automatizadas.

```sql
COMMENT ON TABLE estudio.ventas.pedidos IS
'Tabla transaccional de pedidos; una fila por pedido';

ALTER TABLE estudio.ventas.pedidos
ALTER COLUMN importe COMMENT 'Importe del pedido en EUR';
```

Una definición como «ingresos» sin concretar devoluciones, impuestos o descuentos puede dar lugar a informes incoherentes incluso con un pipeline perfecto.

[Unidad oficial](https://learn.microsoft.com/en-us/training/modules/govern-unity-catalog-objects/2-preserve-table-column-definitions)

## Unidad 03

### Configure ABAC with tags and policies

El **RBAC** concede derechos según roles o grupos. El **ABAC** toma decisiones usando atributos de datos y políticas, por ejemplo etiquetas de clasificación. Las **governed tags** contribuyen a describir de modo normalizado información como sensibilidad y dominio; las políticas ABAC compatibles pueden aplicar controles de fila o máscara en función de esas etiquetas y reglas.

Ejemplo: una columna con una etiqueta de dato personal debe tratarse con una política de protección; la etiqueta por sí sola **no equivale** a una prohibición de lectura. Hay que configurar la política y verificar su alcance.

[Unidad oficial](https://learn.microsoft.com/en-us/training/modules/govern-unity-catalog-objects/3-configure-attribute-based-access-control-tags-policies)

## Unidad 04

### Apply data retention policies

La retención cumple objetivos diferentes: cumplimiento, recuperación, auditoría y coste. En tablas Delta, el historial de versiones está relacionado con el registro de transacciones y la conservación de archivos. **Time travel no es un archivo histórico permanente garantizado.** Las operaciones de mantenimiento y la política de retención pueden eliminar archivos antiguos necesarios para reconstruir versiones.

```sql
DESCRIBE HISTORY estudio.ventas.pedidos;

-- Consulta de versión histórica solo si se conservan sus archivos:
SELECT * FROM estudio.ventas.pedidos VERSION AS OF 3;
```

Para analizar décadas de cambios de perfiles o estados, conviene diseñar una tabla histórica explícita (por ejemplo SCD Tipo 2) además de usar time travel operacional. Ajustar cualquier retención exige comprobar requisitos de streaming, operaciones concurrentes, gobierno y recuperación.

#### Retención y recuperación

Una operación lógica de borrado cambia el snapshot vigente. Los archivos de snapshots anteriores pueden seguir presentes mientras sean necesarios según la retención. Para una operación controlada de mantenimiento, revisa primero las propiedades de retención y el intervalo máximo de las lecturas, escrituras y consumidores que dependan de versiones anteriores. `VACUUM` actúa sobre archivos obsoletos y puede volver inaccesible un snapshot antiguo. `DESCRIBE HISTORY` puede seguir mostrando el commit aunque los archivos que necesitaría time travel ya no existan.

```sql
SHOW TBLPROPERTIES estudio.privacidad.eventos;
DESCRIBE HISTORY estudio.privacidad.eventos;
VACUUM estudio.privacidad.eventos DRY RUN;
```

El modo `DRY RUN` permite inspeccionar candidatos sin borrarlos. Para una solicitud de eliminación de datos, revisa también copias, destinos derivados y políticas de respaldo. La retención de una tabla afecta a ese objeto. Un histórico de negocio o un registro de auditoría requieren su propia definición de conservación.

[Unidad oficial](https://learn.microsoft.com/en-us/training/modules/govern-unity-catalog-objects/4-apply-data-retention-policies)

## Unidad 05

### Set up and manage data lineage

El **lineage** describe dependencias: tablas de origen, procesos transformadores y objetos de destino. A nivel de tabla indica qué datasets alimentan a otros; a nivel de columna ayuda a explicar de qué campos resulta una métrica. Resulta útil para análisis de impacto (¿qué informes se rompen si cambio esta columna?), auditorías y depuración.

```text
crm.clientes ──┐
               ├─→ silver.clientes_limpios ─→ gold.clientes_activos → informe
web.eventos ──┘
```

El linaje registrado depende de los mecanismos y operaciones reconocidos por la plataforma: no debe asumirse que toda transformación en cualquier herramienta externa se captura automáticamente.

[Unidad oficial](https://learn.microsoft.com/en-us/training/modules/govern-unity-catalog-objects/5-manage-data-lineage-tracking)

## Unidad 06

### Configure audit logging

Los registros de auditoría ayudan a responder **quién** accedió, **qué** operación realizó, **cuándo**, desde **dónde** y con qué resultado. Se utilizan en respuesta a incidentes y controles de cumplimiento. Las tablas de sistema y los mecanismos de entrega de logs ofrecen información según configuración, disponibilidad y retención del entorno.

Una auditoría eficaz también exige conservar registros de forma segura, limitar su acceso y evitar almacenar secretos en los propios logs.

[Unidad oficial](https://learn.microsoft.com/en-us/training/modules/govern-unity-catalog-objects/6-configure-audit-logging)

## Unidad 07

### Design secure OpenSharing strategy

El intercambio seguro de datos permite proporcionar conjuntos de datos a destinatarios fuera de un mismo workspace sin exportaciones manuales constantes. **Delta Sharing** se utiliza para compartir datos de manera controlada, con objetos como shares, recipients y permisos. En el temario se introduce la denominación **OpenSharing** para la estrategia de intercambio abierto.

Elegir una estrategia requiere conocer el destinatario, credenciales, revocación, formatos admitidos, posibles restricciones de filtros y residencia de los datos. Compartir una tabla no debe interpretarse como conceder acceso al metastore entero.

[Unidad oficial](https://learn.microsoft.com/en-us/training/modules/govern-unity-catalog-objects/7-secure-open-sharing-strategy)

## Unidad 08

### Exercise - Govern Unity Catalog Objects

Documenta una tabla con su propietario, granularidad y clasificación. Diseña una política que proteja atributos personales, un procedimiento para revisar linaje antes de renombrar una columna y una retención adecuada para auditoría. Explica por qué las versiones Delta antiguas no son sustituto automático de una tabla histórica de clientes.

[Unidad oficial](https://learn.microsoft.com/en-us/training/modules/govern-unity-catalog-objects/8-exercise-govern-unity-catalog-objects)

## Unidad 09

### Module assessment

Un usuario pregunta quién leyó una tabla y qué pipeline la alimenta. El *audit log* aporta hechos sobre acciones y actores. El linaje representa dependencias entre objetos, sujeto a los requisitos de captura de la plataforma. `DESCRIBE HISTORY` examina commits de una tabla Delta. Los tres responden preguntas diferentes. Si necesita compartir la tabla fuera del workspace, identifica el destinatario, protocolo de acceso y activos expuestos antes de publicar el share.

[Unidad oficial](https://learn.microsoft.com/en-us/training/modules/govern-unity-catalog-objects/9-knowledge-check)

## Unidad 10

### Summary

El mantenimiento de una tabla gobernada combina definiciones semánticas, permisos extendidos por atributos, retención física y observabilidad. Para evaluar una política por tags comprueba quién puede asignarlos y qué herencia se aplica. Para evaluar la retención comprueba el horizonte que necesita la operación para recuperar datos y el que exige la organización para auditoría. Conserva los registros de auditoría en un destino y con una política que cubra ese horizonte.

[Unidad oficial](https://learn.microsoft.com/en-us/training/modules/govern-unity-catalog-objects/10-summary)

## Desarrollo aplicado

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

## Laboratorio y comprobaciones

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

### Evidencias de retención, linaje y auditoría

Ejecuta una transformación pequeña del notebook para que haya un vínculo de linaje observable. Abre la tabla de destino en Catalog Explorer y comprueba la relación con su origen. Después revisa las tablas de sistema del laboratorio con el intervalo y la identidad de prueba. Si no aparece actividad, verifica acceso a las system tables, región, latencia y que la acción elegida pertenezca a los eventos capturados. Para retención, guarda una evidencia de propiedades, un historial reciente y los candidatos de VACUUM. El borrado físico corresponde únicamente al dataset de prácticas y al procedimiento oficial. Los filtros de prueba deben reducir el rango de auditoría para evitar inspeccionar acciones de usuarios ajenos.

## Fuentes y navegación

- [Fuentes de la certificación](../FUENTES.md)
- [Preparación y limpieza del entorno](../PREPARACION-LABORATORIOS.md)

[Índice DP-750](../README.md) · [← Proteger objetos de Unity Catalog](../06-proteger-objetos-de-unity-catalog/README.md) · [Diseñar e implementar el modelado de datos →](../08-modelado-de-datos/README.md)
