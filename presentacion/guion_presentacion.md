# Guion de Presentación — Arquitectura Evolutiva Empresa X

> **Estructura:** Problema → Diagrama general → Dominio por dominio → Transversales → Estrategias de evolución

---

## BLOQUE 1 — Contexto y visión general

---

### Slide 1: Descripción del problema

**Título sugerido:** *"El problema: transferencias interbancarias en Colombia"*

**Puntos clave para presentar:**

- Colombia tiene más de **60 instituciones financieras** con ~35 millones de usuarios activos.
- Las transferencias entre bancos toman entre **1 y 5 días hábiles**.
- La **Empresa X** quiere centralizar el proceso de transacción interbancaria mediante una plataforma única.
- La plataforma debe ser **multiplataforma**: web, móvil híbrido y tablet.

**Funcionalidades de alto nivel:**

| # | Funcionalidad | Detalle clave |
|---|---------------|---------------|
| 1 | Consulta unificada de cuentas | Persona natural o jurídica ve todas sus cuentas en bancos filiales |
| 2 | Transferencias P2P | Mismo banco, entre filiales (inmediata), no filiales e internacionales (vía ACH) |
| 3 | Billetera digital | Cuenta propia de Empresa X: acreditar, debitar, pagar a terceros y pasarelas |
| 4 | Pagos empresariales | Nómina individual y masiva (20K–30K transacciones en ventanas específicas) |
| 5 | Histórico y reportes regulatorios | Extracto trimestral a bancos, reporte semestral a Superintendencia Financiera |

**Cifras del contexto:**

| Dato | Valor |
|------|-------|
| Usuarios activos esperados | ~25 millones |
| Empresas aliadas | 15 |
| Empleados activos (nómina) | ~35 mil |
| Picos de transacciones | 20K–30K en días 14–16 y 29–31 de cada mes |
| Disponibilidad | 24/7 |
| Tiempo de respuesta | < 2 segundos |

> **Nota de presentador:** Enfatizar que el sistema debe integrar terceros a demanda sin afectar el estado del sistema en producción. Esto es un requisito explícito del enunciado que marca toda la arquitectura.

---

### Slide 2: Arquitectura seleccionada

**Título sugerido:** *"Microservicios orientados a eventos — ¿por qué?"*

**Arquitectura elegida:** Microservicios orientados a eventos (*Event-Driven Microservices*)

**Los 4 factores que determinaron la decisión:**

| Factor | Evidencia en el contexto | Respuesta arquitectónica |
|--------|--------------------------|--------------------------|
| **Escala extrema** | ~25 millones de usuarios activos | Escalado horizontal independiente por servicio |
| **Picos predecibles** | 20K–30K tx los días 14–16 y 29–31 | Servicios de pagos masivos escalan sin afectar los demás |
| **Extensibilidad a demanda** | Integrar terceros sin afectar el sistema | Patrón Adapter; nuevos terceros sin redespliegue |
| **Disponibilidad 24/7 + < 2 s** | Operación continua + SLA de respuesta | Desacoplamiento asíncrono; fallos en ACH no bloquean filiales |

**3 patrones complementarios:**

1. **API Gateway** — punto de entrada único: autenticación JWT, WAF, rate-limiting, TLS.
2. **Message Broker (Kafka / MSK)** — desacoplamiento entre dominios, entrega garantizada, replay de eventos.
3. **Bounded Contexts (DDD)** — cada dominio = microservicio(s) + base de datos propia. Se pueden evolucionar de forma independiente.

**Arquitectura descartada:** Monolito modular — no soporta escalado diferenciado de pagos masivos ni extensibilidad a demanda de integraciones.

> **Nota de presentador:** Mencionar que se identificaron 8 dominios (bounded contexts) derivados directamente del enunciado. A continuación se mostrará el diagrama general y luego se entrará dominio por dominio.

---

### Slide 3: Diagrama general de contexto (Figura 1)

**Título sugerido:** *"Vista de contexto — Actores, dominios y sistemas externos"*

**Qué mostrar:** El diagrama de contexto completo (Figura 1 de la Sección 3). Se recomienda usar la imagen generada del Mermaid o la imagen ya existente.

**Elementos a señalar al presentar:**

