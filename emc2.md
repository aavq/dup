Да. Для биллинга я бы **не квантовал CPU как «значение CPU каждые 5/10/15 минут»**. Это порождает систематическую ошибку: короткий пик можно пропустить, а долгий пик — наоборот, переоценить.

Правильная модель здесь — **CPU-time accounting**, то есть считать не «процент CPU», а **сколько секунд CPU фактически было потреблено**, а затем переводить это в CPU-hours.

### 1. Что именно считать

Для каждого NAR ID в каждый момент времени есть фактическая скорость потребления CPU:

$$
U(t) = \text{число занятых CPU cores}
$$

Например:

* `1.0` = workload в среднем использует одно полное ядро;
* `0.25` = четверть ядра;
* `4.0` = четыре ядра.

Это принципиально лучше, чем процент от мощности сервера. Для биллинга я бы использовал именно **physical/logical CPU-time**, а не `% of node`.

cAdvisor предоставляет `container_cpu_usage_seconds_total` как **counter накопленного CPU time в секундах**. Это именно тот первичный показатель, который здесь нужен. ([GitHub][1])

---

# 2. Основная единица биллинга — CPU-second

Пусть workload за интервал \(\Delta t\) потреблял в среднем \(U\) CPU cores.

Тогда:

$$
CPU\text{-}seconds = U \times \Delta t
$$

Например:

```text
0.0 core × 5 min =     0 CPU-sec
1.0 core × 5 min =   300 CPU-sec
2.0 cores × 5 min =  600 CPU-sec
```

А дальше:

$$
CPU\text{-}hours = \frac{CPU\text{-}seconds}{3600}
$$

То есть:

```text
1 core × 1 hour = 1 CPU-hour
4 cores × 1 hour = 4 CPU-hours
0.1 core × 10 hours = 1 CPU-hour
```

Это очень удобная и естественная единица для биллинга.

---

# 3. Самое важное: не делать sampling-based billing

Предположим, workload выглядит так:

```text
12:00   0 CPU
12:01   0 CPU
12:02   8 CPU
12:03   8 CPU
12:04   0 CPU
```

Если просто брать значение раз в 5 минут, результат будет зависеть от того, **в какую именно секунду попал sample**.

Например, можно случайно получить:

```text
0 CPU
```

хотя workload реально использовал:

```text
8 cores × 2 minutes = 16 CPU-minutes
```

Для биллинга это плохо.

---

# 4. Поэтому лучше использовать counter

У тебя уже фактически есть идеальный источник:

```text
container_cpu_usage_seconds_total
```

Это накопленный CPU-time.

Например:

```text
10:00   100000
10:01   100060
10:02   100120
10:03   100720
```

Разница:

```text
100060 - 100000 = 60 CPU-sec
100120 - 100060 = 60 CPU-sec
100720 - 100120 = 600 CPU-sec
```

Получаем:

```text
10:00–10:01 → 1 CPU-minute
10:01–10:02 → 1 CPU-minute
10:02–10:03 → 10 CPU-minutes
```

Итого:

```text
12 CPU-minutes
```

Это уже **фактическое потребление**, а не измерение instantaneous utilization.

Prometheus специально предоставляет `rate()` и `increase()` для counters; `rate()` также учитывает reset счётчика и missed scrapes через интерполяцию/экстраполяцию. ([Prometheus][2])

---

# 5. Я бы ввёл три разных понятия

Это очень полезно для архитектуры биллинга.

### Instantaneous CPU utilization

```text
cores
```

Например:

```text
0.7 cores
2.4 cores
8.1 cores
```

Это то, что пользователь видит на графике в реальном времени.

### Average CPU utilization

За период:

$$
\frac{\text{CPU-seconds}}{\text{wall-clock seconds}}
$$

Например:

```text
за месяц:
CPU consumed = 720 CPU-hours
period       = 720 hours

average      = 1.0 CPU
```

### CPU consumption

Для биллинга:

```text
CPU-seconds
CPU-hours
```

Я бы именно **CPU-hours сделал основной billable metric**.

---

