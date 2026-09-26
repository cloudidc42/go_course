# Part 069: WebSocket ด้วย Go

> ภาคที่ 5: Web Development — ตอนที่ 14 จาก 15

## สารบัญของบทนี้

1. HTTP ธรรมดาไม่พอสำหรับอะไร: ปัญหาที่ WebSocket แก้
2. WebSocket ทำงานอย่างไร: Upgrade Handshake
3. ทำไม Go standard library ไม่มี WebSocket ให้
4. เลือก library: `gorilla/websocket` vs `nhooyr.io/websocket`
5. ติดตั้ง `gorilla/websocket`
6. Echo Server สมบูรณ์: Upgrade, Read/Write Loop, Ping/Pong Keepalive
7. อันตรายที่มองไม่เห็น: Concurrent Write ไม่ปลอดภัย
8. ทางแก้ที่ 1: ห่อ Connection ด้วย Mutex
9. ทางแก้ที่ 2: Single Writer Goroutine + Channel (รูปแบบมาตรฐานสำหรับ Chat/Broadcast)
10. Graceful Close
11. สรุปสิ่งที่ได้เรียนในบทนี้
12. แบบฝึกหัดท้ายบท

---

## 1. HTTP ธรรมดาไม่พอสำหรับอะไร: ปัญหาที่ WebSocket แก้

HTTP ที่เราใช้กันมาตลอด (Part 046-047, 056-068) มีรูปแบบการสื่อสารเป็น **request/response**: client ขอมา เซิร์ฟเวอร์ตอบไป จบการเชื่อมต่อ (หรือ keep-alive ไว้รอ request ถัดไป) แต่รูปแบบนี้มีข้อจำกัดสำคัญ: **เซิร์ฟเวอร์ไม่มีทาง "เริ่มพูด" ก่อนได้เอง** ต้องรอให้ client ถามก่อนเสมอ

ลองนึกถึงแอปแชท — เมื่อเพื่อนส่งข้อความมา เราต้องการให้ข้อความปรากฏบนหน้าจอ**ทันที** โดยไม่ต้องกดปุ่ม refresh หรือรอให้ browser ไปถามเซิร์ฟเวอร์ทุกๆ 2 วินาทีว่า "มีข้อความใหม่ไหม" (วิธีหลังนี้เรียกว่า **polling** ซึ่งทำงานได้แต่สิ้นเปลือง resource มาก และยังมีความหน่วงเสมอ)

**WebSocket** คือ protocol ที่แก้ปัญหานี้ตรงจุด: มันเปิด **connection เดียวที่คงอยู่ตลอด (persistent) และสื่อสารได้สองทาง (bidirectional)** — ทั้ง client และ server ส่งข้อความหากันได้ทุกเมื่อโดยไม่ต้องรอถามก่อน เหมาะกับงานที่ต้องการข้อมูล real-time:

- **แอปแชท** — ข้อความใหม่ต้องส่งถึงผู้รับทันที (เราจะสร้างโปรเจกต์นี้เต็มรูปแบบใน **Part 104: Real-time Chat Application**)
- **แจ้งเตือนสด (live notifications)** — เช่นแจ้งเตือนออเดอร์ใหม่, ราคาหุ้นเปลี่ยน
- **การทำงานร่วมกันแบบ real-time** — เช่น cursor ของผู้ใช้คนอื่นขยับบนเอกสารเดียวกัน (คล้าย Google Docs)
- **เกมออนไลน์** — ต้องส่งข้อมูลตำแหน่งผู้เล่นไปมาต่อเนื่องด้วยความหน่วงต่ำ

---

## 2. WebSocket ทำงานอย่างไร: Upgrade Handshake

จุดที่น่าสนใจคือ WebSocket **ไม่ได้เริ่มต้นจากศูนย์** — มันเริ่มจาก HTTP request ธรรมดาแล้ว "อัปเกรด" การเชื่อมต่อนั้นให้กลายเป็น WebSocket แทน:

```
Client                                          Server
  │  GET /ws HTTP/1.1                             │
  │  Upgrade: websocket                           │
  │  Connection: Upgrade                          │
  │  Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==   │
  │  Sec-WebSocket-Version: 13                     │
  │───────────────────────────────────────────────>
  │                                                 │
  │  HTTP/1.1 101 Switching Protocols               │
  │  Upgrade: websocket                             │
  │  Connection: Upgrade                            │
  │  Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZ...  │
  │<───────────────────────────────────────────────│
  │                                                 │
  │  === จากตรงนี้ ไม่ใช่ HTTP อีกต่อไป ===          │
  │  === เป็น WebSocket frame สองทางตลอดเวลา ===     │
  │<═══════════════════════════════════════════════>
```

Response `101 Switching Protocols` คือจุดเปลี่ยนสำคัญ — TCP connection เดิมที่ใช้คุย HTTP ถูก "ยึด" (hijack) มาใช้เป็น WebSocket connection แทน ไม่มีการเปิด connection ใหม่ หลังจากจุดนี้ทั้งสองฝั่งส่งข้อมูลเป็น **WebSocket frame** ไปมาได้อิสระ ไม่ต้องเป็นรูปแบบ request/response อีกต่อไป

---

## 3. ทำไม Go standard library ไม่มี WebSocket ให้

ต่างจาก `net/http` ที่ Go เตรียม HTTP server/client ไว้ครบสมบูรณ์ **`net/http` ของ Go ไม่มี implementation ของ WebSocket protocol ให้ในตัว** มีแค่ฟังก์ชัน `http.Hijacker` ที่เปิดทางให้ดึง raw TCP connection ออกมาจัดการเองได้ (ซึ่งเป็นกลไกระดับล่างที่ library WebSocket ใช้งานอยู่ข้างใต้) แต่การ implement WebSocket protocol แบบเต็มรูปแบบ (frame parsing, masking, ping/pong, close handshake ตาม RFC 6455) เป็นงานที่ซับซ้อนพอสมควร ทีม Go จึงปล่อยให้เป็นหน้าที่ของ third-party library แทนที่จะรวมเข้า standard library

---

## 4. เลือก library: `gorilla/websocket` vs `nhooyr.io/websocket`

สอง library หลักที่ชุมชน Go ใช้กันมากที่สุด:

| | `gorilla/websocket` | `nhooyr.io/websocket` |
|---|---|---|
| ความนิยม | นิยมที่สุด ใช้กันแพร่หลายมาอย่างยาวนาน | ใหม่กว่า ออกแบบให้ API ทันสมัยกว่า รองรับ `context` เป็นหลัก |
| API | คุ้นเคย ใกล้เคียง raw socket | เรียบง่ายกว่า เน้นใช้กับ `context.Context` |
| Community/ตัวอย่างโค้ด | มีตัวอย่างและ tutorial บนอินเทอร์เน็ตเยอะที่สุด | น้อยกว่า แต่เพิ่มขึ้นเรื่อยๆ |
| สถานะการดูแล | Active — ยังคง merge PR และแก้ bug ต่อเนื่อง | Active เช่นกัน |

> **ข้อควรรู้ที่สำคัญ**: `gorilla/websocket` อยู่ภายใต้องค์กร GitHub ชื่อ `gorilla` เดียวกับ `gorilla/mux` ที่เรียนไปใน **Part 058** ซึ่ง `gorilla/mux` ถูกประกาศเข้าสถานะ **maintenance mode** (ไม่พัฒนาฟีเจอร์ใหม่ รับแค่ patch ด้านความปลอดภัย) ไปแล้ว **แต่ `gorilla/websocket` เป็นคนละ repository กัน และยังคงถูกดูแลอย่างต่อเนื่องเป็นปกติ** อย่าเข้าใจผิดว่า "gorilla ทั้งหมดเลิกดูแลแล้ว" — ต้องเช็คสถานะของแต่ละ repository แยกกันเสมอ

บทนี้เลือกใช้ **`gorilla/websocket`** เป็นหลัก เพราะเป็นตัวที่พบเจอมากที่สุดในโค้ดจริงและมีตัวอย่างให้อ้างอิงเยอะที่สุด ทำให้เมื่อไปเจอโค้ดคนอื่นหรือค้นหาวิธีแก้ปัญหา จะเจอเคสของ library ตัวนี้ก่อนเสมอ

---

## 5. ติดตั้ง `gorilla/websocket`

```bash
mkdir websocket-demo && cd websocket-demo
go mod init websocket-demo
go get github.com/gorilla/websocket
```

---

