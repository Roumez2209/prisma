# Prisma – Guía por partes escalables

Oct 26, 2026 · @Cepeda

La misma guía del proyecto, ordenada en partes que se construyen una sobre otra y que la óptica puede usar a medida que se entregan.

## 1. Cómo leer este documento, visión y supuestos

Prisma se construye en 10 partes (de la 0 a la 9). Cada parte agrega una capacidad a la anterior y deja la óptica con algo que ya puede usar. Cada parte trae su objetivo, flujo, historias de usuario, requisitos funcionales, reglas de negocio, datos, criterios de salida y plan. Los códigos de historias (HU), requisitos (RF), reglas (RN) y requisitos no funcionales (RNF) son los mismos de la guía anterior, para poder rastrearlos.

### Principios de escalabilidad

- Cada parte entrega algo que la óptica puede usar el mismo día que se termina.
- La arquitectura final (servidor, base de datos, interfaz, roles) se arma en la parte 0 y no se rehace después.
- El código se separa por módulo: una parte nueva se suma sin tocar las anteriores.
- Los cambios de la base de datos se versionan con migraciones, para crecer sin perder datos.
- Permisos, auditoría y alertas son servicios comunes que cada parte nueva reutiliza.
- Cada parte termina con pruebas automáticas de sus reglas y una prueba de aceptación con la óptica.

### Visión y contexto

Prisma es un sistema para Windows 11 que gestiona la operación completa de una óptica: pacientes, historias clínicas, fórmulas, ventas y facturación electrónica, inventario, pedidos a laboratorios, caja, seguimiento de clientes y finanzas.

**Problema.** Los datos viven dispersos en papel y hojas de cálculo, lo que produce clientes sin seguimiento, errores al facturar, pedidos a laboratorio olvidados y poco control del dinero y del personal.

**Propuesta.** Cada función resuelve y automatiza un proceso concreto de un funcionario: el administrador controla el negocio, el médico atiende y emite fórmulas, y el vendedor vende, cobra y sigue los pedidos.

| Objetivo | Métrica | Meta inicial |
| --- | --- | --- |
| Perder menos clientes | Clientes que regresan a control o reposición en 6 meses | +15% frente a antes de Prisma |
| Vender más rápido | Tiempo para registrar y cobrar una venta | Menos de 2 minutos |
| Atender sin papeles | Tiempo para registrar una consulta y emitir la fórmula | Menos de 5 minutos |
| No perder pedidos | Pedidos entregados tarde sin que nadie lo notara | Cero |
| Inventario confiable | Diferencia entre el sistema y el conteo físico | Menos de 2% |
| Facturar sin errores | Facturas electrónicas aceptadas a la primera | Más de 98% |

### Supuestos y restricciones

- Una sola óptica y un solo local; el diseño deja el campo de sucursal para crecer.
- Dos computadores con Windows 11, en la misma red local, que comparten los mismos datos.
- Debe poder vender, atender y consultar aunque se caiga el internet.
- La facturación electrónica se hace con Siigo por su API. La contabilidad completa queda en Siigo.
- Los lentes se piden a laboratorios externos; se manejan lentes de contacto y terapia visual.
- No hay ventas a crédito ni recordatorios automáticos a clientes por ahora; las alertas son internas.
- Proyecto propio de dos personas, con 15 horas por semana cada una. Ninguno ha hecho interfaces gráficas, así que el plan incluye aprendizaje.
- La tecnología de la interfaz (JavaFX o React) está por decidir; el resto del plan no depende de esa decisión.

## 2. Roles y permisos

Hay tres roles. Cada uno ve y hace solo lo que necesita; el administrador puede ver todo. Los permisos se validan en el servidor, no solo ocultando botones. La columna "Parte" indica cuándo aparece el módulo.

