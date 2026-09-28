
# 4.3. Módulo 3 / Integrante 3 (PAUCAR)

## Tabla Resumen de Decisiones de Diseño

| ID | Título de la Decisión | Categoría de Diseño | Estado |
|---|---|---|---|
| ADR-001 | Recepción de pedidos pagados mediante arquitectura orientada a eventos (EDA) vs. llamadas HTTP REST síncronas | Modelo de Coordinación | Aceptado |
| ADR-002 | Estrategia de consultas analíticas mediante vistas SQL de solo lectura vs. consultas ORM sobre tablas transaccionales | Asignación de Responsabilidades | Aceptado |
| ADR-003 | Selección del motor de persistencia relacional PostgreSQL frente a soluciones NoSQL para la trazabilidad de despachos | Elección de Tecnología | Aceptado |
| ADR-004 | Estructuración del modelo de datos con tablas operativas transaccionales y vistas SQL dedicadas para reportes | Modelo de Datos | Aceptado |
| ADR-005 | Gestión de consultas masivas de reportes mediante paginación a nivel de base de datos vs. carga completa en memoria | Gestión de Recursos | Aceptado |
| ADR-006 | Despliegue del Módulo 3 como un servicio API independiente con acceso a la base de datos compartida del ERP | Mapeo entre Elementos Arquitectónicos | Aceptado |
| ADR-007 | Resolución de parámetros de conexión y umbrales de alerta de stock crítico mediante variables de entorno en tiempo de ejecución | Tiempo de Enlace | Aceptado |
| ADR-008 | Validaciones de secuencia de estados de entrega ("CU-DES-06") en la capa de aplicación vs. triggers en la base de datos | Asignación de Responsabilidades | Aceptado |
| ADR-009 | Mecanismo de reintentos con ID de idempotencia para la recepción de eventos "PedidoPagado" ante caídas de red | Modelo de Coordinación | Aceptado |
| ADR-010 | Selección de Node.js con Express para la construcción del backend del Módulo de Despachos y Reportes | Elección de Tecnología | Aceptado |  

---

## ADR-001 — Recepción de pedidos pagados mediante arquitectura orientada a eventos (EDA) vs. llamadas HTTP REST síncronas

**Categoría:** Modelo de Coordinación

**Contexto:** El Módulo 3 debe recibir notificaciones automáticas del Módulo de Ventas tras confirmarse la compra (`CU-DES-01`). La concurrencia en hora pico exige no bloquear la confirmación del pago en WhatsApp si el servicio de almacén sufre latencia.

**Alternativas:**
1. **Eventos de Dominio (`PedidoPagado`).** Publicación asíncrona desacoplada temporalmente; tolera caídas del receptor sin perder mensajes.
2. **Petición HTTP REST síncrona.** Llamada directa al API de despachos; acoplamiento estricto que falla la venta si almacén no responde.

**Criterios de Elección:** disponibilidad del proceso de venta, desacoplamiento temporal, tolerancia a ráfagas de pedidos.

**Decisión:** Se adopta la recepción asíncrona mediante el evento de dominio `PedidoPagado`.

**Sustento:** La arquitectura orientada a eventos desvincula la venta de la disponibilidad inmediata de almacén, cumpliendo con la tolerancia a fallos exigida. El trade-off es una latencia de pocos segundos en la bandeja de empaque, totalmente asumible.

---

## ADR-002 — Estrategia de consultas analíticas mediante vistas SQL de solo lectura vs. consultas ORM sobre tablas transaccionales

**Categoría:** Asignación de Responsabilidades

**Contexto:** Los reportes administrativos (`RF-REP-01` a `RF-REP-05`) consultan volúmenes consolidados de ventas e inventario. Ejecutar estas agregaciones sobre las tablas de despachos activas bloquea las operaciones de empaque de los almaceneros.

**Alternativas:**
1. **Vistas SQL de solo lectura (`v_reporte_*`).** Consultas optimizadas en el motor PostgreSQL que aíslan la analítica de la escritura transaccional (`RF-REP-06`).
2. **Agregaciones en Backend vía ORM.** Lógica ensamblada en código Node.js con múltiples `JOIN` dinámicos que aumentan la carga de CPU y bloquean filas transaccionales.

**Criterios de Elección:** rendimiento transaccional en almacén, aislamiento de consultas analíticas, eficiencia de cómputo en base de datos.

**Decisión:** Se adopta el uso de vistas SQL de solo lectura a nivel de base de datos para la capa de reportes.

**Sustento:** Asignar la responsabilidad analítica al motor de base de datos mediante vistas precalculadas previene bloqueos en las tablas operativas. El trade-off es la menor flexibilidad en el código backend al requerir scripts SQL para cada nuevo indicador.

--- 

## ADR-003 — Selección del motor de persistencia relacional PostgreSQL frente a soluciones NoSQL para la trazabilidad de despachos

