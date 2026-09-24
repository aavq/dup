Да. Я бы дал агенту **не задачу "напиши диагностический инструмент"**, а задачу на создание **архитектурного MVP + первого набора правил**, причём отдельно заставил бы его сначала исследовать твои How-to Guides. Это существенно снижает риск, что агент сразу начнёт писать десятки частных `if`-ов.

Ниже готовый промпт, который я бы использовал практически без изменений.

# Задача: разработать MVP Kubernetes Workload Diagnostic Framework

## 1. Цель проекта

Нужно разработать extensible, modular, read-only инструмент для диагностики Kubernetes workloads.

Рабочее название: `kdiag` / `kubernetes-workload-diagnostics`.

Инструмент должен принимать Kubernetes namespace и, опционально, selector конкретного workload, анализировать состояние уже работающих ресурсов в Kubernetes cluster и выдавать потенциальные проблемы, найденные на основании формализованных диагностических правил.

Главная идея проекта:

> Формализовать знания и troubleshooting-процессы Kubernetes/platform engineers в виде воспроизводимых, тестируемых диагностических правил.

Это **не AI-agent**, не remediation tool и не admission controller.

Инструмент должен:

* только читать состояние Kubernetes;
* ничего не изменять;
* не выполнять remediation;
* не создавать/удалять/изменять Kubernetes resources;
* работать сначала локально с обычным kubeconfig пользователя;
* в дальнейшем быть пригодным для запуска от имени Kubernetes ServiceAccount;
* быть расширяемым новыми диагностическими правилами без переписывания core engine.

---

# 2. Важный контекст проекта

Перед началом реализации необходимо изучить существующие внутренние материалы проекта.

В репозитории/доступных файлах существуют How-to Guides в XHTML и/или нормализованном виде.

Каждый How-to Guide описывает конкретную инженерную задачу:

* проблему пользователя;
* последовательность проверки;
* Kubernetes resources, которые инженер смотрит;
* условия;
* взаимосвязи между ресурсами;
* возможные причины проблемы;
* ожидаемый результат.

В этих материалах уже сформировано около 20 внутренних правил, специфичных для нашей Kubernetes/platform environment.

Они являются одним из главных источников domain knowledge для первой версии инструмента.

## Первое действие

Перед написанием production code:

1. Найди и изучи существующие How-to Guides.
2. Определи, какие из них непосредственно относятся к диагностике Kubernetes workload.
3. Выдели минимум 10–15 наиболее подходящих troubleshooting scenarios.
4. Не переписывай их механически в код.
5. Для каждого сценария сначала формализуй:

   * problem;
   * scope;
   * required Kubernetes resources;
   * relationships между resources;
   * observations;
   * conditions;
   * evidence;
   * diagnostic conclusion;
   * severity;
   * confidence;
   * возможные remediation hints.

Создай отдельный документ:

`docs/diagnostic-rules.md`

В нём должен находиться первоначальный каталог правил.

---

# 3. Не начинать с частных Kubernetes checks

Ключевое требование:

**Не начинай проект с реализации отдельных checks вроде `checkServicePort()`, `checkPodStatus()` и т.п.**

Сначала спроектируй framework, который позволит реализовывать такие проверки как независимые правила.

Нужно построить архитектуру примерно следующего уровня:

```text
Kubernetes Cluster
        │
        ▼
Resource Discovery
        │
        ▼
Resource Collector
        │
        ▼
Workload / Resource Graph
        │
        ▼
Diagnostic Engine
        │
        ├── Rule 1
        ├── Rule 2
        ├── Rule 3
        ├── ...
        └── Rule N
        │
        ▼
Diagnostic Findings
        │
        ▼
Output / Report
```

Архитура должна позволять добавлять новые правила без изменения core engine.

---

# 4. Scope MVP

MVP должен быть достаточно маленьким, чтобы быть законченным после одного implementation cycle.

Не пытайся реализовать все возможные Kubernetes проблемы.

В MVP должны присутствовать:

### CLI

Минимально:

```bash
kdiag -n <namespace>
```

и:

```bash
kdiag -n <namespace> -l app=my-app
```

где `-l` — Kubernetes label selector.

Желательно также поддержать:

```bash
kdiag --namespace <namespace>
kdiag --selector <selector>
```

Обычный kubeconfig должен использоваться через стандартный Kubernetes client configuration.

Не требовать ServiceAccount для MVP.

Пример:

```bash
kubectl config current-context
kdiag -n my-namespace
```

должен работать с тем же доступом, который пользователь имеет через kubeconfig.

---

# 5. Workload scope

Namespace может содержать ресурсы нескольких команд.

Поэтому инструмент должен иметь два режима:

### Namespace mode

```bash
kdiag -n team-a
```

Анализирует namespace.

### Workload mode

```bash
kdiag -n team-a -l app=my-api
```

Анализирует только ресурсы, относящиеся к workload, определяемому selector.

Необходимо предусмотреть возможность расширить workload identification в будущем.

Важно не делать предположение, что workload всегда состоит только из:

```text
Deployment → ReplicaSet → Pod
```

В архитектуре должны быть предусмотрены разные workload/resource relationships.

Для MVP можно реализовать ограниченный набор Kubernetes workload types, но architecture должна позволять расширение.

---

# 6. Resource Graph

Центральной концепцией проекта должен быть Resource Graph.

Необходимо иметь возможность выразить relationships вроде:

```text
Deployment
    ↓
ReplicaSet
    ↓
Pod

Service
    ↓ selector
Pod

Ingress
    ↓ backend
Service

Ingress
    ↓ TLS
Secret

Certificate
    ↓
Secret

Certificate
    ↓
CertificateRequest
    ↓
Order
    ↓
Challenge

Pod
    ↓
PVC

Pod
    ↓
ServiceAccount
```

Для Istio в дальнейшем:

```text
Gateway
VirtualService
DestinationRule
Service
Pod
Istio sidecar
```

Graph не обязательно должен быть сложной generic graph database.

Можно использовать типизированную in-memory модель.

Главное требование:

> правила должны работать с моделью отношений между ресурсами, а не самостоятельно делать десятки независимых Kubernetes API queries.

---

# 7. Resource Collection

Создай отдельный abstraction layer для получения Kubernetes resources.

Например концептуально:

```text
ResourceCollector
    ├── Pods
    ├── Deployments
    ├── ReplicaSets
    ├── Services
    ├── EndpointSlices
    ├── Events
    ├── Secrets
    ├── ConfigMaps
    ├── Ingresses
    ├── PVCs
    └── ...
```

Для CRDs architecture должна предусматривать возможность добавления dynamic resources.

Не нужно в MVP поддерживать все возможные CRDs.

Но framework должен позволять добавить:

```text
Certificate
CertificateRequest
Order
Challenge
VirtualService
Gateway
DestinationRule
```

без переписывания collector/engine.

---

# 8. Diagnostic Rule abstraction

Разработай формальную модель Rule.

Каждое правило должно иметь минимум:

```text
Rule ID
Name
Category
Description
Severity
Scope
Required resources
Check logic
Evidence
Diagnostic message
Recommendation
```

Желательно также:

```text
Confidence
Documentation reference
```

Пример концептуальной структуры:

```yaml
id: SERVICE-001
name: Service has no ready endpoints
category: service
severity: error

description: >
  Detects Services that do not have any ready backend endpoints.

requires:
  - Service
  - EndpointSlice
  - Pod

condition:
  ...

message:
  ...

recommendation:
  ...
```

Это не обязательно должен быть YAML runtime format.

Можно начать с strongly typed Go structures.

Но правила должны иметь чёткую декларативную идентичность.

---

# 9. Findings

Rule не должен просто возвращать `true/false`.

Нужна структурированная диагностическая информация.

Например:

```text
Finding:
    RuleID
    Severity
    Category
    Resource
    Message
    Evidence
    Recommendation
    Confidence
```

Пример:

```text
ERROR SERVICE-001

Service: my-api

Service has no ready endpoints.

Selector:
    app=my-api

Matching pods:
    3

Ready pods:
    0

Evidence:
    pod/my-api-abc: Ready=False
    pod/my-api-def: Ready=False
    pod/my-api-ghi: Ready=False

Recommendation:
    Check pod readiness and application health.
```

