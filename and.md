---
---
---
```text
У одного из проектов компании есть несколько серверов в разных локациях. Всего серверов где-то примерно половина тысячи, то есть где-то 500 серверов. Это сервера на операционной системе Red Hat, либо RHEL 8, либо RHEL 9. Но вся штука заключается в том, что я подозреваю эту среду в неконсистентности, которая может приводить к неожиданному поведению. И в этой связи я хочу разработать инструмент, который позволил бы мне наверняка утверждать, в каком именно состоянии находятся каждый из серверов. На каждом из серверов установлен Kubernetes, либо в роли control plane, либо в роли worker. И мне нужно, чтобы ты подумал и сказал, на что бы ты смотрел в первую очередь, заходя на сервера, какую ты информацию посмотрел, какую информацию, каким образом бы ты проверял.
```

---
---
---

Да. Я бы здесь **не начинал с Kubernetes**. Если подозрение именно в неконсистентности 500 RHEL-серверов, я бы рассматривал каждый сервер как объект, для которого нужно собрать **машинно-сравнимый baseline**: OS → kernel → packages → systemd → network → security → storage → time → Kubernetes → hardware/runtime.

Главная идея: не пытаться определить «здоров ли сервер» одной пачкой команд. Нужно построить **инвентаризацию фактического состояния**, а затем сравнивать серверы между собой и с эталоном.

Причём я бы собирал не только значения, но и **provenance**: откуда получено значение, когда, какой командой, с каким exit code.

---

# 1. Сначала определить, что именно мы считаем «одинаковым»

Я бы ввёл несколько уровней консистентности.

### Level 1 — Identity

* hostname
* FQDN
* IP addresses
* MAC addresses
* UUID
* machine-id
* location / datacenter
* роль: control-plane / worker
* cluster ID
* environment

### Level 2 — OS

* RHEL 8/9
* minor release
* kernel
* architecture
* booted kernel
* installed kernels
* systemd version
* SELinux
* systemd target
* timezone
* locale

### Level 3 — Software

* весь RPM inventory
* версии критических RPM
* repositories
* enabled modules
* package exclusions
* package locks
* GPG verification
* packages с изменёнными конфигами

### Level 4 — Configuration

* `/etc/sysctl*`
* `/etc/security/*`
* `/etc/systemd/*`
* NetworkManager
* firewalld
* chrony
* SSH
* limits
* udev
* mount configuration
* kernel cmdline

### Level 5 — Kubernetes node

* kubelet
* kubeadm, если используется
* kubectl
* container runtime
* CNI
* kubelet config
* kubelet arguments
* node labels
* taints
* allocatable resources
* conditions
* kernel/runtime requirements

### Level 6 — Runtime state

* running processes
* failed systemd units
* CPU/memory pressure
* disk pressure
* inode exhaustion
* network errors
* kernel errors
* OOM
* filesystem errors
* Kubernetes events

И вот это уже даст действительно интересную картину.

---

# 2. Самое первое, что я бы проверил на RHEL

Начал бы буквально с этого:

```bash
cat /etc/redhat-release
cat /etc/os-release
uname -a
uname -r
uname -m
hostnamectl
```

Но для инструмента я бы не парсил `redhat-release`, а собирал структурированные значения:

```bash
rpm -q redhat-release
rpm -q kernel
```

и:

```bash
uname -r
```

Особенно важна разница:

**installed kernel ≠ running kernel**

Например:

```text
installed:
  5.14.0-503.19.1.el9_5
  5.14.0-503.14.1.el9_5

running:
  5.14.0-503.14.1.el9_5
```

Это абсолютно реальная ситуация после обновления без reboot.

Для 500 машин это может оказаться одним из первых источников неконсистентности.

---

# 3. RPM inventory — для меня это один из главных источников истины

Я бы обязательно собирал **полный список RPM**:

```bash
rpm -qa --qf '%{NAME}\t%{EPOCHNUM}\t%{VERSION}\t%{RELEASE}\t%{ARCH}\n'
```

И отдельно:

```bash
dnf history
```

и информацию о repositories:

```bash
dnf repolist --enabled
```

Это позволит сделать очень интересную вещь:

> не просто сказать «сервер отличается», а сказать **чем именно**.

Например:

```text
Cluster: production-eu
Role: worker

428 nodes: containerd 1.7.18
17 nodes: containerd 1.7.19
8 nodes: containerd 1.7.16
3 nodes: unknown
```

А дальше:

```text
kernel:
  421 × 5.14.0-503.21.1.el9_5
   52 × 5.14.0-503.19.1.el9_5
   19 × 5.14.0-503.14.1.el9_5
    8 × ...
```

Это уже очень мощный инструмент диагностики.

---

# 4. Я бы обязательно проверял repositories

Потому что две машины могут выглядеть одинаково сейчас, но иметь **разные источники пакетов**.

Например:

```bash
dnf repolist --enabled
```

плюс:

```bash
grep -Rhv '^[[:space:]]*#' /etc/yum.repos.d/
```

И отдельно:

```bash
dnf config-manager --dump
```

если доступно.

Для каждого сервера я бы получал не просто список repository ID, но примерно:

```json
{
  "repo": "rhel-9-for-x86_64-baseos-rpms",
  "enabled": true,
  "baseurl": "...",
  "priority": "...",
  "gpgcheck": true
}
```

Особенно интересуют:

* разные BaseOS/AppStream
* разные внутренние mirrors
* разные snapshots
* разные release locks
* RHSM/RHUI
* случайно включённые дополнительные repositories.

