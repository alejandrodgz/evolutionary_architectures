# Sección 2 — Identificación de RNF y Funciones de ajuste (Tabla 10)

> **Estado:** 🔄 En construcción  
> **Trazabilidad:** RNF identificados → Sección 1 (arquitectura) → Sección 3 (componentes) → Sección 4 (stack) → Sección 5 (evolución)

---

## Criterios de identificación

Los RNF se extraen **exclusivamente** del enunciado (`descripcion_del_proyecto.md`). Los marcados como **(H)** son hipótesis razonables no enunciadas explícitamente.

---

## Tabla 10 — Requisitos no funcionales

| # | Requisito no funcional | Descripción del RNF | Funciones de ajuste |
|---|------------------------|---------------------|---------------------|
| RNF-01 | **Disponibilidad** | El sistema debe operar 24/7 sin interrupciones planificadas. | • Replicación activa-activa de servicios críticos (D4, D5, D1) <br>• Health checks y circuit breakers (Resilience4j / Istio) <br>• Failover automático en infraestructura (K8s self-healing) <br>• SLA objetivo: 99.9% uptime |
| RNF-02 | **Rendimiento / Tiempo de respuesta** | El tiempo de respuesta del sistema debe ser menor a 2 segundos para operaciones de usuario. | • Caché de corta duración (TTL) para respuestas del proxy de saldo en tiempo real (Redis); D2 no almacena saldos permanentemente, solo cachea las respuestas de los bancos filiales vía D6 <br>• Procesamiento asíncrono de operaciones no críticas (Kafka) <br>• CDN para activos estáticos del frontend <br>• Métrica: P95 < 2 s en producción |
| RNF-03 | **Escalabilidad** | El sistema debe soportar ~25 millones de usuarios activos y picos de 20K–30K transacciones en ventanas específicas (días 14–16 y 29–31 de cada mes). | • Escalado horizontal automático (HPA en Kubernetes) <br>• Queue-based load leveling para pagos masivos (D7) <br>• Particionamiento de tópicos Kafka por volumen esperado <br>• Pruebas de carga previas a ventanas de pago |
| RNF-04 | **Seguridad** | El sistema debe cumplir restricciones estatales: autenticación/autorización, cifrado en todo el ciclo de vida, canales seguros, monitoreo de actividad y conformidad OWASP. | • OAuth2 + JWT con MFA (D1-IAM) <br>• TLS 1.3 en todos los canales (internos y externos) <br>• Cifrado en reposo (AES-256) para datos sensibles <br>• WAF + detección de intrusiones (IDS) <br>• Validación en línea contra listas blancas/grises/negras antes de ejecutar transferencias (D4 — Primario); D8 retroalimenta las listas mediante análisis post-hoc <br>• Auditoría de accesos y operaciones (append-only log) <br>• Conformidad OWASP Top 10 |
| RNF-05 | **Extensibilidad** | El sistema debe permitir integrar terceros (pasarelas, servicios de pago) a demanda sin afectar el estado del sistema en producción. | • Patrón Plugin/Adapter por canal externo (D6) <br>• Registro dinámico de adaptadores (sin redespliegue del núcleo) <br>• Contratos de API versionados (OpenAPI 3.x) <br>• Feature flags para activar/desactivar integraciones en caliente |
| RNF-06 | **Trazabilidad y cumplimiento regulatorio** | El sistema debe mantener histórico completo de transacciones, generar extractos trimestrales a bancos y reportes semestrales a la Superintendencia Financiera. | • Event Sourcing / append-only store (D8) <br>• Correlación de eventos con IDs únicos de transacción <br>• Jobs programados para generación y envío de reportes (scheduler) <br>• Retención configurable de datos por normativa |
| RNF-07 | **Fiabilidad / Consistencia transaccional** | Las transferencias entre bancos filiales deben reflejarse de forma inmediata; las operaciones distribuidas deben completarse o compensarse ante fallos. | • Patrón Saga coreografiada con compensación (D4, D5) <br>• Idempotencia en todos los endpoints de transacción <br>• Exactly-once delivery en Kafka (transacciones Kafka) <br>• Dead letter queues para eventos fallidos |
| RNF-08 | **Multiplataforma / Usabilidad** | La interfaz debe funcionar en web, smartphones y tablets de gama media/alta (mínimo 6 pulgadas) con experiencia de usuario consistente. | • Diseño responsive (CSS Grid / Flexbox) <br>• Progressive Web App (PWA) para móvil híbrido (H) <br>• Pruebas de compatibilidad en dispositivos objetivo <br>• Guía de estilo/design system unificado (H) |
| RNF-09 | **(H) Observabilidad** | El sistema debe permitir monitorear el estado de los servicios, detectar anomalías y depurar incidentes en producción. | • Trazabilidad distribuida (OpenTelemetry / Jaeger) <br>• Métricas por servicio (Prometheus + Grafana) <br>• Logs centralizados (ELK Stack) <br>• Alertas automáticas ante umbrales críticos |
| RNF-10 | **(H) Mantenibilidad / Evolvabilidad** | La arquitectura debe permitir modificar, versionar o reemplazar dominios individuales sin afectar al resto del sistema. | • Bounded contexts con base de datos por servicio (Database per Service) <br>• Contratos de API semánticos + consumer-driven contract testing (Pact) <br>• ADR (Architecture Decision Records) por cada cambio estructural <br>• CI/CD con pipelines por microservicio |
| RNF-D1-01 | **Disponibilidad 24/7 del servicio IAM (D1)** | El servicio IAM debe operar 24/7; su indisponibilidad bloquea el acceso a todo el sistema. | • Uptime mensual ≥ 99.9% (CloudWatch + SLO) <br>• Tiempo de recuperación ante fallo < 60 s (Auto-healing EKS + Multi-AZ) <br>• P95 login < 1.5 s en picos de carga |
| RNF-D1-02 | **Seguridad de autenticación y emisión de tokens (D1)** | El dominio debe implementar autenticación robusta, MFA y emisión segura de JWT firmados. | • 100% tokens firmados con JWT RS256 + rotación de claves KMS <br>• Access token ≤ 15 min (exp corto + refresh controlado) <br>• 100% tráfico cifrado (TLS 1.3) |
| RNF-D1-03 | **Protección contra fuerza bruta y abuso (D1)** | El sistema debe mitigar ataques de fuerza bruta y abuso del endpoint de login. | • Bloqueo a los 5 intentos fallidos (contador Redis por usuario/IP) <br>• < 1% requests legítimas rechazadas (rate limiting en API Gateway) <br>• Alerta ante abuso < 60 s (WAF rules + GuardDuty) |
| RNF-D1-04 | **Autorización granular — RBAC + scopes (D1)** | El acceso a operaciones críticas debe controlarse por rol y alcance (scope). | • 100% endpoints sensibles protegidos (pruebas automatizadas) <br>• 0 rutas expuestas sin rol (pruebas RBAC por rol en CI/CD) |
| RNF-D1-05 | **Trazabilidad y cumplimiento de accesos (D1)** | Todo acceso (éxito/fallo) debe quedar trazado para auditoría, investigación y cumplimiento. | • 100% eventos de autenticación publicados (D1 → MSK → D8) <br>• P95 latencia hacia auditoría < 500 ms <br>• Retención de logs ≥ 5 años (D8/OpenSearch) |
| RNF-D2-01 | **Sincronización diaria idempotente con bancos (D2)** | La sincronización de cuentas con bancos debe ejecutarse diariamente de forma idempotente, con trazabilidad y sin duplicar registros. | • 0 duplicados al ejecutar misma carga 2 veces (idempotencia) <br>• ≥ 99% tasa de éxito mensual del sync <br>• P95 latencia de actualización < 10 min |
| RNF-D2-02 | **Rendimiento de consultas (D2)** | Operaciones de consulta deben responder en menos de 2 segundos. Los saldos se obtienen en tiempo real desde bancos vía D6; se usa un caché de corta duración (TTL) para reducir latencia. | • P95 tiempo de respuesta < 2 s <br>• Cache hit rate ≥ 80% (Redis, TTL corto para respuestas proxy de saldo) <br>• 5xx < 0.1% en picos |
| RNF-D2-03 | **Escalabilidad — 25M usuarios (D2)** | Debe soportar decenas de millones de usuarios y alto volumen concurrente. | • Mantener P95 < 2 s con escalado horizontal (EKS + HPA) <br>• Sin caída en picos de concurrencia <br>• Sin degradación crítica en crecimiento de BD (Aurora Serverless v2) |
| RNF-D2-04 | **Seguridad y cumplimiento (D2)** | Protección de datos sensibles y control de acceso estricto. | • 100% tablas cifradas en reposo <br>• 0 endpoints expuestos sin autenticación (JWT + WAF) <br>• 0 vulnerabilidades críticas OWASP |
| RNF-D2-05 | **Trazabilidad de sincronizaciones (D2)** | Cada cambio de estado y sincronización debe ser trazable. | • 100% cambios publican evento (Outbox Pattern) <br>• P95 latencia auditoría < 500 ms <br>• Inmutabilidad: append-only en D8 |
| RNF-D3-01 | **Interoperabilidad con APIs de empresas aliadas (D3)** | El dominio debe comunicarse con las APIs de las 15 empresas aliadas, cada una con protocolo, autenticación y esquema de datos potencialmente distinto, sin acoplar la lógica de negocio a ningún proveedor. | • Patrón Adapter por empresa implementado en D6; D3 solo conoce la interfaz `EmployeeDataPort` <br>• P95 resolución de datos del empleado (D3 → D6 → API empresa) < 1.5 s en condiciones normales <br>• Circuit Breaker por empresa (Resilience4j): fallo del API de empresa A no afecta a empresa B ni al servicio global |
| RNF-D3-02 | **Carga masiva idempotente de empresas y empleados (D3)** | El proceso de onboarding inicial debe importar datos de empresas aliadas y referencias de empleados de forma masiva, de manera idempotente y con trazabilidad del resultado. | • Completar la carga de 35 000 registros en < 10 min (Spring Batch en chunks de 500 con commit transaccional por chunk) <br>• Idempotencia: ejecutar el mismo archivo dos veces → resultado idéntico, 0 duplicados <br>• SLA global del sistema no se degrada durante el batch (job asíncrono que no bloquea el hilo principal) |
| RNF-D3-03 | **Minimización de datos PII del empleado (D3)** | La plataforma solo debe persistir la referencia mínima del empleado; ningún dato personal completo (nombre, cédula, cuenta bancaria, salario) puede almacenarse en la base de datos de D3. | • Test automatizado en CI que inspecciona el esquema de D3: 0 columnas que contengan nombre, cédula, cuenta bancaria o salario <br>• Datos del empleado resueltos en tiempo real (vía D6) y manejados solo en memoria durante el pago; nunca se persisten <br>• Campos sensibles de empresa (`tax_id`, `auth_config`) cifrados con AES-256 (AWS KMS) |
| RNF-D4-01 | **Consistencia transaccional con múltiples destinos (D4)** | Una instrucción con N destinos debe garantizar que cada sub-transferencia sea atómica e independiente: el fallo de un destino no afecta los demás, y ningún fondo queda en estado intermedio permanente. | • Patrón Saga coreografiada + Outbox Pattern: 100% de compensaciones exitosas (0 fondos perdidos) <br>• P95 saga completa en escenario inmediato (filial–filial) < 3 s <br>• Job de reconciliación periódico: 0 transacciones huérfanas en producción; idempotencia por `idempotency_key` → 0 duplicados |
| RNF-D4-02 | **Dualidad de liquidación inmediata vs diferida (D4)** | El sistema debe distinguir en tiempo real si una transferencia es entre bancos filiales (liquidación inmediata, segundos) o hacia no filiales/internacionales (diferida vía ACH). El usuario debe ser notificado del canal en cada caso. | • P95 liquidación inmediata (filial–filial): desde `TransferApproved` hasta `TransferSettled` < 500 ms <br>• P95 reflejo tras callback de ACH < 2 s <br>• 100% de respuestas HTTP incluyen campo `settlement_type` (IMMEDIATE / DEFERRED); 0 errores de clasificación de canal |
| RNF-D4-03 | **Resiliencia en la integración con ACH (D4)** | La indisponibilidad temporal del sistema ACH no debe interrumpir las transferencias entre bancos filiales ni bloquear indefinidamente las transferencias diferidas. Las transacciones enviadas a ACH deben poder ser reenviadas sin duplicarlas. | • Detección de ACH no disponible < 5 s (Circuit Breaker en D6) <br>• 0 impacto en liquidaciones inmediatas ante caída de ACH <br>• 0 duplicados en reenvío idempotente a ACH <br>• 100% eventos fallidos trazables (Dead Letter Queue) |
| RNF-D4-04 | **Gestión completa del ciclo de estados ACH (D4)** | El sistema debe procesar correctamente todos los estados posibles que ACH puede comunicar (procedente, rechazada, en revisión, timeout) y transicionar la transacción al estado interno correcto, liberando o revirtiendo fondos según corresponda. | • 100% de estados ACH documentados cubiertos (test unitario por caso) <br>• P95 procesamiento de callback < 2 s <br>• 0 transacciones sin resolución (job de reconciliación diario + alerta) <br>• 100% fallos ACH revierten HOLD (simulación `ACH_REJECTED`) |
| RNF-D4-05 | **Monitoreo antifraude en línea (D4)** | Antes de aprobar cualquier transferencia, el sistema debe evaluar en tiempo real si las cuentas involucradas están en listas de control de fraude (blanca/gris/negra). La evaluación es síncrona y bloquea la transacción si el resultado lo indica. | • P99 evaluación antifraude < 200 ms (listas replicadas en caché Redis de D4 con TTL 60 s) <br>• Propagación de actualizaciones de listas desde D8 < 60 s (evento `FraudListUpdated` invalida caché de D4) <br>• 100% de evaluaciones trazadas con `transfer_id`, timestamp y regla aplicada; < 0.5% de falsos positivos |
| RNF-D4-06 | **Disponibilidad y rendimiento del motor de transferencias (D4)** | El motor de transferencias debe operar 24/7 con un tiempo de respuesta menor a 2 segundos para la confirmación al usuario, y debe mantener el SLA incluso durante picos de carga (días de nómina masiva). | • P95 end-to-end < 2 s (traza HTTP hasta confirmación) <br>• ≥ 99.9% disponibilidad mensual (health checks + uptime monitoring) <br>• < 5 s modo degradado bajo carga extrema; 0 pérdida de datos <br>• Recuperación ante reinicio de pod < 30 s (liveness/readiness probes) |
| RNF-D5-01 | **Atomicidad del ledger (D5)** | Toda operación sobre el saldo de la billetera debe registrarse como movimiento de doble entrada (*double-entry bookkeeping*); ningún débito o crédito puede quedar sin su contraparte en la tabla de movimientos. | • Tabla `wallet_entries` append-only con columnas `debit` / `credit` siempre emparejadas en la misma transacción ACID <br>• Constraint de base de datos: `CHECK (debit > 0 XOR credit > 0)` <br>• Prueba de reconciliación automatizada: `SUM(credit) - SUM(debit) = saldo_actual` por cada `wallet_id` |
| RNF-D5-02 | **Compensación transaccional de billetera (D5)** | Si una pasarela de pago (D6) falla tras haberse debitado el saldo de la billetera, el monto debe revertirse automáticamente mediante el mecanismo de compensación de la Saga, sin intervención manual. | • Patrón Saga coreografiado: `WalletDebited` → D6 → `PaymentGatewayFailed` → `WalletCompensationTriggered` <br>• Tiempo máximo de compensación: < 5 s desde la detección del fallo <br>• Dead Letter Queue (DLQ) en Kafka para eventos de compensación fallidos con alerta automática |
| RNF-D5-03 | **Rendimiento de consulta de saldo (D5)** | La consulta de saldo de la billetera debe responder en menos de 2 segundos, incluso con alto volumen de registros en el ledger. | • P95 consulta de saldo < 500 ms (prueba de carga concurrente) <br>• Sin degradación crítica con billeteras de >100K movimientos <br>• Materialización periódica del saldo en caché Redis; CQRS ligero |
| RNF-D5-04 | **Seguridad y control de acceso (D5)** | Solo el titular de la billetera puede operar sobre ella. Todas las operaciones requieren token JWT válido con el scope correspondiente. | • 0 accesos cruzados (test RBAC: usuario A no opera billetera de B) <br>• 100% datos cifrados en reposo (Aurora + KMS) <br>• 100% tráfico cifrado (TLS 1.3) |
| RNF-D5-05 | **Trazabilidad de movimientos de billetera (D5)** | Todo movimiento de la billetera (débito, crédito, compensación) debe quedar registrado en el log de auditoría de D8 para cumplimiento regulatorio. | • 100% de operaciones trazadas (cada operación genera evento) <br>• P95 latencia del evento al log de D8 < 500 ms <br>• 0 modificaciones permitidas (append-only en D8) |
| RNF-D6-01 | **Aislamiento de adapters (D6)** | Un fallo o degradación en un adapter externo (ej. DRUO fuera de servicio) no puede impactar la disponibilidad ni el rendimiento de los demás adapters (PSE, ACH, Apple Pay, terceros). | • Cada adapter se despliega como un pod independiente en EKS (fallo de un pod ≠ fallo del servicio completo) <br>• Circuit breaker por adapter con configuración independiente de umbral de error y tiempo de apertura (Resilience4j) <br>• Bulkhead pattern: thread pool separado por adapter para evitar saturación cruzada |
| RNF-D6-02 | **Integración de nuevos terceros sin downtime (D6)** | Registrar y activar un nuevo tercero o pasarela de pago no debe generar indisponibilidad en los adapters existentes ni requerir redespliegue del núcleo del servicio de integraciones. | • Adapter Registry dinámico: los adapters se registran en caliente via configuración en base de datos sin reinicio de la aplicación <br>• Despliegue del nuevo adapter como contenedor independiente (sin tocar el contenedor del núcleo de D6) <br>• Smoke test automatizado post-deploy que valida que todos los adapters existentes siguen respondiendo (health check por adapter en CI/CD) |
| RNF-D6-03 | **Seguridad en comunicación con sistemas externos (D6)** | Toda comunicación con sistemas externos debe usar canales cifrados (TLS 1.3 mínimo), con autenticación mutua cuando el tercero lo soporte, y las credenciales deben rotarse automáticamente. | • 100% tráfico cifrado (TLS 1.3); mTLS cuando el tercero lo soporte <br>• Rotación de credenciales cada 90 días (Secrets Manager) <br>• 100% callbacks validados con firma HMAC <br>• 0 fugas de credenciales (auditoría de logs) |
| RNF-D6-04 | **Trazabilidad de integraciones (D6)** | Toda interacción con sistemas externos (éxito, fallo, timeout, circuit open) debe quedar registrada con latencia, correlation_id y resultado para auditoría y diagnóstico. | • 100% llamadas trazadas (cada llamada genera `ExternalCallLog`) <br>• P95 latencia de registro en D8 < 500 ms <br>• 100% registros con `correlation_id` end-to-end |
| RNF-D7-01 | **Escalabilidad bajo picos de nómina (D7)** | El dominio debe soportar 20K–30K pagos en ventanas críticas sin degradar el SLA global. | • Tiempo total de procesamiento de 30K pagos < 15 min <br>• P95 < 2 s durante pico (SLA general) <br>• Nuevos pods activos < 60 s (escalado automático HPA) <br>• 0 mensajes perdidos (Kafka) |
| RNF-D7-02 | **Independencia y fiabilidad de pagos individuales (D7)** | El fallo de un pago no puede afectar el resto del lote. | • 0 pagos duplicados (idempotency key por `payroll_payment_id`) <br>• 0 pagos perdidos <br>• Tiempo máximo de compensación < 5 s (Saga por pago individual) <br>• 0 transacciones huérfanas en producción (job reconciliador) |
| RNF-D7-03 | **Trazabilidad por lote y por empleado (D7)** | Cada pago debe ser trazable individualmente y como parte de su lote padre. | • 100% pagos generan evento de auditoría <br>• Latencia de auditoría < 500 ms <br>• Consulta de lote completo < 2 s |
| RNF-D7-04 | **Automatización de nómina programada (D7)** | Los pagos programados deben ejecutarse automáticamente y solo una vez por fecha configurada. | • 100% ejecuciones sin duplicados (scheduler transaccional + lock distribuido) <br>• Desviación horaria < 1 min <br>• Tolerancia a reinicio: no duplicar ejecución tras recovery |
| RNF-D7-05 | **Aislamiento de fallos por empresa (D7)** | El fallo en la nómina de una empresa no debe impactar las demás. | • 0 impacto cruzado entre empresas (colas separadas por empresa) <br>• Detección de API caída < 5 s (circuit breaker por empresa) <br>• Recuperación automática sin intervención manual (bulkhead pattern) |
| RNF-D8-01 | **Inmutabilidad e integridad del registro de auditoría (D8)** | Todo evento registrado en el audit store debe ser inmutable (append-only). Ningún registro puede ser modificado, eliminado ni reordenado. La integridad se garantiza mediante encadenamiento de hashes y firma digital del payload. | • 0 modificaciones permitidas (test UPDATE/DELETE → rechazado) <br>• 100% integridad hash chain verificada periódicamente <br>• 100% firmas digitales válidas (verificación diaria) <br>• 100% eventos persistidos (cobertura) <br>• 0 duplicados (deduplicación por `event_id`) |
| RNF-D8-02 | **Detección de patrones sospechosos y retroalimentación de listas (D8)** | D8 analiza streams de eventos para detectar patrones sospechosos (transferencias frecuentes al mismo destino, montos atípicos, actividad desde cuentas en lista gris) mediante procesamiento post-hoc y correlación histórica. Al detectar un patrón, actualiza las listas blancas/grises/negras que D4 consume para validación en línea. La validación en tiempo real antes de ejecutar cada transferencia es responsabilidad de D4 (Primario). | • P95 detección de patrón < 5 s (evento → `SuspiciousPatternDetected`) <br>• 100% reglas CEP activas en Flink <br>• < 5% falsos positivos <br>• Propagación de listas a D4 < 60 s (`FraudListUpdated` → invalidación caché D4) |
| RNF-D8-03 | **Generación y envío de reportes regulatorios en plazo (D8)** | El sistema debe generar y enviar automáticamente: (a) extracto trimestral de movimientos a cada banco filial, por usuario; (b) reporte semestral a la Superintendencia Financiera con el detalle de todos los movimientos. | • 100% trimestres cubiertos (scheduler cada 3 meses) <br>• 100% semestres cubiertos (scheduler cada 6 meses) <br>• 100% envíos exitosos a bancos y Superfinanciera (confirmación vía D6) <br>• 0 errores de formato (validación de schema) <br>• 0 faltantes (verificación de completitud del período) |
| RNF-D8-04 | **Observabilidad del sistema completo (D8)** | D8 debe proveer dashboards operacionales que permitan buscar transacciones por correlation_id, consultar alertas de fraude activas, ver estado de lotes de nómina y métricas de cumplimiento en tiempo real. | • P95 búsqueda por correlation_id < 2 s (OpenSearch) <br>• P95 latencia de indexación < 30 s (evento → disponible) <br>• 99.9% uptime del dashboard (health check Dashboard API) <br>• Alertas automáticas ante umbrales críticos < 60 s (Grafana) |
| RNF-D8-05 | **Retención y cifrado de datos de auditoría (D8)** | Los datos de auditoría deben retenerse según la normativa vigente (mínimo 5 años para transacciones financieras). Todo dato en reposo debe estar cifrado. | • 100% registros retenidos > 5 años consultables <br>• 100% cifrados en reposo (Keyspaces + OpenSearch con KMS) <br>• 100% reportes cifrados en S3 (SSE-KMS) <br>• ILM: hot (0–6 meses) → warm (6 meses–2 años) → cold (2–5+ años); nunca eliminar antes de 5 años |

