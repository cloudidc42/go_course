# Part 097: Kubernetes เบื้องต้นสำหรับ Go Developer

> ภาคที่ 9: DevOps และ Deployment — ตอนที่ 3 จาก 5 (Part 95–99)

## สารบัญของบทนี้

1. ทำไม Go Developer (ไม่ใช่แค่ Ops) ต้องเข้าใจ Kubernetes
2. แนวคิดหลัก: Pod, Deployment, Service คืออะไร เทียบกับสิ่งที่รู้จักแล้วจาก Docker Compose
3. จาก Docker Image (Part 095) สู่ Pod: Go binary กลายเป็น Kubernetes workload ได้อย่างไร
4. เขียน Deployment YAML แรกสำหรับ Go API
5. Service: เปิดทางให้ traffic เข้าถึง Pod แบบไม่ต้องรู้ IP
6. Readiness Probe vs Liveness Probe และทำไม Go ได้เปรียบเรื่อง startup เร็ว
7. ConfigMap และ Secret: ย้ายจากไฟล์ `.env` (Part 096) มาสู่ Kubernetes
8. รวมทุกอย่างเป็น manifest ชุดเดียวของบทนี้
9. Horizontal Pod Autoscaler (HPA): scale อัตโนมัติตามโหลด
10. `kubectl` เบื้องต้นสำหรับ debug แอปที่รันอยู่: `get`, `logs`, `exec`, `describe`
11. ทดลองสร้าง cluster จริงในแซนด์บ็อกซ์นี้ด้วย `kind` — และสิ่งที่เกิดขึ้นจริง
12. ตรวจสอบความถูกต้องของ YAML โดยไม่มี cluster จริง: `kubeconform`
13. ขอบเขตของบทนี้: ระดับ App Developer ไม่ใช่หลักสูตร Ops/SRE เต็มรูปแบบ
14. หมายเหตุความซื่อสัตย์เรื่องการรันจริงในบทนี้
15. สรุปสิ่งที่ได้เรียนในบทนี้
16. แบบฝึกหัดท้ายบท

---

## 1. ทำไม Go Developer (ไม่ใช่แค่ Ops) ต้องเข้าใจ Kubernetes

**Part 096** จบด้วย Docker Compose ที่รัน API + PostgreSQL + Redis ร่วมกันได้บนเครื่องเดียว ซึ่งเพียงพอสำหรับพัฒนาในเครื่อง (local development) และระบบขนาดเล็กมาก แต่เมื่อระบบต้องรันจริงใน production ที่ต้องการ **หลายเครื่อง (node) พร้อมกัน**, **restart อัตโนมัติเมื่อ container ตาย**, **scale จำนวน instance ขึ้นลงตามโหลด**, และ **rolling update แบบไม่มี downtime** — Docker Compose เพียงอย่างเดียวไม่เพียงพออีกต่อไป เพราะมันถูกออกแบบมาให้รันบน **เครื่องเดียว** เท่านั้น

**Kubernetes** (มักย่อว่า **K8s** — นับตัวอักษร "ubernete" ระหว่าง K กับ s ได้ 8 ตัวพอดี) คือระบบ **container orchestration** ที่แก้ปัญหาเหล่านี้โดยจัดการ container ให้รันข้าม**หลายเครื่อง**พร้อมกันเป็นกลุ่มเดียว (เรียกกลุ่มนี้ว่า **cluster**) ตามที่ **Part 001** เคยกล่าวถึงไว้ว่า Kubernetes เขียนด้วย Go ทั้งหมดเช่นเดียวกับ Docker

คำถามที่มือใหม่มักถามคือ **"นี่มันงานของทีม Ops/Platform ไม่ใช่หรือ ทำไม Go Developer ต้องรู้ด้วย?"** คำตอบคือในทีมพัฒนาสมัยใหม่ (โดยเฉพาะที่ทำ microservices ตามแนวทาง **Part 088**) เส้นแบ่งระหว่าง "คนเขียนโค้ด" กับ "คนดูแลระบบ" เลือนลงมาก แนวคิด **DevOps** และ **"you build it, you run it"** ทำให้นักพัฒนาต้อง:

- อ่านและแก้ YAML ของ Deployment ตัวเองได้ เมื่อต้องเพิ่ม environment variable หรือปรับ resource limit
- ใช้ `kubectl logs`/`kubectl exec` เพื่อ debug แอปตัวเองตอนมันมีปัญหาบน production ได้ด้วยตัวเอง โดยไม่ต้องรอทีม Ops ทุกครั้ง
- เข้าใจว่าทำไมแอปที่ทำงานปกติในเครื่องตัวเอง ถึง crash-loop บน Kubernetes (มักเกี่ยวกับ probe หรือ resource limit ที่ตั้งไม่ถูกต้อง — หัวข้อ 6 และ 8)
- ออกแบบแอปให้ "เป็นมิตรกับ Kubernetes" ตั้งแต่ต้น (graceful shutdown, health check endpoint, stateless) ซึ่งเป็นสิ่งที่เราวางรากฐานไว้แล้วตั้งแต่ **Part 095**

บทนี้เขียนขึ้นในระดับ **"App Developer ที่ต้อง deploy และ debug แอปของตัวเองบน Kubernetes ได้"** ไม่ใช่ระดับ **"วิศวกรที่ต้องติดตั้งและดูแล cluster เอง"** — ความแตกต่างนี้สำคัญมากและจะกล่าวย้ำอีกครั้งในหัวข้อ 13

---

## 2. แนวคิดหลัก: Pod, Deployment, Service คืออะไร เทียบกับสิ่งที่รู้จักแล้วจาก Docker Compose

Kubernetes มีคำศัพท์เฉพาะทางจำนวนมาก แต่แนวคิดหลักที่ต้องเข้าใจก่อนสำหรับ Go Developer มีแค่ 3 อย่าง เทียบกับสิ่งที่เรียนไปแล้วใน **Part 096**:

| แนวคิด Kubernetes | เทียบเท่ากับอะไรใน Docker Compose | อธิบายสั้นๆ |
|---|---|---|
| **Pod** | ใกล้เคียงกับ 1 container ที่รันจาก 1 service ใน compose | หน่วยที่เล็กที่สุดที่ Kubernetes จัดการได้ ปกติมี 1 container ต่อ 1 Pod (มีได้หลาย container ต่อ Pod ในกรณีพิเศษ เช่น sidecar) |
| **Deployment** | คล้ายการประกาศ `services.api` ใน `docker-compose.yml` พร้อม "สัญญา" ว่าต้องมีกี่ instance ตลอดเวลา | บอก Kubernetes ว่า "ต้องการ Pod แบบนี้กี่ตัว รันจาก image ไหน" แล้ว Kubernetes จะคอยดูแลให้จำนวน Pod ที่รันอยู่จริงตรงกับที่ประกาศไว้เสมอ (ถ้า Pod ตายไป จะสร้างใหม่ให้ทันทีโดยอัตโนมัติ) |
| **Service** | คล้ายกับที่ service คุยกันด้วยชื่อผ่าน internal DNS ใน compose | ให้ **ชื่อคงที่** และ **IP คงที่** สำหรับเข้าถึงกลุ่ม Pod หนึ่งกลุ่ม แม้ Pod แต่ละตัวจะถูกสร้างใหม่/ลบทิ้งตลอดเวลา (Pod มี IP ที่เปลี่ยนได้เสมอ แต่ Service ไม่เปลี่ยน) |