**Categoría:** Elección de Tecnología

**Contexto:** El módulo gestiona el flujo de envíos y reportes financieros del ERP. Requiere trazabilidad estricta y consistencia relacional con las entidades de pedidos, clientes y productos, asegurando transacciones atómicas en los cambios de estado.

**Alternativas:**
1. **PostgreSQL.** Motor relacional con soporte ACID estricto, claves foráneas, vistas complejas e índices optimizados para analítica.
2. **MongoDB.** Base de datos documental de alta velocidad de escritura, pero carente de integridad referencial nativa entre colecciones.

**Criterios de Elección:** consistencia ACID, integridad referencial de despachos, soporte nativo para reportes relacionales.

**Decisión:** Se adopta PostgreSQL como motor relacional estandarizado para la persistencia del módulo.

**Sustento:** La gestión logística exige integridad referencial para evitar despachos sin pedido asociado. PostgreSQL proporciona garantías ACID y capacidades avanzadas de lectura analítica sin introducir motores heterogéneos al stack.

---

## ADR-004 — Estructuración del modelo de datos con tablas operativas transaccionales y vistas SQL dedicadas para reportes

**Categoría:** Modelo de Datos

**Contexto:** El módulo enfrenta dos patrones opuestos: escrituras transaccionales rápidas por parte de almaceneros y lecturas pesadas de datos históricos por administradores. El modelo debe prevenir la degradación por contención de recursos.

**Alternativas:**
1. **Modelo Híbrido (Tablas 3NF + Vistas Denormalizadas).** Escrituras en estructuras normalizadas y lecturas consolidadas en vistas SQL denormalizadas.
2. **Modelo Relacional Puro Normalizado.** Todas las consultas de reportes ejecutan sentencias con múltiples `JOIN` sobre las tablas operativas en tiempo real.

**Criterios de Elección:** velocidad de escritura operativa, simplicidad de lectura analítica, prevención de contención de datos.

**Decisión:** Se adopta un modelo híbrido de tablas operativas normalizadas y vistas denormalizadas de lectura.

**Sustento:** Aísla las lecturas masivas de la analítica respecto a las escrituras diarias de empaque, optimizando ambos patrones. El trade-off es mantener scripts de definición de vistas en la base de datos.

--- 

## ADR-005 — Gestión de consultas masivas de reportes mediante paginación a nivel de base de datos vs. carga completa en memoria

**Categoría:** Gestión de Recursos

**Contexto:** La consulta de historiales de despacho (`CU-DES-05`) y reportes puede retornar decenas de miles de filas[cite: 1]. Cargar estos volúmenes completos en memoria del servidor Node.js causa picos de RAM y riesgo de caídas por *Out of Memory*[cite: 1].

**Alternativas:**
1. **Paginación SQL (`LIMIT` / `OFFSET`).** Recuperación de lotes acotados de registros a nivel de base de datos por petición[cite: 1].
2. **Carga en memoria del Backend.** Recuperación masiva del dataset completo para filtrado y paginación en el servidor de aplicación[cite: 1].

**Criterios de Elección:** consumo de memoria RAM del backend, tiempo de respuesta de interfaz, escalabilidad del dataset[cite: 1].

**Decisión:** Se adopta la paginación obligatoria por base de datos en todas las listas e indicadores[cite: 1].

**Sustento:** Mantiene un perfil de consumo de memoria constante en el backend independientemente del crecimiento histórico de despachos[cite: 1]. El trade-off es la necesidad de peticiones HTTP adicionales al navegar entre páginas[cite: 1].

---

## ADR-006 — Despliegue del Módulo 3 como un servicio API independiente con acceso a la base de datos compartida del ERP

**Categoría:** Mapeo entre Elementos Arquitectónicos

**Contexto:** El ERP se compone de módulos independientes[cite: 1]. Se debe definir la estrategia de despliegue para el servicio de logística y reportes evitando que fallas de memoria afecten a los módulos de ventas o clientes[cite: 1].

**Alternativas:**
1. **Servicio API Independiente + BD Compartida.** Proceso Node.js dedicado que accede al esquema de despachos y reportes de la base de datos PostgreSQL del ERP[cite: 1].
2. **Monolito en Proceso Único.** Compilación y ejecución de todos los módulos del ERP dentro de un mismo proceso de servidor[cite: 1].

**Criterios de Elección:** aislamiento de procesos, mantenibilidad en despliegues, simplicidad de infraestructura[cite: 1].

**Decisión:** Se adopta el despliegue del módulo como un servicio API independiente sobre la base de datos compartida[cite: 1].

**Sustento:** Garantiza que un consumo elevado durante la generación de reportes no reinicie el proceso principal de ventas[cite: 1]. Compartir PostgreSQL evita la sobrecarga operativa de gestionar bases de datos distribuidas[cite: 1].