Это должно быть пригодно не только для terminal output, но и для JSON output в будущем.

---

# 10. Severity

Минимально предусмотреть:

```text
INFO
WARNING
ERROR
```

Желательно предусмотреть:

```text
UNKNOWN
```

Очень важно не выдавать предположение за доказанный факт.

Например:

```text
Service targetPort = 5230
Pod не содержит containerPort = 5230
```

не означает автоматически, что приложение не слушает 5230.

`containerPort` является декларативной информацией и не гарантирует фактическое состояние socket/application.

В таком случае результат может быть:

```text
WARNING / UNKNOWN
```

с объяснением.

Принцип:

> Tool должен предпочитать "не удалось доказать проблему" ложному ERROR.

---

# 11. Первые диагностические правила

Для MVP выбери 10–15 правил на основании реальных How-to Guides.

Если существующие guides позволяют это, желательно покрыть следующие классы проблем:

### Workload

* Pod stuck in Pending
* Pod CrashLoopBackOff
* excessive container restarts
* Deployment has unavailable replicas
* readiness failure

### Scheduling

* FailedScheduling events
* insufficient CPU/memory
* nodeSelector matches no nodes
* required node affinity cannot be satisfied
* taint/toleration mismatch

### Service

* Service selector matches no Pods
* Service has no ready endpoints
* Service/EndpointSlice inconsistency
* suspicious targetPort mismatch

### Ingress/TLS

* Ingress references nonexistent Service
* Ingress TLS references nonexistent Secret
* Certificate is not Ready
* relevant cert-manager resource is in failed state

### Istio

* namespace injection expectation not satisfied
* expected Pod has no Istio proxy
* Istio configuration references nonexistent resource

Но этот список является ориентиром.

**Приоритет имеют реальные внутренние How-to Guides.**

Не реализовывай правило только потому, что оно перечислено выше, если внутренние материалы показывают другой приоритет.

---

# 12. Events

Events должны быть first-class diagnostic input.

Не ограничиваться:

```bash
kubectl get pods
```

Нужно анализировать events namespace, особенно:

```text
FailedScheduling
FailedMount
FailedAttachVolume
FailedCreatePodSandBox
BackOff
Unhealthy
Failed
```

Однако не превращать каждое event в ошибку.

Необходимо:

* группировать повторяющиеся events;
* учитывать count;
* учитывать timestamps;
* связывать event с resource;
* использовать event как evidence для других rules.

Например:

```text
Pod Pending
+
FailedScheduling
+
Insufficient cpu
```

должно приводить к одному осмысленному diagnostic finding, а не к трём независимым шумным сообщениям.

---

# 13. Reusable diagnostics

Особое внимание удели переиспользованию.

Например не должно быть:

```text
Ingress rule самостоятельно ищет Service
Service rule самостоятельно ищет Pods
Istio rule самостоятельно ищет Pods
TLS rule самостоятельно ищет Secrets
```

Вместо этого должны существовать reusable services/components:

```text
WorkloadResolver
ServiceResolver
PodResolver
EndpointResolver
OwnerResolver
ResourceReferenceResolver
EventResolver
TLSResolver
IstioResolver
```

и rules должны использовать их.

Например:

```text
ServiceResolver
    getMatchingPods(service)

EndpointResolver
    getReadyEndpoints(service)

WorkloadResolver
    getPods(workload)

ResourceReferenceResolver
    resolveService(...)
    resolveSecret(...)
```

Это позволит постепенно строить библиотеку reusable Kubernetes diagnostics.

---

# 14. Kubernetes API efficiency

Не писать правило, которое выполняет сотни последовательных `kubectl`-подобных API calls.

Resource collection должен по возможности:

* получать resources пакетно;
* кэшировать их;
* строить index maps;
* использовать namespace-scoped queries там, где это возможно;
* повторно использовать уже загруженные resources.

Например:

```text
podsByNamespace
podsByLabel
servicesByName
deploymentsByName
eventsByInvolvedObject
```

и т.п.

Особенно это важно для больших namespace.

---

# 15. Output

CLI output должен быть удобным для инженера.

Пример:

```text
KDIAG
Cluster:   my-cluster
Namespace: customer-a
Selector:  app=my-api

Collecting resources...
████████████████████████████████ 100%

Analyzing workload...

✓ WORKLOAD-001   Deployment available
✓ SERVICE-001    Service has matching pods
✗ SERVICE-002    Service has no ready endpoints
⚠ TLS-001        Certificate is not Ready
✗ ISTIO-001      Pod has no Istio sidecar

Summary:
    PASS: 2
    WARN: 1
    ERROR: 2
```

Не нужно тратить значительную часть MVP на UI.

Красивый output желателен, но correctness и architecture важнее.

---

# 16. Quiet mode

Обязательно предусмотреть:

```bash
kdiag -n foo --quiet
```

В quiet mode выводить только findings, которые требуют внимания:

```text
ERROR SERVICE-002 Service my-api has no ready endpoints
WARN TLS-001 Certificate api-tls is not Ready
```

PASS entries не выводить.

---

# 17. Exit codes

Предусмотреть machine-readable exit codes:

```text
0 = no errors
1 = warnings only
2 = errors detected
```

Если это усложняет MVP, реализовать хотя бы:

```text
0 = no ERROR findings
1 = ERROR findings
```

Но архитектура должна позволять расширение.

---

# 18. JSON output

В MVP желательно реализовать:

```bash
kdiag -n foo --output json
```

Это важно для дальнейшей интеграции.

JSON должен содержать structured findings, а не terminal-formatted text.

Пример:

```json
{
  "cluster": "cluster-a",
  "namespace": "customer-a",
  "findings": [
    {
      "ruleId": "SERVICE-002",
      "severity": "ERROR",
      "resource": {
        "kind": "Service",
        "name": "my-api"
      },
      "message": "Service has no ready endpoints"
    }
  ]
}
```

---

# 19. Testing architecture

Каждое правило должно быть unit-testable без Kubernetes cluster.

Не писать тесты, которым обязательно нужен live cluster.

Предусмотреть возможность:

```text
Kubernetes API
       ↓
Resource Collector
       ↓
Resource Snapshot
       ↓
Diagnostic Engine
       ↓
Rules
```

Rules должны иметь возможность работать с заранее подготовленным snapshot.

Это позволит создавать fixtures:

```text
testdata/
  service-no-endpoints/
  pending-pod/
  certificate-failed/
  istio-no-sidecar/
```

и тестировать реальные сценарии.

---

# 20. Test fixtures

Создай хотя бы несколько representative fixtures для MVP.

Например:

```text
fixtures/
├── healthy-workload/
├── service-no-pods/
├── service-no-ready-endpoints/
├── pending-pod/
├── failed-scheduling/
├── tls-secret-missing/
└── certificate-not-ready/
```

Не обязательно поднимать настоящий Kubernetes cluster.

Основная часть rule engine должна тестироваться на fixture data.

---

# 21. Local execution

MVP обязан запускаться локально.

Требование:

```bash
kubectl config current-context
kdiag -n some-existing-namespace
```

должно быть достаточным.

Не требуется:

* ServiceAccount;
* Helm installation;
* Kubernetes Deployment;
* Operator;
* custom CRD;
* external database;
* Redis;
* Prometheus;
* Grafana.

Обычный kubeconfig должен быть единственным обязательным runtime dependency.

---

# 22. ServiceAccount

Одновременно с разработкой MVP подготовь документацию:

`docs/service-account.md`

Документ должен описывать production usage от имени Kubernetes ServiceAccount.

Нужно описать:

1. создание ServiceAccount;
2. создание минимально необходимого Role/ClusterRole;
3. RoleBinding/ClusterRoleBinding;
4. получение credentials;
5. использование credentials с kubectl;
6. использование credentials с kdiag;
7. rotation/revocation;
8. security considerations.

Принцип:

> Least privilege.

Не выдавать `cluster-admin`.

Разрешения должны быть обоснованы каждым resource type.

Например концептуально:

```text
pods                 get/list
deployments          get/list
replicasets          get/list
services             get/list
endpointslices       get/list
events               get/list
ingresses             get/list
secrets               get/list
configmaps            get/list
persistentvolumeclaims get/list
nodes                 get/list   # только если конкретные scheduling rules требуют этого
```