---

## RNF por dominio — Detalle completo

A continuación se listan todos los RNF específicos de cada dominio, con sus funciones de ajuste (fitness functions) y tácticas. Esta información complementa la Tabla 10 y se corresponde con el detalle documentado en cada archivo de dominio.

---

### D1 — IAM Service

#### RNF-D1-01 — Disponibilidad 24/7 del servicio IAM

**Tabla 11. Requisito no funcional RNF-D1-01**

| Campo | Detalle |
|-------|---------|
| **Descripción** | El servicio IAM debe operar 24/7; su indisponibilidad bloquea el acceso a todo el sistema. |
| **Origen** | RNF-01 (Primario) |
| **Categoría RNF** | Disponibilidad |

**Funciones de ajuste:**

| # | Función de ajuste | Mecanismo | Métrica objetivo |
|---|-------------------|-----------|-----------------|
| FF-D1-01-A | Uptime mensual del servicio IAM | Monitoreo en CloudWatch + SLO | ≥ 99.9% |
| FF-D1-01-B | Tiempo de recuperación ante fallo | Auto-healing EKS + Multi-AZ | < 60 s |
| FF-D1-01-C | Degradación en picos de login | Prueba de carga en ventana pico | P95 login < 1.5 s |

**Tácticas:**
- Despliegue en Amazon EKS Multi-AZ con réplicas mínimas y HPA.
- Aurora PostgreSQL Multi-AZ para persistencia.
- Readiness/Liveness probes y reinicio automático por K8s.
- Protección en gateway para reducir presión de bots (rate limiting + WAF).