---

## ADR-007 — Resolución de parámetros de conexión y umbrales de alerta de stock crítico mediante variables de entorno en tiempo de ejecución

**Categoría:** Tiempo de Enlace

**Contexto:** El módulo requiere parámetros de entorno como credenciales de base de datos, puertos y el umbral de alerta para stock crítico (`RF-REP-03`)[cite: 1]. Alterar estos valores no debe exigir recomponer el código[cite: 1].

**Alternativas:**
1. **Variables de Entorno en Runtime.** Carga dinámica de parámetros (`DB_URL`, `STOCK_UMBRAL`) desde el entorno al arrancar el proceso[cite: 1].
2. **Constantes en Código Fuente.** Definición estática de parámetros compilados en archivos de configuración del proyecto[cite: 1].

**Criterios de Elección:** seguridad de credenciales, portabilidad entre entornos, flexibilidad de ajuste operativo[cite: 1].

**Decisión:** Se adopta la configuración por variables de entorno en tiempo de ejecución (*Runtime*)[cite: 1].

**Sustento:** Permite modificar credenciales o ajustar umbrales de stock crítico sin reconstruir el código de la aplicación[cite: 1]. El trade-off es la necesidad de validar la existencia de las variables durante el arranque del servicio[cite: 1].

---

## ADR-008 — Validaciones de secuencia de estados de entrega (`CU-DES-06`) en la capa de aplicación vs. triggers en la base de datos

**Categoría:** Asignación de Responsabilidades

**Contexto:** El proceso exige que ningún pedido pase a `ENTREGADO` sin haber estado antes en `DESPACHADO` (`RF-DES-06`). La validación debe garantizar consistencia sin degradar los mensajes de error a la interfaz del usuario.

**Alternativas:**
1. **Validación en Capa de Servicio (Backend).** Evaluación previa del estado actual en el código Node.js con manejo de excepciones informativas.
2. **Triggers en PostgreSQL.** Regla ejecutada mediante procedimientos almacenados en la base de datos que abortan la transacción SQL.

**Criterios de Elección:** usabilidad de mensajes de error, mantenibilidad del código de negocio, facilidad de pruebas unitarias.

**Decisión:** Se asigna la responsabilidad de validación de secuencia a la capa de servicio del backend Node.js.

**Sustento:** Concentrar las reglas de negocio en la aplicación permite retornar mensajes explicativos al almacenero ante estados inválidos, mejorando la usabilidad. Se asume el trade-off de duplicar comprobaciones si otro servicio accediera directamente a la base de datos.

--- 

## ADR-009 — Mecanismo de reintentos con ID de idempotencia para la recepción de eventos `PedidoPagado` ante caídas de red

**Categoría:** Modelo de Coordinación

**Contexto:** Ante fallos breves de red, los reintentos automáticos en la recepción del evento `PedidoPagado` pueden procesar dos veces el mismo mensaje, duplicando las órdenes de empaque en la bandeja del almacén.

**Alternativas:**
1. **Procesamiento Idempotente con `evento_id`.** Registro del ID único de evento en tabla de auditoría para descartar duplicados reintentados.
2. **Procesamiento Directo sin Validación.** Inserción inmediata de registros en la tabla de despachos sin verificación de antecedentes.

**Criterios de Elección:** prevención de registros duplicados, confiabilidad de recepción, consistencia del inventario.

**Decisión:** Se adopta un esquema de coordinación idempotente basado en la verificación previa del `evento_id`.

**Sustento:** Previene la duplicación de órdenes físicas en almacén ante reintentos de red. La consulta ligera de verificación es un costo mínimo frente al impacto operativo de duplicar un despacho.

---

## ADR-0010 — Selección de Node.js con Express para la construcción del backend del Módulo de Despachos y Reportes

**Categoría:** Elección de Tecnología

**Contexto:** El servidor backend gestiona llamadas I/O intensivas (recepción de eventos, consultas SQL paginadas, APIs para la interfaz web). El equipo cuenta con dominio de JavaScript/TypeScript.

**Alternativas:**
1. **Node.js con Express.** Modelo de E/S no bloqueante orientado a eventos, liviano y con alta velocidad de desarrollo en JavaScript.
2. **Java con Spring Boot.** Framework empresarial de tipado estático con mayor consumo de memoria y arranque más lento.

**Criterios de Elección:** eficiencia en I/O asíncrono, velocidad de desarrollo, aprovechamiento de capacidades del equipo.

**Decisión:** Se adopta Node.js con Express como el stack tecnológico backend del módulo.

**Sustento:** Su arquitectura orientada a eventos procesa eficientemente las peticiones I/O del módulo con bajo consumo de recursos. Homogeneizar el lenguaje reduce la curva de aprendizaje y acelera las entregas del proyecto.

---