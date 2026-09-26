# Part 099: Monitoring และ Observability: Prometheus, Grafana, OpenTelemetry

> ภาคที่ 9: DevOps และ Deployment — ตอนที่ 5 จาก 5 (Part 95–99)

## สารบัญของบทนี้

1. เมื่อแอปรันบน Kubernetes แล้ว จะรู้ได้อย่างไรว่ามันยัง "โอเค" อยู่จริง
2. สามเสาหลักของ Observability: Metrics, Logs, Traces
3. เสาหลักที่ 1 (Metrics): Counter, Gauge, Histogram ด้วย `client_golang`
4. เดินสาย `/metrics` เข้า HTTP Server (ต่อยอด Part 047) แล้ว scrape จริงด้วย `curl`
5. RED Method: Metric สามตัวที่ Dashboard ของทุก Go Service ควรมี
6. เสาหลักที่ 2 (Logs): ทบทวน `slog` จาก Part 054 ในบทบาทของ Observability
7. เสาหลักที่ 3 (Traces): Distributed Tracing คืออะไร และทำไมต้องมี
8. OpenTelemetry Go SDK: สร้างและพิมพ์ Span จริงด้วย Console Exporter
9. เชื่อมสามเสาหลักเข้าด้วยกัน: ฝัง Trace ID ลงใน Log จริง
10. Grafana: ชั้นแสดงผลของ Prometheus (แนวคิด และหน้าตา Dashboard ที่ควรมี)
11. จาก Console Exporter สู่ Production: OTLP Exporter และ Collector (แนวคิด)
12. หมายเหตุความซื่อสัตย์เรื่องการรันจริงในบทนี้
13. สรุปสิ่งที่ได้เรียนในบทนี้
14. สรุปภาพรวมภาคที่ 9 (DevOps และ Deployment) ทั้งภาค
15. แบบฝึกหัดท้ายบท

---

## 1. เมื่อแอปรันบน Kubernetes แล้ว จะรู้ได้อย่างไรว่ามันยัง "โอเค" อยู่จริง

ตลอด 4 บทที่ผ่านมาในภาคนี้ เราสร้าง Go API ที่ container ได้ (**Part 095**), รันหลาย service ร่วมกันได้ (**Part 096**), deploy ขึ้น Kubernetes ได้พร้อม auto-scale (**Part 097**), และมี pipeline อัตโนมัติที่ build/test/deploy ให้เองทุกครั้งที่ merge โค้ด (**Part 098**) — คำถามที่ยังไม่มีคำตอบคือ: **เมื่อระบบนี้รันอยู่จริงบน production แล้ว ทีมจะรู้ได้อย่างไรว่า**

- ตอนนี้มีคน**ใช้งานจริงมากแค่ไหน** (request ต่อวินาที)?
- มี**คำขอที่ล้มเหลว**เยอะขึ้นผิดปกติหรือไม่?
- ทำไม request บางอันถึง**ช้าผิดปกติ** และช้าตรงจุดไหนของระบบกันแน่?
- เมื่อเกิดปัญหาขึ้นตอนตี 3 ทีมจะรู้ได้อย่างไรก่อนที่ลูกค้าจะเป็นคนแจ้งเข้ามาเอง?

`readinessProbe`/`livenessProbe` ที่เรียนไปใน **Part 097** ตอบได้แค่คำถามระดับ **"Pod นี้ยังมีชีวิตอยู่ไหม"** เท่านั้น ซึ่งเป็นคำถามที่หยาบเกินไปสำหรับการทำความเข้าใจพฤติกรรมของระบบจริง — **Observability** (ความสามารถในการสังเกตสถานะภายในของระบบจากข้อมูลที่ระบบส่งออกมาภายนอก) คือคำตอบของคำถามเหล่านี้ และเป็นหัวข้อปิดท้ายที่เหมาะสมที่สุดของภาค DevOps เพราะ **การ deploy ระบบให้รันได้เป็นแค่ครึ่งแรกของงาน ส่วนครึ่งหลังคือการรู้ว่าระบบที่ deploy ไปแล้วกำลังทำงานเป็นอย่างไรอยู่ตลอดเวลา**

---

## 2. สามเสาหลักของ Observability: Metrics, Logs, Traces

วงการ Observability ยอมรับกันอย่างกว้างขวางว่ามี **"สามเสาหลัก" (Three Pillars)** ที่เสริมกันและกัน แต่ละเสาตอบคำถามคนละแบบ ไม่มีเสาไหนแทนที่อีกเสาได้ทั้งหมด:

| เสาหลัก | ตอบคำถาม | ลักษณะข้อมูล | เครื่องมือในบทนี้ |
|---|---|---|---|
| **Metrics** | "ระบบเป็นอย่างไร**ในภาพรวม**ตอนนี้" (ตัวเลขที่นับ/วัดได้ตามเวลา) | ตัวเลขที่ถูก aggregate ไว้แล้ว น้ำหนักเบา เก็บได้นานถูก | Prometheus + `client_golang` |
| **Logs** | "เกิด**เหตุการณ์อะไร**ขึ้นบ้าง" (เหตุการณ์แต่ละครั้งแบบละเอียด) | ข้อความ/JSON ต่อเหตุการณ์ ละเอียดแต่มีปริมาณมาก | `log/slog` (ทบทวนจาก Part 054) |
| **Traces** | "request หนึ่งครั้ง**เดินทางผ่านจุดไหนบ้าง**และแต่ละจุดใช้เวลาเท่าไร" | โครงสร้าง parent-child ของ "span" ตามเส้นทางที่ request เดินทางผ่านทั้งระบบ | OpenTelemetry |

เปรียบเทียบให้เห็นภาพ: ถ้าระบบเป็นโรงพยาบาล **Metrics** คือกราฟจำนวนผู้ป่วยต่อชั่วโมงที่หน้าห้องฉุกเฉิน (บอกภาพรวมว่าตอนนี้"หนักไหม"), **Logs** คือบันทึกประจำวันของพยาบาลแต่ละคน (บอกว่าเกิดอะไรขึ้นบ้าง), และ **Traces** คือการติดตามผู้ป่วยคนหนึ่งคนตั้งแต่เข้าประตูจนออกจากโรงพยาบาล ผ่านแผนกไหนบ้าง ใช้เวลาที่แต่ละแผนกนานแค่ไหน (บอกว่า "ทำไม" ผู้ป่วยคนนี้ถึงใช้เวลารวมนานผิดปกติ และ**ช้าตรงแผนกไหน**)

ในทางปฏิบัติ **Metrics คือสิ่งแรกที่บอกว่า "มีปัญหา"** (เช่น error rate พุ่งขึ้น), **Traces คือสิ่งที่บอกว่า "ปัญหาอยู่ตรงไหนของระบบ"** (เช่น service ไหนช้า), และ **Logs คือสิ่งที่บอกรายละเอียดว่า "เกิดอะไรขึ้นแน่ๆ" ที่จุดนั้น** (เช่น error message ที่แท้จริง) — ทั้งสามอย่างทำงานร่วมกันเป็น workflow การ debug ปัญหา production ที่มีประสิทธิภาพที่สุด

---

## 3. เสาหลักที่ 1 (Metrics): Counter, Gauge, Histogram ด้วย `client_golang`

