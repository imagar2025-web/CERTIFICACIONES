# 11 · Create Real-Time Dashboards with Microsoft Fabric

> **Certificación:** DP-700 · Real-Time Intelligence. **Módulo:** [Microsoft Learn](https://learn.microsoft.com/en-us/training/modules/create-real-time-dashboards-microsoft-fabric/) · [archivos fuente](https://github.com/MicrosoftDocs/learn/tree/main/learn-pr/wwl/create-real-time-dashboards-microsoft-fabric). **Laboratorio complementario:** [13-real-time-dashboards.md](https://github.com/MicrosoftLearning/mslearn-fabric/blob/main/Instructions/Labs/13-real-time-dashboards.md), etiquetado en MicrosoftLearning como DP-603, no específicamente DP-700. **Revisión:** noviembre de 2026. No ejecutado.

## Índice

1. [Introduction](#1-introduction)
2. [Get started with Real-Time Dashboards](#2-get-started-with-real-time-dashboards)
3. [Advanced features](#3-advanced-features)
4. [Best practices](#4-best-practices)
5. [Laboratorio complementario](#5-laboratorio-complementario)
6. [Knowledge check y summary](#6-knowledge-check-y-summary)
7. [Fuentes](#7-fuentes)

## 1. Introduction

Un dashboard convierte consultas en información visual para monitorizar un proceso. En **Real-Time Intelligence**, un **Real-Time Dashboard** reúne tiles que presentan resultados de consultas sobre datos actualizados con frecuencia, normalmente de bases KQL en un Eventhouse. Su objetivo es ayudar a detectar cambios y explorar el comportamiento de eventos recientes, no sustituir todo el catálogo de modelos semánticos de una organización.

Es importante entender la diferencia entre **dato actual**, **dato recientemente recibido** y **dato recientemente visualizado**. Si una estación de bicicletas no emite mensajes durante una hora, un dashboard que muestra el último mensaje puede parecer actualizado visualmente sin representar su estado presente. Por ello conviene incluir timestamp de origen y de ingestión cuando sea relevante. Un refresco automático del dashboard no convierte mensajes antiguos en datos frescos.

## 2. Get started with Real-Time Dashboards

### 2.1. Data source y permisos

El primer paso es configurar una **data source** que apunte a una KQL database autorizada. El dashboard no inventa el almacenamiento; requiere una fuente con datos. Las opciones de autenticación y autorización deben revisarse cuidadosamente: conceder acceso al dashboard no equivale a aprobar cualquier acceso a los datos subyacentes. Las identidades y permisos que consulta cada usuario deben ajustarse al modelo configurado por el servicio.

### 2.2. Tiles y consultas

Un **tile** presenta el resultado de una consulta, por ejemplo un contador, tabla, serie temporal o gráfico. La consulta define la semántica del indicador. Si se desea saber cuántos eventos ocurrieron por minuto, se necesita agrupar por ventanas temporales y filtrar el período de interés. Un tile no puede corregir un \`summarize\` que cuenta eventos duplicados.

~~~kql
Bikestream
| summarize Observaciones = count() by Neighbourhood
| sort by Observaciones desc
~~~

*Ejemplo propio sobre la tabla de la muestra Bike Rental*. El resultado cuenta observaciones, no puntos de bicicleta únicos. Si se busca el número de estaciones, habrá que definir e identificar correctamente una clave y el criterio de vigencia.

### 2.3. Visualizar datos

El editor permite elegir el tipo de visualización adecuado a las columnas devueltas por KQL. Para categorías se pueden utilizar barras; para evolución temporal, gráficos de líneas; para distribuciones geográficas, mapas cuando hay coordenadas fiables y el servicio lo soporta. Un indicador aislado sin título, unidad ni ventana temporal es difícil de interpretar.

Conviene utilizar títulos descriptivos como «Mensajes recibidos por barrio — últimos 30 minutos», indicando si es un recuento de eventos o de entidades. Las unidades y los filtros deben ser visibles o documentados para no inducir conclusiones erróneas.

**Fuente:** [Get started with Real-Time Dashboards](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/create-real-time-dashboards-microsoft-fabric/includes/2-get-started-with-real-time-dashboards.md).

## 3. Advanced features

### 3.1. Pages

Un dashboard puede organizarse en páginas para evitar una única pantalla excesivamente cargada. Por ejemplo, una página de operaciones presenta incidentes actuales y otra muestra evolución de capacidad y demanda. Una página no crea un aislamiento de seguridad por sí misma: los permisos siguen dependiendo de la configuración real de acceso.

### 3.2. Parameters

Los **parameters** permiten que el consumidor cambie filtros utilizados por los tiles, por ejemplo seleccionar un barrio o intervalo temporal. La interfaz debe evitar que los parámetros oculten el significado de una métrica. Si se cambia una ventana de diez minutos a siete días, debe quedar claro si se está sumando, promediando o seleccionando el último valor de cada entidad.

### 3.3. Auto refresh

El **auto refresh** actualiza la visualización con una frecuencia configurada. Aumentar la frecuencia puede mejorar la sensación de inmediatez, pero también incrementar consultas y consumo de capacidad. No debe confundirse con tasa real de llegada del origen. Una aplicación con un evento cada veinte minutos no se convierte en streaming de segundos por refrescar el dashboard una vez por segundo.

**Fuente:** [Advanced features](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/create-real-time-dashboards-microsoft-fabric/includes/3-advanced-features.md).

## 4. Best practices

### 4.1. Base queries

Las **base queries** permiten centralizar expresiones comunes y reutilizarlas desde diferentes tiles. Si cuatro gráficos necesitan filtrar a un subconjunto autorizado y normalizar columnas, resulta preferible definir la lógica compartida una vez que copiar cuatro versiones independientes. Sin embargo, debe examinarse si reutilizar una base query afecta a costes y rendimiento; no asumir una materialización automática de sus resultados.

~~~kql
// Expresión conceptual para una consulta base de un dashboard
Bikestream
| where isnotempty(Neighbourhood)
| project Neighbourhood, Street, No_Bikes, No_Empty_Docks
~~~

Cuando se utiliza como base query, los tiles pueden aplicar agregaciones específicas sobre los campos comunes. El detalle exacto de definición y referencia de bases depende del editor.

### 4.2. Rendimiento

En gráficos con múltiples tiles, cada consulta puede solicitar datos al motor. Utilizar filtros temporales apropiados y proyecciones limitadas evita trabajo innecesario. No es adecuado construir un mapa con cientos de miles de puntos sin agregación o un gráfico de línea sin intervalos definidos. La consulta debe devolver la granularidad que necesita el gráfico.

### 4.3. Gobernanza y diseño visual

Es conveniente documentar propietario, versión, fuente, fecha de revisión, permisos y latencia esperada. Un dashboard operativo debería dejar claro cómo interpretar valores ausentes y situaciones de error. Si un tile no puede consultar el Eventhouse, mostrar un dato desactualizado como si fuera actual es peligroso. Separar «no hay eventos» de «no hay conectividad» forma parte del diseño.

**Fuente:** [Best practices](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/create-real-time-dashboards-microsoft-fabric/includes/4-real-time-dashboards-best-practices.md).

## 5. Laboratorio complementario

**Fuente oficial:** [Get started with Real-Time Dashboards in Microsoft Fabric](https://github.com/MicrosoftLearning/mslearn-fabric/blob/main/Instructions/Labs/13-real-time-dashboards.md) · [versión navegable](https://microsoftlearning.github.io/mslearn-fabric/Instructions/Labs/13-real-time-dashboards.html). **Duración aproximada:** 25 minutos. **Catálogo de cursos:** DP-603; incorporado aquí como apoyo técnico a una unidad real de la ruta DP-700. **Sin ejecución.**

### 5.1. Escenario y requisitos

El ejercicio crea un Workspace, Eventhouse y Eventstream para alimentar un dashboard con datos de eventos. El alumno crea tiles, una base query, un parámetro, una página adicional, configura refresco automático y guarda y comparte el dashboard. Requiere capacidad compatible, permisos de creación y acceso a la fuente de prueba.

### 5.2. Preparar la fuente

Crear un Workspace de laboratorio en [Fabric](https://app.fabric.microsoft.com/), Eventhouse con KQL database y Eventstream según las instrucciones oficiales. Verificar que el destino recibe eventos. Sin este paso, la creación de tiles puede ser correcta y aun así mostrar resultados vacíos.

### 5.3. Crear dashboard y tiles

Crear un Real-Time Dashboard y asociarlo a la KQL database correspondiente. Agregar el primer tile con una consulta de inspección que tenga el esquema esperado y elegir una visualización. Revisar título, unidades, filtro de tiempo y formato. Una consulta de \`take\` puede servir para comprobar columnas, pero un gráfico operativo requiere un agregado o selección más precisa.

### 5.4. Base query, parámetro y páginas

Definir una base query, y a continuación crear tiles que utilicen dicha lógica común. Añadir un parámetro de filtro, comprobar que el cambio de valor afecta a los tiles pertinentes y crear una página adicional para agrupar visualizaciones. El ejercicio oficial incluye estas operaciones; no se han inventado valores de parámetros ni resultados de consulta.

### 5.5. Auto refresh, guardar y compartir

Configurar un intervalo de refresco apropiado. Confirmar que las consultas siguen funcionando y guardar el dashboard. Si se comparte, comprobar el destinatario y permisos. Evitar compartir datos de un tenant empresarial con cuentas ajenas durante la práctica.

### 5.6. Diagnóstico

| Problema | Posible causa | Solución |
|---|---|---|
| Tiles vacíos | No hay eventos o consulta no coincide | Comprobar KQL en Queryset |
| Gráfico mal representado | Tipo de visualización inadecuado | Revisar columnas y tipo |
| Parámetro no filtra | No está vinculado a consulta | Revisar definición y uso |
| Base query falla | Identificador o campos erróneos | Probar expresión independientemente |
| Auto refresh no refleja nuevos datos | Fuente detenida o eventos antiguos | Comparar event time e ingest time |
| No puede compartir | Permisos o política de tenant | Verificar destinatario y roles |

**Limpieza:** eliminar únicamente el Workspace de prueba o los elementos propios. Detener el stream de ejemplo si continúa ejecutándose y revisar la capacidad.

## 6. Knowledge check y summary

Preguntas propias:

1. **¿Qué es un tile?** Visualización derivada de una consulta a una fuente.
2. **¿Qué permite una base query?** Reutilizar lógica de consulta.
3. **¿Una página cambia permisos?** No por sí sola.
4. **¿Para qué sirve un parámetro?** Permitir filtros y variaciones controladas.
5. **¿Qué implica auto refresh?** Actualizar la visualización según una frecuencia.
6. **¿Refrescar equivale a obtener datos nuevos?** No.
7. **¿Cuál es el papel de KQL database?** Proporcionar los datos consultados por las visualizaciones.
8. **¿Por qué documentar unidades?** Para impedir interpretaciones incorrectas.
9. **¿Qué riesgo tienen cien tiles con consultas pesadas?** Coste y rendimiento.
10. **¿Este laboratorio se etiqueta como DP-700 en el repositorio?** No, se incorpora como complementario oficial DP-603.

**Summary:** un dashboard es la capa de visualización sobre consultas y datos. Su calidad se mide por claridad semántica, permisos, rendimiento, frescura y mantenimiento, no por número de gráficos.

## 7. Fuentes

- [index.yml de siete unidades](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/create-real-time-dashboards-microsoft-fabric/index.yml).
- [Creación y tiles](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/create-real-time-dashboards-microsoft-fabric/includes/2-get-started-with-real-time-dashboards.md).
- [Pages, parameters y refresh](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/create-real-time-dashboards-microsoft-fabric/includes/3-advanced-features.md).
- [Buenas prácticas](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/create-real-time-dashboards-microsoft-fabric/includes/4-real-time-dashboards-best-practices.md).
- [Laboratorio oficial complementario DP-603](https://github.com/MicrosoftLearning/mslearn-fabric/blob/main/Instructions/Labs/13-real-time-dashboards.md).

**Control:** unidad introductoria, creación, avanzado, buenas prácticas, práctica complementaria, evaluación y síntesis documentadas. No se han ejecutado los pasos.