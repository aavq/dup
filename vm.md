
Ниже запросы для **VictoriaMetrics MetricsQL/PromQL**. Я подставил кластер, namespace, pod и container со скриншота. Основные метрики `container_cpu_usage_seconds_total` и CFS-счётчики throttling экспортирует cAdvisor. 

В запросах ниже окно — **5 минут**. Для графика со всплесками замените `[5m]` на `[1m]`, если данных хватает для такого короткого окна.

**1. Потребление CPU в ядрах**

```promql
sum by (cluster, namespace, pod, container) (
  rate(container_cpu_usage_seconds_total{
    cluster="db-prd-0c2k-fic-hybrid-cloud-de-dcb-cluster-01",
    namespace="torch",
    pod="nexuspay-dcb-cluster-torch-temporal-admintools-7d7c9db48-sktcj",
    container="admin-tools"
  }[5m])
)
```

Например, значение `0.04` означает в среднем `0.04` ядра, то есть `40m`.

**2. CPU limit контейнера в ядрах**

```promql
max by (cluster, namespace, pod, container) (
  kube_pod_container_resource_limits{
    cluster="db-prd-0c2k-fic-hybrid-cloud-de-dcb-cluster-01",
    namespace="torch",
    pod="nexuspay-dcb-cluster-torch-temporal-admintools-7d7c9db48-sktcj",
    container="admin-tools",
    resource="cpu",
    unit="core"
  }
)
```

Здесь используется `max by`, чтобы одинаковые ряды лимита, если они продублированы разными scrape-метками, не суммировались. Если в вашей конфигурации дубликатов нет, значение будет тем же, что при `sum by`.

**3. Утилизация относительно CPU limit, %**

```promql
100 *
sum by (cluster, namespace, pod, container) (
  rate(container_cpu_usage_seconds_total{
    cluster="db-prd-0c2k-fic-hybrid-cloud-de-dcb-cluster-01",
    namespace="torch",
    pod="nexuspay-dcb-cluster-torch-temporal-admintools-7d7c9db48-sktcj",
    container="admin-tools"
  }[5m])
)
/
on (cluster, namespace, pod, container)
max by (cluster, namespace, pod, container) (
  kube_pod_container_resource_limits{
    cluster="db-prd-0c2k-fic-hybrid-cloud-de-dcb-cluster-01",
    namespace="torch",
    pod="nexuspay-dcb-cluster-torch-temporal-admintools-7d7c9db48-sktcj",
    container="admin-tools",
    resource="cpu",
    unit="core"
  }
)
```

Это ближе всего к запросу «CPU usage / CPU limit» со скриншота.

**4. Доля CPU-периодов с throttling, %**

```promql
100 *
sum by (cluster, namespace, pod, container) (
  rate(container_cpu_cfs_throttled_periods_total{
    cluster="db-prd-0c2k-fic-hybrid-cloud-de-dcb-cluster-01",
    namespace="torch",
    pod="nexuspay-dcb-cluster-torch-temporal-admintools-7d7c9db48-sktcj",
    container="admin-tools"
  }[5m])
)
/
on (cluster, namespace, pod, container)
sum by (cluster, namespace, pod, container) (
  rate(container_cpu_cfs_periods_total{
    cluster="db-prd-0c2k-fic-hybrid-cloud-de-dcb-cluster-01",
    namespace="torch",
    pod="nexuspay-dcb-cluster-torch-temporal-admintools-7d7c9db48-sktcj",
    container="admin-tools"
  }[5m])
)
```

Результат `35` здесь означает, что примерно 35% учтённых CFS-периодов были периодами с throttling. Это **не** означает, что контейнер использовал 35% CPU.

**5. Время throttling, мс на секунду**

```promql
1000 *
sum by (cluster, namespace, pod, container) (
  rate(container_cpu_cfs_throttled_seconds_total{
    cluster="db-prd-0c2k-fic-hybrid-cloud-de-dcb-cluster-01",
    namespace="torch",
    pod="nexuspay-dcb-cluster-torch-temporal-admintools-7d7c9db48-sktcj",
    container="admin-tools"
  }[5m])
)
```

Эта метрика поможет понять, велико ли само время ограничения: доля throttled periods показывает, **как часто** оно происходило, а этот запрос — накопленное время throttling. cAdvisor описывает отдельные счётчики для числа throttled periods и общей длительности throttling. 

Если в запросах появляется `No data`, проверьте, доступны ли эти метрики и совпадают ли labels `cluster`, `pod` и `container` с теми, что реально хранятся в VictoriaMetrics. URL запроса алерта на скриншоте обрезан, поэтому точную формулу самого алерта по нему восстановить нельзя.
