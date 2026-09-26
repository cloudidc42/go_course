# Part 104: โปรเจกต์ Real-time Chat Application

> ภาคที่ 10: มืออาชีพและระดับโลก (Professional & World-Class) — ตอนที่ 5 จาก 11 (Part 100–110)

## สารบัญของบทนี้

1. ภาพรวมโปรเจกต์และสถาปัตยกรรม
2. เตรียมโปรเจกต์และติดตั้ง `gorilla/websocket`
3. ออกแบบ `Message`: envelope รูปแบบเดียวสำหรับทุก event
4. `Room`: event loop เดียวเป็นเจ้าของ state ทั้งหมด (channel-based, ไม่ใช้ mutex)
5. `Hub`: จัดการหลายห้องพร้อมกันด้วย `sync.RWMutex`
6. `Client`: หนึ่ง connection คู่กับสอง goroutine (`readPump` / `writePump`)
7. Panic Recovery: กัน client ตัวเดียวพังไม่ให้ทั้งเซิร์ฟเวอร์ล้ม
8. HTTP Server และหน้าเว็บ client ฝังด้วย `embed.FS`
9. `main.go`: ประกอบร่างทั้งหมดเข้าด้วยกัน
10. รันจริง: `go build`, `go vet`, และทดสอบด้วย client จริงสองคน
11. Test อัตโนมัติ: จำลอง WebSocket client พร้อมกันหลายสิบตัว
12. ยืนยันด้วย Race Detector: `go test -race`
13. วิเคราะห์ความปลอดภัยของ concurrency ทีละจุด
14. ข้อจำกัดของ in-memory hub และแนวทาง scale ด้วย Redis Pub/Sub
15. แบบฝึกหัดท้ายบท

---

## 1. ภาพรวมโปรเจกต์และสถาปัตยกรรม

บทนี้คือโปรเจกต์รวบยอด (capstone project) ตัวที่สองของภาคที่ 10 เราจะสร้าง **Real-time Chat Application** ที่รันได้จริง มีหลายห้องแชท (multi-room), broadcast ข้อความให้ทุกคนในห้องเดียวกัน, แจ้งเตือนเข้า/ออกห้อง, และแสดงรายชื่อผู้ใช้ในห้องแบบ real-time — โดยต่อยอดโดยตรงจากรากฐานที่วางไว้ใน **Part 069 (WebSocket ด้วย Go)** ซึ่งจบด้วยรูปแบบ hub + client + `writePump`/`readPump` ที่เป็นโครงสร้างมาตรฐานของ `gorilla/websocket` เอง

ก่อนเริ่มเขียนโค้ด มาดูภาพรวมสถาปัตยกรรมกันก่อน:

```
                     ┌─────────────────────────────────────────┐
                     │                  Hub                     │
                     │  (sync.RWMutex คุ้มครอง map[string]*Room) │
                     └───────────────┬───────────────────────────┘
                                      │ GetOrCreateRoom("general")
                    ┌─────────────────┼─────────────────┐
                    ▼                                     ▼
            ┌───────────────┐                     ┌───────────────┐
            │  Room "general"│                     │  Room "random" │
            │  (goroutine    │                     │  (goroutine    │
            │   เดียว: run()) │                     │   เดียว: run()) │
            │  clients map   │                     │  clients map   │
            └───┬───┬───┬────┘                     └────────────────┘
                │   │   │
        ┌───────┘   │   └───────┐
        ▼           ▼           ▼
   ┌─────────┐ ┌─────────┐ ┌─────────┐
   │ Client A│ │ Client B│ │ Client C│   แต่ละ Client มี 2 goroutine:
   │ (Alice) │ │  (Bob)  │ │ (Carol) │   readPump (อ่านจาก socket)
   └────┬────┘ └────┬────┘ └────┬────┘   writePump (เขียนลง socket)
        │           │           │        เชื่อมกันด้วย channel "send"
     WebSocket   WebSocket   WebSocket
        │           │           │
     Browser A   Browser B   Browser C
```

โครงสร้างนี้ตรงกับสิ่งที่เรียนไว้แล้วในหลายบท:

| องค์ประกอบ | เทคนิคที่ใช้ | อ้างอิงบทก่อนหน้า |
|---|---|---|
| `Hub` จัดการหลายห้อง | `sync.RWMutex` ป้องกัน map ที่เข้าถึงจากหลาย goroutine | Part 039 (sync package) |
| `Room` จัดการ client ในห้องเดียว | goroutine เดียว (`run()`) เป็นเจ้าของ state ทั้งหมด สื่อสารผ่าน channel | Part 037 (Channels), Part 038 (select), Part 045 (Pub/Sub) |
| `Client` แต่ละคน | 2 goroutine แยกอ่าน/เขียน เชื่อมกันด้วย channel `send` | Part 036 (Goroutines), Part 069 (WebSocket) |
| ป้องกัน panic ล้มทั้งระบบ | `defer` + `recover()` ในทุก goroutine ของ client | Part 017 (Panic, Recover) |
| หน้าเว็บ client | ฝังไฟล์ HTML ด้วย `embed.FS` | Part 066 (Static Files) |
| ยืนยันความถูกต้องของ concurrency | `go test -race` | Part 044 (Race Condition และ Race Detector) |
| การ scale ข้ามหลายเซิร์ฟเวอร์ | Redis Pub/Sub (สเก็ตช์แนวคิด ไม่ implement เต็ม) | Part 077 (Redis) |

> **ข้อสังเกตสำคัญ**: โปรเจกต์นี้จงใจใช้ **สองเทคนิค concurrency ที่ต่างกัน** ในสองจุด — `Hub` ใช้ mutex เพราะงานที่ทำ (เช็ค/สร้างห้องใน map) เป็นงานสั้นๆ ไม่มี state ต่อเนื่อง ส่วน `Room` ใช้ channel-based single-owner goroutine เพราะต้องมี event loop ที่ประมวลผลเหตุการณ์ต่อเนื่อง (register, unregister, broadcast) ไปเรื่อยๆ ตลอดอายุของห้อง การเห็นทั้งสองแบบในโปรเจกต์เดียวช่วยตอกย้ำสิ่งที่ Part 039 และ 045 บอกไว้: **Go มีมากกว่าหนึ่งวิธีแก้ปัญหา concurrency ที่ถูกต้อง ต้องเลือกใช้ตามความเหมาะสมของแต่ละจุด ไม่ใช่ยึดติดวิธีเดียวทั้งระบบ**

โครงสร้างโฟลเดอร์ของโปรเจกต์ (อิงตาม standard layout ที่แนะนำไว้ใน **Part 001** และใช้จริงจังตั้งแต่ **Part 100: Clean Architecture**):

```
chatapp/
├── go.mod
├── go.sum
├── cmd/
│   └── server/
│       └── main.go              # entry point
└── internal/
    └── chat/
        ├── message.go            # struct Message (envelope กลาง)
        ├── room.go                # struct Room + event loop
        ├── hub.go                 # struct Hub (จัดการหลายห้อง)
        ├── client.go              # struct Client + readPump/writePump
        ├── server.go              # http.Handler + WebSocket upgrade + embed
        ├── static/
        │   └── index.html         # หน้าเว็บ client (ฝังเข้า binary)
        └── chat_test.go           # ทดสอบด้วย WebSocket client จริง
```

โค้ดทั้งหมดในบทนี้ **build ผ่านจริง, vet ผ่านจริง, test ผ่านจริง (รวมทั้งกับ `-race`) และรันจริงด้วย client สองตัวที่คุยกันผ่าน WebSocket จริง** — ผลลัพธ์ที่แสดงในบทนี้ทุกก้อนคัดลอกมาจาก terminal จริงบน Go 1.24.7 ไม่มีส่วนไหนที่เดาเอาว่า "น่าจะรันได้"

---

## 2. เตรียมโปรเจกต์และติดตั้ง `gorilla/websocket`

```bash
mkdir -p chatapp/cmd/server chatapp/internal/chat/static
cd chatapp
go mod init chatapp
go get github.com/gorilla/websocket
```

ผลลัพธ์จริงตอนรัน:

```
go: creating new go.mod: module chatapp
go: to add module requirements and sums:
	go mod tidy
go: added github.com/gorilla/websocket v1.5.3
```

`go.mod` ที่ได้:

```go
module chatapp

go 1.24.7

require github.com/gorilla/websocket v1.5.3
```

เหตุผลที่เลือก `gorilla/websocket` เหมือนกับที่อธิบายไว้ใน **Part 069 หัวข้อ 4**: เป็น library ที่พบเจอมากที่สุดในโค้ดจริง มีตัวอย่างอ้างอิงเยอะ และยังถูกดูแลต่อเนื่อง (แยกจาก `gorilla/mux` ที่อยู่ใน maintenance mode)

---

## 3. ออกแบบ `Message`: envelope รูปแบบเดียวสำหรับทุก event

ก่อนเขียน `Hub`/`Room`/`Client` ต้องออกแบบ "รูปแบบข้อความ" ที่วิ่งผ่าน WebSocket ก่อน บทนี้ใช้แนวทางที่แนะนำไว้ใน **Part 069 หัวข้อ 11**: ห่อทุก event ด้วย struct เดียวที่มี field `Type` บอกชนิด แล้วให้ทั้งสองฝั่ง (client และ server) `switch` ตามค่านั้น

```go
package chat

import "time"

// MessageType คือชนิดของ event ที่ส่งผ่าน WebSocket connection เดียวกัน
// ใช้รูปแบบ "envelope" แบบเดียวกับที่แนะนำไว้ใน Part 069 หัวข้อ 11:
// ห่อทุก event ด้วย struct เดียว แล้วแยกแยะด้วย field Type
type MessageType string

const (
	TypeMessage  MessageType = "message"  // ข้อความแชทปกติจากผู้ใช้คนหนึ่ง
	TypeJoin     MessageType = "join"     // แจ้งเตือนว่ามีผู้ใช้เข้าห้อง
	TypeLeave    MessageType = "leave"    // แจ้งเตือนว่ามีผู้ใช้ออกจากห้อง
	TypeUserList MessageType = "userlist" // รายชื่อผู้ใช้ปัจจุบันทั้งหมดในห้อง
	TypeError    MessageType = "error"    // แจ้ง client ว่ามีข้อผิดพลาด (เช่น ข้อความ JSON ผิดรูปแบบ)
)

// Message คือ struct เดียวที่ใช้ทั้งขาเข้า (client -> server) และขาออก (server -> client)
// ฝั่ง client ส่งมาแค่ Type กับ Text เท่านั้น ส่วน Room, User, Time, Users
// เซิร์ฟเวอร์เป็นคนเติมให้เอง เพื่อไม่ให้ client ปลอมตัวเป็นคนอื่นได้
type Message struct {
	Type  MessageType `json:"type"`
	Room  string      `json:"room,omitempty"`
	User  string      `json:"user,omitempty"`
	Text  string      `json:"text,omitempty"`
	Users []string    `json:"users,omitempty"`
	Time  time.Time   `json:"time,omitempty"`
}
```

จุดออกแบบที่สำคัญที่สุดในไฟล์นี้คือคอมเมนต์บรรทัดสุดท้าย: **client ส่งมาแค่ `type` กับ `text` เท่านั้น** ส่วน `room`, `user`, `time` เซิร์ฟเวอร์เป็นคนเติมเองเสมอ (ดูใน `client.go` หัวข้อ 6) นี่คือหลักการความปลอดภัยพื้นฐาน: **อย่าเชื่อ field ที่ client ควบคุมได้ ถ้า field นั้นมีผลต่อ identity หรือ authorization** — ถ้าปล่อยให้ client กำหนด `user` เองในทุกข้อความ ก็จะมีใครบางคนปลอมตัวเป็นคนอื่นส่งข้อความได้ทันที

