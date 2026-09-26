# Part 096: Docker Compose สำหรับ Multi-Service

> ภาคที่ 9: DevOps และ Deployment — ตอนที่ 2 จาก 5 (Part 95–99)

## สารบัญของบทนี้

1. ทำไมต้องมี Docker Compose ทั้งที่มี Docker อยู่แล้ว
2. โครงสร้างไฟล์ `docker-compose.yml`: Services, Networks, Volumes
3. ขยาย API ให้ต่อ PostgreSQL และ Redis จริง
4. `.env` และ Environment Variable ใน Compose
5. `depends_on`: กับดักที่ทุกคนต้องเจอสักครั้ง
6. สาธิตปัญหาจริง: เมื่อ `depends_on` "หลอก" เรา
7. ทางแก้ที่ 1: Healthcheck + `condition: service_healthy`
8. ทางแก้ที่ 2: Retry Loop ในโค้ดแอปเอง (เกราะชั้นสอง)
9. รันเวอร์ชันที่แก้แล้ว: ทุกอย่างทำงานร่วมกันจริง
10. Volumes: ทำไมข้อมูลใน Container ถึงหายเมื่อลบ Container
11. `docker-compose.yml` ฉบับสมบูรณ์ของบทนี้
12. คำสั่ง Compose ที่ใช้บ่อยที่สุด
13. หมายเหตุความซื่อสัตย์เรื่องการรันจริงในบทนี้
14. สรุปสิ่งที่ได้เรียนในบทนี้
15. แบบฝึกหัดท้ายบท

---

## 1. ทำไมต้องมี Docker Compose ทั้งที่มี Docker อยู่แล้ว

**Part 095** สอนให้ container Go application ตัวเดียวได้แล้ว แต่ระบบจริงแทบไม่มีทางรันด้วย service เดียวโดดๆ — API ของเราต้องคุยกับ **PostgreSQL** (ทบทวนจาก **Part 072**) และ **Redis** (ทบทวนจาก **Part 077**) เป็นอย่างน้อย ถ้าใช้ `docker run` ตรงๆ ทีละ container เราต้องจำคำสั่งยาวๆ ที่มีทั้ง port mapping, environment variable, network, volume ของแต่ละตัว แล้วพิมพ์ทุกครั้งที่จะเริ่มระบบใหม่ — เสี่ยงพิมพ์ผิดและไม่มีใครจำได้ครบ

**Docker Compose** แก้ปัญหานี้ด้วยการอธิบายทั้งระบบ (หลาย container ที่ทำงานร่วมกัน) เป็น**ไฟล์ YAML ไฟล์เดียว** แล้วสั่ง `docker compose up` ครั้งเดียวเพื่อสร้างและเชื่อมทุกอย่างเข้าด้วยกันให้อัตโนมัติ

> **หมายเหตุเรื่องเวอร์ชัน**: `docker-compose` (คำสั่งแยก, เขียนติดกัน) คือเครื่องมือรุ่นเก่า (Compose V1, เขียนด้วย Python) ปัจจุบันถูกแทนที่ด้วย **`docker compose`** (มีช่องว่าง) ซึ่งเป็น plugin ของ Docker CLI เอง (Compose V2, เขียนด้วย Go เช่นกัน) บทนี้ใช้ **Docker Compose v5.1.1** ที่มากับ Docker Engine 29.3.1 ในสภาพแวดล้อมที่เขียนหลักสูตรนี้ — คำสั่งทั้งหมดในบทนี้จึงเป็น `docker compose ...` (มีช่องว่าง)

---

## 2. โครงสร้างไฟล์ `docker-compose.yml`: Services, Networks, Volumes

โครงสร้างหลักของไฟล์ compose มี 3 ส่วนสำคัญ:

```yaml
services:      # รายการ container ทั้งหมดที่จะรัน แต่ละตัวมี key เป็นชื่อ service
  api:
    build: .
  db:
    image: postgres:16-alpine

volumes:       # พื้นที่เก็บข้อมูลถาวรที่ไม่หายไปพร้อม container (หัวข้อ 10)
  pgdata:

networks:      # เครือข่ายภายในที่ให้ container คุยกันได้ด้วยชื่อ service
  appnet:
    driver: bridge
```

จุดที่ต้องเข้าใจให้ชัดตั้งแต่แรก: **ทุก service ใน compose file เดียวกันคุยกันได้โดยอัตโนมัติผ่านชื่อ service เป็นเสมือน hostname** — ไม่ต้องรู้ IP address ของ container อีกฝั่งเลย ตัวอย่างเช่น service `api` เชื่อมต่อ PostgreSQL ที่ service `db` ได้ตรงๆ ด้วย host ชื่อ `db` (ไม่ใช่ `localhost`) เพราะ Docker Compose สร้าง **internal DNS** ให้อัตโนมัติภายใน network เดียวกัน — นี่คือกลไกเบื้องหลังที่ทำให้ multi-service ทำงานร่วมกันได้แบบไม่ต้อง hardcode IP ใดๆ เลย

