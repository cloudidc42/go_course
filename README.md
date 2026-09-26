# หลักสูตร Go (Golang) ฉบับสมบูรณ์ — จากพื้นฐานสู่ระดับมืออาชีพและระดับโลก

หลักสูตรนี้สอนภาษา Go ตั้งแต่ระดับพื้นฐานที่สุด ไปจนถึงระดับที่สามารถทำงานเป็น Go Developer มืออาชีพระดับโลกได้ โดยแบ่งเนื้อหาออกเป็น **part** ย่อยๆ ไว้ในโฟลเดอร์ [`parts/`](./parts) แต่ละ part มีตัวอย่างโค้ดที่รันได้จริง อธิบายละเอียด และมีแบบฝึกหัดท้ายบท

> เนื้อหาอยู่ระหว่างการเขียนต่อเนื่อง (work in progress) — ดูสถานะความคืบหน้าด้านล่างของแต่ละ part

## วิธีใช้หลักสูตรนี้

1. เรียงตามลำดับ part 1 → part สุดท้าย เพราะแต่ละบทต่อยอดจากบทก่อนหน้า
2. ติดตั้ง Go เวอร์ชันล่าสุด (ดู part 1) และลองรันโค้ดตัวอย่างทุกไฟล์ด้วยตัวเอง
3. ทำแบบฝึกหัดท้ายบทก่อนไป part ถัดไป
4. โปรเจกต์ใหญ่ท้ายหลักสูตร (ภาคที่ 10) ใช้รวบยอดทุกอย่างที่เรียนมา

## สารบัญทั้งหมด (Table of Contents)

### ภาคที่ 1: พื้นฐานภาษา Go (Fundamentals) — Part 1–15
| Part | หัวข้อ |
|---|---|
| 001 | แนะนำภาษา Go, ปรัชญาการออกแบบ, การติดตั้ง และเครื่องมือ |
| 002 | โครงสร้างโปรแกรม, package, Go Module, `go.mod` |
| 003 | ตัวแปร, ชนิดข้อมูล, ค่าคงที่, `iota` |
| 004 | Operators และ Control Flow (`if`/`else`, `switch`) |
| 005 | Loops: `for` ทุกรูปแบบ |
| 006 | Arrays และ Slices เจาะลึก |
| 007 | Maps เจาะลึก |
| 008 | Functions พื้นฐาน, multiple return values |
| 009 | Functions ขั้นสูง: variadic, closures, `defer` |
| 010 | Pointers เจาะลึก |
| 011 | Structs เจาะลึก |
| 012 | Methods และ Receiver (value vs pointer) |
| 013 | Interfaces พื้นฐาน |
| 014 | Interfaces ขั้นสูง: type assertion, type switch, empty interface |
| 015 | Error Handling พื้นฐาน |

### ภาคที่ 2: ระดับกลาง (Intermediate) — Part 16–35
| Part | หัวข้อ |
|---|---|
| 016 | Error Handling ขั้นสูง: custom error, `errors.Is/As`, wrapping |
| 017 | Panic, Recover และการจัดการข้อผิดพลาดร้ายแรง |
| 018 | การจัดการ Package และ Module ขั้นสูง (`go.sum`, replace, private module) |
| 019 | String และแพ็กเกจ `strings` |
| 020 | แพ็กเกจ `strconv` |
| 021 | การจัดรูปแบบด้วย `fmt` เจาะลึก |
| 022 | เวลาและแพ็กเกจ `time` |
| 023 | Regular Expressions ด้วย `regexp` |
| 024 | File I/O: `os`, `io`, `bufio` |
| 025 | JSON ด้วย `encoding/json` |
| 026 | XML และ YAML ใน Go |
| 027 | การเรียงลำดับด้วยแพ็กเกจ `sort` |
| 028 | Generics พื้นฐาน (Go 1.18+) |
| 029 | Generics ขั้นสูง: constraints, generic data structures |
| 030 | Struct Embedding และ Interface Embedding |
| 031 | Reflection พื้นฐานด้วย `reflect` |
| 032 | แพ็กเกจ `context` |
| 033 | Testing พื้นฐานด้วย `testing` |
| 034 | Table-Driven Tests และ Benchmarks |
| 035 | Mocking และ Test Doubles |

### ภาคที่ 3: การทำงานพร้อมกัน (Concurrency) — Part 36–45
| Part | หัวข้อ |
|---|---|
| 036 | Goroutines พื้นฐาน |
| 037 | Channels พื้นฐาน |
| 038 | คำสั่ง `select` |
| 039 | แพ็กเกจ `sync`: Mutex, WaitGroup, Once |
| 040 | `sync/atomic` และ Lock-Free Programming |
| 041 | Worker Pool Pattern |
| 042 | Fan-in / Fan-out Pattern |
| 043 | Context กับ Concurrency: cancellation, timeout |
| 044 | Race Condition และ Race Detector |
| 045 | Concurrency Patterns ขั้นสูง: Pipeline, Pub-Sub |

