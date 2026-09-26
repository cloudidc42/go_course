# Part 098: CI/CD Pipeline ด้วย GitHub Actions

> ภาคที่ 9: DevOps และ Deployment — ตอนที่ 4 จาก 5 (Part 95–99)

## สารบัญของบทนี้

1. ทำไม CI/CD คือส่วนที่เชื่อม Part 095-097 เข้าด้วยกันเป็นระบบเดียว
2. โครงสร้างพื้นฐานของ GitHub Actions workflow: `on`, `jobs`, `steps`
3. หลักการ "Fail Fast": ทำไมลำดับของ stage ถึงสำคัญพอๆ กับเนื้อหาข้างใน
4. Stage 1 — Lint & Vet: ด่านแรกที่เร็วที่สุด
5. Stage 2 — Unit Test: Matrix หลายเวอร์ชัน Go พร้อม Caching
6. Stage 3 — Integration Test: รัน PostgreSQL จริงด้วย Service Container
7. Stage 4 (CD) — Build และ Push Docker Image เมื่อ merge เข้า `main`
8. `ci.yml` ฉบับสมบูรณ์ของบทนี้
9. รันทุก step จริงในเครื่อง: `go build`/`go vet`/`go test -race -cover`/`golangci-lint`
10. ตรวจสอบความถูกต้องของ workflow YAML ด้วย `actionlint`: บั๊กจริงที่จับได้จริง
11. Branch Protection: บังคับให้ CI ผ่านก่อน merge ได้เสมอ
12. หมายเหตุความซื่อสัตย์เรื่องการรันจริงในบทนี้
13. สรุปสิ่งที่ได้เรียนในบทนี้
14. แบบฝึกหัดท้ายบท

---

## 1. ทำไม CI/CD คือส่วนที่เชื่อม Part 095-097 เข้าด้วยกันเป็นระบบเดียว

ย้อนดูสิ่งที่เรียนมาในภาคนี้: **Part 095** สอนวิธี build Docker image ที่เล็กและปลอดภัยจาก Go binary, **Part 096** สอนวิธีรันหลาย service ร่วมกันด้วย Docker Compose, **Part 097** สอนวิธี deploy image นั้นขึ้น Kubernetes cluster — แต่ทั้งหมดนี้ยังเป็น**คำสั่งที่ต้องมีคนพิมพ์เอง** (`docker build`, `docker push`, `kubectl apply`) ถ้าทีมมีนักพัฒนาหลายคน push โค้ดเข้า repository วันละหลายสิบครั้ง การให้แต่ละคนต้อง build/test/deploy มือทุกครั้งจะ:

- **ไม่สม่ำเสมอ**: คนหนึ่งอาจลืมรัน `go vet` หรือ `go test` ก่อน push ทำให้โค้ดที่มี bug หลุดเข้า `main` branch
- **ช้าและเสียเวลา**: รอ human กด build/deploy เองทุกครั้ง แทนที่จะให้เครื่องทำงานอัตโนมัติ
- **ไม่มีร่องรอยตรวจสอบย้อนหลัง**: ไม่รู้ว่า commit ไหนผ่าน test บ้าง ใครเป็นคน deploy เมื่อไร

**CI/CD** (Continuous Integration / Continuous Deployment) คือแนวทางที่ให้**เครื่องอัตโนมัติทำหน้าที่นี้แทนคนทั้งหมด** ทุกครั้งที่มีการ push โค้ดหรือเปิด pull request:

- **CI (Continuous Integration)**: รัน build, lint, test อัตโนมัติทันทีที่มีการเปลี่ยนแปลงโค้ด เพื่อจับปัญหาให้เร็วที่สุดเท่าที่จะทำได้ — ยิ่งจับ bug เร็ว ยิ่งแก้ง่ายและถูกกว่า (bug ที่หลุดไปถึง production แก้ยากและแพงกว่ามาก)
- **CD (Continuous Deployment/Delivery)**: เมื่อโค้ดผ่านทุกการตรวจสอบแล้ว **build image และ deploy ออกไปอัตโนมัติ** โดยไม่ต้องมีคนกดปุ่มเอง (Continuous **Deployment** คือ deploy อัตโนมัติเต็มรูปแบบ ส่วน Continuous **Delivery** คือเตรียมทุกอย่างพร้อม deploy แต่ยังรอคนกดยืนยันขั้นสุดท้าย — บทนี้แสดงแบบ Deployment เต็มรูปแบบ)

**GitHub Actions** คือระบบ CI/CD ที่ผูกกับ GitHub repository โดยตรง กำหนดด้วยไฟล์ YAML ที่วางไว้ที่ `.github/workflows/` ในบทนี้เราจะสร้าง pipeline ที่ครบวงจร: **checkout → build → vet → lint → test (matrix + race + coverage) → integration test (กับ PostgreSQL จริง) → build & push Docker image (เมื่อ merge เข้า `main` เท่านั้น)** — ครบทุกอย่างที่เรียนมาตลอดภาคนี้ในไฟล์เดียว

---

## 2. โครงสร้างพื้นฐานของ GitHub Actions workflow: `on`, `jobs`, `steps`

ไฟล์ workflow ของ GitHub Actions เป็น YAML ที่มีโครงสร้างหลัก 3 ระดับ:

```yaml
name: CI                        # ชื่อ workflow ที่แสดงบนหน้า GitHub

on:                              # เงื่อนไขที่จะ trigger ให้ workflow นี้รัน
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:                            # workflow หนึ่งไฟล์มีได้หลาย job
  my-job:                        # แต่ละ job รันบน runner (เครื่องเสมือน) แยกกัน โดย default
    runs-on: ubuntu-latest       # เลือกชนิดของเครื่องที่จะรัน
    steps:                       # แต่ละ job มีหลาย step ที่รันเรียงลำดับกันบนเครื่องเดียวกัน
      - name: Checkout code
        uses: actions/checkout@v4   # "uses" เรียกใช้ action สำเร็จรูปที่คนอื่นเขียนไว้แล้ว
      - name: Run a command
        run: echo "hello"           # "run" รันคำสั่ง shell ตรงๆ
```