1. **Actores** (izquierda): Persona Natural, Empresa Aliada, Administrador, Bancos Filiales, Superintendencia Financiera
2. **Capa de entrada**: Frontend (Flutter) → API Gateway (Amazon API Gateway con JWT + WAF)
3. **8 dominios como cajas negras**: D1 (IAM), D2 (Usuarios), D3 (Empresas), D4 (Transferencias), D5 (Billetera), D6 (Integraciones), D7 (Pagos Masivos), D8 (Auditoría)
4. **Infraestructura compartida**: Amazon MSK (Kafka), ElastiCache Redis, OpenSearch, S3
5. **Sistemas externos** (derecha): Pasarelas de pago (PSE, DRUO, Apple Pay), ACH, Terceros
6. **Leyenda de conexiones**:
   - Línea con flecha (→): flujo direccional
   - Línea sin flecha (—): acceso bidireccional (BD, Redis)
   - REST mTLS: llamada síncrona inter-servicio (Istio, no pasa por API Gateway)
   - Kafka (MSK): comunicación asíncrona por eventos

> **Nota de presentador:** Este diagrama es la "foto satelital" del sistema. A partir de aquí vamos a hacer zoom dominio por dominio, mostrando para cada uno: qué hace, qué RNF tiene, cómo los mide (funciones de ajuste y umbrales), y cómo se ve internamente.

---

## BLOQUE 2 — Dominio por dominio

---

### Slide 4: D1 — Identidad y Acceso (IAM)

**Título sugerido:** *"D1: IAM — La puerta de entrada al sistema"*

**Responsabilidad:** Gestionar la autenticación y autorización de todos los actores: personas naturales, empresas, operadores internos y sistemas externos.

**Actores:** Persona natural, empresa, sistema externo (bancos, ACH, terceros)

**Funciones clave:** Login MFA, gestión de sesiones, emisión/validación JWT (OAuth2), RBAC, políticas OWASP

**RNF, Funciones de ajuste y Umbrales:**

| RNF | Descripción | Función de ajuste | Umbral |
|-----|-------------|-------------------|--------|
| RNF-D1-01 | Disponibilidad 24/7 del IAM | Uptime mensual (CloudWatch + SLO) | **≥ 99.9%** |
| | | Recuperación ante fallo (Auto-healing EKS Multi-AZ) | **< 60 s** |
| | | Degradación en picos de login | **P95 login < 1.5 s** |
| RNF-D1-02 | Seguridad de autenticación | Tokens firmados (JWT RS256 + rotación KMS) | **100% firmados** |
| | | Ventana de exposición del token | **Access token ≤ 15 min** |
| | | Canal seguro | **100% TLS 1.3** |
| RNF-D1-03 | Protección fuerza bruta | Bloqueo por intentos fallidos (Redis) | **Bloqueo a los 5 intentos** |
| | | Rate limiting (API Gateway) | **< 1% rechazos legítimos** |
| | | Detección de abuso (WAF + GuardDuty) | **Alerta < 60 s** |
| RNF-D1-04 | Autorización granular (RBAC) | Cobertura endpoints protegidos | **100% protegidos** |
| | | Consistencia de permisos | **0 rutas sin rol** |
| RNF-D1-05 | Trazabilidad de accesos | Eventos de autenticación (D1→MSK→D8) | **100% publicados** |
| | | Latencia hacia auditoría | **P95 < 500 ms** |
| | | Retención de logs | **≥ 5 años** |

**Diagrama interno (Figura 2):** Mostrar el diagrama de detalle de D1 con: Auth API (Keycloak 24 + NestJS), RBAC + Scopes, MFA Service, PostgreSQL on-premise (Patroni HA), ElastiCache Redis (vía Direct Connect), Outbox Publisher.

**Comunicación clave:**
- Entrante síncrono: Login/refresh vía API Gateway
- Saliente asíncrono: `UserAuthenticated`, `UserLocked` → D8 (vía Kafka + Direct Connect)
- Entrante asíncrono: `SuspiciousPatternDetected` desde D8

> **Nota de presentador:** D1 es on-premise en Colombia por soberanía de datos. Si D1 cae, nadie accede al sistema — por eso exige 99.9% uptime y auto-healing < 60 s.

---

### Slide 5: D2 — Gestión de Usuarios y Cuentas

**Título sugerido:** *"D2: Usuarios y Cuentas — 25 millones de usuarios, sincronización diaria"*

**Responsabilidad:** Registrar y mantener actualizada la información de personas naturales y jurídicas junto con sus cuentas bancarias en bancos filiales.

**Actores:** Persona natural, empresa aliada, bancos filiales, administrador

**Funciones clave:** Carga masiva desde bancos (ETL), sincronización diaria idempotente, consulta unificada de cuentas, consulta de saldos en tiempo real vía D6

**RNF, Funciones de ajuste y Umbrales:**