## 6. Echo Server สมบูรณ์: Upgrade, Read/Write Loop, Ping/Pong Keepalive

```go
package main

import (
	"log"
	"net/http"
	"time"

	"github.com/gorilla/websocket"
)

var upgrader = websocket.Upgrader{
	ReadBufferSize:  1024,
	WriteBufferSize: 1024,
	// CheckOrigin ควรตรวจ origin จริงจังใน production กัน cross-site websocket hijacking
	// ตัวอย่างนี้อนุญาตทุก origin เพื่อความง่ายในการทดลอง
	CheckOrigin: func(r *http.Request) bool { return true },
}

const (
	pongWait   = 60 * time.Second
	pingPeriod = (pongWait * 9) / 10 // ต้อง ping ถี่กว่า pongWait เสมอ
)

func echoHandler(w http.ResponseWriter, r *http.Request) {
	conn, err := upgrader.Upgrade(w, r, nil)
	if err != nil {
		log.Println("upgrade error:", err)
		return
	}
	defer conn.Close()

	// ตั้ง deadline เริ่มต้น และ handler สำหรับ pong ที่ client ตอบกลับ
	// เพื่อรู้ว่า connection ยังมีชีวิตอยู่จริง ไม่ใช่ค้างเงียบ (dead connection)
	conn.SetReadDeadline(time.Now().Add(pongWait))
	conn.SetPongHandler(func(string) error {
		conn.SetReadDeadline(time.Now().Add(pongWait))
		return nil
	})

	done := make(chan struct{})
	go func() {
		ticker := time.NewTicker(pingPeriod)
		defer ticker.Stop()
		for {
			select {
			case <-ticker.C:
				if err := conn.WriteMessage(websocket.PingMessage, nil); err != nil {
					return
				}
			case <-done:
				return
			}
		}
	}()
	defer close(done)

	for {
		msgType, data, err := conn.ReadMessage()
		if err != nil {
			break
		}
		if err := conn.WriteMessage(msgType, data); err != nil {
			log.Println("write error:", err)
			break
		}
	}
}

func main() {
	http.HandleFunc("/ws", echoHandler)
	log.Println("listening on :8080")
	log.Fatal(http.ListenAndServe(":8080", nil))
}
```

อธิบายกลไก keepalive: **WebSocket connection ที่เปิดค้างไว้นานๆ อาจถูกตัดโดย proxy/load balancer ที่อยู่ระหว่างทางโดยไม่มีใครรู้ตัว** (เพราะดูเหมือนไม่มีการส่งข้อมูลผ่านไปเลยเป็นเวลานาน) วิธีป้องกันคือส่ง **ping frame** เป็นระยะ (ในตัวอย่างนี้ทุก 54 วินาที คือ 90% ของ `pongWait`) — ถ้าอีกฝั่งยังมีชีวิตอยู่จริง มันจะตอบ **pong frame** กลับมาอัตโนมัติ (กลไกนี้ library จัดการให้แล้วผ่าน `SetPongHandler`) ซึ่งจะไปเลื่อน `ReadDeadline` ออกไปอีก ถ้าไม่มี pong ตอบกลับมาภายใน `pongWait` แปลว่า connection ตายแล้วจริงๆ การอ่านครั้งถัดไปจะ timeout และ error ทำให้ loop จบและปิด connection อย่างถูกต้อง

ทดสอบด้วย client จริง (เขียน test ด้วย `httptest.NewServer` และ `websocket.DefaultDialer.Dial`) ยืนยันว่า:

```
ส่ง "hello" ไป -> ได้ "hello" กลับมาถูกต้อง (echo ทำงานตามชื่อ)
ส่ง close frame มาตรฐาน -> เซิร์ฟเวอร์ตอบ close frame กลับให้อัตโนมัติ (graceful close ทำงานถูกต้อง)
```

---

## 7. อันตรายที่มองไม่เห็น: Concurrent Write ไม่ปลอดภัย

โค้ดในหัวข้อที่ 6 ทำงานถูกต้อง **แต่ซ่อนบั๊กที่อันตรายมากไว้อยู่**: สังเกตว่ามี **สอง goroutine** ที่เรียก `conn.WriteMessage` — goroutine หลัก (echo กลับ) และ goroutine ของ ping ticker ที่รันพื้นหลัง