### ภาคที่ 4: Standard Library เชิงลึก — Part 46–55
| Part | หัวข้อ |
|---|---|
| 046 | `net/http` — HTTP Client เจาะลึก |
| 047 | `net/http` — HTTP Server เจาะลึก |
| 048 | `io.Reader` / `io.Writer` และ Interface Composition |
| 049 | `bufio` ขั้นสูง |
| 050 | แพ็กเกจ `bytes` |
| 051 | Cryptography พื้นฐาน: hashing, encryption |
| 052 | `os/exec` — รัน External Command |
| 053 | แพ็กเกจ `flag` — CLI Arguments |
| 054 | แพ็กเกจ `log` และแนวทาง Logging ที่ดี |
| 055 | แพ็กเกจ `runtime` และ Garbage Collector |

### ภาคที่ 5: Web Development — Part 56–70
| Part | หัวข้อ |
|---|---|
| 056 | HTTP Server ด้วย `net/http`: Routing, Handler |
| 057 | Middleware Pattern |
| 058 | Router ยอดนิยม: `gorilla/mux`, `chi` |
| 059 | RESTful API Design หลักการ |
| 060 | Request/Response และ JSON API |
| 061 | Web Framework: Gin เบื้องต้น |
| 062 | Gin ขั้นสูง: Validation, Binding, Grouping |
| 063 | Web Framework: Echo |
| 064 | Web Framework: Fiber |
| 065 | Templates ด้วย `html/template` |
| 066 | Static Files และ File Upload |
| 067 | Authentication ด้วย JWT |
| 068 | Authentication ด้วย OAuth2 และ Session |
| 069 | WebSocket ด้วย Go |
| 070 | GraphQL ด้วย Go |

### ภาคที่ 6: ฐานข้อมูล (Database) — Part 71–78
| Part | หัวข้อ |
|---|---|
| 071 | `database/sql` พื้นฐาน |
| 072 | PostgreSQL กับ Go (`pgx`, `lib/pq`) |
| 073 | MySQL กับ Go |
| 074 | ORM: GORM เบื้องต้น |
| 075 | GORM ขั้นสูง: Relations, Migrations, Hooks |
| 076 | MongoDB กับ Go |
| 077 | Redis กับ Go |
| 078 | Database Transactions และ Connection Pooling |

### ภาคที่ 7: Testing, Tooling, Performance — Part 79–87
| Part | หัวข้อ |
|---|---|
| 079 | Unit Testing ขั้นสูง |
| 080 | Integration Testing |
| 081 | `httptest` — ทดสอบ HTTP Handler |
| 082 | Test Coverage |
| 083 | Profiling ด้วย `pprof` |
| 084 | Benchmarking ขั้นสูง |
| 085 | Memory Management และ Optimization |
| 086 | Linting ด้วย `golangci-lint` และ Code Quality |
| 087 | Debugging ด้วย Delve |

### ภาคที่ 8: Microservices, gRPC, Message Queue — Part 88–94
| Part | หัวข้อ |
|---|---|
| 088 | หลักการ Microservices Architecture |
| 089 | gRPC เบื้องต้นด้วย Protocol Buffers |
| 090 | gRPC ขั้นสูง: Streaming, Interceptors |
| 091 | Message Queue: RabbitMQ กับ Go |
| 092 | Message Queue: Kafka กับ Go |
| 093 | Service Discovery และ API Gateway |
| 094 | Circuit Breaker และ Retry Pattern |

### ภาคที่ 9: DevOps และ Deployment — Part 95–99
| Part | หัวข้อ |
|---|---|
| 095 | Docker กับ Go Application |
| 096 | Docker Compose สำหรับ Multi-Service |
| 097 | Kubernetes เบื้องต้นสำหรับ Go Developer |
| 098 | CI/CD Pipeline ด้วย GitHub Actions |
| 099 | Monitoring และ Observability: Prometheus, Grafana, OpenTelemetry |

### ภาคที่ 10: มืออาชีพและระดับโลก (Professional & World-Class) — Part 100–110
| Part | หัวข้อ |
|---|---|
| 100 | Clean Architecture ใน Go |
| 101 | Domain-Driven Design (DDD) ใน Go |
| 102 | Design Patterns ใน Go |
| 103 | โปรเจกต์: E-Commerce REST API แบบเต็มรูปแบบ |
| 104 | โปรเจกต์: Real-time Chat Application |
| 105 | โปรเจกต์: URL Shortener พร้อม Caching |
| 106 | Security Best Practices สำหรับ Go |
| 107 | Performance Tuning ระดับ Production |
| 108 | Go Modules Best Practices และ Semantic Versioning |
| 109 | การมีส่วนร่วมใน Open Source โปรเจกต์ Go |
| 110 | เส้นทางอาชีพ: จาก Go Developer สู่ระดับ World-Class |

## สถานะความคืบหน้า

- [x] Part 001–078 (ภาคที่ 1–6 ครบทั้งหมด: พื้นฐาน, ระดับกลาง, Concurrency, Standard Library, Web Development, Database)
- [ ] Part 079–110 (กำลังทยอยเขียน)

ดูไฟล์แต่ละ part ได้ที่โฟลเดอร์ [`parts/`](./parts) ชื่อไฟล์รูปแบบ `NNN-slug.md`