| RNF | Descripción | Función de ajuste | Umbral |
|-----|-------------|-------------------|--------|
| RNF-D2-01 | Sincronización diaria idempotente | Idempotencia del job (ejecutar 2 veces) | **0 duplicados** |
| | | Tasa de éxito del sync | **≥ 99% mensual** |
| | | Latencia de actualización | **P95 < 10 min** |
| RNF-D2-02 | Rendimiento de consultas | Tiempo de respuesta | **P95 < 2 s** |
| | | Cache hit rate (Redis, TTL corto) | **≥ 80%** |
| | | Degradación en pico | **5xx < 0.1%** |
| RNF-D2-03 | Escalabilidad (25M usuarios) | Escalado horizontal (EKS + HPA) | **P95 < 2 s sostenido** |
| | | Concurrencia | **Sin caída en picos** |
| | | Crecimiento BD (Aurora Serverless v2) | **Sin degradación crítica** |
| RNF-D2-04 | Seguridad y cumplimiento | Cifrado en reposo | **100% tablas cifradas** |
| | | Control de acceso | **0 endpoints expuestos** |
| | | OWASP | **0 vulnerabilidades críticas** |
| RNF-D2-05 | Trazabilidad de sincronizaciones | Cobertura de eventos (Outbox Pattern) | **100% cambios publicados** |
| | | Latencia auditoría | **P95 < 500 ms** |
| | | Inmutabilidad | **append-only en D8** |

**Diagrama interno (Figura 3):** Users API, Accounts API, Sync Jobs ETL, PostgreSQL on-premise (vía Direct Connect), ElastiCache Redis, Outbox Publisher.

**Comunicación clave:**
- Entrante síncrono: Cliente vía API Gateway, D4 (saldo/límites), D5 (cuentas asociadas), D3 (empresa → cuentas)
- Saliente síncrono: D6 (saldos en tiempo real a bancos)
- Entrante batch: Bancos filiales (archivos/API/SFTP)
- Saliente asíncrono: `UserRegistered`, `AccountSyncCompleted` → D8

> **Nota de presentador:** D2 no almacena saldos permanentemente — los cachea con TTL corto desde los bancos vía D6. La BD está on-premise en Colombia, conectada vía Direct Connect (~15–25 ms mitigados por Redis).

---

### Slide 6: D3 — Gestión de Empresas y Empleados

**Título sugerido:** *"D3: Empresas y Empleados — Mínimo PII, máxima interoperabilidad"*

**Responsabilidad:** Registrar las 15 empresas aliadas y gestionar la referencia mínima de sus empleados. No almacena datos completos del empleado.

**Actores:** Empresa aliada, administrador del sistema

**Funciones clave:** Carga masiva de empresas (Spring Batch), gestión de referencia mínima de empleados, consulta de datos completos del empleado en tiempo real vía D6 → API de la empresa

**RNF, Funciones de ajuste y Umbrales:**

| RNF | Descripción | Función de ajuste | Umbral |
|-----|-------------|-------------------|--------|
| RNF-D3-01 | Interoperabilidad con APIs empresas | Adapter por empresa (D6), interfaz `EmployeeDataPort` | **P95 resolución < 1.5 s** |
| | | Circuit Breaker por empresa (Resilience4j) | **Fallo empresa A ≠ impacto empresa B** |
| RNF-D3-02 | Carga masiva idempotente | Spring Batch en chunks de 500 | **35K registros < 10 min** |
| | | Idempotencia (misma carga 2 veces) | **0 duplicados** |
| | | Aislamiento del sistema | **SLA no se degrada durante batch** |
| RNF-D3-03 | Minimización de datos PII | Test CI inspecciona esquema D3 | **0 columnas con PII** |
| | | Datos resueltos en tiempo real (vía D6) | **Nunca persistidos** |
| | | Cifrado de campos sensibles de empresa | **AES-256 (KMS)** |

**Diagrama interno (Figura 4):** Company API, Employee Ref API, Spring Batch, Aurora PostgreSQL (Company/EmployeeRef/PayrollSchedule), Outbox → Kafka.

**Comunicación clave:**
- Entrante síncrono: Admin (carga empresas vía API Gateway), D7 (empleados activos)
- Saliente síncrono: D2 (empresa → cuentas), D6 (resolve employee vía API empresa)
- Saliente asíncrono: `CompanyImported`, `EmployeeRefCreated/Updated` → D8

> **Nota de presentador:** Restricción de seguridad clave: D3 solo guarda un stub (`employee_ref_id`, `company_id`, `status`). Los datos completos del empleado se resuelven en memoria vía D6 al momento del pago y nunca se persisten.

