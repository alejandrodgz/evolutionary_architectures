# Trabajo Final — Arquitecturas Evolutivas
## Definición de una arquitectura evolutiva de alto nivel
**Universidad EAFIT · 2026-1 · Profesor: Carlos Orozco**

---

## 1. Descripción de la arquitectura seleccionada

### 1.1 Justificación de la arquitectura

Para el sistema de la Empresa X se seleccionó una **arquitectura de microservicios orientada a eventos** (*Event-Driven Microservices*). Esta decisión responde directamente a los cuatro factores dominantes del problema:

| Factor | Evidencia en el contexto | Respuesta arquitectónica |
|--------|--------------------------|--------------------------|
| **Escala extrema** | ~25 millones de usuarios activos | Escalado horizontal independiente por servicio |
| **Picos predecibles de carga** | 20K–30K transacciones los días 14–16 y 29–31 de cada mes | Servicios de pagos masivos escalan sin afectar los demás |
| **Extensibilidad a demanda** | Integrar terceros sin afectar el estado del sistema (requisito explícito) | Patrón Adapter por canal externo; nuevos terceros se incorporan sin redespliegue |
| **Disponibilidad 24/7 y < 2 s** | Operación continua + SLA de tiempo de respuesta | Desacoplamiento asíncrono; fallos en ACH no bloquean transferencias entre filiales |

La arquitectura se apoya en cuatro patrones complementarios:

1. **Capa de Presentación + BFF (Backend for Frontend)** — clientes multiplataforma (web SPA, app móvil híbrida, tablet ≥ 6 pulgadas) que se comunican con el sistema exclusivamente a través del API Gateway. El patrón BFF adapta las respuestas del backend al formato y necesidades de cada tipo de cliente sin exponer la lógica interna de los microservicios.
2. **API Gateway** — punto de entrada único que centraliza autenticación, enrutamiento, rate-limiting y TLS hacia todos los microservicios.
3. **Message Broker / Event Streaming (Kafka)** — desacopla productores y consumidores; garantiza entrega, orden y replay de eventos críticos (transferencias, pagos masivos, sincronizaciones).
4. **Bounded Contexts (DDD)** — cada dominio de negocio se implementa como un microservicio con responsabilidad única y base de datos propia (*Database per Service*), lo que permite evolucionar, versionar o reemplazar un dominio sin impacto en los demás.

**Arquitectura descartada — Monolito modular:** No soporta el requisito de extensibilidad a demanda ni el escalado diferenciado de los módulos de pagos masivos frente a consultas de saldo. Ante picos de nómina, se escalaría innecesariamente todo el sistema.

> **Citas:** [POR DEFINIR — agregar referencias del material del curso: título, autor, unidad]

---

### 1.2 Dominios identificados (Bounded Contexts)

El sistema se divide en **8 dominios** derivados directamente del enunciado. Cada dominio es implementado como uno o más microservicios cohesionados.

---

#### Dominio 1 — Identidad y Acceso (IAM)
**Responsabilidad:** Gestionar la autenticación, autorización y seguridad transversal de todos los actores del sistema (personas naturales, personas jurídicas, operadores internos, sistemas externos).

| Aspecto | Detalle |
|---------|---------|
| **Actores** | Persona natural, persona jurídica, empresa, sistema externo (bancos, ACH, terceros) |
| **Funciones clave** | Login multifactor, gestión de sesiones, emisión y validación de tokens (JWT/OAuth2), control de acceso basado en roles (RBAC), políticas OWASP, detección y prevención de intrusiones (IDS/IPS), firmado digital y cifrado en reposo de datos sensibles, gestión de canales seguros inter-servicio (mTLS) |
| **Eventos que produce** | `UserAuthenticated`, `SessionRevoked`, `UnauthorizedAccessAttempt`, `IntrusionDetected` |
| **RNF dominante** | Seguridad, Disponibilidad |
| **Observaciones** | Centraliza todas las políticas de seguridad exigidas por restricciones estatales: autenticación, autorización, acceso restringido y cifrado en todo el ciclo de vida de la información (creación, manipulación y persistencia). Proporciona el token que todos los demás dominios validan vía API Gateway. El módulo IDS detecta patrones de acceso anómalos y emite eventos que D8 registra en el audit log. Los datos de listas negras/grises/blancas generados por D8 retroalimentan a este dominio para bloqueo preventivo. |

---

