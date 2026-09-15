# ADR-002: Zero-Trust Agent Identity & WIMSE

## Context

Гибридная оркестрация (ADR-001) предполагает, что LLM-агенты
выполняют семантическую работу внутри BPMN-шагов. При этом агенты
должны иметь доступ к корпоративным ресурсам:

- Вызов внутренних API (1С, ERP, MDM, ЕИС).
- Чтение чувствительных данных (документы, справочники).
- Формирование рекомендаций для критичных решений (суммы, договоры).

Традиционные методы идентификации **не подходят** для агентов:

1. **API keys** — статичны, не ротируются, утекают в логи и репозитории.
   Нет привязки к конкретному экземпляру агента.

2. **Service accounts** — долгоживущие, shared между сервисами,
   невозможно отследить, какой именно агент сделал вызов.

3. **OAuth client credentials** — работают для сервисов, но не дают
   криптографической идентичности. Нет автоматической ротации.

4. **mTLS без SPIFFE** — даёт шифрование, но не решает проблему
   identity (кто именно этот workload?).

Дополнительные требования enterprise:

- **Аудит каждого вызова**: какой агент, какой SVID, какой контекст.
- **Zero Trust**: не доверять никому, даже внутри сети.
- **Short-lived credentials**: автоматическая ротация.
- **Fine-grained authorization**: не «доступ к API», а «доступ
  к конкретному ресурсу с конкретными атрибутами».
- **Attestation**: подтверждение, что агент запущен в доверенной среде.

Эти требования описаны в разделе 4.2 Enterprise Agent Orchestration
Blueprint (WIMSE-совместимая идентичность).

## Decision

Принимаем **WIMSE-совместимую архитектуру идентичности** на базе
**SPIFFE/SPIRE**:

### 1. SPIFFE ID для каждого агента

Каждый агент получает уникальный SPIFFE ID:

    spiffe://company.ru/agents/procurement_agent_v1
    spiffe://company.ru/agents/pricing_agent_v1
    spiffe://company.ru/agents/compliance_agent_v1

Формат: `spiffe://<trust-domain>/<workload-path>`.

Trust domain (`company.ru`) — это корпоративный домен доверия,
управляемый через SPIRE Server.

### 2. SVID (SPIFFE Verifiable Identity Document)

Каждый агент получает **short-lived X.509 SVID**:

- **Срок жизни**: 1 час (автоматическая ротация).
- **Подпись**: внутренний CA (SPIRE Server).
- **Содержимое**: SPIFFE ID, публичный ключ, срок действия.
- **Приватный ключ**: хранится в SPIRE Agent на ноде.

При каждом вызове агент предъявляет SVID через **mTLS**.

### 3. mTLS между агентом и Policy Enforcement Point

Все вызовы от агента к PEP (Policy Enforcement Point) происходят
по **mutual TLS**:

- **Агент** → предъявляет SVID.
- **PEP** → проверяет SVID через SPIRE Server.
- **Обе стороны** → получают криптографическую гарантию identity.

Открытый HTTP/API-key **запрещён**.

### 4. Authorization: RBAC + ABAC

SVID даёт **аутентификацию** (кто ты?), но **не авторизацию**
(что тебе можно?). Авторизация реализуется в два уровня:

**RBAC (Role-Based Access Control):**
- SVID агента → роль (Sourcing, Pricing, Compliance).
- Роль → список разрешённых действий (calculate_nmc, verify_price).
- Хранится в Policy Registry.

**ABAC (Attribute-Based Access Control):**
- Каждое действие проверяется по атрибутам:
  - `amount <= 50_000_000` (лимит суммы).
  - `methodology in ['method_1', 'method_2']` (разрешённые методики).
  - `region in allowed_regions` (география).
- Реализуется через OPA/Rego или Cedar.

### 5. Policy Enforcement Point (PEP)

Агент **не вызывает** бизнес-функции напрямую. Он формирует
`RecommendationDTO`, который проходит через **PEP** в BPMN-слое:

    1. Проверка SVID (mTLS) → кто агент?
    2. Проверка RBAC → разрешена ли роль?
    3. Проверка ABAC → разрешено ли действие с этими атрибутами?
    4. Проверка confidence threshold → ≥ 0.85 для автономного вызова?
    5. Audit log → запись с SVID, reasoning, decision.
    6. Если всё ОК → вызов целевого воркера.
       Если нет → BPMN Error → эскалация на человека.

Это реализация принципа наименьших привилегий (Least Privilege).

### 6. Attestation агентов

SPIRE Agent на ноде проверяет, что workload **действительно**
является нашим агентом:

- Проверка hash бинарника.
- Проверка Kubernetes namespace/service account.
- Проверка подписи контейнера.
- Проверка среды (production / staging).

Без attestation SVID не выдаётся.

### 7. Audit log с SVID

Каждое действие агента логируется в append-only лог:

    {
      "timestamp": "2026-09-15T10:00:00Z",
      "agent_svid": "spiffe://company.ru/agents/procurement_agent_v1",
      "action": "calculate_nmc",
      "input": { "lot_id": "12345", "methodology": "method_1" },
      "output": { "nmc_value": 1500000, "confidence": 0.95 },
      "decision": "approved",
      "user_context": { "user_id": "ivanov", "role": "specialist" },
      "hash": "sha256:...",
      "previous_hash": "sha256:..."
    }