**Prometheus** คือระบบเก็บและ query metric ที่เป็นมาตรฐานโดยพฤตินัยของวงการ cloud-native (เป็นโปรเจกต์ของ Cloud Native Computing Foundation เช่นเดียวกับ Kubernetes) หลักการทำงานของ Prometheus ต่างจากระบบ logging ตรงที่ **Prometheus ไม่รอให้แอปส่งข้อมูลเข้ามา (push) แต่ตัว Prometheus server เป็นฝ่าย "เดินไปดึง" (scrape/pull) ข้อมูลจากแอปเป็นระยะเอง** ผ่าน HTTP endpoint ที่แอปต้องเปิดไว้ (ปกติชื่อ `/metrics`)

ฝั่ง Go ใช้ library ทางการ **`github.com/prometheus/client_golang`** สร้าง metric ได้ 3 ชนิดหลักที่ต้องรู้จัก:

| ชนิด | ความหมาย | ตัวอย่างการใช้งาน |
|---|---|---|
| **Counter** | ตัวเลขที่**เพิ่มขึ้นอย่างเดียว** ไม่มีวันลด (reset กลับเป็น 0 ได้แค่ตอนแอป restart) | จำนวน request ทั้งหมดที่รับมา, จำนวน error ที่เกิดขึ้น |
| **Gauge** | ตัวเลขที่**ขึ้นหรือลงได้อิสระ** ตามค่าปัจจุบัน | จำนวน request ที่กำลังประมวลผลอยู่ตอนนี้ (in-flight), จำนวน connection ที่เปิดอยู่ใน pool |
| **Histogram** | การกระจายตัวของค่าที่วัดได้ แบ่งเป็น**ช่วง (bucket)** พร้อมนับจำนวนที่ตกในแต่ละช่วง | เวลาตอบสนองของ request (response time), ขนาดของ request body |

```go
var (
	httpRequestsTotal = promauto.NewCounterVec(
		prometheus.CounterOpts{
			Name: "http_requests_total",
			Help: "จำนวน HTTP request ทั้งหมดที่ server รับ แยกตาม path และ status code",
		},
		[]string{"path", "status"},
	)

	httpRequestDuration = promauto.NewHistogramVec(
		prometheus.HistogramOpts{
			Name:    "http_request_duration_seconds",
			Help:    "เวลาที่ใช้ตอบ HTTP request แต่ละครั้ง (วินาที)",
			Buckets: prometheus.DefBuckets,
		},
		[]string{"path"},
	)

	inFlightRequests = promauto.NewGauge(
		prometheus.GaugeOpts{
			Name: "http_requests_in_flight",
			Help: "จำนวน HTTP request ที่กำลังถูกประมวลผลอยู่ ณ ขณะนี้",
		},
	)
)
```

จุดที่ต้องสังเกต:

- **`promauto`** (แทนที่จะใช้ `prometheus.NewCounterVec` ตรงๆ) คือ helper package ที่ลงทะเบียน metric เข้า **default registry ให้อัตโนมัติทันทีที่สร้าง** ลดโค้ด boilerplate ที่ต้องเขียนซ้ำทุกครั้ง (ถ้าใช้ `prometheus.NewCounterVec` ตรงๆ ต้องเรียก `prometheus.MustRegister(...)` เพิ่มเองอีกบรรทัด)
- **`*Vec` (CounterVec, HistogramVec)** คือ metric ที่มี **label** — ในตัวอย่างนี้ `http_requests_total` แยกตาม `path` และ `status` ทำให้ query ทีหลังได้ละเอียดขึ้น เช่น "จำนวน request ที่ error เฉพาะที่ path `/orders`" ได้โดยไม่ต้องสร้าง metric แยกกันหลายตัว
- **`prometheus.DefBuckets`** คือชุด bucket มาตรฐานที่ Prometheus แนะนำ (`0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10` วินาที) เหมาะกับเวลาตอบสนองของ HTTP request ทั่วไป — เลือก bucket ให้เหมาะกับช่วงเวลาที่คาดว่า metric ของเราจะตกอยู่ (ถ้าแอปตอบช้ากว่านี้มากเป็นปกติ ต้องปรับ bucket เอง ไม่งั้นข้อมูลจะกระจุกอยู่ที่ bucket สุดท้าย `+Inf` ทั้งหมดจนไม่มีประโยชน์)

> **คำเตือนสำคัญเรื่อง label**: **ห้ามใช้ค่าที่มีความเป็นไปได้ไม่จำกัด (unbounded cardinality) เป็น label เด็ดขาด** เช่น user ID, request ID, หรือ IP address — เพราะ Prometheus สร้าง time series ใหม่ 1 ชุดต่อค่า label ที่ไม่ซ้ำกันทุกค่า ถ้า label มีค่าไม่จำกัด (เช่น user ID ที่มีนับล้าน) จำนวน time series จะระเบิดจนกิน memory ของ Prometheus server จนล่มได้ — label ที่ปลอดภัยคือค่าที่มีจำนวนความเป็นไปได้จำกัดชัดเจน เช่น `path`, `status`, `method`

---

## 4. เดินสาย `/metrics` เข้า HTTP Server (ต่อยอด Part 047) แล้ว scrape จริงด้วย `curl`

การเปิด endpoint `/metrics` ทำได้ตรงๆ ด้วย `promhttp.Handler()` ซึ่งเป็น `http.Handler` มาตรฐาน (ตามหลักการ interface เดียวที่เรียนไปใน **Part 047**) เอาไปแปะเข้ากับ `http.ServeMux` ได้ทันทีเหมือน handler ทั่วไป เขียน middleware เล็กๆ ห่อ handler เดิมเพื่อวัด metric อัตโนมัติทุก request:

```go
func instrumented(path string, next http.HandlerFunc) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		inFlightRequests.Inc()
		defer inFlightRequests.Dec()

		start := time.Now()
		rec := &statusRecorder{ResponseWriter: w, status: http.StatusOK}
		next(rec, r)

		httpRequestDuration.WithLabelValues(path).Observe(time.Since(start).Seconds())
		httpRequestsTotal.WithLabelValues(path, http.StatusText(rec.status)).Inc()
	}
}

type statusRecorder struct {
	http.ResponseWriter
	status int
}

func (r *statusRecorder) WriteHeader(code int) {
	r.status = code
	r.ResponseWriter.WriteHeader(code)
}
```

รูปแบบนี้คือ **middleware pattern** ที่เรียนไปใน **Part 057** พอดี: `instrumented` รับ handler เดิมเข้ามา แล้วคืน handler ใหม่ที่ทำงานเพิ่มเติม (วัดเวลา, นับจำนวน) รอบๆ handler เดิม โดยที่ handler เดิมไม่ต้องรู้เรื่อง metric เลยแม้แต่น้อย — `statusRecorder` คือ trick มาตรฐานที่ห่อ `http.ResponseWriter` เพื่อ**ดักจับ status code** ที่ handler เขียนออกไป (เพราะ `http.ResponseWriter` ปกติไม่มีทาง "อ่านย้อนกลับ" ว่า status code ที่ถูกเขียนไปแล้วคือค่าอะไร ต้องดักจับตอนเขียนเท่านั้น)

ประกอบเข้ากับ mux แล้วเปิด `/metrics`:

```go
mux := http.NewServeMux()
mux.HandleFunc("GET /healthz", instrumented("/healthz", handleHealthz))
mux.HandleFunc("GET /work", instrumented("/work", handleWork))
mux.Handle("GET /metrics", promhttp.Handler())   // endpoint สำหรับ Prometheus scrape
```

### รันจริงและ scrape ด้วย `curl` จริง

Build และรัน server ตัวนี้จริงบนเครื่องที่เขียนบทความนี้ (ใช้ `github.com/prometheus/client_golang v1.20.5`) ยิง request เข้า `/work` 8 ครั้งและ `/healthz` 1 ครั้ง แล้ว `curl` ดึงค่าจาก `/metrics`:

```bash
$ ./metricsdemo &
$ for i in $(seq 1 8); do curl -s http://localhost:9100/work -o /dev/null; done
$ curl -s http://localhost:9100/healthz
$ curl -s http://localhost:9100/metrics | grep -E "^(http_requests_total|http_request_duration_seconds|http_requests_in_flight)"
```

**ผลลัพธ์จริงจากการรัน**:

```
{"status":"ok"}
http_request_duration_seconds_bucket{path="/healthz",le="0.005"} 1
...
http_request_duration_seconds_bucket{path="/healthz",le="+Inf"} 1
http_request_duration_seconds_sum{path="/healthz"} 8.55e-06
http_request_duration_seconds_count{path="/healthz"} 1
http_request_duration_seconds_bucket{path="/work",le="0.005"} 0
http_request_duration_seconds_bucket{path="/work",le="0.01"} 1
http_request_duration_seconds_bucket{path="/work",le="0.025"} 1
http_request_duration_seconds_bucket{path="/work",le="0.05"} 3
http_request_duration_seconds_bucket{path="/work",le="0.1"} 5
http_request_duration_seconds_bucket{path="/work",le="0.25"} 8
http_request_duration_seconds_bucket{path="/work",le="0.5"} 8
http_request_duration_seconds_bucket{path="/work",le="+Inf"} 8
http_request_duration_seconds_sum{path="/work"} 0.67444776
http_request_duration_seconds_count{path="/work"} 8
http_requests_in_flight 0
http_requests_total{path="/healthz",status="OK"} 1
http_requests_total{path="/work",status="OK"} 8
```

อ่านผลลัพธ์นี้ออกได้ทันที: `/work` ถูกเรียก 8 ครั้ง (`http_requests_total{path="/work",...} 8`) รวมเวลาทั้งหมด `0.674` วินาที (`_sum`) และจาก bucket จะเห็นว่า **5 จาก 8 ครั้งตอบภายใน 0.1 วินาที** (`le="0.1"} 5`) ในขณะที่ **ทุกครั้งตอบภายใน 0.25 วินาที** (`le="0.25"} 8`) — นี่คือรูปแบบข้อมูลดิบที่ Prometheus server จะ scrape เก็บไปทุกๆ ไม่กี่วินาที (ตามค่า `scrape_interval` ที่ตั้งไว้) แล้วนำไปคำนวณอัตราและเปอร์เซ็นไทล์ต่อได้ด้วยภาษา query ชื่อ **PromQL** เช่น:

```promql
# อัตรา request ต่อวินาทีของ path /work ในช่วง 5 นาทีล่าสุด
rate(http_requests_total{path="/work"}[5m])

# p99 latency จาก histogram (คำนวณเปอร์เซ็นไทล์ที่ 99 จาก bucket)
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket{path="/work"}[5m]))
```

การเขียน PromQL แบบเจาะลึกเป็นหัวข้อของตัวมันเองที่กว้างกว่าขอบเขตบทนี้ — สิ่งสำคัญที่ต้องเข้าใจคือ **ฝั่ง Go application มีหน้าที่แค่ "เปิดเผยข้อมูลดิบ" ผ่าน `/metrics` ให้ถูกต้องเท่านั้น ส่วนการคำนวณอัตรา/เปอร์เซ็นไทล์/แจ้งเตือนเป็นหน้าที่ของ Prometheus server และเครื่องมือฝั่ง query ทั้งหมด**

---

## 5. RED Method: Metric สามตัวที่ Dashboard ของทุก Go Service ควรมี

เมื่อมี metric มากมายให้เลือกวัด คำถามคือ "จะเริ่มจากอะไรก่อนดี" — วงการ SRE มีแนวทางที่ได้รับความนิยมมากชื่อ **RED Method** (คิดค้นโดย Tom Wilkie จาก Weaveworks) ที่บอกว่า **สำหรับทุก service ที่รับ request (HTTP, gRPC ฯลฯ) ควรมี metric พื้นฐาน 3 ตัวนี้เป็นอย่างน้อยเสมอ**:

| ตัวอักษร | ชื่อเต็ม | คำถามที่ตอบ | สร้างจาก metric ไหนในหัวข้อ 3 |
|---|---|---|---|
| **R** | **Rate** | มี request เข้ามากี่ครั้งต่อวินาที? | `rate(http_requests_total[5m])` |
| **E** | **Errors** | ในจำนวนนั้น กี่เปอร์เซ็นต์ที่ล้มเหลว (status 5xx)? | `rate(http_requests_total{status=~"5.."}[5m]) / rate(http_requests_total[5m])` |
| **D** | **Duration** | request ที่สำเร็จใช้เวลานานแค่ไหน (โดยเฉพาะที่ percentile สูงๆ เช่น p99)? | `histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))` |

เหตุผลที่ต้องดู **percentile สูง (p95/p99) ไม่ใช่แค่ค่าเฉลี่ย (average)**: สมมติมี request 100 ครั้ง 99 ครั้งตอบภายใน 50ms แต่ 1 ครั้งใช้เวลา 5 วินาที — ค่าเฉลี่ยจะออกมาแค่ประมาณ 100ms ซึ่งดูเหมือนไม่มีปัญหาอะไรเลย ทั้งที่จริงมีผู้ใช้อย่างน้อย 1% ที่เจอประสบการณ์แย่มาก **ค่าเฉลี่ยซ่อนความผิดปกติที่กระทบผู้ใช้ส่วนน้อยได้ง่ายมาก** นี่คือเหตุผลที่ metric ประเภท Histogram (หัวข้อ 3) ถึงสำคัญกว่าการเก็บแค่ผลรวม/ค่าเฉลี่ยตรงๆ — Histogram เก็บการกระจายตัวทั้งหมดไว้ ทำให้คำนวณ percentile ย้อนหลังได้เสมอ

RED Method ตอบคำถาม "ระบบเป็นอย่างไรจากมุมมองของผู้ใช้" ได้ครบทั้ง 3 มุมโดยใช้ metric แค่ 3 ตัว — เหมาะเป็นจุดเริ่มต้นของทุก dashboard ก่อนจะขยายไปดู metric เฉพาะทางอื่นๆ (เช่น จำนวน connection ใน database pool ตามที่เรียนใน **Part 078**, หรือความยาว queue ของ message broker ตามที่เรียนใน **Part 091-092**)

---

## 6. เสาหลักที่ 2 (Logs): ทบทวน `slog` จาก Part 054 ในบทบาทของ Observability