ความแตกต่างที่สำคัญที่สุดจาก Docker Compose คือ **Kubernetes ไม่ได้ "รัน container" ตรงๆ ตามที่เราสั่ง แต่เราประกาศ "สถานะที่ต้องการ" (desired state) แล้วปล่อยให้ Kubernetes คอยตรวจสอบและปรับระบบจริงให้ตรงกับที่ประกาศไว้ตลอดเวลา** แนวคิดนี้เรียกว่า **reconciliation loop** — ถ้า Pod ตัวหนึ่งใน Deployment ตายไปเพราะเครื่อง (node) ที่มันรันอยู่ล่ม Kubernetes จะสร้าง Pod ใหม่ทดแทนบนเครื่องอื่นให้อัตโนมัติ โดยที่เราไม่ต้องสั่งอะไรเพิ่มเลย ต่างจาก `docker compose restart` ที่ต้องมีคนหรือสคริปต์สั่งเอง

---

## 3. จาก Docker Image (Part 095) สู่ Pod: Go binary กลายเป็น Kubernetes workload ได้อย่างไร

จุดที่สำคัญที่สุดที่ต้องเข้าใจ: **Kubernetes ไม่ได้แข่งกับ Docker แต่ทำงานอยู่ "เหนือ" Docker (หรือ container runtime อื่นที่เข้ากันได้)** — สิ่งที่เราเรียนใน **Part 095** เรื่อง multi-stage build, `CGO_ENABLED=0`, `scratch`/`distroless`, และ `HEALTHCHECK` **ยังคงถูกต้องและจำเป็นทั้งหมดเหมือนเดิมทุกประการ** เพียงแต่แทนที่จะสั่ง `docker run` ตรงๆ เราจะ**อ้างอิงชื่อ image เดียวกันนั้น**ในไฟล์ YAML ของ Kubernetes แทน

พูดให้ชัดเป็นขั้นตอน:

1. `docker build -t ghcr.io/example/goapi:v1.0.0 .` — build image ด้วย Dockerfile แบบ multi-stage ที่เรียนใน **Part 095** เหมือนเดิมทุกประการ
2. `docker push ghcr.io/example/goapi:v1.0.0` — ส่ง image ขึ้น **container registry** (ที่นี่ใช้ GitHub Container Registry `ghcr.io` เป็นตัวอย่าง เพราะ **Part 098** จะสร้าง image ขึ้น registry นี้อัตโนมัติผ่าน CI/CD) เพราะ Kubernetes cluster (โดยเฉพาะ production ที่รันบนหลายเครื่อง) **ไม่มี image อยู่ในเครื่องอยู่แล้วแบบตอนรัน `docker build` ในเครื่องเรา** ต้อง pull จาก registry กลางเสมอ
3. ไฟล์ Deployment YAML (หัวข้อ 4) ระบุ `image: ghcr.io/example/goapi:v1.0.0` — Kubernetes จะไปสั่งให้ container runtime บนแต่ละเครื่อง `pull` image นี้มารันเป็น container ภายใน Pod ให้เอง

ข้อดีของ Go ที่เคยพูดถึงใน **Part 095** (binary เล็ก, static, ไม่มี dependency) ยิ่งทวีความสำคัญขึ้นไปอีกใน Kubernetes เพราะ:

- **Image เล็ก = pull เร็ว = Pod ใหม่พร้อมทำงานเร็ว** — สำคัญมากเวลา Kubernetes ต้อง scale out หรือ reschedule Pod ไปเครื่องใหม่กะทันหัน (เช่นตอน node ล่ม) ยิ่ง pull เสร็จเร็วเท่าไร ระบบยิ่งกลับมาเสถียรเร็วเท่านั้น
- **Go binary เริ่มทำงานเร็วมาก (ไม่ต้อง JIT warm-up แบบ JVM หรือ interpret แบบ Python/Node.js)** — ข้อดีนี้เชื่อมโยงตรงกับหัวขัดที่ 6 เรื่อง readiness/liveness probe

---

## 4. เขียน Deployment YAML แรกสำหรับ Go API

มาเขียน Deployment ตัวแรกสำหรับ API ตัวเดียวกับที่ใช้ตลอด **Part 095-096** (Task API ที่ต่อ PostgreSQL ผ่าน `pgxpool` และ Redis ผ่าน `go-redis`):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: goapi
  labels:
    app: goapi
spec:
  replicas: 3                  # ต้องการ Pod ที่รันอยู่พร้อมกัน 3 ตัวเสมอ
  selector:
    matchLabels:
      app: goapi                # Deployment นี้ดูแล Pod ที่มี label นี้ตรงกันเท่านั้น
  template:                     # แม่แบบสำหรับสร้าง Pod แต่ละตัว
    metadata:
      labels:
        app: goapi               # label ต้องตรงกับ selector ด้านบนเสมอ ไม่งั้น Deployment จะหา Pod ตัวเองไม่เจอ
    spec:
      containers:
        - name: goapi
          image: ghcr.io/example/goapi:v1.0.0
          ports:
            - name: http
              containerPort: 8080
```

จุดที่ต้องสังเกตให้ดี:

- **`spec.replicas: 3`** คือหัวใจของ Deployment — เราไม่ได้บอกว่า "รัน container 3 ตัว" ตรงๆ แต่บอกว่า **"ต้องการให้มี Pod ที่ตรงกับ template นี้อยู่จำนวน 3 ตัวเสมอ"** ถ้า Pod ตัวใดตายไป Kubernetes จะสร้างทดแทนให้อัตโนมัติเพื่อให้กลับมาครบ 3 เสมอ
- **`selector.matchLabels` ต้องตรงกับ `template.metadata.labels`** เป๊ะ — นี่คือกลไกที่ Deployment ใช้ระบุว่า Pod ตัวไหน "เป็นของมัน" ลืมจับคู่ label ผิดคือข้อผิดพลาดคลาสสิกที่สุดของมือใหม่ (อาการที่เจอคือสร้าง Deployment สำเร็จแต่ Pod ไม่ถูกสร้างเลย หรือ Deployment มองไม่เห็น Pod ที่มีอยู่)
- **`containerPort: 8080`** แค่ระบุว่า container ฟัง port อะไร (คล้าย `EXPOSE` ใน Dockerfile) — เป็นข้อมูลบอกให้อ่านง่ายเท่านั้น **ไม่ได้ทำให้ port เปิดออกสู่ภายนอก** ยังต้องมี Service มาช่วยเสมอ (หัวข้อถัดไป)
- **`name: http`** ที่ตั้งให้ port นี้ ทำให้ Service อ้างอิง port ด้วยชื่อแทนตัวเลขได้ (เห็นในหัวข้อ 5) — เปลี่ยนตัวเลข port ทีหลังได้โดยไม่ต้องแก้ที่ Service เลย

---

## 5. Service: เปิดทางให้ traffic เข้าถึง Pod แบบไม่ต้องรู้ IP

Pod แต่ละตัวใน Kubernetes มี **IP ของตัวเอง แต่ IP นี้เปลี่ยนได้ตลอดเวลา** — ทุกครั้งที่ Pod ถูกสร้างใหม่ (ไม่ว่าจะจาก crash, rolling update, หรือ scale) จะได้ IP ใหม่เสมอ ถ้า service อื่นหรือผู้ใช้ต้อง hardcode IP ของ Pod ไว้ ระบบจะพังทันทีที่ Pod ถูกสร้างใหม่ — **Service** คือคำตอบของปัญหานี้ เทียบได้กับ internal DNS ของ Docker Compose ที่เรียนใน **Part 096** แต่ทำงานคล้ายกับ **load balancer ในตัว** ด้วย:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: goapi
spec:
  selector:
    app: goapi          # เลือก Pod ทุกตัวที่มี label นี้ ไม่ว่าจะมีกี่ตัวหรือ IP อะไรก็ตาม
  ports:
    - name: http
      port: 80           # port ที่ Service เปิดให้เรียก
      targetPort: http    # ส่งต่อไปที่ port ชื่อ "http" ของ container (อ้างอิงชื่อจากหัวข้อ 4)
  type: ClusterIP         # ค่า default: เข้าถึงได้แค่จากภายใน cluster เดียวกันเท่านั้น
```

