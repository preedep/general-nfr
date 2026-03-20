# Non-Functional Requirements (NFR)

> **Purpose:** This document defines Non-Functional Requirements for application development. It is intended to be used by GitLab AI during code review in the CI/CD pipeline to verify that implementations comply with organizational standards.

---

## Table of Contents

1. [Application Log](#1-application-log)
2. [PII Log](#2-pii-log)
3. [Application Security Log (SOC Log)](#3-application-security-log-soc-log)
4. [Cloud Agnostic Design – Adapter Layer](#4-cloud-agnostic-design--adapter-layer)
5. [Security Requirements](#5-security-requirements)
6. [Kubernetes Requirements](#6-kubernetes-requirements)

---

## 1. Application Log

### Overview

Application Log is designed for **investigating and troubleshooting system issues**. Logs are shipped to the **Centralized ELK** (shared infrastructure) and are accessible through **Kibana**.

---

### 1.1 Log Structure

Each log entry MUST be a structured JSON object conforming to the schema below.

#### Required Fields

| Field | Type | Description |
|---|---|---|
| `event_date_time` | `DateTime<Utc>` | Timestamp in ISO 8601 format (UTC) |
| `log_type` | `Enum` | Categorizes the log entry (see Log Types below) |
| `level` | `Enum` | Severity level: `debug`, `info`, `warn`, `error` |

#### Optional Fields

**Application Metadata**

| Field | Type | Description |
|---|---|---|
| `app_id` | `string` | Application identifier |
| `app_version` | `string` | Application release version |
| `app_address` | `string` | IP address and port of the application |
| `geo_location` | `string` | Azure region or data center location |

**Service Information**

| Field | Type | Description |
|---|---|---|
| `service_id` | `string` | Service name or identifier |
| `service_version` | `string` | Container image tag |
| `service_pod_name` | `string` | Kubernetes pod identifier |
| `code_location` | `string` | Module or function path |

**Tracing & Correlation**

| Field | Type | Description |
|---|---|---|
| `correlation_id` | `UUID` | End-to-end tracing identifier |
| `request_id` | `UUID` | Per HTTP request identifier (must match linked request/response pairs) |
| `trace_id` | `string` | Distributed tracing identifier (for APM link) |
| `span_id` | `string` | Distributed tracing span identifier |

**Caller Information**

| Field | Type | Description |
|---|---|---|
| `caller_channel_name` | `string` | Channel invoking the request |
| `caller_user` | `string` | User identifier |
| `caller_address` | `string` | IP address of the caller |

**Operational Data**

| Field | Type | Description |
|---|---|---|
| `execution_time` | `int` | Milliseconds elapsed |
| `message` | `string` | Descriptive text |
| `request` | `object` | Request details object |
| `response` | `object` | Response details object |

---

### 1.2 Log Types

| Log Type | Description |
|---|---|
| `APP_LOG` | General application events |
| `REQ_LOG` | Incoming requests |
| `RES_LOG` | Outgoing responses |
| `REQ_EX_LOG` | Calls to external services |
| `RES_EX_LOG` | Responses from external services |

> **Note:** `PII_LOG` is **excluded** from Application Log scope and is handled separately under [Section 2](#2-pii-log).

---

### 1.3 Requirements

- **[MUST]** All log entries MUST include the three required fields: `event_date_time`, `log_type`, and `level`.
- **[MUST]** The `event_date_time` field MUST be in ISO 8601 UTC format.
- **[MUST]** The `log_type` field MUST be one of the defined enum values: `APP_LOG`, `REQ_LOG`, `RES_LOG`, `REQ_EX_LOG`, `RES_EX_LOG`.
- **[MUST]** The `level` field MUST be one of: `debug`, `info`, `warn`, `error`.
- **[MUST NOT]** Log entries MUST NOT include customer PII data or sensitive financial data in application log fields.
- **[SHOULD]** The `correlation_id` and `request_id` fields SHOULD be included to support end-to-end tracing.
- **[SHOULD NOT]** Applications SHOULD NOT add unnecessary attributes beyond what is required for troubleshooting. Excessive fields increase log volume and consume storage on the shared ELK infrastructure.
- **[SHOULD]** The `execution_time` field SHOULD be included for performance-sensitive operations.

---

## 2. PII Log

### Overview

PII Log is a **regulatory log** mandated by Thailand's **Personal Data Protection Act (PDPA)**. It records all access and manipulation of customer personal data by internal staff through organization-owned web applications.

PII Log uses a **pipe-delimited (`|`) plain text format** (legacy format, different from Application Log JSON format).

PII Log is **not** routinely accessed. It is retrieved only when an incident investigation is required.

---

### 2.1 Scope – When to Log

PII Log MUST be recorded whenever an internal staff member accesses customer personal data through an organization-owned web application, including the following operations:

| Operation | Must Log |
|---|---|
| Search / Query customer PII data | ✅ Yes |
| Create / Add new customer PII data | ✅ Yes |
| Update / Edit customer PII data | ✅ Yes |
| Delete customer PII data | ✅ Yes |
| Export customer PII data | ✅ Yes |

---

### 2.2 Examples of PII Data

The following categories are considered personal data under PDPA:

- Full name, middle name
- National ID, Social Security number, Passport number, Driver's License number, Tax ID, Bank account number, Credit card number
- Address, Email address, Phone number, Mobile phone number
- Electronic identifiers: IP address, Device ID, MAC address, Cookie ID
- Financial data: account numbers
- Property information: vehicle registration, land title
- Date of birth, place of birth, weight, height, geolocation
- Biometric data: face recognition, fingerprint, iris recognition, voice recognition
- Health, genetic, and medical history information
- Race, religion, belief, political opinion, sexual behaviour, criminal record
- Occupation and education information
- Internal organization keys that identify a person (e.g., RM ID)

---

### 2.3 Log Format

PII Log uses a **pipe-delimited (`|`) format**. Fields must appear in a fixed order on a single line.

**Formatting Rules:**
- Each field is separated by `|`.
- If a field value itself contains `|`, it MUST be replaced with `{PIPE}` or an equivalent encoded value.
- Newlines within a field MUST be represented as `\n` (not a literal line break).

**Example format:**

```
{date_time}|{log_type}|{app_id}|{app_name}|{caller_user}|{caller_address}|{operation}|{object_type}|{object_id}|{result}|{detail}
```

---

### 2.4 Data Retention

| Storage Tier | Retention Period | Notes |
|---|---|---|
| Application-local storage | 90 days | Complies with Computer Crime Act B.E. 2550 |
| Centralized PII Log System | Up to 2 years (active + archived) | Logs shipped daily to centralized system |

**Storage Pattern:**

- **Pattern 1 (Self-managed):** Application stores PII logs locally with a minimum retention of 2 years (active + archived). Suitable for high-criticality systems.
- **Pattern 2 (Centralized):** Application keeps logs locally for **90 days**, then ships logs **daily** to the Centralized PII Log System (retention up to 2 years at the centralized level).

---

### 2.5 Requirements

- **[MUST]** Applications MUST implement PII logging for all access and manipulation of customer personal data by internal staff.
- **[MUST]** PII logs MUST use the pipe-delimited format as specified in Section 2.3.
- **[MUST NOT]** Field values in PII logs MUST NOT contain literal `|` characters. Replace with `{PIPE}` or equivalent encoding.
- **[MUST NOT]** Literal line breaks MUST NOT appear within a single PII log entry. Use `\n` instead.
- **[MUST]** Local retention MUST be maintained for a minimum of **90 days** on the application side.
- **[MUST]** PII logs MUST NOT be sent to the Centralized ELK (Application Log infrastructure). PII logs must be routed to the designated PII log storage only.
- **[SHOULD]** Applications SHOULD ship PII logs daily to the Centralized PII Log System for long-term retention.

---

## 3. Application Security Log (SOC Log)

### Overview

Application Security Log — also known internally as **SOC Log** — is a security-focused audit log required by the IT Security Requirement (ITSR) standard. It is used by the Security Operations Center (SOC) team for threat monitoring and incident investigation via the **SIEM** (Security Information and Event Management) system.

---

### 3.1 Log Format

Application Security Log MUST use a **pipe-delimited (`|`) format**. Fields MUST appear in the following fixed order. If a field does not have a value, leave it blank (do not omit the delimiter).

| Position | Field | Required | Description |
|---|---|---|---|
| 1 | `date_time` | **Mandatory** | Format: `dd/MM/yyyy hh:mm:ss` |
| 2 | `log_type` | **Mandatory** | Fixed value: `Application_Security_Log` |
| 3 | `app_id` | **Mandatory** | Application ID (e.g., `APxxxx`) |
| 4 | `app_name` | **Mandatory** | Application name |
| 5 | `event_type` | **Mandatory** | Type of security event (e.g., `Login`, `Logout`, `AccessDenied`) |
| 6 | `source_address` | **Mandatory** | Source IP address |
| 7 | `source_hostname` | Optional | Source hostname |
| 8 | `source_user_id` | **Mandatory** | Source user ID or username |
| 9 | `source_object` | Optional | Source object involved |
| 10 | `destination_address` | **Mandatory** | Destination IP address |
| 11 | `destination_hostname` | Optional | Destination hostname |
| 12 | `destination_user_id` | **Mandatory** | Destination user ID or username |
| 13 | `destination_object` | Optional | Destination object involved |
| 14 | `result` | **Mandatory** | Action flag or Success/Failure flag |
| 15 | `message` | Optional | Additional description |
| 16 | `extra_fields` | Optional | Any other fields to improve monitoring |

**Example:**

```
16/06/2024 14:30:00|Application_Security_Log|AP1234|MyApplication|Login|10.1.2.3|host.corp.internal|user123||10.0.0.1|||admin||Success|User logged in successfully|
```

---

### 3.2 Security Events to Log

Applications MUST log, at minimum, the following security-relevant events:

- User authentication events (login, logout, failed login)
- Authorization failures and access denied events
- Privilege escalation or role changes
- Critical data access and modification events
- System configuration changes
- Application exceptions and errors relevant to security

---

### 3.3 Retention Requirements

| Availability | Minimum Duration |
|---|---|
| Readily accessible (online) | 90 days |
| Total retention (online + archived) | 12 months |

---

### 3.4 Requirements

- **[MUST]** Applications MUST produce Application Security Logs in the pipe-delimited format defined in Section 3.1.
- **[MUST]** All 8 mandatory fields MUST be present in every log entry.
- **[MUST]** The `date_time` field MUST use the format `dd/MM/yyyy hh:mm:ss`.
- **[MUST]** The `log_type` field MUST be the fixed string `Application_Security_Log`.
- **[MUST]** Application security logs MUST be sent to the SIEM for monitoring.
- **[MUST NOT]** Application security logs MUST NOT contain sensitive financial information or customer PII. Such data must be excluded before the log is forwarded to the SIEM.
- **[MUST]** Audit logs MUST be retained for a minimum of **12 months** in total, with at least **90 days** of immediately accessible storage.
- **[MUST]** Applications MUST synchronize time using NTP:
  - On-premises: use the organization's authorized NTP servers (e.g., `time1.corp.internal`, `time2.corp.internal`)
  - Cloud environments: use the NTP service provided by the respective cloud provider (AWS, Azure, etc.)
- **[MUST]** Security logs MUST survive application restarts (logs must be persisted to durable storage, not held in memory only).
- **[SHOULD]** The Legal Notice / Terms of Use page displayed to users MUST include the following five elements at minimum:
  1. The user is utilizing systems owned by the organization.
  2. Systems are designated for authorized use only, in adherence to organizational policies.
  3. Systems may undergo monitoring to prevent improper use.
  4. Usage of the system implies consent to monitoring, in compliance with local laws.
  5. Unauthorized use may lead to reprimand, dismissal, financial penalties, and/or legal action.

---

## 4. Cloud Agnostic Design – Adapter Layer

### Overview

When an application integrates with cloud services such as **Vault (Secret Management)** or **Object Storage**, it MUST use an **Adapter Layer** to decouple application code from cloud-vendor-specific SDKs and APIs. This reduces **cloud vendor lock-in** and enables portability across cloud providers (AWS, Azure, etc.) with minimal code changes.

> **Cloud Agnostic Design** means designing the system with loose coupling to any single cloud provider by using standard interfaces or adapter patterns — not by prohibiting the use of managed services.

---

### 4.1 Adapter Options

Three adapter patterns are available. Applications SHOULD choose based on their context:

| Option | Pattern | Description | Recommended For |
|---|---|---|---|
| **Option 1** | App-Level Adapter (Self-develop) | Application team develops a thin adapter layer that maps standard internal interface calls to cloud-specific SDK/API calls. Business logic does not call cloud SDK directly. | Individual applications; best for domain-specific integration logic |
| **Option 2** | Platform-Level Abstraction (K8s CRDs / CSI) | Use Kubernetes-native abstractions such as External Secrets Operator (ESO) or CSI Driver to abstract cloud-specific secret and storage access at the platform level. | COTs/packaged software that cannot be modified at the code level |
| **Option 3** | Framework-Level Adapter (Sidecar / Shared Utility) | Use a sidecar container or shared utility component as an intermediary. This is a fallback scenario and requires additional infrastructure components. | Fallback only; application team is responsible for provisioning the component |

---

### 4.2 Vault / Secret Management

When connecting to cloud-based Vault or Secret Management services, the following decision guideline applies:

| Access Scenario | Recommended Adapter |
|---|---|
| Access secrets as environment variables | ESO (External Secrets Operator) |
| Access secrets as mounted files (e.g., certificate files, keystore files, `/mnt/secrets/keystore.jks`) | CSI Driver (Container Storage Interface) |

> **Technical note:** When using ESO with a large number of secrets, Kubernetes API load may increase significantly. Monitor accordingly.

---

### 4.3 Object Storage

When connecting to cloud-based Object Storage, the following decision guideline applies:

| Scenario | Recommended Adapter |
|---|---|
| Dedicated cluster with access controlled via service endpoint | NFS (CSI Driver) |
| Shared cluster requiring full POSIX file interface (e.g., file locking, atomic rename) | SMB (CSI Driver) |
| Shared cluster — general use, POSIX compatibility must be verified by application team | Object Storage Fuse Driver (e.g., BlobFuse, AWS Mountpoint) |

> **Cost note:** Object Storage Fuse Driver bills per operation (read/write via REST API). Applications must assess read/write frequency carefully before using this option.

---

### 4.4 Requirements

- **[MUST]** Applications MUST NOT call cloud vendor SDKs (e.g., Azure SDK, AWS SDK) directly from business logic code. All cloud service integrations MUST go through an adapter layer.
- **[MUST]** The adapter interface MUST be defined as a standard internal interface. Changing the cloud provider underneath MUST NOT require changes to business logic code.
- **[MUST]** When integrating with Vault/Secret Management services, applications MUST use one of the approved adapter patterns (ESO or CSI Driver), not direct SDK calls.
- **[MUST]** When integrating with Object Storage, applications MUST use one of the approved adapter patterns (App-Level Adapter, CSI NFS/SMB, or Fuse Driver based on the decision guideline in Section 4.3).
- **[MUST NOT]** Configuration values and secrets MUST NOT be hard-coded in application source code. Secrets MUST be injected at runtime via the approved adapter pattern.
- **[SHOULD]** Applications SHOULD use open standards and open-source technologies (e.g., OpenTelemetry, OIDC, Kafka protocol, open-source databases) over proprietary cloud-native equivalents where feasible.
- **[SHOULD NOT]** Applications SHOULD NOT use cloud-provider-proprietary tools or frameworks (e.g., Databricks `dbutils`, Azure-specific Functions, ADF-specific workflows) as the primary integration method if a portable alternative exists.
- **[SHOULD]** New mandatory applications (as defined by the Cloud Agnostic Mandatory Selection Guideline) MUST target the **Cloud Agnostic** level, meaning the adapter layer is in place and business logic is fully decoupled from cloud-vendor-specific integration code.

---

## 5. Security Requirements

### Overview

This section defines security requirements derived from the IT Security Requirement (ITSR) standard that are directly relevant to application development and coding. These requirements apply to all applications and must be validated during code review and security testing.

> **Reference standard:** OWASP Application Security Verification Standard (ASVS)

---

### 5.1 Authentication & Session Management

#### Requirements

- **[MUST]** Applications MUST implement an authentication mechanism for all users accessing the application. Centralized authentication via an Identity Provider (IdP) is strongly preferred.
- **[MUST]** For internet-facing services, brute-force protection MUST be implemented (e.g., CAPTCHA, request rate limiting, login delay, or account lockout after repeated failed attempts).
- **[MUST]** Multi-Factor Authentication (MFA) MUST be implemented for:
  - Internet-facing systems and/or systems accessing customer data
  - Applications within PCI DSS scope
  - Cloud Services (SaaS, PaaS, IaaS)
- **[MUST]** Applications MUST support configuring automated log-off for inactive user sessions (target: within 15 minutes where feasible).
- **[MUST]** Applications MUST support limiting the number of concurrent active sessions per user (target: 1 session per user where feasible).
- **[MUST]** For applications processing financial transactions, concurrent logins with the same user ID MUST NOT be allowed. Any pre-existing session MUST be terminated upon a new successful login.
- **[MUST]** Session IDs MUST be unpredictable and MUST be terminated upon user logout. A new session ID MUST be generated each time the user logs in.
- **[MUST NOT]** Applications MUST NOT be vulnerable to session hijacking or other session management flaws.
- **[MUST]** For system-to-system communication, machine authentication MUST be implemented (e.g., mutual TLS, API keys via secret management, service accounts).

---

### 5.2 Authorization

#### Requirements

- **[MUST]** Applications MUST implement Role-Based Access Control (RBAC).
- **[MUST]** Access rights MUST be granted on the principle of **least privilege** — each role MUST have only the permissions necessary to perform its defined responsibilities.
- **[MUST NOT]** A single account MUST NOT combine the Highest Privilege role and the User Management role. These functions MUST be separated.
- **[MUST]** Highest-privilege accounts MUST be stored and managed securely using a Privileged Access Management (PAM) tool or equivalent.

---

### 5.3 Data Protection & Masking

#### Requirements

- **[MUST]** When displaying sensitive identifiers (e.g., National/Citizen ID), masking MUST be applied. At minimum, only the last few digits should be visible (e.g., `xxxxxxxxx2345`). Masking MUST be performed **server-side** before sending to the client.
- **[MUST]** When displaying financial account numbers (e.g., bank account number), masking MUST be applied. At minimum, only the last few digits should be visible (e.g., `xxxxx12345`). Masking MUST be performed **server-side** before sending to the client.
- **[MUST]** Pages or API responses containing sensitive information MUST disable client-side caching (e.g., via `Cache-Control: no-store` headers).
- **[MUST]** Sensitive data that is no longer needed MUST be removed from memory and storage promptly.
- **[MUST NOT]** Sensitive production data (customer PII, financial data) MUST NOT be used in non-production environments (development, SIT, UAT) without appropriate compensating controls (e.g., data masking or anonymization).

---

### 5.4 Cryptography & Key Management

#### Requirements

**Password & Credential Storage**

- **[MUST]** Passwords stored by the system MUST be protected using an approved password hashing function with a randomly generated per-user salt (minimum 32 bytes of entropy):
  - SHA-2 or SHA-3 with minimum 256 bits
  - Bcrypt (minimum work factor = 10)
  - PBKDF2 (iterations ≥ 310,000)
  - Argon2id (m ≥ 15 MiB)

**Encryption Algorithms**

- **[MUST]** Applications MUST use only approved encryption algorithms:
  - Symmetric: AES ≥ 256-bit with GCM or CCM mode
  - Asymmetric: RSA ≥ 2048-bit, or ECDSA ≥ 256-bit
  - Key Exchange: ECDHE or DHE ≥ 2048-bit
- **[MUST]** Approved hashing functions: SHA-2 or SHA-3 with ≥ 256 bits.
- **[MUST NOT]** Weak or deprecated algorithms (MD5, SHA-1, DES, 3DES, RC4) MUST NOT be used.

**Transport Layer Security**

- **[MUST]** All data in transit MUST be encrypted using TLS 1.2 with forward secrecy at minimum.
- **[SHOULD]** TLS 1.3 with forward secrecy is recommended.

**Randomness**

- **[MUST]** Cryptographically secure random number generators MUST be used for session keys, salts, initialization vectors (IVs), and other security-sensitive values (e.g., `/dev/urandom`, `SecureRandom` in Java, `secrets` module in Python).
- **[MUST NOT]** `Math.random()`, `rand()`, or other non-cryptographic random generators MUST NOT be used for security-sensitive purposes.

**Secrets & Key Storage**

- **[MUST]** All credentials, secrets, API keys, certificates, and cryptographic keys MUST be stored in a secret management system (e.g., HashiCorp Vault, cloud KMS) — never in plaintext files or source code.
- **[MUST NOT]** Default credentials or keys MUST NOT be used. Credentials MUST differ between development and production environments.
- **[MUST]** Encryption keys at rest MUST themselves be encrypted. Key-Encrypting Keys (KEK) MUST be stored separately from Data-Encrypting Keys (DEK).
- **[MUST]** Encryption keys MUST be rotated at least annually (unless governed by a regulatory body with different rotation requirements).
- **[SHOULD]** Highly sensitive or regulated data MUST use customer-managed keys (CMK/BYOK) rather than provider-managed keys.

---

### 5.5 Secure Coding Practices

#### Requirements

- **[MUST]** Applications MUST be developed following OWASP guidelines, covering at minimum:
  - OWASP Top 10 Web Application Security Risks
  - OWASP API Security Top 10
  - OWASP Top 10 CI/CD Security Risks
- **[MUST NOT]** Software components that are end-of-life, unsupported, or have known unpatched vulnerabilities MUST NOT be used in production.
- **[MUST]** Application interfaces MUST exchange only the **minimum required data** with other systems (data minimization principle).
- **[MUST]** Configuration files MUST NOT contain IP addresses, URLs, or configuration values from other environments (e.g., DEV, SIT, UAT) in the production environment.
- **[MUST NOT]** Secrets, credentials, API keys, or environment-specific configuration MUST NOT be hard-coded in source code or committed to version control repositories.
- **[MUST]** Applications MUST be protected against "Time of Check to Time of Use" (TOCTOU) race conditions for all security-sensitive operations (e.g., permission checks before resource access).

---

### 5.6 Input Validation & File Handling

#### Requirements

- **[MUST]** All files received over the network MUST be scanned and validated before processing, regardless of the trustworthiness of the source. Validation MUST include:
  - File content validation (detect malicious or inappropriate data)
  - File extension validation performed **after** decoding the filename (to prevent bypass via encoding tricks)
  - Filename validation to reject special or prohibited characters
- **[MUST]** Uploaded files MUST be stored outside the web server's document root (to prevent direct execution via URL).
- **[MUST]** All user-supplied input MUST be validated, sanitized, and/or parameterized before use to prevent injection attacks (SQL injection, command injection, XSS, etc.).

---

### 5.7 Error Handling & Information Disclosure

#### Requirements

- **[MUST]** Applications MUST return generic, non-descriptive error messages to clients for both expected and unexpected error conditions. Error messages MUST NOT reveal internal system details.

  Example of acceptable messages:
  - `An unexpected error occurred. Please try again later or contact support if the issue persists.`
  - `Authentication failed. Please check your credentials and try again.`

- **[MUST NOT]** Stack traces, exception details, internal paths, database queries, server versions, or other diagnostic information MUST NOT be exposed to end users or in API responses.
- **[MUST]** Exception details and diagnostic information MUST be written only to **internal logs** (not returned in HTTP responses or rendered in UI).

---

## 6. Kubernetes Requirements

### Overview

Applications deployed on Kubernetes MUST be configured to operate reliably and safely within a shared cluster environment. This section covers resource management, health probing, and graceful shutdown — all of which directly affect cluster stability, workload scheduling, and zero-downtime deployments.

---

### 6.1 Resource Requests and Limits

Every container MUST declare both `resources.requests` and `resources.limits` in its pod spec. Without these, the scheduler cannot make informed placement decisions and a single runaway container can starve other workloads on the same node.

- `requests` — the amount of CPU/memory the scheduler reserves for the container. Used for node placement.
- `limits` — the hard ceiling the container is never allowed to exceed. Breaching the memory limit causes the container to be OOM-killed; breaching the CPU limit causes throttling.

#### Requirements

- **[MUST]** Every container MUST declare `resources.requests.cpu` and `resources.requests.memory`.
- **[MUST]** Every container MUST declare `resources.limits.cpu` and `resources.limits.memory`.
- **[MUST NOT]** CPU and memory limits MUST NOT be left unset. Containers without limits can consume unbounded resources and destabilize the node.
- **[SHOULD]** Set `requests` as close as possible to the actual observed usage. Over-requesting wastes cluster capacity; under-requesting risks OOM kills.
- **[SHOULD]** Keep the ratio of `limits` to `requests` reasonable (recommended: limits ≤ 2× requests for memory to avoid burstable-tier instability).

#### Example

```yaml
resources:
  requests:
    cpu: "250m"       # 0.25 vCPU reserved for scheduling
    memory: "256Mi"   # 256 MiB reserved for scheduling
  limits:
    cpu: "500m"       # hard cap at 0.5 vCPU (throttled if exceeded)
    memory: "512Mi"   # hard cap at 512 MiB (OOM-killed if exceeded)
```

> **Note on units:** CPU is expressed in millicores (`m`); `1000m` = 1 vCPU. Memory uses binary suffixes: `Mi` (mebibytes), `Gi` (gibibytes).

---

### 6.2 Liveness Probe and Readiness Probe

Kubernetes uses probes to determine the health of a container. Without them, the platform cannot detect a hung process, route traffic safely, or perform rolling updates without causing errors.

| Probe | Purpose | Action on failure |
|---|---|---|
| **Liveness** | Detects if the container is stuck / deadlocked and needs a restart | Container is restarted by kubelet |
| **Readiness** | Detects if the container is ready to accept traffic | Pod is removed from Service endpoints (no traffic sent) |

#### Requirements

- **[MUST]** Every container MUST define a `livenessProbe`.
- **[MUST]** Every container that serves traffic MUST define a `readinessProbe`.
- **[MUST]** The liveness probe endpoint MUST reflect whether the process is alive and not deadlocked. It MUST NOT call downstream dependencies — a dependency failure should not restart the container.
- **[MUST]** The readiness probe endpoint MUST reflect whether the container is fully initialised and ready to handle requests (e.g., DB connections established, caches warmed).
- **[SHOULD]** Set `initialDelaySeconds` to give the application enough time to start before the first probe fires, avoiding unnecessary restarts during startup.
- **[SHOULD]** Prefer an HTTP `GET` probe over a TCP check when the application exposes an HTTP endpoint, as it more accurately validates application-level health.

#### Example

```yaml
livenessProbe:
  httpGet:
    path: /healthz/live    # lightweight check — is the process alive?
    port: 8080
  initialDelaySeconds: 15  # wait 15 s after container start before first check
  periodSeconds: 20         # check every 20 s
  failureThreshold: 3       # restart after 3 consecutive failures

readinessProbe:
  httpGet:
    path: /healthz/ready   # deeper check — is the app ready for traffic?
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 10
  failureThreshold: 3
```

> **Recommended practice:** Expose two separate endpoints — `/healthz/live` (liveness) and `/healthz/ready` (readiness) — so each can be independently lightweight or thorough as needed.

---

### 6.3 Graceful Shutdown

When Kubernetes terminates a pod (during a rolling update, node drain, or scale-down), it sends `SIGTERM` to the container and waits for `terminationGracePeriodSeconds` before force-killing it with `SIGKILL`. Applications that do not handle `SIGTERM` drop in-flight requests and leave resources in an inconsistent state.

#### Requirements

- **[MUST]** Applications MUST handle the `SIGTERM` signal and begin an orderly shutdown sequence:
  1. Stop accepting new requests (deregister from load balancer / Service endpoints via readiness probe returning unhealthy).
  2. Finish processing all in-flight requests.
  3. Release resources (close DB connections, flush buffers, complete pending writes).
  4. Exit cleanly with a zero exit code.
- **[MUST]** The `terminationGracePeriodSeconds` value in the pod spec MUST be set long enough to allow all in-flight requests to complete. The default (30 s) is often insufficient for long-running jobs.
- **[MUST NOT]** Applications MUST NOT exit immediately on `SIGTERM` without draining in-flight work.
- **[SHOULD]** Background workers and batch jobs MUST checkpoint their progress before shutdown so that work can be resumed, not restarted from scratch, after a restart.

#### Example

```yaml
spec:
  terminationGracePeriodSeconds: 60   # give the app up to 60 s to shut down cleanly

  containers:
    - name: my-app
      lifecycle:
        preStop:
          exec:
            # Optional: sleep briefly to allow the Service endpoint removal
            # to propagate before the app stops accepting connections
            command: ["/bin/sh", "-c", "sleep 5"]
```

Application-side (example in Go):

```go
quit := make(chan os.Signal, 1)
signal.Notify(quit, syscall.SIGTERM, syscall.SIGINT)
<-quit                          // block until signal received

ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
defer cancel()
server.Shutdown(ctx)            // stop accepting new requests, drain existing ones
```

---

*Document version: 1.2 | Last updated: 2026-03-20*