Red Hat прямо предусматривает возможность фиксировать release version через `subscription-manager release --set`; поэтому состояние release lock тоже имеет смысл считать частью baseline. ([Red Hat Documentation][1])

---

# 5. Очень важная вещь — SELinux

Я бы проверял:

```bash
getenforce
sestatus
```

и:

```bash
getenforce
```

Но не ограничивался бы режимом.

Проверил бы:

```bash
semanage boolean -l
```

и наличие AVC:

```bash
ausearch -m AVC -ts recent
```

либо соответствующие audit logs.

Потому что:

```text
Server A:
SELinux = Enforcing
AVC = 0

Server B:
SELinux = Enforcing
AVC = 1847
```

формально оба «Enforcing», но фактически среда совершенно разная.

Red Hat указывает, что SELinux decisions и AVC denial записываются в audit log, поэтому это особенно полезный источник runtime-состояния. ([Red Hat Documentation][2])

---

# 6. systemd

Следующий огромный пласт:

```bash
systemctl --failed
```

Это я бы считал обязательной метрикой.

Но ещё:

```bash
systemctl list-units --type=service --state=running
systemctl list-unit-files --state=enabled
```

И отдельно интересующие Kubernetes/RHEL services:

```bash
systemctl status kubelet
systemctl status containerd
systemctl status crio
systemctl status NetworkManager
systemctl status firewalld
systemctl status chronyd
systemctl status tuned
systemctl status auditd
systemctl status kdump
```

Не нужно предполагать, что у всех одинаковый runtime.

Например, если:

```text
492 × containerd
8 × CRI-O
```

это должно сразу попасть в report.

---

# 7. sysctl — я бы уделил этому очень много внимания

Вот это потенциально **золотая жила**.

Собрать:

```bash
sysctl -a
```

и сравнивать значения.

Особенно:

```text
net.*
vm.*
fs.*
kernel.*
user.*
```

Для Kubernetes особенно интересны:

```text
net.ipv4.ip_forward
net.bridge.bridge-nf-call-iptables
net.bridge.bridge-nf-call-ip6tables
net.ipv4.conf.*
net.ipv6.conf.*
fs.inotify.*
fs.file-max
vm.max_map_count
```

Но я бы не делал hardcoded список сразу.

Лучше:

> собрать весь sysctl → нормализовать → хранить → статистически найти deviations → потом определить policy.

То есть инструмент сначала **наблюдает**, а не утверждает, что какое-то значение «правильное».

---

# 8. Kernel command line

Очень важная проверка:

```bash
cat /proc/cmdline
```

Потому что можно иметь:

```text
kernel = одинаковый
```

но:

```text
node A:
... intel_iommu=on ...

node B:
... intel_iommu=off ...
```

или:

```text
mitigations=off
transparent_hugepage=never
nohz_full=...
isolcpus=...
```

и это потенциально уже объясняет различия в поведении.

Я бы отдельно парсил kernel parameters в key/value структуру.

---

# 9. CPU / memory

Я бы собирал:

```bash
lscpu
cat /proc/meminfo
numactl --hardware
```

и:

```bash
free -h
```

Но важно не только количество CPU.

Например:

```text
CPU model
CPU flags
sockets
cores
threads
NUMA nodes
CPU frequency
```

Особенно если это физические машины.

Два сервера могут оба иметь:

```text
64 CPU
256 GB RAM
```

но один:

```text
2 NUMA nodes
```

а другой:

```text
4 NUMA nodes
```

и Kubernetes workload будет вести себя иначе.

---

# 10. CPU flags

Я бы обязательно сохранял:

```bash
lscpu
```

в частности:

```text
Architecture
CPU op-mode
Vendor ID
Model name
CPU(s)
Thread(s) per core
Core(s) per socket
Socket(s)
NUMA node(s)
Flags
```

Потому что некоторые приложения очень чувствительны к CPU features.

---

# 11. Storage

Я бы собирал:

```bash
lsblk -o NAME,KNAME,TYPE,SIZE,FSTYPE,MOUNTPOINTS,UUID,ROTA,MODEL
```

```bash
df -hT
```

```bash
df -ih
```

и:

```bash
mount
```

Особенно:

* filesystem type
* mount options
* XFS/ext4
* `noatime`
* `discard`
* `nodev`
* `nosuid`
* `noexec`
* отдельный `/var`
* отдельный `/var/lib/containerd`
* отдельный `/var/lib/kubelet`

И отдельно:

```bash
findmnt -R /
```

Я бы **обязательно проверял inode utilization**, а не только disk space.

Например:

```text
/var:
78% disk
99.8% inodes
```

может быть гораздо интереснее.

---

# 12. Network — здесь я бы копал очень глубоко

Собирал бы:

```bash
ip -br addr
ip -br link
ip route
ip -6 route
```

и:

```bash
nmcli device
nmcli connection show
```

Плюс:

```bash
ethtool <interface>
ethtool -k <interface>
```

и:

```bash
ip -s link
```

Мне особенно интересны:

* MTU
* interface speed
* duplex
* offloading
* RX/TX errors
* dropped packets
* VLAN
* bonding
* routes
* default gateway
* DNS
* NetworkManager profiles.

Например, внезапно обнаруживается:

```text
497 nodes MTU 1500
3 nodes MTU 9000
```

И вот это уже потенциальная причина очень странных проблем Kubernetes/networking.

---

# 13. DNS

Отдельный тест.

Не просто конфигурация:

```bash
cat /etc/resolv.conf
```

а **функциональная проверка**:

```bash
getent hosts <internal-service>
getent hosts <some-known-internal-name>
```

И:

```bash
resolvectl status
```

если используется соответствующий resolver.