`Service` ทำงานคล้าย `Deployment` ตรงที่ใช้ **`selector`** จับคู่กับ **label** ของ Pod เช่นกัน (ในที่นี้คือ `app: goapi`) — เมื่อมี Pod ใหม่เกิดขึ้นหรือ Pod เก่าตายไป Service จะปรับรายชื่อ Pod ปลายทางที่จะส่ง traffic ไปให้อัตโนมัติทันที โดยที่**ชื่อ Service (`goapi`) และ ClusterIP ของมันไม่เปลี่ยนเลย** — เหมือนกับที่ **Part 096** service `api` คุยกับ service `db` ผ่านชื่อ `db` ได้ตรงๆ โดยไม่ต้องรู้ IP ของ container ที่รัน PostgreSQL อยู่จริง แนวคิดเดียวกันนี้ก็ใช้ได้ใน Kubernetes เพียงแต่เปลี่ยนจาก "internal DNS ของ compose" เป็น "internal DNS ของ Kubernetes" (`goapi.default.svc.cluster.local` เป็นชื่อเต็ม หรือเรียกสั้นๆ แค่ `goapi` ก็พอถ้าอยู่ namespace เดียวกัน)

### ประเภทของ Service ที่ควรรู้จัก

| type | ใช้เมื่อไร |
|---|---|
| `ClusterIP` (default) | เข้าถึงได้เฉพาะจากภายใน cluster — เหมาะกับ service ภายใน เช่น database, service ที่ service อื่นเรียกกันเอง |
| `NodePort` | เปิด port หนึ่งบนทุกเครื่อง (node) ของ cluster ให้เรียกจากภายนอกได้ตรงๆ — ใช้ทดสอบง่ายๆ แต่ไม่เหมาะกับ production จริง |
| `LoadBalancer` | ขอ load balancer จริงจาก cloud provider (AWS ELB, GCP Load Balancer ฯลฯ) มาชี้เข้า Service นี้ — วิธีมาตรฐานที่ใช้เปิด service ให้โลกภายนอกเข้าถึงได้บน cloud |

บทนี้เน้น `ClusterIP` เป็นหลักเพราะ API ตัวอย่างของเราสมมติว่าอยู่หลัง **Ingress** หรือ **API Gateway** อีกชั้นหนึ่งเสมอ (แนวคิดเดียวกับที่เรียนไปใน **Part 093** เรื่อง Service Discovery และ API Gateway) — การจัดการ Ingress แบบเจาะลึกเป็นเรื่องระดับ Ops ที่นอกเหนือขอบเขตของบทนี้ (ดูหัวข้อ 13)

---

## 6. Readiness Probe vs Liveness Probe และทำไม Go ได้เปรียบเรื่อง startup เร็ว

ใน **Part 095** เราสร้าง endpoint `/healthz` ไว้แล้วใช้กับ Docker `HEALTHCHECK` — Kubernetes มีแนวคิดคล้ายกันแต่ **แยกออกเป็น 2 probe ที่ทำหน้าที่ต่างกันชัดเจน** ซึ่งเป็นจุดที่มือใหม่สับสนบ่อยที่สุด:

| Probe | คำถามที่ตอบ | ถ้า "ไม่ผ่าน" Kubernetes ทำอะไร |
|---|---|---|
| **Readiness Probe** | "Pod นี้**พร้อมรับ traffic ตอนนี้**หรือยัง?" | ถอด Pod ออกจากรายชื่อปลายทางของ Service **ชั่วคราว** (ไม่ส่ง traffic มาให้) แต่**ไม่ restart container** — รอจนกว่าจะผ่านอีกครั้งแล้วค่อยเพิ่มกลับเข้าไป |
| **Liveness Probe** | "process ข้างใน container นี้**ยังทำงานปกติอยู่**หรือไม่?" | **restart container ทันที** (เหมือนกดปุ่ม restart ใหม่ทั้ง process) เพราะถือว่า process นี้พังไปแล้วไม่มีทางฟื้นเองได้ |

ตัวอย่างสถานการณ์ที่ทำให้เห็นความแตกต่างชัดที่สุด: แอปกำลัง**เชื่อมต่อ PostgreSQL อยู่ชั่วคราว** (เช่น database กำลัง failover ตามที่เรียนใน **Part 096** หัวข้อ 8) — ถ้าใช้ **readiness probe** เช็คว่าต่อ database ได้จริง Kubernetes จะแค่หยุดส่ง traffic มาให้ Pod นี้ชั่วคราว โดย**ไม่ restart container** เพราะ process หลักยังทำงานปกติดี รอ database กลับมาก็พร้อมรับ traffic ต่อได้ทันที — แต่ถ้าเผลอใช้ **liveness probe** เช็คแบบเดียวกัน (เช็คว่าต่อ database ได้) Kubernetes จะเข้าใจผิดว่า Pod "ตาย" แล้ว restart container ไปเรื่อยๆ ทั้งที่ปัญหาจริงอยู่ที่ database ไม่ใช่ตัวแอปเลย — เกิดเป็นวังวน **crash-loop** ที่ไม่ช่วยแก้ปัญหาอะไรเลย

เขียนเป็น YAML เพิ่มเข้าไปใน Deployment จากหัวข้อ 4:

```yaml
          readinessProbe:
            httpGet:
              path: /healthz
              port: http
            initialDelaySeconds: 1
            periodSeconds: 5
            timeoutSeconds: 2
            failureThreshold: 3
          livenessProbe:
            httpGet:
              path: /healthz
              port: http
            initialDelaySeconds: 3
            periodSeconds: 10
            timeoutSeconds: 2
            failureThreshold: 3
```

> **ข้อควรระวังในทางปฏิบัติ**: หลายทีมใช้ endpoint `/healthz` เดียวกันกับทั้งสอง probe อย่างที่แสดงไว้ข้างบน (เหมือน `HEALTHCHECK` ตัวเดียวของ Docker ใน **Part 095**) ซึ่งใช้ได้กับแอปง่ายๆ แต่แนวทางที่ดีกว่าสำหรับระบบที่ซับซ้อนขึ้นคือ**แยก endpoint สองตัว**: `/healthz` (liveness) ตอบ `200 OK` ตราบใดที่ process ยังรันอยู่โดยไม่เช็ค dependency ภายนอกเลย ส่วน `/readyz` (readiness) เช็คว่าต่อ database/cache ได้จริงก่อนตอบ `200 OK` — แยกความรับผิดชอบให้ตรงกับความหมายของ probe แต่ละตัวอย่างชัดเจน

### ทำไม Go ได้เปรียบเรื่องนี้เป็นพิเศษ