#### Dominio 2 — Gestión de Usuarios, Cuentas y Consultas
**Responsabilidad:** Registrar y mantener actualizada la información de personas naturales y jurídicas con sus cuentas bancarias en bancos filiales. Expone un modelo de lectura optimizado (CQRS) para que los usuarios consulten el estado actualizado de sus cuentas.

| Aspecto | Detalle |
|---------|---------|
| **Actores** | Bancos filiales (fuente de datos), administrador del sistema, persona natural, persona jurídica (consulta de estado de cuentas) |
| **Funciones clave** | Carga masiva desde bancos, sincronización diaria de información de clientes, exposición del estado consolidado de cuentas registradas mediante modelo de lectura separado (CQRS — Query Side) |
| **Eventos que produce** | `UserRegistered`, `AccountLinked`, `AccountSyncCompleted` |
| **Eventos que consume** | `TransferCompletedImmediately`, `TransferACHResolved`, `WalletCredited`, `WalletDebited` |
| **RNF dominante** | Fiabilidad, Consistencia eventual |
| **Observaciones** | El registro de usuarios NO es autoservicio; es un proceso ETL programado. La sincronización diaria debe ser idempotente para tolerar reintentos. El modelo de lectura (Query Side) se mantiene actualizado a partir de los eventos de sincronización y transacción, y permite a personas naturales y jurídicas consultar el estado consolidado de sus cuentas en filiales sin impactar el proceso ETL de escritura. |

---

#### Dominio 3 — Gestión de Empresas y Empleados
**Responsabilidad:** Registrar las 15 empresas aliadas y gestionar la información básica que permite identificar a sus empleados en cada nómina.

| Aspecto | Detalle |
|---------|---------|
| **Actores** | Empresa aliada, administrador del sistema |
| **Funciones clave** | Carga masiva de empresas, gestión de referencia mínima de empleados (sin PII completa), consulta de empleados activos |
| **Eventos que produce** | `CompanyRegistered`, `EmployeeReferenceUpdated` |
| **RNF dominante** | Seguridad (mínimo PII almacenado), Fiabilidad |
| **Observaciones** | La plataforma solo guarda la referencia mínima del empleado; los datos completos se consultan en tiempo real al API de cada empresa al momento del pago. |

---

#### Dominio 4 — Transferencias y Transacciones
**Responsabilidad:** Orquestar y registrar todas las transferencias de dinero entre cuentas, incluyendo la integración con ACH para operaciones fuera del ecosistema filial.

| Aspecto | Detalle |
|---------|---------|
| **Actores** | Persona natural (cuenta particular), billetera |
| **Funciones clave** | Transferencia inmediata entre bancos filiales (reflejo instantáneo en estado de cuenta), envío a ACH para bancos no filiales e internacionales, registro del movimiento tras resolución de ACH |
| **Eventos que produce** | `TransferInitiated`, `TransferCompletedImmediately`, `TransferSentToACH`, `TransferACHResolved` |
| **Eventos que consume** | `ACHResponseReceived` |
| **RNF dominante** | Consistencia (ACID en filiales), Disponibilidad, Tiempo de respuesta < 2 s |
| **Observaciones** | Patrón **Saga** para coordinar pasos distribuidos (débito → crédito → confirmación). Compensación automática ante fallo parcial. Los eventos `TransferCompleted*` son consumidos por D2 para actualizar el modelo de lectura de cuentas y por D8 para el histórico inmutable. |

---

#### Dominio 5 — Billetera Digital
**Responsabilidad:** Administrar la cuenta propia emitida por la Empresa X para cada usuario, habilitando movimientos internos, pagos a terceros y pagos a cuentas externas.

| Aspecto | Detalle |
|---------|---------|
| **Actores** | Persona natural |
| **Funciones clave** | Acreditar/debitar saldo de la billetera, mover dinero a cuentas activas del usuario, realizar pagos a terceros mediante pasarelas de pago |
| **Eventos que produce** | `WalletCredited`, `WalletDebited`, `ThirdPartyPaymentInitiated` |
| **Eventos que consume** | `PaymentGatewayResult`, `TransferACHResolved` |
| **RNF dominante** | Consistencia, Seguridad, Tiempo de respuesta |
| **Observaciones** | Aplican las mismas reglas de transferencia que en una cuenta particular (dominio 4). Los eventos `WalletCredited` y `WalletDebited` son consumidos por D2 para mantener el estado de cuenta actualizado y por D8 para el histórico. El pago a terceros se delega al dominio 6. |