Я бы сохранял:

```text
nameservers
search domains
options
DNS resolution latency
```

Потому что конфигурация одинаковая, а фактический DNS path может отличаться.

---

# 14. Time synchronization

Для Kubernetes это очень важно.

Проверять:

```bash
timedatectl
chronyc tracking
chronyc sources -v
```

Я бы записывал:

```text
synchronized
stratum
offset
reference server
source state
```

И отдельно искал:

```text
offset > X ms
```

Это уже не configuration drift, а **runtime health**.

---

# 15. Firewall

Проверить:

```bash
firewall-cmd --state
firewall-cmd --get-active-zones
firewall-cmd --list-all-zones
```

Но опять же — не только state.

Нужно сравнивать:

* zones
* interfaces
* sources
* ports
* services
* rich rules
* direct rules
* nftables rules.

Потому что:

```text
firewalld = running
```

ничего не говорит о том, **что именно он разрешает**.

---

# 16. Kubernetes — здесь уже второй большой слой

После OS baseline я бы перешёл к Kubernetes.

Первое:

```bash
kubelet --version
```

и:

```bash
crictl version
```

если `crictl` установлен.

Потом:

```bash
systemctl cat kubelet
```

и:

```bash
ps auxww | grep kubelet
```

Нужно понять:

* откуда kubelet получает config
* какие flags
* какой config file
* какие feature gates
* какой authentication
* какой cgroup driver
* какой CRI endpoint.

---

# 17. Kubernetes node state

С control plane:

```bash
kubectl get nodes -o wide
```

и:

```bash
kubectl describe node <node>
```

Я бы автоматически собирал:

```text
Kubernetes version
OS Image
Kernel Version
Container Runtime Version
Kubelet Version
Kube-Proxy Version
Architecture
Capacity
Allocatable
Conditions
Taints
Labels
Addresses
```

Kubernetes сам показывает container runtime и его версию в node information. ([Kubernetes][3])

---

# 18. Version skew — автоматическая проверка

Это я бы сделал отдельным validator'ом.

Например:

```text
apiserver: 1.35.3
kubelet:   1.34.7
```

→ OK.

А:

```text
apiserver: 1.35.3
kubelet:   1.31.9
```

→ WARNING / FAIL согласно вашей policy.

Текущая Kubernetes version-skew policy допускает kubelet до трёх minor versions старше kube-apiserver, но kubelet не должен быть новее apiserver. ([Kubernetes][4])

При этом я бы **не зашивал эту policy в сам collector**. Collector только собирает факты:

```json
{
  "kubelet": "1.34.7",
  "apiserver": "1.35.3"
}
```

А validator решает:

```json
{
  "check": "kubernetes_version_skew",
  "status": "PASS"
}
```

Это очень важное архитектурное разделение.

---

# 19. Container runtime

Нужно определить:

```bash
crictl info
```

и:

```bash
systemctl status containerd
```

или CRI-O.

Для containerd:

```bash
containerd --version
```

и configuration:

```bash
containerd config dump
```

если допустимо.

Я бы сравнивал:

* runtime version
* CRI endpoint
* cgroup configuration
* snapshotter
* registry configuration
* mirrors
* insecure registries
* sandbox image
* runtime plugins.

Это особенно важно: начиная с Kubernetes 1.26 kubelet требует CRI v1 от runtime. ([Kubernetes][5])

---

# 20. cgroups — обязательная проверка

Вот это я бы выделил красным.

Проверить:

```bash
stat -fc %T /sys/fs/cgroup/
```

и:

```bash
mount | grep cgroup
```

и определить:

```text
cgroup v1
cgroup v2
```

Плюс:

```bash
systemctl show kubelet | grep -i cgroup
```

и runtime configuration.

Для Kubernetes-кластера я бы хотел видеть **единый policy** относительно cgroup version и driver.

---

# 21. CNI

Вот здесь я бы обязательно пошёл из Kubernetes в OS.

Сначала:

```bash
kubectl get pods -A -o wide
```

и определить CNI.

Например:

```text
calico
cilium
flannel
ovn-kubernetes
```

Дальше на node:

```bash
ip link
ip route
iptables-save
nft list ruleset
```

В зависимости от используемого CNI.

Очень часто проблема выглядит как:

> Kubernetes configuration одинаковая, но underlying Linux networking отличается.

Именно поэтому нельзя ограничиваться `kubectl`.

---

# 22. iptables / nftables

На RHEL 8/9 я бы специально проверял, что именно используется:

```bash
iptables --version
nft --version
```

и:

```bash
alternatives --display iptables
```

если механизм alternatives используется.

А затем:

```bash
iptables-save
```

и/или:

```bash
nft list ruleset
```

Потому что одинаковый:

```text
iptables --version
```

ещё не означает одинаковое фактическое состояние firewall/network stack.

---

# 23. Kubelet configuration

Я бы вытаскивал именно **effective configuration**, а не только файл.

Искал бы:

```text
cgroupDriver
containerRuntimeEndpoint
clusterDNS
clusterDomain
maxPods
podsPerCore
kubeReserved
systemReserved
evictionHard
evictionSoft
evictionSoftGracePeriod
imageGCHighThresholdPercent
imageGCLowThresholdPercent
serializeImagePulls
cpuManagerPolicy
topologyManagerPolicy
```

Это потенциально огромный источник drift.

Например:

```text
499 nodes:
  evictionHard.memory.available = 100Mi

1 node:
  evictionHard.memory.available = 500Mi
```

Такой сервер может вести себя иначе, хотя Kubernetes формально «работает».

---

# 24. Kubernetes labels и taints

Проверял бы:

```bash
kubectl get nodes --show-labels
```

и:

```bash
kubectl get nodes -o json
```

Автоматически искать:

* missing labels
* unexpected labels
* different taints
* different roles
* topology labels
* architecture
* hostname mismatch.

---

# 25. Resource capacity

Очень полезно сравнивать:

```text
capacity
allocatable
```

для:

* CPU
* memory
* pods
* ephemeral-storage
* hugepages
* GPUs, если есть.

Например:

```text
same hardware:
  capacity = 64 CPU
  allocatable = 60 CPU

other:
  capacity = 64 CPU
  allocatable = 48 CPU
```

И тогда надо выяснять `kubeReserved/systemReserved`.

---

# 26. HugePages / CPU Manager / NUMA

Если это production infrastructure, я бы обязательно собирал:

```bash
grep -i huge /proc/meminfo
```

и:

```bash
cat /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor
```

а также Kubernetes:

```text
cpuManagerPolicy
topologyManagerPolicy
```

Особенно если workload latency-sensitive.

---

# 27. Kernel / system errors

Я бы добавил отдельный блок:

```bash
journalctl -p warning..alert --since "24 hours ago"
```

и:

```bash
dmesg --level=err,warn
```

И искать:

```text
OOM
I/O error
EXT4
XFS
NETDEV WATCHDOG
soft lockup
hard lockup
hung task
segfault
memory failure
hardware error
MCE
```

Это уже не configuration drift, но очень важно для ответа на вопрос:

> «Почему именно этот сервер ведёт себя странно?»

---

# 28. OOM

Отдельная метрика:

```bash
journalctl -k | grep -i oom
```

и желательно структурированный поиск kernel events.

Например:

```text
node-147:
  OOM events: 17

node-148:
  OOM events: 0
```

При этом оба могут иметь:

```text
MemoryPressure=False
```

прямо сейчас.

---

# 29. Failed boots / reboot history

Я бы собирал:

```bash
uptime
last reboot
who -b
```

и:

```bash
journalctl --list-boots
```

Интересная аномалия:

```text
499 servers rebooted within 30 days
1 server uptime = 418 days
```

Это может объяснить огромное количество различий.

---

# 30. Очень интересная вещь: installed vs configured vs running

Я бы для каждого компонента разделил три понятия.

Например kernel:

```text
installed_version
running_version
configured_default
```

Containerd:

```text
installed_version
binary_version
running_process_version
systemd_unit
config
```

Kubelet:

```text
RPM version
binary version
systemd unit
process arguments
effective config
API-reported version
```

Это гораздо сильнее обычного inventory.

---

# 31. Я бы ещё проверял конфигурационные файлы на изменения

RPM позволяет определить altered configuration.

Например:

```bash
rpm -Va
```

Это очень интересный источник.

Но **не стоит просто считать любой вывод `rpm -Va` ошибкой**.

Например, конфигурационные файлы намеренно изменены.

Я бы классифицировал:

```text
M = mode
5 = checksum
S = size
T = mtime
U = user
G = group
```

и отдельно смотрел:

```text
configuration drift
binary drift
permission drift
```

---

# 32. И обязательно `/etc`

Я бы собрал fingerprints для наиболее важных конфигураций:

```text
/etc/ssh/
/etc/sysctl.d/
/etc/systemd/
/etc/NetworkManager/
/etc/chrony.conf
/etc/containers/
/etc/containerd/
/etc/kubernetes/
/etc/cni/
/etc/yum.repos.d/
/etc/selinux/
/etc/security/
```

Но **не отправлял бы содержимое секретов**.

Лучше:

```text
SHA256(file)
size
mtime
permissions
owner
```

Например:

```text
/etc/containerd/config.toml
sha256: abc123...
```

И сравнивать hash.

---

# 33. Очень важный момент — секреты

Я бы проектировал collector так, чтобы он **по умолчанию не собирал**:

```text
/etc/shadow
private keys
kubeconfig credentials
TLS private keys
tokens
passwords
cloud credentials
```

Для конфигурации:

```text
content → sanitize → hash
```

или:

```text
parse only safe fields
```

Это важно, если потом результаты будут уходить в центральную БД/Elastic/Grafana.

---

# 34. Как я бы построил сам инструмент

Я бы сделал примерно такую архитектуру:

```text
                    ┌──────────────────┐
                    │   Control Plane   │
                    │                  │
                    │ PostgreSQL       │
                    │ API              │
                    │ Rules engine     │
                    └────────┬─────────┘
                             │
                   SSH / Ansible / Agent
                             │
          ┌──────────────────┼──────────────────┐
          │                  │                  │
       server-001         server-002         server-500
          │                  │                  │
      collector           collector           collector
          │                  │                  │
      JSON facts           JSON facts          JSON facts
```

Но collector я бы сделал **максимально тупым**.

Он только говорит:

> Вот что я вижу.

Например:

```json
{
  "host": {
    "hostname": "node-001",
    "os": {
      "family": "rhel",
      "major": 9,
      "minor": "5"
    },
    "kernel": {
      "running": "5.14.0-503.19.1.el9_5"
    }
  }
}
```

И отдельно:

```json
{
  "kubernetes": {
    "kubelet": "1.34.7",
    "runtime": {
      "name": "containerd",
      "version": "1.7.18"
    }
  }
}
```

---

# 35. А validation engine — отдельно

Вот это уже самое интересное.

Например правило:

```yaml
- name: rhel_major_version
  scope: cluster:production
  expected: "9"

- name: kernel
  scope: role:worker
  expected:
    one_of:
      - "5.14.0-503.19.1.el9_5"
      - "5.14.0-503.21.1.el9_5"

- name: selinux
  expected: enforcing

- name: cgroup
  expected: v2

- name: container_runtime
  expected:
    name: containerd

- name: kubelet_version
  policy: kubernetes-version-skew
```

И результат:

```text
node-001   HEALTHY
node-002   WARNING
node-003   CRITICAL
```

Но самое главное — **почему**:

```text
node-003

CRITICAL
├── RHEL 9.4 instead of 9.5
├── running kernel differs from fleet
├── containerd version differs
├── cgroup configuration differs
├── MTU = 9000 instead of 1500
├── SELinux has 312 AVC denials
├── 3 failed systemd units
└── kubelet 2 minor versions behind
```

Вот такой результат уже реально полезен.

---

# 36. Я бы добавил statistical drift detection

Это, на мой взгляд, самая интересная часть проекта.

Не всё нужно задавать руками.

Допустим, collector получил 500 серверов.

Он видит:

```text
kernel
487 × A
 11 × B
  2 × C
```

или:

```text
containerd
498 × 1.7.18
  2 × 1.7.15
```

или:

```text
sysctl vm.max_map_count
499 × 1048576
  1 × 65530
```

Инструмент автоматически говорит:

> **Outlier**

Это ещё не означает «ошибка».

Но означает:

> **проверь этот сервер.**

Именно это позволит тебе обнаруживать вещи, о существовании которых ты пока даже не знаешь.

---

# 37. Я бы сделал три статуса, а не PASS/FAIL

Например:

### NORMAL

Совпадает с baseline.

### OUTLIER

Отличается от большинства.

### POLICY VIOLATION

Отличается от официально установленного требования.

Это принципиально разные вещи.

Например:

```text
node-17:
RHEL 9.5

Fleet:
RHEL 9.4 — 498 nodes
RHEL 9.5 — 2 nodes
```

Это **outlier**, но необязательно нарушение.

А:

```text
SELinux = permissive
Policy = enforcing
```

→ **policy violation**.

---

# 38. И ещё один очень важный уровень — topology

Я бы обязательно добавлял в inventory:

```text
datacenter
rack
availability zone
cluster
control-plane / worker
hardware model
VM host
network segment
```

Потому что потом можно обнаружить:

```text
All 7 problematic nodes are in DC-3.
```

или:

```text
All nodes with packet loss are running on hypervisor pool B.
```

Это уже позволяет выйти за пределы Linux/Kubernetes и обнаружить инфраструктурную причину.

---

# 39. Если бы я заходил на первый сервер прямо сейчас

Я бы не выполнял 100 команд вручную.

Я бы сначала сделал небольшой read-only collector и получил примерно такие группы данных:

```text
IDENTITY
OS
KERNEL
CPU
MEMORY
NUMA
STORAGE
NETWORK
DNS
TIME
SELINUX
FIREWALL
SYSTEMD
SYSCTL
RPM
REPOSITORIES
MOUNTS
KERNEL_CMDLINE
KUBELET
KUBERNETES
CONTAINER_RUNTIME
CNI
CGROUP
RUNTIME_HEALTH
ERRORS
```

И результат — **один JSON на сервер**.

Например:

```text
inventory/
├── node-001.json
├── node-002.json
├── node-003.json
└── ...
```

После этого уже можно делать:

```text
JSON → PostgreSQL
       ↓
   normalization
       ↓
   comparison
       ↓
   policy engine
       ↓
   Grafana / Web UI
```

---

## Что я считаю наиболее приоритетным

Если не хочется сразу строить огромную систему, я бы начал с **20–30 проверок**, которые дадут максимальную отдачу:

| Приоритет | Что                              |
| --------- | -------------------------------- |
| 🔴        | RHEL major/minor                 |
| 🔴        | running vs installed kernel      |
| 🔴        | RPM versions                     |
| 🔴        | enabled repositories             |
| 🔴        | kernel cmdline                   |
| 🔴        | SELinux mode                     |
| 🔴        | sysctl                           |
| 🔴        | cgroup v1/v2                     |
| 🔴        | kubelet version/config           |
| 🔴        | Kubernetes version skew          |
| 🔴        | container runtime/version/config |
| 🔴        | CNI                              |
| 🔴        | IP/routes/MTU                    |
| 🔴        | iptables/nftables                |
| 🔴        | failed systemd units             |
| 🔴        | filesystem/inode usage           |
| 🟠        | chrony/time offset               |
| 🟠        | CPU/NUMA                         |
| 🟠        | memory                           |
| 🟠        | mounts/options                   |
| 🟠        | NIC errors/drops                 |
| 🟠        | OOM/kernel errors                |
| 🟠        | reboot/uptime                    |
| 🟡        | RPM verification                 |
| 🟡        | config file hashes               |
| 🟡        | hardware/firmware                |
| 🟡        | kdump                            |
| 🟡        | tuned/profile                    |

Red Hat, например, отдельно рекомендует проверять работоспособность `kdump`, а не просто наличие пакета/сервиса, поэтому подобные вещи я бы тоже относил к health checks, а не к простому inventory. ([Red Hat Documentation][6])

---

### И главное

Я бы **не пытался определить заранее, какие именно параметры являются причиной проблемы**.

Сначала сделал бы **полную read-only телеметрию состояния 500 машин**, причём максимально нормализованную.

Потом на этой выборке можно буквально посмотреть:

```text
500 servers
   ↓
~3000 facts/server
   ↓
1.5 million facts
   ↓
distribution / clustering / outliers
```

И очень вероятно, что после этого ты увидишь что-то вроде:

> «Ага. 23 машины, которые периодически имеют проблему X, отличаются от остальных только kernel cmdline + MTU + версия containerd».

И это будет гораздо сильнее, чем заранее составленный человеком checklist.

**Если бы это был мой проект, я бы именно с этого и начал: сначала collector + нормальная модель данных, затем statistical diff, и только потом policy engine.** Это позволит не потерять неизвестные тебе сейчас источники неконсистентности.

[1]: https://docs.redhat.com/de/documentation/red_hat_enterprise_linux/9/html/upgrading_from_rhel_8_to_rhel_9/assembly_preparing-for-the-upgrade_upgrading-from-rhel-8-to-rhel-9?utm_source=chatgpt.com "Chapter 3. Preparing for the upgrade | Upgrading from RHEL 8 to RHEL 9 | Red Hat Enterprise Linux | 9 | Red Hat Documentation"
[2]: https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/8/html-single/using_selinux/using_selinux?utm_source=chatgpt.com "Using SELinux | Red Hat Enterprise Linux | 8 | Red Hat Documentation"
[3]: https://kubernetes.io/docs/tasks/administer-cluster/migrating-from-dockershim/find-out-runtime-you-use/?utm_source=chatgpt.com "Find Out What Container Runtime is Used on a Node | Kubernetes"
[4]: https://kubernetes.io/releases/version-skew-policy/?utm_source=chatgpt.com "Version Skew Policy | Kubernetes"
[5]: https://kubernetes.io/docs/concepts/containers/cri/?utm_source=chatgpt.com "Container Runtime Interface (CRI) | Kubernetes"
[6]: https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html-single/managing_monitoring_and_updating_the_kernel/managing_monitoring_and_updating_the_kernel?utm_source=chatgpt.com "Managing, monitoring, and updating the kernel | Red Hat Enterprise Linux | 9 | Red Hat Documentation"



---
---
---


```text
А как собирать такого рода информацию "по взрослому" не в примерах, а в концепциях в экосистемах подобных решений?
```


---
---
---

Да. И здесь есть важный сдвиг в мышлении: **в enterprise это обычно не называют “набором bash-команд для проверки серверов”**. Это уже давно отдельный класс систем — **configuration management, asset/inventory management, compliance, configuration drift detection и observability**.

Если смотреть на зрелые экосистемы, то я бы разделил задачу на несколько концепций.

## 1. Первая концепция — Facts / Inventory

Это фундамент.

У тебя есть **Node/Host**, а про него система постоянно собирает *facts*:

```text
Node
 ├── identity
 ├── OS
 ├── hardware
 ├── networking
 ├── packages
 ├── services
 ├── kernel
 ├── security
 └── runtime
```

Именно так работает, например, **Puppet Facter**. Facter — отдельная система inventory, которая собирает факты о машине; Puppet может использовать стандартные и custom facts, а PuppetDB позволяет искать и анализировать эти данные по инфраструктуре. ([help.puppet.com][1])

То есть концептуально:

> **Host → Fact collection → Fact database**

а не:

> Host → `ssh` → 50 команд → текстовый output.

Это первая вещь, которую я бы взял из Puppet.

---

# 2. Вторая концепция — Desired State

Следующий уровень — уже не просто:

> «Что сейчас находится на сервере?»

а:

> **«Каким этот сервер должен быть?»**

Например:

```text
                    Desired State
                         │
                         ▼
                 ┌───────────────┐
                 │ Worker RHEL 9 │
                 └───────┬───────┘
                         │
       ┌─────────────────┼──────────────────┐
       ▼                 ▼                  ▼
   OS policy         Kubernetes          Security
       │                 │                  │
   RHEL 9.5          kubelet X          SELinux enforcing
   kernel Y          runtime Z          firewall ...
```

И дальше:

```text
desired state
      │
      ▼
actual state
      │
      ▼
    diff
```

Вот это уже **configuration drift**.

Puppet, Rudder и аналогичные системы строятся именно вокруг этой модели. Например, Rudder может работать исключительно в Audit Mode — только обнаруживать расхождения — либо в Enforce Mode, когда система пытается исправить drift. ([docs.rudder.io][2])

Для твоей задачи я бы на первом этапе выбрал **Audit-only**.

---

# 3. Третья концепция — Configuration Management

Здесь находятся:

* Puppet
* Ansible
* Chef
* Salt
* Rudder

Их историческая задача:

> **привести инфраструктуру к нужному состоянию.**

Но у них есть очень полезная для тебя часть — модель:

```text
inventory
    +
facts
    +
policy
    ↓
compliance
```

Например Puppet собирает facts до компиляции configuration catalog. ([help.puppet.com][3])

А Rudder прямо моделирует:

```text
Node
   ↓
Group
   ↓
Rule
   ↓
Directive
   ↓
Expected state
   ↓
Actual state
   ↓
Compliance
```

и хранит историю drift/compliance. ([docs.rudder.io][2])

---

# 4. Четвёртая концепция — Compliance Engine

Это уже ещё интереснее.

Ты не обязательно говоришь:

> kernel должен быть `5.14.0-xxx`.

Ты можешь сказать:

```text
Rule:
RHEL worker

WHEN:
    role == worker
    AND os.major == 9

EXPECT:
    selinux == enforcing
    cgroup == v2
    chrony == synchronized
    container_runtime == approved
    kubelet_version satisfies policy
```

То есть **policy-as-code**.

Получается:

```text
                  Facts
                    │
                    ▼
              Policy Engine
                    │
          ┌─────────┴─────────┐
          ▼                   ▼
       COMPLIANT           NON-COMPLIANT
                              │
                              ▼
                           DETAILS
```

