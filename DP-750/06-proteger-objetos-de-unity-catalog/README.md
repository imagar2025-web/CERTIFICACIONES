# 06 · Proteger objetos de Unity Catalog

[Índice DP-750](../README.md) · [← Crear y organizar objetos de Unity Catalog](../05-crear-y-organizar-objetos-en-unity-catalog/README.md) · [Gobernar objetos de Unity Catalog →](../07-gobernar-objetos-de-unity-catalog/README.md)

**Progreso personal:** Estudiado. [Módulo oficial](https://learn.microsoft.com/en-us/training/modules/secure-unity-catalog-objects/).

## Unidades

1. [Introduction](#unidad-01)
2. [Understand query lifecycle](#unidad-02)
3. [Implement access control strategies](#unidad-03)
4. [Understand fine-grained access control](#unidad-04)
5. [Implement row filtering and column masking](#unidad-05)
6. [Access Azure Key Vault secrets](#unidad-06)
7. [Authenticate data access with service principals](#unidad-07)
8. [Authenticate resource access with managed identities](#unidad-08)
9. [Exercise - Secure Unity Catalog Objects](#unidad-09)
10. [Module assessment](#unidad-10)
11. [Summary](#unidad-11)

- [Desarrollo aplicado](#desarrollo-aplicado)
- [Laboratorio y comprobaciones](#laboratorio-y-comprobaciones)

## Unidad 01

### Introduction

En una plataforma compartida, el usuario ejecuta una operación como un principal concreto y el servicio decide si puede atravesar el espacio de nombres y realizar la acción solicitada. La protección empieza por saber quién ejecuta la carga y qué activo utiliza. Para leer una tabla, los permisos sobre catálogo, esquema y tabla cumplen funciones distintas. Los filtros de fila y las máscaras de columna añaden condiciones sobre el resultado. La conectividad y autorización de Azure Storage resuelven otra parte del recorrido.

[Unidad oficial](https://learn.microsoft.com/en-us/training/modules/secure-unity-catalog-objects/1-introduction)

## Unidad 02

### Understand query lifecycle

Una consulta SQL no «entra directamente» a un archivo Delta. Primero se identifica al usuario o aplicación, se interpreta el objeto solicitado, se comprueban autorizaciones y se ejecuta un plan en el cómputo. El resultado puede depender de políticas adicionales de columnas o filas. Tener permiso en el almacenamiento subyacente no debería ser el mecanismo para saltarse Unity Catalog.

```text
Usuario / identidad de servicio
  → autenticación
  → resolución catalog.schema.object
  → comprobación de privilegios y políticas
  → planificación y ejecución
  → resultados permitidos
```

[Unidad oficial](https://learn.microsoft.com/en-us/training/modules/secure-unity-catalog-objects/2-understand-query-lifecycle)

## Unidad 03

### Implement access control strategies

El principio rector es **mínimo privilegio**: otorgar solo los permisos necesarios durante el tiempo necesario. En lugar de gestionar usuarios uno a uno, se recomienda usar grupos vinculados a roles organizativos. Además, hay que diferenciar quién puede descubrir un objeto, utilizar su contenedor, consultar filas y administrarlo.

```sql
GRANT USE CATALOG ON CATALOG estudio TO `analistas`;
GRANT USE SCHEMA ON SCHEMA estudio.ventas TO `analistas`;
GRANT SELECT ON TABLE estudio.ventas.pedidos TO `analistas`;

SHOW GRANTS ON TABLE estudio.ventas.pedidos;
```

Los permisos efectivos pueden incorporar herencia desde catálogos y esquemas según los privilegios y reglas del objeto. Un privilegio `SELECT` sobre una tabla no sustituye los permisos necesarios para resolver su catálogo/esquema. `MANAGE` y propiedad no son sinónimos de lectura ilimitada en todas las situaciones.

#### Concesión granular frente a herencia

Determina el nivel que refleja el contrato de acceso. Si un grupo debe leer todas las tablas actuales y futuras de un esquema de consumo y la política de publicación admite ese alcance, un grant en el esquema reduce altas manuales. Si solo necesita una tabla, concede `SELECT` sobre ella junto con los permisos de uso de los padres. La delegación para administrar grants pertenece al ámbito administrativo. Para verificar acceso utiliza una cuenta del grupo que no sea propietaria. Al retirar un grant directo, comprueba también los que pueda heredar por pertenencia a otros grupos o desde objetos padres.

[Unidad oficial](https://learn.microsoft.com/en-us/training/modules/secure-unity-catalog-objects/3-implement-access-control-strategies)

## Unidad 04

### Understand fine-grained access control

La seguridad granular cambia qué datos se entregan a un principal autorizado. Una vista dinámica puede encapsular una selección de filas y columnas y exponer un contrato específico. Un row filter se asocia al objeto y determina qué filas pasan un predicado. Una column mask transforma el valor visible de una columna según una función. Comprueba si tus consumidores pueden llegar a la tabla base por otra vía y verifica la compatibilidad de las operaciones de cómputo y DML que utilizará la carga. Una máscara que deja intactos los valores para un grupo de confianza debe probarse con usuarios de dentro y fuera del grupo, usando datos de prueba, y manteniendo el tipo esperado por la columna.

[Unidad oficial](https://learn.microsoft.com/en-us/training/modules/secure-unity-catalog-objects/4-understand-fine-grained-access-control)

## Unidad 05

### Implement row filtering and column masking

**Row filtering** define qué filas puede consultar un usuario. **Column masking** define qué valor ve en una columna según reglas, a menudo basadas en identidad o pertenencia a grupos. Ambos requieren que la política se evalúe en el sistema de consulta autorizado.

Ejemplo conceptual: analistas regionales ven únicamente operaciones de su región; los usuarios no autorizados ven correos enmascarados. Una vista filtrada puede ser útil, pero las políticas nativas ofrecen un mecanismo coherente y gobernado para cubrir más consultas, sujeto a compatibilidad.

```sql
-- Patrón orientativo: crear una función de filtrado y asignarla a una tabla.
CREATE OR REPLACE FUNCTION estudio.ventas.puede_ver_region(region STRING)
RETURNS BOOLEAN
RETURN is_account_group_member('administradores_datos')
   OR region = 'publica';

-- La disponibilidad y los permisos de ALTER TABLE ... SET ROW FILTER
-- dependen del entorno y las funciones admitidas.
```

El filtrado de filas no equivale al cifrado, y la máscara no elimina los datos almacenados: cambia la proyección que se devuelve al consultante.

[Unidad oficial](https://learn.microsoft.com/en-us/training/modules/secure-unity-catalog-objects/5-implement-row-filtering-column-masking)

## Unidad 06

### Access Azure Key Vault secrets

Una API externa, un sistema origen o una integración puede necesitar secretos. **Azure Key Vault** administra secretos y controles de acceso; Databricks dispone de métodos de integración compatibles, como ámbitos de secretos respaldados por Key Vault según la modalidad. Jamás incluyas tokens, contraseñas o claves reales en notebooks de ejemplo, Git o salidas de celdas.

```python
# Ejemplo conceptual en un entorno que tenga el secret scope configurado
api_token = dbutils.secrets.get(scope="scope-datos", key="api-token")
# No mostrar, imprimir ni registrar api_token
```

Aunque un secreto esté enmascarado en algunas salidas, el código debe evitar filtrarlo por trazas, excepciones o llamadas externas no autorizadas.

[Unidad oficial](https://learn.microsoft.com/en-us/training/modules/secure-unity-catalog-objects/6-access-azure-key-vault)

## Unidad 07

### Authenticate data access with service principals

Un **service principal** es una identidad de aplicación que permite ejecutar procesos automatizados sin depender de la sesión de un humano. Puede tener los privilegios mínimos para un job determinado. Si utiliza credenciales de cliente, se deben almacenar con seguridad, rotarlas y controlar quién puede acceder a ellas.

**Caso de uso:** un pipeline nocturno únicamente puede leer `bronze` y escribir `silver`. No es correcto asignarle por comodidad el papel de administrador de metastore.

[Unidad oficial](https://learn.microsoft.com/en-us/training/modules/secure-unity-catalog-objects/7-authenticate-data-access-service-principals)

## Unidad 08

### Authenticate resource access with managed identities

Las **identidades administradas** de Azure permiten autenticación hacia recursos compatibles sin administrar directamente secretos de aplicación. Azure controla su ciclo de credenciales. Se utilizan frecuentemente para conectar a ADLS Gen2 mediante las credenciales de almacenamiento y las ubicaciones externas de Unity Catalog.

Comparación: service principal con secreto → el equipo custodia y rota el secreto; managed identity → Azure administra la credencial, aunque el equipo sigue gestionando asignación de roles y privilegios.

[Unidad oficial](https://learn.microsoft.com/en-us/training/modules/secure-unity-catalog-objects/8-authenticate-resource-access-managed-identities)

## Unidad 09

### Exercise - Secure Unity Catalog Objects

Crear un grupo de lectura, concederle uso de catálogo y esquema y `SELECT` de una tabla; verificar que no puede escribir. Diseñar una política de filtro regional y una columna sensible enmascarada. Identificar qué credenciales se necesitan para leer ADLS sin introducir secretos en el notebook.

**Repaso:** diferencia autenticación y autorización; cuándo usar Key Vault; qué controla una managed identity; por qué el acceso al clúster y el acceso a la tabla son permisos distintos.

[Unidad oficial](https://learn.microsoft.com/en-us/training/modules/secure-unity-catalog-objects/9-exercise-secure-unity-catalog-objects)

## Unidad 10

### Module assessment

**Caso de evaluación.** Un usuario puede ver el esquema, pero la consulta de la tabla devuelve una denegación. Revisa `USE CATALOG`, `USE SCHEMA` y `SELECT` efectivos. Si obtiene resultados con columnas ofuscadas, revisa la política o máscara aplicable. Si el fallo solo ocurre en un job, verifica Run as antes de modificar permisos. Para un secreto de conexión ajeno a Unity Catalog, utiliza un secret scope con acceso restringido y comprueba el estado de la referencia a Key Vault. Evita registrar el secreto en resultados o logs.

[Unidad oficial](https://learn.microsoft.com/en-us/training/modules/secure-unity-catalog-objects/10-knowledge-check)

## Unidad 11

### Summary

Documenta por carga su principal de ejecución, sus grants, sus dependencias de credenciales y el ámbito de exposición de cada objeto. En las comprobaciones de una política específica prueba principal autorizado, principal sin acceso y principal con acceso condicionado. Confirma con un job real cuando la prueba dependa de Run as. Una prueba interactiva con una identidad administradora solo valida ese contexto de ejecución.

[Unidad oficial](https://learn.microsoft.com/en-us/training/modules/secure-unity-catalog-objects/11-summary)

## Desarrollo aplicado

### Autenticación y autorización son preguntas distintas

La autenticación determina qué identidad intenta acceder. La autorización determina qué operaciones puede ejecutar esa identidad sobre un recurso. Que un usuario inicie sesión correctamente en el workspace no significa que tenga acceso a cualquier tabla de Unity Catalog. Los privilegios deben concederse según funciones reales, idealmente a grupos administrados, y revisarse cuando cambia el rol del empleado.

En Unity Catalog, acceder a una tabla normalmente exige poder resolver los niveles superiores del espacio de nombres y el permiso de lectura adecuado sobre el objeto. La herencia y la administración de permisos se deben comprobar en la versión actual del servicio, especialmente cuando se usan esquemas nuevos o modelos de seguridad específicos.

```sql
GRANT USE CATALOG ON CATALOG estudio TO `analistas`;
GRANT USE SCHEMA ON SCHEMA estudio.gold TO `analistas`;
GRANT SELECT ON TABLE estudio.gold.ventas_mensuales TO `analistas`;
```

La sintaxis anterior representa un ejemplo de concesión a un principal existente. No crea el grupo y debe ejecutarla alguien con privilegios suficientes. Un permiso `SELECT` no debe usarse como atajo para dar capacidad de escritura o administración.

### Seguridad por filas y columnas

En organizaciones multirregión, una persona puede necesitar solo los registros de su zona. Los filtros de fila responden a esta necesidad sin duplicar decenas de tablas similares. El enmascaramiento de columna puede ocultar identificadores personales o información sensible en consultas, manteniendo la lógica de acceso centralizada. La implementación concreta depende de funciones, políticas y características compatibles con Unity Catalog.

La diferencia principal es semántica: **filtro de filas** reduce qué registros se devuelven; **máscara** modifica qué valor se expone en una columna. Si una consulta usa datos sensibles para calcular agregados, revisa también la posibilidad de inferencia por totales, no únicamente si la columna aparece oculta.

### Identidades de servicio y secretos

Los pipelines necesitan identidad propia y principio de mínimo privilegio. Es preferible emplear identidades y credenciales administradas a pegar cadenas de conexión en notebooks. Los secretos no deben mostrarse en logs, tablas temporales, errores ni parámetros enviados a sistemas de observabilidad. Un repositorio privado tampoco constituye un almacén de contraseñas seguro: las credenciales deben permanecer fuera del código versionado.

### Amenazas habituales y controles

| Riesgo | Control recomendado | Verificación |
|---|---|---|
| Analista lee una tabla ajena | Grants a grupos con mínimo privilegio | Prueba con identidad restringida |
| Contraseña expuesta en un notebook | Identidad administrada / secret management | Revisión de repositorio y logs |
| Usuario consulta PII no necesaria | Filtros y máscaras | Casos con usuarios de diferentes roles |
| Pipeline ejecuta con cuenta personal | Principal de servicio con permisos delimitados | Revisar propietario del job |
| Cambios invisibles en permisos | Auditoría y revisión periódica | Consultar registros y responsables |

**Práctica reproducible:** crea dos grupos de laboratorio: `lectores` y `editores`. Concede lectura al primero y capacidades limitadas de modificación al segundo. Valida que ambos puedan descubrir los objetos previstos y que `lectores` no consiga escribir. Documenta cada error de autorización y distingue falta de acceso al catálogo, al esquema o a la tabla. Después retira los permisos de pruebas.

## Laboratorio y comprobaciones

**Ejercicio oficial:** [Abrir las instrucciones de MicrosoftLearning](https://microsoftlearning.github.io/DP-750T00-Implement-Data-Engineering-Solutions-using-Azure-Databricks/Instructions/Labs/04-secure-unity-catalog-objects.html)  
**Notebook oficial:** [Importar o consultar notebook `.ipynb`](https://raw.githubusercontent.com/MicrosoftLearning/DP-750T00-Implement-Data-Engineering-Solutions-using-Azure-Databricks/refs/heads/main/Allfiles/04-secure-unity-catalog-objects.ipynb)  
**Naturaleza de esta sección:** itinerario explicado con redacción propia y comprobaciones de aprendizaje. Las celdas exactas y el material de origen están en el notebook oficial enlazado; no se reproducen íntegramente aquí.


### Caso práctico: NorthMart Retail — seguridad granular

**Necesidad del negocio:** los analistas regionales deben ver exclusivamente sus territorios y la mayoría no debería recibir el correo completo de un cliente. La integración con un proveedor externo utiliza un secreto que no debe quedar escrito en el notebook.

**Pasos guiados:**

1. Importa el notebook oficial; examina la tabla de clientes y la columna con el territorio antes de aplicar controles.
2. En los ajustes de identidad de Databricks crea el grupo `retail-analysts` (solo si tienes permiso) y añade tu usuario de prueba. Concede al grupo los privilegios mínimos: uso del catálogo, uso del esquema y lectura de los objetos necesarios. Verifica con `SHOW GRANTS`.
3. Escribe una **función de filtro de filas** que compare la región de cada registro con el ámbito autorizado del usuario o del grupo. Vincúlala a la tabla; prueba la misma consulta con identidades diferentes cuando sea posible.
4. Implementa una **función de máscara** para el correo electrónico. La salida de usuarios autorizados puede mostrar el dato íntegro; otros deben ver una transformación irreversible o una versión parcialmente oculta, conforme al requisito.
5. Desde Azure Portal crea un **Key Vault** dedicado al laboratorio y guarda allí una clave de prueba, nunca una credencial productiva. El ejercicio de Microsoft usa políticas de acceso y enlaza el almacén con un **secret scope** del workspace (`retail-kv-scope`). Los requisitos administrativos y las políticas del servicio deben verificarse antes de configurar la vinculación.
6. Obtén el secreto desde el notebook con `dbutils.secrets.get(scope=..., key=...)`, pero **no lo imprimas**, no lo registres ni lo guardes en una variable de salida persistente. Comprueba solamente que existe o que se ha podido utilizar en una petición ficticia autorizada.

**Diagnóstico:** si un GRANT no basta, revisa el permiso de uso en toda la jerarquía; si una máscara no se aplica, verifica en qué tabla y columna se registró; si falla el secret scope, revisa permisos, URI del vault y políticas de identidad.

**Comprobaciones finales:** explica por qué los privilegios no sustituyen la seguridad de filas, por qué una máscara debe revisarse con distintos roles y por qué Key Vault no elimina la necesidad de controlar las identidades de acceso.

### Validación del ejercicio de seguridad

El laboratorio incluye pasos fuera del notebook. Prepara el grupo y sus miembros según las instrucciones. Antes del ejercicio de secretos revisa el modelo de permisos de Key Vault que exige el secret scope de la práctica y crea un valor ficticio. En el notebook, prueba la consulta como dos identidades o con dos usuarios de aula para observar el filtro de filas y la máscara de columna. Documenta qué filas fueron visibles y si el valor fue ofuscado, evitando incluir valores sensibles. La lectura del secreto debe comprobarse por el éxito de la operación que lo necesita, sin imprimir el valor. Si una identidad privilegiada muestra todos los registros, repite como consumidor. Si el scope falla, revisa el permiso del usuario sobre el scope y la configuración y conectividad del vault. Retira el scope y los recursos dedicados cuando dejen de usarse.

## Fuentes y navegación

- [Fuentes de la certificación](../FUENTES.md)
- [Preparación y limpieza del entorno](../PREPARACION-LABORATORIOS.md)

[Índice DP-750](../README.md) · [← Crear y organizar objetos de Unity Catalog](../05-crear-y-organizar-objetos-en-unity-catalog/README.md) · [Gobernar objetos de Unity Catalog →](../07-gobernar-objetos-de-unity-catalog/README.md)