---

### Slide 7: D4 — Transferencias y Transacciones

**Título sugerido:** *"D4: Transferencias — El núcleo financiero del sistema"*

**Responsabilidad:** Orquestar y registrar todas las transferencias de dinero: entre filiales (inmediata), no filiales e internacionales (diferida vía ACH). Validación antifraude en línea.

**Actores:** Persona natural, Billetera (D5), Empresa (vía D7)

**Funciones clave:** Transferencias P2P, interbancarias y múltiples destinos; dualidad inmediata/diferida; Saga coreografiada con compensación; validación contra listas B/G/N

**RNF, Funciones de ajuste y Umbrales:**

| RNF | Descripción | Función de ajuste | Umbral |
|-----|-------------|-------------------|--------|
| RNF-D4-01 | Consistencia transaccional | Saga coreografiada + Outbox Pattern | **0 fondos perdidos** |
| | | Saga completa (filial–filial) | **P95 < 3 s** |
| | | Reconciliación (0 huérfanas) | **Job periódico** |
| RNF-D4-02 | Liquidación inmediata vs diferida | Liquidación inmediata (filial–filial) | **P95 < 500 ms** |
| | | Reflejo tras callback ACH | **P95 < 2 s** |
| | | Clasificación de canal | **100% incluyen `settlement_type`** |
| RNF-D4-03 | Resiliencia ACH | Detección ACH no disponible (Circuit Breaker D6) | **< 5 s** |
| | | Aislamiento (caída ACH → filial no afectada) | **0 impacto inmediatas** |
| | | Reenvío idempotente | **0 duplicados en ACH** |
| RNF-D4-04 | Ciclo de estados ACH | Cobertura de estados (test unitario) | **100% casos documentados** |
| | | Procesamiento callback | **P95 < 2 s** |
| | | Fallo ACH revierte HOLD | **100% fallos revierten** |
| RNF-D4-05 | Antifraude en línea | Evaluación antifraude (Redis, TTL 60 s) | **P99 < 200 ms** |
| | | Propagación listas desde D8 | **< 60 s** |
| | | Falsos positivos | **< 0.5%** |
| RNF-D4-06 | Disponibilidad y rendimiento | Tiempo respuesta end-to-end | **P95 < 2 s** |
| | | Disponibilidad mensual | **≥ 99.9%** |
| | | Modo degradado bajo carga extrema | **< 5 s, 0 pérdida datos** |

**Diagrama interno (Figura 5):** Transfer API, FraudChecker, LiquidationRouter, Saga Coordinator, Transfer State Store (PostgreSQL on-premise vía DC).

**Comunicación clave:**
- Entrante síncrono: Cliente (POST /transfers vía API Gateway), D2 (valida cuentas/límites)
- Redis: listas B/G/N (TTL 60 s) + bancos filiales (TTL 5 min)
- Saliente asíncrono: `TransferSentToACH` → D6; eventos de estado → D8
- Entrante asíncrono: `ACHResponseReceived` desde D6, `FraudListUpdated` desde D8

> **Nota de presentador:** D4 es el dominio más complejo. Tiene 3 caminos: inmediato (filial-filial, < 500 ms), diferido (ACH, depende del externo) y rechazado (antifraude). La BD es on-premise en Colombia. El antifraude se evalúa en < 200 ms consultando listas en Redis.

---

### Slide 8: D5 — Billetera Digital

**Título sugerido:** *"D5: Billetera Digital — La cuenta propia de Empresa X"*

**Responsabilidad:** Administrar la cuenta propia emitida por Empresa X para cada usuario. Permite movimientos internos, pagos a terceros y transferencias a cuentas externas.

**Actores:** Persona natural

**Funciones clave:** Acreditar/debitar saldo, mover dinero a cuentas activas/externas/internacionales, pagos a terceros vía pasarelas de pago

**RNF, Funciones de ajuste y Umbrales:**