นี่คือกฎที่ **ต้องจำให้ขึ้นใจ** เมื่อใช้ `gorilla/websocket`:

> **`*websocket.Conn` รองรับ "นักอ่านหนึ่งคน (one reader) + นักเขียนหนึ่งคน (one writer) พร้อมกัน" เท่านั้น ถ้ามีมากกว่าหนึ่ง goroutine เรียก `WriteMessage` (หรือ `NextWriter`) พร้อมกัน โดยไม่มีการ synchronize จะทำให้ frame ข้อมูลปนกัน (interleaved) เกิดข้อมูลเสียหายที่ฝั่งรับ หรือถึงขั้น panic**

โค้ดในหัวข้อที่ 6 มีความเสี่ยงนี้จริง เพราะถ้าจังหวะ ping ticker ยิง `WriteMessage(PingMessage, nil)` พร้อมกับที่ loop หลักกำลัง `WriteMessage(msgType, data)` เพื่อ echo ข้อความ (ทั้งสองมาจาก goroutine คนละตัว) การเขียนสองครั้งนี้อาจสลับปนกันได้ นี่คือ **race condition** แบบเดียวกับที่เรียนไปใน **Part 044 (Race Condition และ Race Detector)** เพียงแต่เกิดขึ้นบน connection object แทนที่จะเป็นตัวแปรธรรมดา

**หลักการเดียวกับ concurrency ทั่วไปที่เรียนมาตลอด**: "อย่าแชร์ memory (หรือในที่นี้คือ connection) โดยไม่ป้องกัน" — เราต้องบังคับให้การเขียนเกิดขึ้นทีละครั้งเท่านั้น มีสองวิธีมาตรฐานที่ใช้แก้ปัญหานี้

---

## 8. ทางแก้ที่ 1: ห่อ Connection ด้วย Mutex

วิธีตรงไปตรงมาที่สุดคือใช้ `sync.Mutex` ล้อมรอบทุกจุดที่เรียก `WriteMessage` (แนวคิดตรงจาก **Part 039: sync package**):

```go
// safeConn ห่อ *websocket.Conn ด้วย mutex เพื่อบังคับว่าจะมีการเขียน (WriteMessage)
// เกิดขึ้นทีละครั้งเท่านั้น ไม่ว่าจะถูกเรียกจาก goroutine ไหนก็ตาม
type safeConn struct {
	conn *websocket.Conn
	mu   sync.Mutex
}

func (s *safeConn) WriteMessage(messageType int, data []byte) error {
	s.mu.Lock()
	defer s.mu.Unlock()
	return s.conn.WriteMessage(messageType, data)
}
```

แล้วแก้ handler ให้เขียนผ่าน `safeConn` แทนการเรียก `conn.WriteMessage` ตรงๆ:

```go
func echoHandler(w http.ResponseWriter, r *http.Request) {
	conn, err := upgrader.Upgrade(w, r, nil)
	if err != nil {
		return
	}
	defer conn.Close()
	safe := &safeConn{conn: conn}

	conn.SetReadDeadline(time.Now().Add(pongWait))
	conn.SetPongHandler(func(string) error {
		conn.SetReadDeadline(time.Now().Add(pongWait))
		return nil
	})

	done := make(chan struct{})
	go func() {
		ticker := time.NewTicker(pingPeriod)
		defer ticker.Stop()
		for {
			select {
			case <-ticker.C:
				// เขียนผ่าน safe.WriteMessage เท่านั้น ไม่เรียก conn.WriteMessage ตรงๆ อีก
				if err := safe.WriteMessage(websocket.PingMessage, nil); err != nil {
					return
				}
			case <-done:
				return
			}
		}
	}()
	defer close(done)

	for {
		msgType, data, err := conn.ReadMessage() // อ่านมีแค่ goroutine เดียวอยู่แล้ว ปลอดภัย
		if err != nil {
			break
		}
		if err := safe.WriteMessage(msgType, data); err != nil {
			break
		}
	}
}
```

ทดสอบด้วย `go test -race` ยืนยันว่าเวอร์ชันนี้ไม่มี data race เกิดขึ้นแล้ว — ทุกการเขียนต้องผ่าน mutex เดียวกันเสมอ ไม่มีทางที่สอง goroutine จะเขียนพร้อมกันได้อีก