Это намного масштабируемее, чем hardcoded:

```python
if kernel != "...":
    alert()
```

---

# 5. Отдельный класс — Security Compliance

Здесь уже появляются:

* CIS Benchmarks
* OpenSCAP
* NIST
* DISA STIG
* SCAP Security Guide
* Puppet Security Compliance
* Red Hat Insights Compliance

Это немного другой мир.

Он отвечает на:

> «Соответствует ли машина определённому стандарту безопасности?»

Например:

```text
CIS RHEL 9
    ↓
200+ controls
    ↓
PASS / FAIL / NOT APPLICABLE
```

Puppet Security Compliance Management, например, интегрирует CIS assessor с Puppet и позволяет запускать CIS scans и custom profiles. ([help.puppet.com][4])

Но я бы **не смешивал security compliance с твоей основной задачей**.

Тебе нужен более широкий:

> **Infrastructure State Compliance**

а security compliance — только один из его domains.

---

# 6. Ещё один очень интересный подход — osquery

Вот здесь я бы особенно внимательно посмотрел на **osquery**.

Концепция совсем другая:

> превратить операционную систему в **SQL database**.

То есть вместо:

```bash
rpm -qa
ip addr
ps aux
mount
systemctl ...
```

ты концептуально спрашиваешь:

```sql
SELECT * FROM rpm_packages;
```

или:

```sql
SELECT * FROM interfaces;
```

или:

```sql
SELECT * FROM mounts;
```

или:

```sql
SELECT * FROM processes;
```

Fleet, например, использует osquery tables для получения OS, hardware и software information. ([fleetdm.com][5])

Это очень интересная модель для твоего проекта:

```text
                  Linux host
                      │
                   osquery
                      │
              ┌───────┴────────┐
              │                │
         local tables       scheduled
              │              queries
              └───────┬────────┘
                      ▼
                central DB
```

И затем:

```sql
SELECT hostname, kernel_version
FROM system_info
WHERE ...
```

---

# 7. И вот здесь появляется важное разделение

Я бы **не пытался найти один продукт, который делает всё**.

В зрелой инфраструктуре обычно существуют отдельные слои:

```text
                         ┌─────────────────────┐
                         │      Dashboard      │
                         └──────────┬──────────┘
                                    │
                         ┌──────────▼──────────┐
                         │ Compliance / Policy │
                         └──────────┬──────────┘
                                    │
                    ┌───────────────┴───────────────┐
                    │                               │
             Configuration                    Security
             Management                       Compliance
                    │                               │
              Puppet/Ansible                    SCAP/CIS
                    │
                    ▼
              Fact / Inventory
                    │
             Facter / osquery
                    │
                    ▼
                  HOSTS
```

И это, на мой взгляд, **самая важная архитектурная идея для твоего проекта**.

---

# 8. А где здесь Ansible?

Ansible я бы рассматривал немного иначе.

Он прекрасен как:

> **orchestration / execution layer**

Например:

```text
500 hosts
   ↓
Ansible
   ↓
collect facts
   ↓
execute checks
```

Ansible фактически может быть транспортом/механизмом исполнения.

Но я бы не делал Ansible единственной моделью данных.

То есть:

```text
Ansible
    ↓
collection
    ↓
normalized facts
    ↓
database
```

а не:

```text
Ansible output
    ↓
grep
    ↓
some report
```

---

# 9. Puppet Bolt — интересный промежуточный вариант

Есть ещё довольно красивая концепция у Puppet — **Bolt**.

Bolt умеет иметь inventory с группами машин и запускать задачи/планы на группах targets. ([help.puppet.com][6])

То есть можно представить:

```text
inventory

production
├── control-plane
│   ├── node1
│   ├── node2
│   └── ...
│
└── workers
    ├── node100
    ├── node101
    └── ...
```

И выполнять одинаковые операции над группами.

Но Bolt — это скорее **orchestration**, а не полноценная система continuous state/compliance.

---

# 10. Rudder — пожалуй, наиболее близкий conceptual match

Если посмотреть именно на твою задачу, то **Rudder очень близок концептуально**.

Потому что у него одновременно есть:

```text
inventory
facts
groups
policies
compliance
drift
audit
history
API
```

Причём он умеет именно:

> **Audit without modification**

и сообщает `compliant / non-compliant / error / not applicable`, а также хранит историю изменений compliance. ([docs.rudder.io][2])

Даже osquery можно использовать внутри Rudder как источник audit facts. ([docs.rudder.io][7])

---

# 11. Но для твоего случая я бы посмотрел ещё на Red Hat Insights

Поскольку у тебя **500 RHEL**, я бы обязательно исследовал Red Hat Insights как референсную архитектуру.

Это уже native ecosystem для RHEL:

```text
RHEL
  ↓
Insights client
  ↓
Red Hat
  ↓
Inventory / Advisor / Compliance / Vulnerability / Patch
```

И концептуально это гораздо ближе к тому, что тебе нужно, чем просто Ansible.

Но есть важное отличие:

**Insights отвечает прежде всего на Red Hat-centric вопросы.**

А тебе нужно что-то вроде:

```text
RHEL
+
Kubernetes
+
containerd
+
CNI
+
internal corporate configuration
+
network topology
+
company-specific policies
```

Поэтому Insights может быть **одним из источников**, но я бы не делал его моделью всего проекта.

---

# 12. Самое интересное: Asset Inventory ≠ Configuration Inventory ≠ Compliance

Я бы прямо разделил это в архитектуре.

### Asset inventory

> Что это за машина?

```text
hostname
serial
VM/physical
location
hardware
owner
cluster
role
```