| Módulo | Parte | Administrador | Médico | Vendedor |
| --- | --- | --- | --- | --- |
| Usuarios, configuración y auditoría | 0 | Todo | Sin acceso | Sin acceso |
| Inventario | 1 | Todo, incluidos ajustes | Solo consulta | Consulta y descuento por venta |
| Proveedores | 1 | Todo | Sin acceso | Solo consulta |
| Ventas y recibos | 2 | Todo | Sin acceso | Crear y consultar las propias |
| Caja y cuadre | 2 | Todo, incluido el cuadre general | Sin acceso | Abrir, movimientos y cierre de su caja |
| Pacientes | 3 | Todo | Ver, crear, editar | Ver, crear, editar datos de contacto |
| Historias clínicas | 3 | Solo lectura | Ver, crear, editar | Sin acceso |
| Fórmulas | 3 | Solo lectura | Crear, editar, imprimir | Solo lectura e imprimir |
| Pedidos a laboratorio | 4 | Todo | Consulta los de sus pacientes | Crea y hace seguimiento |
| Alertas | 4 | Todas | Las de su rol | Las de su rol |
| Facturas electrónicas y notas crédito | 5 | Todo, incluida la anulación | Sin acceso | Crear y consultar las propias |
| Cotizaciones | 6 | Todo | Sin acceso | Crear y convertir en venta |
| Garantías y devoluciones | 6 | Aprueba y resuelve | Sin acceso | Registra |
| Página principal | 7 | Resumen general | Citas del día y sus pacientes | Ventas, pedidos y alertas propias |
| CRM y mis actividades | 7 | Todo | Mis actividades | Sus clientes y mis actividades |
| Agenda y horarios | 7 | Define horarios, ve todas las citas | Ve sus citas | Agenda citas y ve las suyas |
| Finanzas | 8 | Todo | Sin acceso | Sin acceso |
| Reportes avanzados y empleados | 9 | Todo | Sin acceso | Sin acceso |

## 3. Arquitectura base y requisitos no funcionales

### 3.1 Arquitectura base (se arma en la parte 0)

- **Servidor**, en uno de los dos computadores: lógica de negocio, base de datos y tareas programadas (alertas, cola de envío a Siigo, copias de seguridad). Debe estar encendido en horario de atención.
- **Clientes**, en los dos computadores: la interfaz con la que trabaja cada usuario. La tecnología de la interfaz (JavaFX o React) está por decidir.
- **Siigo**: se conecta solo desde el servidor, por su API.
- **Propuesta del servidor:** Java con Spring Boot y MySQL, que es lo que el equipo ya conoce o está aprendiendo. Se confirma en la sección de decisiones pendientes.

### 3.2 Requisitos no funcionales

Las cifras son metas iniciales. Las normas citadas deben validarse con un abogado antes de implementar. La columna "Desde" indica la parte en la que el requisito empieza a aplicar.