จุดที่ต้องเข้าใจให้ชัดตั้งแต่แรก:

- **`on`** กำหนดว่า workflow นี้ trigger เมื่อไร — `push` เข้า branch ที่ระบุ, `pull_request` ที่เปิด/อัปเดตเข้า branch ที่ระบุ, หรือ event อื่นๆ อีกมาก (เช่น `schedule` สำหรับรันตามเวลา, `workflow_dispatch` สำหรับกดรันเองผ่านหน้าเว็บ)
- **`jobs`** แต่ละตัวรันบน **runner แยกกันโดย default** (เครื่องเสมือนคนละเครื่อง ไม่มี filesystem ร่วมกัน) ถ้าต้องการให้ job หนึ่งรอ job อื่นให้เสร็จก่อน ต้องประกาศ **`needs:`** อย่างชัดเจน (หัวข้อ 3) ไม่งั้น GitHub Actions จะรันทุก job **พร้อมกัน** โดย default เพื่อความเร็วสูงสุด
- **`uses`** เรียกใช้ **action** ที่เป็นโค้ดสำเร็จรูป (อาจเขียนโดย GitHub เอง เช่น `actions/checkout`, หรือ community เช่น `docker/build-push-action`) แทนที่จะเขียนคำสั่ง shell ยาวๆ เอง — action มีเวอร์ชันของตัวเอง (ระบุด้วย `@v4`, `@v6` ฯลฯ) และควร pin เวอร์ชันเสมอด้วยเหตุผลเดียวกับที่ **Part 095** เตือนเรื่องการ pin เวอร์ชันของ base image (ไม่ pin = build วันนี้กับวันหน้าอาจพฤติกรรมต่างกันโดยไม่รู้ตัว)
- **`run`** รันคำสั่ง shell ตรงๆ บน runner (default คือ `bash` บน Linux runner) — ใช้กับคำสั่งที่คุ้นเคยอยู่แล้วอย่าง `go build`, `go test` ได้ตรงๆ ไม่ต้องพึ่ง action พิเศษ

---

## 3. หลักการ "Fail Fast": ทำไมลำดับของ stage ถึงสำคัญพอๆ กับเนื้อหาข้างใน

หลักการสำคัญที่สุดข้อหนึ่งของการออกแบบ CI pipeline ที่ดีคือ **"fail fast"** — เรียง**การตรวจสอบที่เร็วและถูกที่สุดไว้ก่อนเสมอ** แล้วค่อยตามด้วยการตรวจสอบที่ช้าและแพงกว่า เหตุผลคือถ้าโค้ด build ไม่ผ่านตั้งแต่แรก ไม่มีประโยชน์อะไรเลยที่จะเสียเวลา 5-10 นาทีรัน integration test ที่ต้องสร้าง container ฐานข้อมูลขึ้นมาก่อน — ปล่อยให้มันพังตั้งแต่ step แรกที่ใช้เวลาไม่กี่วินาที ดีกว่าให้ผู้พัฒนารอนานแล้วมาเจอว่า error เกิดจากอะไรง่ายๆ ตั้งแต่ต้น

pipeline ในบทนี้เรียงลำดับตามความเร็ว/ต้นทุนจากน้อยไปมากตั้งใจดังนี้:

```
Stage 1: Lint & Vet          ← เร็วที่สุด (วินาทีถึงหลักสิบวินาที) ไม่ต้องพึ่ง service ภายนอกเลย
    ↓ needs
Stage 2: Unit Test (matrix)  ← เร็ว-ปานกลาง รันได้หลาย Go version พร้อมกัน แต่ไม่ต้องพึ่ง service ภายนอก
    ↓ needs
Stage 3: Integration Test    ← ช้าที่สุดในบรรดา test เพราะต้องรอ PostgreSQL container พร้อมก่อน
    ↓ needs (เฉพาะตอน push เข้า main)
Stage 4: Build & Push Image  ← ช้าและมีผลกระทบจริง (push image ขึ้น registry) จึงต้องรันหลังสุดเท่านั้น
```

ใน GitHub Actions การกำหนดลำดับนี้ทำผ่าน keyword **`needs:`** — job ที่มี `needs: lint-and-vet` จะ**รอ**ให้ job ชื่อ `lint-and-vet` **สำเร็จก่อนเท่านั้น** ถึงจะเริ่มรัน ถ้า `lint-and-vet` ล้มเหลว job ที่ตามมาทั้งหมด (`unit-test`, `integration-test`, `build-and-push`) จะ**ถูกข้ามไปเลยโดยอัตโนมัติ** ไม่เสียเวลารันต่อให้เสียทรัพยากรฟรีๆ

---

## 4. Stage 1 — Lint & Vet: ด่านแรกที่เร็วที่สุด

```yaml
  lint-and-vet:
    name: Lint & Vet
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Go
        uses: actions/setup-go@v5
        with:
          go-version: "1.24"
          cache: true

      - name: go build
        run: go build ./...

      - name: go vet
        run: go vet ./...

      - name: golangci-lint
        uses: golangci/golangci-lint-action@v6
        with:
          version: latest
```