> สังเกตว่า **การอ่าน (`ReadMessage`) ไม่จำเป็นต้องมี mutex** ในตัวอย่างนี้ เพราะมีแค่ goroutine เดียว (loop หลัก) ที่อ่านจาก connection อยู่แล้ว กฎ "reader หนึ่งคน + writer หนึ่งคน" ของ gorilla/websocket จึงยังคงเป็นจริงอยู่สำหรับฝั่งอ่าน แต่ถ้าออกแบบให้มีหลาย goroutine อ่านพร้อมกันด้วย ก็ต้อง synchronize การอ่านเช่นกัน

---

## 9. ทางแก้ที่ 2: Single Writer Goroutine + Channel (รูปแบบมาตรฐานสำหรับ Chat/Broadcast)

เมื่อระบบซับซ้อนขึ้น เช่นต้องการ **broadcast ข้อความจากหลาย client ไปหาผู้ใช้ทุกคนที่เชื่อมต่ออยู่** (เหมือนแอปแชทที่จะสร้างเต็มรูปแบบใน **Part 104**) การใช้ mutex ล้วนๆ เริ่มไม่สะดวก เพราะมีหลายจุดในระบบที่อยากส่งข้อความไปหา client คนเดียวกัน (จาก hub กลาง, จาก timer, จาก event อื่นๆ)

รูปแบบที่นิยมและปลอดภัยกว่าคือ **ให้แต่ละ connection มี goroutine เดียวเท่านั้นที่ได้รับอนุญาตให้เขียนลง socket** ส่วนโค้ดส่วนอื่นทั้งหมดในระบบ "ส่งข้อความ" ผ่าน **channel** แทนการเรียก `WriteMessage` ตรงๆ — ตรงกับหลักการที่เรียนมาตั้งแต่ **Part 001**: *"Do not communicate by sharing memory; share memory by communicating"*

```go
// client แทนผู้ใช้หนึ่งคนที่เชื่อมต่อเข้ามา
// send คือ "กล่องจดหมาย" ของ client นี้ — ทุกอย่างที่จะส่งออกไปหา client
// ต้องผ่านช่องทางนี้เท่านั้น ไม่มีใครเรียก conn.WriteMessage ตรงๆ นอกจาก writePump
// goroutine เพียงตัวเดียว
type client struct {
	hub  *hub
	conn *websocket.Conn
	send chan []byte
}

type hub struct {
	mu      sync.Mutex
	clients map[*client]bool
}

func newHub() *hub {
	return &hub{clients: map[*client]bool{}}
}

func (h *hub) register(c *client) {
	h.mu.Lock()
	h.clients[c] = true
	h.mu.Unlock()
}

func (h *hub) unregister(c *client) {
	h.mu.Lock()
	if _, ok := h.clients[c]; ok {
		delete(h.clients, c)
		close(c.send)
	}
	h.mu.Unlock()
}

// broadcast ส่งข้อความไปหาทุก client โดยแค่ "ยัดใส่ channel"
// ไม่ได้เขียนลง socket ตรงๆ เลย ปลอดภัยแม้จะเรียกจากหลาย goroutine พร้อมกัน
func (h *hub) broadcast(msg []byte) {
	h.mu.Lock()
	defer h.mu.Unlock()
	for c := range h.clients {
		select {
		case c.send <- msg:
		default:
			// client รับข้อความไม่ทัน (buffer เต็ม) ตัดทิ้งกันค้าง
			delete(h.clients, c)
			close(c.send)
		}
	}
}
```

`writePump` คือ goroutine เดียวเท่านั้นที่แตะ `conn.WriteMessage`:

```go
// writePump เป็น goroutine เดียวที่ได้รับอนุญาตให้เขียนลง c.conn
// อ่านค่าจาก c.send มาเขียนทีละอัน และยังรับผิดชอบส่ง ping เป็นระยะด้วย
// เพื่อรวมการเขียนทั้งหมดไว้ที่เดียว ไม่ชนกันเอง
func (c *client) writePump() {
	ticker := time.NewTicker(pingPeriod)
	defer func() {
		ticker.Stop()
		c.conn.Close()
	}()
	for {
		select {
		case msg, ok := <-c.send:
			c.conn.SetWriteDeadline(time.Now().Add(writeWait))
			if !ok {
				c.conn.WriteMessage(websocket.CloseMessage, []byte{})
				return
			}
			if err := c.conn.WriteMessage(websocket.TextMessage, msg); err != nil {
				return
			}
		case <-ticker.C:
			c.conn.SetWriteDeadline(time.Now().Add(writeWait))
			if err := c.conn.WriteMessage(websocket.PingMessage, nil); err != nil {
				return
			}
		}
	}
}

// readPump เป็น goroutine เดียวที่อ่านจาก c.conn
// เมื่ออ่านได้ข้อความ จะส่งต่อให้ hub กระจายให้ทุกคน
func (c *client) readPump() {
	defer func() {
		c.hub.unregister(c)
		c.conn.Close()
	}()
	c.conn.SetReadDeadline(time.Now().Add(pongWait))
	c.conn.SetPongHandler(func(string) error {
		c.conn.SetReadDeadline(time.Now().Add(pongWait))
		return nil
	})
	for {
		_, data, err := c.conn.ReadMessage()
		if err != nil {
			break
		}
		c.hub.broadcast(data)
	}
}

func serveWs(h *hub, w http.ResponseWriter, r *http.Request) {
	conn, err := upgrader.Upgrade(w, r, nil)
	if err != nil {
		return
	}
	c := &client{hub: h, conn: conn, send: make(chan []byte, 16)}
	h.register(c)

	go c.writePump()
	go c.readPump()
}
```

รูปแบบนี้คือโครงสร้างเดียวกับตัวอย่าง chat application อย่างเป็นทางการของ `gorilla/websocket` เอง และเป็นรากฐานที่เราจะนำไปต่อยอดเต็มรูปแบบใน **Part 104: Real-time Chat Application** ทดสอบด้วยการเปิด 2 client เชื่อมต่อเข้า hub เดียวกัน แล้วให้ client A ส่งข้อความ ยืนยันว่า **ทั้ง client A และ B ได้รับข้อความเดียวกันถูกต้อง** — และรันผ่าน `go test -race` โดยไม่พบ data race ใดๆ เลย แม้จะมีหลาย goroutine ทำงานพร้อมกัน (readPump, writePump ของแต่ละ client, และ hub broadcast)

### เปรียบเทียบสองวิธี

| | Mutex | Channel + Writer Goroutine |
|---|---|---|
| ความซับซ้อน | ง่ายกว่า เหมาะกับ connection เดี่ยวๆ | ซับซ้อนกว่าเล็กน้อย แต่ scale ได้ดีกว่า |
| เหมาะกับ | Echo server, connection ที่ไม่มีใครอื่นต้องส่งข้อความแทน | Chat, broadcast, ระบบที่หลายส่วนของโปรแกรมต้องส่งข้อความหา client เดียวกัน |
| Backpressure (client รับไม่ทัน) | ต้องจัดการเอง | จัดการได้ง่ายในจุดเดียว (เช่น buffered channel + ตัดทิ้งถ้าเต็ม) |

---

## 10. Graceful Close

WebSocket มี **close handshake** ของตัวเอง (ไม่ใช่แค่ปิด TCP connection ทิ้งดื้อๆ): ฝั่งที่ต้องการปิดส่ง **close frame** ที่มี close code และเหตุผลแนบไปด้วย อีกฝั่งควรตอบ close frame กลับมาเพื่อยืนยัน แล้วจึงปิด TCP connection จริง

`gorilla/websocket` จัดการเรื่องนี้ให้เกือบทั้งหมดโดยอัตโนมัติผ่าน **default close handler**: เมื่อ `ReadMessage()` อ่านเจอ close frame ที่ส่งเข้ามา มันจะ:

1. ส่ง close frame ตอบกลับให้อัตโนมัติ
2. คืนค่า error ประเภท `*websocket.CloseError` ออกมาจาก `ReadMessage()`

โค้ดของเราแค่ต้องเช็ค error แล้ว break ออกจาก loop ให้ถูกต้อง (ซึ่งทำอยู่แล้วในทุกตัวอย่างของบทนี้) ทดสอบยืนยันจริงว่า: เมื่อ client ส่ง `websocket.FormatCloseMessage(websocket.CloseNormalClosure, "")` เซิร์ฟเวอร์ตอบ close frame กลับมาด้วย code `websocket.CloseNormalClosure` ถูกต้องตามมาตรฐาน