CRD permissions должны добавляться только при необходимости.

Особенно внимательно относиться к `secrets`.

Если конкретное диагностическое правило может работать без чтения Secret data, не выдавать доступ к secret contents.

Например для проверки:

```text
Secret exists?
```

может быть достаточно metadata.

Но если правило действительно проверяет содержимое TLS Secret, это должно быть явно описано.

Не предполагать автоматически, что `secrets` нужно разрешить cluster-wide.

---

# 23. Production credentials

Не делать в MVP собственный механизм token management.

Для начала достаточно стандартных Kubernetes credentials.

Если cluster version поддерживает современный TokenRequest mechanism, документация должна предпочитать короткоживущие credentials вместо долгоживущих Secret-based ServiceAccount tokens.

Но не делать credential-management частью самого kdiag.

kdiag должен просто использовать стандартный Kubernetes authentication mechanism.

---

# 24. Existing tools

Перед реализацией обязательно проведи короткое исследование существующих инструментов.

Рассмотри как минимум:

* `istioctl analyze`
* KubeLinter
* kube-score
* Kubevious
* другие релевантные Kubernetes diagnostic/linting tools, если найдёшь.

Цель исследования не в том, чтобы добавить все эти проекты как dependencies.

Цель:

> определить, какие функции разумно reuse/integrate, а какие являются уникальной частью нашего diagnostic framework.

Результат оформить в:

`docs/existing-tools.md`

Особенно внимательно изучить возможность использования `istioctl analyze` для Istio-specific diagnostics вместо повторной реализации уже существующих Istio checks.

---

# 25. Internal How-to Guides → Rules

Для каждого выбранного How-to Guide создай mapping:

```text
How-to Guide
     ↓
Problem statement
     ↓
Diagnostic scenario
     ↓
Rules
     ↓
Kubernetes resources
     ↓
Evidence
```

Например:

```text
HOWTO-XXX
"Ingress TLS troubleshooting"

becomes:

TLS-001
TLS-002
SERVICE-XXX
CERT-XXX
```

Но не обязательно один How-to Guide = одно правило.

Один guide может породить несколько reusable rules.

И наоборот, одно reusable rule может использоваться несколькими guides.

Это важно для архитектуры.

---

# 26. Правило против workflow

Разделяй:

### Rule

Отвечает на конкретный вопрос:

> Есть ли у Service ready endpoints?

### Diagnostic workflow

Отвечает на вопрос:

> Почему Ingress не работает?

Workflow может использовать:

```text
Ingress rule
+
Service rule
+
Endpoint rule
+
Pod rule
+
TLS rule
+
Istio rule
```

Не смешивать эти два понятия.

В MVP достаточно реализовать Rule Engine.

Но архитектура должна позволять в будущем строить workflows поверх rules.

---

# 27. Не реализовывать remediation

Даже если finding содержит:

```text
Recommendation:
  Add label app=my-api
```

инструмент не должен выполнять:

```bash
kubectl label ...
```

Никаких mutations.

Никаких:

```text
fix
repair
apply
patch
restart
delete
rollout restart
```

в MVP.

---

# 28. Документация

В результате первого implementation cycle должны появиться:

```text
README.md

docs/
├── architecture.md
├── diagnostic-rules.md
├── existing-tools.md
├── service-account.md
└── development.md
```

README должен содержать:

1. Что делает инструмент.
2. Что он не делает.
3. Installation/build.
4. Local usage.
5. Examples.
6. Supported diagnostics.
7. JSON output.
8. Exit codes.
9. Architecture overview.
10. How to add a new rule.

---

# 29. Definition of Done

Первый implementation cycle считается завершённым, когда:

### Architecture

* существует modular diagnostic engine;
* resource collection отделён от rules;
* resource relationships представлены отдельным слоем;
* rules независимы друг от друга;
* новые rules можно добавлять без изменения core engine.

### Functionality

* работает `kdiag -n <namespace>`;
* работает label selector;
* используется текущий kubeconfig;
* инструмент read-only;
* есть минимум 10 реальных diagnostic rules;
* правила основаны преимущественно на существующих How-to Guides;
* есть Events analysis;
* есть human-readable output;
* есть quiet mode;
* есть JSON output;
* есть meaningful exit code.