`initialDelaySeconds` คือเวลาที่ Kubernetes **รอก่อน** จะเริ่มเช็ค probe ครั้งแรกหลัง container เริ่มทำงาน — ค่านี้ต้องตั้งให้นานพอที่แอปจะ "warm up" เสร็จก่อน ไม่งั้น probe จะเช็คตอนแอปยังไม่พร้อมแล้วเข้าใจผิดว่าล้มเหลว ภาษาที่ต้องมี JVM warm-up (Java) หรือต้อง interpret code ทุกครั้ง (Python) มักต้องตั้งค่า `initialDelaySeconds` สูงถึงหลักสิบวินาที แต่ **Go binary ที่ compile ไว้ล่วงหน้าเป็น native machine code (ตามที่เรียนใน Part 001 และ Part 095) เริ่มทำงานได้ในระดับมิลลิวินาทีถึงหลักร้อยมิลลิวินาทีเท่านั้น** ทำให้ตั้ง `initialDelaySeconds: 1` แบบในตัวอย่างได้อย่างมั่นใจ — Pod ของแอป Go จึง **"พร้อมรับ traffic" เร็วกว่าภาษาอื่นอย่างเห็นได้ชัด** ซึ่งมีผลจริงต่อความเร็วในการ scale out และ rolling update (หัวข้อ 9)

---

## 7. ConfigMap และ Secret: ย้ายจากไฟล์ `.env` (Part 096) มาสู่ Kubernetes

**Part 096** ใช้ไฟล์ `.env` ให้ Docker Compose อ่านค่าอย่าง `POSTGRES_USER`, `POSTGRES_PASSWORD` แล้วส่งเข้า container ผ่าน `environment:` — Kubernetes มีกลไกคล้ายกันแต่**แยกเป็นสองประเภทตามความอ่อนไหวของข้อมูล**:

- **ConfigMap** — เก็บค่า config ที่**ไม่ลับ** เช่น port, log level, feature flag, hostname ของ service ภายใน
- **Secret** — เก็บค่าที่**อ่อนไหว** เช่น password, API key, connection string ที่มี credential ฝังอยู่

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: goapi-config
data:
  PORT: "8080"
  REDIS_ADDR: "cache:6379"
  LOG_LEVEL: "info"
---
apiVersion: v1
kind: Secret
metadata:
  name: goapi-secret
type: Opaque
stringData:
  DATABASE_URL: "postgres://appuser:apppass@db:5432/appdb?sslmode=disable"
```

แล้วดึงทั้งสองเข้าไปเป็น environment variable ของ container ด้วย `envFrom`:

```yaml
          envFrom:
            - configMapRef:
                name: goapi-config
            - secretRef:
                name: goapi-secret
```

โค้ด Go **ไม่ต้องแก้อะไรเลยแม้แต่บรรทัดเดียว** เพราะ `os.Getenv("DATABASE_URL")` และ `os.Getenv("REDIS_ADDR")` ที่เขียนไว้ตั้งแต่ **Part 096** อ่านค่าจาก environment variable อยู่แล้ว — Kubernetes แค่เป็น**อีกกลไกหนึ่ง**ที่ทำหน้าที่ "ใส่ environment variable เข้า container" เหมือนที่ `docker-compose.yml` เคยทำ นี่คือประโยชน์โดยตรงของการยึดหลัก [12-Factor App](https://12factor.net/) (อ่านค่า config ผ่าน environment variable) ที่พูดถึงไว้ตั้งแต่ **Part 095**: **image เดียวกัน โค้ดเดียวกัน รันได้ทั้งบนเครื่อง, ใน Docker Compose, และบน Kubernetes โดยไม่ต้องแก้โค้ดหรือ build ใหม่เลยสักครั้ง**

### ความจริงที่ต้องรู้เรื่อง Secret: "เข้ารหัส" ≠ "ปลอดภัยสมบูรณ์"

ชื่อ `Secret` ทำให้มือใหม่จำนวนมากเข้าใจผิดว่าข้อมูลถูก **เข้ารหัส (encrypted)** โดยอัตโนมัติ ความจริงคือ **ค่าใน Secret ถูกเก็บเป็น base64-encoded เท่านั้น ไม่ใช่ encrypted** — base64 เป็นแค่การเข้ารหัส**รูปแบบ** (encoding) ไม่ใช่การเข้ารหัส**เพื่อความปลอดภัย** (encryption) ใครก็ตามที่มีสิทธิ์อ่าน object `Secret` นี้ผ่าน `kubectl get secret goapi-secret -o yaml` สามารถ `base64 -d` ถอดค่ากลับมาเป็น plaintext ได้ทันที

ในทางปฏิบัติ production จริงจึงมักเสริมด้วย:

- **RBAC** (Role-Based Access Control) จำกัดว่าใคร/service account ไหนอ่าน Secret ตัวไหนได้บ้าง
- **Encryption at rest** เปิดใช้งานที่ระดับ etcd (ฐานข้อมูลภายในที่ Kubernetes ใช้เก็บ state ทั้งหมดรวมถึง Secret) ซึ่งเป็นการตั้งค่าระดับ cluster ที่ทีม Ops/Platform เป็นผู้ดูแล
- **External secret manager** เช่น HashiCorp Vault, AWS Secrets Manager แล้วดึงเข้า cluster แบบ dynamic แทนที่จะเก็บใน `Secret` object ตรงๆ

เรื่อง security ของ credential และ container จะกลับมาเจาะลึกอีกครั้งใน **Part 106**

---

## 8. รวมทุกอย่างเป็น manifest ชุดเดียวของบทนี้

รวม ConfigMap, Secret, Deployment (พร้อม probe และ resource limit), Service, และ HPA (หัวข้อถัดไป) เป็นไฟล์เดียว — ไฟล์นี้ผ่านการตรวจสอบด้วย `kubeconform` จริงตามที่แสดงในหัวข้อ 12:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: goapi-config
data:
  PORT: "8080"
  REDIS_ADDR: "cache:6379"
  LOG_LEVEL: "info"
---
apiVersion: v1
kind: Secret
metadata:
  name: goapi-secret
type: Opaque
stringData:
  DATABASE_URL: "postgres://appuser:apppass@db:5432/appdb?sslmode=disable"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: goapi
  labels:
    app: goapi
spec:
  replicas: 3
  selector:
    matchLabels:
      app: goapi
  template:
    metadata:
      labels:
        app: goapi
    spec:
      containers:
        - name: goapi
          image: ghcr.io/example/goapi:latest
          ports:
            - name: http
              containerPort: 8080
          envFrom:
            - configMapRef:
                name: goapi-config
            - secretRef:
                name: goapi-secret
          resources:
            requests:
              cpu: "100m"
              memory: "64Mi"
            limits:
              cpu: "500m"
              memory: "128Mi"
          readinessProbe:
            httpGet:
              path: /healthz
              port: http
            initialDelaySeconds: 1
            periodSeconds: 5
            timeoutSeconds: 2
            failureThreshold: 3
          livenessProbe:
            httpGet:
              path: /healthz
              port: http
            initialDelaySeconds: 3
            periodSeconds: 10
            timeoutSeconds: 2
            failureThreshold: 3
---
apiVersion: v1
kind: Service
metadata:
  name: goapi
spec:
  selector:
    app: goapi
  ports:
    - name: http
      port: 80
      targetPort: http
  type: ClusterIP
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: goapi
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: goapi
  minReplicas: 3
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

จุดที่เพิ่มเข้ามาใหม่ที่ยังไม่ได้อธิบาย: **`resources.requests`/`resources.limits`** — `requests` บอก Kubernetes ว่า Pod นี้ **"ต้องการ" ทรัพยากรขั้นต่ำเท่าไร** (ใช้ตอนตัดสินใจว่าจะวาง Pod ลงเครื่องไหน) ส่วน `limits` คือ **"เพดานสูงสุด"** ที่ห้ามเกิน (ถ้า memory เกิน limit container จะถูกฆ่าทิ้งด้วย `OOMKilled` ทันที) การตั้งค่าคู่นี้ให้เหมาะสมสำคัญมากต่อทั้งเสถียรภาพและค่าใช้จ่าย แต่เป็นหัวข้อที่ลึกพอจะแยกเขียนเป็นบททั้งบท จึงกล่าวถึงในระดับแนะนำให้รู้จักไว้ในบทนี้เท่านั้น

`YAML` หลายเอกสารในไฟล์เดียวกันแยกด้วย **`---`** (three dashes) ตามมาตรฐาน YAML — `kubectl apply -f full.yaml` จะสร้างหรืออัปเดต resource ทั้ง 5 ตัวในไฟล์นี้พร้อมกันในคำสั่งเดียว

---

## 9. Horizontal Pod Autoscaler (HPA): scale อัตโนมัติตามโหลด

`replicas: 3` ที่ตั้งไว้ตายตัวในหัวข้อ 4 เหมาะกับโหลดที่ค่อนข้างคงที่ แต่ระบบจริงมักมีช่วงโหลดสูง-ต่ำต่างกันมาก (เช่น ช่วงเวลาทำการเทียบกับกลางดึก หรือช่วง flash sale) การตั้ง replicas สูงตลอดเวลาเพื่อรองรับ peak โหลด**สิ้นเปลืองทรัพยากรมาก**ในช่วงเวลาที่โหลดต่ำ **Horizontal Pod Autoscaler (HPA)** แก้ปัญหานี้ด้วยการ **เพิ่ม/ลดจำนวน replicas อัตโนมัติ** ตาม metric ที่กำหนด (เช่น CPU utilization เฉลี่ยของทุก Pod):

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: goapi
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: goapi
  minReplicas: 3
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

อ่านเป็นภาษาคน: **"ดูแล Deployment ชื่อ `goapi` ให้มี Pod อย่างน้อย 3 ตัวเสมอ แต่ถ้า CPU เฉลี่ยของทุก Pod เกิน 70% ให้เพิ่ม Pod ขึ้นไปได้สูงสุดถึง 10 ตัว แล้วลดกลับลงมาเมื่อโหลดลดลง"**

**สิ่งสำคัญที่ต้องเข้าใจ (แนวคิด ไม่ใช่แค่ syntax)**:

- HPA ต้องการ **metrics-server** ทำงานอยู่ใน cluster ก่อนถึงจะอ่านค่า CPU/memory ของ Pod ได้ (เป็น component เสริมที่ทีม Ops ติดตั้งไว้ให้ระดับ cluster) ถ้าไม่มี metrics-server HPA จะสร้างได้แต่ไม่ทำงาน (ไม่มีข้อมูลให้ตัดสินใจ scale)
- Scale ขึ้นเร็ว แต่ **scale ลงจะช้ากว่าโดยตั้งใจ** (มี cooldown period ป้องกันการ scale ขึ้นๆ ลงๆ ถี่เกินไปจนระบบไม่นิ่ง เรียกว่า "flapping")
- **"App ที่ scale ได้ดี" ต้องเป็น stateless เท่านั้น** — Pod ตัวใหม่ที่ถูกสร้างขึ้นมาต้องทำงานได้ทันทีโดยไม่ต้องมี state เดิมติดตัวมา (ตรงกับที่ **Part 096** อธิบายไว้ว่า `api` ไม่มี volume เลยเพราะ state ทั้งหมดอยู่ที่ PostgreSQL/Redis) นี่คือเหตุผลสำคัญที่ต้องออกแบบ Go service ให้ stateless มาตั้งแต่ต้น ไม่เก็บข้อมูลสำคัญไว้ใน memory ของ process เอง
- Kubernetes รุ่นใหม่ยังรองรับ **custom metrics** (เช่น scale ตามความยาว queue หรือ requests per second จาก Prometheus) และ **Vertical Pod Autoscaler (VPA)** ที่ปรับ `resources.requests`/`limits` แทนที่จะปรับจำนวน replicas — ทั้งสองเรื่องนี้เป็นหัวข้อขั้นสูงกว่าที่ควรค้นคว้าต่อหลังเข้าใจ HPA พื้นฐานแล้ว

---

## 10. `kubectl` เบื้องต้นสำหรับ debug แอปที่รันอยู่: `get`, `logs`, `exec`, `describe`

`kubectl` คือ CLI หลักที่ใช้คุยกับ Kubernetes cluster ทุกชนิด (เทียบเท่ากับ `docker`/`docker compose` CLI ใน 2 บทก่อนหน้า) คำสั่งที่ Go Developer ใช้บ่อยที่สุดในการ debug แอปตัวเองมีไม่กี่คำสั่งเท่านั้น:

```bash
# ดู Pod ทั้งหมดที่กำลังรันอยู่ พร้อมสถานะ
kubectl get pods