---

#### Dominio 6 — Integración con Terceros y Pasarelas de Pago
**Responsabilidad:** Abstraer la comunicación con sistemas externos: pasarelas de pago (PSE, DRUO, Apple Pay) y terceros de servicios (servicios públicos, impuestos, transporte, etc.).

| Aspecto | Detalle |
|---------|---------|
| **Actores** | Billetera, Pagos Masivos, sistemas externos |
| **Funciones clave** | Adaptar el protocolo de cada pasarela o tercero, integrar nuevos canales a demanda (patrón Plugin/Adapter), gestionar reintentos y timeouts con sistemas externos, establecer canales de comunicación seguros con cada tercero (TLS/HTTPS con validación de certificados) |
| **Eventos que produce** | `PaymentGatewayResult`, `ThirdPartyServiceResponse` |
| **Eventos que consume** | `ThirdPartyPaymentInitiated`, `MassivePaymentDispatched` |
| **RNF dominante** | Extensibilidad, Disponibilidad, Resiliencia, Seguridad de canales externos |
| **Observaciones** | **Requisito clave del enunciado:** integrar terceros sin afectar el estado del sistema. Se implementa con un registro de adaptadores dinámico; añadir un nuevo tercero = desplegar un nuevo adapter sin cambiar el núcleo. Los canales con cada sistema externo se establecen sobre HTTPS/TLS con validación mutua de certificados, cumpliendo el requisito de canales de comunicación seguros con terceros. |

---

#### Dominio 7 — Pagos Masivos a Empleados
**Responsabilidad:** Gestionar el proceso de nómina de las empresas aliadas: pagos individuales, pagos masivos programados/manuales y trazabilidad por empleado.

| Aspecto | Detalle |
|---------|---------|
| **Actores** | Empresa aliada, administrador de nómina |
| **Funciones clave** | Programar lotes de pago (manual o automático), consultar información del empleado vía API de la empresa en el momento del pago, ejecutar 20K–30K transacciones en ventanas de alta demanda, mantener trazabilidad individual por empleado |
| **Eventos que produce** | `PayrollJobScheduled`, `MassivePaymentDispatched`, `EmployeePaymentCompleted`, `PayrollJobFinished` |
| **Eventos que consume** | `PaymentGatewayResult` |
| **RNF dominante** | Escalabilidad, Fiabilidad, Trazabilidad |
| **Observaciones** | Procesa lotes mediante colas de mensajes con workers escalables. La información del empleado se consulta al API de la empresa en tiempo de ejecución (no se almacena PII). Soporta pagos programados (calendarizados para los días 14–16 y 29–31) y manuales, tanto individuales como masivos. Patrón **Batch + Queue-based load leveling**. |

---

#### Dominio 8 — Reportes, Auditoría y Cumplimiento Regulatorio
**Responsabilidad:** Mantener el histórico completo e inmutable de transacciones, registrar toda la actividad del sistema (audit log) y generar los reportes regulatorios exigidos (extracto trimestral a bancos, reporte semestral a la Superintendencia Financiera).

| Aspecto | Detalle |
|---------|---------|
| **Actores** | Sistema (scheduler), bancos filiales, Superintendencia Financiera |
| **Funciones clave** | Almacenar histórico inmutable de transacciones, generar y enviar extracto trimestral por usuario/banco, generar reporte semestral a la Superfinanciera, monitoreo de transacciones sospechosas (listas blancas/grises/negras), monitoreo y registro de actividad del sistema (audit log de accesos, operaciones y modificaciones de datos) |
| **Eventos que consume** | Todos los eventos de transacción de D4, D5 y D7; eventos de seguridad de D1 (`IntrusionDetected`, `UnauthorizedAccessAttempt`) |
| **RNF dominante** | Integridad, Trazabilidad, Seguridad, Cumplimiento normativo |
| **Observaciones** | Usa un modelo **Event Sourcing** / append-only para el histórico. El monitoreo de fraude se implementa como un stream processor (Kafka Streams / Flink) sobre los eventos en tiempo real. El audit log centraliza todos los registros de actividad exigidos por restricciones estatales (accesos, modificaciones, operaciones sensibles). Los datos de listas negras/grises/blancas retroalimentan al dominio D1 para bloqueo preventivo. |

---

### 1.3 Mapa de dominios (resumen)