---

#### RNF-D1-02 — Seguridad de autenticación y emisión de tokens

**Tabla 12. Requisito no funcional RNF-D1-02**

| Campo | Detalle |
|-------|---------|
| **Descripción** | El dominio debe implementar autenticación robusta, MFA y emisión segura de JWT firmados. |
| **Origen** | RNF-04 (Primario) |
| **Categoría RNF** | Seguridad |

**Funciones de ajuste:**

| # | Función de ajuste | Mecanismo | Métrica objetivo |
|---|-------------------|-----------|-----------------|
| FF-D1-02-A | Fuerza criptográfica de tokens | JWT RS256 + rotación de claves KMS | 100% tokens firmados |
| FF-D1-02-B | Ventana de exposición del token | exp corto + refresh controlado | Access token ≤ 15 min |
| FF-D1-02-C | Canal seguro extremo a extremo | TLS 1.3 | 100% tráfico cifrado |

**Tácticas:**
- Firma JWT con claves en AWS KMS (rotación planificada).
- TLS 1.3 en API Gateway y mTLS opcional inter-servicio (Istio).
- MFA obligatorio para roles críticos (ROLE_SECURITY_ADMIN, ROLE_PAYROLL_MANAGER).

---

#### RNF-D1-03 — Protección contra fuerza bruta y abuso