# ดูรายละเอียดของ Pod ที่ระบุ พร้อม label เพิ่มเติม
kubectl get pods -l app=goapi -o wide

# ดู log ของ container ใน Pod (เทียบเท่า docker logs ใน Part 095)
kubectl logs goapi-7d4f8b9c9d-x2k9p

# ดู log แบบ follow (real-time) เหมือน docker compose logs -f จาก Part 096
kubectl logs -f goapi-7d4f8b9c9d-x2k9p

# ดู log ของ container ก่อนหน้า (กรณี container เพิ่ง restart ไป อยากรู้ว่า crash เพราะอะไร)
kubectl logs goapi-7d4f8b9c9d-x2k9p --previous

# เข้า shell ข้างในเพื่อ debug (เทียบเท่า docker exec -it <container> sh)
kubectl exec -it goapi-7d4f8b9c9d-x2k9p -- sh

# ดูรายละเอียดเชิงลึกที่สุด: event ล่าสุด, สาเหตุที่ restart, ผลของ probe แต่ละครั้ง
kubectl describe pod goapi-7d4f8b9c9d-x2k9p

# ดูสถานะของ Deployment โดยรวม (จำนวน replicas ที่ต้องการ vs พร้อมจริง)
kubectl get deployment goapi

# ดูสถานะ Service และ ClusterIP ที่ได้รับ
kubectl get service goapi

# ดู event ทั้ง namespace เรียงตามเวลา (มีประโยชน์มากตอนสงสัยว่า Pod ทำไมไม่ยอมขึ้น)
kubectl get events --sort-by=.lastTimestamp
```

**หมายเหตุเรื่องชื่อ Pod**: สังเกตว่าชื่อ Pod ไม่ใช่ `goapi` ตรงๆ แต่เป็น `goapi-7d4f8b9c9d-x2k9p` — Deployment ไม่ได้สร้าง Pod โดยตรง แต่สร้างผ่านชั้นกลางที่ชื่อ **ReplicaSet** อีกที (ReplicaSet คือสิ่งที่ทำหน้าที่ "รักษาจำนวน Pod ให้ตรงตามที่ประกาศ" จริงๆ ส่วน Deployment ทำหน้าที่จัดการ ReplicaSet อีกชั้นหนึ่งเพื่อรองรับ rolling update) ทำให้ชื่อ Pod มีส่วนต่อท้ายแบบสุ่มเสมอ ในทางปฏิบัติจึงมักใช้ `kubectl get pods -l app=goapi` เพื่อหาชื่อ Pod ปัจจุบันก่อน แล้วค่อยคัดลอกชื่อไปใช้กับคำสั่งอื่น หรือใช้ `kubectl logs -l app=goapi` ที่รับ label selector ได้โดยตรงในบางกรณี

### `describe` คือเพื่อนที่ดีที่สุดตอน Pod มีปัญหา

เมื่อ Pod ไม่ยอม "Running" หรือ probe ไม่ผ่านตามที่ตั้งค่าไว้ในหัวข้อ 6 คำสั่งแรกที่ควรรันเสมอคือ `kubectl describe pod <ชื่อ>` — ผลลัพธ์ส่วนที่มีค่ามากที่สุดมักอยู่ท้ายผลลัพธ์ ในหัวข้อ **`Events:`** ที่แสดง timeline ของทุกอย่างที่เกิดกับ Pod นี้ เช่น:

```
Events:
  Type     Reason     Age                From               Message
  ----     ------     ----               ----               -------
  Normal   Scheduled  2m                 default-scheduler  Successfully assigned default/goapi-7d4f8b9c9d-x2k9p to node-1
  Normal   Pulling    2m                 kubelet            Pulling image "ghcr.io/example/goapi:latest"
  Normal   Pulled     1m58s              kubelet            Successfully pulled image in 1.2s
  Normal   Created    1m58s              kubelet            Created container goapi
  Normal   Started    1m58s              kubelet            Started container goapi
  Warning  Unhealthy  30s (x3 over 50s)  kubelet            Readiness probe failed: connection refused
