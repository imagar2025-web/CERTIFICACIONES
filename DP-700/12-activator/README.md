# 12 · Use Activator in Microsoft Fabric

> **DP-700 · Implement Real-Time Intelligence.** [Módulo de Microsoft Learn](https://learn.microsoft.com/en-us/training/modules/use-fabric-activator/) · [archivos de origen](https://github.com/MicrosoftDocs/learn/tree/main/learn-pr/wwl/use-fabric-activator). **Práctica complementaria oficial:** [11-data-activator.md](https://github.com/MicrosoftLearning/mslearn-fabric/blob/main/Instructions/Labs/11-data-activator.md), catalogada para DP-603. **Estado:** teoría y procedimiento documentados, no ejecutados.

## Índice

1. [Introducción](#1-introducción)
2. [Configure Activator](#2-configure-activator)
3. [Create rules](#3-create-rules)
4. [Configure actions](#4-configure-actions)
5. [Laboratorio complementario](#5-laboratorio-complementario)
6. [Knowledge check y summary](#6-knowledge-check-y-summary)
7. [Fuentes y control](#7-fuentes-y-control)

## 1. Introducción

Los dashboards permiten que una persona observe indicadores, pero muchos procesos necesitan una respuesta automatizada. Una flota de dispositivos podría comunicar fallos de temperatura o un sistema de alquiler de bicicletas podría necesitar intervención cuando una estación queda sin plazas libres. **Fabric Activator** conecta la observación de datos y la ejecución de acciones cuando se detectan condiciones concretas.

El concepto esencial es pasar de la lógica «un usuario consulta cada cierto tiempo» a «una regla supervisa datos y ejecuta una acción». Eso no significa que la regla sea infalible ni que tenga las mismas garantías que un sistema de control industrial. Deben revisarse latencia, duplicados, filtros, umbrales, destinatarios y comportamiento en caso de fallos.

## 2. Configure Activator

### 2.1. Business objects

Los **business objects** representan entidades del mundo real sobre las que se evalúan condiciones: una estación de bicicleta, un envío, una máquina o un cliente. Cada objeto tiene un identificador estable y propiedades observadas en los eventos, como temperatura, tiempo transcurrido o disponibilidad.

Un error habitual consiste en evaluar un flujo sin definir correctamente qué mensajes corresponden a la misma entidad. Si una estación emite cien mensajes por minuto, no debería confundirse con cien estaciones diferentes. La clave de objeto permite agrupar sucesivas observaciones del mismo elemento; su calidad condiciona la validez de la regla.

### 2.2. Properties y tipos

Las **properties** describen valores observables como \`temperatura\`, \`status\`, \`No_Empty_Docks\` o \`location\`. Debe definirse su tipo y semántica. Un valor de temperatura sin unidad no permite establecer un umbral confiable. Si la propiedad es nullable, la regla debe decidir si los datos ausentes representan una alarma, un estado desconocido o una observación descartable.

### 2.3. Configurar la fuente

Activator puede utilizar entradas de eventos mediante integraciones admitidas, incluida la de Eventstream. Antes de crear reglas, revisar que los datos recibidos contienen claves y propiedades, y que el objeto está mapeado correctamente. Los eventos mal mapeados pueden producir estados inexistentes o condiciones que nunca se cumplen.

**Fuente:** [Configure Activator](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/use-fabric-activator/includes/2-configure-activator.md).

## 3. Create rules

### 3.1. Monitor, Condition y Property filter

Una regla identifica **qué observar**, **bajo qué condición actuar** y **sobre qué subconjunto de entidades**. La interfaz oficial distingue campos como Monitor, Condition y Property filter. El ejemplo de una empresa de entregas que transporta medicamentos puede detectar si la temperatura de un paquete supera el límite durante un período relevante.

No basta con escribir un umbral de 30 grados. Es necesario conocer si el sensor mide grados Celsius, cuándo se recibió el evento, qué paquete produjo el dato y si el valor es una anomalía aislada o un patrón continuado.

### 3.2. Evitar falsos positivos

Una condición inmediata puede dispararse por ruido de sensor. Una regla productiva podría exigir que el umbral se mantenga durante varias observaciones o cierto tiempo, cuando la funcionalidad elegida lo permita. Los mensajes ausentes necesitan tratamiento diferente de las mediciones fuera de rango. En ocasiones la ausencia de eventos es precisamente la situación que debe alertarse.

La lógica también necesita decidir qué ocurre cuando la condición desaparece y vuelve a producirse. Sin una estrategia de repetición o recuperación, un valor que oscila cerca del umbral puede enviar numerosos avisos.

### 3.3. Ámbito y filtros

Un **Property filter** restringe las entidades que interesan, por ejemplo solo estaciones de una calle o dispositivos de una región. La comparación debe ser coherente con la capitalización, normalización y tipos del origen. Un filtro demasiado estricto puede impedir cualquier acción, mientras otro demasiado amplio puede incluir recursos no autorizados.

**Fuente:** [Create rules](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/use-fabric-activator/includes/3-create-rules.md).

## 4. Configure actions

El módulo distingue cuatro familias de acciones que deben seleccionarse según la necesidad: **Email**, **Teams**, **Power Automate** y **Fabric item actions**. Las primeras se orientan a avisar personas; Power Automate integra flujos adicionales; las acciones sobre elementos Fabric pueden disparar procesos admitidos. Los tipos exactos de acción y disponibilidad dependen de la configuración del servicio y tenant.

| Acción | Uso razonable | Riesgo |
|---|---|---|
| Email | Notificación documentable, no necesariamente inmediata | Saturación de buzón o difusión de datos sensibles |
| Teams | Coordinación de equipos operativos | Demasiadas alertas en un canal |
| Power Automate | Integración de un workflow más amplio | Permisos, bucles y efectos secundarios |
| Fabric item | Continuar procesamiento o respuesta de datos | Reejecuciones y consumo de capacidad |

### 4.1. Seguridad de acciones

Una acción que desencadena un Notebook o Pipeline no debe tener más privilegios de los necesarios. Si existe una integración externa, hay que revisar conexiones, credenciales, identidad de ejecución y datos incluidos en las notificaciones. No conviene enviar en el cuerpo de un correo todos los campos de un cliente porque la regla necesita solamente un identificador de incidencia.

### 4.2. Acciones idempotentes

Si una regla se activa dos veces, una acción podría ejecutarse dos veces. En un proceso que genera tickets, facturas o modifica tablas, esto puede producir duplicados. Diseñar identificadores de incidente y mecanismos de deduplicación es una responsabilidad adicional a la configuración visual de Activator. Para acciones destructivas o financieramente relevantes se requieren controles explícitos y aprobación cuando corresponda.

**Fuente:** [Configure actions](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/use-fabric-activator/includes/4-configure-actions.md).

~~~mermaid
flowchart LR
  S["Evento de estación"] --> O["Object: station_id"]
  O --> R{"No_Empty_Docks == 0"}
  R -->|Sí| A["Notificación a operaciones"]
  R -->|No| F["Estado normal"]
  A --> L["Registrar incidencia"]
~~~

*Diagrama conceptual propio.* El registro de incidencia es una ampliación de arquitectura, no necesariamente una acción automática del laboratorio.

## 5. Laboratorio complementario

**Fuente:** [Use Activator in Fabric](https://github.com/MicrosoftLearning/mslearn-fabric/blob/main/Instructions/Labs/11-data-activator.md) · [laboratorio navegable](https://microsoftlearning.github.io/mslearn-fabric/Instructions/Labs/11-data-activator.html). **Duración orientativa:** 30 minutos. **Identificación:** ejercicio oficial catalogado para DP-603, añadido como complemento del módulo DP-700. **No ejecutado.**

### 5.1. Escenario y objetivo

En un sistema de alquiler de bicicletas, una estación situada en **Allington Street** puede quedarse sin puntos libres para devolver una bicicleta. El ejercicio construye una regla que envía un correo de aviso cuando una estación relevante tiene \`No_Empty_Docks = 0\`. El objetivo técnico es crear un objeto, definir sus propiedades, configurar el filtro de calle y establecer la acción.

### 5.2. Crear Workspace y Activator

Abrir [Fabric](https://app.fabric.microsoft.com/), crear Workspace de pruebas en capacidad compatible y crear el elemento Activator. La muestra oficial contiene ejemplos de reglas; inspeccionarlas ayuda a entender el modelo antes de modificar una configuración. No reutilizar destinatarios reales de otro usuario ni enviar pruebas fuera de una cuenta propia autorizada.

### 5.3. Objeto y properties

Crear el objeto que identifique estaciones de bicicleta, utilizando el identificador que esté disponible en la fuente. Revisar la propiedad \`No_Empty_Docks\` y la propiedad de calle. Si el tipo numérico se ha interpretado como texto, la comparación con 0 podría no funcionar según la evaluación del servicio. Verificar el valor mostrado por la fuente.

### 5.4. Regla y acción

Crear regla para la propiedad de plazas libres, utilizar la condición igual a cero y restringir el ámbito a Allington Street. Configurar como destinatario un correo de prueba que pertenezca al usuario autorizado. Revisar si el ejemplo contiene datos suficientes para cumplir la condición y observar el estado de evaluación. No se afirma que el correo se haya enviado: el laboratorio no se ha ejecutado.

### 5.5. Diagnóstico

| Problema | Causa posible | Corrección |
|---|---|---|
| La condición nunca se cumple | Valor no llega a cero | Revisar sample y filtro |
| Se incluyen todas las estaciones | Property filter incorrecto | Restringir la calle |
| No aparecen objetos | Identificador ausente o mal mapeado | Revisar definición de object |
| No se recibe email | Acción no habilitada o destinatario incorrecto | Revisar estado y configuración |
| Se reciben muchos avisos | Regla demasiado amplia o repetitiva | Evaluar alcance y política de disparo |
| El Workspace no permite crear Activator | Capacidad o permisos | Revisar configuración de tenant |

### 5.6. Limpieza

Eliminar el Workspace de prueba o el elemento Activator y las fuentes creadas para el ejercicio. Deshabilitar notificaciones para no dejar alertas activas. Revisar costes de capacidad. No borrar Workspaces compartidos.

## 6. Knowledge check y summary

Preguntas originales:

1. **¿Qué representa un business object?** Entidad identificable sobre la que se supervisan propiedades.
2. **¿Qué es una property?** Valor observado, como temperatura o plazas libres.
3. **¿Cuál es la función de una rule?** Detectar condiciones definidas y activar acciones configuradas.
4. **¿Para qué sirve Property filter?** Reducir las entidades sujetas a evaluación.
5. **¿Qué ventaja tiene Teams?** Facilitar coordinación operativa.
6. **¿Cuándo es útil Power Automate?** Para conectar la alerta con un proceso de integración.
7. **¿Por qué importa la idempotencia?** Una condición repetida puede generar efectos duplicados.
8. **¿El estado desconocido equivale siempre a cero?** No.
9. **¿Qué riesgo presenta un destinatario incorrecto?** Exposición de información y alertas a terceros.
10. **¿La práctica 11-data-activator está catalogada en DP-700?** No, es complementaria procedente de DP-603.

**Summary:** Activator transforma condiciones sobre datos en respuestas automatizadas. Para usarlo correctamente se necesitan objetos identificables, propiedades con semántica conocida, reglas específicas, acciones seguras y control de alertas repetidas.

## 7. Fuentes y control

- [index.yml de siete unidades](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/use-fabric-activator/index.yml).
- [Configure Activator](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/use-fabric-activator/includes/2-configure-activator.md).
- [Create rules](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/use-fabric-activator/includes/3-create-rules.md).
- [Configure actions](https://github.com/MicrosoftDocs/learn/blob/main/learn-pr/wwl/use-fabric-activator/includes/4-configure-actions.md).
- [Lab oficial complementario DP-603](https://github.com/MicrosoftLearning/mslearn-fabric/blob/main/Instructions/Labs/11-data-activator.md).

**Control:** conceptos introductorios, configuración, reglas, acciones, ejercicio complementario, preguntas y síntesis documentados. Sin ejecución ni resultados medidos.