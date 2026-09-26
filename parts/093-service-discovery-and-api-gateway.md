# Part 093: Service Discovery และ API Gateway

> ภาคที่ 8: Microservices, gRPC, Message Queue — ตอนที่ 6 จาก 7

## สารบัญของบทนี้

1. ปัญหาที่ Service Discovery แก้: IP ที่เปลี่ยนตลอดเวลา
2. DNS-based Discovery: วิธีที่ Kubernetes ใช้ (foreshadow Part 097)
3. Client-side Discovery vs Server-side Discovery
4. Consul: เครื่องมือ Service Discovery แบบเฉพาะทาง
5. ตัวอย่างที่รันจริง: register + discover + health check ด้วย Consul
6. API Gateway Pattern คืออะไร และทำไมต้องมี
7. เชื่อมกับที่เรียนมาแล้ว: Middleware (Part 057) และ JWT (Part 067)
8. สร้าง API Gateway ด้วย `httputil.ReverseProxy`
9. ตัวอย่างที่รันจริง: Gateway พร้อม routing, auth, rate limiting, logging
10. เมื่อไหร่ควรใช้ Kong/Traefik แทนการเขียนเอง
11. สรุปสิ่งที่ได้เรียนในบทนี้
12. แบบฝึกหัดท้ายบท

---

## 1. ปัญหาที่ Service Discovery แก้: IP ที่เปลี่ยนตลอดเวลา

ใน Part 088 เราวางรากฐานว่า microservices คือระบบที่แตกเป็นหลาย service อิสระ สื่อสารกันผ่านเครือข่าย คำถามที่ตามมาทันทีคือ: **service A จะรู้ได้อย่างไรว่า service B อยู่ที่ไหน (IP + port อะไร)?**

ในยุคที่ deploy แอปลง server เครื่องเดียวแบบดั้งเดิม คำตอบง่ายมาก — เขียน IP ตายตัวไว้ใน config: `order-service` อยู่ที่ `10.0.1.5:8080` เสมอ จบ แต่ในโลกของ container orchestration (Docker, Kubernetes ที่จะเรียนใน Part 097) สมมติฐานนี้ **ใช้ไม่ได้อีกต่อไป** เพราะ:

- **Instance ถูกสร้างและทำลายตลอดเวลา** — autoscaling เพิ่ม instance ใหม่ตอนโหลดสูง แล้วลดตอนโหลดต่ำ, deployment ใหม่ฆ่า container เก่าสร้าง container ใหม่ (rolling update)
- **แต่ละ instance ใหม่ได้ IP ใหม่เสมอ** — container ที่ถูกสร้างใหม่ไม่ได้ IP เดิมของ container ที่ตายไป
- **จำนวน instance เปลี่ยนตามโหลด** — ตอนเที่ยงคืนอาจมี `order-service` แค่ 2 instance ตอนพีคช่วงเทศกาลอาจมี 50 instance

ถ้ายังเขียน IP ตายตัวไว้ใน config เหมือนเดิม ระบบจะพังทันทีที่มีการ scale หรือ deploy ใหม่ นี่คือปัญหาที่ **Service Discovery** เข้ามาแก้: มันคือกลไกที่ทำให้ service หนึ่งหา**ที่อยู่ปัจจุบัน**ของอีก service หนึ่งได้แบบไดนามิก โดยไม่ต้อง hardcode IP ไว้ล่วงหน้า

หัวใจของ service discovery มีสองส่วนเสมอ:

1. **Registration** — เมื่อ instance ใหม่พร้อมรับ traffic มันต้อง "ประกาศตัว" ว่า "ฉันชื่อ `order-service` อยู่ที่ IP นี้ port นี้"
2. **Lookup/Resolution** — เมื่อ service อื่นต้องการเรียก `order-service` มันถาม registry ว่า "ตอนนี้ `order-service` มี instance ไหนอยู่บ้างที่**ยังใช้งานได้** (healthy)"

---

## 2. DNS-based Discovery: วิธีที่ Kubernetes ใช้ (foreshadow Part 097)

วิธีที่ง่ายและใช้กันแพร่หลายที่สุดคือใช้ **DNS** เป็นกลไก discovery แนวคิดคือแทนที่จะจำ IP ตายตัว ให้จำ**ชื่อ** แล้วให้ DNS server แปลงชื่อเป็น IP ปัจจุบันให้เอง

ใน **Kubernetes** (ซึ่งเราจะเรียนเจาะลึกใน Part 097) กลไกนี้ฝังอยู่ในระบบเป็นค่าเริ่มต้นผ่านออบเจกต์ที่ชื่อ **Service**:

```
                          ┌─────────────────────┐
                          │   Kubernetes Service │
                          │   name: order-service│
                          │   (มี ClusterIP คงที่)│
                          └──────────┬───────────┘
                                     │ load balance
                    ┌────────────────┼────────────────┐
                    ▼                ▼                ▼
              ┌──────────┐    ┌──────────┐      ┌──────────┐
              │  Pod #1  │    │  Pod #2  │      │  Pod #3  │
              │ IP: .5   │    │ IP: .9   │      │ IP: .14  │
              └──────────┘    └──────────┘      └──────────┘
              (IP เปลี่ยนได้ตลอดเวลาเมื่อ pod ถูกสร้างใหม่)
```

