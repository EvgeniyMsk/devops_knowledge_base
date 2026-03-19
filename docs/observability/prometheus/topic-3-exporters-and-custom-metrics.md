# 3. Экспортеры и кастомные метрики

---
Цель — мониторить не только «что дали из коробки», а **то, что важно бизнесу и SLO**: метрики ОС/сети/БД через **exporters** и **свои** counter / gauge / histogram в сервисе через **client libraries**. Ниже — обзор популярных экспортёров, как они устроены, production-замечания и минимальные примеры **Go** и **Python**.

---

## Как устроены exporters

1. Процесс (или модуль) периодически собирает данные из источника (CPU, HTTP check, БД).
2. Отдаёт текст в формате **OpenMetrics / Prometheus exposition** по HTTP, чаще всего **`GET /metrics`**.
3. **Prometheus** делает **scrape** этого endpoint’а.

Best practice: exporter **не обязан** быть отдельным Pod на каждый сервис — но **сетевой периметр** и **аутентификация** должны соответствовать чувствительности данных (например строка подключения к БД только на localhost sidecar).

---

## Популярные exporters

| Exporter | Что даёт | Production-комментарий |
|----------|-----------|-------------------------|
| **node_exporter** | CPU, память, диск, сеть хоста | Исключайте ненужные **collectors** (`--no-collector.*`) для снижения нагрузки; не путать с **container**-метриками в K8s |
| **blackbox_exporter** | Проверки **снаружи**: HTTP(S), TCP, ICMP, DNS | Удобен для **synthetic** мониторинга и SLO с точки зрения пользователя; держите рядом с **надёжной** сетью |
| **postgres_exporter** | Подключения, статистика БД, custom queries | `DATA_SOURCE_NAME` через секрет; **не** выставляйте порт в интернет; ограничьте `auto discover` |
| **nginx_exporter** / **nginx-prometheus-exporter** | Запросы, активные соединения (часто из **stub_status**) | Включите **stub_status** только на внутреннем интерфейсе; защитите endpoint |

Меньше — лучше: один экспортёр на **роль** (например один `postgres_exporter` на инстанс, а не «все базы в одном процессе без изоляции»).

---

## Blackbox: пример `scrape_configs`

Blackbox обычно вызывают так: Prometheus ходит на **blackbox**, а цель проверки передаётся параметром `target`.

```yaml
scrape_configs:
  - job_name: blackbox-http
    metrics_path: /probe
    params:
      module: [http_2xx]
    static_configs:
      - targets:
          - https://api.example.com/health
          - https://status.example.com
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: blackbox-exporter:9115
```

Комментарий: модуль `http_2xx` и таймауты задаются в **`blackbox.yml`** на стороне blackbox_exporter.

---

## postgres_exporter: идея запуска

```bash
# Переменная окружения из секрет-хранилища, не из shell history в production
export DATA_SOURCE_NAME="postgresql://exporter:REDACTED@db.internal:5432/postgres?sslmode=require"
postgres_exporter --web.listen-address=:9187
```

Best practice: отдельный DB user **только чтение** и минимальные привилегии; при необходимости — **custom queries** в отдельном файле с review.

---

## Кастомные метрики через client libraries

### Именование и типы (напоминание)

| Тип | Пример использования |
|-----|----------------------|
| **Counter** | Всего обработанных запросов, ошибок (растёт, сбрасывается только рестартом) |
| **Gauge** | Текущая длина очереди, число открытых соединений |
| **Histogram** | Длительность запросов, размеры ответов (с `_bucket`, `_sum`, `_count`) |

Правила:

- суффиксы `_total` для counter’ов — **соглашение** Prometheus;
- **HELP** и **TYPE** генерируются библиотекой — заполняйте содержательно;
- **кардинальность**: не добавляйте в лейблы `user_id`, `order_id` для каждого запроса.

---

## Практика: минимальный сервис на Go

Используется `github.com/prometheus/client_golang` (версию уточняйте в `go.mod`).

```go
package main

import (
	"net/http"
	"strconv"
	"time"

	"github.com/prometheus/client_golang/prometheus"
	"github.com/prometheus/client_golang/prometheus/promhttp"
)

var (
	httpRequests = prometheus.NewCounterVec(
		prometheus.CounterOpts{
			Name: "demo_http_requests_total",
			Help: "Всего HTTP-запросов",
		},
		[]string{"path", "code"},
	)
	inFlight = prometheus.NewGauge(prometheus.GaugeOpts{
		Name: "demo_http_in_flight",
		Help: "Текущее число обрабатываемых запросов",
	})
	requestDuration = prometheus.NewHistogramVec(
		prometheus.HistogramOpts{
			Name:    "demo_http_request_duration_seconds",
			Help:    "Длительность обработки",
			Buckets: prometheus.DefBuckets,
		},
		[]string{"path"},
	)
)

func init() {
	prometheus.MustRegister(httpRequests, inFlight, requestDuration)
}

func main() {
	mux := http.NewServeMux()
	mux.Handle("/metrics", promhttp.Handler())
	mux.HandleFunc("/api/hello", func(w http.ResponseWriter, r *http.Request) {
		inFlight.Inc()
		defer inFlight.Dec()
		start := time.Now()
		defer func() {
			requestDuration.WithLabelValues("/api/hello").Observe(time.Since(start).Seconds())
		}()
		httpRequests.WithLabelValues("/api/hello", strconv.Itoa(http.StatusOK)).Inc()
		w.WriteHeader(http.StatusOK)
		_, _ = w.Write([]byte("ok"))
	})
	_ = http.ListenAndServe(":8080", mux)
}
```

Комментарий: в production вынесите маршруты в константы для стабильных лейблов `path`, иначе **взорвётся** кардинальность.

---

## Практика: минимальный сервис на Python

```bash
pip install prometheus-client
```

```python
from prometheus_client import Counter, Gauge, Histogram, start_http_server
import time
import random

REQUESTS = Counter(
    "demo_requests_total",
    "Всего запросов",
    ["endpoint", "method"],
)
ACTIVE = Gauge("demo_jobs_active", "Активные фоновые задачи")
LATENCY = Histogram(
    "demo_operation_seconds",
    "Длительность операции",
    buckets=(0.01, 0.05, 0.1, 0.5, 1.0, 5.0),
)


def handle():
    ACTIVE.inc()
    try:
        start = time.time()
        time.sleep(random.uniform(0.01, 0.05))
        LATENCY.observe(time.time() - start)
        REQUESTS.labels(endpoint="/work", method="POST").inc()
    finally:
        ACTIVE.dec()


if __name__ == "__main__":
    start_http_server(8000)
    while True:
        handle()
        time.sleep(1)
```

`GET http://localhost:8000/metrics` — для проверки scrape в Prometheus (`static_configs` или SD).

---

## Production checklist

| # | Практика |
|---|----------|
| 1 | `/metrics` не в публичный интернет без **TLS/auth** или mesh-policy |
| 2 | **Histogram buckets** подбирать под SLO (не оставлять «случайные» по умолчанию везде) |
| 3 | Exporters БД и админских API — **sidecar + localhost** |
| 4 | Документировать **единый префикс** метрик команды (`myteam_` / `corp_api_`) |
| 5 | Нагрузочное тестирование: cost scrape на стороне Prometheus при росте кардинальности |

---

## Дополнительные материалы

- [Exporters and integrations](https://prometheus.io/docs/instrumenting/exporters/)
- [Client libraries](https://prometheus.io/docs/instrumenting/clientlibs/)
- [client_golang](https://github.com/prometheus/client_golang)
- [prometheus_client (Python)](https://github.com/prometheus/client_python)