| RNF | Descripción | Función de ajuste | Umbral |
|-----|-------------|-------------------|--------|
| RNF-D5-01 | Atomicidad del ledger | Doble entrada (debit/credit emparejados, ACID) | **CHECK constraint** |
| | | Reconciliación `SUM(credit) - SUM(debit) = saldo` | **100% por wallet_id** |
| RNF-D5-02 | Compensación transaccional | Saga coreografiada (D6 falla → reverso automático) | **< 5 s desde fallo** |
| | | Dead Letter Queue para compensaciones fallidas | **Alerta automática** |
| RNF-D5-03 | Rendimiento consulta saldo | Prueba de carga concurrente | **P95 < 500 ms** |
| | | Billeteras con >100K movimientos | **Sin degradación** |
| RNF-D5-04 | Seguridad y control de acceso | Test RBAC (usuario A ≠ billetera B) | **0 accesos cruzados** |
| | | Cifrado en reposo (Aurora + KMS) | **100% cifrado** |
| | | Canal seguro | **100% TLS 1.3** |
| RNF-D5-05 | Trazabilidad movimientos | Cada operación genera evento | **100% trazadas** |
| | | Latencia evento al log D8 | **P95 < 500 ms** |
| | | Inmutabilidad del log | **0 modificaciones** |

**Diagrama interno (Figura 6):** Wallet API (NestJS), Wallet Service, Kafka Consumer (saga coreografiada), wallet_entries (Aurora PostgreSQL, append-only).

**Comunicación clave:**
- Entrante síncrono: API Gateway (usuario)
- Saliente síncrono: D2 (saldo combinado vía Istio mTLS)
- Entrante asíncrono: `PaymentGatewayResult/Failed` desde D6, `TransferACHResolved` desde D4
- Saliente asíncrono: `WalletDebited`, `WalletCredited`, `ThirdPartyPaymentInitiated`, `WalletCompensationTriggered` → D6 y D8

> **Nota de presentador:** La tabla `wallet_entries` es append-only — nunca se modifica. Funciona como un ledger con doble entrada contable. Si la pasarela falla después del débito, la saga coreografiada revierte automáticamente en < 5 s.

---

### Slide 9: D6 — Integraciones y Pasarelas de Pago

**Título sugerido:** *"D6: Integraciones — Plug-and-play sin downtime"*

**Responsabilidad:** Abstraer la comunicación con todos los sistemas externos: pasarelas de pago (PSE, DRUO, Apple Pay), ACH, terceros de servicios (servicios públicos, impuestos, transporte).

**Actores:** Billetera (D5), Transferencias (D4), Pagos Masivos (D7), sistemas externos

**Funciones clave:** Adapter por pasarela/tercero, registro dinámico de adaptadores (sin redespliegue), reintentos, timeouts, circuit breaker por adapter

**RNF, Funciones de ajuste y Umbrales:**

| RNF | Descripción | Función de ajuste | Umbral |
|-----|-------------|-------------------|--------|
| RNF-D6-01 | Aislamiento de adapters | Pod independiente por adapter (EKS) | **Fallo adapter A ≠ impacto adapter B** |
| | | Circuit Breaker por adapter (opossum, instancia independiente) | **Configuración independiente** |
| | | Bulkhead pattern (thread pool separado) | **Sin saturación cruzada** |
| RNF-D6-02 | Nuevos terceros sin downtime | Adapter Registry dinámico (BD, sin reinicio) | **0 downtime existentes** |
| | | Nuevo adapter = contenedor independiente | **Sin tocar núcleo D6** |
| | | Smoke test post-deploy | **100% adapters healthy** |
| RNF-D6-03 | Seguridad comunicación externa | Canal cifrado | **100% TLS 1.3** |
| | | Rotación de credenciales (Secrets Manager) | **Cada 90 días** |
| | | Validación callbacks (HMAC) | **100% validados** |
| | | No exposición de credenciales | **0 fugas** |
| RNF-D6-04 | Trazabilidad integraciones | Cada llamada genera `ExternalCallLog` | **100% trazadas** |
| | | Latencia registro en D8 | **P95 < 500 ms** |
| | | Correlación end-to-end | **100% con `correlation_id`** |

**Diagrama interno (Figura 7):** Adapter Registry (registro dinámico), Circuit Breaker (instancia por adapter), PSE/DRUO/Apple Pay/ACH Adapters, Adapter Tercero (plug-and-play).

**Comunicación clave:**
- Entrante asíncrono: `ThirdPartyPaymentInitiated` (D5), `MassivePaymentDispatched` (D7), `TransferSentToACH` (D4)
- Saliente externo: HTTPS/TLS 1.3 a PSE, DRUO, Apple Pay, ACH, terceros
- Saliente asíncrono: `PaymentGatewayResult`, `PaymentGatewayFailed`, `ACHResponseReceived` → D5, D4, D8

> **Nota de presentador:** Requisito explícito del enunciado: "integrar terceros a demanda sin afectar el estado del sistema". D6 lo logra con el patrón Adapter + registro dinámico. Agregar un nuevo tercero = desplegar un contenedor nuevo y registrarlo en la BD sin reiniciar nada.