**Tabla 13. Requisito no funcional RNF-D1-03**

| Campo | Detalle |
|-------|---------|
| **Descripción** | El sistema debe mitigar ataques de fuerza bruta y abuso del endpoint de login. |
| **Origen** | RNF-04 (Primario) |
| **Categoría RNF** | Seguridad / Resiliencia |

**Funciones de ajuste:**

| # | Función de ajuste | Mecanismo | Métrica objetivo |
|---|-------------------|-----------|-----------------|
| FF-D1-03-A | Bloqueo por intentos fallidos | contador Redis por usuario/IP | Bloquear a los 5 intentos |
| FF-D1-03-B | Rate limiting | API Gateway throttling | < 1% requests rechazadas legítimas |
| FF-D1-03-C | Detección temprana de abuso | WAF rules + GuardDuty | alerta < 60 s |

**Tácticas:**
- Contadores y ventanas temporales en ElastiCache Redis.
- AWS WAF con reglas OWASP + reputación IP.
- GuardDuty para señales de ataque (VPC/CloudTrail/DNS).

---

#### RNF-D1-04 — Autorización granular (RBAC + scopes)

**Tabla 14. Requisito no funcional RNF-D1-04**

| Campo | Detalle |
|-------|---------|
| **Descripción** | El acceso a operaciones críticas debe controlarse por rol y alcance (scope). |
| **Origen** | RNF-04 (Primario) |
| **Categoría RNF** | Seguridad / Control de acceso |

**Funciones de ajuste:**

| # | Función de ajuste | Mecanismo | Métrica objetivo |
|---|-------------------|-----------|-----------------|
| FF-D1-04-A | Cobertura de endpoints protegidos | pruebas automatizadas | 100% endpoints sensibles protegidos |
| FF-D1-04-B | Consistencia de permisos | pruebas RBAC por rol | 0 rutas expuestas sin rol |

**Tácticas:**
- JWT con claims roles y scope.
- Validación en API Gateway (authorizer) y enforcement adicional en cada microservicio.
- Pruebas automatizadas por rol (CI/CD).

---

#### RNF-D1-05 — Trazabilidad y cumplimiento de accesos

**Tabla 15. Requisito no funcional RNF-D1-05**

