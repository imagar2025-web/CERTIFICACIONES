# 18 · Secure a Microsoft Fabric data warehouse

> **DP-700 · Warehouse Security.** [Módulo Microsoft Learn](https://learn.microsoft.com/en-us/training/modules/secure-data-warehouse-in-microsoft-fabric/) · [fuente en \`wwl-data-ai\`](https://github.com/MicrosoftDocs/learn/tree/main/learn-pr/wwl-data-ai/secure-data-warehouse-in-microsoft-fabric) · [laboratorio oficial](https://github.com/MicrosoftLearning/mslearn-fabric/blob/main/Instructions/Labs/06d-secure-data-warehouse.md). **Revisión:** noviembre de 2026. No ejecutado.

## Índice

1. [Introducción y modelo de amenazas](#1-introducción-y-modelo-de-amenazas)
2. [Dynamic Data Masking](#2-dynamic-data-masking)
3. [Row-Level Security](#3-row-level-security)
4. [Column-Level Security](#4-column-level-security)
5. [SQL granular permissions](#5-sql-granular-permissions)
6. [Laboratorio oficial](#6-laboratorio-oficial)
7. [Knowledge check y summary](#7-knowledge-check-y-summary)
8. [Fuentes](#8-fuentes)

## 1. Introducción y modelo de amenazas

Un Warehouse corporativo puede contener datos personales, importes, credenciales de referencia, contratos o información de negocio confidencial. **Seguridad de datos** significa determinar quién puede descubrir un recurso, ejecutar una operación, consultar una fila o visualizar una columna. No es suficiente con ocultar campos desde Power BI si una identidad tiene acceso directo al Warehouse.

Fabric combina **workspace roles**, permisos de ítem y seguridad SQL granular. En el Warehouse también se pueden aplicar mecanismos como **Dynamic Data Masking (DDM)**, **Row-Level Security (RLS)** y **Column-Level Security (CLS)**. Cada uno resuelve un problema distinto. El principio de **least privilege** obliga a conceder únicamente los permisos necesarios y durante el tiempo requerido.

~~~mermaid
flowchart TB
  I["Identidad Microsoft Entra"] --> W["Workspace / Item permissions"]
  W --> S["SQL permissions"]
  S --> R["Row-Level Security"]
  R --> C["Column-Level Security"]
  C --> M["Dynamic Data Masking"]
  M --> Q["Resultado autorizado de consulta"]
~~~

**Diagrama lógico propio.** La evaluación de privilegios no tiene por qué seguir exactamente este orden físico; representa capas conceptuales de control.

## 2. Dynamic Data Masking

### 2.1. Qué protege y qué no

**DDM** modifica la presentación de determinados valores en las consultas de usuarios sin el privilegio correspondiente. El dato original permanece almacenado. Es útil para minimizar la exposición accidental de emails u otros campos a perfiles de trabajo que necesitan analizar las filas pero no ver todos los detalles.

No debe confundirse **masking** con cifrado, anonimización irreversible o un control de fila. DDM no reemplaza las restricciones de acceso directas; una identidad privilegiada puede tener acceso a valores sin enmascarar. Además, diseños de consultas y privilegios mal configurados pueden producir inferencias sobre el dato: hay que revisar la amenaza real.

### 2.2. Regla de enmascaramiento

En un entorno SQL compatible, la configuración de una columna puede incorporar un método de masking (por ejemplo, email) al definir el campo o mediante DDL admitido. Un ejemplo conceptual es:

~~~sql
CREATE TABLE dbo.ContactsDemo (
    ContactID INT NOT NULL,
    Email VARCHAR(200) MASKED WITH (FUNCTION = 'email()')
);
~~~

**Ejemplo propio para un Warehouse de laboratorio.** Comprobar compatibilidad y permisos concretos de la función en la versión de Fabric. Antes de ejecutar un DDL sobre un objeto real, revisar que el Warehouse admite la sintaxis y que el recurso no existe.

### 2.3. Comprobar con distintas identidades

Una prueba válida ejecuta la misma consulta con una identidad que posee privilegios de exposición y otra sin ellos. No es suficiente con que el administrador vea los datos completos. El resultado correcto depende de los permisos efectivos; una cuenta de prueba sin privilegios de lectura no demostrará el masking porque ni siquiera podrá consultar.

**Fuente:** [DDM](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl-data-ai/secure-data-warehouse-in-microsoft-fabric/includes/2-explore-dynamic-data-masking.md).

## 3. Row-Level Security

### 3.1. Idea general

**RLS** restringe qué filas puede leer una identidad. Por ejemplo, un responsable de zona norte puede acceder únicamente a ventas de su región. A diferencia de un filtro implementado en una consulta de la aplicación, una política SQL se aplica desde el motor cuando se cumplen sus condiciones y permisos.

En SQL, este control suele combinar una función de predicado y una **security policy**. La función determina la correspondencia entre fila e identidad o contexto. Una configuración incorrecta puede denegar todos los registros o exponer más de lo previsto.

### 3.2. Diseño de predicado

Es necesario decidir cómo se mapea una identidad de Microsoft Entra o de base de datos a una región o entidad. Una tabla de autorización puede asociar usuarios y regiones, pero debe gobernarse por administradores autorizados y no por los propios analistas. Si las regiones cambian, el acceso debe seguir la política vigente, no quedar vinculado indefinidamente a una cadena guardada en el cliente.

No inventar funciones de identificación ni políticas sin consultar el SQL exacto y la compatibilidad actual. El módulo oficial proporciona sintaxis de RLS para Fabric Warehouse, que debe utilizarse en el laboratorio con los roles y tablas indicados.

### 3.3. Pruebas adversarias

Probar acceso con al menos dos identidades diferentes, incluyendo una que no deba ver ciertas filas. Verificar también que los agregados y joins no revelan filas de otros ámbitos. Es importante probar escenarios de usuario sin región asignada, usuarios con varias regiones y cambios de asignación.

**Fuente:** [RLS](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl-data-ai/secure-data-warehouse-in-microsoft-fabric/includes/3-implement-row-level-security.md).

## 4. Column-Level Security

### 4.1. Diferencia con DDM

**CLS** restringe acceso a columnas. Si un usuario intenta seleccionar una columna sin privilegio, recibe una denegación, en lugar de un valor enmascarado. Esto es útil cuando una parte de una tabla es apta para análisis y otra contiene información sensible que no debe estar disponible.

| Requisito | DDM | RLS | CLS |
| --- | --- | --- | --- |
| Mostrar parte del valor oculto | Sí, según máscara | No es su función | Deniega acceso a columna |
| Filtrar filas según identidad | No | Sí | No |
| Impedir seleccionar una columna | No por sí solo | No por sí solo | Sí |
| Sustituir la autorización general | No | No | No |
| Ámbito de control | Presentación de campos | Conjunto de filas | Columnas |

### 4.2. Permisos

Cuando el motor admite permisos sobre columnas, se configura la autorización solo para las columnas necesarias. Un usuario podría obtener \`ContactID\` y \`Region\`, pero no \`PersonalNotes\`. Al mantener múltiples roles, verificar permisos acumulados y reglas de denegación. La seguridad no debe depender de que la aplicación omita voluntariamente una columna en su SELECT.

**Fuente:** [CLS](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl-data-ai/secure-data-warehouse-in-microsoft-fabric/includes/4-implement-column-level-security.md).

## 5. SQL granular permissions

### 5.1. GRANT, DENY y REVOKE

Fabric Warehouse utiliza permisos SQL sobre objetos admitidos. **GRANT** concede un derecho, **DENY** lo deniega explícitamente y **REVOKE** retira la concesión o denegación. REVOKE no significa siempre «denegar», porque pueden existir permisos heredados de otros roles. Según la documentación del módulo, cuando una identidad recibe un GRANT y DENY efectivos sobre el mismo permiso, **DENY prevalece**.

~~~sql
GRANT SELECT ON OBJECT::dbo.vw_PublicSales TO [ReportingReaders];
DENY SELECT ON OBJECT::dbo.PrivateSales TO [ReportingReaders];
~~~

*Ejemplo conceptual propio*. \`ReportingReaders\` debe existir y la sintaxis de principals debe ajustarse a los roles efectivos del entorno. Ejecutar exclusivamente en un Warehouse de prueba, no sobre roles corporativos reales.

### 5.2. Objetos y operaciones

Los permisos pueden aplicarse sobre tablas, vistas, funciones o procedimientos según las capacidades admitidas. Una identidad de ingesta podría necesitar INSERT en staging y EXECUTE de un procedimiento específico, pero no ALTER sobre todo el esquema. Para consultas de reporting, puede bastar SELECT sobre vistas curadas.

### 5.3. Auditoría

Documentar cada concesión, motivo, propietario y revisión periódica. Las políticas de seguridad deben versionarse y desplegarse con controles. Revisar el acceso efectivo de identidades técnicas y humanas después de cambios en Workspaces o roles. Un usuario añadido a un rol administrativo de Workspace podría tener más derechos de los esperados; no evaluar únicamente permisos SQL individuales.

**Fuente:** [SQL granular permissions](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl-data-ai/secure-data-warehouse-in-microsoft-fabric/includes/5-configure-sql-granular-permissions.md).

## 6. Laboratorio oficial

**Fuente:** [Secure data in a data warehouse](https://github.com/MicrosoftLearning/mslearn-fabric/blob/main/Instructions/Labs/06d-secure-data-warehouse.md). Duración orientativa: **45 minutos**. Asociado a la formación de seguridad Warehouse de Fabric. **No ejecutado.**

### 6.1. Requisitos y escenario

Se crea un Workspace y un Warehouse de prueba, se aplican reglas de DDM a columnas, se implementa RLS, se configura CLS y se practican GRANT, DENY y REVOKE. Los permisos son el núcleo del ejercicio: se deben utilizar identidades y roles de prueba, no hacer modificaciones sobre datos o usuarios productivos.

### 6.2. Crear Warehouse y datos

En [Fabric](https://app.fabric.microsoft.com/) crear Workspace aislado y Warehouse siguiendo la guía. Ejecutar el SQL original de creación de tablas y carga de datos. Identificar columnas sensibles y usuarios utilizados en las pruebas de seguridad. No inventar valores de prueba ni atribuir un resultado sin ejecutarlo.

### 6.3. DDM

Configurar una máscara sobre la columna que indica el laboratorio. Consultar como identidad privilegiada y como identidad con masking efectivo. Registrar qué valores se ocultan y cuáles permanecen visibles. Si el usuario de prueba no tiene SELECT, otorgar el permiso mínimo requerido sin convertirlo en administrador.

### 6.4. RLS

Crear la función de predicado y política siguiendo el SQL oficial. Ejecutar SELECT bajo perfiles con ámbitos distintos y comprobar que solo se devuelven filas autorizadas. Un filtro WHERE añadido manualmente al SELECT no sustituye la prueba de una política RLS real.

### 6.5. CLS y SQL permissions

Aplicar permisos granulares de columnas y comprobar que los usuarios sin autorización reciben un error al seleccionarlas. A continuación ensayar GRANT, DENY y REVOKE. Revisar los permisos efectivos después de cada modificación: retirar un GRANT puede exponer que todavía existía otro permiso por pertenencia a rol.

### 6.6. Diagnóstico y limpieza

| Incidencia | Posible causa | Solución |
|---|---|---|
| Todos ven datos completos | Cuenta privilegiada o política no efectiva | Probar identidades de bajo privilegio |
| Nadie ve filas | Predicado RLS demasiado restrictivo | Revisar mapeo de usuarios |
| Usuario no puede consultar nada | Falta permiso de lectura | Conceder mínimo acceso |
| REVOKE no bloquea | Otros permisos efectivos | Revisar roles y DENY |
| Sintaxis no admitida | Diferencia frente a SQL Server | Revisar características de Fabric Warehouse |
| CLS se salta desde otra vía | Otra identidad o exposición no gobernada | Auditar permisos en Workspace y ítems |

Eliminar el Workspace de prueba o restaurar los permisos modificados. No mantener usuarios de prueba con autorizaciones excesivas. Revisar capacidad y coste.

## 7. Knowledge check y summary

Preguntas propias: ¿Qué diferencia hay entre DDM y CLS? ¿Cómo aplica RLS un predicado? ¿Por qué una máscara no cifra los datos? ¿Qué implica DENY frente a GRANT? ¿REVOKE es idéntico a DENY? ¿Cómo comprobar acceso con dos identidades? ¿Por qué no bastan los roles de Workspace?

**Summary:** una protección efectiva del Warehouse combina ámbitos, objetos, filas, columnas y visualización de valores, siempre con pruebas de privilegio mínimo.

## 8. Fuentes

- [Index oficial, ocho unidades](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl-data-ai/secure-data-warehouse-in-microsoft-fabric/index.yml).
- [DDM](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl-data-ai/secure-data-warehouse-in-microsoft-fabric/includes/2-explore-dynamic-data-masking.md).
- [RLS](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl-data-ai/secure-data-warehouse-in-microsoft-fabric/includes/3-implement-row-level-security.md).
- [CLS](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl-data-ai/secure-data-warehouse-in-microsoft-fabric/includes/4-implement-column-level-security.md).
- [SQL permissions](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl-data-ai/secure-data-warehouse-in-microsoft-fabric/includes/5-configure-sql-granular-permissions.md).
- [Lab oficial](https://github.com/MicrosoftLearning/mslearn-fabric/blob/main/Instructions/Labs/06d-secure-data-warehouse.md).

**Control:** contenido conceptual, controles, laboratorio y evaluación documentados; falta ejecución y validación sobre un tenant real.