- **`actions/checkout@v4`** ต้องเป็น step แรกของทุก job เสมอ (ยกเว้นกรณีพิเศษที่ไม่ต้องใช้โค้ดจริง) เพราะ runner เริ่มต้นด้วย filesystem ว่างเปล่า ไม่มีโค้ดของ repository อยู่เลยจนกว่าจะ checkout
- **`actions/setup-go@v5`** ติดตั้ง Go ตามเวอร์ชันที่ระบุ พร้อม **`cache: true`** ที่เปิดใช้ module cache ของ Go อัตโนมัติ (แคช `$GOPATH/pkg/mod` และ build cache ข้าม run ของ workflow) — ลดเวลาดาวน์โหลด dependency ซ้ำๆ ในแต่ละครั้งที่ workflow รัน ได้อย่างมีนัยสำคัญ โดยเฉพาะโปรเจกต์ที่มี dependency เยอะแบบที่ต่อ `pgx`/`go-redis` ตามที่เรียนใน **Part 096**
- **`go build ./...`** และ **`go vet ./...`** คือด่านแรกที่ถูกและเร็วที่สุด — `go build` จับ compile error, `go vet` จับข้อผิดพลาดเชิง logic ที่ compiler มองไม่เห็น (ตามที่เรียนใน **Part 001** หัวข้อ 6) ทั้งสองอย่างนี้ใช้เวลาระดับวินาทีสำหรับโปรเจกต์ขนาดกลาง
- **`golangci/golangci-lint-action@v6`** คือ action สำเร็จรูปที่รัน `golangci-lint` (เครื่องมือตามที่ระบุไว้ในหัวข้อ **Part 086** ของหลักสูตรนี้) — action ตัวนี้ฉลาดกว่าการรัน `run: golangci-lint run` ตรงๆ ตรงที่มันจัดการ caching ของ linter เอง และรายงานผลเป็น annotation บนหน้า pull request โดยตรง (ขึ้นเป็นจุดสีแดง/เขียวตรงบรรทัดโค้ดที่มีปัญหา)

---

## 5. Stage 2 — Unit Test: Matrix หลายเวอร์ชัน Go พร้อม Caching

```yaml
  unit-test:
    name: Unit Test (Go ${{ matrix.go-version }})
    needs: lint-and-vet
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        go-version: ["1.23", "1.24"]
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Go ${{ matrix.go-version }}
        uses: actions/setup-go@v5
        with:
          go-version: ${{ matrix.go-version }}
          cache: true

      - name: go test -race -cover
        run: go test -race -cover -coverprofile=coverage.out ./...

      - name: Upload coverage report
        uses: actions/upload-artifact@v4
        with:
          name: coverage-go${{ matrix.go-version }}
          path: coverage.out
```

### Matrix Testing