```

จากตัวอย่างข้างบน (รูปแบบมาตรฐานที่ `kubectl describe` แสดงเสมอ) อ่านออกได้ทันทีว่า image pull สำเร็จ, container เริ่มทำงานได้ปกติ แต่ **readiness probe ล้มเหลวเพราะ connection refused** — บอกใบ้ชัดเจนว่าต้องไปดูว่าทำไม `/healthz` endpoint ไม่ตอบ (เช่น แอปยัง bind port ไม่เสร็จ, หรือ `containerPort` ในหัวข้อ 4 ตั้งเลขไม่ตรงกับที่แอป listen จริง)

---

## 11. ทดลองสร้าง cluster จริงในแซนด์บ็อกซ์นี้ด้วย `kind` — และสิ่งที่เกิดขึ้นจริง

ก่อนเขียนบทนี้ ได้ทดลองติดตั้งเครื่องมือจริงในสภาพแวดล้อมแซนด์บ็อกซ์ที่ใช้เขียนหลักสูตรนี้เพื่อดูว่าจะสร้าง Kubernetes cluster จริงมาทดสอบคำสั่งทั้งหมดในบทนี้แบบ end-to-end ได้หรือไม่ (เหมือนที่ **Part 095-096** รัน Docker/Docker Compose จริงได้ทั้งหมด) ขั้นตอนที่ทำจริงมีดังนี้:

1. ติดตั้ง `kubectl` (v1.37.1) และ `kind` (v0.30.0, "Kubernetes IN Docker" — เครื่องมือมาตรฐานสำหรับสร้าง cluster ทดสอบบนเครื่องเดียวโดยรัน node ของ Kubernetes เป็น container ผ่าน Docker) — ทั้งสองติดตั้งสำเร็จ เพราะแซนด์บ็อกซ์นี้มี Docker Engine ทำงานอยู่จริงอยู่แล้ว (ตามที่ **Part 095** ยืนยันไว้)
2. รันคำสั่งสร้าง cluster จริง:

```bash
kind create cluster --name go-course-demo
```

**ผลลัพธ์จริงที่ได้**:

```
Creating cluster "go-course-demo" ...
 ✓ Ensuring node image (kindest/node:v1.34.0) 🖼
 ✓ Preparing nodes 📦
 ✓ Writing configuration 📜
 • Starting control-plane 🕹️  ...
```

กระบวนการ**ค้างอยู่ที่ขั้นตอน "Starting control-plane" นานเกินกว่าปกติ** (ปกติขั้นตอนนี้ใช้เวลาไม่กี่นาที) เมื่อเข้าไปตรวจสอบ log ข้างในตัว node container ที่ `kind` สร้างขึ้น (`docker logs go-course-demo-control-plane` และ `docker exec ... systemctl status kubelet`) พบสาเหตุที่แท้จริงจาก log ของ `kubelet` โดยตรง:

```
kubelet[253]: E0926 ... "RunPodSandbox from runtime service failed" err="rpc error: code = Unknown desc =
  failed to start sandbox ...: failed to create containerd task: failed to create shim task:
  OCI runtime create failed: runc create failed: unable to start container process:
  can't get final child's PID from pipe: EOF"
```

**คำอธิบายที่ตรงไปตรงมา**: `kind` ทำงานโดยรัน "node" ของ Kubernetes เป็น container หนึ่งตัว (ผ่าน `dockerd` ของแซนด์บ็อกซ์นี้) แล้วภายใน container นั้นต้องรัน `containerd`+`runc` **อีกชั้นหนึ่ง** เพื่อสร้าง Pod ของ Kubernetes เอง (etcd, kube-apiserver, kube-scheduler ฯลฯ) — เท่ากับเป็นการรัน container **ซ้อนกันสามชั้น** (แซนด์บ็อกซ์ตัวนี้เอง → `dockerd` ของแซนด์บ็อกซ์ → `containerd`/`runc` ข้างใน node ของ `kind`) การซ้อนกันระดับนี้ต้องพึ่งความสามารถของ kernel เรื่อง namespace/cgroup ที่มักถูกจำกัดไว้ในสภาพแวดล้อมแบบ sandboxed container เพื่อความปลอดภัย (ป้องกันไม่ให้โค้ดที่รันอยู่ข้างในสร้าง container ใหม่ที่หลุดออกไปควบคุมเครื่องจริงได้) — error `runc create failed: unable to start container process: can't get final child's PID from pipe: EOF` เป็นอาการทั่วไปของข้อจำกัดนี้พอดี

หลังยืนยันแล้วว่า control-plane ไม่มีทาง Running สำเร็จได้ในสภาพแวดล้อมนี้ จึงล้าง cluster ทดสอบทิ้ง:

```bash
kind delete cluster --name go-course-demo
```

**สรุปตรงไปตรงมา**: **การรัน Kubernetes cluster จริง (แม้เป็นแค่ cluster ทดสอบขนาดเล็กผ่าน `kind`) ไม่สามารถทำได้ในสภาพแวดล้อมแซนด์บ็อกซ์ที่ใช้เขียนหลักสูตรนี้** เนื่องจากข้อจำกัดของ container-in-container ระดับลึกตามที่อธิบายไว้ข้างบน — นี่ตรงกับสิ่งที่คาดไว้ตั้งแต่ต้น (Kubernetes cluster ปกติต้องการ node จริงหรือ VM ที่มีสิทธิ์ระดับ kernel มากกว่า container ธรรมดา) ดังนั้น**คำสั่ง `kubectl` ทั้งหมดในหัวข้อ 10 ข้างต้น รวมถึงผลลัพธ์ตัวอย่างของ `kubectl describe` เป็นตัวอย่างรูปแบบผลลัพธ์มาตรฐานที่ `kubectl` แสดงจริงเมื่อรันกับ cluster ที่ใช้งานได้ปกติ (อ้างอิงจาก Kubernetes documentation อย่างเป็นทางการ) ไม่ใช่ผลลัพธ์ที่ capture จากการรันจริงในแซนด์บ็อกซ์นี้** — สิ่งที่ตรวจสอบได้จริงในสภาพแวดล้อมนี้คือ**ความถูกต้องของ syntax และ schema ของ YAML** ซึ่งอธิบายไว้ในหัวข้อถัดไป

---

## 12. ตรวจสอบความถูกต้องของ YAML โดยไม่มี cluster จริง: `kubeconform`

เมื่อไม่มี cluster จริงให้ทดสอบ คำถามคือ "แล้วจะรู้ได้อย่างไรว่า YAML ในบทนี้ถูกต้องจริง ไม่ใช่แค่เขียนขึ้นมาลอยๆ" — คำสั่ง `kubectl apply --dry-run=client` ที่หลายคนคุ้นเคยว่าใช้ตรวจ YAML แบบไม่กระทบ cluster จริงนั้น **ในทางปฏิบัติยังคง"ต้องการ" ต่อ API server อยู่ดี** เพื่อดึง OpenAPI schema และ REST mapping ของแต่ละชนิด resource มาใช้ตรวจสอบ ทดลองรันจริงในสภาพแวดล้อมนี้ (โดยไม่มี cluster ให้เชื่อมต่อเลย) ก็ยืนยันตรงตามนั้น:

```bash
$ kubectl apply --dry-run=client -f full.yaml
error: error validating "full.yaml": error validating data: failed to download openapi:
  Get "http://localhost:8080/openapi/v2?timeout=32s": dial tcp 127.0.0.1:8080: connect: connection refused;
  if you choose to ignore these errors, turn validation off with --validate=false