# 6. Как агрегировать workload → namespace → NAR ID

У тебя фактически иерархия:

```text
container
   ↓
workload
   ↓
namespace
   ↓
NAR ID
```

Поэтому вычисление должно идти снизу вверх.

Например:

```text
Pod A
 ├─ container A1 = 0.3 CPU
 └─ container A2 = 0.7 CPU

Pod B
 └─ container B1 = 1.4 CPU
```

Получаем:

```text
namespace = 2.4 CPU
```

А затем:

```text
NAR ID = сумма всех namespace,
          относящихся к этому NAR ID
```

При этом очень важно выполнять `rate()` **до aggregation**. Prometheus прямо рекомендует сначала вычислять `rate()` на отдельных series, а затем делать `sum()`, поскольку иначе можно потерять информацию о counter resets. ([Prometheus][2])

---

# 7. PromQL-модель

Допустим, у тебя есть label:

```text
nar_id="12345"
```

И метрика:

```text
container_cpu_usage_seconds_total
```

Тогда фундаментальная формула примерно такая:

```promql
sum by (nar_id) (
  rate(
    container_cpu_usage_seconds_total{
      nar_id!="",
      container!="",
      image!=""
    }[5m]
  )
)
```

Результат:

```text
nar_id=12345 → 2.37
```

Это означает:

> данный NAR ID сейчас в среднем потребляет 2.37 CPU cores.

Причём это не «2.37%», а именно **2.37 CPU cores**.

---

# 8. Но для биллинга я бы не считал месячные значения непосредственно из `rate()[5m]`

Это важная архитектурная деталь.

Я бы создал **recording rule**:

```promql
nar:cpu_usage_cores:sum
```

например:

```promql
sum by (nar_id) (
  rate(
    container_cpu_usage_seconds_total{
      container!="",
      image!=""
    }[5m]
  )
)
```

и сохранял её, например, каждую минуту.

Prometheus recording rules как раз предназначены для предварительного вычисления часто используемых выражений, особенно для dashboard/query workloads. ([Prometheus][3])

---

# 9. Как из этого получить billing

Если recording rule записывает:

```text
nar:cpu_usage_cores:sum
```

раз в 60 секунд, то концептуально каждый sample представляет:

```text
CPU cores × 60 seconds
```

То есть:

```text
CPU-seconds ≈ sum(samples × 60)
```

Но я бы сделал ещё лучше:

## Хранить отдельно `CPU-seconds` как накопительный ряд

Например, сделать recording rule на уровне маленького интервала и потом суммировать его.

Или, если source counter доступен напрямую, использовать `increase()` на нужных интервалах.

Например:

```promql
sum(
  increase(
    container_cpu_usage_seconds_total{
      nar_id="12345"
    }[1h]
  )
)
```

Результат:

```text
123456 CPU-seconds
```

То есть:

```text
34.293 CPU-hours
```

потому что:

$$
123456 / 3600 = 34.293
$$

`increase()` фактически является `rate() × duration`, с обработкой counter resets. ([Prometheus][2])

---

# 10. Какой интервал квантования я бы выбрал

Вот здесь уже появляется инженерный компромисс.

Я бы рекомендовал:

| Уровень             |        Интервал |
| ------------------- | --------------: |
| raw scrape          |         15–30 s |
| billing calculation |       **1 min** |
| dashboard           |         1–5 min |
| daily aggregation   |             1 h |
| monthly billing     | 1 day / 1 month |

### Почему 1 минута

Для billing:

```text
60 seconds
```

даёт очень маленькую погрешность даже при крайне bursty workloads.

Допустим, workload действительно использовал:

```text
8 CPU
```

в течение 20 секунд.

Это:

```text
8 × 20 = 160 CPU-seconds
                 = 0.044 CPU-hours
```

Если считать по минутным buckets, ошибка будет порядка нескольких десятков CPU-seconds, а не нескольких минут CPU на большой мощности.

Для вашей платформы 1 minute выглядит очень разумным стандартом.

---

# 11. Но есть ещё более важная проблема: границы жизни namespace

Ты упомянул:

> есть данные о том, когда первый namespace был создан для определённого NAR ID.

Это отлично подходит для определения **billing start**.

Например:

```text
NAR ID 12345
created: 2026-05-17 14:23:11
```

Тогда:

```text
billing_start =
    2026-05-17 14:23:11
```

Но я бы **не округлял это до начала часа/дня**.

То есть не:

```text
2026-05-17 00:00
```

а именно:

```text
2026-05-17 14:23:11
```

Первый billing bucket просто будет частичным.

Например:

```text
14:23:11 ───────── 15:00:00
```

У тебя реально только:

```text
36m 49s
```

доступной истории.

Это важно для честного биллинга.

---

# 12. Что делать, если NAR ID имеет несколько namespace

Я бы определил строгий lifecycle rule:

```text
billing_start =
    earliest(namespace creation timestamp for NAR ID)
```

И:

```text
billing_end =
    latest namespace deletion timestamp
```

либо текущий момент, если NAR активен.

Получается:

$$
BillingPeriod =
[
first\_namespace\_created,\;
last\_namespace\_deleted
]
$$

А CPU consumption внутри этого периода:

$$
Consumption =
\sum CPU\text{-}seconds
$$

---

# 13. Очень важное правило: не считать отсутствие данных как нулевое потребление

Допустим:

```text
12:00 metric = 2 CPU
12:01 metric = 2 CPU
12:02 отсутствует
12:03 отсутствует
12:04 metric = 2 CPU
```

Нельзя автоматически сказать:

```text
12:02 = 0
12:03 = 0
```

Это будет означать, что workload реально не использовал CPU.

Но:

```text
missing metric
```

и

```text
CPU = 0
```

— совершенно разные вещи.

Для billing я бы ввёл состояние:

```text
VALID
MISSING
STALE
UNKNOWN
```

И отдельно считал coverage.

---

# 14. Billing должен иметь coverage

Это, на мой взгляд, одна из самых важных вещей для production-системы.

Например:

```text
NAR ID: 12345
Period: July 2026

CPU consumption: 1842.7 CPU-hours
Observed time:   738.2 hours
Expected time:   744.0 hours
Coverage:         99.22%
```

Тогда можно иметь правило:

```text
coverage >= 99%
    → billing valid

95–99%
    → billing with warning

<95%
    → billing not reliable
```

Конкретные thresholds — это уже **политика платформы**, а не технический факт.

Я бы их формально закрепил именно как billing policy.

---

# 15. Не надо использовать `irate()` для billing

`irate()` смотрит только на последние два samples и предназначен прежде всего для отображения быстро меняющихся counters. Prometheus рекомендует `rate()` для более стабильных расчётов. ([Prometheus][4])

Для биллинга:

```text
❌ irate()
✅ rate()
✅ increase()
✅ raw counter
```

---

# 16. Что делать с `rate(...[5m])`

Есть интересный нюанс.

Если workload:

```text
0 CPU
```

а затем:

```text
10 CPU в течение 20 секунд
```

`rate()[5m]` сгладит этот burst.

Для **визуализации** это нормально.

Для **billing** я бы не трактовал `rate()[5m]` как истину саму по себе.

Источник истины должен быть:

```text
CPU counter
```

а `rate()` — производное представление.

Это важное разделение:

```text
Source of truth
    ↓
container_cpu_usage_seconds_total
    ↓
CPU seconds consumed
    ↓
billing
```

а:

```text
rate()
    ↓
observability / dashboard
```

---

# 17. Архитектура, которую я бы предложил

Примерно такая:

```text
                    cAdvisor / kubelet
                           │
                           ▼
           container_cpu_usage_seconds_total
                           │
                           ▼
                      Prometheus
                           │
               ┌───────────┴──────────┐
               │                      │
               ▼                      ▼
        operational metrics       billing rules
               │                      │
               │                      ▼
               │              nar:cpu_usage:sum
               │                      │
               │                      ▼
               │                CPU-seconds
               │                      │
               │                      ▼
               │                CPU-hours
               │                      │
               └───────────────┬──────┘
                               ▼
                            Grafana
```