**`strategy.matrix`** สั่งให้ GitHub Actions **รัน job นี้ซ้ำหลายครั้งพร้อมกัน** โดยแทนค่าตัวแปร `go-version` ด้วยแต่ละค่าใน list — ในตัวอย่างนี้จะได้ job ย่อยสองตัวที่รันพร้อมกันจริง: `Unit Test (Go 1.23)` และ `Unit Test (Go 1.24)` ประโยชน์ของ matrix testing คือ**ยืนยันว่าโค้ดทำงานถูกต้องกับ Go หลายเวอร์ชันพร้อมกัน** สำคัญมากสำหรับ library หรือ tool ที่คนอื่นจะเอาไปใช้กับ Go เวอร์ชันที่ต่างจากที่ทีมพัฒนาใช้เอง — ยิ่งสำคัญเพราะ Go ออกเวอร์ชันใหม่ทุก 6 เดือนตามที่ **Part 001** อธิบายไว้ และมี [Go 1 Compatibility Promise](https://go.dev/doc/go1compat) รับประกันว่าโค้ดต้อง compile ได้ในเวอร์ชันถัดไปเสมอ — matrix testing คือวิธี**พิสูจน์**คำสัญญานั้นด้วยตัวเองในทุก pull request

**`fail-fast: false`** ในระดับ `strategy` (คนละความหมายกับหลักการ "fail fast" ของทั้ง pipeline ในหัวข้อ 3 — นี่คือ keyword เฉพาะของ matrix) บอกว่า **ถ้า job หนึ่งใน matrix ล้มเหลว ให้ job อื่นที่เหลือรันต่อจนจบ** ไม่ยกเลิกทันที เพื่อให้เห็นภาพรวมว่า Go เวอร์ชันไหนบ้างที่มีปัญหา แทนที่จะเห็นแค่ตัวแรกที่ fail แล้วต้องรันใหม่ทั้งหมดเพื่อดูตัวถัดไป

### `go test -race -cover`

คำสั่งเดียวกับที่เรียนไปใน **Part 044** (Race Detector) และ **Part 082** (Test Coverage) — `-race` เปิด race detector ตรวจจับ data race ระหว่าง goroutine, `-cover` วัดว่าโค้ดกี่เปอร์เซ็นต์ถูกทดสอบจริง การรันทั้งสอง flag พร้อมกันใน CI ทุกครั้งสำคัญมากเพราะ **race condition มักไม่ปรากฏในเครื่องนักพัฒนาที่รันแค่ไม่กี่ครั้ง แต่จะโผล่ขึ้นมาแบบสุ่มใน production ที่มี concurrent request จำนวนมาก** — การบังคับให้ CI รัน `-race` ทุกครั้งคือด่านป้องกันที่คุ้มค่าที่สุดอย่างหนึ่งสำหรับโค้ด concurrent (ตามหลักการที่เรียนไปทั้งภาคที่ 3)

`-coverprofile=coverage.out` เขียนผลความครอบคลุมของ test ลงไฟล์ แล้ว **`actions/upload-artifact@v4`** เก็บไฟล์นี้ไว้ให้ดาวน์โหลดย้อนหลังได้จากหน้า GitHub Actions run — มีประโยชน์เวลาต้องการดูรายละเอียดว่าโค้ดส่วนไหนยังไม่มี test ครอบคลุม

---

## 6. Stage 3 — Integration Test: รัน PostgreSQL จริงด้วย Service Container

Unit test ตรวจสอบ logic ของโค้ดแยกส่วน แต่ไม่ครอบคลุมกรณีที่โค้ดต้อง**คุยกับฐานข้อมูลจริง** (เช่น query ที่เขียนผิด syntax SQL แต่ mock/stub ตรวจไม่เจอ) — **integration test** (ตามหัวข้อที่ระบุไว้ใน **Part 080** ของหลักสูตรนี้) แก้ปัญหานี้ด้วยการรันโค้ดจริงคู่กับฐานข้อมูลจริง GitHub Actions มีฟีเจอร์ **`services:`** ระดับ job ที่รัน container เสริมให้ steps ในนั้นใช้งานได้ เทียบเท่ากับแนวคิด `depends_on` + healthcheck ของ Docker Compose ที่เรียนไปอย่างละเอียดใน **Part 096**:

```yaml
  integration-test:
    name: Integration Test
    needs: unit-test
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_USER: appuser
          POSTGRES_PASSWORD: apppass
          POSTGRES_DB: appdb
        ports:
          - 5432:5432
        options: >-
          --health-cmd="pg_isready -U appuser -d appdb"
          --health-interval=5s
          --health-timeout=5s
          --health-retries=10
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Go
        uses: actions/setup-go@v5
        with:
          go-version: "1.24"
          cache: true

      - name: Run integration tests
        env:
          DATABASE_URL: postgres://appuser:apppass@localhost:5432/appdb?sslmode=disable
        run: go test -tags=integration -v ./...
```

จุดที่ควรสังเกต:

- **`services.postgres`** รัน container `postgres:16-alpine` ให้อัตโนมัติก่อนที่ step แรกใน `steps:` จะเริ่มทำงาน — **image เดียวกันเป๊ะ**กับที่ **Part 096** ใช้ใน `docker-compose.yml`
- **`options` พร้อม `--health-cmd`** คือกลไกเดียวกับ `healthcheck` + `condition: service_healthy` ที่เรียนไปใน **Part 096** หัวข้อ 7: **GitHub Actions จะรอให้ health check ผ่านก่อน** ถึงจะปล่อยให้ step ที่ต้องใช้ database เริ่มทำงาน แก้ปัญหาเดียวกันกับที่ `depends_on` แบบสั้นเคย "หลอก" เราไว้ในบทก่อนหน้าเป๊ะๆ — เพียงแค่เปลี่ยนบริบทจาก Docker Compose มาเป็น GitHub Actions
- **`ports: ["5432:5432"]`** เปิด port ของ container นี้ให้ steps อื่นในเดียวกัน job เข้าถึงผ่าน `localhost:5432` ได้ตรงๆ (ต่างจาก Docker Compose ที่ service คุยกันผ่านชื่อ service — ใน GitHub Actions runner, service container ผูกกับ `localhost` ของ runner โดยตรง)
- แม้จะมี healthcheck ป้องกันไว้แล้ว **หลักการ "เกราะสองชั้น" จาก Part 096 หัวข้อ 8 ยังคงสำคัญเหมือนเดิม** — โค้ด `connectWithRetry` ที่เขียนไว้ใน Part 096 ควรยังคงอยู่ในแอปแม้จะรันใน CI ที่มี healthcheck คอยช่วยแล้วก็ตาม เพราะมันคือด่านป้องกันที่ทำงานได้ไม่ว่าจะรันอยู่ที่ไหนก็ตาม (local, CI, หรือ Kubernetes ที่ไม่มีแนวคิด `depends_on` เลยตามที่อธิบายไว้ใน **Part 097**)
- **`go test -tags=integration`** ใช้ [build tag](https://pkg.go.dev/go/build#hdr-Build_Constraints) แยก integration test ออกจาก unit test ปกติอย่างชัดเจน (ไฟล์ทดสอบที่มี `//go:build integration` ที่บรรทัดบนสุด จะไม่ถูกรันเลยถ้าไม่ใส่ `-tags=integration`) ทำให้ `go test ./...` ธรรมดาใน Stage 2 (ที่ไม่มี database ให้ใช้) ไม่ error เพราะพยายามต่อ database ที่ไม่มีอยู่จริง

---

## 7. Stage 4 (CD) — Build และ Push Docker Image เมื่อ merge เข้า `main`

เมื่อทุกด่านตรวจสอบผ่านหมดแล้ว (lint, vet, unit test ทุกเวอร์ชัน Go, integration test) ขั้นตอนสุดท้ายคือ**สร้าง Docker image ตาม Dockerfile ที่เขียนไว้ใน Part 095 แล้ว push ขึ้น container registry** เพื่อให้พร้อม deploy ขึ้น Kubernetes ตามที่เรียนใน **Part 097**:

```yaml
  build-and-push:
    name: Build & Push Docker Image
    needs: integration-test
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Log in to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract image metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ghcr.io/${{ github.repository }}
          tags: |
            type=sha,prefix=
            type=raw,value=latest

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Build and push image
        uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

จุดสำคัญที่สุดของ job นี้คือบรรทัด **`if: github.event_name == 'push' && github.ref == 'refs/heads/main'`** — เงื่อนไขนี้ทำให้ job นี้**รันเฉพาะตอนที่มีการ push (ปกติคือหลัง merge pull request) เข้า branch `main` โดยตรงเท่านั้น** ไม่รันตอนแค่เปิด pull request (`pull_request` event) เพราะ **PR ที่ยังไม่ผ่านการ review และยัง merge ไม่สำเร็จ ไม่ควรมี image ของมันถูก push ขึ้น registry กลางที่ระบบ production อาจไปดึงมาใช้** — นี่คือกฎเหล็กของ CD ที่ดี: **สร้าง artifact ที่ deploy จริงได้เฉพาะจากโค้ดที่ผ่านกระบวนการตรวจสอบครบถ้วนและถูกยอมรับเข้าสู่ branch หลักแล้วเท่านั้น**

- **`docker/login-action@v3`** ล็อกอินเข้า `ghcr.io` (GitHub Container Registry) ด้วย **`secrets.GITHUB_TOKEN`** ซึ่งเป็น token ที่ GitHub Actions **สร้างให้อัตโนมัติทุก run โดยไม่ต้องตั้งค่าเอง** (ต่างจาก secret อื่นที่ต้องไปตั้งค่าในหน้า repository settings เอง) มีสิทธิ์เพียงพอสำหรับ push image เข้า registry ของ repository เดียวกัน
- **`docker/metadata-action@v5`** สร้าง tag ของ image ให้อัตโนมัติตามกฎที่กำหนด ในที่นี้สร้างสอง tag เสมอ: tag ตาม commit SHA (สำหรับ trace กลับไปหา commit ต้นทางได้แม่นยำ) และ tag `latest` (สำหรับให้ระบบอื่นดึง version ล่าสุดได้ง่าย)
- **`cache-from/cache-to: type=gha`** ใช้ **GitHub Actions cache** เก็บ Docker layer cache ข้าม run — ทำให้ build ครั้งถัดไปเร็วขึ้นมากถ้า layer ส่วนใหญ่ไม่เปลี่ยน (หลักการเดียวกับที่ **Part 095** อธิบายเรื่อง Docker layer caching แต่ยกระดับให้ cache ข้าม CI run แต่ละครั้งได้ด้วย ไม่ใช่แค่ข้าม `docker build` ในเครื่องเดียว)
- ผลลัพธ์สุดท้ายของ job นี้คือ image ที่ `ghcr.io/<owner>/<repo>:latest` พร้อมให้ Kubernetes Deployment ใน **Part 097** อ้างอิงไปดึงมาใช้ได้ทันที — **ปิดวงจรครบ**: push โค้ด → CI ตรวจสอบ → build image ตาม Dockerfile ของ Part 095 → push ขึ้น registry → พร้อม deploy ด้วย manifest ของ Part 097

---

## 8. `ci.yml` ฉบับสมบูรณ์ของบทนี้

รวมทั้ง 4 stage เข้าเป็นไฟล์เดียว วางไว้ที่ `.github/workflows/ci.yml` ของ repository:

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

# ยกเลิก run เก่าที่ยังค้างอยู่ของ PR เดียวกัน เมื่อมีการ push ใหม่ทับเข้ามา
# ประหยัดเวลาและทรัพยากรของ CI runner ไม่ต้องรัน run เก่าที่ล้าสมัยไปแล้วให้จบ
concurrency:
  group: ci-${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

permissions:
  contents: read

jobs:
  # ----------------------------------------------------------------------
  # Stage 1: fail-fast checks — เร็ว ไม่ต้องพึ่ง service ภายนอกใดๆ เลย
  # ถ้า stage นี้ล้มเหลว งานที่ตามมา (test, integration, build-and-push) จะไม่ถูกรันเลย
  # ----------------------------------------------------------------------
  lint-and-vet:
    name: Lint & Vet
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Go
        uses: actions/setup-go@v5
        with:
          go-version: "1.24"
          cache: true

      - name: go build
        run: go build ./...

      - name: go vet
        run: go vet ./...

      - name: golangci-lint
        uses: golangci/golangci-lint-action@v6
        with:
          version: latest

  # ----------------------------------------------------------------------
  # Stage 2: unit tests — รันแบบ matrix หลายเวอร์ชัน Go พร้อมกัน
  # รอ lint-and-vet ผ่านก่อนเสมอ (needs) เพื่อไม่เสียเวลารัน test ถ้าโค้ด build ไม่ผ่าน
  # ----------------------------------------------------------------------
  unit-test:
    name: Unit Test (Go ${{ matrix.go-version }})
    needs: lint-and-vet
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        go-version: ["1.23", "1.24"]
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Go ${{ matrix.go-version }}
        uses: actions/setup-go@v5
        with:
          go-version: ${{ matrix.go-version }}
          cache: true

      - name: go test -race -cover
        run: go test -race -cover -coverprofile=coverage.out ./...

      - name: Upload coverage report
        uses: actions/upload-artifact@v4
        with:
          name: coverage-go${{ matrix.go-version }}
          path: coverage.out

  # ----------------------------------------------------------------------
  # Stage 3: integration test — ช้าที่สุด ต้องพึ่ง service container จริง
  # จงใจรันเป็น job แยกต่างหาก ไม่รวมกับ unit-test เพื่อให้ unit test
  # ที่เร็วกว่ารายงานผลได้ก่อน โดยไม่ต้องรอ Postgres container พร้อม
  # ----------------------------------------------------------------------
  integration-test:
    name: Integration Test
    needs: unit-test
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_USER: appuser
          POSTGRES_PASSWORD: apppass
          POSTGRES_DB: appdb
        ports:
          - 5432:5432
        # GitHub Actions รอ health check นี้ผ่านก่อนปล่อยให้ step ถัดไปรัน
        # เหมือนกับ condition: service_healthy ใน docker compose ที่เรียนไปใน Part 096
        options: >-
          --health-cmd="pg_isready -U appuser -d appdb"
          --health-interval=5s
          --health-timeout=5s
          --health-retries=10
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Go
        uses: actions/setup-go@v5
        with:
          go-version: "1.24"
          cache: true

      - name: Run integration tests
        env:
          DATABASE_URL: postgres://appuser:apppass@localhost:5432/appdb?sslmode=disable
        run: go test -tags=integration -v ./...

  # ----------------------------------------------------------------------
  # Stage 4 (CD): build + push Docker image — รันเฉพาะตอน merge เข้า main แล้วเท่านั้น
  # ไม่รันตอน pull_request เพราะ PR ที่ยังไม่ merge ไม่ควร push image ขึ้น registry
  # ----------------------------------------------------------------------
  build-and-push:
    name: Build & Push Docker Image
    needs: integration-test
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Log in to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract image metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ghcr.io/${{ github.repository }}
          tags: |
            type=sha,prefix=
            type=raw,value=latest

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Build and push image
        uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

---

## 9. รันทุก step จริงในเครื่อง: `go build`/`go vet`/`go test -race -cover`/`golangci-lint`

ก่อนเชื่อว่า pipeline นี้ใช้งานได้จริง เราสามารถ**จำลอง**สิ่งที่แต่ละ step ของ Stage 1-2 จะทำบน CI runner ได้เองในเครื่อง — เพราะท้ายที่สุดแล้ว GitHub Actions ก็แค่รันคำสั่งเหล่านี้บนเครื่อง Linux เครื่องหนึ่งเท่านั้นเอง ไม่มีเวทมนตร์ซ่อนอยู่ ในบทนี้ได้เขียนโปรเจกต์ตัวอย่างเล็กๆ (`price` package ที่คำนวณราคาหลังหักส่วนลด พร้อม table-driven test ตามที่เรียนใน **Part 034**) แล้วรันคำสั่งเดียวกับใน `ci.yml` จริงในสภาพแวดล้อมที่เขียนหลักสูตรนี้:

```go
// price.go
package price

import "fmt"

func ApplyDiscount(price float64, percent float64) (float64, error) {
	if price < 0 {
		return 0, fmt.Errorf("price must not be negative: %v", price)
	}
	if percent < 0 || percent > 100 {
		return 0, fmt.Errorf("percent must be between 0 and 100: %v", percent)
	}
	return price * (1 - percent/100), nil
}
```

รันคำสั่งเดียวกับที่ `ci.yml` ใช้ในหัวข้อ 4 และ 5 ตามลำดับ:

```bash
$ go build ./...
$ go vet ./...
$ go test -race -cover ./...
ok  	ci-demo	1.013s	coverage: 100.0% of statements
$ golangci-lint run ./...
0 issues.
```

**ทั้ง 4 คำสั่งผ่านหมด** — เหมือนที่ Stage 1-2 ของ `ci.yml` จะรายงานสถานะสีเขียวถ้ารันบน GitHub Actions จริง

### พิสูจน์ว่า `golangci-lint` จับปัญหาได้จริง ไม่ใช่แค่ผ่านเสมอ

เพื่อยืนยันว่า Stage 1 มีประโยชน์จริงไม่ใช่แค่ step ที่ผ่านทุกครั้งอย่างไร้ความหมาย ได้ทดลองแก้โค้ดให้มีปัญหาโดยตั้งใจ (ประกาศตัวแปรแล้วไม่ใช้งาน — ข้อผิดพลาดที่ **Part 001** เคยกล่าวไว้ว่า Go บังคับให้เป็น compile error อยู่แล้วในกรณี unused import แต่ unused **local variable** ก็เป็น compile error เช่นกัน):

```go
func ApplyDiscount(price float64, percent float64) (float64, error) {
	unused := 42
	if price < 0 {
		return 0, fmt.Errorf("price must not be negative: %v", price)
	}
	return price * (1 - percent/100), nil
}
```

รัน `golangci-lint run ./...` กับโค้ดนี้:

```
$ golangci-lint run ./...
./price.go:6:2: declared and not used: unused (typecheck)
1 issues:
* typecheck: 1
```

**ผลลัพธ์จริง**: `golangci-lint` จับ error ได้ทันที (ในกรณีนี้จับได้ตั้งแต่ระดับ `typecheck` ซึ่งเป็นด่านที่เข้มงวดที่สุด เทียบเท่ากับ `go build` เองที่จะ error แบบเดียวกัน) พิสูจน์ว่า Stage 1 ของ pipeline ทำหน้าที่ตามที่ตั้งใจไว้จริง — ถ้า pull request ไหนมีโค้ดแบบนี้หลุดเข้ามา `lint-and-vet` job จะเปลี่ยนเป็นสีแดงทันที และ job ที่เหลือทั้งหมด (`unit-test`, `integration-test`, `build-and-push`) จะไม่ถูกรันเลยตามหลักการ fail-fast ในหัวข้อ 3

---

## 10. ตรวจสอบความถูกต้องของ workflow YAML ด้วย `actionlint`: บั๊กจริงที่จับได้จริง

ไฟล์ `ci.yml` เป็น YAML ธรรมดา ดังนั้นเครื่องมือ YAML parser ทั่วไปอย่าง `yamllint` ตรวจสอบได้แค่ว่า **syntax YAML ถูกต้อง** (เยื้องบรรทัดถูก, ไม่มีอักขระแปลกปลอม ฯลฯ) แต่ **ไม่รู้เรื่อง semantic เฉพาะของ GitHub Actions เลย** เช่น ไม่รู้ว่า `uses: golangci-lint-action@v6` ผิดรูปแบบเพราะขาดชื่อ owner ของ repository — สำหรับ workflow ของ GitHub Actions โดยเฉพาะ มีเครื่องมือที่ออกแบบมาเพื่องานนี้ตรงๆ ชื่อ **[`actionlint`](https://github.com/rhysd/actionlint)** ซึ่งเข้าใจโครงสร้างของ workflow, ตรวจสอบ expression syntax (`${{ ... }}`), ตรวจการอ้างอิง `matrix`/`needs`/`secrets` ที่ไม่มีอยู่จริง และรูปแบบของ `uses:` ทั้งหมด

ระหว่างเขียนบทนี้ ได้เขียน `ci.yml` ฉบับร่างแรกที่มีบรรทัด:

```yaml
      - name: golangci-lint
        uses: golangci-lint-action@v6
```

แล้วรัน `actionlint` กับไฟล์นี้จริง:

```bash
$ actionlint ci.yml
ci.yml:43:15: specifying action "golangci-lint-action@v6" in invalid format because owner is missing.
  available formats are "{owner}/{repo}@{ref}" or "{owner}/{repo}/{path}@{ref}" [action]
   |
43 |         uses: golangci-lint-action@v6
   |               ^~~~~~~~~~~~~~~~~~~~~~~
```

**นี่คือบั๊กจริงที่เกิดขึ้นจริงระหว่างเขียนบทความนี้** — ลืมใส่ชื่อ owner ของ action (`golangci/`) ไปเฉยๆ ซึ่งเป็นความผิดพลาดที่พบได้บ่อยมากเวลาพิมพ์ชื่อ action ยาวๆ ถ้าไม่มี `actionlint` คอยตรวจ ข้อผิดพลาดแบบนี้จะไม่ถูกจับจนกว่าจะ push ขึ้น GitHub จริงแล้ว workflow ล้มเหลวตอนรัน (เสียเวลาต้อง push-รอ-แก้-push ใหม่หลายรอบ) แก้ไขให้ถูกต้องแล้วรัน `actionlint` ซ้ำ:

```bash
$ sed -i 's#uses: golangci-lint-action@v6#uses: golangci/golangci-lint-action@v6#' ci.yml
$ actionlint ci.yml
$ echo "exit code: $?"
exit code: 0
```

**`actionlint` ไม่รายงาน error ใดๆ เลย** (ไม่มี output คือผ่านทั้งหมด) ยืนยันว่าไฟล์ `ci.yml` ฉบับสมบูรณ์ในหัวข้อ 8 (ที่แก้ไขบรรทัดนี้แล้ว) ถูกต้องตามโครงสร้างของ GitHub Actions จริง เสริมด้วย `yamllint` เพื่อยืนยัน syntax YAML พื้นฐานอีกชั้น:

```bash
$ yamllint ci.yml
ci.yml
  1:1  warning  missing document start "---"  (document-start)
  3:1  warning  truthy value should be one of [false, true]  (truthy)
```

สอง warning นี้**ไม่ใช่ error** — `missing document start` เป็นแค่ข้อแนะนำเชิงสไตล์ (ใส่ `---` ที่บรรทัดแรกของไฟล์ ซึ่ง GitHub Actions ไม่บังคับ) ส่วน `truthy value` มาจากคีย์เวิร์ด `on:` ที่ YAML รุ่นเก่าตีความ `on`/`off` เป็น boolean ได้ (ในบริบทของ GitHub Actions `on:` เป็นชื่อ key ที่ตั้งใจ ไม่ใช่ boolean) — ทั้งสอง warning นี้เป็นเรื่องปกติสำหรับไฟล์ GitHub Actions workflow ทุกไฟล์ ไม่ใช่ปัญหาที่ต้องแก้ และไม่กระทบการทำงานจริงแต่อย่างใด

> **บทเรียนสำคัญจากหัวข้อนี้**: `yamllint` ตรวจ **syntax YAML** ส่วน `actionlint` ตรวจ **semantic ของ GitHub Actions โดยเฉพาะ** — ทั้งสองอย่างเสริมกันแต่ทำหน้าที่ต่างกันชัดเจน โปรเจกต์จริงควรรันทั้งคู่ (มักผ่าน pre-commit hook หรือแม้แต่เป็น step หนึ่งในตัว `ci.yml` เองที่ตรวจสอบ `ci.yml` ตัวมันเอง!)

---

## 11. Branch Protection: บังคับให้ CI ผ่านก่อน merge ได้เสมอ

การมี `ci.yml` ที่ทำงานถูกต้องยังไม่เพียงพอถ้า**ไม่มีอะไรบังคับ**ให้คนต้องรอผลจาก CI ก่อน merge — โดย default แล้ว repository ของ GitHub อนุญาตให้ merge pull request ได้แม้ CI จะกำลังรันอยู่หรือ**ล้มเหลว**ก็ตาม **Branch Protection Rules** คือฟีเจอร์ของ GitHub ที่แก้ปัญหานี้โดยตรง ตั้งค่าผ่านหน้าเว็บที่ **Repository → Settings → Branches → Add branch protection rule** (ตั้งชื่อ pattern เป็น `main`) แล้วเลือก:

- **Require a pull request before merging** — ห้าม push ตรงเข้า `main` เลย ต้องผ่าน pull request เท่านั้น
- **Require status checks to pass before merging** — เลือก job ที่ต้องผ่านให้ครบ (`lint-and-vet`, `unit-test`, `integration-test`) ปุ่ม **Merge** บนหน้า PR จะถูก**ปิดใช้งานโดยอัตโนมัติ**จนกว่า job เหล่านี้จะรายงานผลเป็นสีเขียวครบทุกตัว
- **Require branches to be up to date before merging** — บังคับให้ branch ของ PR merge เอา `main` ล่าสุดเข้ามาก่อน ป้องกันปัญหา "PR สองอันต่างผ่าน CI แยกกัน แต่พอรวมกันแล้วพัง" (classic integration bug)
- **Require review from Code Owners** (ถ้ามีไฟล์ `CODEOWNERS`) — บังคับให้มีคนที่เกี่ยวข้องกับโค้ดส่วนนั้น review ก่อน merge เสมอ

ผลลัพธ์เชิงพฤติกรรมของทีมหลังตั้งค่านี้: **ไม่มีทางที่โค้ดที่ `go build` ไม่ผ่าน, มี data race, หรือ lint ไม่ผ่าน จะเข้าไปอยู่ใน `main` branch ได้เลย** เพราะระบบบังคับในระดับ infrastructure ไม่ใช่แค่อาศัยวินัยของแต่ละคน — นี่คือเหตุผลว่าทำไม branch protection ถึงเป็นมาตรฐานพื้นฐานของแทบทุกทีมวิศวกรรมซอฟต์แวร์ที่ทำงานเป็นทีม ไม่ว่าจะขนาดเล็กหรือใหญ่แค่ไหนก็ตาม

การตั้งค่านี้ทำผ่านหน้าเว็บของ GitHub ล้วนๆ ไม่มีไฟล์ YAML ให้ verify แบบเดียวกับ `ci.yml` — บทนี้จึงอธิบายในเชิงแนวคิดและขั้นตอนเท่านั้น (ตามที่ระบุไว้ในโจทย์ของบทนี้) ไม่มีอะไรให้รันทดสอบจริงในสภาพแวดล้อมแซนด์บ็อกซ์นี้

---

## 12. หมายเหตุความซื่อสัตย์เรื่องการรันจริงในบทนี้

- **รันจริงและยืนยันผลได้**: `go build`, `go vet`, `go test -race -cover` (ได้ coverage 100%), และ `golangci-lint run` ทั้งกรณีโค้ดถูกต้อง (0 issues) และกรณีจงใจใส่บั๊ก (`unused variable`) เพื่อพิสูจน์ว่า linter จับได้จริง — รันบนเครื่องที่เขียนหลักสูตรนี้ด้วย Go 1.24.7 และ `golangci-lint` 2.5.0 จริง
- **รันจริงและยืนยันผลได้**: `actionlint` v1.7.7 กับไฟล์ `ci.yml` — พบบั๊กจริง (`uses: golangci-lint-action@v6` ขาด owner) ระหว่างเขียนบทความนี้จริง ไม่ใช่ตัวอย่างที่แต่งขึ้น และ `yamllint` v1.38.0 ตรวจ syntax YAML พื้นฐานจริงเช่นกัน
- **ไม่ได้รันจริง**: การรัน workflow นี้บน GitHub Actions runner จริง (ต้องมี GitHub repository ของจริงพร้อม push event เพื่อ trigger, และสภาพแวดล้อมนี้ไม่ได้เชื่อมกับ repository ที่จะใช้ทดสอบ CI แบบ end-to-end) รวมถึงขั้นตอน Branch Protection ในหัวข้อ 11 ที่เป็นการตั้งค่าผ่านหน้าเว็บ GitHub ล้วนๆ ไม่มีอะไรให้รันในเทอร์มินัล — ทั้งสองส่วนนี้อธิบายจากพฤติกรรมมาตรฐานของ GitHub Actions ตามเอกสารทางการ

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- **CI/CD** เชื่อม Docker (Part 095), Docker Compose (Part 096), และ Kubernetes (Part 097) เข้าเป็นระบบอัตโนมัติเดียว — ไม่ต้องมีคนพิมพ์คำสั่ง build/test/deploy เองอีกต่อไป
- โครงสร้าง workflow ของ GitHub Actions ประกอบด้วย `on` (trigger), `jobs` (รันแยกกันโดย default), และ `steps` (รันเรียงลำดับในเครื่องเดียวกัน) — `needs:` คือกลไกหลักที่ทำให้ job หนึ่งรอ job อื่นให้เสร็จก่อน
- หลักการ **"fail fast"** เรียง stage จากเร็ว/ถูกไปหาช้า/แพง: lint & vet → unit test (matrix) → integration test → build & push image — ถ้า stage ต้นล้มเหลว stage หลังจะไม่ถูกรันเลย ประหยัดทั้งเวลาและทรัพยากร
- **Matrix testing** (`strategy.matrix`) รัน test ข้ามหลายเวอร์ชัน Go พร้อมกันจริง เพื่อพิสูจน์ Go 1 Compatibility Promise ด้วยตัวเอง, **`cache: true`** ของ `setup-go` ลดเวลา build ด้วย module cache
- **Service container** ของ GitHub Actions (`services:`) รัน PostgreSQL จริงคู่กับ integration test พร้อม health check — แนวคิดเดียวกับ `depends_on` + `condition: service_healthy` ของ Docker Compose ใน Part 096 เพียงเปลี่ยนบริบท
- Job สุดท้าย build และ push Docker image ขึ้น `ghcr.io` **เฉพาะตอน push เข้า `main` เท่านั้น** (ผ่าน `if:` condition) ไม่ใช่ตอนเปิด pull request — image ที่ได้พร้อมให้ Kubernetes Deployment ใน Part 097 ดึงไปใช้ได้ทันที
- รันจริงยืนยันว่า `go build`/`go vet`/`go test -race -cover`/`golangci-lint` ทำงานถูกต้อง และ `golangci-lint` จับ `unused variable` ได้จริงเมื่อจงใจใส่บั๊ก
- `actionlint` จับบั๊กจริงที่เกิดขึ้นจริงระหว่างเขียนบทนี้ (action ชื่อขาด owner) พิสูจน์ว่าการตรวจสอบ workflow YAML ด้วยเครื่องมือเฉพาะทางมีประโยชน์จริง ไม่ใช่แค่ทฤษฎี
- **Branch Protection Rules** บังคับให้ CI ต้องผ่านก่อน merge ได้เสมอในระดับ infrastructure ของ GitHub เอง ไม่ใช่แค่อาศัยวินัยของทีม

## แบบฝึกหัดท้ายบท

1. สร้าง repository ใหม่บน GitHub ใส่โค้ด `price` package จากหัวข้อ 9 พร้อมไฟล์ `.github/workflows/ci.yml` จากหัวข้อ 8 แล้ว push จริง ดูผลลัพธ์บนแท็บ **Actions** ของ GitHub ว่า job ทั้ง 4 ตัวรันและรายงานสถานะตามที่บทความอธิบายไว้หรือไม่ (Stage 3-4 จะข้ามหรือ error เพราะยังไม่มี integration test/Dockerfile จริงในโปรเจกต์ตัวอย่างนี้ — ลองปรับให้ครบ)
2. ลองใส่บั๊กแบบเดียวกับหัวข้อ 9 (unused variable) เข้าไปใน branch ใหม่ แล้วเปิด pull request ดูว่า `lint-and-vet` job ล้มเหลวจริงบนหน้า GitHub และ job อื่นถูกข้ามไปตามหลักการ fail-fast หรือไม่
3. ติดตั้ง `actionlint` บนเครื่องตัวเอง (หรือใช้ pre-commit hook) แล้วลองแก้ `ci.yml` ให้มี typo แบบอื่น (เช่น อ้างอิง `${{ matrix.go-verion }}` สะกดผิด) แล้วดูว่า `actionlint` จับได้หรือไม่ อธิบายว่าทำไมถึงจับได้หรือจับไม่ได้
4. ตั้งค่า Branch Protection Rule จริงบน repository ทดสอบของตัวเองตามหัวข้อ 11 แล้วลองสร้าง PR ที่มีบั๊ก ยืนยันว่าปุ่ม Merge ถูกปิดใช้งานจริงจนกว่า CI จะผ่าน
5. เพิ่ม job ใหม่ชื่อ `lint-workflow` เข้าไปใน `ci.yml` ที่รัน `actionlint` กับไฟล์ `ci.yml` เอง (ให้ CI ตรวจสอบตัวเองทุกครั้งที่มีการแก้ไข workflow) — ลองคิดว่า job นี้ควรอยู่ก่อนหรือหลัง `lint-and-vet` ตามหลักการ fail-fast
6. ค้นคว้าเพิ่มเติมเรื่อง **reusable workflows** (`workflow_call`) ของ GitHub Actions — ถ้าองค์กรมี microservices หลายสิบตัวที่ใช้ pattern การ build/test แบบเดียวกันหมด (ตามที่เรียนใน Part 088) จะรวมโค้ด `ci.yml` ที่ซ้ำกันให้เหลือไฟล์เดียวแล้วเรียกใช้ซ้ำได้อย่างไร

---

**ต่อไป**: [Part 099 — Monitoring และ Observability: Prometheus, Grafana, OpenTelemetry](./099-monitoring-and-observability.md)