หากต้องการปิด connection จากฝั่งเซิร์ฟเวอร์เอง (เช่นเซิร์ฟเวอร์กำลัง shutdown) ควรส่ง close frame ก่อนปิดจริง แทนที่จะเรียก `conn.Close()` ทันที เพื่อให้ client รู้เหตุผลของการตัดการเชื่อมต่อ:

```go
c.conn.WriteMessage(websocket.CloseMessage,
	websocket.FormatCloseMessage(websocket.CloseGoingAway, "server shutting down"))
time.Sleep(time.Second) // ให้เวลา client รับ close frame ก่อนตัดจริง
c.conn.Close()
```

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- WebSocket แก้ปัญหาที่ HTTP request/response ธรรมดาทำไม่ได้: การสื่อสารสองทางแบบ real-time บน connection เดียวที่คงอยู่ตลอด
- WebSocket เริ่มต้นด้วย HTTP request ที่ "อัปเกรด" (`101 Switching Protocols`) แล้วเปลี่ยนเป็น WebSocket frame หลังจากนั้น
- Go standard library ไม่มี WebSocket ให้ในตัว ต้องใช้ third-party library เช่น `gorilla/websocket`
- `gorilla/mux` อยู่ใน maintenance mode แต่ **`gorilla/websocket` เป็นคนละ repository ที่ยังดูแลต่อเนื่องปกติ**
- Ping/Pong keepalive ป้องกัน connection ถูกตัดโดย proxy ระหว่างทางเมื่อไม่มีข้อมูลไหลผ่านเป็นเวลานาน
- **`*websocket.Conn` ไม่ปลอดภัยสำหรับการเขียนพร้อมกันจากหลาย goroutine** — ต้อง serialize การเขียนเสมอ
- สองวิธีมาตรฐานในการ serialize การเขียน: **mutex ห่อ connection** (ง่าย เหมาะกับ connection เดี่ยว) หรือ **single writer goroutine + channel** (scale ดีกว่า เหมาะกับ broadcast/chat) — ทั้งสองผ่านการทดสอบด้วย `go test -race` แล้วว่าไม่มี data race
- Close handshake ของ WebSocket จัดการเกือบทั้งหมดโดย `gorilla/websocket` อัตโนมัติผ่าน default close handler

## แบบฝึกหัดท้ายบท

1. รัน echo server ในหัวข้อที่ 6 แล้วเชื่อมต่อด้วย client เขียนเอง (หรือใช้เครื่องมืออย่าง `websocat`/browser dev console) ทดลองส่งข้อความหลายแบบและสังเกตว่า echo กลับมาถูกต้อง
2. ใช้ `go test -race` รันกับเวอร์ชัน "buggy" ในหัวข้อที่ 6 ที่มี ping goroutine กับ echo loop เขียนพร้อมกัน (ลองย่อ `pingPeriod` ให้สั้นมากๆ เพื่อเพิ่มโอกาสชนกัน) แล้วเทียบกับเวอร์ชันที่แก้แล้วในหัวข้อที่ 8
3. ขยาย hub ในหัวข้อที่ 9 ให้รองรับ "ห้องแชท" หลายห้อง โดยแต่ละ client อยู่ในห้องเดียว และ broadcast จะส่งเฉพาะคนในห้องเดียวกันเท่านั้น
4. เพิ่ม endpoint `/ws/stats` ที่ตอบ (ผ่าน HTTP ธรรมดา ไม่ใช่ WebSocket) จำนวน client ที่เชื่อมต่ออยู่ในปัจจุบันของ hub
5. ทดลองปิด client กลางคันโดยไม่ส่ง close frame ที่ถูกต้อง (เช่น kill process กลางทาง) แล้วสังเกตว่า `pongWait` timeout ทำงานอย่างไรฝั่งเซิร์ฟเวอร์ในการตรวจจับว่า connection ตายแล้ว
6. ลองเปลี่ยนไปใช้ `nhooyr.io/websocket` เขียน echo server แบบเดียวกับหัวข้อที่ 6 ใหม่ เปรียบเทียบ API และความยากง่ายกับ `gorilla/websocket`

---

**ต่อไป**: [Part 070 — GraphQL ด้วย Go](./070-graphql.md)
