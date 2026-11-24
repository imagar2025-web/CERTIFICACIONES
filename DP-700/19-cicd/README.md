# 19 · Implement CI/CD in Microsoft Fabric

> **DP-700 · Manage Microsoft Fabric.** [Módulo Microsoft Learn](https://learn.microsoft.com/en-us/training/modules/implement-cicd-in-fabric/) · [archivos fuente (\`wwl-data-ai\`)](https://github.com/MicrosoftDocs/learn/tree/main/learn-pr/wwl-data-ai/implement-cicd-in-fabric) · [laboratorio DP-700](https://github.com/MicrosoftLearning/mslearn-fabric/blob/main/Instructions/Labs/21-implement-cicd.md). **Revisión:** noviembre de 2026. Procedimiento documentado, no ejecutado.

## Índice

1. [Introducción a CI/CD](#1-introducción-a-cicd)
2. [Understand CI/CD](#2-understand-cicd)
3. [Version control and Git integration](#3-version-control-and-git-integration)
4. [Deployment pipelines](#4-deployment-pipelines)
5. [Automation with Fabric APIs](#5-automation-with-fabric-apis)
6. [Laboratorio oficial](#6-laboratorio-oficial)
7. [Knowledge check y summary](#7-knowledge-check-y-summary)
8. [Fuentes](#8-fuentes)

## 1. Introducción a CI/CD

En desarrollo de software y de datos, un proceso manual en el que un ingeniero modifica directamente objetos de producción dificulta la auditoría y la recuperación. **Continuous Integration (CI)** se centra en integrar cambios de múltiples colaboradores y validarlos de manera frecuente; **Continuous Delivery / Deployment (CD)** prepara y/o automatiza la promoción de cambios a otros entornos. La diferencia entre delivery y deployment está en si la puesta en producción requiere un paso de aprobación manual o se realiza automáticamente una vez satisfechas las condiciones.

Fabric incorpora integraciones de Git y **deployment pipelines** para gestionar el ciclo de vida de elementos analíticos. Estos mecanismos se complementan. Git versiona definiciones de artefactos admitidos; los deployment pipelines gestionan la promoción de contenido entre etapas. Ninguno sustituye una estrategia completa de pruebas, autorización, secretos y migración de datos.

## 2. Understand CI/CD

### 2.1. Artefactos frente a datos

Un Notebook versionado describe código, no el estado actual de todas las tablas que procesa. Un Pipeline versionado conserva su definición, pero no representa automáticamente el histórico de archivos copiados. Esta distinción es fundamental: **Git no es el backup de OneLake** ni guarda cada transacción Delta por el mero hecho de estar conectado al Workspace.

### 2.2. Entornos de desarrollo, prueba y producción

Normalmente se diseñan Workspaces separados para **dev**, **test** y **prod**. Desarrollo contiene objetos y datasets de prueba; test valida procesos con datos representativos; producción aloja elementos y políticas autorizados para consumidores. La separación debe extenderse a conexiones, destinos de carga y credenciales. Un Pipeline desplegado desde dev puede contener una referencia al origen o destino equivocado si no se parametriza correctamente.

~~~mermaid
flowchart LR
  D["Git: feature branch"] --> P["Pull request + validación"]
  P --> M["Rama principal"]
  M --> W["Workspace DEV"]
  W --> T["Deployment Pipeline: TEST"]
  T --> A{"Pruebas y aprobación"}
  A -->|OK| R["Workspace PROD"]
  A -->|Fallan| F["Corregir cambios"]
~~~

*Diagrama propio.* La secuencia de validaciones y aprobaciones es un patrón recomendado, no un pipeline listo por defecto tras crear el servicio.

### 2.3. Pruebas útiles

Las pruebas no deben limitarse a que un Notebook pueda abrirse. Se deben ejecutar validaciones de esquemas, contratos de columnas, calidad de datos, permisos, referencias a Workspace y resultados del proceso. En un Pipeline, comprobar que su actividad Notebook usa el Lakehouse adecuado. Una prueba de integración debería detectar referencias a una tabla de desarrollo que accidentalmente permanezcan en producción.

**Fuente:** [Understand CI/CD](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl-data-ai/implement-cicd-in-fabric/includes/2-understand-cicd.md).

## 3. Version control and Git integration

### 3.1. Git y colaboración

La integración Git de Fabric permite sincronizar definiciones de elementos admitidos con un repositorio configurado. Según el entorno y permisos, puede utilizarse GitHub o Azure DevOps. Antes de realizar cambios, comprobar los roles de Workspace, permiso de acceso al repositorio y la compatibilidad del tipo de elemento con Git.

El flujo típico usa una rama de trabajo, cambios revisados mediante pull request y su integración en la rama principal tras pruebas. Una PR permite discutir código y asegurar trazabilidad de decisiones. No se debe introducir en un repositorio contraseñas, claves o tokens; las conexiones deben gestionarse mediante mecanismos de identidad y configuración del entorno.

### 3.2. Conflictos y sincronización

Si dos colaboradores modifican el mismo artefacto, pueden surgir conflictos. Deben revisarse los cambios de ambas ramas y sus dependencias, no elegir automáticamente la última versión. Una modificación del schema de un Lakehouse puede obligar a ajustar Notebooks y modelos. El commit confirma una modificación de código, no que el proceso desplegado sea exitoso en datos reales.

### 3.3. Seguridad del repositorio

El repositorio puede ser privado y mantener ramas protegidas con revisiones obligatorias. Los permisos Git no equivalen a permisos para consultar datos en OneLake. Un desarrollador autorizado a modificar definiciones de Notebook no debería recibir automáticamente acceso a datos sensibles de producción.

**Fuente:** [Git integration](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl-data-ai/implement-cicd-in-fabric/includes/3-implement-version-control-and-git-integration.md).

## 4. Deployment pipelines

### 4.1. Etapas y asignación de Workspaces

Un **deployment pipeline** organiza etapas y facilita promover elementos compatibles entre entornos. Habitualmente las etapas son Development, Test y Production. Los Workspaces asociados deben contar con la configuración y permisos adecuados. Una etapa no debe utilizar un Workspace que contenga elementos productivos ajenos al ciclo de despliegue.

### 4.2. Qué se promueve

Los deployment pipelines promocionan los tipos de objetos y sus definiciones admitidos por la plataforma. **No hay que suponer que la promoción de una definición copie todo el contenido de datos del Lakehouse o Warehouse.** Los destinos, conexiones y parámetros necesitan una estrategia específica por entorno y unas pruebas posteriores al despliegue.

### 4.3. Reglas y diferencias entre entornos

La configuración de una conexión que apunta al Lakehouse de desarrollo no debería mantenerse en producción. Antes de ejecutar, validar referencias de Workspace, Lakehouse y Warehouse y los permisos de la identidad de producción. El cambio debe ser atómico en la medida que las herramientas lo permitan y, en caso de errores, existir procedimiento de reversión.

### 4.4. Pruebas post-deployment

Una promoción puede haberse completado sin que la ejecución posterior de un Notebook haya sido satisfactoria. Deben verificarse dependencias, parámetros, acceso al dato y resultados. La revisión del historial de versiones permite localizar qué cambio introdujo una regresión, pero la recuperación de datos puede necesitar medidas independientes de Git.

**Fuente:** [Deployment pipelines](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl-data-ai/implement-cicd-in-fabric/includes/4-implement-deployment-pipelines.md).

## 5. Automation with Fabric APIs

Las **Fabric REST APIs** permiten automatizar operaciones admitidas del ciclo de vida, evitando clics manuales repetidos. Un orquestador de CI/CD puede ejecutar autenticación mediante la identidad correspondiente, verificar elementos disponibles, gestionar despliegues y revisar estados de operación.

### 5.1. Autenticación y permisos

La automatización requiere una identidad autorizada y permisos del tenant, Workspace y API. No incrustar tokens en archivos del repositorio. Gestionar secretos en un almacén y utilizar identidades de servicio adecuadas. Conviene distinguir permiso para consultar un elemento de permiso para desplegarlo o ejecutar pipelines.

### 5.2. Idempotencia de despliegues

La operación debe ser segura ante reintentos. Si un servicio devuelve un error de timeout después de crear un elemento, repetir ciegamente una creación podría producir objetos duplicados o errores de conflicto. Antes de reintentar, consultar el estado real de la operación, identificador o recurso. Las APIs pueden evolucionar; verificar versiones, límites y campos en la referencia oficial.

### 5.3. Pipeline de validación

Un pipeline de CI/CD completo debería validar formato, rutas y dependencias de la definición antes de desplegar, revisar que las conexiones están parametrizadas, ejecutar tests y registrar la versión desplegada. Es un diseño complementario que debe ajustarse a los servicios y endpoints realmente admitidos.

**Fuente:** [Fabric APIs](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl-data-ai/implement-cicd-in-fabric/includes/5-automate-cicd-using-fabric-apis.md).

## 6. Laboratorio oficial

**Fuente:** [Implement deployment pipelines in Microsoft Fabric](https://github.com/MicrosoftLearning/mslearn-fabric/blob/main/Instructions/Labs/21-implement-cicd.md). **Duración aproximada:** 20 minutos; clasificado oficialmente como DP-700. **No ejecutado.**

### 6.1. Objetivo y requisitos

El ejercicio crea tres Workspaces y un deployment pipeline con etapas de desarrollo, prueba y producción. Después crea contenido y lo despliega entre las etapas. Se necesitan permisos de Workspace y capacidad compatible, además de autorización para crear y administrar pipelines de despliegue.

### 6.2. Crear Workspaces

Abrir [Fabric](https://app.fabric.microsoft.com/), crear Workspaces separados para desarrollo, test y producción, con nombres inequívocos. No emplear Workspaces reales de negocio. Revisar que todos están en la capacidad y configuración adecuada.

### 6.3. Crear y asignar deployment pipeline

Crear un deployment pipeline desde la experiencia correspondiente, definir sus etapas y asociar cada Workspace. Comprobar que la etapa Development no referencia el Workspace de producción. Registrar las asignaciones antes de crear contenido.

### 6.4. Crear contenido

Generar el objeto de ejemplo descrito en el laboratorio dentro del Workspace de desarrollo. Confirmar que el pipeline reconoce el objeto como elemento desplegable. Los tipos y propiedades compatibles dependen de la plataforma actual.

### 6.5. Desplegar y comprobar

Desplegar de Development a Test y después, cuando se compruebe el resultado, a Production según la práctica. Verificar que aparecen los elementos correctos en sus Workspaces. No atribuir datos operativos ni resultados de consultas a la copia de un artefacto sin ejecutar las actividades correspondientes.

### 6.6. Errores frecuentes y limpieza

| Síntoma | Posible causa | Corrección |
|---|---|---|
| No se puede asignar Workspace | Permisos o capacidad | Revisar roles |
| Elemento no desplegable | Tipo o configuración no admitidos | Verificar compatibilidad |
| Conexión sigue apuntando a Dev | Falta parametrización de destino | Revisar reglas antes de ejecución |
| Workspaces contienen objetos inesperados | Entorno compartido | Detener y verificar antes de borrar |
| Deployment correcto, ejecución falla | Dependencias y accesos diferentes | Probar permisos de runtime |

El laboratorio termina con limpieza. Eliminar únicamente el deployment pipeline y Workspaces creados para prueba, comprobando que no contienen datos ajenos.

## 7. Knowledge check y summary

Preguntas propias: ¿Qué diferencia CI de CD? ¿Qué conserva Git y qué no? ¿Qué representa una etapa de deployment? ¿Por qué hay que cambiar conexiones por entorno? ¿Cuál es el valor de un pull request? ¿Cómo debe gestionarse la identidad de una API? ¿Qué probar después del despliegue?

**Summary:** CI/CD convierte cambios en artefactos de Fabric en entregas trazables y repetibles. Git, deployment pipelines y APIs son piezas diferentes que deben integrarse con pruebas, seguridad y control de datos.

## 8. Fuentes

- [Index de ocho unidades](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl-data-ai/implement-cicd-in-fabric/index.yml).
- [CI/CD](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl-data-ai/implement-cicd-in-fabric/includes/2-understand-cicd.md).
- [Git](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl-data-ai/implement-cicd-in-fabric/includes/3-implement-version-control-and-git-integration.md).
- [Deployments](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl-data-ai/implement-cicd-in-fabric/includes/4-implement-deployment-pipelines.md).
- [APIs](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl-data-ai/implement-cicd-in-fabric/includes/5-automate-cicd-using-fabric-apis.md).
- [Lab DP-700](https://github.com/MicrosoftLearning/mslearn-fabric/blob/main/Instructions/Labs/21-implement-cicd.md).

**Control:** teoría, ejercicio, repaso y síntesis redactados; no ejecutados.