<<<<<<< HEAD
# Домашнее задание к занятию «13. Системы мониторинга»

**Выполнил:** [Ваше имя]  
**Дата:** [Дата сдачи]

---

## Содержание

1. [Задание 1. Минимальный набор метрик](#задание-1)
2. [Задание 2. Объяснение метрик менеджеру продукта](#задание-2)
3. [Задание 3. Сбор ошибок приложений без бюджета на логи](#задание-3)
4. [Задание 4. Ошибка в расчёте SLA](#задание-4)
5. [Задание 5. Плюсы и минусы pull и push](#задание-5)
6. [Задание 6. Классификация систем мониторинга](#задание-6)
7. [Задание 7. Запуск TICK-стека и Chronograf](#задание-7)
8. [Дополнительное задание (*). Скрипт сбора метрик и дашборд](#дополнительное-задание)

---

## Задание 1

**Вопрос:** Вас пригласили настроить мониторинг на проект. Проект — платформа для вычислений с выдачей текстовых отчётов, которые сохраняются на диск. Взаимодействие по HTTP. Вычисления загружают CPU. Какой минимальный набор метрик вы выведите в мониторинг и почему?

**Ответ:**

- **CPU:** `usage` + `load average` — вычисления грузят процессор, LA показывает очередь задач.
- **RAM:** `used`/`available`/`swap` — риск OOM и деградации.
- **Диск:** `free space`, `inodes used`, `disk I/O` — отчёты пишутся на диск; inodes могут кончиться раньше места.
- **HTTP:** RPS, коды ответов, latency p95/p99 — доступность и скорость платформы.
- **Специфика вычислений:** длительность расчёта и успешность сохранения отчёта — «успех» для продукта шире, чем HTTP 2xx.

---

## Задание 2

**Вопрос:** Менеджер продукта сказал, что ему непонятно, что такое RAM/inodes/CPUla. Он хочет понимать, насколько мы выполняем обязанности перед клиентами и какое качество обслуживания. Что вы можете ему предложить?

**Ответ:**

Перевести мониторинг на язык **SLI/SLO/SLA** и сделать бизнес-дашборд:
- **Доступность** — % успешных пользовательских запросов.
- **Latency p95/p99** — сколько ждут клиенты.
- **Доля ошибок** — % запросов с 5xx.
- **Error budget** — сколько «плохих» событий допустимо в рамках SLO и сколько израсходовано.

Технические метрики (RAM/inodes/CPUla) оставить для SRE/DevOps «под капотом».

---

## Задание 3

**Вопрос:** DevOps-команде не выделили финансирование на построение системы сбора логов. Разработчики хотят видеть все ошибки, которые выдают приложения. Какое решение предпринять?

**Ответ:**

Собирать **только ошибки** open-source/self-hosted средствами:
- **Sentry self-hosted** — трекинг ошибок со стектрейсами и уведомлениями.
- **Grafana Loki + Promtail** — сбор логов и алерты по `ERROR`/`Exception`.
- Минимальный вариант — отправка ошибок из приложения webhook’ом в **Telegram-бота/чат разработки**.

Принцип: фокус на ошибках, использование уже имеющейся инфраструктуры.

---

## Задание 4

**Вопрос:** Вы вывели SLA=99% по формуле `summ_2xx_requests / summ_all_requests`. Параметр не поднимается выше 70%, но кодов 5xx и 4xx в системе нет. Где ошибка?

**Ответ:**

Ошибка в **знаменателе** или в определении «успеха»:
- В `summ_all_requests` попадают **3xx/1xx** (редиректы, информационные), health-check’и, внутренние вызовы — их надо исключать или считать успешными.
- Для платформы «успех» — не только HTTP 2xx, но и **успешное сохранение отчёта** на диск.

Правильно: `успешные_пользовательские_запросы / все_пользовательские_запросы` с согласованным определением успеха.

---

## Задание 5

**Вопрос:** Опишите основные плюсы и минусы pull и push систем мониторинга.

**Ответ:**

- **Pull:** плюсы — централизованный контроль, естественная проверка доступности, не надо открывать входящие порты на целях; минусы — плохо для короткоживущих целей, сложно через NAT/файрволы, постоянный фоновый опрос.
- **Push:** плюсы — удобно за NAT/файрволом, хорошо для batch/serverless, цель сама буферизует; минусы — сложнее понять, упала цель или агент, риск перегрузки приёмника, меньше контроля со стороны сервера.

---

## Задание 6

**Вопрос:** Какие из систем относятся к push, какие к pull, а какие гибридные: Prometheus, TICK, Zabbix, VictoriaMetrics, Nagios?

**Ответ:**

| Система | Модель |
|---|---|
| **Prometheus** | Pull + Pushgateway (гибридная) |
| **TICK** | Push |
| **Zabbix** | Гибридная (passive = pull, active = push) |
| **VictoriaMetrics** | Pull + поддержка push-протоколов (гибридная) |
| **Nagios** | Гибридная (активные проверки = pull, пассивные = push) |

---

## Задание 7

**Вопрос:** Склонируйте репозиторий `https://github.com/influxdata/sandbox/tree/master` и запустите TICK-стек, используя Docker и Docker Compose. Приведите скриншот веб-интерфейса Chronograf (`http://localhost:8888`).

**Ответ:**

### 7.1. Подготовка и запуск TICK-стека

Репозиторий `influxdata/sandbox` рассчитан на использование управляющего скрипта `./sandbox`, который сам подготавливает окружение. В современной системе с плагином `docker compose` (без дефиса) скрипт может не найти устаревшую команду `docker-compose`. Решение — создать обёртку:

```bash
sudo tee /usr/local/bin/docker-compose > /dev/null <<'EOF'
#!/bin/bash
docker compose "$@"
EOF
sudo chmod +x /usr/local/bin/docker-compose
```

Затем:

```bash
git clone https://github.com/influxdata/sandbox.git
cd sandbox
./sandbox up
```

### 7.2. Исправленные проблемы при запуске

В ходе запуска возникли три типовые проблемы, которые были устранены:

| Проблема | Симптом | Решение |
|---|---|---|
| **Права на тома Chronograf/Kapacitor** | `permission denied` при открытии `/var/lib/chronograf/chronograf-v1.db` и `/var/lib/kapacitor/cluster.id` | `sudo chown -R 999:999 ./chronograf ./kapacitor` |
| **Устаревшие параметры Telegraf** | `configuration specified the fields ["total" "container_names" "perdevice"]` | Удалить строки `total`, `container_names`, `perdevice` из секции `[[inputs.docker]]` в `telegraf/telegraf.conf` |
| **Доступ Telegraf к docker.sock** | `permission denied while trying to connect to the docker API` | Добавить `group_add` с GID группы `docker` в `docker-compose.yml` для сервиса `telegraf` |

Пример исправленной секции `telegraf` в `docker-compose.yml`:

```yaml
  telegraf:
    image: telegraf:1.40.3
    user: telegraf
    group_add:
      - "998"   # GID группы docker, узнать: stat -c '%g' /var/run/docker.sock
    volumes:
      - ./telegraf/telegraf.conf:/etc/telegraf/telegraf.conf:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
    links:
      - influxdb
    ports:
      - "8092:8092/udp"
      - "8094:8094"
      - "8125:8125/udp"
```

### 7.3. Итоговое состояние стека

После исправлений все пять контейнеров запущены:

```bash
$ docker ps
CONTAINER ID   IMAGE                   STATUS         PORTS
b7129df1a871   telegraf                Up             8092/udp, 8125/udp, 8094/tcp
290c48697fb9   chrono_config           Up             0.0.0.0:8888->8888/tcp
915c3e7ca9a9   kapacitor               Up             0.0.0.0:9092->9092/tcp
89a534a3d553   influxdb                Up             0.0.0.0:8086->8086/tcp
7646efd36c12   sandbox-documentation   Up             0.0.0.0:3010->3000/tcp
```

### 7.4. Скриншот веб-интерфейса Chronograf

Chronograf доступен по адресу `http://158.160.55.39:8888` (порт `8888` проброшен на `0.0.0.0`, в security group облака открыт входящий TCP `8888`).

**Скриншот стартовой страницы Chronograf:**

![Chronograf UI](screenshots/chronograf-main.png)

*На скриншоте виден интерфейс Chronograf: боковая панель (Host List, Data Explorer, Dashboards, Alerting, Configuration) и стартовая страница.*

### 7.5. Data Explorer: график утилизации CPU

В Chronograf открыта вкладка **Data Explorer** → **Add a query** → база **`telegraf.autogen`** → measurement **`cpu`** → tag `host` = `telegraf-getting-started` → field **`usage_system`**. Внизу построен график утилизации CPU. Вверху отображается сгенерированный InfluxQL-запрос.

**Скриншот графика CPU и запроса:**

![Chronograf Data Explorer CPU](screenshots/chronograf-cpu.png)

*На скриншоте виден график `usage_system` и InfluxQL-запрос вида:*
```sql
SELECT mean("usage_system") AS "mean_usage_system"
FROM "telegraf"."autogen"."cpu"
WHERE time > now() - 15m AND "host" = 'telegraf-getting-started'
GROUP BY time(10s) fill(null)
```

**Эксперименты с запросом:**
- Интервал наблюдений изменён с `15m` на `1h` — график стал шире по времени.
- Группировка изменена с `time(10s)` на `time(1m)` — точки стали реже.
- Добавлена группировка `GROUP BY "cpu"` — появились отдельные линии по ядрам (`cpu-total`, `cpu0` и т.д.).

### 7.6. Метрики Docker в Telegraf

В конфигурацию Telegraf добавлен плагин `inputs.docker`:

```toml
[[inputs.docker]]
  endpoint = "unix:///var/run/docker.sock"
```

После перезапуска Telegraf в логах видно успешную загрузку плагина:

```
I! Loaded inputs: cpu docker influxdb syslog system
```

В Chronograf → **Data Explorer** → база **`telegraf.autogen`** → список **Measurements** появились метрики с префиксом `docker_`: `docker_container_cpu`, `docker_container_mem`, `docker_container_net`, `docker_container_blkio`, `docker_container_status`, `docker_engine`, `docker_images`, `docker_n_containers` и др.

**Скриншот списка Measurements с метриками Docker:**

![Chronograf Docker Measurements](screenshots/chronograf-docker-measurements.png)

*На скриншоте виден список измерений базы `telegraf.autogen`, включая `docker_container_cpu`, `docker_container_mem`, `docker_engine` и другие.*

---

## Дополнительное задание (*)

**Вопрос:** Написать простой python3-скрипт для сбора основных метрик сервера из `/proc`, складывать их в файл `YY-MM-DD-awesome-monitoring.log` в `/var/log` в виде JSON-строк, запускать каждую минуту по cron. Привести: а) работающий код скрипта, б) конфигурацию cron, в) пример лога с не менее 5 записями. Дополнительно: создать dashboard в Chronograf с отображением утилизации CPU, использованной RAM, утилизации дисков, количества контейнеров, аптайма и др.

### а) Код python3-скрипта

Файл `/usr/local/bin/awesome-monitoring.py`:

```python
#!/usr/bin/env python3
"""
awesome-monitoring.py — простой сборщик метрик сервера из /proc.
Каждый запуск дописывает одну JSON-строку в /var/log/YY-MM-DD-awesome-monitoring.log
"""
import json
import os
import time
import shutil
from datetime import datetime

PROC = "/proc"
LOG_DIR = "/var/log"


# ---------- CPU ----------
def read_cpu_times():
    """Возвращает dict с суммарными тиками CPU из /proc/stat (строка 'cpu ')."""
    with open(os.path.join(PROC, "stat")) as f:
        for line in f:
            if line.startswith("cpu "):
                parts = line.split()[1:]
                keys = ["user", "nice", "system", "idle", "iowait",
                        "irq", "softirq", "steal", "guest", "guest_nice"]
                return {k: int(v) for k, v in zip(keys, parts)}
    raise RuntimeError("не найдена строка cpu в /proc/stat")


def cpu_percent(prev, curr):
    """Считает % использования CPU между двумя замерами (не-idle / всего)."""
    if not prev:
        return None
    prev_idle = prev["idle"] + prev["iowait"]
    curr_idle = curr["idle"] + curr["iowait"]
    prev_total = sum(prev.values())
    curr_total = sum(curr.values())
    total_diff = curr_total - prev_total
    idle_diff = curr_idle - prev_idle
    if total_diff <= 0:
        return None
    return round((total_diff - idle_diff) / total_diff * 100, 2)


# ---------- RAM ----------
def read_meminfo():
    """Возвращает dict с ключевыми полями из /proc/meminfo (в байтах)."""
    data = {}
    with open(os.path.join(PROC, "meminfo")) as f:
        for line in f:
            key, val = line.split(":", 1)
            val = val.strip().split()[0]  # значение в kB
            data[key] = int(val) * 1024
    total = data.get("MemTotal", 0)
    available = data.get("MemAvailable", data.get("MemFree", 0))
    used = total - available
    return {
        "mem_total_bytes": total,
        "mem_available_bytes": available,
        "mem_used_bytes": used,
        "mem_used_percent": round(used / total * 100, 2) if total else None,
        "swap_total_bytes": data.get("SwapTotal", 0),
        "swap_free_bytes": data.get("SwapFree", 0),
    }


# ---------- LOAD AVERAGE ----------
def read_loadavg():
    """Возвращает load average 1/5/15 минут из /proc/loadavg."""
    with open(os.path.join(PROC, "loadavg")) as f:
        parts = f.read().split()
    return {
        "load1": float(parts[0]),
        "load5": float(parts[1]),
        "load15": float(parts[2]),
        "running_procs": int(parts[3].split("/")[0]),
        "total_procs": int(parts[3].split("/")[1]),
    }


# ---------- UPTIME ----------
def read_uptime():
    """/proc/uptime: секунды с момента загрузки и суммарное idle-время."""
    with open(os.path.join(PROC, "uptime")) as f:
        up, idle = f.read().split()
    return {"uptime_seconds": float(up), "idle_seconds": float(idle)}


# ---------- DISK ----------
def read_disk_usage(path="/"):
    """Занятость корневого раздела через statvfs (аналог df)."""
    st = shutil.disk_usage(path)
    total = st.total
    used = st.used
    return {
        "disk_total_bytes": total,
        "disk_used_bytes": used,
        "disk_free_bytes": st.free,
        "disk_used_percent": round(used / total * 100, 2) if total else None,
    }


# ---------- INODES ----------
def read_inodes(path="/"):
    """Использование inodes (важно для мелких файлов, например отчётов)."""
    st = os.statvfs(path)
    total = st.f_files
    free = st.f_ffree
    used = total - free
    return {
        "inodes_total": total,
        "inodes_free": free,
        "inodes_used": used,
        "inodes_used_percent": round(used / total * 100, 2) if total else None,
    }


# ---------- NETWORK ----------
def read_netdev():
    """Суммарный трафик по всем интерфейсам (кроме lo) из /proc/net/dev."""
    rx = tx = 0
    with open(os.path.join(PROC, "net", "dev")) as f:
        for line in f:
            if ":" not in line:
                continue
            iface, rest = line.split(":", 1)
            iface = iface.strip()
            if iface == "lo":
                continue
            cols = rest.split()
            rx += int(cols[0])   # bytes received
            tx += int(cols[8])   # bytes transmitted
    return {"net_rx_bytes": rx, "net_tx_bytes": tx}


# ---------- DOCKER CONTAINERS (опционально) ----------
def read_docker_containers():
    """Количество запущенных контейнеров через docker CLI (если установлен)."""
    try:
        import subprocess
        out = subprocess.check_output(
            ["docker", "ps", "-q"], stderr=subprocess.DEVNULL, timeout=5
        )
        return {"docker_containers_running": len(out.decode().split())}
    except Exception:
        return {"docker_containers_running": None}


# ---------- СБОРКА ----------
def collect():
    cpu_now = read_cpu_times()
    # Для % CPU нужен предыдущий замер. Читаем два раза с паузой 1 секунда.
    time.sleep(1)
    cpu_next = read_cpu_times()
    cpu_usage = cpu_percent(cpu_now, cpu_next)

    metrics = {
        "timestamp": int(time.time()),
        "cpu_usage_percent": cpu_usage if cpu_usage is not None else 0.0,
    }
    metrics.update(read_loadavg())
    metrics.update(read_meminfo())
    metrics.update(read_uptime())
    metrics.update(read_disk_usage("/"))
    metrics.update(read_inodes("/"))
    metrics.update(read_netdev())
    metrics.update(read_docker_containers())
    return metrics


def log_path():
    # Формат имени файла — YY-MM-DD (как в задании)
    today = datetime.now().strftime("%y-%m-%d")
    return os.path.join(LOG_DIR, f"{today}-awesome-monitoring.log")


def main():
    metrics = collect()
    path = log_path()
    line = json.dumps(metrics, ensure_ascii=False)
    with open(path, "a") as f:
        f.write(line + "\n")
    print(line)


if __name__ == "__main__":
    main()
```

Скрипт собирает **более 4 метрик**: `cpu_usage_percent`, `load1/5/15`, `mem_used_bytes`, `mem_used_percent`, `uptime_seconds`, `disk_used_percent`, `inodes_used_percent`, `net_rx_bytes`, `net_tx_bytes`, `docker_containers_running` и др. (всего 24 поля).

Сделать исполняемым:

```bash
sudo chmod +x /usr/local/bin/awesome-monitoring.py
```

### б) Конфигурация cron

Добавлено в crontab root (скрипт пишет в `/var/log`, нужны права):

```bash
sudo crontab -e
```

Строка:

```cron
* * * * * /usr/bin/python3 /usr/local/bin/awesome-monitoring.py >> /var/log/awesome-monitoring-cron.log 2>&1
```

Проверка:

```bash
$ sudo crontab -l
* * * * * /usr/bin/python3 /usr/local/bin/awesome-monitoring.py >> /var/log/awesome-monitoring-cron.log 2>&1
```

> **Примечание.** `awesome-monitoring-cron.log` — это отладочный лог cron (туда попадает stdout скрипта, т.е. дубликат JSON-строк). Основные данные пишутся самим скриптом в `/var/log/YY-MM-DD-awesome-monitoring.log`.

### в) Пример `YY-MM-DD-awesome-monitoring.log` (5 записей)

Файл: `/var/log/26-11-12-awesome-monitoring.log` (создан cron-задачей, запускаемой каждую минуту). Проверка:

```bash
$ sudo ls -la /var/log/??-??-??-awesome-monitoring.log
-rw-r--r-- 1 root root 4017 Nov 12 09:53 /var/log/26-11-12-awesome-monitoring.log
```

Содержимое (5 записей):

```json
{"timestamp": 1794476942, "cpu_usage_percent": 0.5, "load1": 0.1, "load5": 0.06, "load15": 0.04, "running_procs": 1, "total_procs": 376, "mem_total_bytes": 2067382272, "mem_available_bytes": 1220624384, "mem_used_bytes": 846757888, "mem_used_percent": 40.96, "swap_total_bytes": 0, "swap_free_bytes": 0, "uptime_seconds": 14467.3, "idle_seconds": 28229.55, "disk_total_bytes": 31620399104, "disk_used_bytes": 22162337792, "disk_free_bytes": 8121114624, "disk_used_percent": 70.09, "inodes_total": 1966080, "inodes_free": 1379045, "inodes_used": 587035, "inodes_used_percent": 29.86, "net_rx_bytes": 800007603, "net_tx_bytes": 78102772, "docker_containers_running": 5}
{"timestamp": 1794477002, "cpu_usage_percent": 2.03, "load1": 0.1, "load5": 0.07, "load15": 0.05, "running_procs": 1, "total_procs": 376, "mem_total_bytes": 2067382272, "mem_available_bytes": 1225830400, "mem_used_bytes": 841551872, "mem_used_percent": 40.71, "swap_total_bytes": 0, "swap_free_bytes": 0, "uptime_seconds": 14527.4, "idle_seconds": 28346.81, "disk_total_bytes": 31620399104, "disk_used_bytes": 22162685952, "disk_free_bytes": 8120766464, "disk_used_percent": 70.09, "inodes_total": 1966080, "inodes_free": 1379045, "inodes_used": 587035, "inodes_used_percent": 29.86, "net_rx_bytes": 800860671, "net_tx_bytes": 78955365, "docker_containers_running": 5}
{"timestamp": 1794477062, "cpu_usage_percent": 0.5, "load1": 0.08, "load5": 0.07, "load15": 0.05, "running_procs": 1, "total_procs": 375, "mem_total_bytes": 2067382272, "mem_available_bytes": 1220820992, "mem_used_bytes": 846561280, "mem_used_percent": 40.95, "swap_total_bytes": 0, "swap_free_bytes": 0, "uptime_seconds": 14586.5, "idle_seconds": 28462.18, "disk_total_bytes": 31620399104, "disk_used_bytes": 22163030016, "disk_free_bytes": 8120422400, "disk_used_percent": 70.09, "inodes_total": 1966080, "inodes_free": 1379045, "inodes_used": 587035, "inodes_used_percent": 29.86, "net_rx_bytes": 801671339, "net_tx_bytes": 79766499, "docker_containers_running": 5}
{"timestamp": 1794477122, "cpu_usage_percent": 1.01, "load1": 0.03, "load5": 0.06, "load15": 0.04, "running_procs": 1, "total_procs": 376, "mem_total_bytes": 2067382272, "mem_available_bytes": 1225945088, "mem_used_bytes": 841437184, "mem_used_percent": 40.7, "swap_total_bytes": 0, "swap_free_bytes": 0, "uptime_seconds": 14646.61, "idle_seconds": 28579.33, "disk_total_bytes": 31620399104, "disk_used_bytes": 22163378176, "disk_free_bytes": 8120074240, "disk_used_percent": 70.09, "inodes_total": 1966080, "inodes_free": 1379045, "inodes_used": 587035, "inodes_used_percent": 29.86, "net_rx_bytes": 802508054, "net_tx_bytes": 80605605, "docker_containers_running": 5}
{"timestamp": 1794477182, "cpu_usage_percent": 1.98, "load1": 0.01, "load5": 0.05, "load15": 0.04, "running_procs": 1, "total_procs": 376, "mem_total_bytes": 2067382272, "mem_available_bytes": 1229557760, "mem_used_bytes": 837824512, "mem_used_percent": 40.53, "swap_total_bytes": 0, "swap_free_bytes": 0, "uptime_seconds": 14706.72, "idle_seconds": 28696.59, "disk_total_bytes": 31620399104, "disk_used_bytes": 22163722240, "disk_free_bytes": 8119730176, "disk_used_percent": 70.09, "inodes_total": 1966080, "inodes_free": 1379045, "inodes_used": 587035, "inodes_used_percent": 29.86, "net_rx_bytes": 803326614, "net_tx_bytes": 81424441, "docker_containers_running": 5}
```

Проверка накопления записей:

```bash
$ sudo tail -n 5 /var/log/$(date +%y-%m-%d)-awesome-monitoring.log
$ sudo wc -l /var/log/$(date +%y-%m-%d)-awesome-monitoring.log
```

### г) Dashboard в Chronograf

Создан дашборд **«Awesome Monitoring»** в Chronograf (`Dashboards` → `Create Dashboard` → `Add Cell`). Используется база **`telegraf.autogen`**.

| Панель | Measurement | Field | Запрос (InfluxQL) |
|---|---|---|---|
| **Утилизация CPU** | `cpu` | `usage_system` | `SELECT mean("usage_system") AS "CPU system %" FROM "telegraf"."autogen"."cpu" WHERE time > now() - 15m GROUP BY time(10s), "host" fill(null)` |
| **Использование RAM** | `mem` | `used_percent` | `SELECT mean("used_percent") AS "RAM used %" FROM "telegraf"."autogen"."mem" WHERE time > now() - 15m GROUP BY time(10s) fill(null)` |
| **Утилизация диска** | `disk` | `used_percent` | `SELECT mean("used_percent") AS "Disk used %" FROM "telegraf"."autogen"."disk" WHERE time > now() - 15m AND "path" = '/' GROUP BY time(10s) fill(null)` |
| **Количество контейнеров** | `docker_n_containers` | `running` | `SELECT mean("running") AS "Containers running" FROM "telegraf"."autogen"."docker_n_containers" WHERE time > now() - 15m GROUP BY time(30s) fill(null)` |
| **Аптайм** | `system` | `uptime` | `SELECT mean("uptime") / 3600 AS "Uptime, h" FROM "telegraf"."autogen"."system" WHERE time > now() - 15m GROUP BY time(1m) fill(null)` |
| **Сетевой трафик** (доп.) | `net` | `bytes_recv` / `bytes_sent` | `SELECT mean("bytes_recv") AS "RX", mean("bytes_sent") AS "TX" FROM "telegraf"."autogen"."net" WHERE time > now() - 15m GROUP BY time(10s), "interface" fill(null)` |
| **Load average** (доп.) | `system` | `load1` / `load5` / `load15` | `SELECT mean("load1"), mean("load5"), mean("load15") FROM "telegraf"."autogen"."system" WHERE time > now() - 15m GROUP BY time(10s) fill(null)` |

**Скриншот дашборда Chronograf:**

![Chronograf Dashboard](screenshots/chronograf-dashboard.png)

*На скриншоте видны панели: утилизация CPU, использование RAM, утилизация диска, количество запущенных контейнеров, аптайм, сетевой трафик и load average.*

---

## Итог

В ходе выполнения домашнего задания:

1. Разобраны теоретические вопросы о минимальном наборе метрик, SLI/SLO/SLA, сборе ошибок без бюджета, ошибках в расчёте SLA, плюсах и минусах pull/push и классификации систем мониторинга.
2. Развёрнут TICK-стек (InfluxDB, Telegraf, Chronograf, Kapacitor) через Docker Compose, устранены типовые проблемы с правами, конфигурацией Telegraf и доступом к docker.sock.
3. В Chronograf построен график утилизации CPU и получены метрики Docker (`docker_container_cpu`, `docker_container_mem`, `docker_engine` и др.).
4. Написан python3-скрипт для сбора метрик из `/proc`, настроен cron-запуск каждую минуту, получен лог с JSON-строками в формате `YY-MM-DD-awesome-monitoring.log`.
5. Создан дашборд в Chronograf с панелями CPU, RAM, диска, контейнеров, аптайма, сети и load average.

Все скриншоты приложены в папке `screenshots/`.
=======
# monitoring-homework-02
>>>>>>> origin/main