---

### Slide 10: D7 — Pagos Masivos a Empleados

**Título sugerido:** *"D7: Pagos Masivos — 30K transacciones en ventanas críticas"*

**Responsabilidad:** Gestionar la nómina de las 15 empresas aliadas: pagos individuales, masivos programados/manuales, trazabilidad por empleado.

**Actores:** Empresa aliada, administrador de nómina

**Funciones clave:** Programar lotes de pago, consultar empleados vía API empresa (en tiempo de ejecución), ejecutar 20K–30K transacciones en ventanas de alta demanda, trazabilidad individual por empleado

**RNF, Funciones de ajuste y Umbrales:**

| RNF | Descripción | Función de ajuste | Umbral |
|-----|-------------|-------------------|--------|
| RNF-D7-01 | Escalabilidad bajo picos | Tiempo total 30K pagos | **< 15 minutos** |
| | | SLA general durante pico | **P95 < 2 s** |
| | | Escalado automático (HPA) | **Nuevos pods < 60 s** |
| | | Pérdida de mensajes (Kafka) | **0 perdidos** |
| RNF-D7-02 | Independencia de pagos | Pagos duplicados (idempotency key) | **0 duplicados** |
| | | Pagos perdidos | **0 perdidos** |
| | | Compensación por pago individual | **< 5 s** |
| | | Transacciones huérfanas | **0 en producción** |
| RNF-D7-03 | Trazabilidad por lote/empleado | Cobertura eventos | **100% generan evento** |
| | | Latencia auditoría | **< 500 ms** |
| | | Consulta de lote completo | **< 2 s** |
| RNF-D7-04 | Automatización nómina | Ejecución única (scheduler + lock distribuido) | **100% sin duplicados** |
| | | Precisión horaria | **Desviación < 1 min** |
| | | Tolerancia a reinicio | **No duplicar ejecución** |
| RNF-D7-05 | Aislamiento por empresa | Impacto cruzado | **0 entre empresas** |
| | | Detección API caída (circuit breaker) | **< 5 s** |
| | | Recuperación automática (bulkhead) | **Sin intervención manual** |

**Diagrama interno (Figura 8):** Payroll API (Spring Boot), Payroll Scheduler (ShedLock + Aurora), Batch Manager, Worker Pool (Kafka consumers particionados por company_id, HPA), Saga Handler, Payroll DB (Aurora PostgreSQL).

**Comunicación clave:**
- Entrante síncrono: Empresa (portal empresarial → API Gateway)
- Saliente síncrono: D3 (empleados activos vía Istio mTLS)
- Saliente asíncrono: `PayrollPaymentInitiated` → D4 (ejecuta transferencia)
- Entrante asíncrono: `TransferSettled/Failed` desde D4
- Saliente asíncrono: `PayrollBatchCreated`, `PayrollPaymentSucceeded/Failed`, `PayrollBatchCompleted` → D8

> **Nota de presentador:** Los workers están particionados por `company_id` en Kafka, así que la nómina de cada empresa se procesa de forma independiente. ShedLock sobre Aurora garantiza que el scheduler programado se ejecute una sola vez, incluso con múltiples réplicas.

---

### Slide 11: D8 — Reportes, Auditoría y Cumplimiento

**Título sugerido:** *"D8: Auditoría — Observador pasivo y guardián regulatorio"*

**Responsabilidad:** Mantener el histórico completo e inmutable de todas las transacciones. Generar reportes regulatorios (extracto trimestral a bancos, reporte semestral a Superintendencia). Detección de patrones sospechosos y retroalimentación de listas antifraude.

**Actores:** Sistema (scheduler), bancos filiales, Superintendencia Financiera

**Funciones clave:** Event Ingester (append-only + hash chain), Report Generator (PDF/CSV/XML), Fraud Stream Processor (Flink + CEP), Fraud List Manager, Dashboard API

**RNF, Funciones de ajuste y Umbrales:**