Hash-цепочка делает лог **tamper-evident**: любое изменение
предыдущей записи нарушит целостность.

## Consequences

### Положительные

- **Zero Trust:** каждый вызов аутентифицирован и авторизован,
  независимо от сетевой зоны.
- **Криптографическая identity:** невозможно подделать SVID
  без компрометации SPIRE Server.
- **Автоматическая ротация:** SVID живёт 1 час, нет долгоживущих
  секретов в коде.
- **Аудит:** каждое действие привязано к конкретному агенту и SVID.
- **Fine-grained authorization:** RBAC + ABAC позволяют
  контролировать не только «что», но и «с какими атрибутами».
- **Attestation:** невозможно запустить «поддельного» агента
  и получить SVID.
- **Соответствие требованиям:** WIMSE, Zero Trust, ФСТЭК.

### Отрицательные / Риски

- **Сложность инфраструктуры:** нужен SPIRE Server + SPIRE Agents
  на каждой ноде.
- **Дополнительный latency:** mTLS handshake + проверка SVID
  добавляют 5–20 мс на вызов.
- **Обучение команды:** SPIFFE/SPIRE — новая технология для
  большинства разработчиков.
- **Отладка:** сложнее диагностировать проблемы (mTLS vs RBAC vs ABAC).
- **Single point of failure:** SPIRE Server критичен, нужен HA.

### Митигации

- **Сложность:** использовать managed SPIRE (например, Istio SPIFFE
  integration, или cloud provider support).
- **Latency:** кэширование SVID на стороне агента (1 час),
  connection pooling для mTLS.
- **Обучение:** внутренние workshop, документация, runbook.
- **Отладка:** observability через Langfuse + Jaeger, чтобы видеть
  всю цепочку (SVID → RBAC → ABAC → Execution).
- **HA:** SPIRE Server в cluster mode (3+ nodes), health checks.

## Alternatives Considered

### Альтернатива 1: API Keys

**Плюсы:** простота, работает везде.

**Минусы:** статичны, утекают, нет привязки к workload,
нет ротации, невозможно отследить конкретный экземпляр.

**Отвергнуто:** не соответствует Zero Trust и требованиям аудита.

### Альтернатива 2: Service Accounts (Kubernetes SA)

**Плюсы:** нативная интеграция с K8s, автоматическая ротация.

**Минусы:** привязаны к namespace, а не к конкретному workload.
Не работает за пределами K8s (on-prem, VM).

**Отвергнуто:** не покрывает гибридные среды и не даёт
криптографической identity на уровне агента.

### Альтернатива 3: OAuth 2.0 Client Credentials

**Плюсы:** стандарт, много библиотек, работает с любым языком.

**Минусы:**
- Нет криптографической identity (токен — это просто строка).
- Нет attestation.
- Нет автоматической ротации без дополнительной инфраструктуры.
- Токен можно украсть и использовать с другого IP.

**Отвергнуто:** OAuth может быть **слоем поверх** SVID (для
совместимости с legacy), но не заменой.

### Альтернатива 4: mTLS без SPIFFE

**Плюсы:** шифрование, взаимная аутентификация.

**Минусы:**
- Нет стандарта для identity (каждый сам решает, что класть
  в CN сертификата).
- Нет автоматической ротации без custom-кода.
- Нет attestation.
- Нет централизованного trust domain.

**Отвергнуто:** mTLS — транспорт, SPIFFE — identity. Нужны оба.

### Альтернатива 5: HashiCorp Vault + Dynamic Secrets

**Плюсы:** мощный, зрелый, много интеграций.

**Минусы:**
- Фокус на секретах (пароли, ключи), не на identity.
- Нет attestation workloads.
- Нет стандарта SPIFFE ID.

**Отвергнуто:** Vault решает другую задачу (secrets management).
Может использоваться **вместе** с SPIFFE (Vault как backend для
секретов, SPIFFE — для identity).

### Альтернатива 6: Custom PKI + JWTs

**Плюсы:** полный контроль, можно адаптировать под специфику.

**Минусы:**
- Много custom-кода → много багов.
- Нет готовых библиотек и стандартов.
- Нет сообщества и best practices.
- Сложно поддерживать ротацию, отзыв, attestation.

**Отвергнуто:** изобретение велосипеда. SPIFFE/SPIRE — это
industry standard, поддерживаемый CNCF.

## References

- SPIFFE Specification: https://spiffe.io/docs/latest/spiffe-about/
- SPIRE Documentation: https://spiffe.io/docs/latest/spire-about/
- WIMSE (IETF Working Group): https://datatracker.ietf.org/wg/wimse/about/
- NIST SP 800-207 (Zero Trust Architecture): https://csrc.nist.gov/publications/detail/sp/800-207/final
- Enterprise Agent Orchestration Blueprint, раздел 4.2 (WIMSE Identity)
- ADR-001: Hybrid Orchestration Core
- ADR-003: MCP for Legacy Gateways
- ADR-004: Observability, Audit & Cost Boundary
