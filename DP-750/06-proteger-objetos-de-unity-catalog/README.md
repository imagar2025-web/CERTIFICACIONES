[🏠 Índice](../README.md) · [← Anterior](../05-crear-y-organizar-objetos-en-unity-catalog/README.md) · [Siguiente →](../07-gobernar-objetos-de-unity-catalog/README.md)

> **Estado de estudio:** ✅ Visto · **Unidades oficiales:** 11 · [Microsoft Learn](https://learn.microsoft.com/en-us/training/modules/secure-unity-catalog-objects/)  
> **Versión de apuntes:** 14-11-2026 · Contenido técnico redactado de forma original a partir del itinerario de Microsoft.

# 6. Proteger objetos de Unity Catalog

**Referencia Microsoft Learn:** [Proteger objetos de Unity Catalog](https://learn.microsoft.com/en-us/training/modules/secure-unity-catalog-objects/) · **11 unidades oficiales**.

## 6.1. El ciclo de vida de una consulta segura

Una consulta SQL no «entra directamente» a un archivo Delta. Primero se identifica al usuario o aplicación, se interpreta el objeto solicitado, se comprueban autorizaciones y se ejecuta un plan en el cómputo. El resultado puede depender de políticas adicionales de columnas o filas. Tener permiso en el almacenamiento subyacente no debería ser el mecanismo para saltarse Unity Catalog.

```text
Usuario / identidad de servicio
  → autenticación
  → resolución catalog.schema.object
  → comprobación de privilegios y políticas
  → planificación y ejecución
  → resultados permitidos
```

## 6.2. Privilegios, propietarios y grupos

El principio rector es **mínimo privilegio**: otorgar solo los permisos necesarios durante el tiempo necesario. En lugar de gestionar usuarios uno a uno, se recomienda usar grupos vinculados a roles organizativos. Además, hay que diferenciar quién puede descubrir un objeto, utilizar su contenedor, consultar filas y administrarlo.

```sql
GRANT USE CATALOG ON CATALOG estudio TO `analistas`;
GRANT USE SCHEMA ON SCHEMA estudio.ventas TO `analistas`;
GRANT SELECT ON TABLE estudio.ventas.pedidos TO `analistas`;

SHOW GRANTS ON TABLE estudio.ventas.pedidos;
```

Los permisos efectivos pueden incorporar herencia desde catálogos y esquemas según los privilegios y reglas del objeto. Un privilegio `SELECT` sobre una tabla no sustituye los permisos necesarios para resolver su catálogo/esquema. `MANAGE` y propiedad no son sinónimos de lectura ilimitada en todas las situaciones.

## 6.3. Control granular: filas y columnas

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

## 6.4. Secretos con Azure Key Vault

Una API externa, un sistema origen o una integración puede necesitar secretos. **Azure Key Vault** administra secretos y controles de acceso; Databricks dispone de métodos de integración compatibles, como ámbitos de secretos respaldados por Key Vault según la modalidad. Jamás incluyas tokens, contraseñas o claves reales en notebooks de ejemplo, Git o salidas de celdas.

```python
# Ejemplo conceptual en un entorno que tenga el secret scope configurado
api_token = dbutils.secrets.get(scope="scope-datos", key="api-token")
# No mostrar, imprimir ni registrar api_token
```

Aunque un secreto esté enmascarado en algunas salidas, el código debe evitar filtrarlo por trazas, excepciones o llamadas externas no autorizadas.

## 6.5. Service principals

Un **service principal** es una identidad de aplicación que permite ejecutar procesos automatizados sin depender de la sesión de un humano. Puede tener los privilegios mínimos para un job determinado. Si utiliza credenciales de cliente, se deben almacenar con seguridad, rotarlas y controlar quién puede acceder a ellas.

**Caso de uso:** un pipeline nocturno únicamente puede leer `bronze` y escribir `silver`. No es correcto asignarle por comodidad el papel de administrador de metastore.

## 6.6. Managed identities

Las **identidades administradas** de Azure permiten autenticación hacia recursos compatibles sin administrar directamente secretos de aplicación. Azure controla su ciclo de credenciales. Se utilizan frecuentemente para conectar a ADLS Gen2 mediante las credenciales de almacenamiento y las ubicaciones externas de Unity Catalog.

Comparación: service principal con secreto → el equipo custodia y rota el secreto; managed identity → Azure administra la credencial, aunque el equipo sigue gestionando asignación de roles y privilegios.

## 6.7. Laboratorio y evaluación

Crear un grupo de lectura, concederle uso de catálogo y esquema y `SELECT` de una tabla; verificar que no puede escribir. Diseñar una política de filtro regional y una columna sensible enmascarada. Identificar qué credenciales se necesitan para leer ADLS sin introducir secretos en el notebook.

**Repaso:** diferencia autenticación y autorización; cuándo usar Key Vault; qué controla una managed identity; por qué el acceso al clúster y el acceso a la tabla son permisos distintos.

---

---

## 🧪 Guía de laboratorio y comprobación práctica

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


---

### 📚 Fuentes y navegación

- [Contenido oficial de Microsoft Learn](https://learn.microsoft.com/en-us/training/modules/secure-unity-catalog-objects/)
- [Repositorio docente de laboratorios DP-750T00](https://microsoftlearning.github.io/DP-750T00-Implement-Data-Engineering-Solutions-using-Azure-Databricks/)
- [Guía oficial de competencias del examen DP-750](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-750)

[🏠 Índice](../README.md) · [← Anterior](../05-crear-y-organizar-objetos-en-unity-catalog/README.md) · [Siguiente →](../07-gobernar-objetos-de-unity-catalog/README.md)