```
┌──────────────────────────────────────────────────────────────────┐
│                  Capa de Presentación (BFF)                      │
│       [Web SPA]     [App Móvil Híbrida]     [Tablet ≥ 6"]       │
└──────────────────────────────┬───────────────────────────────────┘
                               │ HTTPS / WSS
┌──────────────────────────────▼───────────────────────────────────┐
│                         API Gateway                              │
│          (enrutamiento, autenticación, TLS, rate-limiting)       │
└────────────┬─────────────────────────────┬────────────────────────┘
             │                             │
    ┌────────▼──────────┐      ┌───────────▼────────┐
    │  D1: IAM          │      │  D8: Reportes /    │
    │ Identidad/Acceso  │      │  Auditoría /       │
    │ IDS · mTLS        │      │  Audit Log         │
    └────────┬──────────┘      └───────────┬────────┘
             │                             │  (consume eventos)
     ┌───────▼────────────────────┐        │
     │       Message Broker       │◄───────┘
     │          (Kafka)           │
     └───┬───┬────┬───┬───┬───┬──┘
         │   │    │   │   │   │
  ┌──────▼─┐ │ ┌──▼───▼─┐ │ ┌─▼────────────┐
  │ D2:    │ │ │  D4:   │ │ │ D6:          │
  │Usuarios│ │ │ Trans. │ │ │Terceros/     │
  │Cuentas │ │ │  y Tx  │ │ │Pasarelas     │
  │(CQRS)  │ │ └────────┘ │ └──────────────┘
  └────────┘ │         ┌──▼───────┐
         ┌───▼──┐      │  D5:     │
         │ D3:  │      │Billetera │
         │Empr. │      │Digital   │
         │Empl. │      └──────────┘
         └──────┘
                    ┌──────────────┐
                    │ D7: Pagos    │
                    │  Masivos     │
                    └──────────────┘
```

---

### Pendientes Sección 1

- [ ] Agregar citas del material del curso (títulos, autores, unidad) — marcado como [POR DEFINIR]
- [ ] Revisar si el nombre de la empresa tiene nombre real o se mantiene como "Empresa X"
- [ ] Confirmar si el diagrama de la sección 1 es suficiente o se quiere un C4 Level 1 aquí también

---

*Sección 2 (RNF y Funciones de ajuste) y Sección 3 (Diagrama C4) — pendientes*

---

## 4. Stack tecnológico (Tabla 2)

La siguiente tabla mapea cada capa del sistema a su tecnología recomendada. Las decisiones se tomaron considerando: escala (25M usuarios), contexto financiero regulado, operación 24/7 y la necesidad de extensibilidad a demanda.

### 4.1 Presentación y acceso

| Capa / Componente | Tecnología recomendada | Alternativa | Justificación y trade-offs | RNF vinculado |
|---|---|---|---|---|
| **Web SPA + BFF** | React 18 + Next.js 14 | Angular 17 | Next.js habilita el patrón BFF con SSR/API Routes en el mismo proyecto; ecosistema maduro; hidratación progresiva reduce tiempo de carga inicial en dispositivos de gama media. Angular es más opinionado, útil si el equipo ya lo domina. | Usabilidad, Tiempo de respuesta < 2 s |
| **App Móvil / Tablet** | Flutter 3 (Dart) | React Native | Un solo codebase para iOS, Android y tablet ≥ 6"; compilación nativa (no WebView), rendimiento estable en gama media-alta; motor Skia garantiza consistencia visual. React Native tiene mayor ecosistema JS pero mayor overhead de puente nativo. | Usabilidad, Compatibilidad multiplataforma |
| **API Gateway** | Kong Gateway (OSS) | AWS API Gateway | Plugin ecosystem maduro: auth JWT, rate-limiting, logging, transformación de peticiones. Self-hosted evita vendor lock-in y facilita cumplimiento de soberanía de datos. AWS API Gateway es más simple de operar pero menos flexible para plugins personalizados. | Seguridad, Disponibilidad, Escalabilidad |

---

### 4.2 Mensajería y comunicación asíncrona