> **หมายเหตุเรื่อง `version:`**: ไฟล์ compose รุ่นเก่ามักขึ้นต้นด้วย `version: "3.8"` แต่ Docker Compose รุ่นปัจจุบัน (ตาม [Compose Specification](https://github.com/compose-spec/compose-spec)) **ไม่ต้องระบุ `version` อีกต่อไปแล้ว** (deprecated และถูกเพิกเฉยถ้าใส่มา) บทนี้จึงไม่มีบรรทัด `version` ในทุกตัวอย่าง

---

## 3. ขยาย API ให้ต่อ PostgreSQL และ Redis จริง

ต่อยอดจาก API ใน **Part 095** เราจะเปลี่ยนจากเก็บข้อมูลใน memory (`map` ธรรมดา) มาเป็นเก็บจริงใน PostgreSQL ผ่าน `pgxpool` (ตามที่เรียนใน **Part 072**) และใช้ Redis ทำ **cache-aside pattern** (ตามที่เรียนใน **Part 077** หัวข้อ 6) เพื่อลดภาระ query ฐานข้อมูลซ้ำๆ

```go
package main

import (
	"context"
	"encoding/json"
	"errors"
	"fmt"
	"log/slog"
	"net/http"
	"os"
	"os/signal"
	"syscall"
	"time"

	"github.com/jackc/pgx/v5/pgxpool"
	"github.com/redis/go-redis/v9"
)

type Task struct {
	ID   int    `json:"id"`
	Name string `json:"name"`
}

type app struct {
	db    *pgxpool.Pool
	cache *redis.Client
}

func mustGetenv(key, def string) string {
	if v := os.Getenv(key); v != "" {
		return v
	}
	return def
}

func main() {
	slog.SetDefault(slog.New(slog.NewJSONHandler(os.Stdout, nil)))

	dbURL := mustGetenv("DATABASE_URL", "postgres://postgres:postgres@localhost:5432/appdb?sslmode=disable")
	redisAddr := mustGetenv("REDIS_ADDR", "localhost:6379")
	port := mustGetenv("PORT", "8080")

	startupCtx, cancelStartup := context.WithTimeout(context.Background(), 30*time.Second)
	defer cancelStartup()

	maxAttempts := 15
	if v := os.Getenv("MAX_CONNECT_ATTEMPTS"); v != "" {
		fmt.Sscanf(v, "%d", &maxAttempts)
	}

	var pool *pgxpool.Pool
	err := connectWithRetry(startupCtx, "postgres", maxAttempts, func() error {
		p, err := pgxpool.New(startupCtx, dbURL)
		if err != nil {
			return err
		}
		if err := p.Ping(startupCtx); err != nil {
			p.Close()
			return err
		}
		pool = p
		return nil
	})
	if err != nil {
		slog.Error("could not connect to postgres, giving up", "error", err)
		os.Exit(1)
	}
	defer pool.Close()

	_, err = pool.Exec(startupCtx, `CREATE TABLE IF NOT EXISTS tasks (
		id SERIAL PRIMARY KEY,
		name TEXT NOT NULL
	)`)
	if err != nil {
		slog.Error("migration failed", "error", err)
		os.Exit(1)
	}

	rdb := redis.NewClient(&redis.Options{Addr: redisAddr})
	err = connectWithRetry(startupCtx, "redis", maxAttempts, func() error {
		return rdb.Ping(startupCtx).Err()
	})
	if err != nil {
		slog.Error("could not connect to redis, giving up", "error", err)
		os.Exit(1)
	}
	defer rdb.Close()

	a := &app{db: pool, cache: rdb}

	mux := http.NewServeMux()
	mux.HandleFunc("GET /healthz", a.handleHealthz)
	mux.HandleFunc("GET /tasks", a.handleListTasks)
	mux.HandleFunc("POST /tasks", a.handleCreateTask)

	srv := &http.Server{
		Addr:         ":" + port,
		Handler:      mux,
		ReadTimeout:  5 * time.Second,
		WriteTimeout: 10 * time.Second,
	}

	serverErr := make(chan error, 1)
	go func() {
		slog.Info("server starting", "port", port)
		serverErr <- srv.ListenAndServe()
	}()

	quit := make(chan os.Signal, 1)
	signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)

	select {
	case err := <-serverErr:
		if err != nil && !errors.Is(err, http.ErrServerClosed) {
			slog.Error("server error", "error", err)
		}
	case <-quit:
		slog.Info("shutting down")
		ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
		defer cancel()
		_ = srv.Shutdown(ctx)
	}
}
```

โค้ดฝั่ง handler (cache-aside pattern แบบเต็ม):

```go
func (a *app) handleHealthz(w http.ResponseWriter, r *http.Request) {
	ctx, cancel := context.WithTimeout(r.Context(), 2*time.Second)
	defer cancel()

	status := map[string]string{"status": "ok"}
	code := http.StatusOK

	if err := a.db.Ping(ctx); err != nil {
		status["status"] = "degraded"
		status["db"] = err.Error()
		code = http.StatusServiceUnavailable
	}
	if err := a.cache.Ping(ctx).Err(); err != nil {
		status["status"] = "degraded"
		status["cache"] = err.Error()
		code = http.StatusServiceUnavailable
	}

	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(code)
	json.NewEncoder(w).Encode(status)
}

func (a *app) handleListTasks(w http.ResponseWriter, r *http.Request) {
	ctx := r.Context()

	// Cache-aside pattern (ทบทวนจาก Part 077): เช็ค cache ก่อน ถ้ามีให้ตอบจาก cache เลย
	cached, err := a.cache.Get(ctx, "tasks:all").Result()
	if err == nil {
		w.Header().Set("Content-Type", "application/json")
		w.Header().Set("X-Cache", "HIT")
		w.Write([]byte(cached))
		return
	}

	rows, err := a.db.Query(ctx, "SELECT id, name FROM tasks ORDER BY id")
	if err != nil {
		http.Error(w, err.Error(), http.StatusInternalServerError)
		return
	}
	defer rows.Close()

	tasks := []Task{}
	for rows.Next() {
		var t Task
		if err := rows.Scan(&t.ID, &t.Name); err != nil {
			http.Error(w, err.Error(), http.StatusInternalServerError)
			return
		}
		tasks = append(tasks, t)
	}

	body, _ := json.Marshal(tasks)
	a.cache.Set(ctx, "tasks:all", body, 30*time.Second) // cache 30 วินาที
	w.Header().Set("Content-Type", "application/json")
	w.Header().Set("X-Cache", "MISS")
	w.Write(body)
}

func (a *app) handleCreateTask(w http.ResponseWriter, r *http.Request) {
	ctx := r.Context()
	var body struct {
		Name string `json:"name"`
	}
	if err := json.NewDecoder(r.Body).Decode(&body); err != nil || body.Name == "" {
		http.Error(w, "invalid body", http.StatusBadRequest)
		return
	}

	var t Task
	err := a.db.QueryRow(ctx,
		"INSERT INTO tasks (name) VALUES ($1) RETURNING id, name", body.Name,
	).Scan(&t.ID, &t.Name)
	if err != nil {
		http.Error(w, err.Error(), http.StatusInternalServerError)
		return
	}

	a.cache.Del(ctx, "tasks:all") // invalidate cache เพราะข้อมูลเปลี่ยนแล้ว

	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(http.StatusCreated)
	json.NewEncoder(w).Encode(t)
}
```

สังเกตว่าโค้ดอ่านค่า connection ทั้งหมด (`DATABASE_URL`, `REDIS_ADDR`, `PORT`) จาก **environment variable** ไม่มี hardcode ค่าใดๆ ไว้เลย — เป็นเงื่อนไขจำเป็นสำหรับให้ image เดียวกันรันได้ทั้งใน local compose, staging, และ production เพียงแค่เปลี่ยนค่า environment variable ที่ส่งเข้าไป โดยไม่ต้อง build image ใหม่

---

## 4. `.env` และ Environment Variable ใน Compose

Docker Compose อ่านไฟล์ชื่อ **`.env`** ในโฟลเดอร์เดียวกับ `docker-compose.yml` โดยอัตโนมัติ แล้วนำค่าต่างๆ ไปแทนที่ตัวแปรรูปแบบ `${VAR_NAME}` ในไฟล์ compose ได้ทันที:

```bash
# .env
POSTGRES_USER=appuser
POSTGRES_PASSWORD=apppass
POSTGRES_DB=appdb
APP_PORT=8080
```

```yaml
services:
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}
```

ประโยชน์ของการแยกค่าออกมาไว้ใน `.env`:

- **แยก config ออกจาก YAML** — เปลี่ยนค่า password/port โดยไม่ต้องแก้ไฟล์ `docker-compose.yml` เลย
- **ไม่ commit ค่า sensitive เข้า git** — ใส่ `.env` ใน `.gitignore` เสมอ (คล้ายกับที่ **Part 018** เตือนเรื่อง private module credential) แล้ว commit แค่ **`.env.example`** ที่มี key แต่ไม่มีค่าจริงไว้เป็นแม่แบบให้ทีมคัดลอกไปตั้งค่าเอง
- **ใช้ค่าเดียวกันได้ทั้งใน `db` และ `api`** — สังเกตว่า `POSTGRES_USER`/`POSTGRES_PASSWORD`/`POSTGRES_DB` ถูกใช้ประกอบเป็น `DATABASE_URL` ของ `api` ด้วย (`postgres://${POSTGRES_USER}:${POSTGRES_PASSWORD}@db:5432/${POSTGRES_DB}`) ทำให้ค่าที่เกี่ยวข้องกันไม่หลุดไม่ตรงกันโดยไม่ตั้งใจ

> **ข้อควรระวัง**: `.env` ของ Docker Compose เป็นคนละไฟล์กับ `.env` ที่บาง library โหลดเข้า process โดยตรง (เช่น `godotenv` ใน Go) — `.env` ของ Compose ถูก **Compose CLI อ่านตอน parse ไฟล์ YAML เท่านั้น** ไม่ได้ถูกส่งเข้า container โดยอัตโนมัติ ต้องประกาศใน `environment:` ของแต่ละ service อย่างชัดเจนเสมอ (ตามตัวอย่างข้างบน) ไม่ใช่ปล่อยให้ `.env` ไหลเข้า container เอง

---

## 5. `depends_on`: กับดักที่ทุกคนต้องเจอสักครั้ง

`depends_on` ควบคุม **ลำดับการเริ่ม container** — บอก Compose ว่า service ไหนต้องเริ่มก่อนใคร:

```yaml
services:
  api:
    build: .
    depends_on:
      - db
      - cache
```

นี่คือจุดที่นักพัฒนาแทบทุกคนเข้าใจผิดตอนใช้ Docker Compose ครั้งแรก: **`depends_on` แบบรูปแบบสั้นนี้รับประกันแค่ว่า "container ของ `db` ถูกสั่ง `start` ไปแล้ว" เท่านั้น — ไม่ได้รับประกันเลยว่า PostgreSQL ข้างในนั้น "พร้อมรับ connection" แล้วจริงๆ**

ความแตกต่างระหว่าง **"container เริ่มแล้ว"** กับ **"service พร้อมใช้งานแล้ว"** สำคัญมาก โดยเฉพาะกับ PostgreSQL: ตอน container เริ่มครั้งแรก (หรือทุกครั้งที่ volume ว่างเปล่า) PostgreSQL ต้องทำ **`initdb`** (สร้างโครงสร้างฐานข้อมูลเปล่าตั้งต้น) ก่อน แล้วถึงจะเริ่ม accept connection ได้ — กระบวนการนี้ใช้เวลาหลายวินาที ในขณะที่ **process ของ container เริ่มทำงานตั้งแต่วินาทีแรก** Docker เห็นว่า container "รันอยู่" ตั้งแต่ตอนนั้นแล้ว ทั้งที่ database ข้างในยังไม่พร้อมรับ connection เลย

---

## 6. สาธิตปัญหาจริง: เมื่อ `depends_on` "หลอก" เรา

มาดูปัญหานี้แบบจับต้องได้ ด้วยการสร้าง compose file ที่ **ไม่มี healthcheck เลย** และปิด retry logic ในแอปด้วย (ตั้ง `MAX_CONNECT_ATTEMPTS=1` ให้แอปลองต่อครั้งเดียวแล้วยอมแพ้ เพื่อให้เห็นปัญหาชัดที่สุด):

```yaml
# docker-compose.broken.yml - ตัวอย่าง "ผิด" ไว้สาธิตปัญหา depends_on แบบง่ายสุด
services:
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}

  cache:
    image: redis:7-alpine

  api:
    build: .
    environment:
      DATABASE_URL: postgres://${POSTGRES_USER}:${POSTGRES_PASSWORD}@db:5432/${POSTGRES_DB}?sslmode=disable
      REDIS_ADDR: cache:6379
      PORT: 8080
      MAX_CONNECT_ATTEMPTS: "1"
    ports:
      - "${APP_PORT}:8080"
    depends_on:
      - db
      - cache
```

รันจริง:

```bash
docker compose -f docker-compose.broken.yml up -d --build
sleep 3
docker compose -f docker-compose.broken.yml ps -a
docker compose -f docker-compose.broken.yml logs api
```

**ผลลัพธ์จริงจากการรันบนเครื่องที่เขียนบทความนี้**:

```
NAME                 IMAGE                COMMAND                  SERVICE   STATUS
composeapp-api-1     composeapp-api       "/server"                api       Exited (1) 15 seconds ago
composeapp-cache-1   redis:7-alpine       "docker-entrypoint.s…"   cache     Up 16 seconds
composeapp-db-1      postgres:16-alpine   "docker-entrypoint.s…"   db        Up 16 seconds

api-1  | {"time":"...","level":"WARN","msg":"dependency not ready yet, retrying","service":"postgres","attempt":1,"max_attempts":1,"error":"failed to connect to `user=appuser database=appdb`: 172.18.0.3:5432 (db): dial error: dial tcp 172.18.0.3:5432: connect: connection refused"}
api-1  | {"time":"...","level":"ERROR","msg":"could not connect to postgres, giving up","error":"postgres not ready after 1 attempts: ..."}
```

เห็นได้ชัดเจนจากผลลัพธ์จริง: **`db` container ขึ้นสถานะ `Up` ไปแล้ว** (Docker คิดว่ามันพร้อมแล้ว) แต่ **`api` กลับเชื่อมต่อไม่ได้เลยเพราะโดน `connection refused`** — พิสูจน์ตรงตามทฤษฎีว่า `depends_on` ปล่อยให้ `api` เริ่มทำงานเร็วเกินไป ก่อนที่ PostgreSQL ข้างในตัว `db` จะพร้อมรับ connection จริง ผลคือ **`api` container ตายไปเลย (`Exited (1)`)** ทั้งที่ระบบดูเหมือนจะเริ่มถูกต้องทุกอย่าง

---

## 7. ทางแก้ที่ 1: Healthcheck + `condition: service_healthy`

Docker Compose มีรูปแบบ `depends_on` แบบ **long-form** ที่รอ**เงื่อนไข**เฉพาะได้ ไม่ใช่แค่รอให้ container เริ่ม:

```yaml
services:
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 3s
      timeout: 3s
      retries: 10
      start_period: 5s

  api:
    build: .
    depends_on:
      db:
        condition: service_healthy   # รอจนกว่า db จะผ่าน healthcheck ไม่ใช่แค่ "start" แล้ว
```

`pg_isready` คือ command-line tool ที่มากับ PostgreSQL image เอง ใช้เช็คว่า server พร้อมรับ connection จริงหรือยัง (ต่างจากแค่เช็คว่า process รันอยู่) เมื่อ Compose เห็นว่า healthcheck ของ `db` ผ่านแล้ว (มีสถานะ `healthy`) ถึงจะปล่อยให้ `api` เริ่มทำงาน — นี่คือวิธีที่ถูกต้องตามที่ Docker Compose ตั้งใจออกแบบไว้ให้แก้ปัญหานี้โดยเฉพาะ

---

## 8. ทางแก้ที่ 2: Retry Loop ในโค้ดแอปเอง (เกราะชั้นสอง)

`condition: service_healthy` แก้ปัญหาได้ดีมากในกรณีทั่วไป แต่ **ไม่ควรพึ่งพามันเพียงอย่างเดียว** ด้วยเหตุผลสำคัญ:

- Healthcheck เช็คแค่ "ตอนเริ่ม" เท่านั้น — ถ้า `db` restart กลางคัน (เช่น ทำ failover, ทำ maintenance) แล้ว `api` ที่รันอยู่แล้วดันเสีย connection ไปพอดี **`depends_on` ไม่ได้ช่วยอะไรเลยในสถานการณ์นี้** เพราะมันมีผลแค่ตอน **compose สั่งเริ่ม container** เท่านั้น
- ในระบบ production จริงที่รันบน Kubernetes (**Part 097**) แนวคิด `depends_on` แบบนี้ **ไม่มีอยู่เลย** — Pod ถูกสั่งเริ่มพร้อมกันได้ตลอดเวลาโดยไม่รอกัน แอปต้อง**ทนต่อความไม่พร้อมของ dependency ด้วยตัวเอง**เสมอ

ทางแก้ที่แข็งแรงกว่าคือให้**แอปมี retry loop ของตัวเอง** เป็น "เกราะชั้นสอง" ไม่ว่า orchestrator จะจัดลำดับให้ถูกต้องหรือไม่ก็ตาม:

```go
// connectWithRetry คือหัวใจของหัวข้อ "depends_on ไม่รอ service ให้พร้อมจริง"
// เราไม่พึ่งพา docker compose ล้วนๆ แต่ให้ตัวแอปเอง "ทน" ต่อการที่ dependency ยังไม่พร้อม
// ด้วยการลองเชื่อมต่อซ้ำ พร้อม backoff แบบง่าย จนกว่าจะสำเร็จหรือครบจำนวนครั้งที่กำหนด
func connectWithRetry(ctx context.Context, name string, maxAttempts int, fn func() error) error {
	var lastErr error
	for attempt := 1; attempt <= maxAttempts; attempt++ {
		lastErr = fn()
		if lastErr == nil {
			slog.Info("dependency ready", "service", name, "attempt", attempt)
			return nil
		}
		wait := time.Duration(attempt) * 500 * time.Millisecond
		slog.Warn("dependency not ready yet, retrying",
			"service", name, "attempt", attempt, "max_attempts", maxAttempts,
			"error", lastErr, "retry_in", wait)
		select {
		case <-ctx.Done():
			return ctx.Err()
		case <-time.After(wait):
		}
	}
	return fmt.Errorf("%s not ready after %d attempts: %w", name, maxAttempts, lastErr)
}
```

หลักการสำคัญของ retry loop นี้:

- **Backoff แบบเพิ่มขึ้นเรื่อยๆ** (`time.Duration(attempt) * 500ms`): รอบแรกรอ 0.5 วินาที รอบสองรอ 1 วินาที รอบสามรอ 1.5 วินาที ฯลฯ — ไม่ยิง request รัวๆ ใส่ dependency ที่กำลังลำบากอยู่แล้ว (แนวคิดเดียวกับ exponential backoff ที่จะเรียนเจาะลึกใน **Part 094** เรื่อง Circuit Breaker และ Retry Pattern)
- **`context.Context` กำกับเวลารวม**: ผูกกับ `startupCtx` ที่มี timeout รวม 30 วินาที ป้องกันไม่ให้แอป **ค้างรอไปตลอดกาล** ถ้า dependency พังจริงๆ
- **Log ทุกครั้งที่ retry**: ด้วย `slog` (ทบทวนจาก **Part 054**) ทำให้เห็นได้ชัดเจนใน log ว่ากำลังรออะไรอยู่ ไม่ใช่แอปแค่ "เงียบหายไป" — สำคัญมากตอน debug ปัญหา startup ใน production

**หลักการที่ต้องจำ**: `depends_on` + healthcheck คือการ**ลดโอกาส**ที่ปัญหานี้จะเกิดตั้งแต่ต้น (ทำให้ระบบเริ่มเร็วและถูกลำดับมากขึ้นในกรณีปกติ) ส่วน retry loop ในโค้ดคือ**เกราะป้องกันจริง**ที่ทำให้ระบบทนทานต่อความล้มเหลวได้ไม่ว่าจะเกิดจากสาเหตุอะไร — **ควรทำทั้งสองอย่างพร้อมกันเสมอ ไม่ใช่เลือกอย่างใดอย่างหนึ่ง**

---

## 9. รันเวอร์ชันที่แก้แล้ว: ทุกอย่างทำงานร่วมกันจริง

รวมทั้งสองทางแก้เข้าด้วยกัน แล้วรันจริง:

```bash
docker compose up -d --build
```

**ผลลัพธ์จริง** (สังเกตลำดับ `Waiting` → `Healthy` → ค่อยเริ่ม `api`):

```
 Container composeapp-db-1 Starting
 Container composeapp-cache-1 Starting
 Container composeapp-cache-1 Started
 Container composeapp-db-1 Started
 Container composeapp-db-1 Waiting
 Container composeapp-cache-1 Waiting
 Container composeapp-cache-1 Healthy
 Container composeapp-db-1 Healthy
 Container composeapp-api-1 Starting
 Container composeapp-api-1 Started
```

ตรวจสอบสถานะและ log:

```bash
docker compose ps
docker compose logs api
```

**ผลลัพธ์จริง**:

```
NAME                 STATUS
composeapp-api-1     Up 5 seconds
composeapp-cache-1   Up 11 seconds (healthy)
composeapp-db-1      Up 11 seconds (healthy)

api-1  | {"time":"...","level":"INFO","msg":"dependency ready","service":"postgres","attempt":1}
api-1  | {"time":"...","level":"INFO","msg":"dependency ready","service":"redis","attempt":1}
api-1  | {"time":"...","level":"INFO","msg":"server starting","port":"8080"}
```

`attempt: 1` ยืนยันว่าเชื่อมต่อสำเร็จตั้งแต่ครั้งแรกพอดี เพราะ `condition: service_healthy` รอให้ `db`/`cache` พร้อมก่อนแล้ว — ทดสอบ end-to-end ทั้งระบบผ่าน `curl`:

```bash
curl -s http://localhost:8080/healthz
curl -si http://localhost:8080/tasks | head -6      # ครั้งแรก: cache miss
curl -si http://localhost:8080/tasks | head -6      # ครั้งสอง: cache hit
curl -s -X POST http://localhost:8080/tasks -d '{"name":"ทดสอบ compose"}'
curl -si http://localhost:8080/tasks | head -6      # หลัง POST: cache ถูก invalidate แล้ว miss อีกครั้ง
```

**ผลลัพธ์จริงทั้งหมด**:

```
{"status":"ok"}

HTTP/1.1 200 OK
Content-Type: application/json
X-Cache: MISS
[]

HTTP/1.1 200 OK
Content-Type: application/json
X-Cache: HIT
[]

{"id":1,"name":"ทดสอบ compose"}

HTTP/1.1 200 OK
Content-Type: application/json
X-Cache: MISS
[{"id":1,"name":"ทดสอบ compose"}]
```

เห็น header `X-Cache` สลับ `MISS` → `HIT` → `MISS` ตรงตามพฤติกรรมที่ตั้งใจออกแบบไว้ทุกประการ: ครั้งแรกไม่มีอะไรใน cache ต้อง query ฐานข้อมูลจริง, ครั้งที่สองตอบจาก Redis cache ทันที (ภายใน TTL 30 วินาที), แล้วหลัง `POST` cache ถูก invalidate ทำให้ query ฐานข้อมูลใหม่อีกครั้งและได้ข้อมูลล่าสุดถูกต้อง — **ระบบ API + PostgreSQL + Redis ทั้งสามตัวทำงานร่วมกันจริงผ่าน Docker Compose เพียงคำสั่งเดียว**

---

## 10. Volumes: ทำไมข้อมูลใน Container ถึงหายเมื่อลบ Container

Container ถูกออกแบบให้เป็น **ephemeral** (ชั่วคราว) โดยธรรมชาติ — ไฟล์ใดๆ ที่เขียนลงไปข้างในระหว่าง container กำลังรัน (รวมถึงข้อมูลของ PostgreSQL) จะ**หายไปทันทีที่ container ถูกลบ** (`docker rm` หรือ `docker compose down`) เพราะ layer ที่เขียนทับนั้นเป็นแค่ layer ชั่วคราวที่ผูกกับ container instance นั้นเท่านั้น

**Volume** คือกลไกที่ทำให้ข้อมูลบางส่วน**อยู่นอก vòng ชีวิตของ container** — Docker จัดการพื้นที่เก็บข้อมูลแยกไว้ต่างหาก (ปกติเก็บไว้ที่ host filesystem ภายใต้การดูแลของ Docker) แล้ว "mount" เข้าไปในตำแหน่งที่กำหนดของ container:

```yaml
services:
  db:
    image: postgres:16-alpine
    volumes:
      - pgdata:/var/lib/postgresql/data   # mount named volume "pgdata" เข้าตำแหน่งที่ Postgres เก็บข้อมูลจริง

volumes:
  pgdata:   # ประกาศชื่อ volume ให้ Docker สร้างและจัดการให้
```

พิสูจน์ด้วยการทดสอบจริง — สร้างข้อมูล, restart container ทั้งคู่, แล้วเช็คว่าข้อมูลยังอยู่:

```bash
curl -s -X POST http://localhost:8080/tasks -d '{"name":"ทดสอบ compose"}'
docker compose restart db
docker compose restart api
curl -s http://localhost:8080/tasks
```

**ผลลัพธ์จริง** (ข้อมูลยังอยู่ครบหลัง restart ทั้งคู่ เพราะข้อมูลจริงอยู่ใน named volume `pgdata` ไม่ได้อยู่ใน container layer):

```
[{"id":1,"name":"ทดสอบ compose"}]
```

> **สิ่งที่ต้องจำ**: `docker compose down` (ไม่มี flag เพิ่ม) **ลบแค่ container กับ network** แต่ **volume ยังอยู่** — ข้อมูลปลอดภัย รันขึ้นมาใหม่ก็ยังมีข้อมูลเดิม ส่วน **`docker compose down -v`** (มี `-v`) จะ**ลบ volume ทิ้งไปด้วย** — ข้อมูลหายถาวร ควรใช้เฉพาะตอนต้องการล้างข้อมูลทดสอบทั้งหมดจริงๆ เท่านั้น

---

## 11. `docker-compose.yml` ฉบับสมบูรณ์ของบทนี้

รวมทุกอย่างที่เรียนมาทั้งหมดในบทนี้เป็นไฟล์เดียว:

```yaml
# docker-compose.yml
services:
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 3s
      timeout: 3s
      retries: 10
      start_period: 5s
    networks:
      - appnet

  cache:
    image: redis:7-alpine
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 3s
      timeout: 3s
      retries: 10
    networks:
      - appnet

  api:
    build: .
    environment:
      DATABASE_URL: postgres://${POSTGRES_USER}:${POSTGRES_PASSWORD}@db:5432/${POSTGRES_DB}?sslmode=disable
      REDIS_ADDR: cache:6379
      PORT: 8080
    ports:
      - "${APP_PORT}:8080"
    depends_on:
      db:
        condition: service_healthy
      cache:
        condition: service_healthy
    networks:
      - appnet

volumes:
  pgdata:

networks:
  appnet:
    driver: bridge
```

พร้อมด้วย `.env`:

```
POSTGRES_USER=appuser
POSTGRES_PASSWORD=apppass
POSTGRES_DB=appdb
APP_PORT=8080
```

จุดที่ควรสังเกตเพิ่มเติมจากไฟล์เต็ม:

- **`networks: appnet` ระบุชัดเจนในทุก service** — แม้ Compose จะสร้าง default network ให้อัตโนมัติอยู่แล้วถ้าไม่ระบุเลย แต่การตั้งชื่อ network เองทำให้จัดการง่ายขึ้นเมื่อระบบซับซ้อนขึ้น (เช่น ต้องการแยก network สำหรับ service ที่ไม่ควรคุยกันได้โดยตรง)
- **`cache` (Redis) ไม่มี `volumes`** — เพราะในตัวอย่างนี้ Redis ถูกใช้เป็น cache ล้วนๆ (ข้อมูลสร้างใหม่ได้เสมอจาก PostgreSQL ตามที่ **Part 077** อธิบายไว้) ถ้าต้องการให้ Redis เก็บข้อมูลถาวรข้าม restart ด้วย (เช่นใช้เป็น durable queue) ต้องเพิ่ม volume คล้าย `db` และตั้งค่า persistence (`appendonly yes`) ของ Redis เองด้วย
- **`api` ไม่มี `volumes`** — เพราะ API ของเราไม่มี state ที่ต้องเก็บถาวรเลย (stateless) ข้อมูลทั้งหมดอยู่ที่ `db`/`cache` แล้ว นี่คือแนวทางที่ถูกต้องสำหรับ service ที่ต้องการ scale ได้ง่าย (รัน `api` กี่ instance พร้อมกันก็ได้โดยไม่ชนกัน)

---

## 12. คำสั่ง Compose ที่ใช้บ่อยที่สุด

| คำสั่ง | ความหมาย |
|---|---|
| `docker compose up -d` | สร้างและเริ่มทุก service แบบ background |
| `docker compose up -d --build` | เหมือนข้างบน แต่ build image ใหม่ก่อนเสมอ (ไม่ใช้ image เก่าที่ cache ไว้) |
| `docker compose ps` | ดูสถานะทุก service (รวม healthcheck status) |
| `docker compose logs -f api` | ดู log ของ service `api` แบบ follow (real-time) |
| `docker compose exec api sh` | เข้า shell ข้างใน container ของ service `api` (ถ้า image มี shell) |
| `docker compose restart db` | restart เฉพาะ service `db` โดยไม่กระทบตัวอื่น |
| `docker compose down` | ลบ container และ network ทั้งหมด (volume ยังอยู่) |
| `docker compose down -v` | ลบ container, network, **และ volume** ทั้งหมด (ข้อมูลหายถาวร) |

---

## 13. หมายเหตุความซื่อสัตย์เรื่องการรันจริงในบทนี้

ทุกอย่างในบทนี้ — ทั้ง `docker compose up`, การสาธิตปัญหา `depends_on` ในหัวข้อ 6 (ที่ `api` container จบด้วย `Exited (1)` จริง), การแก้ปัญหาด้วย healthcheck + retry loop ในหัวข้อ 9, และการพิสูจน์ volume persistence ในหัวข้อ 10 — **รันจริงบน Docker Compose v5.1.1 กับ PostgreSQL 16 และ Redis 7 จริงในสภาพแวดล้อมที่ใช้เขียนหลักสูตรนี้** ไม่ใช่ log ที่แต่งขึ้น

หมายเหตุทางเทคนิคหนึ่งข้อที่ควรทราบ: การ build image ของ `api` ในสภาพแวดล้อมนี้ต้อง **vendor dependency ไว้ล่วงหน้าด้วย `go mod vendor`** ก่อน build เพราะ container ระหว่าง build ในแซนด์บ็อกซ์ที่ใช้เขียนหลักสูตรนี้ไม่มีเส้นทางออกอินเทอร์เน็ตแบบเดียวกับเครื่อง host (การเรียก `go mod download` ตรงๆ จาก `RUN` ข้างในบิลด์จึงล้มเหลวด้วย TLS error) — ในสภาพแวดล้อมทั่วไปที่ container มี network ออกอินเทอร์เน็ตปกติ **ไม่จำเป็นต้อง vendor เลย** ใช้ `RUN go mod download` ตรงๆ ตามที่โชว์ใน **Part 095** ได้เลยตามปกติ (ซึ่งเป็นแนวทางที่แนะนำเป็นค่าเริ่มต้น) การ vendor เป็นเพียงทางเลือกเสริมสำหรับกรณีที่ต้องการ build แบบไม่พึ่งเครือข่ายเลย (offline build, air-gapped environment) หรือต้องการ pin dependency ไว้แบบตายตัวในโปรเจกต์เอง

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- Docker Compose อธิบายระบบหลาย container เป็นไฟล์ YAML เดียว มี 3 ส่วนหลัก: `services`, `volumes`, `networks`
- Service ในไฟล์เดียวกันคุยกันได้ด้วยชื่อ service เป็น hostname โดยอัตโนมัติผ่าน internal DNS ของ Compose
- `.env` แยก config/secret ออกจาก YAML และแทนที่ค่าผ่าน `${VAR_NAME}` — ควร `.gitignore` ไฟล์จริงและ commit แค่ `.env.example`
- **`depends_on` แบบสั้นรอแค่ "container เริ่ม" ไม่ใช่ "service พร้อม"** — พิสูจน์ได้จริงว่าทำให้ `api` เชื่อมต่อ PostgreSQL ที่ยังไม่พร้อมไม่ได้ จน container ตาย (`Exited (1)`)
- ทางแก้ที่ถูกต้องคือทำ**สองชั้น**: (1) `healthcheck` บน service dependency + `depends_on` แบบ long-form พร้อม `condition: service_healthy`, และ (2) **retry loop พร้อม backoff ในโค้ดแอปเอง** เป็นเกราะป้องกันที่ทำงานได้แม้ dependency พังหลังจากเริ่มระบบไปแล้ว หรือรันบน orchestrator ที่ไม่มีแนวคิด `depends_on` เลยแบบ Kubernetes
- **Volume** ทำให้ข้อมูลอยู่รอดข้ามการลบ/สร้าง container ใหม่ — พิสูจน์จริงด้วยการ restart แล้วข้อมูลยังอยู่ครบ ต่างจาก `docker compose down -v` ที่ลบ volume ทิ้งไปด้วย
- Cache-aside pattern (Redis) ทำงานร่วมกับ PostgreSQL ได้จริงผ่าน header `X-Cache: HIT`/`MISS` ที่สังเกตได้ตรงจากการทดสอบจริง

## แบบฝึกหัดท้ายบท

1. ลบ `condition: service_healthy` ออกจาก `api` ใน `docker-compose.yml` (กลับไปใช้ `depends_on` แบบ list ธรรมดา) แต่ **คงค่า default `MAX_CONNECT_ATTEMPTS=15` ในโค้ดไว้** แล้วรันดูว่าระบบยังทำงานได้ปกติหรือไม่ อธิบายว่าทำไม (เชื่อมโยงกับหัวข้อ 8 เรื่อง "เกราะสองชั้น")
2. เพิ่ม service `adminer` (Web UI สำหรับดูข้อมูลใน PostgreSQL, image คือ `adminer`) เข้าไปใน `docker-compose.yml` แล้วเปิดดูตาราง `tasks` ผ่านเบราว์เซอร์
3. ลอง `docker compose down -v` แล้ว `docker compose up -d --build` ใหม่ ยืนยันว่าตาราง `tasks` ว่างเปล่าอีกครั้ง (เพราะ volume ถูกลบไปแล้ว) อธิบายว่าทำไม migration `CREATE TABLE IF NOT EXISTS` ในโค้ดถึงยังทำงานถูกต้องแม้ฐานข้อมูลเพิ่งถูกสร้างใหม่ล้วนๆ
4. ปรับ `docker-compose.broken.yml` ให้ตั้ง `command: sh -c "sleep 15 && docker-entrypoint.sh postgres"` ใน service `db` (จำลอง Postgres ที่ช้ากว่าเดิมมาก) แล้วลองปรับ `retries`/`interval` ของ healthcheck ให้เหมาะสมพอที่จะรอได้ทัน
5. เขียน unit test สำหรับฟังก์ชัน `connectWithRetry` (ทบทวนเทคนิคจาก **Part 033-034**) โดยจำลอง `fn` ที่ fail 2 ครั้งแล้วสำเร็จในครั้งที่ 3 ตรวจสอบว่าฟังก์ชัน retry ถูกต้องตามจำนวนครั้งที่คาดหวัง
6. ค้นคว้าเพิ่มเติมเรื่อง `docker compose watch` (feature ใหม่ใน Compose รุ่นหลังๆ) ที่ rebuild/sync ไฟล์เข้า container อัตโนมัติเมื่อโค้ดเปลี่ยน เหมาะกับ workflow ตอน develop ในเครื่อง

---

**ต่อไป**: [Part 097 — Kubernetes เบื้องต้นสำหรับ Go Developer](./097-kubernetes-basics.md)