เมื่อ pod ของ `order-service` ถูกสร้าง/ทำลาย IP ของแต่ละ pod เปลี่ยนไปเรื่อยๆ แต่ **ชื่อ Service** (เช่น `order-service.default.svc.cluster.local`) **คงที่เสมอ** — โค้ดของ service อื่นแค่เรียก `http://order-service:8080` ตรงๆ เหมือนเป็น hostname ธรรมดา แล้ว Kubernetes DNS จะ resolve ชื่อนี้ไปยัง **ClusterIP** ที่คงที่ตัวหนึ่ง ซึ่งข้างหลังมันคือ load balancer ระดับ kernel (kube-proxy) ที่กระจาย traffic ไปยัง pod ที่ยัง healthy อยู่ ณ ขณะนั้น

ข้อดีของวิธีนี้คือ **โค้ดแอปพลิเคชันไม่ต้องรู้เรื่อง service discovery เลย** — ใช้ `net/http` เรียก URL ธรรมดาตามปกติ (เหมือนที่เรียนใน Part 046) เพราะความซับซ้อนทั้งหมดถูกซ่อนอยู่ในชั้น infrastructure เราจะเห็นวิธีตั้งค่าจริงเมื่อถึง Part 097

---

## 3. Client-side Discovery vs Server-side Discovery

DNS + load balancer แบบ Kubernetes ที่เพิ่งเห็นคือตัวอย่างของ **Server-side Discovery** แต่มีอีกแบบคือ **Client-side Discovery** ทั้งสองแบบแก้ปัญหาเดียวกันแต่ย้ายความรับผิดชอบไปอยู่คนละจุด:

### Server-side Discovery

```
Client ──► Load Balancer / Router ──► เลือก instance ที่ healthy ──► Service Instance
           (รู้ทะเบียน instance ทั้งหมด)
```

Client แค่ยิง request ไปยังจุดเดียว (เช่น DNS name หรือ load balancer address) ตัวกลาง (load balancer, Kubernetes Service, API Gateway) เป็นคนรู้ว่า instance ไหนมีอยู่บ้างและเลือกให้ **Client ไม่ต้องรู้อะไรเรื่อง discovery เลย** — นี่คือแบบที่ Kubernetes Service และ API Gateway (หัวข้อ 6) ใช้

### Client-side Discovery

```
Client ──► ถาม Service Registry ว่ามี instance ไหนบ้าง ──► เลือก instance เอง ──► Service Instance
           (เช่น ถาม Consul)
```

Client เชื่อมต่อกับ **service registry** (เช่น Consul) โดยตรง ถามว่า instance ไหนของ service เป้าหมาย healthy อยู่บ้าง แล้ว**เลือกเองว่าจะยิงไปตัวไหน** (round-robin เอง, หรือใช้ library ช่วย เช่น client-side load balancing) วิธีนี้ให้ client ควบคุม logic การเลือก instance ได้ละเอียดกว่า (เช่น เลือก instance ที่ latency ต่ำสุด) แต่แลกกับความซับซ้อนที่เพิ่มขึ้นในทุก client

| | Server-side Discovery | Client-side Discovery |
|---|---|---|
| ใครรู้เรื่อง registry | ตัวกลาง (load balancer/gateway) เท่านั้น | ทุก client ต้องรู้และคุยกับ registry เอง |
| ความซับซ้อนในโค้ด client | แทบไม่มี (เรียก URL ธรรมดา) | ต้องมี logic เลือก instance เอง |
| ตัวอย่างที่ใช้จริง | Kubernetes Service, AWS ELB, API Gateway | Netflix Eureka + Ribbon (ยุคก่อน Kubernetes), Consul แบบ query ตรงจาก client |
| จุดคอขวด | ตัวกลางต้องรับ traffic ทั้งหมด | ไม่มีตัวกลาง แต่ client หนักขึ้น |

แนวโน้มอุตสาหกรรมปัจจุบันเอียงไปทาง **server-side discovery** เป็นหลัก (ผ่าน Kubernetes + service mesh) เพราะลดความซับซ้อนที่ต้องเขียนซ้ำในทุก client แต่ **Consul แบบ client-side query** ยังใช้กันมากในระบบที่ไม่ได้รันบน Kubernetes หรือใช้ผสมกับ server-side ผ่าน Consul's built-in DNS interface

---

## 4. Consul: เครื่องมือ Service Discovery แบบเฉพาะทาง

**Consul** จาก HashiCorp (ผู้สร้าง Terraform, Vault ที่กล่าวถึงใน Part 001) เป็นเครื่องมือ service discovery + health checking แบบเฉพาะทางที่ใช้ได้ทั้งนอกและใน Kubernetes หลักการทำงาน:

1. **Agent** รันอยู่บนทุกเครื่อง (หรือทุก node) ทำหน้าที่รับการ register service และตรวจ health check
2. **Service registration** — แต่ละ service instance เมื่อ start ขึ้นมา จะเรียก Consul API เพื่อบอกว่า "ฉันคือ service ชื่อนี้ อยู่ที่ address/port นี้ ตรวจ health ได้ที่ endpoint นี้"
3. **Health checking** — Consul เรียก health check endpoint ของแต่ละ instance เป็นระยะ (เช่น ทุก 2 วินาที) ถ้า instance ไม่ตอบหรือตอบ error ต่อเนื่อง Consul จะทำเครื่องหมายว่า instance นั้น **critical** (ไม่ healthy)
4. **Discovery/Query** — service อื่นถาม Consul ว่า "ขอ instance ที่ **healthy** ของ service X หน่อย" Consul กรอง instance ที่ critical ออกให้อัตโนมัติ

จุดเด่นที่ทำให้ Consul ต่างจาก DNS ธรรมดา (แม้ Consul จะมี DNS interface ให้ใช้ด้วย) คือ **health checking ในตัว** — DNS ธรรมดารู้แค่ว่า "มี instance นี้อยู่" แต่ไม่รู้ว่ามันตอบสนองได้จริงหรือไม่ ส่วน Consul ตัดสินใจจาก health check จริงแบบ active monitoring

Go client อย่างเป็นทางการคือ `github.com/hashicorp/consul/api`

---

## 5. ตัวอย่างที่รันจริง: register + discover + health check ด้วย Consul

> **หมายเหตุความโปร่งใส**: ตัวอย่างนี้รันจริงในสภาพแวดล้อมที่เขียนบทนี้ โดยใช้ Docker รัน Consul agent จริง (image ทางการ `hashicorp/consul`) แล้วรันโปรแกรม Go เชื่อมต่อเข้าไปจริง ผลลัพธ์ที่แสดงคือผลลัพธ์จากการรันจริง

### เตรียม Consul agent

```bash
docker run -d --name consul-dev --network host hashicorp/consul:1.19 \
  agent -dev -client=0.0.0.0
```

(`-dev` รัน Consul ในโหมด single-node สำหรับพัฒนา/ทดลอง ไม่เหมาะกับโปรดักชัน — โปรดักชันจริงต้องตั้ง cluster หลาย node ตามคู่มือ Consul)

```bash
go mod init consul-demo
go get github.com/hashicorp/consul/api@latest
```

### โค้ด: register 2 instance, ดู health, แล้วจำลอง instance หนึ่งล่ม

```go
package main

import (
	"fmt"
	"log"
	"net/http"
	"time"

	consulapi "github.com/hashicorp/consul/api"
)

// startBackend จำลอง instance หนึ่งของ "order-service" ที่มี /health endpoint
func startBackend(port string) *http.Server {
	mux := http.NewServeMux()
	mux.HandleFunc("/health", func(w http.ResponseWriter, r *http.Request) {
		w.WriteHeader(http.StatusOK)
		w.Write([]byte("ok"))
	})
	mux.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintf(w, "hello from instance on port %s", port)
	})
	srv := &http.Server{Addr: ":" + port, Handler: mux}
	go srv.ListenAndServe()
	return srv
}

func main() {
	// 1. เริ่ม service instance สองตัว (จำลอง 2 replica ของ order-service)
	srv1 := startBackend("9101")
	srv2 := startBackend("9102")
	defer srv1.Close()
	defer srv2.Close()
	time.Sleep(300 * time.Millisecond)

	// 2. เชื่อมต่อ Consul agent ที่รันอยู่ในเครื่อง
	cfg := consulapi.DefaultConfig()
	cfg.Address = "127.0.0.1:8500"
	client, err := consulapi.NewClient(cfg)
	if err != nil {
		log.Fatalf("consul client: %v", err)
	}

	// 3. register ทั้งสอง instance เป็น service ชื่อเดียวกัน "order-service"
	//    พร้อมตั้ง health check ที่ Consul จะเรียกเองทุก 2 วินาที
	for _, port := range []string{"9101", "9102"} {
		reg := &consulapi.AgentServiceRegistration{
			ID:      "order-service-" + port,
			Name:    "order-service",
			Port:    mustAtoi(port),
			Address: "127.0.0.1",
			Check: &consulapi.AgentServiceCheck{
				HTTP:                           "http://127.0.0.1:" + port + "/health",
				Interval:                       "2s",
				Timeout:                        "1s",
				DeregisterCriticalServiceAfter: "30s",
			},
		}
		if err := client.Agent().ServiceRegister(reg); err != nil {
			log.Fatalf("register %s: %v", port, err)
		}
		fmt.Println("registered", reg.ID)
	}
	defer client.Agent().ServiceDeregister("order-service-9101")
	defer client.Agent().ServiceDeregister("order-service-9102")

	// รอให้ Consul ตรวจ health อย่างน้อย 1-2 รอบก่อน
	time.Sleep(4 * time.Second)

	// 4. discover: ขอเฉพาะ instance ที่ "healthy" (passing=true) ของ order-service
	services, _, err := client.Health().Service("order-service", "", true, nil)
	if err != nil {
		log.Fatalf("discover: %v", err)
	}
	fmt.Printf("\nConsul reports %d healthy instance(s) of order-service:\n", len(services))
	for _, s := range services {
		fmt.Printf("  -> %s:%d (service id: %s)\n", s.Service.Address, s.Service.Port, s.Service.ID)
	}

	// 5. จำลอง instance หนึ่งล่ม แล้วดูว่า Consul ตัดมันออกจากผลลัพธ์อัตโนมัติ
	fmt.Println("\nstopping instance on port 9101 to simulate a crash...")
	srv1.Close()
	time.Sleep(6 * time.Second) // รอให้ health check รอบถัดไปตรวจพบ

	services, _, err = client.Health().Service("order-service", "", true, nil)
	if err != nil {
		log.Fatalf("discover after crash: %v", err)
	}
	fmt.Printf("Consul now reports %d healthy instance(s):\n", len(services))
	for _, s := range services {
		fmt.Printf("  -> %s:%d (service id: %s)\n", s.Service.Address, s.Service.Port, s.Service.ID)
	}
}

func mustAtoi(s string) int {
	n := 0
	for _, c := range s {
		n = n*10 + int(c-'0')
	}
	return n
}
```