### Configuration inventory

> Как она настроена?

```text
OS
kernel
packages
sysctl
systemd
network
mounts
Kubernetes
runtime
CNI
```

### Runtime state

> Что происходит прямо сейчас?

```text
CPU
memory
disk
OOM
errors
failed services
network errors
Kubernetes conditions
```

### Compliance

> Соответствует ли она требованиям?

```text
PASS
FAIL
WARNING
UNKNOWN
N/A
```

Это **четыре разных понятия**.

И если их смешать в одну таблицу `servers`, через полгода получится кошмар.

---

# 13. Я бы даже не называл твой будущий инструмент "monitoring"

Prometheus/Grafana здесь не являются основой.

Prometheus отлично отвечает:

> «Что происходит сейчас?»

Например:

```text
CPU = 73%
memory = 81%
packet drops = 12/s
```

А твой инструмент должен отвечать:

> **«Почему этот сервер отличается от остальных и соответствует ли он требуемому состоянию?»**

Это скорее:

**Configuration State Management / Infrastructure Compliance**

чем monitoring.

---

# 14. Если перенести это на твою задачу

Я бы строил концептуально вот так:

```text
                         ┌───────────────────────┐
                         │       Policies        │
                         │                       │
                         │ company rules         │
                         │ Kubernetes rules      │
                         │ RHEL rules            │
                         │ security rules        │
                         └───────────┬───────────┘
                                     │
                                     ▼
                           ┌──────────────────┐
                           │ Compliance Engine│
                           └────────┬─────────┘
                                    │
                                    │ compare
                                    ▼
┌──────────────┐          ┌──────────────────┐
│ RHEL hosts   │─────────►│ Normalized Facts │
│              │          └────────┬─────────┘
│ collector    │                   │
└──────────────┘                   │
                                   ▼
                           ┌──────────────────┐
                           │ Historical State │
                           └────────┬─────────┘
                                    │
                          ┌─────────┴─────────┐
                          ▼                   ▼
                     Drift detection      Analytics
                          │                   │
                          ▼                   ▼
                       Alerts             Dashboard
```

И вот **Normalized Facts + Historical State** я считаю сердцем системы.

---

# 15. И я бы взял идеи сразу из нескольких экосистем

| Система              | Что я бы у неё взял                     |
| -------------------- | --------------------------------------- |
| **Puppet/Facter**    | модель `node → facts`                   |
| **PuppetDB**         | централизованный поиск по facts         |
| **Ansible**          | orchestration/execution                 |
| **Rudder**           | audit → compliance → drift              |
| **osquery**          | SQL-модель для OS inventory             |
| **OpenSCAP**         | security/compliance model               |
| **Red Hat Insights** | RHEL-centric fleet inventory/advisor    |
| **Prometheus**       | time-series/runtime metrics             |
| **Grafana**          | visualization                           |
| **Kubernetes**       | desired state / actual state philosophy |

Причём я бы **не писал свой collector с нуля**, пока не изучил Facter + osquery + Ansible facts + Red Hat Insights.

---

## И есть одна архитектурная идея, которую я бы особенно рекомендовал

Не делай:

```text
check_kernel()
check_sysctl()
check_kubernetes()
check_network()
```

как основу программы.

Сделай **универсальную модель фактов**:

```text
Fact
 ├── subject: node-123
 ├── namespace: kernel
 ├── key: version
 ├── value: 5.14....
 ├── collected_at
 ├── collector
 ├── source
 └── confidence/status
```

А дальше уже поверх неё:

```text
Fact
   ↓
Policy
   ↓
Evaluation
   ↓
Compliance result
```

Тогда через год ты сможешь добавить хоть:

```text
kernel
sysctl
RPM
systemd
Kubernetes
containerd
CNI
firmware
BIOS
DNS
network
security
```

**без переписывания самой системы.**

Именно это, на мой взгляд, отличает «скрипт, который проверяет 500 серверов» от **нормальной enterprise fleet/configuration compliance платформы**.

Если бы я проектировал это для твоей компании, я бы следующим шагом разобрал **конкретно архитектуру такого собственного решения уровня 500–5000 RHEL nodes**: agent vs agentless, SSH/Ansible vs osquery, схема PostgreSQL, модель `Fact → Snapshot → Policy → Evaluation → Drift`, как хранить историю и как делать сравнение 500 серверов так, чтобы система не превратилась в самописный Puppet.

[1]: https://help.puppet.com/bolt/current/topics/glossary.htm?utm_source=chatgpt.com "Glossary | Bolt | 5.1.2"
[2]: https://docs.rudder.io/reference/9.1/usage/configuration_management.html?utm_source=chatgpt.com "Configuration concepts :: Rudder Documentation"
[3]: https://help.puppet.com/core/current/Content/PuppetCore/lang_facts_and_builtin_vars.htm?utm_source=chatgpt.com "Facts and built-in variables | Puppet Core | 8.20.0"
[4]: https://help.puppet.com/scm/current/Content/UserGuide/SCM/scm_overview.htm?utm_source=chatgpt.com "Security Compliance Management overview | Security Compliance Management | 3.8.0"
[5]: https://fleetdm.com/tables/etc_hosts?utm_source=chatgpt.com "Fleet | \"etc_hosts\" in osquery | Fleet documentation"
[6]: https://help.puppet.com/bolt/current/topics/configuring_bolt.htm?utm_source=chatgpt.com "Configure Bolt"
[7]: https://docs.rudder.io/techniques/9.1/audit_from_osquery.html?utm_source=chatgpt.com "audit_from_osquery - Rudder techniques"



---
---
---