| ID | Categoría | Requisito | Desde |
| --- | --- | --- | --- |
| RNF-01 | Seguridad | Las contraseñas se guardan con un hash seguro y salado, nunca en texto plano. | 0 |
| RNF-02 | Seguridad | Los datos sensibles (historias clínicas) se cifran en la base de datos, y las comunicaciones con servicios externos usan TLS. | 3 |
| RNF-03 | Seguridad | Los permisos por rol se validan en el servidor, no solo ocultando botones. | 0 |
| RNF-04 | Seguridad | La sesión se bloquea tras 30 minutos de inactividad y pide la contraseña. | 0 |
| RNF-05 | Seguridad | Las credenciales de la API de Siigo se guardan cifradas y nunca en el código ni en los registros. | 5 |
| RNF-06 | Cumplimiento | Protección de datos personales y de salud según la Ley 1581 de 2012: consentimiento del paciente y registro de accesos. | 3 |
| RNF-07 | Cumplimiento | Las historias clínicas se conservan según la Resolución 1995 de 1999 y no se pueden borrar. | 3 |
| RNF-08 | Cumplimiento | La facturación electrónica se hace por la API de Siigo, que cumple la normativa de la DIAN; Prisma no genera el XML ni la firma. | 5 |
| RNF-09 | Rendimiento | Las pantallas principales cargan en menos de 3 segundos y las búsquedas responden en menos de 2 segundos con 50.000 pacientes. | 1 |
| RNF-10 | Rendimiento | Confirmar una venta toma menos de 2 segundos, sin contar la respuesta de Siigo. | 2 |
| RNF-11 | Disponibilidad | El sistema funciona en la red local sin internet para vender, atender y consultar; las facturas se envían a Siigo cuando vuelva la conexión. | 5 |
| RNF-12 | Disponibilidad | Los dos computadores usan una sola base de datos; si el servidor está apagado o sin red, el cliente avisa con un mensaje claro y no pierde lo guardado. | 0 |
| RNF-13 | Concurrencia | Dos usuarios pueden trabajar a la vez sin pisarse: si dos editan el mismo registro, el sistema avisa y no sobrescribe en silencio. | 1 |
| RNF-14 | Respaldo | Copia de seguridad automática diaria, conservada al menos 30 días, con una copia fuera del servidor y una restauración probada cada mes. | 0 |
| RNF-15 | Usabilidad | Interfaz en español; un vendedor nuevo debe poder facturar tras 30 minutos de capacitación; las acciones frecuentes tienen atajos de teclado. | 2 |
| RNF-16 | Usabilidad | Textos legibles, mensajes de error que explican qué hacer y confirmación antes de acciones irreversibles. | 0 |
| RNF-17 | Compatibilidad | Funciona en Windows 11 con 4 GB de RAM como mínimo e imprime en impresoras comunes (láser y, si se confirma, de tirilla). | 0 |
| RNF-18 | Instalación | Un instalador deja el sistema listo y permite actualizar sin perder datos; cada versión actualiza la base de datos con migraciones. | 0 |
| RNF-19 | Mantenibilidad | Código modular por dominio, con pruebas automáticas de las reglas de negocio y documentación técnica actualizada. | 0 |
| RNF-20 | Escalabilidad | La base de datos incluye el campo de sucursal desde el inicio. | 0 |
| RNF-21 | Auditoría | Los registros de auditoría no se pueden modificar y se conservan al menos 5 años. | 0 |
| RNF-22 | Observabilidad | Se guarda un registro técnico de errores, sin datos personales, para diagnosticar fallas. | 0 |
| RNF-23 | Portabilidad de datos | Todos los datos se pueden exportar a Excel o CSV. | 1 |

## 4. Mapa de partes y cronograma

Son 24 sprints de 2 semanas, del 2 de noviembre de 2026 al 1 de octubre de 2027, con 60 horas por sprint entre los dos: unas 1.440 horas. El plan incluye el aprendizaje en los primeros sprints. Si la velocidad real queda por debajo, las fechas se corren.

| Parte | Nombre | La óptica puede, al terminar | Depende de | Sprints | Fechas | Horas |
| --- | --- | --- | --- | --- | --- | --- |
| 0 | Fundamentos | Entrar con su rol, configurar la óptica y tener auditoría y copias de seguridad | Ninguna | 0 y 1 | 2 al 27 de noviembre de 2026 | 120 |
| 1 | Inventario y proveedores | Saber qué productos y cuánto stock tiene, y a quién se los compra | 0 | 2 y 3 | 30 de noviembre al 25 de diciembre | 120 |
| 2 | Ventas y caja | Vender con recibo interno, cobrar, abonar y cuadrar la caja | 1 | 4 y 5 | 28 de diciembre al 22 de enero de 2027 | 120 |
| 3 | Pacientes, historia clínica y fórmulas | Atender pacientes y emitir e imprimir fórmulas | 0 y 2 | 6 a 8 | 25 de enero al 5 de marzo | 180 |
| 4 | Pedidos a laboratorio y alertas | Pedir lentes a laboratorios, seguirlos y recibir avisos | 2 y 3 | 9 y 10 | 8 de marzo al 2 de abril | 120 |
| 5 | Facturación electrónica y cierre del MVP | Facturar por Siigo, anular con nota crédito y operar el día a día | 2 | 11 a 13 | 5 de abril al 14 de mayo | 180 |
| 6 | Cotizaciones, garantías y devoluciones | Cotizar y atender la postventa | 2 y 5 | 14 y 15 | 17 de mayo al 11 de junio | 120 |
| 7 | Inicio, CRM y agenda | Ver el panel principal, seguir clientes y agendar citas | 3 y 4 | 16 a 19 | 14 de junio al 6 de agosto | 240 |
| 8 | Finanzas | Controlar gastos, presupuesto, metas, facturas por pagar y por cobrar | 2 y 4 | 20 y 21 | 9 de agosto al 3 de septiembre | 120 |
| 9 | Empleados, reportes y puesta en marcha | Medir al personal, ver reportes avanzados y quedar capacitada | Todas | 22 y 23 | 6 de septiembre al 1 de octubre | 120 |