| Capa / Componente | Tecnología recomendada | Alternativa | Justificación y trade-offs | RNF vinculado |
|---|---|---|---|---|
| **Message Broker** | Apache Kafka (Confluent Platform) | Amazon MSK | Throughput > 1 M msg/s, durabilidad configurable por topic, capacidad de replay para Event Sourcing (D8) y reintento de pagos masivos (D7). Confluent añade Schema Registry y monitoreo. MSK simplifica operaciones pero limita configuración avanzada. | Escalabilidad, Fiabilidad, Consistencia eventual |
| **Service Mesh (mTLS inter-servicio)** | Istio 1.x | Linkerd 2 | Cifrado mTLS automático entre todos los microservicios sin cambios en el código; circuit breaker, retry policies y observabilidad de tráfico integrados. Linkerd es más liviano pero con menor superficie de configuración avanzada. | Seguridad (canales inter-servicio), Disponibilidad |

---

### 4.3 Microservicios — Runtime

| Capa / Componente | Tecnología recomendada | Alternativa | Justificación y trade-offs | RNF vinculado |
|---|---|---|---|---|
| **Dominios financieros — D4, D7** | Java 21 + Spring Boot 3 + Spring Cloud | Kotlin + Spring Boot | Soporte ACID nativo con Spring Data JPA; librerías Saga maduras (Eventuate Tram, Axon Framework); ecosistema fintech amplio; GraalVM Native reduce tiempo de arranque. Kotlin reduce verbosidad pero con curva de adopción. | Consistencia, Fiabilidad, Trazabilidad |
| **Dominios API / integración — D1, D2, D5, D6** | Node.js 20 + NestJS | Go (Golang) | Alta concurrencia I/O para servicios orientados a API; TypeScript garantiza tipado estricto; inyección de dependencias y arquitectura modular de NestJS facilita testing. Go ofrece menor latencia pero ecosistema de librerías financieras más reducido. | Tiempo de respuesta < 2 s, Extensibilidad |
| **Stream processing — Fraude (D8)** | Apache Flink | Kafka Streams | Procesamiento stateful complejo sobre ventanas de tiempo (detección de patrones sospechosos); mayor expresividad que Kafka Streams para CEP (Complex Event Processing); escalado independiente del broker. Kafka Streams es más simple si los patrones son básicos. | Seguridad, Trazabilidad en tiempo real |

---

### 4.4 Almacenamiento

| Capa / Componente | Tecnología recomendada | Alternativa | Justificación y trade-offs | RNF vinculado |
|---|---|---|---|---|
| **BD relacional — D1, D2, D3, D4, D5, D7** | PostgreSQL 16 | MySQL 8 | ACID estricto, row-level security, JSONB para datos semiestructurados, particionado nativo para tablas de alto volumen. Amplio soporte en fintech colombiana. MySQL es alternativa viable pero con menor soporte para features avanzados de seguridad y particionado. | Consistencia, Integridad, Seguridad |
| **Caché / CQRS Query Side — D2** | Redis 7 (cluster mode) | Amazon ElastiCache | Lecturas sub-milisegundo para consultas de estado de cuenta; pub/sub para invalidación de caché ante eventos de transacción; soporta los 25 M de usuarios con escalado horizontal de shards. ElastiCache simplifica operaciones en AWS pero genera dependencia de proveedor. | Tiempo de respuesta < 2 s, Escalabilidad |
| **Event Sourcing / histórico immutable — D8** | Apache Cassandra 4 | ClickHouse | Escrituras append-only con altísima throughput; escalado lineal horizontal sin punto único de fallo; TTL nativo para gestión de retención regulatoria. ClickHouse es superior para consultas analíticas ad-hoc pero con mayor complejidad operacional para escrituras masivas continuas. | Integridad, Trazabilidad, Escalabilidad |
| **Búsqueda y reportes — D8** | Elasticsearch 8 | OpenSearch | Indexación full-text sobre el audit log y el histórico de transacciones; dashboards Kibana para cumplimiento regulatorio; integración nativa con ELK. OpenSearch (fork AWS) es alternativa open-source sin cambios de licencia. | Trazabilidad, Cumplimiento normativo |

---

### 4.5 Seguridad e identidad