### ผลลัพธ์จากการรันจริง

```
registered order-service-9101
registered order-service-9102

Consul reports 2 healthy instance(s) of order-service:
  -> 127.0.0.1:9101 (service id: order-service-9101)
  -> 127.0.0.1:9102 (service id: order-service-9102)

stopping instance on port 9101 to simulate a crash...
Consul now reports 1 healthy instance(s):
  -> 127.0.0.1:9102 (service id: order-service-9102)
```

สิ่งที่พิสูจน์ได้จากผลลัพธ์นี้: เมื่อ instance บน port 9101 ถูกปิดลง (จำลองการล่ม) โดย**ไม่มีใครไปบอก Consul ตรงๆ ว่า instance นี้ตายแล้ว** — Consul รู้เองจาก health check ที่มันเรียก `/health` ทุก 2 วินาทีแล้วเจอ connection refused ต่อเนื่อง จึงตัด instance นั้นออกจากรายการที่ query ด้วย `passing=true` โดยอัตโนมัติ นี่คือคุณค่าหลักของ Consul: **service ที่มาเรียกไม่มีทางยิง request ไปโดน instance ที่ตายแล้วเลย** ตราบใดที่มัน query ผ่าน Consul เสมอ

---

## 6. API Gateway Pattern คืออะไร และทำไมต้องมี

เมื่อระบบแตกเป็นหลาย microservice คำถามต่อมาคือ: **client ภายนอก (mobile app, web frontend, third-party) ควรเรียก service ไหนโดยตรง?**

ถ้าปล่อยให้ client เรียก `user-service`, `order-service`, `payment-service` ตรงๆ ทีละตัว จะเจอปัญหา:

- Client ต้องรู้ที่อยู่ของทุก service (ผูก client เข้ากับโครงสร้างภายในที่ควรเปลี่ยนได้อิสระ)
- ทุก service ต้อง implement authentication, rate limiting, logging, CORS **ซ้ำกันเอง**
- เปลี่ยนโครงสร้างภายใน (แตก service เพิ่ม, รวม service, ย้าย service) กระทบ client โดยตรง
- ไม่มีจุดกลางสำหรับ monitoring traffic ทั้งระบบ

**API Gateway** คือ service ตัวหนึ่งที่ยืนอยู่หน้าสุด เป็น **จุดเข้าเดียว (single entry point)** ให้ client ภายนอกคุยด้วย แล้ว gateway เป็นผู้ตัดสินใจว่าจะ route request ไปยัง backend service ตัวไหน:

```
                              ┌───────────────┐
Mobile App ──┐                │               │──► user-service
             │                │  API Gateway  │
Web App ─────┼──── HTTPS ────►│  - routing    │──► order-service
             │                │  - auth       │
3rd-party ───┘                │  - rate limit │──► payment-service
                               │  - logging    │
                               └───────────────┘
```

หน้าที่ทั่วไปของ API Gateway:

- **Routing**: ส่ง request ไปยัง backend service ที่ถูกต้องตาม path/host
- **Authentication/Authorization**: ตรวจสอบตัวตนผู้เรียก**ครั้งเดียวที่ gateway** แทนที่จะให้ทุก backend service ตรวจซ้ำ
- **Rate Limiting**: จำกัดจำนวน request ต่อ client เพื่อป้องกัน abuse และปกป้อง backend
- **Logging/Monitoring**: จุดเดียวที่เห็น traffic ทั้งหมดของระบบ เหมาะสำหรับเก็บ metrics รวม
- **Response aggregation** (บางกรณี): รวมผลลัพธ์จากหลาย backend service เป็น response เดียว

---

## 7. เชื่อมกับที่เรียนมาแล้ว: Middleware (Part 057) และ JWT (Part 067)

API Gateway ที่จะสร้างในหัวข้อถัดไปแท้จริงแล้วคือการเอาความรู้ที่เรียนมาแล้วสอง part มาประกอบกัน:

- **Part 057 (Middleware Pattern)**: เราเรียนไปแล้วว่า middleware คือฟังก์ชันที่ wrap `http.Handler` เพื่อทำงานบางอย่างก่อน/หลัง handler จริง (logging, recovery, CORS) — API Gateway คือการเอา middleware chain นั้นมาวางไว้**หน้า reverse proxy** แทนที่จะวางหน้า business logic handler โดยตรง
- **Part 067 (JWT Authentication)**: การตรวจ JWT ที่เคยเขียนเป็น middleware ในแต่ละ service ตอนนี้ย้ายมาอยู่ที่ gateway จุดเดียว — backend service ที่อยู่หลัง gateway **ไม่จำเป็นต้องตรวจ JWT ซ้ำอีก** (ถ้า design ให้เชื่อ gateway อย่างเต็มที่ หรือ อาจตรวจซ้ำเบาๆ อีกชั้นเป็น defense-in-depth ก็ได้ แล้วแต่ระดับความปลอดภัยที่ต้องการ)

พูดง่ายๆ: **API Gateway คือ middleware chain ที่ endpoint สุดท้ายไม่ใช่ business logic handler แต่เป็น reverse proxy ไปยัง service อื่น**

---

## 8. สร้าง API Gateway ด้วย `httputil.ReverseProxy`

Standard library ของ Go มี `net/http/httputil.ReverseProxy` ให้ใช้ทำ reverse proxy ได้โดยไม่ต้องพึ่ง library ภายนอกเลย นี่คือฟังก์ชันสร้างพื้นฐานที่สุด:

```go
package main

import (
	"net/http"
	"net/http/httputil"
	"net/url"
)

func newProxyTo(target *url.URL) http.Handler {
	return httputil.NewSingleHostReverseProxy(target)
}
```

`NewSingleHostReverseProxy` สร้าง `http.Handler` ที่ forward ทุก request ที่มันได้รับไปยัง `target` โดยคง path เดิม, copy header ส่วนใหญ่, และ stream response กลับมาให้ client — ทั้งหมดนี้เป็นสิ่งที่ standard library จัดการให้อัตโนมัติ ไม่ต้องเขียน HTTP client เองแบบ Part 046

การสร้าง gateway ที่ route หลาย backend ตาม path prefix ทำได้ด้วยการผูก `http.ServeMux` เข้ากับ proxy หลายตัว:

```go
mux := http.NewServeMux()
mux.Handle("/users/", httputil.NewSingleHostReverseProxy(usersServiceURL))
mux.Handle("/orders/", httputil.NewSingleHostReverseProxy(ordersServiceURL))
```

แล้วห่อ `mux` ด้วย middleware chain (logging, auth, rate limit) แบบเดียวกับ Part 057

---

## 9. ตัวอย่างที่รันจริง: Gateway พร้อม routing, auth, rate limiting, logging

ตัวอย่างต่อไปนี้เป็นโปรแกรม Go เดี่ยว ไม่ต้องพึ่ง library ภายนอกเลย (pure standard library) — สร้าง backend สองตัวจำลอง (`users`, `orders`), gateway หนึ่งตัวที่ทำ logging + auth + rate limiting + routing แล้วยิง request จริงทดสอบทุก path