---

## 4. `Room`: event loop เดียวเป็นเจ้าของ state ทั้งหมด (channel-based, ไม่ใช้ mutex)

`Room` คือหัวใจของระบบ broadcast แต่ละห้องแชทหนึ่งห้องคือ `Room` หนึ่งตัว ที่มี goroutine ของตัวเองรัน event loop ไม่รู้จบ

```go
package chat

import (
	"encoding/json"
	"log"
	"sort"
	"time"
)

// Room คือห้องแชทหนึ่งห้อง เก็บรายชื่อ client ที่เชื่อมต่ออยู่ในห้องนี้
//
// จุดออกแบบสำคัญ: Room ไม่ใช้ sync.Mutex ป้องกัน field "clients" เลย
// เพราะ map ตัวนี้ถูกแตะโดย goroutine เดียวเท่านั้นคือ run() -- โค้ดส่วนอื่น
// ทั้งหมด (readPump ของ client ต่างๆ, Hub) "คุย" กับ Room ผ่าน channel
// (register / unregister / broadcast) เท่านั้น ตรงกับหลักการ "Do not
// communicate by sharing memory; share memory by communicating" ที่กล่าวถึง
// ตั้งแต่ Part 001 และสาธิตให้เห็นจริงใน Part 037 (Channels พื้นฐาน) และ
// Part 045 (Pub/Sub pattern) -- นี่คือทางเลือกที่ต่างจาก Hub ระดับบนซึ่งใช้
// sync.RWMutex (ดู hub.go) เพื่อให้เห็นทั้งสองเทคนิคเทียบกันในโปรเจกต์เดียว
type Room struct {
	name string

	clients    map[*Client]bool
	register   chan *Client
	unregister chan *Client
	broadcast  chan Message

	// done ใช้บอก Hub ว่า Room นี้ไม่มี client เหลือแล้ว ให้ลบทิ้งจาก Hub ได้
	// (ป้องกัน memory leak จากห้องที่ทุกคนออกไปหมดแล้วแต่ยังค้างอยู่ในหน่วยความจำ)
	done chan string
}

func newRoom(name string, done chan string) *Room {
	return &Room{
		name:       name,
		clients:    make(map[*Client]bool),
		register:   make(chan *Client),
		unregister: make(chan *Client),
		broadcast:  make(chan Message, 64),
		done:       done,
	}
}

// run คือ event loop เดียวของ Room ต้องถูกเรียกด้วย "go room.run()" แค่ครั้งเดียว
// ต่อหนึ่ง Room ทุกการอ่าน/เขียน map clients เกิดขึ้นในนี้ที่เดียว จึงไม่มี
// data race แม้จะมีหลาย client goroutine ส่งเข้ามาผ่าน channel พร้อมกัน
// (แนวคิดเดียวกับ select ที่เรียนใน Part 038 และ worker pool ใน Part 041)
func (r *Room) run() {
	for {
		select {
		case c := <-r.register:
			r.clients[c] = true
			log.Printf("[room %s] %s joined (total=%d)", r.name, c.username, len(r.clients))
			r.broadcastLocked(Message{
				Type: TypeJoin,
				Room: r.name,
				User: c.username,
				Time: time.Now(),
			})
			r.sendUserList()

		case c := <-r.unregister:
			if _, ok := r.clients[c]; ok {
				delete(r.clients, c)
				close(c.send)
				log.Printf("[room %s] %s left (total=%d)", r.name, c.username, len(r.clients))
				r.broadcastLocked(Message{
					Type: TypeLeave,
					Room: r.name,
					User: c.username,
					Time: time.Now(),
				})
				if len(r.clients) == 0 {
					// ไม่มีใครอยู่ในห้องแล้ว แจ้ง Hub ให้เก็บกวาด Room นี้ทิ้ง
					// แล้วปิดตัวเอง -- ทำเป็นลำดับสุดท้ายเสมอ (goroutine นี้จบทันที)
					select {
					case r.done <- r.name:
					default:
					}
					return
				}
				r.sendUserList()
			}

		case msg := <-r.broadcast:
			r.broadcastLocked(msg)
		}
	}
}

// broadcastLocked ส่ง msg ไปหาทุก client ในห้อง (ชื่อ "Locked" ในที่นี้หมายถึง
// "ปลอดภัยเพราะรันอยู่ใน goroutine เดียวของ run() เท่านั้น" ไม่ได้ใช้ mutex จริง)
func (r *Room) broadcastLocked(msg Message) {
	data, err := json.Marshal(msg)
	if err != nil {
		log.Printf("marshal error: %v", err)
		return
	}
	for c := range r.clients {
		select {
		case c.send <- data:
		default:
			// client ตัวนี้รับข้อความไม่ทัน (buffer ของ send channel เต็ม)
			// แปลว่า client ช้าเกินไปหรือ connection ค้าง -- ตัดทิ้งเพื่อไม่ให้
			// client ตัวเดียวทำให้ broadcast ของทั้งห้องช้าตาม (backpressure
			// handling แบบเดียวกับที่กล่าวถึงใน Part 069 หัวข้อ 9)
			delete(r.clients, c)
			close(c.send)
		}
	}
}

func (r *Room) sendUserList() {
	users := make([]string, 0, len(r.clients))
	for c := range r.clients {
		users = append(users, c.username)
	}
	sort.Strings(users)
	r.broadcastLocked(Message{
		Type:  TypeUserList,
		Room:  r.name,
		Users: users,
		Time:  time.Now(),
	})
}
```

สังเกตชื่อฟังก์ชัน `broadcastLocked` — ตั้งใจตั้งชื่อแบบนี้ **แม้ไม่มี mutex จริงในฟังก์ชัน** เพื่อสื่อสารกับคนอ่านโค้ดในอนาคตว่า "ฟังก์ชันนี้ปลอดภัยเพราะมันถูกเรียกจาก `run()` เท่านั้น อย่าไปเรียกมันจากที่อื่น" นี่คือรูปแบบการตั้งชื่อที่ใช้กันจริงในโค้ด production ของหลายบริษัท เพื่อบันทึก **invariant ที่คอมไพเลอร์ตรวจให้ไม่ได้** ไว้ในชื่อฟังก์ชันเอง

จุดที่ควรเน้นอีกจุดคือ `select` ที่มี `default` ใน `broadcastLocked` — เป็น **non-blocking send** (เรียนไว้ใน **Part 038: คำสั่ง select**) ถ้า `c.send <- data` ทำไม่ได้ทันที (buffer เต็ม) จะตกไปที่ `default` ทันทีแทนที่จะบล็อกรอ วิธีนี้ป้องกันไม่ให้ client ที่ช้าหรือค้างตัวเดียว **ทำให้ทั้ง event loop ของ `run()` ค้างตามไปด้วย** (ถ้า `run()` ค้าง แปลว่าห้องทั้งห้องหยุดทำงาน ไม่มีใครในห้องได้รับข้อความอะไรอีกเลย)

---

## 5. `Hub`: จัดการหลายห้องพร้อมกันด้วย `sync.RWMutex`

`Hub` อยู่เหนือ `Room` อีกชั้นหนึ่ง มีหน้าที่แค่ "หาห้องที่ชื่อนี้ ถ้าไม่มีให้สร้างใหม่" และ "เก็บกวาดห้องที่ว่างเปล่าทิ้ง"

```go
package chat

import (
	"sync"
)

// Hub เก็บรายการ Room ทั้งหมดที่มีอยู่ในเซิร์ฟเวอร์ (multi-room chat)
//
// ต่างจาก Room ที่ใช้ channel คุมการเข้าถึง map ทั้งหมด Hub เลือกใช้
// sync.RWMutex ตรงๆ (เทคนิคจาก Part 039: sync package) เพราะ GetOrCreateRoom
// ถูกเรียกจาก HTTP handler ของทุก connection ที่เพิ่งเข้ามาใหม่ -- เป็นงาน
// สั้นๆ (เช็ค/สร้าง entry ใน map) ที่ไม่มี logic ต่อเนื่องยาวเหมือน Room.run()
// การใช้ RWMutex ตรงนี้เรียบง่ายกว่าการเปิด channel-based actor อีกชั้น
// และทำให้เห็นในบทเดียวว่า Go มีมากกว่าหนึ่งวิธีแก้ปัญหา concurrency ที่ถูกต้อง
// ทั้งคู่ -- เลือกใช้ตามความเหมาะสมของแต่ละจุด ไม่ใช่ใช้แบบเดียวทั้งระบบ
type Hub struct {
	mu    sync.RWMutex
	rooms map[string]*Room

	// done รับชื่อห้องที่ไม่มี client เหลือแล้วจาก Room.run() เพื่อลบทิ้งจาก
	// map -- มี goroutine เดียว (reapLoop) ที่คอยอ่าน channel นี้และแก้ไข map
	// จึงไม่ชนกับ mu ที่ป้องกัน GetOrCreateRoom ในเวลาเดียวกันได้ (ยังคง lock
	// ด้วย mu เหมือนกันเพื่อความถูกต้อง 100%)
	done chan string
}

// NewHub สร้าง Hub เปล่าๆ พร้อมเริ่ม goroutine เก็บกวาดห้องที่ว่างเปล่า
func NewHub() *Hub {
	h := &Hub{
		rooms: make(map[string]*Room),
		done:  make(chan string),
	}
	go h.reapLoop()
	return h
}

func (h *Hub) reapLoop() {
	for name := range h.done {
		h.mu.Lock()
		delete(h.rooms, name)
		h.mu.Unlock()
	}
}

// GetOrCreateRoom คืน Room ที่ชื่อ name อยู่แล้ว หรือสร้างใหม่พร้อม goroutine
// run() ของตัวเองถ้ายังไม่มี ปลอดภัยเมื่อเรียกพร้อมกันจากหลาย goroutine
// (หลาย client เชื่อมต่อห้องเดียวกันพร้อมกันตอนเริ่มระบบ)
func (h *Hub) GetOrCreateRoom(name string) *Room {
	// อ่านก่อนด้วย RLock: กรณีทั่วไปที่ห้องมีอยู่แล้วจะได้ไม่ต้องแย่ง write lock
	h.mu.RLock()
	r, ok := h.rooms[name]
	h.mu.RUnlock()
	if ok {
		return r
	}

	h.mu.Lock()
	defer h.mu.Unlock()
	// เช็คซ้ำอีกครั้งหลังได้ write lock (double-checked locking) เพราะระหว่าง
	// รอ Lock() อาจมี goroutine อื่นสร้างห้องเดียวกันไปแล้ว
	if r, ok := h.rooms[name]; ok {
		return r
	}
	r = newRoom(name, h.done)
	h.rooms[name] = r
	go r.run()
	return r
}

// RoomNames คืนชื่อห้องทั้งหมดที่มีอยู่ตอนนี้ ใช้ทำ endpoint สถิติ /rooms
func (h *Hub) RoomNames() []string {
	h.mu.RLock()
	defer h.mu.RUnlock()
	names := make([]string, 0, len(h.rooms))
	for name := range h.rooms {
		names = append(names, name)
	}
	return names
}
```