| Campo | Detalle |
|-------|---------|
| **Descripción** | Todo acceso (éxito/fallo) debe quedar trazado para auditoría, investigación y cumplimiento. |
| **Origen** | RNF-06 (Primario) |
| **Categoría RNF** | Trazabilidad / Cumplimiento |

**Funciones de ajuste:**

| # | Función de ajuste | Mecanismo | Métrica objetivo |
|---|-------------------|-----------|-----------------|
| FF-D1-05-A | Cobertura de eventos de autenticación | integración D1→MSK→D8 | 100% eventos publicados |
| FF-D1-05-B | Latencia hacia auditoría | medición end-to-end | P95 < 500 ms |
| FF-D1-05-C | Retención de logs | política en D8/OpenSearch | ≥ 5 años (según política) |

**Tácticas:**
- Publicación de eventos a Kafka con Outbox Pattern (entrega garantizada).
- Logs en CloudWatch con correlación por correlation_id.
- D8 persiste histórico append-only (cumplimiento).

---

### D2 — Usuarios y Cuentas

#### RNF-D2-01 — Sincronización diaria idempotente con bancos

**Tabla 16. Requisito no funcional RNF-D2-01**

| Campo | Detalle |
|-------|---------|
| **Descripción** | La sincronización de cuentas con bancos debe ejecutarse diariamente de forma idempotente, con trazabilidad y sin duplicar registros. |
| **Origen** | RNF-07 |
| **Categoría RNF** | Fiabilidad / Consistencia |

**Funciones de ajuste:**

| # | Función de ajuste | Mecanismo | Métrica objetivo |
|---|-------------------|-----------|-----------------|
| FF-D2-01-A | Idempotencia del job | Ejecutar misma carga 2 veces | 0 duplicados |
| FF-D2-01-B | Tasa de éxito del sync | % SUCCESS mensual | ≥ 99% |
| FF-D2-01-C | Latencia de actualización | Cambio en banco → reflejo | P95 < 10 min |

**Tácticas:**
- Upsert por clave natural.
- Registro de SyncJob con resumen.
- Reintentos con backoff.

---

#### RNF-D2-02 — Rendimiento de consultas (< 2 s)

**Tabla 17. Requisito no funcional RNF-D2-02**

| Campo | Detalle |
|-------|---------|
| **Descripción** | Operaciones de consulta deben responder en menos de 2 segundos. Los saldos se obtienen en tiempo real desde bancos filiales vía D6; se usa un caché de corta duración (TTL) para reducir latencia sin almacenar saldos permanentemente. |
| **Origen** | RNF-02 |
| **Categoría RNF** | Rendimiento |

**Funciones de ajuste:**

| # | Función de ajuste | Métrica objetivo |
|---|-------------------|-----------------|
| FF-D2-02-A | Tiempo respuesta | P95 < 2 s |
| FF-D2-02-B | Cache hit rate (TTL corto, proxy de saldo) | ≥ 80% |
| FF-D2-02-C | Degradación pico | 5xx < 0.1% |

**Tácticas:**
- Redis con TTL corto para cachear respuestas de saldo desde bancos (proxy cache, no almacenamiento permanente).
- Índices por user_id y bank_id para metadata de cuentas.
- Paginación server-side.

---

#### RNF-D2-03 — Escalabilidad (25M usuarios)

**Tabla 18. Requisito no funcional RNF-D2-03**

| Campo | Detalle |
|-------|---------|
| **Descripción** | Debe soportar decenas de millones de usuarios y alto volumen concurrente. |
| **Origen** | RNF-03 |
| **Categoría RNF** | Escalabilidad |

**Funciones de ajuste:**

| # | Función de ajuste | Métrica objetivo |
|---|-------------------|-----------------|
| FF-D2-03-A | Escalado horizontal | Mantener P95 < 2 s |
| FF-D2-03-B | Concurrencia | Sin caída en picos |
| FF-D2-03-C | Crecimiento BD | Sin degradación crítica |

**Tácticas:**
- EKS + HPA.
- Aurora Serverless v2.
- CQRS ligero.

---

#### RNF-D2-04 — Seguridad y cumplimiento

**Tabla 19. Requisito no funcional RNF-D2-04**

| Campo | Detalle |
|-------|---------|
| **Descripción** | Protección de datos sensibles y control de acceso estricto. |
| **Origen** | RNF-04 |
| **Categoría RNF** | Seguridad |

**Funciones de ajuste:**

| # | Función de ajuste | Métrica objetivo |
|---|-------------------|-----------------|
| FF-D2-04-A | Cifrado en reposo | 100% tablas cifradas |
| FF-D2-04-B | Control de acceso | 0 endpoints expuestos |
| FF-D2-04-C | OWASP | 0 vulnerabilidades críticas |

**Tácticas:**
- JWT validado por gateway.
- WAF + rate limiting.
- Auditoría de accesos.

---

#### RNF-D2-05 — Trazabilidad de sincronizaciones

**Tabla 20. Requisito no funcional RNF-D2-05**

| Campo | Detalle |
|-------|---------|
| **Descripción** | Cada cambio de estado y sincronización debe ser trazable. |
| **Origen** | RNF-06 |
| **Categoría RNF** | Trazabilidad |

**Funciones de ajuste:**

| # | Función de ajuste | Métrica objetivo |
|---|-------------------|-----------------|
| FF-D2-05-A | Cobertura de eventos | 100% cambios publican evento |
| FF-D2-05-B | Latencia auditoría | P95 < 500 ms |
| FF-D2-05-C | Inmutabilidad | append-only en D8 |

**Tácticas:**
- Outbox Pattern.
- Correlation ID.
- SyncJob.summary persistido.

---

### D4 — Transferencias y Transacciones (complemento)

#### RNF-D4-03 — Resiliencia en la integración con ACH

**Tabla 21. Requisito no funcional RNF-D4-03**

| Campo | Detalle |
|-------|---------|
| **Descripción** | La indisponibilidad temporal del sistema ACH no debe interrumpir las transferencias entre bancos filiales ni bloquear indefinidamente las transferencias diferidas. Las transacciones enviadas a ACH deben poder ser reenviadas de forma segura sin duplicarlas. |
| **Origen** | Consideración 9 (Secundario en D4 / Primario técnico en D6) |
| **Categoría RNF** | Resiliencia / Disponibilidad |

**Funciones de ajuste:**

| # | Función de ajuste | Mecanismo | Métrica objetivo |
|---|-------------------|-----------|-----------------|
| FF-D4-03-A | Tiempo de detección de ACH no disponible | Circuit Breaker en D6 | < 5 s |
| FF-D4-03-B | Aislamiento del fallo de ACH | Test de caos: caída ACH → filial–filial no afectada | 0 impacto en liquidaciones inmediatas |
| FF-D4-03-C | Reenvío seguro a ACH tras recuperación | Reenvío idempotente | 0 duplicados en ACH |
| FF-D4-03-D | Cobertura de DLQ para eventos fallidos | Dead Letter Queue | 100% eventos fallidos trazables |

**Tácticas:**
- Circuit Breaker en D6 hacia ACH (Resilience4j).
- Idempotency Key en cada llamada a ACH.
- Dead Letter Queue en Kafka para mensajes fallidos.
- Job de retransmisión: transacciones en `SENT_TO_ACH` sin callback en > 30 min se reencolan.

---

#### RNF-D4-04 — Gestión completa del ciclo de estados ACH

**Tabla 22. Requisito no funcional RNF-D4-04**

