# Preparación de laboratorios DP-750

[🏠 Volver al índice](README.md)

> Material de apoyo personal basado en el [laboratorio de preparación oficial](https://microsoftlearning.github.io/DP-750T00-Implement-Data-Engineering-Solutions-using-Azure-Databricks/Instructions/Labs/00-setup.html). No ejecutes scripts de recursos sin revisar su contenido, coste y alcance.

## Antes de comenzar

- Necesitas una suscripción Azure con permisos para crear un **Azure Databricks Premium workspace** y recursos asociados.
- Algunos ejercicios exigen metastore de Unity Catalog, privilegios para crear catálogos, cómputo y, para Key Vault, permisos administrativos adicionales.
- Las instrucciones del curso utilizan nombres como `adb-dp750` y `rg-dp750`, además de cómputo **Serverless** cuando está disponible.
- Los servicios, el cómputo y el almacenamiento pueden generar costes. Revisa la suscripción, la región elegida, límites presupuestarios y las políticas del laboratorio.

## Aprovisionamiento oficial, sin ejecutar automáticamente

1. Abre [portal.azure.com](https://portal.azure.com) con la suscripción apropiada.
2. Accede a **Azure Cloud Shell (Bash)**, selecciona la suscripción y revisa el script de aprovisionamiento referenciado en la documentación de MicrosoftLearning antes de ejecutarlo.
3. Verifica en el portal el grupo de recursos y el workspace creados. Abre el workspace desde su recurso en Azure.
4. En Databricks, comprueba **Catalog Explorer**, **Compute**, **Jobs & Pipelines** y **Workspace**. Revisa que tu identidad tenga permiso para crear los recursos que necesiten los ejercicios.
5. Importa los notebooks desde sus URLs oficiales, dentro de carpetas propias de tu workspace. **No importes los notebooks dentro de este repositorio de apuntes** salvo que quieras guardar tus propias soluciones y hayas comprobado las licencias y permisos del código de Microsoft.

## Antes y después de cada laboratorio

| Antes | Después |
|---|---|
| Verificar permisos, ubicación y costes | Capturar resultados y métricas principales |
| Identificar catálogo/esquema de destino | Verificar tablas, vistas, volúmenes y jobs creados |
| Leer la guía y el notebook antes de ejecutar | Terminar el cómputo de desarrollo innecesario |
| Anotar el nombre del recurso de pruebas | Eliminar solo recursos propios y descartables |

> **Atención a los borrados:** instrucciones con `DROP CATALOG ... CASCADE` y scripts de limpieza de grupos de recursos pueden suprimir datos permanentes. Revisa siempre el contexto y conserva cualquier trabajo necesario.

## Colección oficial de laboratorios

[MicrosoftLearning · DP-750T00](https://microsoftlearning.github.io/DP-750T00-Implement-Data-Engineering-Solutions-using-Azure-Databricks/). Los módulos de **arquitectura** e **integraciones** se estudian en teoría; la colección de laboratorios independiente consultada incluye ejercicios para los otros módulos y labores de configuración inicial.