`GetOrCreateRoom` ใช้ลวดลาย **double-checked locking**: อ่านด้วย `RLock` ก่อน (เร็ว เพราะหลาย goroutine ถือ read lock พร้อมกันได้) ถ้าไม่เจอห้องค่อยขอ `Lock` แบบเขียน แล้ว **เช็คซ้ำอีกครั้ง** เพราะระหว่างที่รอคิว `Lock()` อาจมี goroutine อื่นสร้างห้องเดียวกันเสร็จไปแล้ว ถ้าไม่เช็คซ้ำ อาจสร้าง `Room` ซ้ำสองตัวสำหรับชื่อห้องเดียวกัน (ตัวหนึ่งจะถูกทิ้งไปเฉยๆ พร้อม goroutine `run()` ที่รันค้างอยู่ทำงานเปล่าๆ ตลอดไป — memory/goroutine leak) — ลวดลายนี้เป็นเทคนิคมาตรฐานที่กล่าวถึงใน **Part 039** และเป็นเหตุผลที่ต้องเข้าใจ mutex ให้ลึกกว่าการ "ใส่ `Lock()`/`Unlock()` ให้ครบ"

ส่วน `reapLoop` แก้ปัญหาที่มักถูกมองข้ามในโปรเจกต์ demo ทั่วไป: **ถ้าไม่มีใครลบห้องที่ว่างเปล่าทิ้ง ห้องที่คนเข้ามาคุยแล้วออกไปหมดจะยังค้างเป็น goroutine ที่ทำงานอยู่เรื่อยๆ (บล็อกรอที่ `select` ใน `run()`) และ entry ใน map ตลอดไป** ยิ่งมีคนสร้างห้องใหม่ๆ มากเท่าไหร่ (เช่นห้องแชทแบบ ephemeral ที่สร้างต่อการสนทนาหนึ่งครั้ง) ยิ่งรั่วมากขึ้นเรื่อยๆ `Room.run()` จึงส่งชื่อห้องผ่าน channel `done` กลับมาบอก `Hub` ทุกครั้งที่ client คนสุดท้ายออกจากห้อง

---

## 6. `Client`: หนึ่ง connection คู่กับสอง goroutine (`readPump` / `writePump`)

`Client` คือชั้นที่คุยกับ `*websocket.Conn` โดยตรง ยึดรูปแบบ **single writer goroutine + channel** ที่วางรากฐานไว้ใน **Part 069 หัวข้อ 9** ทุกประการ:

```go
package chat

import (
	"encoding/json"
	"log"
	"time"

	"github.com/gorilla/websocket"
)

const (
	writeWait  = 10 * time.Second
	pongWait   = 60 * time.Second
	pingPeriod = (pongWait * 9) / 10
	maxMsgSize = 4096 // จำกัดขนาดข้อความกัน client ส่ง payload มหาศาลมากวน
)

// Client แทนผู้ใช้หนึ่งคนที่เชื่อมต่อเข้ามาในห้องหนึ่งห้อง
//
// เช่นเดียวกับ Part 069 หัวข้อ 9: "send" คือกล่องจดหมายเดียวของ client นี้
// มีแค่ writePump goroutine เดียวเท่านั้นที่เรียก conn.WriteMessage ตรงๆ
// ส่วนที่เหลือของระบบ (Room.broadcastLocked) ส่งข้อความผ่านช่องทางนี้เท่านั้น
type Client struct {
	room     *Room
	conn     *websocket.Conn
	send     chan []byte
	username string
}
```

### `readPump`: มีแค่ goroutine เดียวเท่านั้นที่อ่านจาก socket

```go
// readPump อ่านข้อความจาก client แล้วส่งต่อให้ Room broadcast ต้องรันด้วย
// "go client.readPump()" เพียง goroutine เดียวต่อ client (กฎ "one reader"
// ของ gorilla/websocket ตามที่อธิบายไว้ใน Part 069 หัวข้อ 7)
func (c *Client) readPump() {
	// defer + recover ป้องกันไม่ให้ panic จาก client ตัวเดียว (เช่น bug ใน
	// การแปลง JSON ที่ไม่คาดคิด) ทำให้ทั้งโปรเซสเซิร์ฟเวอร์ล้มตายไปด้วย --
	// แนวคิดตรงจาก Part 017 (Panic, Recover) ที่ว่า goroutine แต่ละตัวต้อง
	// จัดการ panic ของตัวเอง เพราะ panic ใน goroutine หนึ่งที่ไม่ถูก recover
	// จะทำให้ "ทั้งโปรแกรม" ตายทันที ไม่ใช่แค่ goroutine นั้น
	defer func() {
		if r := recover(); r != nil {
			log.Printf("recovered panic in readPump for %s: %v", c.username, r)
		}
		c.room.unregister <- c
		c.conn.Close()
	}()

	c.conn.SetReadLimit(maxMsgSize)
	c.conn.SetReadDeadline(time.Now().Add(pongWait))
	c.conn.SetPongHandler(func(string) error {
		c.conn.SetReadDeadline(time.Now().Add(pongWait))
		return nil
	})

	for {
		_, data, err := c.conn.ReadMessage()
		if err != nil {
			// ครอบคลุมทั้งกรณี client ปิด connection ปกติ (CloseNormalClosure),
			// ปิดกะทันหันโดยไม่ส่ง close frame (network error), และ
			// ReadDeadline หมดอายุเพราะไม่มี pong ตอบกลับมา (connection ตาย)
			break
		}

		var in Message
		if err := json.Unmarshal(data, &in); err != nil {
			// ข้อความ JSON ผิดรูปแบบ -- แจ้ง client ตัวนี้ตัวเดียว ไม่ทำให้
			// ทั้งห้องพัง และไม่ crash เซิร์ฟเวอร์
			c.send <- mustMarshal(Message{
				Type: TypeError,
				Text: "invalid message format: " + err.Error(),
				Time: time.Now(),
			})
			continue
		}

		// เซิร์ฟเวอร์เป็นคนกำหนด Type/Room/User/Time เองเสมอ ไม่เชื่อค่าที่
		// client ส่งมาในฟิลด์เหล่านี้ (กัน client ปลอมตัวเป็นคนอื่น หรือยิง
		// event type แปลกๆ เข้าห้อง)
		in.Type = TypeMessage
		in.Room = c.room.name
		in.User = c.username
		in.Time = time.Now()

		c.room.broadcast <- in
	}
}
```

### `writePump`: มีแค่ goroutine เดียวเท่านั้นที่เขียนลง socket

```go
// writePump เป็น goroutine เดียวเท่านั้นที่เรียก c.conn.WriteMessage ทั้งข้อความ
// แชทจริง (จาก c.send) และ ping keepalive (จาก ticker) มารวมกันในลูป select
// เดียว จึงไม่มีทางที่การเขียนสองงานนี้จะชนกันได้ -- แก้ปัญหา concurrent write
// ที่อธิบายไว้ใน Part 069 หัวข้อ 7-9 (ทางเลือกที่ 2: single writer goroutine)
func (c *Client) writePump() {
	ticker := time.NewTicker(pingPeriod)
	defer func() {
		if r := recover(); r != nil {
			log.Printf("recovered panic in writePump for %s: %v", c.username, r)
		}
		ticker.Stop()
		c.conn.Close()
	}()

	for {
		select {
		case msg, ok := <-c.send:
			c.conn.SetWriteDeadline(time.Now().Add(writeWait))
			if !ok {
				// Room ปิด channel นี้แล้ว (เช่นถูกตัดออกเพราะรับข้อความไม่ทัน
				// หรือ unregister ไปแล้ว) -- ปิด connection อย่างสุภาพ
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

func mustMarshal(m Message) []byte {
	data, err := json.Marshal(m)
	if err != nil {
		// Message struct เป็น type ง่ายๆ ที่ marshal ไม่มีทางพังในทางปฏิบัติ
		// แต่ถ้าพังจริงๆ ให้เห็น log ชัดเจนแทนที่จะเงียบหาย
		log.Printf("marshal error: %v", err)
		return []byte(`{"type":"error","text":"internal error"}`)
	}
	return data
}
```

สังเกตว่าทั้ง `readPump` และ `writePump` **จบด้วยการปิด connection เสมอ ไม่ว่าจะออกจาก loop ด้วยเหตุผลอะไรก็ตาม** (error จากการอ่าน/เขียน, panic, หรือ channel ถูกปิด) — นี่คือหัวใจของการ "จัดการ disconnect อย่างเรียบร้อย (graceful)": ทุกเส้นทางที่ทำให้ goroutine จบ ต้องเดินไปจบที่จุดเดียวกันคือ cleanup path ใน `defer` เสมอ ไม่มีทางที่ resource (connection, entry ใน map ของ Room) จะรั่วเพราะ "ลืม cleanup กรณีพิเศษกรณีหนึ่ง"

---

## 7. Panic Recovery: กัน client ตัวเดียวพังไม่ให้ทั้งเซิร์ฟเวอร์ล้ม

หัวข้อนี้ขยายความเรื่อง `recover()` ใน `client.go` ให้ชัดเจนขึ้น เพราะเป็นจุดที่มักถูกมองข้ามในโปรเจกต์ chat แบบ demo ทั่วไป

**ข้อเท็จจริงสำคัญของ Go ที่เรียนไปแล้วใน Part 017**: ถ้า goroutine หนึ่ง panic และไม่มีใคร `recover()` เอาไว้ตลอดทาง panic นั้นจะไต่ขึ้นไปเรื่อยๆ จนถึงจุดที่ runtime ของ Go จับไม่ได้ แล้ว **ทั้งโปรเซส (ทุก goroutine ทุกตัว ทุก connection ที่เปิดอยู่) จะถูกฆ่าทิ้งทันที** — นี่ต่างจากภาษาอื่นที่ thread หนึ่งตายไม่จำเป็นต้องกระทบ thread อื่น

ลองนึกภาพว่าไม่มี `recover()` ใน `readPump`: สมมติภายหลังมีคนมาต่อยอดโค้ดนี้เพื่อรองรับ binary frame แบบกำหนดเอง แล้วเขียนโค้ดแบบ `buf[10]` โดยไม่เช็คความยาวก่อน ถ้ามี clientคนหนึ่งส่งข้อความสั้นกว่า 10 byte เข้ามา จะเกิด `index out of range` panic ขึ้นใน goroutine ของ client คนนั้น **โดยไม่มี `recover()`** panic นี้จะทำให้ **ทุกคนในทุกห้องหลุดออกจากแชทพร้อมกันทันที** เพราะทั้งโปรเซสตายไปด้วย ทั้งที่ต้นเหตุจริงๆ มาจาก client แค่คนเดียว

โค้ดของเรามี `defer func() { if r := recover(); r != nil { ... } ... }()` ครอบทั้ง `readPump` และ `writePump` ไว้แล้ว ทำให้ต่อให้เกิด panic แบบสมมติข้างต้นขึ้นจริงในอนาคต **จะมีแค่ client ตัวนั้นตัวเดียวที่หลุดออกจากห้อง** (ผ่าน cleanup path ปกติที่เรียก `c.room.unregister <- c` และ `c.conn.Close()`) ส่วนคนอื่นในห้องเดียวกันและห้องอื่นๆ ทั้งหมดไม่ได้รับผลกระทบเลย

เพื่อยืนยันว่า idiom นี้ทำงานถูกต้องจริง (ไม่ใช่แค่ "ดูน่าจะถูก") บทนี้เขียนเทสต์แยกทดสอบ pattern เดียวกันเป๊ะๆ:

```go
// TestRecoverGuardPattern ยืนยันว่า defer/recover idiom ที่ใช้จริงใน
// readPump และ writePump (client.go) ทำงานถูกต้อง: panic ที่เกิดขึ้นภายใน
// goroutine หนึ่งจะถูกดักไว้ ไม่ทำให้ทั้งโปรแกรมล้ม (Part 017: Panic, Recover)
//
// เทสต์นี้จำลอง idiom เดียวกันเป๊ะๆ แยกออกมาทดสอบเดี่ยวๆ เพราะการบังคับให้
// readPump/writePump จริง panic ต้องแก้โค้ด production เพื่อจุดประสงค์เทสต์
// เท่านั้น (เช่นแอบใส่ index out of range) ซึ่งจะทำให้โค้ดจริงสกปรกโดยใช่เหตุ
// -- แต่ idiom ที่ทดสอบตรงนี้คือ pattern เดียวกับที่ readPump/writePump ใช้จริง
// ทุกตัวอักษร (defer -> recover -> log -> cleanup)
func TestRecoverGuardPattern(t *testing.T) {
	cleanedUp := make(chan struct{})
	recoveredValue := make(chan any, 1)

	run := func() {
		defer func() {
			if r := recover(); r != nil {
				recoveredValue <- r
			}
			close(cleanedUp)
		}()
		panic("simulated bug: e.g. index out of range while parsing a client frame")
	}

	go run()

	select {
	case <-cleanedUp:
		// ผ่าน: cleanup path (close(cleanedUp)) ถูกเรียกแม้จะ panic กลางทาง
		// เหมือนที่ readPump รับประกันว่า room.unregister กับ conn.Close()
		// จะถูกเรียกเสมอไม่ว่า loop จะจบแบบปกติหรือ panic กลางทาง
	case <-time.After(2 * time.Second):
		t.Fatal("recover did not run cleanup within timeout -- goroutine likely crashed the process instead")
	}

	select {
	case v := <-recoveredValue:
		if v != "simulated bug: e.g. index out of range while parsing a client frame" {
			t.Fatalf("unexpected recovered value: %v", v)
		}
	default:
		t.Fatal("expected recover() to capture the panic value")
	}
}
```

รันแล้วผ่านจริง (ดูผลรวมทั้งชุดเทสต์ในหัวข้อ 12) — ยืนยันว่า `defer`/`recover` ดักค่า panic ได้ถูกต้อง และ cleanup path ทำงานเสมอไม่ว่า goroutine จะจบแบบปกติหรือจบด้วย panic

---

## 8. HTTP Server และหน้าเว็บ client ฝังด้วย `embed.FS`

ถัดมาคือชั้น HTTP ที่รับ WebSocket upgrade และเสิร์ฟหน้าเว็บทดสอบ บทนี้เลือกใช้ **`embed.FS` (Part 066: Static Files)** แทน **`html/template` (Part 065)** เพราะหน้าเว็บของเราเป็น static page ล้วนๆ ไม่มีตัวแปรฝั่งเซิร์ฟเวอร์ที่ต้อง render เข้าไปใน HTML (ข้อมูลทั้งหมดที่ต้องแสดงผล — ข้อความ, รายชื่อผู้ใช้ — มาจาก WebSocket ฝั่ง JavaScript ล้วนๆ) กฎง่ายๆ ที่ใช้ตัดสินใจ: **ถ้าหน้าเว็บต้องใส่ค่าจากฝั่งเซิร์ฟเวอร์ตอน render (เช่นชื่อผู้ใช้ที่ login, CSRF token) ให้ใช้ `html/template`; ถ้าเป็นหน้า static ล้วนที่ JavaScript จัดการข้อมูลเองทั้งหมด ใช้ `embed.FS` ธรรมดาก็พอ ไม่ต้องเปิด template engine มาโดยไม่จำเป็น**

```go
package chat

import (
	"embed"
	"encoding/json"
	"log"
	"net/http"

	"github.com/gorilla/websocket"
)

// static ฝัง index.html (หน้าเว็บ client) เข้าไปในตัว binary โดยตรงด้วย
// embed.FS (Part 066: Static Files) ทำให้ deploy ได้แค่ไฟล์ executable
// ไฟล์เดียว ไม่ต้องแจกไฟล์ .html แยกไปด้วย
//
//go:embed static/index.html
var static embed.FS

var upgrader = websocket.Upgrader{
	ReadBufferSize:  1024,
	WriteBufferSize: 1024,
	// production จริงต้องตรวจ Origin ให้เข้มงวดกว่านี้ (ดู Part 069 หัวข้อ 6)
	CheckOrigin: func(r *http.Request) bool { return true },
}

// NewServer คืน http.Handler ตัวเดียวที่รวม endpoint ทั้งหมดของแอปแชท
// รับ Hub เข้ามาจากภายนอก (dependency injection) เพื่อให้ทดสอบง่าย --
// test สร้าง Hub ของตัวเองแยกจาก main ได้ ไม่ต้องพึ่ง global state
func NewServer(hub *Hub) http.Handler {
	mux := http.NewServeMux()

	mux.HandleFunc("/", indexHandler)
	mux.HandleFunc("/ws", wsHandler(hub))
	mux.HandleFunc("/rooms", roomsHandler(hub))

	return mux
}

func indexHandler(w http.ResponseWriter, r *http.Request) {
	if r.URL.Path != "/" {
		http.NotFound(w, r)
		return
	}
	data, err := static.ReadFile("static/index.html")
	if err != nil {
		http.Error(w, "index not found", http.StatusInternalServerError)
		return
	}
	w.Header().Set("Content-Type", "text/html; charset=utf-8")
	w.Write(data)
}

// wsHandler รับการเชื่อมต่อ WebSocket ใหม่ อ่าน query param "room" และ "user"
// เพื่อกำหนดว่า client นี้เข้าห้องไหนด้วยชื่ออะไร ตัวอย่าง:
// ws://localhost:8080/ws?room=general&user=Alice
func wsHandler(hub *Hub) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		roomName := r.URL.Query().Get("room")
		username := r.URL.Query().Get("user")
		if roomName == "" {
			roomName = "general"
		}
		if username == "" {
			http.Error(w, "query param 'user' is required", http.StatusBadRequest)
			return
		}

		conn, err := upgrader.Upgrade(w, r, nil)
		if err != nil {
			log.Printf("upgrade error: %v", err)
			return
		}

		room := hub.GetOrCreateRoom(roomName)
		client := &Client{
			room: room,
			conn: conn,
			// buffer ขนาด 256 กันคน "ตกห้อง" ทันทีเมื่อมีข้อความมาถี่ๆ พร้อมกัน
			// หลายคน (เช่นห้องคึกคักที่ทุกคนพิมพ์พร้อมกัน) แต่ยังคงจำกัดไว้
			// เสมอ ไม่ใช่ unbounded channel -- ถ้า client ช้าเกินขนาด buffer นี้
			// จริงๆ (เช่น connection หลุดครึ่งทาง) Room.broadcastLocked จะตัด
			// ทิ้งเพื่อปกป้องคนอื่นในห้องไม่ให้ broadcast ช้าตามไปด้วย
			send:     make(chan []byte, 256),
			username: username,
		}
		room.register <- client

		// writePump และ readPump ต้องรันคนละ goroutine เสมอ (Part 036:
		// Goroutines พื้นฐาน) เพราะทั้งคู่เป็น loop ไม่รู้จบที่บล็อกอยู่
		// ตลอดเวลา -- ถ้ารันใน goroutine เดียวกัน อีกฝั่งจะไม่มีทางได้ทำงาน
		go client.writePump()
		go client.readPump()
	}
}

// roomsHandler ตอบ HTTP ธรรมดา (ไม่ใช่ WebSocket) บอกรายชื่อห้องที่มีคน
// อยู่ตอนนี้ เพื่อ debug/monitoring
func roomsHandler(hub *Hub) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		w.Header().Set("Content-Type", "application/json")
		json.NewEncoder(w).Encode(map[string]any{
			"rooms": hub.RoomNames(),
		})
	}
}
```

ตัวเลขที่น่าสนใจคือ **buffer ของ `client.send` ถูกปรับจาก 16 (ตามตัวอย่างพื้นฐานใน Part 069) เป็น 256** — ตัวเลขนี้ปรับขึ้นระหว่างทำโปรเจกต์จริงในบทนี้ หลังพบว่าตอนทดสอบด้วย client 20 ตัวส่งข้อความพร้อมกันถี่ๆ (หัวข้อ 11) buffer ขนาด 16 เล็กเกินไปจนทำให้ `broadcastLocked` ตัด client ทิ้งทั้งที่ client ยังทำงานปกติอยู่ (แค่ยังอ่านข้อความที่ค้างอยู่ไม่ทัน) — เป็นตัวอย่างจริงว่า **ตัวเลข buffer size ไม่ใช่ค่าที่เดาได้ลอยๆ ต้องวัดผลจากพฤติกรรมโหลดจริงของระบบ** และ trade-off เสมอ: buffer ใหญ่เกินไปกินหน่วยความจำต่อ client มากขึ้น (256 × ขนาดข้อความ × จำนวน client ทั้งหมด), buffer เล็กเกินไปตัด client ที่ยังปกติดีทิ้งเร็วเกินจำเป็น

ไฟล์ `static/index.html` เป็นหน้าเว็บเรียบง่ายที่มี input ใส่ชื่อผู้ใช้/ชื่อห้อง, ปุ่มเชื่อมต่อ, กล่องแสดงรายชื่อผู้ใช้ และช่องพิมพ์ข้อความ — เปิด `WebSocket` ไปที่ `/ws?room=...&user=...` แล้ว `switch` ตาม `msg.type` เพื่อแสดงผลให้ถูกประเภท (ตรงกับแนวคิด envelope ที่ออกแบบไว้ในหัวข้อ 3) โค้ดเต็มอยู่ในไฟล์ `internal/chat/static/index.html` ของโปรเจกต์ ไม่แสดงซ้ำในบทความเพราะเป็น HTML/JavaScript พื้นฐานที่ไม่มีเทคนิค Go เพิ่มเติม

---

## 9. `main.go`: ประกอบร่างทั้งหมดเข้าด้วยกัน

```go
package main

import (
	"log"
	"net/http"

	"chatapp/internal/chat"
)

func main() {
	hub := chat.NewHub()
	handler := chat.NewServer(hub)

	log.Println("chat server listening on :8080")
	log.Println("open http://localhost:8080 in a browser to try the demo client")
	log.Fatal(http.ListenAndServe(":8080", handler))
}
```

สั้นมาก เพราะ logic ทั้งหมดถูกแยกไว้ใน package `internal/chat` แล้ว — `main.go` ทำหน้าที่แค่ **ประกอบร่าง (wiring)** ตามหลัก dependency injection ที่เห็นซ้ำๆ ตลอดภาคที่ 10 (Part 100: Clean Architecture, Part 101: DDD) `main()` ไม่รู้จักรายละเอียดภายในของ `Hub`/`Room`/`Client` เลย รู้แค่ว่า `NewHub()` คืนอะไรบางอย่างที่ `NewServer()` รับเข้าไปแล้วได้ `http.Handler` กลับมา

---

## 10. รันจริง: `go build`, `go vet`, และทดสอบด้วย client จริงสองคน

ก่อนไปดูผลเทสต์อัตโนมัติ มายืนยันกันแบบ end-to-end ก่อนว่าระบบรันได้จริงด้วย client จริง

### Build และ vet

```bash
go build ./...
go vet ./...
```

ทั้งสองคำสั่งผ่านโดย **ไม่มี output ใดๆ เลย** (นิ่งสนิท = ผ่าน ไม่มี error/warning) และ `gofmt -l .` (เช็คว่ามีไฟล์ไหนไม่ตรง format มาตรฐานหรือไม่) ก็ไม่มี output เช่นกัน — โค้ดทั้งหมดผ่าน `gofmt` แล้ว

Build binary จริงและดูขนาด:

```bash
go build -o chatserver ./cmd/server
```