**Part 054** สอนเรื่อง `log/slog` ไว้อย่างละเอียดแล้วในฐานะเครื่องมือ logging มาตรฐานของ Go — ในบริบทของ observability เราแค่เปลี่ยนมุมมองเล็กน้อย: **structured JSON log ที่ `slog` สร้างให้ ไม่ใช่แค่ "ข้อความที่มนุษย์อ่านง่ายขึ้น" แต่คือข้อมูลที่ระบบภายนอก (log aggregator เช่น Loki, Elasticsearch, CloudWatch Logs) สามารถ query แบบมีโครงสร้างได้จริง**

ตัวอย่างเช่น log ธรรมดาแบบไม่มีโครงสร้าง:

```
2026/09/26 handling request for user alice, order 42, took 150ms
```

เทียบกับ structured log ด้วย `slog`:

```go
slog.Info("request handled",
	"user", "alice",
	"order_id", 42,
	"duration_ms", 150,
)
```

```json
{"time":"2026-09-26T...","level":"INFO","msg":"request handled","user":"alice","order_id":42,"duration_ms":150}
```

รูปแบบ JSON ทำให้ query อย่าง **"หา request ทั้งหมดของ user `alice` ที่ใช้เวลานานกว่า 100ms"** เป็นการ query แบบมี field ตรงๆ (`duration_ms > 100 AND user = "alice"`) แทนที่จะต้องเขียน regular expression มาแยก string เอง — ยิ่งระบบมี log จำนวนมหาศาลต่อวินาทีจากหลาย Pod พร้อมกัน (ตามที่เรียนใน **Part 097**) ความแตกต่างนี้ยิ่งมีผลมาก

หลักปฏิบัติที่ดีสำหรับ log ในบริบท observability ที่ควรทบทวนจาก **Part 054**:

- **Log ระดับ `Info` สำหรับเหตุการณ์ปกติที่สำคัญ** (request handled, dependency connected) ไม่ log ทุกบรรทัดโค้ดแบบ debug ยิบย่อยเข้า production เพราะจะทำให้ log ท่วมจนหาของสำคัญไม่เจอ
- **Log ระดับ `Error` ต้องแนบ context ให้ครบ** (error object เอง, ID ที่เกี่ยวข้อง, ค่า input ที่ทำให้เกิด error) — log ที่บอกแค่ `"error occurred"` เปล่าๆ ไม่มีประโยชน์อะไรเวลา debug จริง
- **อย่า log ข้อมูลอ่อนไหว** (password, token, PII) แม้จะเป็นระดับ debug ก็ตาม — เพราะ log มักถูกเก็บไว้นานและมีคนเข้าถึงได้กว้างกว่าฐานข้อมูลจริง (เชื่อมโยงกับหลักการเรื่อง Secret ใน **Part 097** หัวข้อ 7)
- ที่สำคัญที่สุดสำหรับบทนี้: **แนบ trace ID เข้าไปในทุก log line ที่เกี่ยวข้องกับ request เดียวกัน** — รายละเอียดเต็มอยู่ในหัวข้อ 9 หลังจากเรียนเรื่อง tracing ในหัวข้อถัดไปก่อน

---

## 7. เสาหลักที่ 3 (Traces): Distributed Tracing คืออะไร และทำไมต้องมี

ลองจินตนาการ request หนึ่งครั้งจากผู้ใช้ที่ต้องผ่านหลาย service (ตามสถาปัตยกรรม microservices ที่เรียนใน **Part 088**): **API Gateway → Order Service → Payment Service → Database** — ถ้า request นี้ใช้เวลารวม 3 วินาที (ช้าผิดปกติ) **metrics บอกได้แค่ว่า "Order Service ช้า" ในภาพรวม** แต่ **ไม่บอกว่าใน 3 วินาทีนั้น เวลาไปอยู่ที่ไหนบ้างในการเรียกครั้งนี้ครั้งเดียว** — อาจเป็นเพราะ Payment Service ช้า หรือ query database ช้า หรือ network ระหว่าง service ช้า ซึ่งแตกต่างกันไปในแต่ละ request

**Distributed Tracing** แก้ปัญหานี้ด้วยการติดตาม **request เดียวตลอดเส้นทางการเดินทางทั้งหมด** ผ่านแนวคิด 2 อย่าง:

- **Trace** คือการเดินทางทั้งหมดของ request หนึ่งครั้ง ระบุด้วย **Trace ID** ตัวเดียวที่คงที่ตลอดทั้งเส้นทาง (ต้องถูกส่งต่อ (propagate) ไปพร้อมกับทุก call ข้าม service)
- **Span** คือ "หน่วยงานย่อยหนึ่งชิ้น" ภายใน trace นั้น (เช่น "เรียก Payment Service", "query ฐานข้อมูล") แต่ละ span มี **Span ID** ของตัวเอง, มีเวลาเริ่ม-จบชัดเจน, และมี **Parent Span ID** ที่ชี้ไปยัง span ที่เรียกมันขึ้นมา — ทำให้สร้างเป็น**โครงสร้างต้นไม้ (tree)** ของงานทั้งหมดที่เกิดขึ้นภายใน request เดียวได้

เมื่อนำ span ทั้งหมดของ trace เดียวกันมาเรียงตามเวลา จะได้ภาพที่เรียกว่า **flame graph** หรือ **waterfall diagram** ที่เห็นชัดเจนทันทีว่าเวลาส่วนใหญ่ไปอยู่ที่ span ไหน — นี่คือเครื่องมือที่ทรงพลังที่สุดสำหรับ debug ปัญหาประสิทธิภาพใน microservices ที่ metrics และ logs แยกกันตอบไม่ได้ชัดเจนเท่า

**OpenTelemetry** (มักย่อว่า **OTel**) คือมาตรฐานเปิด (เป็นโปรเจกต์ของ CNCF เช่นเดียวกับ Kubernetes และ Prometheus) สำหรับสร้าง เก็บ และส่งออกทั้ง metrics, logs, และ traces แบบไม่ผูกติดกับ vendor ใดวนเดอร์หนึ่ง — เป็นสิ่งที่เข้ามาแทนที่เครื่องมือ tracing เฉพาะทางรุ่นก่อนๆ (เช่น Jaeger client, Zipkin client) ให้กลายเป็นมาตรฐานเดียวที่ใช้ได้กับ backend หลายแบบ

---

## 8. OpenTelemetry Go SDK: สร้างและพิมพ์ Span จริงด้วย Console Exporter

มาสร้าง trace จริงด้วย OpenTelemetry Go SDK (`go.opentelemetry.io/otel` เวอร์ชัน `v1.31.0`) จำลองสถานการณ์ HTTP handler ที่เรียก function ไปดึงข้อมูล user จากฐานข้อมูล — ใช้ **`stdouttrace`** exporter ที่พิมพ์ span ออกทาง stdout แทนที่จะส่งไป backend จริง เพื่อให้เห็นโครงสร้างข้อมูลของ span ชัดเจนโดยไม่ต้องมี infrastructure เพิ่ม:

```go
func fetchUser(ctx context.Context, tracer trace.Tracer, userID int) (string, error) {
	// StartSpan สร้าง child span ใหม่ที่ผูกกับ span ของ ctx ปัจจุบันโดยอัตโนมัติ
	// (ความสัมพันธ์ parent-child นี้คือหัวใจของ distributed tracing)
	ctx, span := tracer.Start(ctx, "db.fetchUser",
		trace.WithAttributes(attribute.Int("user.id", userID)),
	)
	defer span.End()

	time.Sleep(15 * time.Millisecond) // จำลองเวลา query ฐานข้อมูล

	if userID <= 0 {
		err := errors.New("invalid user id")
		span.RecordError(err)
		span.SetStatus(codes.Error, err.Error())
		return "", err
	}

	span.AddEvent("user found in database")
	return "somchai", nil
}

func handleGetUser(ctx context.Context, tracer trace.Tracer, userID int) {
	ctx, span := tracer.Start(ctx, "GET /users/{id}",
		trace.WithAttributes(
			attribute.String("http.method", "GET"),
			attribute.Int("user.id", userID),
		),
	)
	defer span.End()

	name, err := fetchUser(ctx, tracer, userID)
	if err != nil {
		span.SetStatus(codes.Error, "failed to fetch user")
		return
	}
	span.SetAttributes(attribute.String("user.name", name))
}
```

ส่วน setup ของ `TracerProvider`:

```go
exporter, _ := stdouttrace.New(stdouttrace.WithPrettyPrint())

res, _ := resource.New(context.Background(),
	resource.WithAttributes(
		semconv.ServiceName("user-service"),
		semconv.ServiceVersion("1.0.0"),
	),
)

tp := sdktrace.NewTracerProvider(
	sdktrace.WithBatcher(exporter, sdktrace.WithBatchTimeout(time.Second)),
	sdktrace.WithResource(res),
)
defer tp.Shutdown(context.Background())

otel.SetTracerProvider(tp)
tracer := otel.Tracer("user-service")
```

จุดสำคัญที่ต้องเข้าใจ:

- **`tracer.Start(ctx, name, ...)` คืน `ctx` ใหม่ที่ผูก span ปัจจุบันไว้ข้างใน** — ต้องส่ง `ctx` ตัวใหม่นี้ต่อไปยังทุก function ที่เรียกถัดไป (เหมือนหลักการส่งต่อ `context.Context` ที่เรียนไปใน **Part 032** และ **Part 043**) ถ้าลืมส่งต่อ `ctx` ใหม่ span ลูกจะไม่รู้ว่าใครเป็น parent ของมัน ทำให้ trace ขาดออกจากกันเป็นชิ้นๆ แทนที่จะเป็นต้นไม้เดียว
- **`span.SetStatus(codes.Error, ...)`** และ **`span.RecordError(err)`** ทำเครื่องหมายว่า span นี้ล้มเหลว — เมื่อดู trace ผ่าน UI จริง (เช่น Jaeger, Grafana Tempo) span ที่ error จะถูกไฮไลต์เป็นสีแดงให้เห็นทันทีว่าปัญหาอยู่ตรงไหนของ tree
- **`resource.WithAttributes(semconv.ServiceName(...))`** ติดป้ายชื่อ service ให้กับ**ทุก span**ที่สร้างจาก `TracerProvider` ตัวนี้ — สำคัญมากเมื่อมีหลาย service ส่ง trace เข้า backend เดียวกัน (ตามสถาปัตยกรรม microservices ที่เรียนใน **Part 088**) ทำให้แยกได้ว่า span แต่ละอันมาจาก service ไหน

### รันจริงและผลลัพธ์จริง

Build และรัน สร้าง 2 request จำลอง (1 สำเร็จ, 1 ที่ทำให้เกิด error โดยตั้งใจ):

```bash
$ go run .
```

**ผลลัพธ์จริงจากการรัน** (log ข้อความ):

```
2026/09/26 06:00:53 === request 1: user id ที่ถูกต้อง ===
2026/09/26 06:00:53 request succeeded: user=somchai
2026/09/26 06:00:53 === request 2: user id ที่ไม่ถูกต้อง (เพื่อดู error span) ===
2026/09/26 06:00:53 request failed: invalid user id
```

ตามด้วย span จริงที่ `stdouttrace` พิมพ์ออกมา (ตัดมาเฉพาะส่วนสำคัญของ span `db.fetchUser` ในกรณี error):

```json
{
  "Name": "db.fetchUser",
  "SpanContext": {
    "TraceID": "f5f4735988ba10d2814b59247b11f8f6",
    "SpanID": "60f283d3d091e449"
  },
  "Parent": {
    "TraceID": "f5f4735988ba10d2814b59247b11f8f6",
    "SpanID": "1baec1d0677797c2"
  },
  "Attributes": [{"Key": "user.id", "Value": {"Type": "INT64", "Value": -1}}],
  "Events": [{
    "Name": "exception",
    "Attributes": [
      {"Key": "exception.type", "Value": {"Type": "STRING", "Value": "*errors.errorString"}},
      {"Key": "exception.message", "Value": {"Type": "STRING", "Value": "invalid user id"}}
    ]
  }],
  "Status": {"Code": "Error", "Description": "invalid user id"}
}
```

สังเกตสิ่งที่พิสูจน์ทฤษฎีในหัวข้อ 7 ได้ตรงๆ จากข้อมูลจริงนี้:

- **`TraceID` ของ span `db.fetchUser` (`f5f47359...`) ตรงกับ `TraceID` ของ span แม่ `GET /users/{id}` (`Parent.TraceID`) เป๊ะ** — ยืนยันว่าทั้งสอง span อยู่ใน trace เดียวกันจริง แม้จะเป็นคนละ span
- **`Parent.SpanID` (`1baec1d0...`) ตรงกับ `SpanID` ของ span แม่พอดี** — ยืนยันโครงสร้าง parent-child ที่เกิดจากการส่งต่อ `ctx` ที่ได้จาก `tracer.Start()` ของ span แม่เข้าไปใน `fetchUser`
- **`span.RecordError(err)`** สร้าง `Events` ชื่อ `"exception"` พร้อม `exception.type`/`exception.message` ให้อัตโนมัติ ตรงตามที่เรียกในโค้ด และ **`Status.Code: "Error"`** ถูกตั้งค่าจาก `span.SetStatus(codes.Error, ...)` จริง

request แรก (userID ที่ถูกต้อง) จะให้ span `db.fetchUser` ที่มี `Status.Code: "Unset"` (ไม่มี error) และ `Events` เป็น `"user found in database"` แทน — เห็นความแตกต่างชัดเจนระหว่าง request ที่สำเร็จกับที่ล้มเหลวจากข้อมูล span ตรงๆ โดยไม่ต้องเดา

---

## 9. เชื่อมสามเสาหลักเข้าด้วยกัน: ฝัง Trace ID ลงใน Log จริง

เสาหลักทั้งสามจะทรงพลังที่สุดเมื่อ**เชื่อมกันได้** — สถานการณ์ที่พบบ่อยที่สุดคือ: เห็น metric ว่า error rate พุ่งขึ้น (เสาที่ 1) → เปิดดู trace ที่ error เพื่อหาว่า span ไหนที่ล้มเหลว (เสาที่ 3) → **แต่ต้องการรายละเอียดของ error message เต็มๆ ที่ log เก็บไว้** (เสาที่ 2) → คำถามคือจะหา log ที่ตรงกับ trace นั้นเจอได้อย่างไรในบรรดา log นับล้านบรรทัดต่อวัน

