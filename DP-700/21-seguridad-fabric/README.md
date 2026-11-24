# 21 · Secure data access in Microsoft Fabric

> **DP-700 · Manage Microsoft Fabric.** [Módulo de Microsoft Learn](https://learn.microsoft.com/en-us/training/modules/secure-data-access-in-fabric/) · [fuentes en \`wwl-data-ai\`](https://github.com/MicrosoftDocs/learn/tree/main/learn-pr/wwl-data-ai/secure-data-access-in-fabric) · [laboratorio oficial DP-700](https://github.com/MicrosoftLearning/mslearn-fabric/blob/main/Instructions/Labs/19-secure-data-access.md). **Revisión:** noviembre de 2026. Teoría documentada, laboratorio no ejecutado.

## Índice

1. [Introducción](#1-introducción)
2. [Understand Fabric security model](#2-understand-fabric-security-model)
3. [Workspace and item permissions](#3-workspace-and-item-permissions)
4. [Granular security: SQL and OneLake](#4-granular-security-sql-and-onelake)
5. [Laboratorio oficial](#5-laboratorio-oficial)
6. [Knowledge check y summary](#6-knowledge-check-y-summary)
7. [Fuentes y control](#7-fuentes-y-control)

## 1. Introducción

Una plataforma analítica compartida debe proteger tanto los datos como los procesos que los transforman. En Microsoft Fabric, un equipo puede necesitar acceso al mismo Workspace para desarrollar modelos, pero no todos los miembros deben leer datos personales o ejecutar escrituras de producción. La seguridad se define por **identidad, recurso, acción y ámbito**. Se deben distinguir derechos de administración del Workspace, permisos para descubrir o utilizar elementos y autorización sobre datos concretos dentro de Lakehouses o Warehouses.

La seguridad no consiste en marcar una opción de privacidad en un informe. Una identidad puede acceder mediante diferentes superficies: SQL analytics endpoint, Notebook Spark, interfaz del Lakehouse, Warehouse o conexiones compatibles. Por tanto, hay que comprobar el acceso efectivo en todas las vías relevantes.

## 2. Understand Fabric security model

### 2.1. Identidad y autenticación

Microsoft Fabric utiliza identidades y controles integrados con Microsoft Entra para autenticar usuarios y procesos, dentro de las configuraciones admitidas. La autenticación responde a **quién eres**; la autorización responde a **qué puedes hacer**. Un inicio de sesión válido no concede permiso de lectura universal sobre OneLake.

Cuando un Pipeline automatizado accede a datos, la identidad efectiva puede ser distinta de la persona que diseñó el flujo. Esto debe documentarse y probarse. Las conexiones técnicas deben adoptar el principio de mínimo privilegio y evitar credenciales compartidas o secretos incrustados en Notebooks.

### 2.2. Capas de seguridad

Podemos distinguir cuatro niveles conceptuales: tenant y administración general; Workspace y colaboración; elementos individuales; controles de lectura y escritura de datos dentro de engines, tablas y rutas.

~~~mermaid
flowchart TB
  I["Microsoft Entra identity"] --> T["Tenant / políticas"]
  T --> W["Workspace role"]
  W --> P["Item permissions"]
  P --> D["SQL / OneLake data permissions"]
  D --> C["Resultado autorizado"]
~~~

*Diagrama conceptual propio.* No implica que una autorización en un nivel sustituya automáticamente las demás. Debe estudiarse cómo se componen permisos y roles en cada engine.

### 2.3. Mínimo privilegio

Un usuario de informes normalmente debería consumir tablas Gold o modelos autorizados sin modificar pipelines de carga. Una identidad de ingesta puede necesitar escribir en Bronze sin acceso a los informes de negocio. Un administrador de capacidad necesita monitorizar recursos, pero no necesariamente ver todos los datos personales. La separación de funciones reduce el impacto de errores y compromisos de credenciales.

### 2.4. Gobernanza

Documentar quién es propietario de cada conjunto de datos, nivel de clasificación, periodo de retención y personas autorizadas. La revisión periódica es importante porque las responsabilidades cambian. Los permisos asignados durante una práctica pueden permanecer activos si no se limpian.

**Fuente:** [Security model](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl-data-ai/secure-data-access-in-fabric/includes/2-understand-fabric-security-model.md).

## 3. Workspace and item permissions

### 3.1. Workspace roles

Fabric utiliza roles de colaboración en los Workspaces, con capacidades de administración, contribución o lectura según el rol. Para un proyecto con varios miembros, evitar conceder acceso administrativo a todos por comodidad. La configuración debe realizarse teniendo en cuenta quién puede crear elementos, editarlos, compartirlos y acceder a los datos asociados.

La denominación y permisos de los roles deben consultarse en la documentación actual. Los roles comunes como Admin, Member, Contributor y Viewer tienen diferencias relevantes; no inferir que un rol Viewer concede cualquier permiso de lectura de filas en todas las superficies.

### 3.2. Item permissions

Los **item permissions** permiten compartir o autorizar un elemento concreto sin conceder el mismo nivel de acceso al conjunto de elementos del Workspace. Esto es útil cuando una persona necesita leer o utilizar un Warehouse o Lakehouse específico. La configuración debe revisarse conjuntamente con los roles de Workspace porque el acceso final puede depender de ambas rutas.

### 3.3. Diseño de Workspaces por responsabilidad

Una organización puede separar entornos dev/test/prod, dominios de datos o capas Bronze/Silver/Gold. Es una decisión que influye en aislamiento, costes y administración. La separación física de Workspaces no sustituye la protección de datos dentro de cada uno si contiene información sensible. Una estructura demasiado fragmentada puede dificultar el mantenimiento; una estructura única con todos los recursos puede abrir ámbitos innecesariamente amplios.

### 3.4. Validación con usuarios de prueba

No comprobar seguridad únicamente utilizando la cuenta del administrador. Crear una matriz explícita de roles y recursos: qué persona puede abrir el Workspace, consultar una tabla Gold, leer Bronze, modificar un Pipeline o administrar permisos. Ejecutar pruebas con usuarios autorizados diferentes y registrar el comportamiento observado.

**Fuente:** [Workspace and item permissions](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl-data-ai/secure-data-access-in-fabric/includes/3-configure-workspace-and-item-permissions.md).

## 4. Granular security: SQL and OneLake

### 4.1. Seguridad granular T-SQL

Fabric Warehouse admite controles sobre objetos y, según la capacidad, filas y columnas. Una vista de solo lectura puede ser una superficie de publicación, pero debe estar acompañada de permisos efectivos para evitar que el usuario consulte directamente la tabla subyacente.

~~~sql
GRANT SELECT ON OBJECT::dbo.vw_ventas_publicas TO [LectoresInformes];
~~~

*Ejemplo propio.* El principal debe existir; adaptar sintaxis y permisos a la identidad real. No utilizar GRANT generalizado en producción ni confiar en un ejemplo como si hubiese sido ejecutado.

### 4.2. OneLake security roles

**OneLake security roles** permiten controlar el acceso a datos almacenados en OneLake, incluyendo escenarios de permisos granulares sobre tablas o carpetas compatibles dentro de Lakehouses. La funcionalidad debe configurarse en la experiencia adecuada y contrastarse con el modelo de autorización de cada workload.

No confundir un OneLake security role con un Workspace role. Una persona puede poder abrir el Lakehouse y no tener autorización sobre todas las tablas o rutas. Los accesos permitidos dependen de la identidad, del permiso y de la superficie usada; hay que verificar cada ruta.

### 4.3. Separación por capas de datos

Un diseño razonable puede permitir a ingenieros autorizados consultar Bronze y Silver y a analistas solo Gold. Si Bronze contiene datos personales, conceder permiso de lectura de la carpeta completa por comodidad puede ser demasiado amplio. Se pueden definir rutas específicas o utilizar elementos y Workspaces aislados cuando el control de acceso y el gobierno lo requieran.

### 4.4. Pruebas negativas

Una prueba de seguridad no está completa si solo demuestra acceso correcto. También debe confirmar que una identidad sin permiso recibe la denegación esperada al intentar leer datos sensibles. Probar acceso mediante las superficies relevantes para evitar vías alternativas no contempladas en el diseño.

**Fuente:** [Granular permissions](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl-data-ai/secure-data-access-in-fabric/includes/4-apply-granular-permissions.md).

## 5. Laboratorio oficial

**Fuente:** [Secure data access in Microsoft Fabric](https://github.com/MicrosoftLearning/mslearn-fabric/blob/main/Instructions/Labs/19-secure-data-access.md). Duración aproximada oficial: **45 minutos**, asociado a DP-700 y DP-600. **No ejecutado.**

### 5.1. Objetivo

Aplicar varias capas de seguridad: roles de Workspace, permisos por elemento y OneLake security roles sobre tablas y carpetas de un Lakehouse. El laboratorio crea un Workspace, un Warehouse y un Lakehouse para observar diferencias de ámbito y capacidades.

### 5.2. Preparar entorno

Abrir [Fabric](https://app.fabric.microsoft.com/) con una capacidad compatible. Crear Workspace de laboratorio y los objetos indicados en el tutorial. Identificar quién tiene rol de administración antes de añadir usuarios de prueba. No utilizar identidades corporativas reales para simular accesos indebidos.

### 5.3. Workspace access controls

Configurar los roles indicados en el procedimiento. Revisar qué elementos puede descubrir y administrar cada usuario. Documentar el efecto de cambiar un rol de Contributor a Viewer o de retirar un rol, utilizando cuentas de prueba autorizadas. No se deben inventar resultados si no se han realizado pruebas reales.

### 5.4. Item access control

Aplicar permisos a un elemento concreto, por ejemplo Warehouse o Lakehouse, y comprobar cómo difiere del acceso a todo el Workspace. Registrar la identidad, recurso, acción permitida y resultado observado. La matriz ayuda a detectar permisos excesivos.

### 5.5. OneLake security

Crear los OneLake security roles del laboratorio, restringiendo el acceso a determinadas tablas o carpetas según las instrucciones originales. Comprobar tanto una lectura autorizada como una lectura que debería denegarse. Si una identidad continúa accediendo, revisar otros roles de Workspace y los permisos efectivos, no asumir que la política falló sin diagnosticar.

### 5.6. Diagnóstico

| Incidencia | Posible causa | Acción |
|---|---|---|
| No se puede editar seguridad | Rol o permisos insuficientes | Revisar identidad administradora |
| Se ve un elemento pero no sus datos | Permiso de elemento distinto al del dato | Revisar seguridad granular |
| Usuario sigue con acceso | Múltiples roles o permisos | Auditar rutas efectivas |
| Spark y SQL parecen comportarse distinto | Superficies y permisos diferentes | Comprobar soporte concreto |
| Un usuario ve Bronze por error | Permiso demasiado amplio | Reducir ámbitos |
| No aparece OneLake security | Capacidad o experiencia no habilitada | Revisar disponibilidad del servicio |

### 5.7. Limpieza

Eliminar únicamente los objetos y Workspaces de prueba. Retirar usuarios, roles y permisos concedidos para el ejercicio. La capacidad pagada debe revisarse de forma separada. No borrar recursos ajenos ni afirmar que se revocaron accesos sin hacerlo.

## 6. Knowledge check y summary

Cuestiones propias: ¿Qué diferencia autenticación de autorización? ¿Para qué sirve un Workspace role? ¿En qué se distingue un item permission de un permiso de datos? ¿Qué son OneLake security roles? ¿Cómo aplicar mínimo privilegio? ¿Por qué realizar pruebas negativas? ¿Por qué una cuenta administradora no es buena identidad para verificar restricciones?

**Summary:** la seguridad de Fabric es multinivel. Para proteger información hay que diseñar permisos sobre Workspaces, objetos y datos, y validarlos con distintas identidades y vías de acceso.

## 7. Fuentes y control

- [index.yml, siete unidades](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl-data-ai/secure-data-access-in-fabric/index.yml).
- [Modelo de seguridad](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl-data-ai/secure-data-access-in-fabric/includes/2-understand-fabric-security-model.md).
- [Permisos Workspace / Item](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl-data-ai/secure-data-access-in-fabric/includes/3-configure-workspace-and-item-permissions.md).
- [Seguridad granular](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl-data-ai/secure-data-access-in-fabric/includes/4-apply-granular-permissions.md).
- [Laboratorio oficial DP-700](https://github.com/MicrosoftLearning/mslearn-fabric/blob/main/Instructions/Labs/19-secure-data-access.md).

**Control:** conceptos, configuración, práctica, evaluación y síntesis documentados. Ejercicio sin ejecutar; requiere validación futura con roles de prueba.