```
-rwxr-xr-x 1 root root 9014317 Sep 26 06:35 chatserver
```

ได้ executable ไฟล์เดียวขนาดประมาณ 9 MB (รวม HTML client ที่ฝังไว้ด้วย `embed.FS` แล้ว) — ตรงกับที่โปรโมทไว้ใน **Part 001**: "Binary เดียวจบ ไม่ต้องติดตั้ง runtime หรือ dependency เพิ่ม" ไฟล์นี้ก็อปปี้ไปรันบนเครื่องอื่นที่มี OS/architecture เดียวกันได้ทันทีโดยไม่ต้องติดตั้ง Go หรือไฟล์ `.html` แยกต่างหาก

### รันเซิร์ฟเวอร์จริง แล้วยิงด้วย client จริง

รันเซิร์ฟเวอร์:

```bash
./chatserver
```

```
2026/09/26 06:35:06 chat server listening on :8080
2026/09/26 06:35:06 open http://localhost:8080 in a browser to try the demo client
```

เช็คหน้าเว็บด้วย `curl`:

```bash
curl -s http://localhost:8080/ | head -5
```

```
<!DOCTYPE html>
<html lang="th">
<head>
<meta charset="UTF-8">
<title>Go Real-time Chat</title>
```

เช็ค endpoint สถิติก่อนมีใครเชื่อมต่อ:

```bash
curl -s http://localhost:8080/rooms
```

```
{"rooms":[]}
```

ต่อมาเขียนโปรแกรม Go เล็กๆ (ไม่ใช่ browser แต่เป็น WebSocket client จริงที่คุยกับเซิร์ฟเวอร์ด้วย protocol เดียวกันทุกประการ — ตรงกับที่โจทย์ต้องการ: "ทดสอบผ่าน Go-based test client แม้ไม่มีเบราว์เซอร์จริง") เพื่อจำลองผู้ใช้สองคนคุยกัน:

```go
package main

import (
	"fmt"
	"os"
	"time"

	"github.com/gorilla/websocket"
)

func main() {
	user := os.Args[1]
	url := fmt.Sprintf("ws://localhost:8080/ws?room=general&user=%s", user)
	conn, _, err := websocket.DefaultDialer.Dial(url, nil)
	if err != nil {
		panic(err)
	}
	defer conn.Close()

	go func() {
		for {
			_, data, err := conn.ReadMessage()
			if err != nil {
				return
			}
			fmt.Printf("[%s recv] %s\n", user, data)
		}
	}()

	time.Sleep(500 * time.Millisecond)
	if user == "Alice" {
		conn.WriteJSON(map[string]string{"type": "message", "text": "สวัสดีครับทุกคน"})
	}
	time.Sleep(2 * time.Second)
}
```

รัน Bob ก่อน แล้วรัน Alice ตามหลัง (พร้อมกัน) เข้าห้องเดียวกัน:

```bash
./manualclient Bob &
./manualclient Alice &
wait
```

ผลลัพธ์จริงที่ได้ (คัดลอกจาก terminal ตรงๆ ไม่ได้แก้ไข):

```
[Alice recv] {"type":"join","room":"general","user":"Alice","time":"2026-09-26T06:35:16.773033398Z"}
[Alice recv] {"type":"userlist","room":"general","users":["Alice"],"time":"2026-09-26T06:35:16.773109749Z"}
[Alice recv] {"type":"join","room":"general","user":"Bob","time":"2026-09-26T06:35:16.773828221Z"}
[Alice recv] {"type":"userlist","room":"general","users":["Alice","Bob"],"time":"2026-09-26T06:35:16.773840807Z"}
[Bob recv] {"type":"join","room":"general","user":"Bob","time":"2026-09-26T06:35:16.773828221Z"}
[Bob recv] {"type":"userlist","room":"general","users":["Alice","Bob"],"time":"2026-09-26T06:35:16.773840807Z"}
[Bob recv] {"type":"message","room":"general","user":"Alice","text":"สวัสดีครับทุกคน","time":"2026-09-26T06:35:17.273928078Z"}
[Alice recv] {"type":"message","room":"general","user":"Alice","text":"สวัสดีครับทุกคน","time":"2026-09-26T06:35:17.273928078Z"}
```

อ่านผลลัพธ์นี้ทีละบรรทัดจะเห็นพฤติกรรมที่ออกแบบไว้ทำงานถูกต้องครบทุกจุด:

1. Alice เชื่อมต่อก่อน ได้ `join` ของตัวเองและ `userlist` ที่มีแค่ `["Alice"]`
2. Bob เชื่อมต่อตามมา — **Alice เห็นทั้ง `join` ของ Bob และ `userlist` ใหม่ที่อัปเดตเป็น `["Alice","Bob"]` แบบ real-time** โดยไม่ต้อง refresh หน้าเว็บ (นี่คือเหตุผลทั้งหมดที่ต้องใช้ WebSocket แทน HTTP ธรรมดา ตามที่อธิบายไว้ใน Part 069 หัวข้อ 1)
3. Bob เห็น `join`/`userlist` ของตัวเองเช่นกัน (เพราะ `broadcastLocked` ส่งให้ทุก client ในห้องรวมถึงคนที่เพิ่งเข้ามาด้วย)
4. Alice ส่งข้อความ "สวัสดีครับทุกคน" — **ทั้ง Bob และ Alice เองได้รับข้อความนี้กลับมา** (การ broadcast ให้ผู้ส่งได้รับข้อความตัวเองคืนด้วยเป็นพฤติกรรมปกติของ chat hub ฝั่ง client/UI ต่างหากที่เป็นคนตัดสินใจไม่แสดงข้อความตัวเองซ้ำสอง ไม่ใช่หน้าที่ของ server)

หลังจากทั้งสอง client ตัดการเชื่อมต่อ (จบ `time.Sleep` ในโปรแกรม) ตรวจ `/rooms` อีกครั้ง:

```bash
curl -s http://localhost:8080/rooms
```

```
{"rooms":[]}
```

ห้อง `"general"` หายไปจาก Hub เอง — ยืนยันว่า `reapLoop` ในหัวข้อ 5 ทำงานถูกต้อง: เมื่อไม่มี client เหลือในห้อง ห้องนั้นถูกลบทิ้งอัตโนมัติ ไม่มี memory/goroutine ค้าง และ log ฝั่งเซิร์ฟเวอร์ก็ยืนยันลำดับเหตุการณ์เดียวกัน:

```
2026/09/26 06:35:06 chat server listening on :8080
2026/09/26 06:35:06 open http://localhost:8080 in a browser to try the demo client
2026/09/26 06:35:16 [room general] Alice joined (total=1)
2026/09/26 06:35:16 [room general] Bob joined (total=2)
2026/09/26 06:35:19 [room general] Bob left (total=1)
2026/09/26 06:35:19 [room general] Alice left (total=0)
```

---

## 11. Test อัตโนมัติ: จำลอง WebSocket client พร้อมกันหลายสิบตัว

การทดสอบด้วยมือในหัวข้อก่อนช่วยยืนยันความถูกต้องแบบผิวเผิน แต่โปรเจกต์ที่ดีต้องมี **automated test** ที่รันซ้ำได้ทุกครั้ง เชื่อมกับหลักการ testing ที่เรียนมาตั้งแต่ **Part 033-035** และ **Part 079-081 (httptest)** ไฟล์ `chat_test.go` ใช้ `httptest.NewServer` ห่อ `chat.NewServer(hub)` แล้วเปิด WebSocket connection จริงเข้าไปทดสอบ (ไม่ mock อะไรเลย — ทดสอบ stack ทั้งหมดตั้งแต่ HTTP upgrade จนถึง JSON message)

### Helper functions

```go
package chat

import (
	"encoding/json"
	"fmt"
	"net/http/httptest"
	"strings"
	"sync"
	"testing"
	"time"

	"github.com/gorilla/websocket"
)

// testDial เปิด WebSocket connection ไปยัง test server ในห้อง room ด้วยชื่อ user
// ที่กำหนด และคืน connection ให้ผู้เรียกจัดการเอง (ปิดเองด้วย conn.Close())
func testDial(t *testing.T, server *httptest.Server, room, user string) *websocket.Conn {
	t.Helper()
	wsURL := "ws" + strings.TrimPrefix(server.URL, "http") +
		fmt.Sprintf("/ws?room=%s&user=%s", room, user)
	conn, _, err := websocket.DefaultDialer.Dial(wsURL, nil)
	if err != nil {
		t.Fatalf("dial %s failed: %v", user, err)
	}
	return conn
}

// readMessage อ่านหนึ่งข้อความจาก conn โดยมี timeout กันเทสต์ค้างตลอดกาล
// ถ้า server มีบั๊กแล้วไม่ส่งอะไรกลับมาเลย
func readMessage(t *testing.T, conn *websocket.Conn) Message {
	t.Helper()
	conn.SetReadDeadline(time.Now().Add(3 * time.Second))
	_, data, err := conn.ReadMessage()
	if err != nil {
		t.Fatalf("read message failed: %v", err)
	}
	var msg Message
	if err := json.Unmarshal(data, &msg); err != nil {
		t.Fatalf("unmarshal message failed: %v", err)
	}
	return msg
}

// readUntil อ่านข้อความไปเรื่อยๆ จนกว่าจะเจอ Type ที่ต้องการ (ข้ามข้อความ
// ประเภทอื่นที่แทรกมาก่อน เช่น userlist ที่อัปเดตหลายครั้งติดกัน)
func readUntil(t *testing.T, conn *websocket.Conn, want MessageType) Message {
	t.Helper()
	for i := 0; i < 20; i++ {
		msg := readMessage(t, conn)
		if msg.Type == want {
			return msg
		}
	}
	t.Fatalf("did not see message type %q within 20 messages", want)
	return Message{}
}
```

`t.Helper()` (จาก **Part 033**) ทำให้ถ้า assertion ล้มเหลวใน helper พวกนี้ ข้อความ error จะชี้ไปที่บรรทัดที่ *เรียก* helper ในเทสต์จริง แทนที่จะชี้เข้ามาในไฟล์ helper เอง ทำให้ debug ง่ายขึ้นมาก

### เทสต์ที่ 1: join แล้วต้องได้ userlist ที่อัปเดตแบบ real-time

```go
func TestJoinBroadcastsUserList(t *testing.T) {
	hub := NewHub()
	server := httptest.NewServer(NewServer(hub))
	defer server.Close()

	alice := testDial(t, server, "general", "Alice")
	defer alice.Close()

	// Alice เข้าห้องคนแรก ต้องเห็น userlist ที่มีแค่ตัวเอง
	ul := readUntil(t, alice, TypeUserList)
	if len(ul.Users) != 1 || ul.Users[0] != "Alice" {
		t.Fatalf("expected userlist [Alice], got %v", ul.Users)
	}

	bob := testDial(t, server, "general", "Bob")
	defer bob.Close()

	// Alice ต้องได้ทั้ง join event ของ Bob และ userlist ใหม่ที่มี 2 คน
	joinMsg := readUntil(t, alice, TypeJoin)
	if joinMsg.User != "Bob" {
		t.Fatalf("expected join event from Bob, got %+v", joinMsg)
	}
	ul2 := readUntil(t, alice, TypeUserList)
	if len(ul2.Users) != 2 {
		t.Fatalf("expected 2 users after Bob joins, got %v", ul2.Users)
	}
}
```

### เทสต์ที่ 2: broadcast ต้องถึงทุกคนในห้อง