**Hitos:** MVP listo el 14 de mayo de 2027 (partes 0 a 5); segunda entrega el 6 de agosto (partes 6 y 7); tercera entrega el 1 de octubre (partes 8 y 9).

Hasta la parte 5, las ventas son con recibo interno. Mientras la parte 5 no esté lista, no deben usarse como venta real si la óptica está obligada a facturar electrónicamente; hay que confirmarlo con el contador.

## 5. Parte 0: Fundamentos

Sprints 0 y 1, del 2 al 27 de noviembre de 2026, 120 horas. No depende de ninguna otra parte.

### Objetivo

Dejar armado el esqueleto sobre el que crece todo lo demás: servidor, base de datos compartida entre los dos computadores, interfaz, acceso por roles, auditoría y copias de seguridad. Al terminar, los usuarios entran con su rol desde los dos computadores y se hace una copia diaria de los datos.

### Qué se hace

- **Sprint 0:** herramientas y repositorio listos, aprendizaje de la tecnología elegida para la interfaz, modelo de datos diseñado, prototipos de las pantallas principales, y credenciales de Siigo pedidas con una prueba de concepto.
- **Sprint 1:** servidor y base de datos compartida por los dos computadores, inicio de sesión, usuarios y roles, configuración de la óptica, auditoría y copia de seguridad diaria.

### Historias de usuario

| ID | Historia | Criterios de aceptación | Prior. |
| --- | --- | --- | --- |
| HU-01 | Como administrador, quiero crear, editar y desactivar usuarios con un rol, para controlar quién entra y qué hace. | Un usuario desactivado no inicia sesión pero conserva su historial. Solo el administrador gestiona usuarios. | M |
| HU-02 | Como empleado, quiero iniciar y cerrar sesión de forma segura, para proteger los datos. | La cuenta se bloquea tras 5 intentos fallidos. La sesión se bloquea tras 30 minutos sin uso. | M |
| HU-50 | Como administrador, quiero configurar los datos de la óptica y los parámetros (días de alerta, descuento máximo, venta sin factura, umbral de diferencia de caja), para adaptar Prisma. | Los cambios quedan en la auditoría. | M |
| HU-51 | Como administrador, quiero consultar quién hizo qué y cuándo en los datos críticos, para tener trazabilidad. | Registra usuario y hora de cada acción crítica. No se puede editar. | S |
| HU-52 | Como administrador, quiero una copia de seguridad automática y poder restaurarla, para no perder los datos. | Una copia diaria, una copia fuera del servidor y una restauración probada. | M |

### Requisitos funcionales

| ID | Requisito |
| --- | --- |
| RF-01 | El sistema debe permitir iniciar y cerrar sesión con usuario y contraseña, y bloquear la cuenta tras 5 intentos fallidos. |
| RF-02 | El sistema debe permitir crear, editar y desactivar usuarios y asignarles uno de los tres roles. |
| RF-03 | El sistema debe aplicar los permisos de cada rol en todas las operaciones. |
| RF-04 | El sistema debe permitir configurar los datos de la óptica y los parámetros: días de anticipación de alertas, descuento máximo del vendedor, habilitar la venta sin factura electrónica y umbral de diferencia de caja. |
| RF-63 | El sistema debe registrar quién creó, modificó o anuló cada dato crítico, con fecha y hora. |
| RF-64 | El sistema debe hacer una copia de seguridad automática diaria y permitir restaurarla. |

### Reglas de negocio

| ID | Regla |
| --- | --- |
| RN-52 | Cada acción crítica queda registrada con usuario, fecha y hora. |
| RN-53 | Un usuario cuyo empleo terminó se desactiva, nunca se elimina, para conservar la trazabilidad. |

### Datos