```go
package main

import (
	"fmt"
	"io"
	"log"
	"net/http"
	"net/http/httptest"
	"net/http/httputil"
	"net/url"
	"strings"
	"sync"
	"time"
)

// ---- backend service 1: users ----
func newUsersService() *httptest.Server {
	mux := http.NewServeMux()
	mux.HandleFunc("/users/", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintf(w, `{"service":"users","path":"%s"}`, r.URL.Path)
	})
	return httptest.NewServer(mux)
}

// ---- backend service 2: orders ----
func newOrdersService() *httptest.Server {
	mux := http.NewServeMux()
	mux.HandleFunc("/orders/", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintf(w, `{"service":"orders","path":"%s"}`, r.URL.Path)
	})
	return httptest.NewServer(mux)
}

// loggingMiddleware บันทึก method, path, status, latency ของทุก request ที่ผ่าน gateway
// (แนวคิดเดียวกับ middleware chain ใน Part 057)
func loggingMiddleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		start := time.Now()
		rec := &statusRecorder{ResponseWriter: w, status: http.StatusOK}
		next.ServeHTTP(rec, r)
		log.Printf("%s %s -> %d (%s)", r.Method, r.URL.Path, rec.status, time.Since(start))
	})
}

type statusRecorder struct {
	http.ResponseWriter
	status int
}

func (r *statusRecorder) WriteHeader(code int) {
	r.status = code
	r.ResponseWriter.WriteHeader(code)
}

// authMiddleware ทำหน้าที่แทนการตรวจ JWT จาก Part 067: gateway ตรวจ auth
// "ครั้งเดียว" ที่ขอบระบบ แทนที่จะให้ backend แต่ละตัวตรวจซ้ำเอง
func authMiddleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		authz := r.Header.Get("Authorization")
		if !strings.HasPrefix(authz, "Bearer ") || strings.TrimPrefix(authz, "Bearer ") == "" {
			http.Error(w, "missing or invalid bearer token", http.StatusUnauthorized)
			return
		}
		// โปรดักชันจริงจะเรียก jwt.ParseWithClaims ตรงนี้ (ดู Part 067)
		next.ServeHTTP(w, r)
	})
}

// rateLimiter แบบ fixed-window อย่างง่าย แยกนับตาม client IP เพื่อสาธิต
type rateLimiter struct {
	mu       sync.Mutex
	limit    int
	window   time.Duration
	counters map[string]int
	resetAt  time.Time
}

func newRateLimiter(limit int, window time.Duration) *rateLimiter {
	return &rateLimiter{limit: limit, window: window, counters: map[string]int{}, resetAt: time.Now().Add(window)}
}

func (rl *rateLimiter) middleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		rl.mu.Lock()
		if time.Now().After(rl.resetAt) {
			rl.counters = map[string]int{}
			rl.resetAt = time.Now().Add(rl.window)
		}
		key := r.RemoteAddr
		rl.counters[key]++
		count := rl.counters[key]
		rl.mu.Unlock()

		if count > rl.limit {
			http.Error(w, "rate limit exceeded", http.StatusTooManyRequests)
			return
		}
		next.ServeHTTP(w, r)
	})
}

// buildGateway ประกอบ reverse proxy สองตัวเข้ากับ ServeMux: request ที่ขึ้นต้น
// ด้วย /users/ ไปที่ users service, /orders/ ไปที่ orders service เท่านั้นที่
// gateway ต้องรู้ที่อยู่จริงของแต่ละ backend -- นี่คือสิ่งที่ทำให้ gateway
// ทำหน้าที่เป็น "service registry อย่างง่าย" ไปในตัวด้วย
func buildGateway(usersURL, ordersURL *url.URL) http.Handler {
	usersProxy := httputil.NewSingleHostReverseProxy(usersURL)
	ordersProxy := httputil.NewSingleHostReverseProxy(ordersURL)

	mux := http.NewServeMux()
	mux.Handle("/users/", usersProxy)
	mux.Handle("/orders/", ordersProxy)
	mux.HandleFunc("/healthz", func(w http.ResponseWriter, r *http.Request) {
		w.Write([]byte("gateway ok"))
	})

	limiter := newRateLimiter(5, time.Minute)
	var handler http.Handler = mux
	handler = authMiddleware(handler)
	handler = limiter.middleware(handler)
	handler = loggingMiddleware(handler)
	return handler
}

func main() {
	users := newUsersService()
	defer users.Close()
	orders := newOrdersService()
	defer orders.Close()

	usersURL, _ := url.Parse(users.URL)
	ordersURL, _ := url.Parse(orders.URL)

	gateway := httptest.NewServer(buildGateway(usersURL, ordersURL))
	defer gateway.Close()

	fmt.Println("users backend :", users.URL)
	fmt.Println("orders backend:", orders.URL)
	fmt.Println("gateway       :", gateway.URL)
	fmt.Println()

	client := &http.Client{Timeout: 5 * time.Second}

	// 1. request ที่ไม่มี token -> gateway บล็อกก่อนถึง backend เลย
	resp, _ := client.Get(gateway.URL + "/users/42")
	fmt.Println("no token       ->", resp.StatusCode)
	io.Copy(io.Discard, resp.Body)
	resp.Body.Close()

	// 2. request ที่มี token -> ถูก route ไปยัง users service
	req, _ := http.NewRequest("GET", gateway.URL+"/users/42", nil)
	req.Header.Set("Authorization", "Bearer demo-token")
	resp, _ = client.Do(req)
	respBody, _ := io.ReadAll(resp.Body)
	resp.Body.Close()
	fmt.Printf("GET /users/42  -> %d %s\n", resp.StatusCode, respBody)

	// 3. request ที่มี token -> ถูก route ไปยัง orders service
	req, _ = http.NewRequest("GET", gateway.URL+"/orders/99", nil)
	req.Header.Set("Authorization", "Bearer demo-token")
	resp, _ = client.Do(req)
	respBody, _ = io.ReadAll(resp.Body)
	resp.Body.Close()
	fmt.Printf("GET /orders/99 -> %d %s\n", resp.StatusCode, respBody)

	// 4. ยิงรัวๆ เพื่อชน rate limiter (ใช้ client เดิมเพื่อให้ RemoteAddr คงที่)
	blocked := 0
	for i := 0; i < 10; i++ {
		req, _ = http.NewRequest("GET", gateway.URL+"/users/1", nil)
		req.Header.Set("Authorization", "Bearer demo-token")
		resp, _ = client.Do(req)
		io.Copy(io.Discard, resp.Body)
		resp.Body.Close()
		if resp.StatusCode == http.StatusTooManyRequests {
			blocked++
		}
	}
	fmt.Printf("of 10 rapid requests, %d were rejected by the rate limiter (limit=5/min)\n", blocked)
}
```

### ผลลัพธ์จากการรันจริง