```go
func TestBroadcastReachesAllClientsInRoom(t *testing.T) {
	hub := NewHub()
	server := httptest.NewServer(NewServer(hub))
	defer server.Close()

	alice := testDial(t, server, "general", "Alice")
	defer alice.Close()
	bob := testDial(t, server, "general", "Bob")
	defer bob.Close()
	carol := testDial(t, server, "general", "Carol")
	defer carol.Close()

	// เคลียร์ join/userlist event ที่ค้างอยู่ก่อนของแต่ละ client ให้หมดก่อน
	// ด้วยการรอ userlist ที่มีครบ 3 คนจากทุก connection
	for _, c := range []*websocket.Conn{alice, bob, carol} {
		for {
			msg := readUntil(t, c, TypeUserList)
			if len(msg.Users) == 3 {
				break
			}
		}
	}

	if err := alice.WriteJSON(Message{Type: TypeMessage, Text: "สวัสดีทุกคน"}); err != nil {
		t.Fatalf("alice write failed: %v", err)
	}

	// Bob และ Carol ต้องได้รับข้อความของ Alice (Alice ไม่ได้รับข้อความของตัวเอง
	// ซ้ำสองครั้ง เพราะ broadcast ส่งให้ทุก client ในห้องรวมถึงคนส่งด้วย เป็น
	// เรื่องปกติของ chat app -- client ฝั่ง UI เป็นคนตัดสินใจว่าจะไม่ echo
	// ข้อความตัวเองซ้ำ)
	for _, c := range []*websocket.Conn{bob, carol} {
		msg := readUntil(t, c, TypeMessage)
		if msg.User != "Alice" || msg.Text != "สวัสดีทุกคน" {
			t.Fatalf("expected message from Alice, got %+v", msg)
		}
	}
}
```

### เทสต์ที่ 3: ห้องต้องแยกจากกันจริง (isolation)

```go
func TestRoomsAreIsolated(t *testing.T) {
	hub := NewHub()
	server := httptest.NewServer(NewServer(hub))
	defer server.Close()

	alice := testDial(t, server, "room-a", "Alice")
	defer alice.Close()
	dave := testDial(t, server, "room-b", "Dave")
	defer dave.Close()

	readUntil(t, alice, TypeUserList)
	readUntil(t, dave, TypeUserList)

	if err := alice.WriteJSON(Message{Type: TypeMessage, Text: "hello room-a"}); err != nil {
		t.Fatalf("write failed: %v", err)
	}

	// Dave อยู่คนละห้อง ต้อง "ไม่" เห็นข้อความของ Alice เลย -- เช็คด้วยการตั้ง
	// deadline สั้นๆ แล้วคาดหวังว่าจะ timeout (ไม่มีอะไรส่งมาจริงๆ)
	dave.SetReadDeadline(time.Now().Add(500 * time.Millisecond))
	_, _, err := dave.ReadMessage()
	if err == nil {
		t.Fatal("expected Dave to receive nothing from room-a, but got a message")
	}
}
```

เทสต์นี้สาธิตเทคนิคที่มีประโยชน์: การพิสูจน์ว่า "**ไม่ควรเกิดอะไรขึ้น**" ทำได้ด้วยการตั้ง `ReadDeadline` สั้นๆ แล้วยืนยันว่า `ReadMessage()` timeout จริง (คืน error) — ถ้าคาดหวังผิดและมีข้อความรั่วข้ามห้องมาจริง เทสต์จะ fail ทันทีเพราะ `err == nil`

### เทสต์ที่ 4: ออกจากห้องต้องอัปเดต userlist

```go
func TestLeaveRemovesFromUserList(t *testing.T) {
	hub := NewHub()
	server := httptest.NewServer(NewServer(hub))
	defer server.Close()

	alice := testDial(t, server, "general", "Alice")
	defer alice.Close()
	bob := testDial(t, server, "general", "Bob")

	readUntil(t, alice, TypeUserList) // Alice คนเดียว
	readUntil(t, alice, TypeJoin)     // Bob เข้ามา
	readUntil(t, alice, TypeUserList) // Alice+Bob

	bob.Close() // ปิดแบบไม่ส่ง close frame ที่สมบูรณ์ จำลอง client หลุดกะทันหัน

	leaveMsg := readUntil(t, alice, TypeLeave)
	if leaveMsg.User != "Bob" {
		t.Fatalf("expected leave event for Bob, got %+v", leaveMsg)
	}
	ul := readUntil(t, alice, TypeUserList)
	if len(ul.Users) != 1 || ul.Users[0] != "Alice" {
		t.Fatalf("expected userlist [Alice] after Bob left, got %v", ul.Users)
	}
}
```

`bob.Close()` ในเทสต์นี้ปิด TCP connection ตรงๆ **โดยไม่ทำ close handshake ที่สมบูรณ์แบบ WebSocket** จำลองสถานการณ์จริงที่ผู้ใช้ปิด browser tab กะทันหันหรือ network หลุด — ฝั่งเซิร์ฟเวอร์ต้องตรวจจับได้ผ่าน error จาก `ReadMessage()` ใน `readPump` (ตามที่อธิบายไว้ในหัวข้อ 6) แล้วเข้า cleanup path ตามปกติ ผลลัพธ์คือ Alice ยังคงได้รับ `leave` event และ `userlist` ที่อัปเดตถูกต้อง แม้ Bob จะไม่ได้ "บอกลา" อย่างสุภาพก็ตาม

### เทสต์ที่ 5: โหลดจริงจากหลาย client พร้อมกัน (เทสต์สำคัญที่สุดของบทนี้)

```go
// TestManyConcurrentClients จำลองโหลดที่มีหลาย client เชื่อมต่อและส่งข้อความ
// พร้อมกันจริงๆ (ไม่ใช่ทีละคน) เพื่อบีบให้ readPump/writePump/Room.run ของ
// หลาย goroutine ทำงานแข่งกันเยอะที่สุดเท่าที่จะทำได้ -- นี่คือเทสต์ที่ควรรัน
// คู่กับ `go test -race` เพื่อยืนยันว่าไม่มี data race ตามที่อธิบายใน
// Part 044 (Race Condition และ Race Detector)
func TestManyConcurrentClients(t *testing.T) {
	const numClients = 20
	const messagesPerClient = 5

	hub := NewHub()
	server := httptest.NewServer(NewServer(hub))
	defer server.Close()

	conns := make([]*websocket.Conn, numClients)
	for i := 0; i < numClients; i++ {
		conns[i] = testDial(t, server, "stress", fmt.Sprintf("user%02d", i))
	}
	defer func() {
		for _, c := range conns {
			c.Close()
		}
	}()

	// รอให้ทุกคนเข้าห้องครบก่อน (userlist สุดท้ายที่แต่ละ client เห็นควรมีครบ
	// numClients คน) กันไม่ให้ยังไม่ทันเข้าห้องแล้วเริ่มส่งข้อความ
	var wgJoin sync.WaitGroup
	for _, c := range conns {
		wgJoin.Add(1)
		go func(conn *websocket.Conn) {
			defer wgJoin.Done()
			for {
				msg := readUntil(t, conn, TypeUserList)
				if len(msg.Users) == numClients {
					return
				}
			}
		}(c)
	}
	wgJoin.Wait()

	// สำคัญ: ต้องเริ่ม "อ่าน" ของทุก client ให้พร้อมทำงานคู่ขนานไปกับตอนที่
	// กำลัง "ส่ง" เลย ไม่ใช่ส่งให้เสร็จทั้งหมดก่อนแล้วค่อยเริ่มอ่านทีหลัง
	// เพราะ client.send เป็น buffered channel ขนาดจำกัด (256) ถ้าไม่มีใครอ่าน
	// ระหว่างที่ยังส่งกันอยู่ Room.broadcastLocked จะเห็น buffer เต็มและตัด
	// client ตัวนั้นทิ้งไปเลย (ดู room.go) ซึ่งเป็นพฤติกรรมที่ถูกต้องของระบบ
	// จริง (backpressure) แต่ทำให้เทสต์นี้ต้องจำลองพฤติกรรม client จริงที่
	// อ่านและเขียนพร้อมกันตลอดเวลา ไม่ใช่แค่ยิงข้อความทิ้งแล้วค่อยมาเก็บทีหลัง
	want := numClients * messagesPerClient

	var wg sync.WaitGroup
	for i, c := range conns {
		wg.Add(2)

		go func(idx int, conn *websocket.Conn) {
			defer wg.Done()
			for j := 0; j < messagesPerClient; j++ {
				err := conn.WriteJSON(Message{
					Type: TypeMessage,
					Text: fmt.Sprintf("msg-%d-%d", idx, j),
				})
				if err != nil {
					t.Errorf("client %d write failed: %v", idx, err)
					return
				}
			}
		}(i, c)

		go func(conn *websocket.Conn) {
			defer wg.Done()
			got := 0
			deadline := time.Now().Add(10 * time.Second)
			for got < want && time.Now().Before(deadline) {
				conn.SetReadDeadline(time.Now().Add(10 * time.Second))
				_, data, err := conn.ReadMessage()
				if err != nil {
					return
				}
				var msg Message
				if err := json.Unmarshal(data, &msg); err != nil {
					continue
				}
				if msg.Type == TypeMessage {
					got++
				}
			}
			if got != want {
				t.Errorf("expected %d messages, got %d", want, got)
			}
		}(c)
	}
	wg.Wait()
}
```

เทสต์นี้เปิด client จริง 20 ตัวเข้าห้องเดียวกัน แล้วให้**ทุกตัวส่งข้อความพร้อมกันจริงๆ ผ่าน 40 goroutine คู่ขนาน** (20 goroutine ส่ง + 20 goroutine อ่าน) — จำนวนข้อความที่คาดหวังต่อ client คือ `20 × 5 = 100` ข้อความ (เพราะ broadcast กระจายให้ทุกคนในห้องรวมถึงตัวเอง) นี่คือเทสต์ที่ **บีบให้เกิด race มากที่สุด** ถ้ามีจุดไหนในโค้ดที่เข้าถึง shared state โดยไม่ synchronize ให้ถูกต้อง เทสต์นี้จะมีโอกาสสูงที่สุดที่จะทำให้ race detector (หัวข้อถัดไป) จับได้

**บันทึกจากการพัฒนาจริง**: ระหว่างพัฒนาโปรเจกต์นี้ เทสต์รุ่นแรกของ `TestManyConcurrentClients` เขียนให้ "ส่งให้เสร็จก่อน แล้วค่อยเริ่มอ่าน" (แยกสอง phase) ซึ่งรันผ่านตอนใช้ `go test` ธรรมดา แต่พอรันด้วย `go test -race -count=3` กลับ **fail จริง** เพราะภายใต้ overhead ของ race detector ที่หนักกว่าเดิม (การ instrument ทุก memory access) client ยังไม่ทันเริ่มอ่านข้อความ ในขณะที่อีกฝั่งส่งครบทั้ง 100 ข้อความเข้า buffer ขนาด 256 ของบาง client แล้ว ทำให้ `broadcastLocked` ตัดบาง client ทิ้งเพราะ buffer เต็มจริง (พฤติกรรมนี้ถูกต้องตามที่ออกแบบไว้! ไม่ใช่ bug) วิธีแก้คือปรับเทสต์ให้จำลองพฤติกรรม client จริงที่ **อ่านและเขียนพร้อมกันตลอดเวลา** (ตามโค้ดด้านบน) ซึ่งสะท้อนวิธีที่ client จริงทำงานอยู่แล้ว — บทเรียนนี้เป็นตัวอย่างจริงว่า **การเขียนเทสต์ concurrency ที่ดีต้องจำลองพฤติกรรมจริงของระบบ ไม่ใช่แค่เขียนให้ผ่านในเคสที่ง่ายที่สุด**