| Capa / Componente | Tecnología recomendada | Alternativa | Justificación y trade-offs | RNF vinculado |
|---|---|---|---|---|
| **Identity Provider — D1** | Keycloak 24 | Auth0 | Open source; MFA, RBAC, OAuth2/OIDC y SSO out-of-the-box; self-hosted garantiza soberanía de datos (requisito regulatorio colombiano); ampliamente adoptado en sector público y financiero de LATAM. Auth0 reduce carga operacional pero implica dependencia de tercero para datos de autenticación. | Seguridad, Cumplimiento normativo |
| **Gestión de secretos y cifrado** | HashiCorp Vault | AWS Secrets Manager | Dynamic secrets con rotación automática; cifrado como servicio (Transit Secrets Engine) para datos en reposo; PKI integrada para gestión de certificados mTLS. Vault cubre el requisito de cifrado en todo el ciclo de vida. AWS Secrets Manager es más simple pero con menor control sobre políticas de cifrado personalizadas. | Seguridad (cifrado en reposo, firmado) |
| **Detección de intrusiones (IDS)** | Falco + Wazuh | AWS GuardDuty | Falco detecta comportamiento anómalo a nivel de contenedor/kernel en tiempo real; Wazuh agrega SIEM con correlación de eventos de seguridad del audit log. Combinación cubre detección en runtime y análisis forense. GuardDuty cubre solo el perímetro cloud sin visibilidad interna de contenedores. | Seguridad (IDS/IPS), Trazabilidad |

---

### 4.6 Infraestructura y orquestación

| Capa / Componente | Tecnología recomendada | Alternativa | Justificación y trade-offs | RNF vinculado |
|---|---|---|---|---|
| **Orquestación de contenedores** | Kubernetes 1.29 + Docker | Amazon EKS (managed) | Auto-scaling horizontal (HPA) para absorber picos de nómina; rolling deployments sin downtime; self-healing de pods ante fallos. EKS reduce carga operacional del plano de control pero introduce dependencia de AWS para actualizaciones del cluster. | Disponibilidad 24/7, Escalabilidad |
| **Infraestructura como Código** | Terraform | Pulumi | Declarativo, multi-cloud, state management con backends remotos; ampliamente adoptado con ecosistema de módulos financieros. Habilita entornos reproducibles para auditorías de cumplimiento. Pulumi permite IaC en lenguajes de propósito general pero con menor comunidad de módulos listos. | Evolución del sistema, Cumplimiento |

---

### 4.7 Observabilidad

| Capa / Componente | Tecnología recomendada | Alternativa | Justificación y trade-offs | RNF vinculado |
|---|---|---|---|---|
| **Métricas y alertas** | Prometheus + Grafana | Datadog | Stack open source nativo en K8s; dashboards operacionales y alertas sobre SLOs (latencia < 2 s, disponibilidad 24/7). Datadog simplifica la operación pero con costo variable por volumen de datos a escala de 25 M usuarios. | Disponibilidad, Tiempo de respuesta |
| **Logs centralizados / Audit Log** | ELK Stack (Elasticsearch + Logstash + Kibana) | Grafana Loki + Promtail | Búsqueda full-text sobre logs del audit log de D8; dashboards Kibana para auditorías regulatorias; retención configurable por política. Loki tiene menor costo de almacenamiento pero sin indexación full-text nativa (limitante para auditorías). | Trazabilidad, Cumplimiento normativo |
| **Trazas distribuidas** | Jaeger | Zipkin | Trazabilidad end-to-end de peticiones a través de todos los microservicios; identifica cuellos de botella en el SLA de < 2 s; integración con Istio e instrumentación OpenTelemetry. Zipkin es más liviano pero con menor soporte para sampling adaptativo. | Tiempo de respuesta < 2 s, Disponibilidad |

---

### 4.8 CI/CD

| Capa / Componente | Tecnología recomendada | Alternativa | Justificación y trade-offs | RNF vinculado |
|---|---|---|---|---|
| **Pipelines CI/CD** | GitLab CI/CD | GitHub Actions | Pipelines declarativos YAML con runners self-hosted; integración nativa con registro de contenedores, K8s y Terraform; soporte para entornos regulados con aprobaciones manuales por etapa. GitHub Actions tiene mayor ecosistema de actions de comunidad pero runners self-hosted tienen menos madurez en ambientes financieros. | Evolución del sistema, Disponibilidad |

---

### Pendientes Sección 4

- [ ] Confirmar proveedor de nube principal (AWS, GCP o Azure) para alinear alternativas gestionadas
- [ ] Definir política de retención de datos en Cassandra/Elasticsearch (exigencia regulatoria Superfinanciera)
- [ ] Validar si Keycloak self-hosted cumple con los lineamientos de la entidad reguladora o si se requiere proveedor certificado

---

*Próxima sección: **Sección 2 — RNF y Funciones de ajuste (Tabla 1)** · Sección 3 — Diagrama C4 (Figura 1)***