| Campo | Detalle |
|-------|---------|
| **Descripción** | El sistema debe procesar correctamente todos los estados posibles que ACH puede comunicar (procedente, rechazada, en revisión, timeout) y transicionar la transacción al estado interno correcto, liberando o revirtiendo fondos según corresponda. |
| **Origen** | Consideración 10 (Primario) |
| **Categoría RNF** | Correctitud / Fiabilidad |

**Funciones de ajuste:**

| # | Función de ajuste | Mecanismo | Métrica objetivo |
|---|-------------------|-----------|-----------------|
| FF-D4-04-A | Cobertura de estados ACH | Test unitario por cada respuesta posible | 100% de casos documentados |
| FF-D4-04-B | Tiempo de procesamiento del callback | Traza end-to-end | P95 < 2 s |
| FF-D4-04-C | 0 transacciones sin resolución | Job de reconciliación diario | Alerta automática + escalación |
| FF-D4-04-D | Liberación de fondos ante fallo ACH | Simular `ACH_REJECTED` → verificar liberación HOLD | 100% fallos revierten HOLD |

**Tácticas:**
- Máquina de estados explícita (patrón State o tabla de transiciones en DB).
- Endpoint dedicado para callbacks de ACH (vía D6) con validación HMAC.
- Cada transición es transaccional: estado + evento en el mismo Outbox.
- Estado `UNKNOWN_ACH_RESPONSE` + alerta inmediata para respuestas desconocidas.

---

#### RNF-D4-06 — Disponibilidad y rendimiento del motor de transferencias

**Tabla 23. Requisito no funcional RNF-D4-06**

| Campo | Detalle |
|-------|---------|
| **Descripción** | El motor de transferencias debe operar 24/7 con un tiempo de respuesta menor a 2 segundos para la confirmación al usuario, y debe mantener el SLA incluso durante picos de carga (días de nómina masiva). |
| **Origen** | RNF-01 y RNF-02 globales |
| **Categoría RNF** | Disponibilidad / Rendimiento |

**Funciones de ajuste:**

| # | Función de ajuste | Mecanismo | Métrica objetivo |
|---|-------------------|-----------|-----------------|
| FF-D4-06-A | Tiempo de respuesta end-to-end | Traza HTTP hasta confirmación | P95 < 2 s |
| FF-D4-06-B | Disponibilidad del servicio | Health checks + uptime monitoring | ≥ 99.9% mensual |
| FF-D4-06-C | Degradación controlada bajo carga extrema | Test de carga 5× volumen pico | < 5 s modo degradado; 0 pérdida de datos |
| FF-D4-06-D | Recuperación ante reinicio de pod | K8s liveness/readiness probes | < 30 s |

**Tácticas:**
- Separación de la confirmación al usuario del proceso de liquidación.
- Escalado horizontal automático (HPA) al detectar latencia > 1.5 s o CPU > 70%.
- Back-pressure: HTTP 429 con `Retry-After` si la cola de sagas supera umbral.
- Replicación activa-activa en al menos 2 zonas de disponibilidad.

---

### D5 — Billetera Digital (complemento)

#### RNF-D5-03 — Rendimiento de consulta de saldo

**Tabla 24. Requisito no funcional RNF-D5-03**

| Campo | Detalle |
|-------|---------|
| **Descripción** | La consulta de saldo de la billetera debe responder en menos de 2 segundos, incluso con alto volumen de registros en el ledger. |
| **Origen** | RNF-02 (Primario) |
| **Categoría RNF** | Rendimiento |

**Funciones de ajuste:**

| # | Función de ajuste | Mecanismo | Métrica objetivo |
|---|-------------------|-----------|-----------------|
| FF-D5-03-A | Tiempo de respuesta de consulta de saldo | Prueba de carga concurrente | P95 < 500 ms |
| FF-D5-03-B | Escalabilidad del cálculo | Prueba con billeteras de >100K movimientos | Sin degradación crítica |

**Tácticas:**
- Materialización periódica del saldo en caché Redis.
- Índice compuesto por `wallet_id` + `created_at`.
- CQRS ligero: lecturas desde vista materializada; escrituras al ledger.

---

#### RNF-D5-04 — Seguridad y control de acceso

**Tabla 25. Requisito no funcional RNF-D5-04**

| Campo | Detalle |
|-------|---------|
| **Descripción** | Solo el titular de la billetera puede operar sobre ella. Todas las operaciones requieren token JWT válido con el scope correspondiente. |
| **Origen** | RNF-04 (Primario) |
| **Categoría RNF** | Seguridad |

**Funciones de ajuste:**

| # | Función de ajuste | Mecanismo | Métrica objetivo |
|---|-------------------|-----------|-----------------|
| FF-D5-04-A | Control de acceso por titular | Test RBAC: usuario A no puede operar billetera de B | 0 accesos cruzados |
| FF-D5-04-B | Cifrado en reposo | Aurora + KMS | 100% datos cifrados |
| FF-D5-04-C | Canal seguro | TLS 1.3 | 100% tráfico cifrado |

**Tácticas:**
- JWT validado por API Gateway con scope `wallet:read`, `wallet:write`, `wallet:pay`.
- Enforcement adicional: `wallet.user_id == token.sub`.
- Cifrado en reposo con AWS KMS.

---

#### RNF-D5-05 — Trazabilidad de movimientos de billetera

**Tabla 26. Requisito no funcional RNF-D5-05**

| Campo | Detalle |
|-------|---------|
| **Descripción** | Todo movimiento de la billetera (débito, crédito, compensación) debe quedar registrado en el log de auditoría de D8 para cumplimiento regulatorio. |
| **Origen** | RNF-06 (Primario) |
| **Categoría RNF** | Trazabilidad / Cumplimiento |

**Funciones de ajuste:**

| # | Función de ajuste | Mecanismo | Métrica objetivo |
|---|-------------------|-----------|-----------------|
| FF-D5-05-A | Cobertura de eventos publicados | Test: cada operación genera evento | 100% de operaciones trazadas |
| FF-D5-05-B | Latencia del evento al log | Publicación a D8 | P95 < 500 ms |
| FF-D5-05-C | Inmutabilidad del log | append-only en D8 | 0 modificaciones permitidas |

**Tácticas:**
- Outbox Pattern (misma transacción ACID).
- Correlation ID en todos los eventos.
- D8 consume y persiste en append-only store.

---

### D6 — Integraciones y Pasarelas (complemento)

#### RNF-D6-03 — Seguridad en comunicación con sistemas externos

**Tabla 27. Requisito no funcional RNF-D6-03**

| Campo | Detalle |
|-------|---------|
| **Descripción** | Toda comunicación con sistemas externos debe usar canales cifrados (TLS 1.3 mínimo), con autenticación mutua cuando el tercero lo soporte, y las credenciales deben rotarse automáticamente. |
| **Origen** | Consideración 26 (Secundario) / RNF-04 (Seguridad) |
| **Categoría RNF** | Seguridad |

**Funciones de ajuste:**

| # | Función de ajuste | Mecanismo | Métrica objetivo |
|---|-------------------|-----------|-----------------|
| FF-D6-03-A | Canal cifrado | TLS en todas las conexiones salientes | 100% tráfico cifrado (TLS 1.3) |
| FF-D6-03-B | Rotación de credenciales | Secrets Manager rotation schedule | Cada 90 días |
| FF-D6-03-C | Validación de callbacks | Firma HMAC | 100% callbacks validados |
| FF-D6-03-D | No exposición de credenciales | Auditoría de logs | 0 fugas |