$ kubectl apply --dry-run=client --validate=false -f full.yaml
error: unable to recognize "full.yaml": Get "http://localhost:8080/api?timeout=32s":
  dial tcp 127.0.0.1:8080: connect: connection refused
```

แม้จะปิด `--validate` แล้วก็ยังต้องคุยกับ API server อยู่ดี (เพื่อรู้ว่า `Deployment`/`Service` ต้อง map ไปที่ REST endpoint ไหน) — สรุปคือ **`kubectl apply --dry-run=client` ใช้ตรวจ YAML แบบไม่มี cluster เลยไม่ได้จริงๆ** ทางออกที่ใช้จริงในบทนี้คือเครื่องมือ **`kubeconform`** ซึ่งเป็น schema validator ที่ตรวจ YAML เทียบกับ **JSON Schema ของ Kubernetes API ที่ฝังไว้ล่วงหน้า (หรือดาวน์โหลดจาก schema store)** โดย**ไม่ต้องเชื่อมต่อ cluster ใดๆ เลย** — ติดตั้งและรันจริงกับไฟล์ manifest เต็มของหัวข้อ 8:

```bash
kubeconform -strict -summary full.yaml
```

**ผลลัพธ์จริงจากการรันในแซนด์บ็อกซ์นี้**:

```
Summary: 5 resources found in 1 file - Valid: 5, Invalid: 0, Errors: 0, Skipped: 0
```

ยืนยันว่าทั้ง 5 resource (`ConfigMap`, `Secret`, `Deployment`, `Service`, `HorizontalPodAutoscaler`) ในไฟล์เดียวกันของหัวข้อ 8 **ถูกต้องตาม schema จริงของ Kubernetes API ทุกฟิลด์** เพื่อพิสูจน์ว่า `kubeconform` ตรวจจับข้อผิดพลาดได้จริง ไม่ใช่แค่ผ่านทุกอย่างเสมอ ได้ลองแก้ YAML ให้ผิดโดยตั้งใจ (เปลี่ยน field `replicas` เป็น string ที่ไม่ใช่ตัวเลข และพิมพ์ `ports` ผิดเป็น `portz`) แล้วรันซ้ำ:

```
$ kubeconform -strict -summary bad.yaml
bad.yaml - Deployment goapi is invalid: problem validating schema. Check JSON formatting:
  jsonschema: '/spec/template/spec/containers/0' does not validate with
  .../deployment-apps-v1.json#/properties/spec/properties/template/properties/spec/properties/containers/items/additionalProperties:
  additionalProperties 'portz' not allowed
Summary: 1 resource found in 1 file - Valid: 0, Invalid: 1, Errors: 0, Skipped: 0
```

`kubeconform` จับ field ที่พิมพ์ผิด (`portz` ที่ไม่มีอยู่ใน schema จริงของ `Deployment`) ได้ถูกต้องทันที ยืนยันว่าเครื่องมือนี้ตรวจสอบ schema จริงอย่างเข้มงวด ไม่ใช่แค่ตรวจว่าเป็น YAML ที่ parse ได้เฉยๆ — เสริมด้วยการ parse ไฟล์เดียวกันผ่าน PyYAML เพื่อยืนยันว่าโครงสร้าง multi-document YAML (คั่นด้วย `---`) ถูกต้องตรงตามที่ตั้งใจ:

```python
import yaml
with open("full.yaml") as f:
    docs = list(yaml.safe_load_all(f))