| RNF | Descripción | Función de ajuste | Umbral |
|-----|-------------|-------------------|--------|
| RNF-D8-01 | Inmutabilidad del audit log | Test UPDATE/DELETE → rechazado | **0 modificaciones** |
| | | Hash chain verificada periódicamente | **100% integridad** |
| | | Firmas digitales válidas | **100% válidas** |
| | | Cobertura de eventos | **100% persistidos** |
| | | Deduplicación por event_id | **0 duplicados** |
| RNF-D8-02 | Detección patrones sospechosos | Latencia detección (Flink CEP) | **P95 < 5 s** |
| | | Cobertura reglas CEP | **100% activas** |
| | | Falsos positivos | **< 5%** |
| | | Propagación listas a D4 (Redis) | **< 60 s** |
| RNF-D8-03 | Reportes regulatorios en plazo | Generación trimestral (scheduler) | **100% trimestres** |
| | | Generación semestral | **100% semestres** |
| | | Envío exitoso a bancos y Superfinanciera | **100% en plazo** |
| | | Formato correcto | **0 errores** |
| RNF-D8-04 | Observabilidad del sistema | Búsqueda por correlation_id (OpenSearch) | **P95 < 2 s** |
| | | Latencia indexación | **P95 < 30 s** |
| | | Disponibilidad dashboard | **99.9% uptime** |
| | | Alertas ante umbrales críticos | **< 60 s** |
| RNF-D8-05 | Retención y cifrado | Registros > 5 años consultables | **100% retenidos** |
| | | Cifrado en reposo (Keyspaces + OpenSearch + KMS) | **100% cifrados** |
| | | ILM: hot → warm → cold | **Nunca eliminar < 5 años** |

**Diagrama interno (Figura 9):** Desplegado en modelo híbrido:
- **On-premise (Colombia):** Event Ingester (Java), Audit Store Writer (append-only → Cassandra 4.1), Report Scheduler, Report Generator (Python 3.12)
- **AWS:** Fraud Stream Processor (Managed Flink), Fraud List Manager (EKS Fargate), Dashboard API (NestJS, EKS Fargate)
- **Almacenamiento:** Cassandra on-premise, S3 (reportes cifrados), OpenSearch Multi-AZ, Redis (listas B/G/N)

**Comunicación clave:**
- Entrante asíncrono: Todos los eventos de D1–D7 (observador pasivo universal)
- Saliente asíncrono: `SuspiciousPatternDetected`, `FraudListUpdated` → D1 (bloqueo) y D4 (invalidación caché)
- Saliente batch: Reportes vía D6 → Bancos (HTTPS/SFTP trimestral) y Superintendencia (HTTPS/SFTP semestral)
- Saliente síncrono: Dashboard API para consultas operacionales

> **Nota de presentador:** D8 es un dominio híbrido. Lo que es inmutable (audit log + reportes) vive on-premise en Colombia con Cassandra y hash chains. Lo que necesita procesamiento en tiempo real (detección de fraude con Flink, dashboards) vive en AWS. Los dos se conectan vía Direct Connect.

---

## BLOQUE 3 — Transversales y cierre

---

### Slide 12: Elementos transversales al sistema

**Título sugerido:** *"Cross-cutting concerns — Lo que toca a todos los dominios"*

Estos elementos no pertenecen a un solo dominio, sino que son capacidades de plataforma que impactan a todos:

| Concern transversal | Qué implica | Dónde se implementa | RNF vinculado |
|---------------------|-------------|----------------------|---------------|
| **Seguridad** | TLS 1.3 en todos los canales, WAF + OWASP Top 10, MFA para roles críticos, cifrado AES-256 en reposo, firmado digital | API Gateway (WAF, JWT), D1 (MFA, RBAC), Istio (mTLS inter-servicio), KMS/Vault (cifrado) | RNF-04 |
| **Disponibilidad 24/7** | Auto-healing, replicación, failover automático, Multi-AZ | K8s (liveness/readiness probes), EKS Multi-AZ, Aurora Multi-AZ, Patroni on-premise, Resilience4j (circuit breakers) | RNF-01 |
| **Rendimiento < 2 s** | Caché, CDN, procesamiento asíncrono, escalado horizontal | ElastiCache Redis (TTL por caso de uso), CloudFront (activos estáticos), Kafka (async), HPA (pods), EC2 node groups (picos) | RNF-02 |
| **Escalabilidad** | Escalar horizontalmente los servicios bajo demanda, absorber picos predecibles | HPA en EKS, particionamiento Kafka, Queue-based load leveling (D7), Aurora Serverless v2 | RNF-03 |
| **Multiplataforma** | Web + móvil + tablet (≥ 6") con UX consistente | Flutter 3 (un codebase, compilación nativa), design system unificado, responsive CSS | RNF-08 |
| **Observabilidad** | Trazas, métricas, logs centralizados, alertas automáticas | OpenTelemetry Collector, X-Ray, CloudWatch, Managed Grafana, OpenSearch | RNF-09 |
| **Detección de intrusiones** | Análisis de amenazas a nivel de red, API y contenedor | GuardDuty, Security Hub, WAF, Falco (EKS + K8s on-premise) | RNF-04 |

**Diagrama Mermaid — Vista transversal:**


> **Nota de presentador:** Este diagrama muestra que la seguridad no es solo un dominio (D1) sino una capa envolvente. Lo mismo la observabilidad, la disponibilidad y el rendimiento. Son concerns transversales implementados como infraestructura compartida, no como responsabilidad de un solo microservicio.

---

### Slide 13: RNF globales — Tabla resumen

**Título sugerido:** *"10 RNF globales y sus umbrales clave"*

| # | RNF | Umbral principal | Mecanismo clave |
|---|-----|------------------|-----------------|
| RNF-01 | **Disponibilidad** | ≥ 99.9% uptime | Replicación activa-activa, circuit breakers, K8s self-healing |
| RNF-02 | **Rendimiento** | P95 < 2 s | Redis cache, Kafka async, CDN, HPA |
| RNF-03 | **Escalabilidad** | 25M usuarios + 30K tx pico | HPA, Queue-based load leveling, particionamiento Kafka |
| RNF-04 | **Seguridad** | OWASP Top 10, 100% TLS 1.3 | OAuth2+JWT+MFA, WAF, cifrado AES-256, IDS |
| RNF-05 | **Extensibilidad** | Nuevos terceros sin downtime | Plugin/Adapter (D6), feature flags, registro dinámico |
| RNF-06 | **Trazabilidad** | 100% eventos registrados | Event Sourcing, append-only, hash chain, reportes regulatorios |
| RNF-07 | **Fiabilidad** | 0 fondos perdidos | Saga + compensación, idempotencia, exactly-once Kafka |
| RNF-08 | **Multiplataforma** | Web + móvil + tablet ≥ 6" | Flutter 3, responsive, design system unificado |
| RNF-09 | **Observabilidad** (H) | Alertas < 60 s | OpenTelemetry, Grafana, X-Ray, CloudWatch |
| RNF-10 | **Mantenibilidad** (H) | Dominios independientes | DDD, Database per Service, contract testing, ADRs |

> **(H)** = Hipótesis razonable (no explícita en el enunciado, pero justificada por el contexto)

---

### Slide 14: Estrategias de evolución (opcional)

**Título sugerido:** *"9 estrategias para que la arquitectura evolucione sin romperse"*

| # | Estrategia | Qué permite | Dónde aplica |
|---|------------|-------------|--------------|
| E1 | Bounded Contexts (DDD) | Modificar/reemplazar un dominio sin afectar otros | D1–D8 |
| E2 | Versionado de APIs | Cambios incompatibles sin romper consumidores | API Gateway, D6, D4 |
| E3 | Consumer-Driven Contracts (Pact) | Detectar rupturas de contrato antes de producción | D4↔D6, D5↔D6, D7↔D6 |
| E4 | Feature Flags | Activar funcionalidades sin redespliegue | D6, D7, D5 |
| E5 | CI/CD por microservicio | Despliegue independiente por dominio | D1–D8 + GitOps/ArgoCD |
| E6 | Infrastructure as Code (Terraform) | Entornos reproducibles y auditables | Infra completa |
| E7 | ADRs | Preservar razonamiento de decisiones | Repositorio central |
| E8 | Observabilidad (OpenTelemetry) | Identificar problemas antes de producción | D1–D8 + Kafka + GW |
| E9 | Strangler Fig | Reemplazar dominios gradualmente sin big-bang | D2, D6 (futuro) |

> **Nota de presentador:** La evolución de la arquitectura no es solo un diseño, son prácticas concretas. Pact garantiza que al cambiar D6, D4 no se rompa. Feature flags permiten integrar un nuevo tercero en producción desactivado y activarlo con un toggle. ADRs documentan por qué se tomó cada decisión para que el equipo futuro no la revierta sin entender el contexto.

---

### Slide 15: Cierre

**Título sugerido:** *"Resumen ejecutivo"*

**Puntos de cierre:**

1. **Problema:** Transferencias interbancarias lentas en Colombia → plataforma centralizada multiplataforma
2. **Arquitectura:** Microservicios orientados a eventos, 8 bounded contexts, Kafka como columna vertebral
3. **RNF:** 10 globales + 38 específicos por dominio, todos con funciones de ajuste y umbrales medibles
4. **Infraestructura:** Modelo híbrido (on-premise Colombia + AWS) por cumplimiento normativo
5. **Evolución:** 9 estrategias concretas (DDD, versionado, contracts, feature flags, CI/CD, IaC, ADRs, observabilidad, Strangler Fig)

---

> **Fin del guion de presentación**