**Tácticas:**
- TLS 1.3 obligatorio; mTLS cuando el tercero lo soporte.
- Credenciales en AWS Secrets Manager con rotación automática.
- Validación HMAC en callbacks de ACH y pasarelas.
- Sanitización de logs.

---

#### RNF-D6-04 — Trazabilidad de integraciones

**Tabla 28. Requisito no funcional RNF-D6-04**

| Campo | Detalle |
|-------|---------|
| **Descripción** | Toda interacción con sistemas externos (éxito, fallo, timeout, circuit open) debe quedar registrada con latencia, correlation_id y resultado para auditoría y diagnóstico. |
| **Origen** | RNF-06 (Trazabilidad) / RNF-09 (Observabilidad) |
| **Categoría RNF** | Trazabilidad / Observabilidad |

**Funciones de ajuste:**

| # | Función de ajuste | Mecanismo | Métrica objetivo |
|---|-------------------|-----------|-----------------|
| FF-D6-04-A | Cobertura de logs | Cada llamada genera `ExternalCallLog` | 100% trazadas |
| FF-D6-04-B | Latencia publicada a D8 | Tiempo hasta registro en D8 | P95 < 500 ms |
| FF-D6-04-C | Correlación end-to-end | `correlation_id` en todos los registros | 100% |

**Tácticas:**
- `ExternalCallLog` con latencia, estado y correlation_id.
- Publicación de eventos a Kafka para D8.
- Dashboards de latencia por adapter en Grafana.

---

### D7 — Pagos Masivos a Empleados

#### RNF-D7-01 — Escalabilidad bajo picos de nómina

**Tabla 29. Requisito no funcional RNF-D7-01**

| Campo | Detalle |
|-------|---------|
| **Descripción** | El dominio debe soportar 20K–30K pagos en ventanas críticas sin degradar el SLA global. |
| **Origen** | Consideraciones 21, 22, 23, 24 |
| **Categoría RNF** | Escalabilidad |

**Funciones de ajuste:**

| # | Función de ajuste | Métrica objetivo |
|---|-------------------|-----------------|
| FF-D7-01-A | Tiempo total de procesamiento de 30K pagos | < 15 minutos |
| FF-D7-01-B | SLA general durante pico | P95 < 2 s |
| FF-D7-01-C | Escalado automático | Nuevos pods activos < 60 s |
| FF-D7-01-D | Pérdida de mensajes | 0 mensajes perdidos |

**Tácticas:**
- Queue-based load leveling con Kafka.
- Worker Pool escalable horizontalmente (HPA).
- Particionamiento por empresa.
- Pruebas de carga periódicas.

---

#### RNF-D7-02 — Independencia y fiabilidad de pagos individuales

**Tabla 30. Requisito no funcional RNF-D7-02**

| Campo | Detalle |
|-------|---------|
| **Descripción** | El fallo de un pago no puede afectar el resto del lote. |
| **Categoría RNF** | Fiabilidad |

**Funciones de ajuste:**

| # | Función de ajuste | Métrica objetivo |
|---|-------------------|-----------------|
| FF-D7-02-A | Pagos duplicados | 0 duplicados |
| FF-D7-02-B | Pagos perdidos | 0 pagos perdidos |
| FF-D7-02-C | Tiempo máximo de compensación | < 5 s |
| FF-D7-02-D | Transacciones huérfanas | 0 en producción |

**Tácticas:**
- Idempotency key por `payroll_payment_id`.
- Saga por pago individual.
- Dead Letter Queue.
- Job reconciliador.

---

#### RNF-D7-03 — Trazabilidad por lote y por empleado

**Tabla 31. Requisito no funcional RNF-D7-03**

| Campo | Detalle |
|-------|---------|
| **Descripción** | Cada pago debe ser trazable individualmente y como parte de su lote padre. |
| **Categoría RNF** | Trazabilidad |

**Funciones de ajuste:**

| # | Función de ajuste | Métrica objetivo |
|---|-------------------|-----------------|
| FF-D7-03-A | Cobertura de eventos | 100% pagos generan evento |
| FF-D7-03-B | Latencia de auditoría | < 500 ms |
| FF-D7-03-C | Consulta de lote | < 2 s |

**Tácticas:**
- `payroll_batch_id` como Correlation ID.
- Event Sourcing en D8.
- Registro append-only.

---

#### RNF-D7-04 — Automatización de nómina programada

**Tabla 32. Requisito no funcional RNF-D7-04**

| Campo | Detalle |
|-------|---------|
| **Descripción** | Los pagos programados deben ejecutarse automáticamente y solo una vez por fecha configurada. |
| **Categoría RNF** | Automatización |

**Funciones de ajuste:**

| # | Función de ajuste | Métrica objetivo |
|---|-------------------|-----------------|
| FF-D7-04-A | Ejecución única | 100% sin duplicados |
| FF-D7-04-B | Precisión horaria | Desviación < 1 min |
| FF-D7-04-C | Tolerancia a reinicio | No duplicar ejecución |

**Tácticas:**
- Scheduler transaccional.
- Lock distribuido.
- Registro de ejecución por fecha.

---

#### RNF-D7-05 — Aislamiento de fallos por empresa

**Tabla 33. Requisito no funcional RNF-D7-05**

| Campo | Detalle |
|-------|---------|
| **Descripción** | El fallo en la nómina de una empresa no debe impactar las demás. |
| **Categoría RNF** | Resiliencia |

**Funciones de ajuste:**

| # | Función de ajuste | Métrica objetivo |
|---|-------------------|-----------------|
| FF-D7-05-A | Impacto cruzado | 0 impacto entre empresas |
| FF-D7-05-B | Detección de API caída | < 5 s |
| FF-D7-05-C | Recuperación automática | Sin intervención manual |

**Tácticas:**
- Colas separadas por empresa.
- Circuit breaker por empresa.
- Bulkhead pattern.

---

### D8 — Reportes, Auditoría y Cumplimiento

#### RNF-D8-01 — Inmutabilidad y integridad del registro de auditoría

**Tabla 34. Requisito no funcional RNF-D8-01**

| Campo | Detalle |
|-------|---------|
| **Descripción** | Todo evento registrado en el audit store debe ser inmutable (append-only). Ningún registro puede ser modificado, eliminado ni reordenado. La integridad se garantiza mediante encadenamiento de hashes y firma digital del payload. |
| **Origen** | Consideración 11, 30 (Primario) / RNF-06 (Trazabilidad) |
| **Categoría RNF** | Integridad / Trazabilidad |

**Funciones de ajuste:**

| # | Función de ajuste | Mecanismo | Métrica objetivo |
|---|-------------------|-----------|-----------------|
| FF-D8-01-A | Inmutabilidad del store | Test UPDATE/DELETE → rechazado | 0 modificaciones permitidas |
| FF-D8-01-B | Integridad del hash chain | Verificación periódica | 100% integridad verificada |
| FF-D8-01-C | Firma digital de payloads | Verificar firma diariamente | 100% firmas válidas |
| FF-D8-01-D | Cobertura de eventos | % registrados vs publicados | 100% persistidos |
| FF-D8-01-E | Deduplicación | Enviar mismo evento 2 veces → 1 registro | 0 duplicados |