คำตอบคือ **แนบ Trace ID เดียวกันเข้าไปในทุก log line ที่เกิดขึ้นระหว่าง span นั้นทำงานอยู่** ทำให้ค้นหา log ทั้งหมดที่เกี่ยวข้องกับ trace หนึ่งได้ด้วยการ query แค่ `trace_id = "..."` ตัวเดียว:

```go
// logWithTrace คือ helper ที่ดึง trace_id/span_id จาก context ปัจจุบัน
// มาแนบเข้ากับทุก log line โดยอัตโนมัติ — นี่คือกลไกหลักที่เชื่อม
// เสาหลักที่ 2 (logs) เข้ากับเสาหลักที่ 3 (traces) เข้าด้วยกัน
func logWithTrace(ctx context.Context, logger *slog.Logger, msg string, args ...any) {
	span := trace.SpanFromContext(ctx)
	sc := span.SpanContext()
	if sc.IsValid() {
		args = append(args, "trace_id", sc.TraceID().String(), "span_id", sc.SpanID().String())
	}
	logger.InfoContext(ctx, msg, args...)
}
```

`trace.SpanFromContext(ctx)` คือฟังก์ชันที่ดึง span ปัจจุบันกลับออกมาจาก `ctx` (ตรงข้ามกับ `tracer.Start()` ที่ใส่ span เข้าไปใน `ctx`) — ทำงานได้เพราะ `ctx` ที่ส่งต่อกันไปทั้งโปรแกรม (ตามหลักการ **Part 032**) พก span ปัจจุบันติดตัวไปด้วยเสมอ รันจริง:

```go
ctx, span := tracer.Start(ctx, "GET /orders/42")
defer span.End()

logWithTrace(ctx, logger, "request started", "method", "GET", "path", "/orders/42")
logWithTrace(ctx, logger, "order fetched from database", "order_id", 42)
logWithTrace(ctx, logger, "request completed", "status", 200)
```

**ผลลัพธ์จริงจากการรัน**:

```json
{"time":"...","level":"INFO","msg":"request started","method":"GET","path":"/orders/42","trace_id":"9aad7eb8850c7e04e9a3050969571ed3","span_id":"cea5b7d2d0532cbc"}
{"time":"...","level":"INFO","msg":"order fetched from database","order_id":42,"trace_id":"9aad7eb8850c7e04e9a3050969571ed3","span_id":"cea5b7d2d0532cbc"}
{"time":"...","level":"INFO","msg":"request completed","status":200,"trace_id":"9aad7eb8850c7e04e9a3050969571ed3","span_id":"cea5b7d2d0532cbc"}
```

**ทั้งสามบรรทัด log มี `trace_id` เดียวกันเป๊ะ** (`9aad7eb8...`) ยืนยันว่าทั้งหมดเกิดขึ้นภายใน request/span เดียวกันจริง — ถ้าเป็นระบบจริงที่มี log aggregator (Loki, Elasticsearch) ต่ออยู่ การเห็น trace หนึ่งใน tracing UI ที่ error แล้วต้องการดู log โดยละเอียดของ request นั้น ก็แค่คัดลอก `trace_id` นี้ไป query ใน log aggregator ได้ทันที — **นี่คือจุดที่ metrics บอกว่า "มีปัญหา", traces บอกว่า "ปัญหาอยู่ตรงไหน", และ logs บอกว่า "เกิดอะไรขึ้นแน่ๆ" ทำงานประสานกันเป็นกระบวนการเดียว**

ในทางปฏิบัติจริง มักไม่ต้องเขียน helper `logWithTrace` เองแบบข้างบน เพราะมี library เชื่อม OpenTelemetry เข้ากับ `slog` โดยตรง (เช่น `go.opentelemetry.io/contrib/bridges/otelslog` หรือเขียน custom `slog.Handler` ที่ดึง trace context อัตโนมัติทุกครั้งที่ log) แต่หลักการเบื้องหลังเหมือนกันทุกประการกับที่แสดงในตัวอย่างนี้

---

## 10. Grafana: ชั้นแสดงผลของ Prometheus (แนวคิด และหน้าตา Dashboard ที่ควรมี)

Prometheus เก็บข้อมูลดิบและให้ query ผ่าน PromQL ได้ แต่**ไม่ได้ออกแบบมาเป็นเครื่องมือแสดงผลกราฟสวยๆ** — **Grafana** คือเครื่องมือ visualization ที่นิยมที่สุดสำหรับต่อกับ Prometheus (รวมถึงต่อกับ Loki สำหรับ logs และ Tempo/Jaeger สำหรับ traces ได้ในตัวเดียวกัน ทำให้ดูทั้งสามเสาหลักในที่เดียวได้) การตั้ง Grafana server จริงต้องมี infrastructure เพิ่มเติม (Grafana server + Prometheus server ที่ scrape metric จริง) ซึ่งเกินขอบเขตที่จะรันสาธิตในแซนด์บ็อกซ์ของหลักสูตรนี้ได้ (ดูหมายเหตุในหัวข้อ 12) แต่แนวคิดสำคัญที่ต้องเข้าใจมีดังนี้:

- Grafana เชื่อมต่อกับ Prometheus ในฐานะ **data source** แล้วผู้ใช้เขียน PromQL query ต่อ panel แต่ละอันเพื่อวาดกราฟ
- **Dashboard** คือชุดของ panel ที่จัดวางร่วมกันในหน้าเดียว โดยทั่วไปแชร์ **time range** เดียวกัน (เลื่อนดูย้อนหลัง 1 ชั่วโมง/1 วัน/1 สัปดาห์พร้อมกันทุก panel)
- ทีมมักสร้าง **Alert Rule** ผูกกับ query (เช่น "ถ้า error rate เกิน 5% ติดต่อกัน 5 นาที ให้แจ้งเตือนเข้า Slack/PagerDuty") แยกออกมาจากตัว dashboard เอง

### หน้าตา Dashboard ที่มีประโยชน์สำหรับ Go Service หนึ่งตัว (ยึดตาม RED Method จากหัวข้อ 5)

| Panel | Query แนวคิด | สิ่งที่มองหา |
|---|---|---|
| **Request Rate** (กราฟเส้นตามเวลา) | `rate(http_requests_total[5m])` แยกตาม `path` | รูปแบบโหลดปกติเป็นอย่างไร มีช่วง peak ตรงไหน มีการเปลี่ยนแปลงกะทันหันหรือไม่ |
| **Error Rate** (กราฟเส้น หรือ single stat เป็น %) | `rate(http_requests_total{status=~"5.."}[5m]) / rate(http_requests_total[5m])` | ควรใกล้ 0% เสมอ — ขึ้นกะทันหันคือสัญญาณอันตรายอันดับหนึ่งที่ต้องสนใจก่อนเรื่องอื่น |
| **p50 / p95 / p99 Latency** (กราฟเส้นหลายเส้นซ้อนกัน) | `histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))` | p50 ปกติควรนิ่ง แต่ p99 มักผันผวนมากกว่า — ช่องว่างระหว่าง p50 กับ p99 ที่กว้างขึ้นเรื่อยๆ คือสัญญาณเตือนล่วงหน้าก่อนที่ผู้ใช้ส่วนใหญ่จะรู้สึกถึงปัญหา |
| **In-Flight Requests** (gauge) | `http_requests_in_flight` | ดูว่ามี request ค้างอยู่ผิดปกติหรือไม่ (สัญญาณของ dependency ที่ตอบช้าหรือค้าง) |
| **Pod Count / CPU / Memory** (จาก Kubernetes metrics) | มาจาก metrics-server ที่กล่าวถึงใน Part 097 หัวข้อ 9 | ดูว่า HPA scale ทำงานถูกต้องตามโหลดจริงหรือไม่ |

