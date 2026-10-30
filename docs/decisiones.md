# Decisiones técnicas

Registro breve de decisiones de diseño. Cada una explica el contexto, la elección, qué más se evaluó y qué consecuencias tuvo.

## 1. Aislamiento entre estudios en la base de datos

**Contexto.** Varios estudios comparten una misma base. Una consulta que olvide un filtro no puede mostrar datos de otro estudio.

**Decisión.** Tres capas: filtro por estudio en Hibernate, claves foráneas compuestas (`id`, `estudio_id`) y Row Level Security en PostgreSQL. La aplicación se conecta con un rol que no es dueño de las tablas, y fija el estudio en cada transacción. El estudio sale solo del JWT.

**Alternativas.** Solo filtro en la aplicación (un olvido filtra datos). Una base por estudio (más aislamiento, pero más costo operativo para estudios de 1 a 5 personas).

**Consecuencias.** El aislamiento se verifica con tests de integración que se conectan con el mismo rol sin privilegios. Todo acceso a datos debe pasar por servicios transaccionales; sin transacción, la base devuelve cero filas en vez de datos ajenos. Las operaciones que cruzan estudios van por funciones de base acotadas, no desactivando la seguridad.

## 2. De instalación local a escritorio + nube

**Contexto.** El portal judicial se consulta con las credenciales del abogado y puede bloquear la cuenta ante intentos fallidos. Al principio el sistema se pensó como instalación local en el estudio.

**Decisión.** Evolucionó a un modelo híbrido. La nube guarda los datos, reparte el trabajo y manda los mails. Un agente en las PCs del estudio hace el login y las consultas al portal, con la IP del estudio. La web sigue funcionando en navegador y celular. La nube nunca consulta el portal.

**Alternativas.** Un worker en el servidor que se loguea solo: más simple, pero consulta el portal desde una IP de datacenter y depende de que el portal no lo bloquee. Instalación local completa: se evita la nube, pero cada estudio queda con la responsabilidad de operar el servidor y los backups. La versión local sigue en una rama del repositorio privado.

**Consecuencias.** El agente necesita una PC encendida para traer novedades; si ninguna lo está, la pantalla de inicio lo avisa. Se necesitó una app de escritorio con instalador y actualización automática. Cada PC se vincula con un token revocable.

## 3. Solo lectura, lista blanca y respeto al portal

**Contexto.** El sistema usa una cuenta real de un abogado en un portal oficial. Un error podría bloquear la cuenta o tener efecto procesal.

**Decisión.**
- El cliente del portal solo tiene métodos explícitos para endpoints aprobados, sin una función genérica "llamar URL".
- Un test falla si aparece un método público nuevo.
- Nunca se evade un captcha: si aparece, la cuenta se marca y se avisa.
- Nunca se reintenta un login fallido por credenciales.
- Hay una sola sesión por cuenta, con pausas entre llamadas y backoff ante errores.
- Acciones con efecto en el portal, como abrir una cédula, solo ocurren cuando una persona las pide y quedan auditadas.

**Alternativas.** Reintentar logins automáticamente (riesgo de bloqueo). Resolver captchas con servicios externos (descartado por principio).

**Consecuencias.** Se prefiere perder una sincronización antes que arriesgar la cuenta. Una cuenta con credenciales rechazadas sale de la cola hasta que una persona las corrija.

## 4. Identificar datos cuando el portal cambia los ids

**Contexto.** Se observó que los ids de expedientes, operaciones y cédulas del portal cambian entre días. Una primera versión que deduplicaba por id creó duplicados.

**Decisión.** La identidad real de cada dato es otra. Los expedientes se identifican por número. Las operaciones, por número de operación dentro del expediente. Las cédulas, por cantidad en cada combinación de número de expediente, fecha y tipo. El id del portal se guarda como dato del día.

**Alternativas.** Deduplicar por contenido completo (frágil ante cambios menores). Borrar y reimportar todo (pierde el estado de leído y los datos propios del estudio).

**Consecuencias.** Hay casos ambiguos, como operaciones idénticas el mismo día. Ante la duda se muestran todas, con un aviso, en vez de adivinar. Hay una defensa extra: si llegan muchos movimientos "nuevos" de golpe, se registra un aviso por si el portal renumeró.

## 5. Cola de trabajos en una tabla de PostgreSQL

**Contexto.** Hay que repartir trabajos de sincronización entre varias PCs sin que dos trabajen sobre la misma cuenta a la vez.

**Decisión.** La cola es una tabla. Se toma un trabajo bloqueando la fila de la cuenta, y las PCs mandan una señal de vida mientras trabajan. Una función de base devuelve a la cola los trabajos cuya PC dejó de avisar.

**Alternativas.** Kafka o RabbitMQ: más piezas para operar sin necesidad con este volumen.

**Consecuencias.** Menos infraestructura y transaccionalidad con el resto de los datos. Es un diseño pensado para pocos estudios; con mucha más escala habría que revisarlo. Un caso real, una PC suspendida que dejó trabada una cuenta durante horas, llevó a agregar la señal de vida y un tope de duración por trabajo.

## 6. Manejo de datos sensibles

**Contexto.** El sistema guarda usuarios y contraseñas del portal y datos de causas y personas.

**Decisión.**
- Las credenciales se guardan cifradas con AES-256-GCM, atadas a su fila y campo, con la clave solo en variables de entorno.
- La API nunca las devuelve a una persona; solo se entregan a una PC vinculada junto con un trabajo, y cada entrega queda auditada.
- Los logs no incluyen credenciales ni contenido de causas.
- Los documentos se guardan como estructura JSON del editor, nunca como HTML, para no almacenar ni renderizar HTML arbitrario.
- Los adjuntos se descargan siempre como archivo y se borran a los 7 días.
- Hay roles (administrador, abogado, asistente) y visibilidad por persona aplicada en la base.
- Hay auditoría de solo agregado.

**Alternativas.** Guardar las credenciales solo en cada PC. Fue el diseño anterior; se cambió para que cualquier PC del estudio pueda sincronizar cualquier cuenta sin depender de la que la cargó.

**Consecuencias.** La nube pasa a custodiar secretos cifrados, así que la gestión de la clave es crítica. Hay una rotación de claves prevista por versión.