**Tácticas:**
- Amazon Keyspaces append-only: solo INSERT; no UPDATE ni DELETE.
- `hash_chain = SHA-256(prev_hash + event_id + payload)` para integridad secuencial.
- Payload firmado con clave de AWS KMS dedicada para D8.
- Deduplicación por `event_id` en el Event Ingester.

---

#### RNF-D8-02 — Detección de patrones sospechosos y retroalimentación de listas

**Tabla 35. Requisito no funcional RNF-D8-02**

| Campo | Detalle |
|-------|---------|
| **Descripción** | D8 analiza streams de eventos para detectar patrones sospechosos (transferencias frecuentes al mismo destino, montos atípicos, actividad desde cuentas en lista gris) mediante procesamiento post-hoc y correlación histórica. Al detectar un patrón, actualiza las listas blancas/grises/negras que D4 consume para validación en línea. La validación en tiempo real antes de ejecutar cada transferencia es responsabilidad de D4 (Primario). |
| **Origen** | Consideración 30 (Primario) / RNF-04 (Seguridad) |
| **Categoría RNF** | Seguridad / Detección de fraude |

**Funciones de ajuste:**

| # | Función de ajuste | Mecanismo | Métrica objetivo |
|---|-------------------|-----------|-----------------|
| FF-D8-02-A | Latencia de detección de patrón | Evento → `SuspiciousPatternDetected` | P95 < 5 s |
| FF-D8-02-B | Cobertura de reglas CEP | % de patrones implementados en Flink | 100% reglas activas |
| FF-D8-02-C | Tasa de falsos positivos | % alertas FALSE_POSITIVE | < 5% |
| FF-D8-02-D | Propagación de listas a D4 | Actualización → invalidación caché D4 | < 60 s |

**Tácticas:**
- Amazon Managed Flink con reglas CEP sobre ventanas deslizantes.
- Reglas configurables: frecuencia, desviación de montos, actividad geográfica atípica.
- Al detectar un patrón, D8 publica `SuspiciousPatternDetected` a Kafka y actualiza las listas via `FraudListUpdated`; D4 consume las listas actualizadas para validación en línea y D1 puede bloquear sesiones preventivamente.
- Fraud List Manager propaga a Redis (TTL 60 s) y publica `FraudListUpdated`.

---

#### RNF-D8-03 — Generación y envío de reportes regulatorios en plazo

**Tabla 36. Requisito no funcional RNF-D8-03**

| Campo | Detalle |
|-------|---------|
| **Descripción** | El sistema debe generar y enviar automáticamente: (a) extracto trimestral de movimientos a cada banco filial, por usuario; (b) reporte semestral a la Superintendencia Financiera con el detalle de todos los movimientos. |
| **Origen** | Consideraciones 33, 34, 35 (Primario) / RNF-06 (Cumplimiento) |
| **Categoría RNF** | Cumplimiento regulatorio |

**Funciones de ajuste:**

| # | Función de ajuste | Mecanismo | Métrica objetivo |
|---|-------------------|-----------|-----------------|
| FF-D8-03-A | Generación del extracto trimestral | Scheduler cada 3 meses | 100% trimestres cubiertos |
| FF-D8-03-B | Generación del reporte semestral | Scheduler cada 6 meses | 100% semestres cubiertos |
| FF-D8-03-C | Envío exitoso a bancos | Confirmación vía D6 | 100% bancos notificados |
| FF-D8-03-D | Envío exitoso a Superfinanciera | Confirmación vía D6 | 100% en plazo |
| FF-D8-03-E | Formato correcto | Validación de schema | 0 errores de formato |
| FF-D8-03-F | Completitud de datos | Verificar todos los movimientos del período | 0 faltantes |

**Tácticas:**
- Report Scheduler con cron configurable.
- Report Generator en Python consulta OpenSearch.
- Formatos: PDF/CSV (bancos), XML (Superfinanciera).
- Archivos en S3 (cifrado KMS) antes de envío a D6.
- Ejecución manual ad-hoc para auditorías extraordinarias.

---

#### RNF-D8-04 — Observabilidad del sistema completo

**Tabla 37. Requisito no funcional RNF-D8-04**

| Campo | Detalle |
|-------|---------|
| **Descripción** | D8 debe proveer dashboards operacionales que permitan buscar transacciones por correlation_id, consultar alertas de fraude activas, ver estado de lotes de nómina y métricas de cumplimiento en tiempo real. |
| **Origen** | RNF-09 (Observabilidad) / Consideración 30 |
| **Categoría RNF** | Observabilidad |

**Funciones de ajuste:**

| # | Función de ajuste | Mecanismo | Métrica objetivo |
|---|-------------------|-----------|-----------------|
| FF-D8-04-A | Tiempo de búsqueda por correlation_id | Consulta OpenSearch | P95 < 2 s |
| FF-D8-04-B | Latencia de indexación | Evento → disponible en OpenSearch | P95 < 30 s |
| FF-D8-04-C | Disponibilidad del dashboard | Health check Dashboard API | 99.9% uptime |
| FF-D8-04-D | Alertas automáticas | Grafana sobre métricas de D8 | Alerta en < 60 s |

**Tácticas:**
- Amazon OpenSearch Service (Multi-AZ) como backend full-text.
- Ingesta vía Kinesis Data Firehose (streaming).
- Dashboard API vía API Gateway (protegido por D1).
- Amazon Managed Grafana para dashboards operacionales.

---

#### RNF-D8-05 — Retención y cifrado de datos de auditoría

**Tabla 38. Requisito no funcional RNF-D8-05**

| Campo | Detalle |
|-------|---------|
| **Descripción** | Los datos de auditoría deben retenerse según la normativa vigente (mínimo 5 años para transacciones financieras). Todo dato en reposo debe estar cifrado. |
| **Origen** | Consideración 25 (Secundario) / RNF-04 (Seguridad) / RNF-06 (Cumplimiento) |
| **Categoría RNF** | Cumplimiento / Seguridad |

**Funciones de ajuste:**

| # | Función de ajuste | Mecanismo | Métrica objetivo |
|---|-------------------|-----------|-----------------|
| FF-D8-05-A | Retención mínima | Registros > 5 años consultables | 100% retenidos |
| FF-D8-05-B | Cifrado en reposo | Keyspaces + OpenSearch con KMS | 100% cifrados |
| FF-D8-05-C | Cifrado de reportes | S3 con SSE-KMS | 100% cifrados |
| FF-D8-05-D | Política de TTL en OpenSearch | Index Lifecycle Management | Hot → warm → cold; nunca eliminar antes de 5 años |

**Tácticas:**
- Amazon Keyspaces con TTL deshabilitado (retención indefinida, mínimo 5 años).
- OpenSearch con Index Lifecycle Management: hot (0–6 meses), warm (6 meses–2 años), cold (2–5+ años).
- Cifrado en reposo con AWS KMS en Keyspaces, OpenSearch y S3.
- Reportes en S3 con SSE-KMS y versionado habilitado.

---

## Pendientes

- [ ] Revisar con el equipo si hay RNF adicionales identificados en clase
- [ ] Confirmar métricas concretas para RNF-01 (uptime %) y RNF-02 (P95 < 2 s) con el enunciado
- [ ] Añadir citas del material del curso que respalden las funciones de ajuste [POR DEFINIR]
- [ ] Decidir si RNF-09 y RNF-10 (hipótesis) se incluyen en el reporte o se justifican solo internamente