Dashboard แบบนี้มักเป็นหน้าจอแรกที่ทีมเปิดดูเวลามีการแจ้งเตือนเข้ามา (จาก Alert Rule) หรือเปิดค้างไว้บนจอในห้อง on-call — ความสามารถในการอ่าน dashboard แบบนี้ให้ออกและรู้ว่าควรมองอะไรก่อน เป็นทักษะที่มีค่ามากพอๆ กับความสามารถในการเขียนโค้ดสร้าง metric ขึ้นมาตั้งแต่แรก

---

## 11. จาก Console Exporter สู่ Production: OTLP Exporter และ Collector (แนวคิด)

ตัวอย่างในหัวข้อ 8-9 ใช้ `stdouttrace` (console exporter) เพื่อให้เห็นโครงสร้างข้อมูลของ span ได้ตรงๆ โดยไม่ต้องมี infrastructure เพิ่มเติม — **ในระบบ production จริง จะเปลี่ยนไปใช้ OTLP exporter แทน** (`go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracegrpc` หรือ `otlptracehttp`) ที่ส่ง span ผ่านโปรโตคอลมาตรฐาน **OTLP (OpenTelemetry Protocol)** ไปยัง **OpenTelemetry Collector** ซึ่งเป็นตัวกลางที่รับข้อมูลจากหลายแอปพร้อมกัน แล้วส่งต่อไปยัง backend การเก็บข้อมูลจริง (Jaeger, Grafana Tempo, หรือ cloud vendor ต่างๆ)

จุดที่สำคัญที่สุดที่ต้องเข้าใจคือ **โค้ดฝั่งแอปที่เขียนไว้ในหัวข้อ 8 (การสร้าง span, `tracer.Start()`, `span.SetAttributes()`) แทบไม่ต้องเปลี่ยนเลยแม้แต่บรรทัดเดียว** เปลี่ยนแค่ตอน setup `TracerProvider` ให้สลับ exporter จาก `stdouttrace.New()` เป็น `otlptracegrpc.New()` เท่านั้น — นี่คือประโยชน์ตรงของสถาปัตยกรรมแบบ **exporter pattern** ที่ OpenTelemetry ออกแบบไว้: โค้ดที่สร้างข้อมูล telemetry (instrumentation) แยกออกจากปลายทางที่ข้อมูลจะถูกส่งไป (exporter) อย่างชัดเจน ทำให้เปลี่ยน backend การมอนิเตอร์ทีหลังได้โดยไม่ต้องแก้โค้ด business logic เลย

---

## 12. หมายเหตุความซื่อสัตย์เรื่องการรันจริงในบทนี้

- **รันจริงและยืนยันผลได้ (หัวข้อ 3-4)**: HTTP server ที่เปิด `/metrics` ด้วย `github.com/prometheus/client_golang v1.20.5` จริง, ยิง request จริงด้วย `curl` และอ่านค่า Counter/Gauge/Histogram จาก `/metrics` จริงตามที่แสดงไว้ทั้งหมด — รันบนเครื่องที่เขียนหลักสูตรนี้ด้วย Go 1.24.7
- **รันจริงและยืนยันผลได้ (หัวข้อ 8)**: โปรแกรม OpenTelemetry Go SDK (`go.opentelemetry.io/otel v1.31.0`) ที่สร้าง parent/child span จริง พร้อม `stdouttrace` exporter พิมพ์ span ออกทาง stdout จริง รวมถึง error span ที่มี `RecordError`/`SetStatus` ทำงานถูกต้องตามที่แสดง
- **รันจริงและยืนยันผลได้ (หัวข้อ 9)**: การเชื่อม trace ID เข้ากับ `slog` log line จริง ยืนยันว่า `trace_id` ตรงกันทุกบรรทัดภายใน span เดียวกันจริง
- **ไม่ได้รันจริง**: Prometheus server ตัวจริงที่ scrape `/metrics` (แสดงเฉพาะฝั่งแอปที่เปิดเผยข้อมูลถูกต้อง ไม่ได้ตั้ง Prometheus server มา scrape จริงในสภาพแวดล้อมนี้), Grafana server และ dashboard จริง (หัวข้อ 10 อธิบายเชิงแนวคิดและอ้างอิงจากการออกแบบ dashboard มาตรฐานตาม RED Method เท่านั้น), และ OTLP Collector/backend จริงอย่าง Jaeger (หัวข้อ 11 อธิบายเชิงแนวคิดจากเอกสารทางการของ OpenTelemetry) — ทั้งสามส่วนนี้ต้องการ service เพิ่มเติมที่รันต่อเนื่อง (long-running server) ซึ่งเกินขอบเขตของสภาพแวดล้อมแซนด์บ็อกซ์แบบ short-lived ที่ใช้เขียนหลักสูตรนี้ แต่โค้ดฝั่ง Go application ที่แสดงไว้ทั้งหมดคือโค้ดจริงที่ใช้ต่อกับ Prometheus/OpenTelemetry backend ได้ตรงๆ โดยไม่ต้องแก้ไข

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- **Observability** มีสามเสาหลักที่เสริมกัน: **Metrics** (ภาพรวมเชิงตัวเลข), **Logs** (รายละเอียดของแต่ละเหตุการณ์), **Traces** (เส้นทางของ request เดียวข้ามทั้งระบบ) — ไม่มีเสาไหนแทนที่อีกเสาได้ทั้งหมด
- **Prometheus** ใช้โมเดล **pull-based scraping** ผ่าน endpoint `/metrics` ต่างจาก logging ที่ push เข้าไปเก็บ — ฝั่ง Go ใช้ `client_golang` สร้าง **Counter** (เพิ่มอย่างเดียว), **Gauge** (ขึ้นลงได้), **Histogram** (การกระจายตัว) ได้จริง และเปิด `/metrics` ผ่าน `promhttp.Handler()` ที่เป็น `http.Handler` ธรรมดา ต่อเข้ากับ mux ของ Part 047 ได้ทันที
- **RED Method** (Rate, Errors, Duration) คือ metric สามตัวขั้นต่ำที่ทุก service ที่รับ request ควรมี — Duration ต้องดูที่ percentile สูง (p99) ไม่ใช่แค่ค่าเฉลี่ย เพราะค่าเฉลี่ยซ่อนปัญหาที่กระทบผู้ใช้ส่วนน้อยได้ง่าย
- **`slog`** จาก **Part 054** คือเครื่องมือของเสาหลัก Logs — JSON structured log ทำให้ query แบบมี field ได้จริง ต่างจาก plain text log ที่ต้องพึ่ง regex
- **Distributed Tracing** ติดตาม request หนึ่งครั้งผ่านโครงสร้าง **Trace** (การเดินทางทั้งหมด, มี Trace ID เดียวคงที่) และ **Span** (หน่วยงานย่อย มี parent-child relationship) — **OpenTelemetry** คือมาตรฐานเปิดสำหรับสร้างและส่งออกข้อมูลนี้ ทดสอบจริงด้วย console exporter แล้วยืนยันว่า Trace ID/Span ID/Parent relationship ถูกต้องตามทฤษฎีทุกจุด
- การเชื่อมสามเสาหลักเข้าด้วยกันทำได้จริงด้วยการฝัง **trace ID** ลงในทุก log line ที่เกิดในบริบทของ span เดียวกัน — พิสูจน์จริงว่า log สามบรรทัดจาก request เดียวมี `trace_id` ตรงกันทั้งหมด
- **Grafana** คือชั้นแสดงผลมาตรฐานสำหรับ Prometheus (และเชื่อมกับ Loki/Tempo ได้ในตัวเดียว) — dashboard ที่ดีของ Go service เริ่มจาก RED Method ก่อนเสมอ
- Instrumentation code (การสร้าง span/metric) แยกออกจาก exporter อย่างชัดเจนในสถาปัตยกรรมของ OpenTelemetry ทำให้เปลี่ยนจาก console exporter ไปเป็น OTLP exporter สู่ production ได้โดยแทบไม่ต้องแก้โค้ด business logic เลย