```
users backend : http://127.0.0.1:38829
orders backend: http://127.0.0.1:41261
gateway       : http://127.0.0.1:41437

2026/09/26 03:51:57 GET /users/42 -> 401 (55.044µs)
no token       -> 401
2026/09/26 03:51:57 GET /users/42 -> 200 (893.213µs)
GET /users/42  -> 200 {"service":"users","path":"/users/42"}
2026/09/26 03:51:57 GET /orders/99 -> 200 (583.401µs)
GET /orders/99 -> 200 {"service":"orders","path":"/orders/99"}
2026/09/26 03:51:57 GET /users/1 -> 200 (354.164µs)
2026/09/26 03:51:57 GET /users/1 -> 200 (180.548µs)
2026/09/26 03:51:57 GET /users/1 -> 429 (4.257µs)
2026/09/26 03:51:57 GET /users/1 -> 429 (6.4µs)
2026/09/26 03:51:57 GET /users/1 -> 429 (3.484µs)
2026/09/26 03:51:57 GET /users/1 -> 429 (14.06µs)
2026/09/26 03:51:57 GET /users/1 -> 429 (4.579µs)
2026/09/26 03:51:57 GET /users/1 -> 429 (4.962µs)
2026/09/26 03:51:57 GET /users/1 -> 429 (5.633µs)
2026/09/26 03:51:57 GET /users/1 -> 429 (4.814µs)
of 10 rapid requests, 8 were rejected by the rate limiter (limit=5/min)
```

สังเกตพฤติกรรมที่ยืนยันว่า gateway ทำงานถูกต้องครบทุกหน้าที่:

- Request ที่ไม่มี `Authorization` header ถูกปฏิเสธด้วย `401` **ก่อน**ไปถึง reverse proxy เลย (auth middleware ทำงานตัดหน้า)
- Request ที่มี token ถูก route ไปยัง backend ที่ถูกต้องตาม path prefix — `/users/*` ไปที่ users service, `/orders/*` ไปที่ orders service โดยที่ client ไม่รู้เลยว่า backend จริงอยู่ที่ port ไหน
- หลังจาก request ที่ 5 (นับรวมทุก request ที่ผ่าน auth ตั้งแต่ต้นโปรแกรม) rate limiter เริ่มตอบ `429 Too Many Requests` — ปกป้อง backend จากการถูกยิงรัวเกินขีดจำกัด
- Log ทุกบรรทัดมาจากจุดเดียว (gateway) แสดง method, path, status, latency ของทุก request ที่เข้าระบบ — เป็นจุดเดียวที่เห็นภาพรวม traffic ทั้งหมด

---

## 10. เมื่อไหร่ควรใช้ Kong/Traefik แทนการเขียนเอง

ตัวอย่างในหัวข้อ 9 คือ gateway "ของจริง" ที่ทำงานได้ แต่มันยังห่างไกลจาก gateway ระดับโปรดักชันของบริษัทใหญ่ ก่อนตัดสินใจว่าจะเขียนเองหรือใช้ของสำเร็จรูป ควรรู้ว่าแต่ละทางเลือกให้อะไร:

### เขียนเอง (hand-rolled Go gateway แบบที่เพิ่งทำ)

**เหมาะเมื่อ**:
- ระบบมีไม่กี่ backend service, requirement เรื่อง routing/auth ไม่ซับซ้อน
- ต้องการควบคุม logic เฉพาะทางที่ product ของสำเร็จรูปไม่รองรับ (เช่น business logic พิเศษที่ต้องรันที่ gateway)
- ทีมถนัด Go อยู่แล้ว อยากลดจำนวนเทคโนโลยีที่ต้องดูแลในระบบ (ไม่อยากเพิ่ม Lua scripting ของ Kong หรือ config YAML ซับซ้อนของเครื่องมืออื่น)
- ต้องการ binary เดียวที่ deploy ง่าย (ตรงกับปรัชญา Go ที่เจอมาตั้งแต่ Part 001)

**ข้อจำกัด**: ต้องดูแลเองทุกอย่าง — TLS termination, circuit breaking (Part 094), retries, observability, hot-reload config, plugin ecosystem, WAF/security rules ที่ของสำเร็จรูปมักมีให้ครบแล้ว

### Kong

Kong เป็น API Gateway ที่สร้างบน nginx/OpenResty มี plugin ecosystem ใหญ่มาก (authentication หลายแบบ, rate limiting ละเอียด, logging ไปหลายปลายทาง, transformation) ตั้งค่าผ่าน REST Admin API หรือ declarative config (YAML) เหมาะกับองค์กรที่มี backend service จำนวนมาก ต้องการ policy การจัดการ API ที่ซับซ้อนและเปลี่ยนบ่อย โดยไม่อยากแก้โค้ดทุกครั้งที่เปลี่ยน policy

### Traefik

Traefik โดดเด่นเรื่อง **auto-discovery** — มันอ่าน label/annotation จาก Docker, Kubernetes, Consul โดยตรง แล้ว config ตัวเองอัตโนมัติเมื่อมี service ใหม่ขึ้น/ลง (ไม่ต้องแก้ config file ด้วยมือ) เหมาะมากกับสภาพแวดล้อมที่ deploy บ่อยและ service เปลี่ยนแปลงตลอดเวลา (จะเจอแนวคิดนี้อีกครั้งเมื่อพูดถึง Ingress Controller ใน Part 097)

### ตารางสรุปการตัดสินใจ

| ปัจจัย | เขียนเอง (Go) | Kong / Traefik |
|---|---|---|
| จำนวน backend service | น้อย (< 10) | มาก และเพิ่มขึ้นเรื่อยๆ |
| ความซับซ้อนของ policy | ตรงไปตรงมา | ซับซ้อน เปลี่ยนบ่อย ต้องการ UI/config จัดการ |
| ทีมต้องการควบคุมเต็มที่ | ใช่ | ไม่จำเป็น ยอมรับ convention ของเครื่องมือได้ |
| ต้องการ auto-discovery จาก orchestrator | ต้องเขียนเพิ่มเอง | มีในตัว (โดยเฉพาะ Traefik) |
| Operational overhead ที่ยอมรับได้ | ต่ำ (แค่ binary Go) | ต้องดูแล component เพิ่ม |