# ได้ 5 documents: ConfigMap goapi-config, Secret goapi-secret,
# Deployment goapi, Service goapi, HorizontalPodAutoscaler goapi
```

**สรุปแนวทางที่ใช้จริงในบทนี้**: ไม่มี Kubernetes cluster จริงให้ deploy ทดสอบในสภาพแวดล้อมนี้ (ตามที่หัวข้อ 11 อธิบายไว้) แต่ **ทุกไฟล์ YAML ที่แสดงในบทนี้ผ่านการตรวจสอบ schema จริงด้วย `kubeconform` และ parse ผ่าน PyYAML จริง** ซึ่งเป็นระดับการยืนยันที่สมเหตุสมผลที่สุดเท่าที่ทำได้โดยไม่มี cluster

---

## 13. ขอบเขตของบทนี้: ระดับ App Developer ไม่ใช่หลักสูตร Ops/SRE เต็มรูปแบบ

ต้องพูดให้ตรงไปตรงมาที่สุด: **Kubernetes เป็นระบบที่ใหญ่และซับซ้อนมาก** จนมีหนังสือหนาหลายร้อยหน้าและ certification เฉพาะทาง (เช่น CKA — Certified Kubernetes Administrator) เขียนขึ้นมาเพื่อสอนเรื่องนี้โดยเฉพาะ บทนี้ **ตั้งใจครอบคลุมเฉพาะสิ่งที่ Go Developer ทั่วไปต้องรู้เพื่อ deploy และ debug แอปของตัวเองได้** เท่านั้น หัวข้อสำคัญที่**ไม่ได้กล่าวถึงเลย**ในบทนี้ เพราะเป็นความรับผิดชอบของทีม Ops/Platform โดยทั่วไป ได้แก่:

- การติดตั้งและดูแล cluster เอง (control plane, etcd, networking layer อย่าง CNI)
- **Ingress Controller** และการตั้งค่า TLS/routing ระดับ domain สำหรับ traffic จากภายนอก
- **Namespace**, **RBAC**, **Network Policy** สำหรับแบ่งสิทธิ์และแยกความปลอดภัยระหว่างทีม/environment
- **StatefulSet** สำหรับ workload ที่มี state (เช่น database เอง) ต่างจาก Deployment ที่เหมาะกับ stateless app เท่านั้น
- **Helm** และเครื่องมือจัดการ manifest ขนาดใหญ่ (templating, packaging)
- **Service Mesh** (Istio, Linkerd) สำหรับ traffic management, mTLS ระหว่าง service ขั้นสูง
- การวางแผน capacity, cost optimization, และ multi-cluster/multi-region architecture ระดับองค์กร

ถ้าทำงานในทีมที่มีทีม Platform/SRE คอยดูแล cluster ให้อยู่แล้ว ความรู้จากบทนี้ (Deployment, Service, ConfigMap/Secret, probe, และการ debug ด้วย `kubectl`) มักเพียงพอสำหรับการทำงานประจำวันในฐานะ Go Developer ได้จริง — แต่ถ้าต้องรับผิดชอบดูแล cluster เองด้วย แนะนำให้ศึกษาต่อจาก [Kubernetes เอกสารทางการ](https://kubernetes.io/docs/home/) และพิจารณาสอบ certification อย่าง CKA/CKAD เพิ่มเติม

---

## 14. หมายเหตุความซื่อสัตย์เรื่องการรันจริงในบทนี้

สรุปให้ชัดเจนอีกครั้งว่าอะไรรันจริงและอะไรไม่ได้รันจริงในบทนี้:

- **รันจริงและยืนยันผลได้**: การติดตั้ง `kubectl` v1.37.1 และ `kind` v0.30.0, ความพยายามสร้าง cluster จริงด้วย `kind create cluster` (และ log ข้อผิดพลาดจริงที่แสดงในหัวข้อ 11), การตรวจสอบ YAML ทั้งหมดในบทนี้ด้วย `kubeconform -strict` จริง (ทั้งกรณี valid และกรณีจงใจทำผิดเพื่อพิสูจน์ว่าเครื่องมือจับ error ได้จริง), และการ parse ไฟล์ YAML หลายเอกสารด้วย PyYAML จริง
- **ไม่ได้รันจริง (เอกสารอ้างอิงจาก Kubernetes documentation อย่างเป็นทางการ)**: คำสั่ง `kubectl get/logs/exec/describe` ในหัวข้อ 10 และรูปแบบผลลัพธ์ของมัน เพราะไม่มี cluster ที่ใช้งานได้จริงให้ทดสอบในสภาพแวดล้อมนี้ตามที่อธิบายไว้ในหัวข้อ 11 — รูปแบบผลลัพธ์ที่แสดงไว้ (ชื่อ Pod, โครงสร้าง event, ข้อความ error) เป็นรูปแบบมาตรฐานที่ `kubectl` แสดงจริงเมื่อใช้กับ cluster ปกติ ไม่ใช่ค่าที่แต่งขึ้นเอง แต่ก็ไม่ใช่ผลลัพธ์ที่ capture จากการรันจริงเช่นกัน

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- **Kubernetes** คือ container orchestration ที่ทำงาน "เหนือ" Docker เพื่อจัดการ container ข้ามหลายเครื่องพร้อมกัน แก้ปัญหาที่ Docker Compose (Part 096) ทำไม่ได้เพราะถูกออกแบบมาให้รันเครื่องเดียวเท่านั้น
- **Pod** คือหน่วยเล็กที่สุด, **Deployment** ประกาศ "สถานะที่ต้องการ" (จำนวน replicas + template ของ Pod) แล้วปล่อยให้ Kubernetes คอยดูแลให้ตรงเสมอ, **Service** ให้ชื่อ/IP คงที่สำหรับเข้าถึงกลุ่ม Pod ที่ IP เปลี่ยนได้ตลอดเวลา
- Docker image เดียวกันจาก **Part 095** (multi-stage build, `CGO_ENABLED=0`, `scratch`/`distroless`) ใช้กับ Kubernetes ได้ตรงๆ ไม่ต้องแก้ไขอะไร เพียงแค่ push ขึ้น container registry แล้วอ้างอิงชื่อ image ใน Deployment YAML
- **Readiness probe** ควบคุมว่า Pod รับ traffic หรือไม่ (ไม่ restart), **Liveness probe** ควบคุมว่าต้อง restart container หรือไม่ — สับสนสองอย่างนี้คือสาเหตุคลาสสิกของ crash-loop ที่ไม่จำเป็น
- Go binary เริ่มทำงานเร็วมากเมื่อเทียบกับภาษาที่ต้อง warm-up (Java) หรือ interpret (Python) ทำให้ตั้งค่า `initialDelaySeconds` ของ probe ต่ำได้อย่างมั่นใจ — Pod ของ Go service พร้อมรับ traffic เร็วกว่าภาษาอื่นเห็นได้ชัด
- **ConfigMap** เก็บ config ไม่ลับ, **Secret** เก็บค่าอ่อนไหว (แต่เป็นแค่ base64 ไม่ใช่ encrypted) — ทั้งคู่ต่อเข้า container ผ่าน `envFrom` โดยโค้ด Go ที่อ่านค่าจาก `os.Getenv` (ตามที่เขียนไว้ตั้งแต่ Part 096) ไม่ต้องแก้ไขอะไรเลย
- **HPA (Horizontal Pod Autoscaler)** ปรับจำนวน replicas อัตโนมัติตาม metric เช่น CPU utilization — ใช้ได้ดีก็ต่อเมื่อแอปเป็น stateless เท่านั้น
- `kubectl get/logs/exec/describe` คือชุดคำสั่งหลักสำหรับ debug แอปที่รันอยู่บน Kubernetes ในฐานะ App Developer
- ทดลองสร้าง Kubernetes cluster จริงด้วย `kind` ในแซนด์บ็อกซ์นี้แล้วไม่สำเร็จ เนื่องจาก container-in-container สามชั้นเกินกว่าที่ sandbox อนุญาต — ยืนยันด้วย log จริงของ `kubelet`/`runc` จึงใช้ `kubeconform` ตรวจสอบ schema ของ YAML แทน (ผ่านทั้งหมด 5/5 resource)
- บทนี้ครอบคลุมความรู้ระดับ App Developer เท่านั้น ไม่ใช่หลักสูตร Kubernetes Ops/SRE เต็มรูปแบบ

## แบบฝึกหัดท้ายบท

1. ติดตั้ง `kubectl` และ `kind` (หรือ `minikube`) บนเครื่องของตัวเอง (ที่ไม่มีข้อจำกัดแบบแซนด์บ็อกซ์ในบทนี้) แล้วสร้าง cluster ทดสอบจริง ใช้ manifest เต็มจากหัวข้อ 8 สั่ง `kubectl apply -f full.yaml` แล้วรันคำสั่งทั้งหมดในหัวข้อ 10 ดูผลลัพธ์จริงเทียบกับที่บทความอธิบายไว้
2. แก้ Deployment ในหัวข้อ 8 ให้ `selector.matchLabels` กับ `template.metadata.labels` ไม่ตรงกันโดยตั้งใจ (เช่นสะกดผิดคำเดียว) แล้วลอง `kubectl apply` ดูว่าเกิด error หรือพฤติกรรมแปลกอะไรขึ้น อธิบายว่าทำไม
3. เขียน `livenessProbe` และ `readinessProbe` แยก endpoint กันจริง (`/healthz` กับ `/readyz`) ในโค้ด Go จาก Part 096 โดยให้ `/readyz` เช็คการเชื่อมต่อ PostgreSQL/Redis จริงก่อนตอบ `200 OK` (ทบทวนโค้ด `handleHealthz` จาก Part 096) แล้วปรับ YAML ให้ probe แต่ละตัวเรียก endpoint ที่ถูกต้องของตัวเอง
4. ใช้ `kubeconform` ตรวจสอบ manifest ของแบบฝึกหัดข้อ 3 ที่แก้ไขแล้ว ยืนยันว่ายังผ่าน schema validation ทั้งหมด
5. ลองลด `resources.limits.memory` ในหัวข้อ 8 ให้ต่ำมากๆ (เช่น `16Mi`) แล้วสร้าง cluster จริงตามข้อ 1 มาทดสอบ สังเกตว่า Pod ถูก `OOMKilled` หรือไม่ ใช้ `kubectl describe pod` ดู event เพื่อยืนยันสาเหตุ
6. ค้นคว้าเพิ่มเติมเรื่อง **StatefulSet** — ทำไม PostgreSQL (ที่มี state ต้องเก็บถาวรตามที่เรียนใน Part 096) ถึงไม่เหมาะจะรันเป็น Deployment ธรรมดาบน Kubernetes production จริง และ StatefulSet แก้ปัญหาอะไรที่ Deployment แก้ไม่ได้

---

**ต่อไป**: [Part 098 — CI/CD Pipeline ด้วย GitHub Actions](./098-cicd-github-actions.md)