รันชุดเทสต์ทั้งหมดแบบละเอียด (`-v`) ได้ผลลัพธ์จริง:

```bash
go test ./... -v
```

```
?   	chatapp/cmd/server	[no test files]
=== RUN   TestJoinBroadcastsUserList
--- PASS: TestJoinBroadcastsUserList (0.01s)
=== RUN   TestBroadcastReachesAllClientsInRoom
--- PASS: TestBroadcastReachesAllClientsInRoom (0.00s)
=== RUN   TestRoomsAreIsolated
--- PASS: TestRoomsAreIsolated (0.50s)
=== RUN   TestLeaveRemovesFromUserList
--- PASS: TestLeaveRemovesFromUserList (0.00s)
=== RUN   TestManyConcurrentClients
--- PASS: TestManyConcurrentClients (0.04s)
=== RUN   TestRecoverGuardPattern
--- PASS: TestRecoverGuardPattern (0.00s)
PASS
ok  	chatapp/internal/chat	1.568s
```

ครบ 6 เทสต์ ผ่านทั้งหมด

---

## 12. ยืนยันด้วย Race Detector: `go test -race`

การที่เทสต์ผ่านด้วย `go test` ธรรมดา **ไม่ได้แปลว่าไม่มี data race** — ตามที่เรียนไว้ใน **Part 044** race condition มักไม่แสดงอาการทุกครั้งที่รัน เพราะขึ้นอยู่กับจังหวะการ schedule ของ goroutine ซึ่งเปลี่ยนไปในแต่ละครั้งที่รัน ต้องใช้ **race detector** (`-race` flag) ที่ instrument ทุกการเข้าถึงหน่วยความจำตอน runtime เพื่อจับ race ที่แฝงอยู่จริงๆ

รันด้วย `-race`:

```bash
go test ./... -race -v
```

```
?   	chatapp/cmd/server	[no test files]
=== RUN   TestJoinBroadcastsUserList
--- PASS: TestJoinBroadcastsUserList (0.01s)
=== RUN   TestBroadcastReachesAllClientsInRoom
--- PASS: TestBroadcastReachesAllClientsInRoom (0.00s)
=== RUN   TestRoomsAreIsolated
--- PASS: TestRoomsAreIsolated (0.50s)
=== RUN   TestLeaveRemovesFromUserList
--- PASS: TestLeaveRemovesFromUserList (0.00s)
=== RUN   TestManyConcurrentClients
--- PASS: TestManyConcurrentClients (0.05s)
=== RUN   TestRecoverGuardPattern
--- PASS: TestRecoverGuardPattern (0.00s)
PASS
ok  	chatapp/internal/chat	1.583s
```

ไม่มี `WARNING: DATA RACE` ปรากฏเลย — สะอาด 100% แต่เพราะ race มีธรรมชาติที่ไม่แสดงอาการทุกครั้ง จึงรันซ้ำอีกหลายรอบด้วย `-count` เพื่อเพิ่มโอกาสเจอ race ที่อาจซ่อนอยู่ (rerun ทั้งชุดเทสต์ในโปรเซสเดียวกันหลายรอบติดกัน เพิ่มความหลากหลายของจังหวะ scheduling):

```bash
go test ./... -race -count=10
```

```
?   	chatapp/cmd/server	[no test files]
ok  	chatapp/internal/chat	6.449s
```

รัน `TestManyConcurrentClients` (เทสต์ที่หนักที่สุด) ซ้ำ **10 รอบติดกันภายใต้ race detector สะอาดหมด** — เป็นหลักฐานที่หนักแน่นว่าการออกแบบ concurrency ในบทนี้ถูกต้องจริง ไม่ใช่แค่ "บังเอิญไม่ชนกันตอนรัน"

> **หมายเหตุจากการพัฒนาจริง**: ตัวเลข buffer `256` ที่ใช้ใน `client.send` (หัวข้อ 8) ถูกปรับขึ้นจาก 16 ก็เพราะการรัน `-race -count=3` ในระหว่างพัฒนาจริงเจอเทสต์ fail (ไม่ใช่ data race — เป็น assertion fail เพราะ client ถูกตัดออกจากห้องจริงตามที่ออกแบบไว้ เนื่องจาก buffer เล็กเกินไปสำหรับโหลดของเทสต์) นี่คือตัวอย่างที่ดีว่า **การรัน `-race` ซ้ำหลายรอบไม่ได้มีประโยชน์แค่จับ data race แต่ยังช่วยเปิดโปงเงื่อนไข edge case ด้าน backpressure ที่ยากจะสังเกตเห็นตอนรันแบบปกติ** เพราะ race detector ทำให้ทุกอย่างช้าลงและ schedule แปลกไปจากปกติ จนไปกระตุกเงื่อนไขที่ไม่ค่อยเกิด

---

## 13. วิเคราะห์ความปลอดภัยของ concurrency ทีละจุด

มาสรุปกันทีละจุดว่า **shared state ทุกจุดในระบบนี้ถูกป้องกันด้วยอะไร** — เป็นแบบฝึกหัดการวิเคราะห์ที่ควรทำกับทุกระบบ concurrent ก่อนขึ้น production จริง (แนวทางเดียวกับที่ต้องทำก่อนเขียนโค้ดใน Part 039, 040, 044)

| Shared state | ใครเข้าถึงได้บ้าง | กลไกป้องกัน | ทำไมถึงปลอดภัย |
|---|---|---|---|
| `Hub.rooms` (map) | HTTP handler ของทุก connection ใหม่ (หลาย goroutine พร้อมกัน), `reapLoop` | `sync.RWMutex` | ทุกการอ่าน/เขียนผ่าน `RLock`/`Lock` เสมอ ไม่มีที่ไหนแตะ `h.rooms` ตรงๆ นอก `hub.go` |
| `Room.clients` (map) | `readPump` ของทุก client ในห้อง (ผ่าน channel), `Hub` (ตอนสร้างห้อง) | goroutine เดียว (`run()`) เป็นเจ้าของ ไม่มี mutex | ไม่มี goroutine อื่นแตะ map นี้ตรงๆ เลย ทุกอย่างส่งผ่าน channel `register`/`unregister`/`broadcast` |
| `*websocket.Conn` ของแต่ละ client | `writePump` (เขียน), `readPump` (อ่าน) | แยก reader/writer คนละ goroutine ตายตัว + single writer เขียนผ่าน `send` channel เท่านั้น | มีแค่ `writePump` เท่านั้นที่เรียก `WriteMessage` ตรงกับกฎ "one reader + one writer" ของ gorilla/websocket (Part 069 หัวข้อ 7) |
| `Client.send` (channel) | `Room.broadcastLocked` (ส่ง), `writePump` (รับ), `Room.run()` (ปิด) | ธรรมชาติของ channel เอง (thread-safe โดย design ของภาษา) | Go รับประกันว่า send/receive/close บน channel เดียวกันจากหลาย goroutine ปลอดภัยเสมอ (ไม่ต้องมี mutex เพิ่ม) |
| `Message` แต่ละก้อน | สร้างใหม่ทุกครั้งใน `readPump`/`broadcastLocked` ไม่มีการแชร์ struct เดียวกันข้าม goroutine | Value semantics — ส่งเป็นค่า (หรือ `[]byte` ที่ marshal เสร็จแล้ว ไม่ mutate ซ้ำ) | ไม่มี state ที่แชร์ให้ race ได้ตั้งแต่แรก เพราะแต่ละข้อความ immutable หลัง marshal |

ข้อสังเกตสำคัญจากตารางนี้: **ไม่มีจุดไหนในระบบที่สอง goroutine เขียนตัวแปรเดียวกันพร้อมกันโดยไม่มี synchronization เลยแม้แต่จุดเดียว** — นี่คือเป้าหมายของการออกแบบ concurrent system ที่ดี ไม่ใช่แค่ "ใส่ mutex ให้เยอะเข้าไว้" แต่คือ **คิดล่วงหน้าว่า state แต่ละก้อนควรมีเจ้าของกี่ goroutine** แล้วเลือกกลไกที่เหมาะสมที่สุดสำหรับ state ก้อนนั้น (mutex, channel-owned-by-one-goroutine, หรือ immutable value)

จุดที่มักเป็นบั๊กในโปรเจกต์ chat ที่เขียนแบบรีบๆ (ที่บทนี้หลีกเลี่ยงไว้ตั้งแต่แรก) ได้แก่:

- **เขียน `WriteMessage` จากหลาย goroutine ตรงๆ** — บทนี้เลี่ยงด้วยการบังคับให้มีแค่ `writePump` goroutine เดียวเท่านั้นที่แตะ `conn.WriteMessage`
- **วน `for c := range clients` พร้อมกับ `register`/`unregister` แก้ map เดียวกันจากอีก goroutine** — บทนี้เลี่ยงด้วยการให้ `Room.run()` เป็นเจ้าของ map แต่เพียงผู้เดียว ทุกการแก้ไขเกิดใน goroutine เดียวเสมอ
- **ลืม synchronize การอ่านจำนวน client ปัจจุบัน** (เช่น endpoint `/rooms` ที่ต้องอ่าน `len(hub.rooms)`) — บทนี้เลี่ยงด้วย `RoomNames()` ที่ขอ `RLock` ก่อนอ่านเสมอ

---

## 14. ข้อจำกัดของ in-memory hub และแนวทาง scale ด้วย Redis Pub/Sub

ระบบที่สร้างในบทนี้ทำงานได้ดีมาก **ตราบใดที่มีเซิร์ฟเวอร์ instance เดียว** — `Hub` เก็บทุกอย่างไว้ในหน่วยความจำของ process เดียว (`map[string]*Room` ที่มี `map[*Client]bool` อยู่ข้างในอีกที) รองรับผู้ใช้พร้อมกันได้หลักพันคนสบายๆ ด้วยเครื่องเดียว (ตามที่ประเมินไว้ใน Part 069 หัวข้อ 12)

แต่ทันทีที่ระบบต้อง **scale แนวนอน** (รันหลาย instance หลัง load balancer เพื่อรองรับผู้ใช้มากขึ้นหรือเพิ่ม availability) จะเจอปัญหาทันที:

```
Client A (Alice) ──ws──> Instance 1 ── Hub 1 (มีแค่ Alice)
Client B (Bob)   ──ws──> Instance 2 ── Hub 2 (มีแค่ Bob)
```

ถ้า Alice กับ Bob ถูก load balancer ส่งไปคนละ instance กัน (ซึ่งเป็นเรื่องปกติมากเมื่อมีหลาย instance) **Alice จะไม่มีทางเห็นข้อความของ Bob เลย แม้ทั้งคู่จะคิดว่าอยู่ในห้อง "general" เดียวกัน** เพราะ `Hub` ของ instance 1 กับ instance 2 เป็นคนละ process เป็นคนละหน่วยความจำ ไม่รู้จักกันเลย — Room ของ instance 1 ไม่มีทางรู้ว่า Room ชื่อเดียวกันบน instance 2 มีใครอยู่บ้าง

### ทางแก้: Redis Pub/Sub เป็น message broker กลาง

แนวทางมาตรฐานที่อธิบายไว้แล้วใน **Part 069 หัวข้อ 12** และจะเรียนเจาะลึกการต่อ Redis จริงใน **Part 077** คือเพิ่มชั้น **message broker กลาง** ที่ทุก instance เชื่อมต่อร่วมกัน:

```
Client A ──ws──> Instance 1 ──PUBLISH──┐
                                         ├──> Redis channel "room:general"
Client B ──ws──> Instance 2 <─SUBSCRIBE┘
```