Óptica (configuración), usuario y rol, auditoría y copia de seguridad.

### Criterios de salida

- Dos usuarios con roles distintos entran a la vez desde los dos computadores y cada uno ve solo lo suyo.
- Una copia de seguridad se restaura con éxito en otro equipo.
- La auditoría registra la creación de un usuario y un cambio de configuración.
- El esqueleto está documentado: cómo levantar el servidor, la base de datos y la interfaz.

## 6. Parte 1: Inventario y proveedores

Sprints 2 y 3, del 30 de noviembre al 25 de diciembre de 2026, 120 horas. Depende de la parte 0.

### Objetivo

Que la óptica sepa qué productos tiene, cuánto stock queda y a quién se los compra. Es el módulo más sencillo y no maneja datos de salud, así que sirve para aprender el recorrido completo (interfaz, servidor y base de datos) con poco riesgo.

### Flujo

1. **Administrador:** registra a los proveedores y laboratorios con su NIT y contacto.
2. **Administrador:** crea los productos con referencia, categoría, costo, precio, stock mínimo y proveedores, o los importa desde Excel.
3. **Administrador:** registra las entradas por compra y los ajustes con motivo.
4. **Vendedor:** consulta qué hay, cuánto stock tiene y de qué proveedor es.

### Historias de usuario

| ID | Historia | Criterios de aceptación | Prior. |
| --- | --- | --- | --- |
| HU-21 | Como vendedor, quiero consultar qué producto hay y cuánto stock tiene, para decirle al cliente qué puede comprar. | Muestra stock, precio y proveedor. Filtra por categoría y estado (normal, bajo, agotado). | M |
| HU-22 | Como administrador, quiero gestionar productos y proveedores, para saber qué tengo y a quién pedirle. | El proveedor tiene NIT único, contacto y productos asociados. El producto tiene stock mínimo. | M |
| HU-23 | Como administrador, quiero ajustar el inventario con un motivo (pérdida, daño, conteo), para mantener cifras correctas. | Todo ajuste queda con usuario, fecha y motivo, y no se borra. | S |
| HU-53 | Como administrador, quiero importar productos desde Excel o CSV, para no digitar todo desde cero. | Valida los datos, muestra los errores y no duplica referencias. | S |

### Requisitos funcionales

| ID | Requisito |
| --- | --- |
| RF-26 | El sistema debe gestionar productos con referencia, categoría, costo, precio, stock mínimo y proveedores. |
| RF-27 | El sistema debe permitir consultar qué producto hay y cuánto stock tiene, filtrando por categoría, proveedor o estado. |
| RF-29 | El sistema debe registrar entradas por compra y ajustes con motivo. |
| RF-30 | El sistema debe mantener un directorio de proveedores y laboratorios con nombre, NIT, contacto, productos que suministra y condiciones de pago. |
| RF-31 | El sistema debe asociar cada producto a uno o más proveedores. |
| RF-65 | El sistema debe importar productos desde Excel o CSV, validando los datos y mostrando los errores (la importación de pacientes llega en la parte 3). |

### Reglas de negocio

| ID | Regla |
| --- | --- |
| RN-32 | El stock solo cambia por compra, venta, devolución o ajuste, y todo movimiento queda con usuario, fecha y motivo. |
| RN-33 | Los ajustes manuales solo los hace el administrador y requieren motivo. |
| RN-35 | El precio de venta no puede ser menor que el costo, salvo autorización del administrador. |
| RN-36 | El NIT del proveedor es único; un proveedor con productos o pedidos asociados se desactiva, no se elimina. |

### Datos

Producto y categoría, proveedor y laboratorio, relación producto-proveedor y movimiento de inventario.

### Criterios de salida

- La óptica carga su inventario desde Excel y lo consulta con filtros.
- Cada ajuste de stock aparece en la auditoría con su motivo.
- No se puede eliminar un proveedor que tiene productos.
- Las búsquedas responden en menos de 2 segundos.

### Plan

- **Sprint 2 (30 de noviembre al 11 de diciembre):** productos, proveedores y consulta de stock.
- **Sprint 3 (14 al 25 de diciembre):** entradas por compra, ajustes e importación desde Excel.
