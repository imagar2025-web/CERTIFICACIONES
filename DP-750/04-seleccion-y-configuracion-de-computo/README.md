[🏠 Índice](../README.md) · [← Anterior](../03-integraciones-de-azure-databricks/README.md) · [Siguiente →](../05-crear-y-organizar-objetos-en-unity-catalog/README.md)

> **Estado de estudio:** ✅ Visto · **Unidades oficiales:** 9 · [Microsoft Learn](https://learn.microsoft.com/en-us/training/modules/select-and-configure-compute/)  
> **Versión de apuntes:** 15-11-2026 · Contenido técnico redactado de forma original a partir del itinerario de Microsoft.

# 4. Selección y configuración de cómputo

**Referencia Microsoft Learn:** [Selección y configuración de cómputo](https://learn.microsoft.com/en-us/training/modules/select-and-configure-compute/) · **9 unidades oficiales**.

## 4.1. Introducción: el cómputo es una decisión de arquitectura

El dato puede permanecer en almacenamiento duradero mientras el cómputo se crea, escala o termina. Esta separación permite controlar coste y rendimiento. La mejor configuración no es necesariamente el clúster más potente: el objetivo es cubrir los requisitos de tiempo de respuesta, concurrencia, seguridad, conectividad y presupuesto.

## 4.2. Elegir el tipo de cómputo

- **Serverless:** recursos administrados por Databricks que simplifican el aprovisionamiento. Adecuados cuando la carga está soportada y no se necesitan controles de infraestructura exclusivos de classic.
- **All-purpose / interactive:** entorno pensado para exploración, desarrollo iterativo y notebooks de usuarios. Mantenerlo encendido sin uso supone coste.
- **Jobs compute:** ejecución automatizada de trabajos o pipelines, con ciclo de vida asociado a tareas. Resulta habitual para ETL programado.
- **SQL warehouse:** optimizado para consultas SQL, dashboards y clientes BI; conviene estudiar tamaño, autoescalado, concurrencia y pausa automática.

| Escenario | Decisión habitual | Motivo |
|---|---|---|
| Pruebas interactivas de notebook | All-purpose o serverless compatible | Iteración rápida |
| ETL diario automatizado | Jobs compute / serverless compatible | Aislamiento por ejecución |
| Consultas BI concurrentes | SQL warehouse | Motor y escalado para SQL |
| Requisitos especiales de red | Classic, si es necesario | Control en la suscripción |

## 4.3. Rendimiento: nodos, escalado y terminación

Spark reparte el trabajo entre un **driver**, que coordina, y los **executors**, que procesan particiones y realizan cálculos distribuidos. El tamaño de los nodos depende de CPU, memoria y, para algunos escenarios, de recursos especializados. Si hay muy poca memoria, aparecen derrames a disco y errores OOM; si hay pocos cores, falta paralelismo; si se amplía indiscriminadamente, crece el coste sin mejora proporcional.

El **autoscaling** ajusta el número de workers dentro de límites establecidos, pero no elimina un mal diseño de particiones o un shuffle excesivo. **Auto termination** apaga cómputo inactivo para evitar gasto. Una carga con muchísimos ficheros pequeños puede rendir peor que otra con archivos compactados, aunque tenga el mismo volumen total.

## 4.4. Databricks Runtime, Photon y características

**Databricks Runtime (DBR)** reúne versiones compatibles de Spark, bibliotecas y optimizaciones. Elegir una versión LTS puede favorecer la estabilidad de los proyectos. **Photon** es un motor vectorizado para acelerar operaciones analíticas compatibles; no supone una mejora garantizada para todo código Python o cualquier carga externa al motor de consultas.

Las políticas del clúster pueden restringir tamaños, versiones, etiquetas y valores de configuración; esto ayuda a gobernar seguridad y costes. También se examinan detalles como el soporte de arquitecturas concretas y límites de cada modalidad.

## 4.5. Instalar bibliotecas

Las dependencias pueden instalarse mediante mecanismos compatibles con notebooks, recursos de compute, repositorios de paquetes y archivos del proyecto. Para un equipo es importante declarar las versiones en un `requirements.txt` o equivalente. Las instalaciones efímeras en un notebook no garantizan que una ejecución posterior utilice exactamente el mismo entorno.

```text
pyspark>=3.5
pandas>=2.0
```

> Ejemplo ilustrativo: el runtime puede incorporar sus propias versiones de PySpark; **no** instales una versión diferente de Spark en el clúster sin verificar compatibilidad.

Las bibliotecas de fuentes no confiables representan un riesgo de ejecución de código. Usar ubicaciones gobernadas, perfiles mínimos y dependencias revisadas.

## 4.6. Accesos al cómputo

Hay que diferenciar **permiso para usar un clúster**, **permiso para administrarlo** y **permiso para leer los datos**. Unity Catalog verifica privilegios sobre objetos incluso cuando un usuario puede conectarse al cómputo. Los modos de acceso, incluidos los modelos de acceso compartido estándar y dedicado, determinan compatibilidad y aislamiento según la configuración disponible.

Seguridad recomendada: conceder permisos a grupos, usar identidades de servicio para automatizaciones, controlar bibliotecas, aplicar políticas de clúster y evitar clústeres compartidos innecesariamente privilegiados.

## 4.7. Ejercicio guiado propio

1. Identificar una tarea ETL que corre una vez al día y una carga BI con consultas concurrentes.
2. Seleccionar jobs compute para la primera y SQL warehouse para la segunda, justificando excepciones.
3. Proponer mínimo y máximo de workers, apagado automático y versión LTS.
4. Enumerar permisos requeridos para un desarrollador, un administrador y una identidad de automatización.
5. Registrar métricas de tiempo y coste, y comparar otra configuración sobre la misma carga.

**Claves de evaluación:** `serverless ≠ gratuito`, Photon no acelera todo, tener acceso al cluster no concede acceso a tablas.

---

---


## Desarrollo en profundidad: dimensionar, aislar y pagar el cómputo

### Por qué el número de nodos no explica el rendimiento

El rendimiento depende de los recursos disponibles y de la forma real en que Spark divide el trabajo. Los cores condicionan el número de tareas que pueden progresar simultáneamente; la memoria determina el tamaño de estructuras intermedias antes de escribir en disco; el almacenamiento local se emplea en shuffle y cachés. Un cuello de botella de CPU puede responder al aumento de paralelismo, mientras que un join que provoca *spill* necesita investigar memoria, cardinalidad y estrategia de unión. No hay una configuración universalmente más rápida.

Para estudiar una carga, registra volumen de entrada, duración, número de tareas, utilización de CPU, memoria, bytes de shuffle y datos escritos a disco. Cambia **una** variable de cada vez en las pruebas. Si al duplicar recursos solo reduces un 5 % la duración, quizá la consulta está limitada por I/O, skew, sobrecarga inicial o paralelismo insuficiente.

### Tamaño de nodos frente a cantidad de workers

Dos clústeres con el mismo total de cores pueden comportarse de modo diferente. Un conjunto pequeño de nodos grandes reduce algunas comunicaciones entre máquinas, pero ofrece menos unidades físicas de escalado. Uno de nodos pequeños puede mejorar distribución de tareas y elasticidad, aunque introduce más comunicación en ciertos patrones. Los tipos de VM optimizados para memoria, CPU o almacenamiento tienen usos diferentes; hay que comprobar su disponibilidad regional y compatibilidad actual en Azure.

### Autoscaling y auto-termination

El escalado automático ajusta trabajadores dentro de límites configurados. Es útil ante variabilidad de carga, pero no convierte automáticamente una consulta mal diseñada en eficiente. La terminación automática reduce consumo de clústeres inactivos; debe configurarse según si el cómputo es interactivo o de trabajos programados. Una carga corta frecuente puede sufrir por arranques repetidos, mientras que mantener cómputo encendido sin uso supone coste. Distingue siempre costes de infraestructura, consumo de DBU y los posibles costes de servicios dependientes.

### Modos de acceso y autorización

Seleccionar recursos implica también establecer quién puede usarlos, modificarlos o administrarlos. Los modos de acceso apropiados dependen del aislamiento, compatibilidad de Runtime, bibliotecas y políticas corporativas. El permiso de adjuntar un notebook no debe interpretarse como permiso de administrar todo el recurso. Es recomendable conceder el nivel mínimo necesario y verificar que los usuarios acceden a Unity Catalog con su propia identidad y políticas.

### Comparación operativa de modalidades

| Necesidad | Modalidad a evaluar | Aspectos que debes comprobar |
|---|---|---|
| Análisis puntual mediante SQL | SQL warehouse | Latencia, escalado, concurrencia, coste |
| Notebook de exploración | Cómputo interactivo o serverless compatible | Sesión, bibliotecas, permisos y arranque |
| ETL nocturno repetitivo | Cómputo de job | Terminación, reintentos, dependencias |
| Machine learning especializado | Runtime y hardware adecuados | GPU, bibliotecas, límites del servicio |

No des por supuesto que todas las características están disponibles en todas las regiones, versiones y planes. La documentación oficial es la autoridad para la lista exacta de modalidades soportadas.

### Mini laboratorio: comparar dos configuraciones

Elige una transformación repetible, por ejemplo agregación de pedidos y join con clientes. Ejecuta tres veces el mismo notebook en un entorno pequeño y registra la mediana de tiempo y métricas de Spark UI. Repite con un recurso diferente manteniendo datos y código iguales. Calcula tiempo, consumo y coste aproximado según la tarifa vigente de tu región. Después fuerza una salida amplia a disco y observa si aparece *spill*; explica por qué aumentar la memoria podría ayudar o por qué un filtro temprano sería más económico.

```python
from pyspark.sql import functions as F
resultado = (spark.table("estudio.silver.pedidos")
    .filter(F.col("estado") == "confirmado")
    .groupBy("id_cliente")
    .agg(F.sum("importe").alias("total")))
resultado.explain("formatted")
```

Recuerda que `explain()` muestra el plan estimado y las métricas de la ejecución real se consultan en las herramientas de perfilado correspondientes. No compares solo la duración de la celda si en una prueba hay caché y en otra no.

---

## 🧪 Guía de laboratorio y comprobación práctica

**Ejercicio oficial:** [Abrir las instrucciones de MicrosoftLearning](https://microsoftlearning.github.io/DP-750T00-Implement-Data-Engineering-Solutions-using-Azure-Databricks/Instructions/Labs/02-select-and-configure-compute.html)  
**Notebook oficial:** [Importar o consultar notebook `.ipynb`](https://raw.githubusercontent.com/MicrosoftLearning/DP-750T00-Implement-Data-Engineering-Solutions-using-Azure-Databricks/refs/heads/main/Allfiles/02-select-and-configure-compute.ipynb)  
**Naturaleza de esta sección:** itinerario explicado con redacción propia y comprobaciones de aprendizaje. Las celdas exactas y el material de origen están en el notebook oficial enlazado; no se reproducen íntegramente aquí.


### Caso práctico: HealthBridge Analytics — preparar el cómputo

**Objetivo:** elegir un clúster de desarrollo compartido, definir escalado y parada automática, instalar dependencias y analizar datos sanitarios sintéticos. El laboratorio combina acciones en el panel de cómputo con ejercicios en notebook.

**Prerrequisitos:** workspace Premium y permiso para crear cómputo. Algunas políticas de la organización pueden impedir seleccionar configuraciones sin restricciones.

**Recorrido contrastado con la guía oficial:**

1. Abre **Compute → Create compute** y cambia a la configuración avanzada. Prepara un clúster de uso general, de varios nodos, con modo de acceso **Standard**, llamado `healthbridge-dev`.
2. Escoge una versión reciente **LTS** de Databricks Runtime. Para este caso orientado a joins y análisis de historiales, considera nodos con memoria suficiente; los tamaños disponibles dependen de cuota y región.
3. Configura **autoscaling** con mínimo **1** y máximo **3** workers; activa auto-terminación a los **15 minutos** de inactividad. Comprueba que Photon se encuentre habilitado cuando lo admita la configuración.
4. Crea el clúster. En su pestaña **Libraries** agrega desde PyPI la dependencia `faker==40.8.0` y comprueba el estado de instalación. Fijar la versión evita que el comportamiento cambie silenciosamente.
5. Al finalizar la parte de infraestructura, termina o elimina el clúster para evitar costes. Esta acción es relevante: el notebook restante usa **Serverless**, no el clúster que acabas de aprovisionar.
6. Importa el notebook de Microsoft, selecciónale **Serverless** y sigue las prácticas de `%pip`, verificación de bibliotecas, generación de registros ficticios de pacientes y exploración con PySpark. **Los datos son sintéticos**, no información clínica real.

**Qué debes comprobar:** diferencia entre biblioteca instalada para todo el clúster y una biblioteca de ámbito notebook; uso de `import` y consulta de versiones; cómo cambian coste y disponibilidad según el ciclo de vida del cómputo.

**Problemas frecuentes:** falta de privilegios para crear clústeres, restricciones de cuota Azure, confusión entre clúster clásico y serverless, instalación de dependencias incompatible con el Runtime. Registra el error exacto antes de modificar ajustes.


---

### 📚 Fuentes y navegación

- [Contenido oficial de Microsoft Learn](https://learn.microsoft.com/en-us/training/modules/select-and-configure-compute/)
- [Repositorio docente de laboratorios DP-750T00](https://microsoftlearning.github.io/DP-750T00-Implement-Data-Engineering-Solutions-using-Azure-Databricks/)
- [Guía oficial de competencias del examen DP-750](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-750)

[🏠 Índice](../README.md) · [← Anterior](../03-integraciones-de-azure-databricks/README.md) · [Siguiente →](../05-crear-y-organizar-objetos-en-unity-catalog/README.md)