แนวคิดคร่าวๆ ในเชิงโค้ด (สเก็ตช์แสดงแนวคิด **ไม่ implement เต็มในบทนี้** เพราะต้องมี Redis server จริงรันคู่ด้วยจึงจะทดสอบได้ ซึ่งเป็นเนื้อหาเต็มรูปแบบของ Part 077):

```go
// แนวคิดคร่าวๆ ของ Room ที่ผูกกับ Redis Pub/Sub เพื่อ scale ข้าม instance
// (โค้ดสเก็ตช์ ไม่ใช่โค้ดที่ build ผ่านจริงในบทนี้ -- ดู Part 077 สำหรับ
// การต่อ go-redis แบบเต็มรูปแบบ)
type DistributedRoom struct {
	Room                       // ยังใช้ event loop เดิมสำหรับ client ในเครื่องตัวเอง
	redisClient *redis.Client
	pubsub      *redis.PubSub
}

func (r *DistributedRoom) broadcastLocked(msg Message) {
	data, _ := json.Marshal(msg)

	// ขั้นตอนที่เพิ่มเข้ามา: publish ข้อความไปที่ Redis channel กลางด้วย
	// เพื่อให้ instance อื่นที่ subscribe ห้องเดียวกันอยู่ได้รับข้อความนี้ไปด้วย
	r.redisClient.Publish(ctx, "room:"+r.name, data)

	// ยังคงส่งให้ client ในเครื่องตัวเองตามปกติเหมือนเดิมทุกประการ
	for c := range r.clients {
		select {
		case c.send <- data:
		default:
			delete(r.clients, c)
			close(c.send)
		}
	}
}

// รันแยกต่างหากตอนเปิดห้อง: subscribe channel เดียวกันไว้ตลอดเวลา
// เมื่อมีข้อความมาจาก Redis (ไม่ว่าจะ publish มาจาก instance ตัวเองหรือ
// instance อื่น) ให้ส่งต่อเข้า c.send ของ client ทุกตัวที่ต่ออยู่กับ
// instance นี้เท่านั้น (instance อื่นจัดการ client ของตัวเองแยกกันไป)
func (r *DistributedRoom) subscribeLoop() {
	ch := r.pubsub.Channel()
	for redisMsg := range ch {
		var msg Message
		json.Unmarshal([]byte(redisMsg.Payload), &msg)
		for c := range r.clients {
			select {
			case c.send <- []byte(redisMsg.Payload):
			default:
			}
		}
	}
}
```

หลักการสำคัญ: **instance แต่ละตัวไม่จำเป็นต้องรู้จักกันโดยตรงเลย รู้จักแค่ Redis ตัวกลาง** เมื่อ instance ไหน publish ข้อความเข้า channel `"room:general"` ทุก instance ที่ subscribe channel เดียวกันอยู่ (รวมถึงตัวเองที่ publish ไปด้วย) จะได้รับข้อความกลับมา แล้วแต่ละ instance ก็ค่อยกระจายต่อให้ client ในเครื่องตัวเองตามปกติ

รูปแบบนี้เรียกว่า **pub/sub (publish/subscribe)** — สังเกตว่าเป็นแนวคิดเดียวกับที่ `Room.broadcastLocked` ทำอยู่แล้วในบทนี้ (publisher หนึ่งตัวกระจายให้ subscriber หลายตัว) เพียงแต่ขยายขอบเขตจาก "ภายใน process เดียว" ไปเป็น "ข้าม process ผ่าน Redis" เท่านั้นเอง — เป็นตัวอย่างที่ดีว่าทำไม pattern เดียวกันถึงเจอซ้ำในหลายระดับของสถาปัตยกรรม ตั้งแต่ระดับ in-process (**Part 045: Pub/Sub pattern**) ไปจนถึงระดับ distributed system (**Part 077: Redis**, **Part 091-092: RabbitMQ/Kafka**)

> **ข้อคิดสำคัญที่ย้ำอีกครั้ง (ตรงกับที่กล่าวไว้ใน Part 069)**: อย่าเริ่มออกแบบระบบด้วยความซับซ้อนแบบ multi-instance ตั้งแต่วันแรกถ้ายังไม่จำเป็น hub แบบในหน่วยความจำเดียวของบทนี้รองรับโหลดจริงได้เยอะกว่าที่คิด ค่อยเพิ่มชั้น Redis Pub/Sub เมื่อวัดผลจริงแล้วว่าเซิร์ฟเวอร์ตัวเดียวไม่พอ — การเพิ่มความซับซ้อนก่อนเวลาอันควร (premature scaling) มักทำให้ระบบดูแลยากขึ้นโดยไม่ได้ประโยชน์อะไรกลับมาเลย

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- สร้าง real-time chat application แบบ multi-room เต็มรูปแบบด้วย `gorilla/websocket` ที่ build, vet, test (รวม `-race`) ผ่านจริงทั้งหมด และรันจริงกับ client สองตัวจนเห็นข้อความไหลถึงกันแบบ real-time
- `Room` ใช้ **event loop เดียว (`run()`) เป็นเจ้าของ state** สื่อสารกับภายนอกผ่าน channel เท่านั้น (`register`/`unregister`/`broadcast`) ตรงกับหลักการ "share memory by communicating" ที่วางไว้ตั้งแต่ Part 001 และ Part 037
- `Hub` ใช้ **`sync.RWMutex`** จัดการหลายห้องพร้อมกัน พร้อม double-checked locking กันสร้างห้องซ้ำ (Part 039) — เห็นทั้งสองเทคนิค concurrency เทียบกันในโปรเจกต์เดียว และเลือกใช้ตามความเหมาะสมของแต่ละจุด ไม่ใช่ยึดติดวิธีเดียว
- `Client` แยก **`readPump`/`writePump` คนละ goroutine** ตามกฎ "one reader + one writer" ของ `gorilla/websocket` (Part 069) โดยมีแค่ `writePump` เท่านั้นที่เขียนลง socket ตรงๆ
- `defer` + `recover()` ในทุก goroutine ของ client ป้องกันไม่ให้ panic จาก client ตัวเดียวทำให้ **ทั้งโปรเซสล้มตายไปด้วย** (Part 017) — ยืนยันด้วยเทสต์ `TestRecoverGuardPattern` ที่พิสูจน์ว่า idiom นี้ทำงานถูกต้องจริง
- `Message` ออกแบบเป็น envelope เดียวที่มี `Type` แยกแยะ event (message/join/leave/userlist/error) และเซิร์ฟเวอร์เป็นคนกำหนด field ที่มีผลต่อ identity (`User`, `Room`, `Time`) เสมอ ไม่เชื่อค่าจาก client
- หน้าเว็บ client เป็น static HTML+JS ธรรมดา ฝังเข้า binary ด้วย `embed.FS` (Part 066) แทน `html/template` (Part 065) เพราะไม่มีข้อมูลฝั่งเซิร์ฟเวอร์ที่ต้อง render เข้า HTML
- เทสต์ `TestManyConcurrentClients` จำลอง client จริง 20 ตัวส่ง/รับข้อความพร้อมกันจริง ผ่านทั้ง `go test` ปกติและ `go test -race -count=10` โดยไม่มี data race แม้แต่ครั้งเดียว
- ระหว่างพัฒนาจริง เจอและแก้ปัญหาจริงสองเรื่อง: (1) buffer ของ `send` channel เล็กเกินไปทำให้ client ถูกตัดทิ้งโดยไม่จำเป็นภายใต้โหลดสูง ต้องปรับจาก 16 เป็น 256 และ (2) การออกแบบเทสต์ concurrency ต้องจำลองพฤติกรรมอ่าน/เขียนพร้อมกันของ client จริง ไม่ใช่แยกเป็น phase ส่งก่อนอ่านทีหลัง
- วิเคราะห์ทุกจุดที่มี shared state ในระบบอย่างละเอียด ยืนยันว่าไม่มีจุดไหนถูกเข้าถึงพร้อมกันโดยไม่มี synchronization
- in-memory hub รองรับได้แค่เซิร์ฟเวอร์ตัวเดียว การ scale ข้ามหลาย instance ต้องอาศัย **Redis Pub/Sub** เป็น message broker กลาง (สเก็ตช์แนวคิดไว้ในบทนี้ รายละเอียดเต็มรูปแบบอยู่ใน Part 077) — และไม่ควรเพิ่มความซับซ้อนนี้ก่อนวัดผลจริงว่าจำเป็น

## แบบฝึกหัดท้ายบท

1. เพิ่มฟีเจอร์ **ข้อความส่วนตัว (direct message)**: ผู้ใช้ส่งข้อความหาผู้ใช้อีกคนในห้องเดียวกันโดยตรง (เช่น `{"type":"dm","to":"Bob","text":"..."}`) โดยที่คนอื่นในห้องไม่เห็นข้อความนี้เลย ต้องแก้ทั้ง `Message` struct, `Room.run()` (เพิ่ม case ใหม่ใน select หรือเพิ่ม field แยกแยะปลายทาง), และเขียนเทสต์ยืนยันว่าคนที่สามในห้องไม่ได้รับข้อความ DM
2. เพิ่ม **การเก็บประวัติข้อความ (message history)**: เมื่อ client เข้าห้องใหม่ ให้ส่งข้อความ 20 รายการล่าสุดของห้องนั้นกลับไปก่อน (เก็บใน slice ในหน่วยความจำของ `Room` ก็พอสำหรับแบบฝึกหัดนี้ ยังไม่ต้องต่อฐานข้อมูล) ให้คิดว่าจะแก้ปัญหา "ข้อมูลหายเมื่อเซิร์ฟเวอร์ restart" ต่ออย่างไรด้วย `database/sql` (Part 071) หรือ MongoDB (Part 076)
3. ต่อยอดจากข้อ 2: เปลี่ยนจากเก็บใน slice ในหน่วยความจำ เป็นเขียนทุกข้อความลง PostgreSQL จริงด้วย `pgx` (Part 072) แล้วดึงประวัติจากฐานข้อมูลแทน ทดสอบว่าประวัติยังอยู่ครบหลัง restart เซิร์ฟเวอร์
4. Implement ชั้น **Redis Pub/Sub** ที่สเก็ตช์ไว้ในหัวข้อ 14 ให้ทำงานได้จริง (ต้องมี Redis server รันอยู่ เช่นผ่าน Docker) แล้วทดสอบด้วยการรันเซิร์ฟเวอร์แชทสองตัวอย่างพร้อมกันคนละพอร์ต (จำลองสอง instance) แล้วยืนยันว่า client ที่ต่ออยู่คนละ instance กันยังคุยกันได้ผ่านห้องเดียวกัน
5. เพิ่ม endpoint `/rooms/{name}/history` ที่ตอบ HTTP ธรรมดา (ไม่ใช่ WebSocket) คืนจำนวนข้อความทั้งหมดและผู้ใช้ที่เคยพูดในห้องนั้น ใช้ทดสอบด้วย `httptest` (Part 081)
6. เพิ่มการจำกัดอัตราการส่งข้อความ (rate limiting) ต่อ client เช่น ไม่เกิน 5 ข้อความต่อวินาที ถ้าเกินให้ส่ง `TypeError` กลับและไม่ broadcast ข้อความนั้น ลองออกแบบด้วย `time.Ticker`/`golang.org/x/time/rate` แล้วเขียนเทสต์จำลอง client ที่ส่งถี่เกินขีดจำกัดยืนยันว่าถูกบล็อกจริง

---

**ต่อไป**: [Part 105 — โปรเจกต์ URL Shortener พร้อม Caching](./105-project-url-shortener.md)