ในทางปฏิบัติ หลายทีมเริ่มจากเขียน gateway เองแบบง่ายๆ (เหมือนหัวข้อ 9) ตอนระบบยังเล็ก แล้วค่อยย้ายไปใช้ Kong/Traefik เมื่อจำนวน service และความซับซ้อนของ policy โตเกินจุดที่โค้ดเองจัดการไหว — เป็นการตัดสินใจที่ผูกกับขนาดและอัตราการเปลี่ยนแปลงของระบบ ไม่ใช่กฎตายตัวว่าต้องใช้อะไรตั้งแต่ต้น

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- Service Discovery แก้ปัญหา IP ของ service instance ที่เปลี่ยนตลอดเวลาในสภาพแวดล้อมไดนามิก (autoscaling, rolling deployment) — ประกอบด้วย registration (instance ประกาศตัว) และ lookup (service อื่นถามหาที่อยู่ปัจจุบัน)
- **DNS-based discovery** คือแนวทางที่ Kubernetes Service ใช้: ชื่อคงที่ ข้างหลังคือ IP ที่เปลี่ยนได้ตลอด โค้ดแอปไม่ต้องรู้เรื่อง discovery เลย
- **Server-side discovery** (client เรียกจุดเดียว ตัวกลางเลือก instance ให้) vs **client-side discovery** (client คุยกับ registry แล้วเลือกเอง) — อุตสาหกรรมปัจจุบันเอียงไปทาง server-side ผ่าน Kubernetes
- **Consul** คือเครื่องมือ service discovery + health checking เฉพาะทาง ที่แสดงให้เห็นจริงในบทนี้ว่ามันตัด instance ที่ล่มออกจากผลลัพธ์การ query โดยอัตโนมัติ
- **API Gateway** คือจุดเข้าเดียวของระบบ ทำหน้าที่ routing, auth, rate limiting, logging รวมศูนย์ — ต่อยอดจาก middleware pattern (Part 057) และ JWT (Part 067) ที่เรียนมาแล้ว
- สร้าง gateway จริงด้วย `httputil.ReverseProxy` ได้โดยไม่ต้องพึ่ง library ภายนอกเลย และตัวอย่างในบทนี้พิสูจน์ว่า routing, auth, rate limiting ทำงานร่วมกันได้จริงในโปรแกรมเดียว
- Kong และ Traefik คือทางเลือกระดับโปรดักชันเมื่อระบบใหญ่และซับซ้อนเกินกว่าที่ gateway เขียนเองจะดูแลไหว โดยเฉพาะเรื่อง auto-discovery และ plugin ecosystem

## แบบฝึกหัดท้ายบท

1. รันตัวอย่าง Consul ในหัวข้อ 5 ด้วยตัวเอง แล้วลองเพิ่ม instance ตัวที่สามของ `order-service` เข้าไประหว่างโปรแกรมทำงาน (เขียนโปรแกรมแยกต่างหากเพื่อ register instance เพิ่ม) สังเกตว่า `client.Health().Service(...)` เห็น instance ใหม่ทันทีหรือไม่
2. เพิ่ม backend service ตัวที่สาม (เช่น `payment-service`) เข้าไปในตัวอย่าง API Gateway ในหัวข้อ 9 พร้อม path prefix `/payments/`
3. แก้ `rateLimiter` ในตัวอย่างให้จำกัดตาม `Authorization` token แทนที่จะจำกัดตาม `RemoteAddr` — เพราะเหตุใดการจำกัดตาม token (ระบุตัวผู้ใช้จริง) มักแม่นยำกว่าการจำกัดตาม IP ในระบบจริงที่มี proxy/NAT อยู่หน้า client
4. เขียน middleware เพิ่มเติมที่ inject header `X-Request-ID` แบบสุ่มเข้าไปในทุก request ก่อนส่งต่อไปยัง backend (ใช้เพื่อ trace request ข้าม service — เชื่อมโยงกับแนวคิด observability ที่จะเจอใน Part 099)
5. อธิบายด้วยคำพูดของตัวเอง: ทำไม client-side discovery ถึงต้องการ logic เพิ่มเติมในทุก client ในขณะที่ server-side discovery ไม่ต้องการ ยกตัวอย่างสถานการณ์ที่ client-side discovery อาจให้ผลลัพธ์ดีกว่า
6. (ขั้นสูง) ค้นคว้าเพิ่มเติมเกี่ยวกับ Traefik's Kubernetes/Docker provider แล้วอธิบายว่ากลไก "auto-discovery" ของมันทำงานต่างจากการต้องเขียน routing rule เองใน `buildGateway` ของหัวข้อ 9 อย่างไร

---

**ต่อไป**: [Part 094 — Circuit Breaker และ Retry Pattern](./094-circuit-breaker-and-retry.md)