---

## 13. สรุปภาพรวมภาคที่ 9 (DevOps และ Deployment) ทั้งภาค

ภาคที่ 9 พา Go service ตัวหนึ่งเดินทางครบทุกขั้นตอนตั้งแต่ **"โค้ดที่รันบนเครื่องเรา"** ไปจนถึง **"ระบบ production ที่ deploy อัตโนมัติและมองเห็นสถานะได้ตลอดเวลา"**:

1. **Part 095 (Docker)** — แปลง Go binary ให้เป็น container image ที่เล็กที่สุดเท่าที่เป็นไปได้ ด้วย multi-stage build, `CGO_ENABLED=0`, และ `scratch`/`distroless`
2. **Part 096 (Docker Compose)** — รันหลาย service (API + PostgreSQL + Redis) ร่วมกันบนเครื่องเดียว พร้อมแก้ปัญหา `depends_on` ที่ไม่รอ service ให้พร้อมจริงด้วย healthcheck และ retry loop สองชั้น
3. **Part 097 (Kubernetes)** — ยกระดับจากเครื่องเดียวสู่หลายเครื่องพร้อมกัน ด้วย Deployment, Service, ConfigMap/Secret, probe, และ autoscaling — ในระดับความรู้ของ App Developer ที่ deploy/debug แอปตัวเองได้
4. **Part 098 (CI/CD)** — ทำให้ทุกขั้นตอนก่อนหน้าเกิดขึ้นอัตโนมัติทุกครั้งที่มีการเปลี่ยนโค้ด ด้วยหลักการ fail-fast และ pipeline ที่ตรวจสอบครบตั้งแต่ lint ไปจนถึง build & push image
5. **Part 099 (Observability)** — ให้ทีมมองเห็นสถานะของระบบที่ deploy ไปแล้วได้ตลอดเวลา ผ่านสามเสาหลัก metrics/logs/traces แทนที่จะรอให้ผู้ใช้เป็นคนแจ้งปัญหาเข้ามาก่อน

ทักษะทั้ง 5 บทนี้คือสิ่งที่แยก **"คนที่เขียนโค้ด Go เก่ง"** ออกจาก **"Go Developer ที่พร้อมทำงานในทีมวิศวกรรมซอฟต์แวร์สมัยใหม่ได้จริง"** — เพราะในโลกการทำงานจริง โค้ดที่ดีที่สุดก็ไร้ความหมายถ้า deploy ไม่ได้ ดูแลไม่ได้ หรือไม่มีใครรู้ว่ามันกำลังทำงานผิดปกติอยู่หรือไม่

จากนี้หลักสูตรจะเข้าสู่ **ภาคที่ 10: มืออาชีพและระดับโลก (Professional & World-Class)** ที่นำทุกอย่างที่เรียนมาตลอด 9 ภาค (ภาษา, concurrency, standard library, web, database, testing, microservices, และ DevOps) มารวมกันเป็นสถาปัตยกรรมซอฟต์แวร์ระดับมืออาชีพ เริ่มต้นด้วย **Clean Architecture** ในบทถัดไป

## แบบฝึกหัดท้ายบท

1. รันโปรแกรมสาธิต `/metrics` จากหัวข้อ 4 บนเครื่องตัวเอง เพิ่ม endpoint ใหม่ที่จำลอง error แบบสุ่มบ่อยขึ้น (เช่น 30% ของ request) แล้วสังเกตการเปลี่ยนแปลงของ `http_requests_total{status=...}` ระหว่าง `OK` กับ status อื่น
2. เพิ่ม label ใหม่ให้ `httpRequestsTotal` เป็น `method` (GET/POST) นอกเหนือจาก `path`/`status` ที่มีอยู่แล้ว แล้วทดสอบว่า metric แยกตาม method ได้ถูกต้องจริง
3. รันโปรแกรม OpenTelemetry จากหัวข้อ 8 แล้วเพิ่ม span ลูกอีกชั้น (เช่น `cache.checkRedis` ที่เป็น child ของ `db.fetchUser`) จำลองการเช็ค cache ก่อน query ฐานข้อมูลจริงตาม cache-aside pattern จาก **Part 096** สังเกตว่า `Parent`/`TraceID` ของ span ใหม่ถูกต้องตามที่คาดหรือไม่
4. เขียน helper อย่าง `logWithTrace` จากหัวข้อ 9 ให้เป็น `slog.Handler` ที่กำหนดเอง (custom handler ตามที่เรียนแนวคิดพื้นฐานจาก **Part 054**) แทนการเรียกฟังก์ชัน wrapper ทุกครั้ง เพื่อให้ trace ID ถูกแนบอัตโนมัติทุกครั้งที่เรียก `logger.InfoContext(ctx, ...)` โดยไม่ต้องเขียนโค้ดเพิ่มที่ call site เลย
5. ติดตั้ง Prometheus และ Grafana จริงด้วย Docker Compose (ทบทวนเทคนิคจาก **Part 096** — เขียน `docker-compose.yml` ที่มี service `prometheus`, `grafana`, และ `goapi` จากหัวข้อ 4) ตั้งค่า `prometheus.yml` ให้ scrape endpoint `/metrics` ของ `goapi` แล้วสร้าง dashboard บน Grafana ตาม RED Method ในหัวข้อ 10
6. ค้นคว้าเพิ่มเติมเรื่อง **`otlptracegrpc`** exporter และ **OpenTelemetry Collector** — ลองตั้งค่า Collector บนเครื่องตัวเอง (ผ่าน Docker) แล้วเปลี่ยน exporter ในโค้ดหัวข้อ 8 จาก `stdouttrace` เป็น `otlptracegrpc` ส่ง trace เข้า Collector แล้วต่อไปยัง Jaeger UI เพื่อดู flame graph ของ trace จริงแบบภาพ

---

**ต่อไป**: [Part 100 — Clean Architecture ใน Go](./100-clean-architecture.md)