### Testing

* rule engine тестируется без live Kubernetes;
* есть fixtures;
* основные правила имеют unit tests.

### Documentation

* архитектура описана;
* правила описаны;
* существующие tools исследованы;
* описано создание ServiceAccount;
* описаны необходимые RBAC permissions;
* описано локальное использование.

---

# 30. Что НЕ делать в первом implementation cycle

Не делать:

* web UI;
* Grafana integration;
* Prometheus integration;
* database;
* Kubernetes Operator;
* CRD для самого kdiag;
* automatic remediation;
* AI/LLM integration;
* autonomous agent mode;
* distributed architecture;
* multi-cluster controller;
* сложную plugin marketplace architecture;
* полный анализ всех Kubernetes resources;
* сотни правил;
* собственный authentication system.

Также не нужно пытаться решить каждую потенциальную Kubernetes проблему.

MVP должен доказать архитектуру:

```text
How-to Guide
    ↓
formal diagnostic rule
    ↓
resource collection
    ↓
resource graph
    ↓
rule engine
    ↓
finding
    ↓
human + machine readable output
```

---

# 31. Приоритеты

Если времени недостаточно, приоритеты следующие:

1. Изучение How-to Guides.
2. Formalization of 10–15 rules.
3. Architecture.
4. Resource snapshot / graph.
5. Rule engine.
6. 10+ working rules.
7. Unit tests.
8. CLI.
9. JSON.
10. Documentation.
11. ServiceAccount documentation.
12. Cosmetic improvements.

Не жертвовать архитектурой ради большого количества rules.

---

# 32. Expected first response / progress report

Перед началом большого объёма coding сначала покажи краткий результат анализа:

```text
1. Какие How-to Guides найдены.
2. Какие 10–15 scenarios выбраны для MVP.
3. Какие resource types нужны.
4. Предлагаемая architecture.
5. Какие existing tools будут reused/integrated.
6. Какие assumptions сделаны.
7. Какие вопросы действительно блокируют реализацию.
```

Если существенных blocking questions нет — **не останавливайся для согласования каждого решения**.

Выбери разумные defaults и продолжай реализацию.

В конце первого цикла предоставь:

```text
- source code
- build/run instructions
- test results
- example output
- list of implemented rules
- architecture documentation
- ServiceAccount/RBAC documentation
- known limitations
- next logical steps
```

---

# 33. Главный принцип проекта

Не строить набор отдельных Kubernetes checks.

Строить **framework для формализации инженерной диагностики**.

Ценность проекта должна быть в том, что со временем новые знания инженеров можно превращать в новые rules:

```text
Engineer troubleshooting knowledge
             ↓
        How-to Guide
             ↓
   Formal diagnostic scenario
             ↓
        Reusable rules
             ↓
        kdiag engine
```

Если через несколько месяцев появится новый How-to Guide:

> "User cannot connect to SFTP workload"

идеальным результатом должно быть не написание ещё одного большого специального скрипта, а возможность выразить эту диагностику через уже существующие reusable components:

```text
WorkloadResolver
ServiceResolver
EndpointResolver
PortResolver
PodResolver
IstioResolver
EventResolver
NetworkResolver
```

и добавить только действительно новую diagnostic logic.

Главный критерий качества архитектуры:

> **Новая проблема должна добавляться как новый набор rules, а не как новый одноразовый инструмент.**

Я бы ещё сделал **одно небольшое изменение в стратегии агента**: не заставлял бы его сначала ждать твоего approval после пункта 32. В промпте специально оставил: *показать краткий анализ, но если блокирующих вопросов нет — продолжать*. Иначе агент может потратить весь первый запуск на красивый architecture document и остановиться.

И ещё важный момент: **твои XHTML How-to Guides здесь могут стать главным конкурентным преимуществом этого инструмента**. Generic Kubernetes tooling уже умеет много чего проверять. А вот ваши внутренние 20 правил — это фактически накопленная экспертиза команды, которую можно превратить в executable knowledge base. Я бы поэтому сделал `docs/diagnostic-rules.md` не просто документацией, а **первым артефактом проекта, из которого дальше развивается код**.