---

# 18. Я бы сделал отдельную billing metric

Например:

```text
platform_nar_cpu_usage_cores
platform_nar_cpu_usage_seconds_total
platform_nar_cpu_usage_hours_total
platform_nar_cpu_data_coverage_ratio
```

Это лучше, чем заставлять Grafana каждый раз вычислять всё из десятков миллионов container series.

Recording rules для этого как раз подходят; Prometheus отдельно рекомендует использовать их для дорогих повторяющихся запросов dashboard'ов. ([Prometheus][3])

---

# 19. Grafana dashboard для одного NAR ID

Я бы сделал dashboard примерно такого вида.

### Верхняя строка

```text
NAR ID: 12345

Billing start:
17 May 2026 14:23:11

Current CPU:
2.34 cores

Average CPU:
1.17 cores

CPU consumed:
──────────────
Last 24h       28.4 CPU-h
Last 7d       172.3 CPU-h
Last 30d      711.8 CPU-h
Last 6m      4217.5 CPU-h
Last 12m     8122.3 CPU-h

Data coverage:
99.87%
```

---

# 20. Главный график

Не просто CPU utilization.

Я бы сделал:

### `CPU consumption — cores`

Time series:

```text
CPU cores
10 ┤       ╭╮
 8 ┤      ╭╯╰╮
 6 ┤   ╭──╯  ╰────╮
 4 ┤───╯          ╰──
 2 ┤
 0 ┼────────────────────
```

Это отвечает на вопрос:

> «Сколько CPU использовал NAR в каждый момент?»

---

# 21. Второй график — cumulative CPU-hours

Вот это уже непосредственно billing.

Например:

```text
CPU-hours accumulated

800 ┤                         ╭──
700 ┤                    ╭────╯
600 ┤               ╭────╯
500 ┤          ╭────╯
400 ┤      ╭───╯
300 ┤  ╭───╯
200 ┤──╯
```

Очень удобно видеть:

> сколько CPU-time реально накоплено.

---

# 22. Ещё один очень полезный график

### Daily CPU consumption

```text
Date        CPU-hours

01 Aug      21.4
02 Aug      17.2
03 Aug      31.8
04 Aug       4.3
05 Aug      28.7
...
```

Для финансового/финансово-похожего отчёта этот график часто полезнее, чем raw CPU utilization.

---

# 23. Для периода «месяц / 6 месяцев / год»

Я бы **не показывал raw 1-minute samples за год**.

Для этого делается downsampling:

```text
raw:
15–30 sec

billing:
1 min

dashboard:
5 min

daily:
1h / 1day

long-term:
1 day
```

Например:

```text
2026-08-01 → 182.4 CPU-hours
2026-08-02 → 174.2 CPU-hours
...
```

А за полгода Grafana уже строит график из daily aggregates.

---

# 24. Важная ловушка с namespace creation

Есть ещё одна концептуальная проблема.

Допустим:

```text
NAR 12345

namespace-a     created 1 Jan
namespace-b     created 1 Feb
namespace-c     created 1 Mar
```

Если namespace-a уничтожили 15 февраля, а namespace-b продолжает жить, то:

```text
NAR active period
```

и

```text
namespace existence
```

— не одно и то же.

Поэтому я бы хранил lifecycle:

```text
NAR ID
namespace
created_at
deleted_at
```

и CPU usage связывал именно с конкретным namespace/workload.

Тогда можно корректно ответить:

```text
CPU of NAR =
sum(CPU of all namespace intervals belonging to NAR)
```

---

# 25. Ещё одна важная вещь: requests/limits не должны попадать в consumption

Например:

```text
CPU request = 4
CPU limit   = 8
actual      = 0.7
```

Для usage billing:

```text
0.7 CPU
```

а не:

```text
4 CPU
```

и не:

```text
8 CPU
```

Requests/limits можно показать отдельными метриками:

```text
Requested CPU
Limit CPU
Actual CPU
```

Это будет очень наглядно.

---

# 26. Я бы добавил utilisation относительно request

Для пользователя NAR ID это может быть очень полезно:

$$
RequestUtilization =
\frac{ActualCPU}{CPURequest}
$$

Например:

```text
requested: 4 CPU
actual:    0.8 CPU

utilization = 20%
```

Это **не billing metric**, а operational metric.

Таким образом dashboard показывает одновременно:

```text
Actual consumption    0.8 CPU
CPU request            4 CPU
Request utilization   20%
CPU-hours today       19.2
```

---

# 27. Что я считаю наиболее правильной итоговой моделью

Я бы формализовал billing следующим образом:

$$
\boxed{
CPUHours_{NAR,P}
=
\frac{
\sum CPUSeconds_{NAR,P}
}{3600}
}
$$

где:

$$
CPUSeconds_{NAR,P}
=
\sum_i
\Delta counter_i
$$

для всех workloads/containers, принадлежащих NAR ID и попадающих в billing period.

А billing period:

$$
P =
[max(first\ namespace\ creation,\ period\ start),
min(last\ namespace\ deletion,\ period\ end)]
$$

при этом:

**нулевое CPU ≠ отсутствие telemetry.**

Это принципиальное правило.

---

# 28. Конкретная рекомендация по вашей платформе

Я бы зафиксировал такую policy:

```text
Raw metric:
container_cpu_usage_seconds_total

Raw scrape:
15–30 s

Billing quantum:
1 minute

Billing unit:
CPU-hour

Aggregation:
container
 → workload
 → namespace
 → NAR ID

Billing start:
earliest namespace creation for NAR ID

Billing end:
latest namespace deletion / query end

Primary billing source:
CPU counter, not sampled gauge

Allowed missing telemetry:
explicitly measured as coverage

Prometheus:
recording rules

Grafana:
current CPU
daily CPU-hours
monthly CPU-hours
6-month CPU-hours
12-month CPU-hours
coverage
```

И ещё одно архитектурное решение я бы принял сразу: **не привязывать биллинг к Grafana вообще**. Grafana должна только визуализировать уже рассчитанную billing-модель. Иначе спустя год вы измените PromQL, retention или шаг Grafana — и получите другой исторический счёт.

Для долгосрочного биллинга я бы сделал отдельный слой **billing records / recording rules**, который является контрактом между monitoring и billing.

Поскольку recording rules появляются только с момента их создания, старую историю автоматически получить нельзя; при необходимости Prometheus поддерживает backfilling recording-rule data через `promtool`. ([Prometheus][5])

### Итого

Самая важная идея здесь очень простая:

> **Не считать «сколько CPU было в момент измерения». Считать интеграл CPU utilization по времени.**

То есть:

```text
CPU utilization × time
        ↓
CPU-seconds
        ↓
CPU-hours
        ↓
billing
```

И это хорошо совпадает с тем, как cAdvisor предоставляет CPU accounting: `container_cpu_usage_seconds_total` — именно накопленное CPU time, а Prometheus `rate()`/`increase()` предназначены для работы с таким counter. ([GitHub][1])

Если строить это именно как production-решение для вашей Kubernetes-платформы, следующий шаг — определить **точную схему Prometheus recording rules + PromQL + структуру labels + Grafana dashboard**, включая корректный учёт нескольких namespace одного NAR ID, исчезнувших pods и telemetry gaps. Я бы именно это сделал следующим слоем методики.

[1]: https://github.com/google/cadvisor/blob/master/docs/storage/prometheus.md?utm_source=chatgpt.com "cadvisor/docs/storage/prometheus.md at master · google/cadvisor · GitHub"
[2]: https://prometheus.io/docs/prometheus/2.55/querying/functions/?utm_source=chatgpt.com "Query functions | Prometheus"
[3]: https://prometheus.io/docs/prometheus/latest/configuration/recording_rules/?utm_source=chatgpt.com "Defining recording rules | Prometheus"
[4]: https://prometheus.io/docs/prometheus/1.8/querying/functions/?utm_source=chatgpt.com "Query functions | Prometheus"
[5]: https://prometheus.io/docs/prometheus/latest/storage/?utm_source=chatgpt.com "Storage | Prometheus"
