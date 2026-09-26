# Part 070: GraphQL ด้วย Go

> ภาคที่ 5: Web Development — ตอนที่ 15 จาก 15 (Part 56–70)

## สารบัญของบทนี้

1. GraphQL คืออะไร แก้ปัญหาอะไรที่ REST มี
2. เปรียบเทียบแนวคิด: REST vs GraphQL
3. เลือก library: `gqlgen` แบบ Schema-First
4. ติดตั้งและสร้างโปรเจกต์เริ่มต้นด้วย `gqlgen init`
5. อ่านไฟล์ที่ gqlgen สร้างให้: schema, model, resolver
6. เขียน Schema ของเราเอง: Todo List
7. Generate โค้ดด้วย `gqlgen generate`
8. Implement Resolver จริง
9. รันเซิร์ฟเวอร์และทดสอบผ่าน Playground/curl
10. GraphQL Subscriptions: ข้อมูล Real-time ผ่าน WebSocket
11. เมื่อไหร่ GraphQL คุ้มค่ากับความซับซ้อน เมื่อไหร่มันเกินความจำเป็น
12. สรุปสิ่งที่ได้เรียนในบทนี้
13. แบบฝึกหัดท้ายบท

---

## 1. GraphQL คืออะไร แก้ปัญหาอะไรที่ REST มี

ตลอด **Part 059-062** เราสร้าง REST API กันมาเยอะแล้ว — แต่ละ resource มี endpoint ของตัวเอง (`/users`, `/users/{id}/orders`) และแต่ละ endpoint ตอบข้อมูลในรูปแบบที่กำหนดไว้ตายตัว **GraphQL** (พัฒนาโดย Facebook เปิดตัวปี 2015) เป็นแนวทางที่ต่างออกไปโดยสิ้นเชิง: มัน**ไม่ใช่ protocol ทดแทน HTTP** แต่เป็น **query language สำหรับ API** ที่แก้ปัญหาสองเรื่องหลักที่ REST เจอบ่อยในระบบขนาดใหญ่:

### ปัญหาที่ 1: Over-fetching (ได้ข้อมูลเกินที่ต้องการ)

สมมติหน้าจอ mobile app ต้องการแค่ชื่อกับรูปโปรไฟล์ของผู้ใช้ แต่ endpoint REST `/users/1` ถูกออกแบบมาให้ตอบข้อมูลผู้ใช้ทั้งหมด (email, ที่อยู่, เบอร์โทร, ประวัติการสั่งซื้อ ฯลฯ) — mobile app ต้องดาวน์โหลดข้อมูลทั้งหมดนั้นทั้งที่ใช้จริงแค่สอง field เปลืองทั้ง bandwidth และเวลาโดยเฉพาะบนเน็ตมือถือที่ช้า

### ปัญหาที่ 2: Under-fetching (ได้ข้อมูลไม่พอ ต้องยิงหลาย request)

สมมติต้องการแสดงหน้ารายการ "โพสต์พร้อมชื่อผู้เขียนแต่ละโพสต์" ด้วย REST มักต้องยิง `/posts` ก่อนเพื่อได้ post ID กับ author ID มา แล้วยิง `/users/{id}` ซ้ำอีกทีสำหรับผู้เขียนแต่ละคน (หรือต้องออกแบบ endpoint พิเศษ `/posts?include=author` ที่ผูกกับ use case เดียวนี้โดยเฉพาะ) กลายเป็นต้องยิงหลาย request หรือสร้าง endpoint เฉพาะกิจเพิ่มขึ้นเรื่อยๆ ตามหน้าจอที่ต้องการ

### แนวทางของ GraphQL

GraphQL แก้ทั้งสองปัญหาด้วยแนวคิดเดียว: **client เป็นคนกำหนดเองว่าต้องการ field อะไรบ้าง ในคำขอเดียว ผ่าน endpoint เดียว**

```graphql
query {
  post(id: "1") {
    title
    author {
      name
    }
  }
}
```

Request เดียวนี้ได้ทั้ง `title` ของโพสต์และ `name` ของผู้เขียนกลับมาในคำตอบเดียวกัน (แก้ under-fetching) และไม่มี field อื่นที่ไม่ได้ขอเช่น `email`, `createdAt` ปนมาเลย (แก้ over-fetching)

---

## 2. เปรียบเทียบแนวคิด: REST vs GraphQL

| | REST | GraphQL |
|---|---|---|
| จำนวน endpoint | หลาย endpoint ตาม resource (`/users`, `/posts`, `/posts/{id}/comments`) | **endpoint เดียว** เสมอ (มักเป็น `/query` หรือ `/graphql`) |
| ใครกำหนดรูปแบบข้อมูลตอบกลับ | เซิร์ฟเวอร์ (ตายตัวต่อ endpoint) | **client** (เลือก field เองทุกครั้ง) |
| Type system | ไม่บังคับ (มักอธิบายแยกด้วย OpenAPI/Swagger) | **บังคับมี schema ที่ type ชัดเจนเสมอ** เป็นส่วนหนึ่งของตัวภาษา |
| Over/Under-fetching | เกิดได้บ่อย | แก้ปัญหานี้โดยตรง |
| Caching ด้วย HTTP cache มาตรฐาน | ทำง่าย (ใช้ URL + HTTP cache header ได้ตรงๆ) | ทำยากกว่า (ทุก request เป็น POST ไป endpoint เดียวกัน ต้องทำ caching เองในชั้น application) |
| ความซับซ้อนฝั่งเซิร์ฟเวอร์ | ต่ำกว่า เข้าใจง่าย | สูงกว่า ต้องคิดเรื่อง schema, resolver, N+1 query problem |
| เหมาะกับ | CRUD ตรงไปตรงมา, ระบบที่ client ต้องการข้อมูลคล้ายกันหมด | หลาย client ที่ต้องการข้อมูลรูปแบบต่างกันมาก (web, mobile, ปุ่ม widget เล็กๆ) |

จุดที่สำคัญที่สุดคือ **GraphQL เป็น schema-first โดยธรรมชาติ** — ก่อนเขียนโค้ดสักบรรทัด เราต้องนิยาม "รูปร่างข้อมูล" ทั้งหมดของระบบไว้ในไฟล์ schema ก่อนเสมอ คล้ายกับการเขียน `struct` ที่มี type ชัดเจนใน Go (Part 011) แต่ขยายมาถึงระดับ API ทั้งระบบ

---

## 3. เลือก library: `gqlgen` แบบ Schema-First

Go ไม่มี GraphQL ให้ใน standard library เช่นเดียวกับ WebSocket ใน Part 069 — ต้องพึ่ง third-party library ตัวที่นิยมที่สุดในระบบนิเวศ Go คือ **[`99designs/gqlgen`](https://github.com/99designs/gqlgen)**

จุดเด่นของ `gqlgen` คือแนวทาง **schema-first code generation**:

1. เราเขียนไฟล์ `.graphqls` นิยาม schema ก่อน (type, query, mutation ทั้งหมด)
2. รันคำสั่ง `gqlgen generate`
3. `gqlgen` **อ่าน schema แล้ว generate โค้ด Go ที่ type-safe ให้อัตโนมัติ** — ทั้ง struct ของ model และ interface ของ resolver ที่เราต้อง implement
4. เราแค่เติม **logic จริง** ลงใน resolver ที่ generate มาให้เป็นโครงไว้แล้ว (ไม่ต้องเขียน parsing, validation ของ GraphQL query เองเลย)

แนวทางนี้ตรงข้ามกับ "code-first" (เขียน struct/resolver ก่อนแล้วให้ library อนุมาน schema จากโค้ด) ซึ่งบาง library ในภาษาอื่นใช้ ข้อดีของ schema-first คือ **schema เป็นสัญญา (contract) ที่ชัดเจนระหว่าง frontend กับ backend** ทีม frontend อ่านไฟล์ `.graphqls` แล้วรู้ทันทีว่า API มีอะไรบ้าง โดยไม่ต้องไปไล่อ่านโค้ด Go เลย

---

## 4. ติดตั้งและสร้างโปรเจกต์เริ่มต้นด้วย `gqlgen init`

```bash
mkdir graphql-example && cd graphql-example
go mod init graphql-example
go get github.com/99designs/gqlgen
go run github.com/99designs/gqlgen init
```

> **หมายเหตุเรื่องเวอร์ชัน**: `gqlgen` เวอร์ชันล่าสุดบางรุ่นกำหนดให้ต้องใช้ Go เวอร์ชันที่ใหม่กว่าที่ติดตั้งอยู่ในเครื่อง ถ้าเจอ error ทำนอง `requires go >= x.x.x` ให้ระบุเวอร์ชันของ `gqlgen` ที่รองรับ Go เวอร์ชันปัจจุบันของท่านตรงๆ เช่น `go get github.com/99designs/gqlgen@v0.17.60` (บทเรียนนี้ทดสอบและยืนยันผลจริงด้วย Go 1.24.7 และ `gqlgen v0.17.60`) หรือจะอัปเดต Go ให้เป็นเวอร์ชันล่าสุดก็แก้ปัญหาได้เช่นกัน

คำสั่ง `gqlgen init` สร้างโครงสร้างโปรเจกต์เริ่มต้นให้ทันที:

```
Creating gqlgen.yml
Creating graph/schema.graphqls
Creating server.go
Generating...

Exec "go run ./server.go" to start GraphQL server
```

ไฟล์และโฟลเดอร์ที่ได้:

```
graphql-example/
├── go.mod
├── go.sum
├── gqlgen.yml              # ไฟล์ config หลักของ gqlgen
├── server.go               # entry point ของเซิร์ฟเวอร์
└── graph/
    ├── schema.graphqls     # นิยาม schema (เราจะแก้ไฟล์นี้)
    ├── resolver.go         # struct Resolver (dependency injection ของเรา)
    ├── schema.resolvers.go # resolver ที่ต้อง implement logic จริง
    ├── generated.go        # โค้ดที่ gqlgen generate ให้ (ห้ามแก้มือ)
    └── model/
        └── models_gen.go   # struct ของ type ต่างๆ ที่ generate จาก schema
```

---

## 5. อ่านไฟล์ที่ gqlgen สร้างให้: schema, model, resolver

`gqlgen init` มาพร้อม schema ตัวอย่างที่จำลอง Todo list ไว้ให้แล้วใน `graph/schema.graphqls`:

```graphql
type Todo {
  id: ID!
  text: String!
  done: Boolean!
  user: User!
}

type User {
  id: ID!
  name: String!
}

type Query {
  todos: [Todo!]!
}

input NewTodo {
  text: String!
  userId: String!
}

type Mutation {
  createTodo(input: NewTodo!): Todo!
}
```

อ่าน syntax ทีละส่วน:

- **`type Todo { ... }`** — นิยาม object type ชื่อ `Todo` มี field 4 ตัว
- **`String!` / `ID!`** — เครื่องหมาย `!` หมายถึง **field นี้ห้ามเป็น null** (non-nullable) ถ้าไม่มี `!` แปลว่า field นั้นเป็น null ได้
- **`[Todo!]!`** — array ของ `Todo` ที่ทั้ง array เองและสมาชิกแต่ละตัวห้ามเป็น null
- **`type Query { ... }`** — นิยามว่า "อ่านข้อมูลอะไรได้บ้าง" เทียบเท่ากับ `GET` endpoint ทั้งหมดใน REST รวบมาไว้ที่เดียว
- **`type Mutation { ... }`** — นิยามว่า "แก้ไขข้อมูลอะไรได้บ้าง" เทียบเท่ากับ `POST`/`PUT`/`DELETE` ใน REST
- **`input NewTodo { ... }`** — `input` type ใช้เป็น parameter ของ query/mutation (ต่างจาก `type` ที่ใช้เป็นผลลัพธ์)

`gqlgen generate` (ที่รันไปแล้วตอน `init`) อ่าน schema นี้แล้วสร้าง Go struct ให้อัตโนมัติใน `graph/model/models_gen.go`:

```go
// Code generated by github.com/99designs/gqlgen, DO NOT EDIT.
package model

type NewTodo struct {
	Text   string `json:"text"`
	UserID string `json:"userId"`
}

type Todo struct {
	ID   string `json:"id"`
	Text string `json:"text"`
	Done bool   `json:"done"`
	User *User  `json:"user"`
}

type User struct {
	ID   string `json:"id"`
	Name string `json:"name"`
}
```

สังเกตว่า `Todo.User` เป็น `*User` ตรงๆ (ไม่ใช่แค่ ID) — เพราะ schema นิยาม `user: User!` ไว้ ถ้า field ไหนใน struct ตรงกับชื่อ field ใน schema พอดี `gqlgen` จะ bind ให้อัตโนมัติโดยไม่ต้องเขียน resolver แยกสำหรับ field นั้น (resolver จะถูก generate เฉพาะ field ที่ต้องมี logic เพิ่มเติม เช่น query จาก database)

ส่วน `graph/schema.resolvers.go` คือจุดที่เราต้องเติม logic เอง (ตอนแรก gqlgen ใส่ `panic("not implemented")` ไว้เป็น placeholder):

```go
func (r *mutationResolver) CreateTodo(ctx context.Context, input model.NewTodo) (*model.Todo, error) {
	panic(fmt.Errorf("not implemented: CreateTodo - createTodo"))
}

func (r *queryResolver) Todos(ctx context.Context) ([]*model.Todo, error) {
	panic(fmt.Errorf("not implemented: Todos - todos"))
}
```

---

## 6. เขียน Schema ของเราเอง: Todo List

บทเรียนนี้ใช้ schema ตัวอย่างเดิมที่ `gqlgen init` สร้างให้เลย (Todo list ที่ผูกกับ User) เพราะครอบคลุมแนวคิดสำคัญครบ: query, mutation, relationship ระหว่าง type, และ input type — เหมาะเป็นตัวอย่างเริ่มต้นที่ไม่ซับซ้อนเกินไป

หากต้องการเพิ่ม field ใหม่ เช่นวันที่สร้าง แก้ไขที่ `graph/schema.graphqls` แล้วรัน `gqlgen generate` ใหม่ — **workflow หลักของการพัฒนา GraphQL API ด้วย gqlgen คือวนซ้ำแบบนี้เสมอ**: แก้ schema → generate → เติม logic ใน resolver → ทดสอบ

---

## 7. Generate โค้ดด้วย `gqlgen generate`

ทุกครั้งที่แก้ไข `schema.graphqls` ต้องรันคำสั่งนี้ใหม่เพื่อให้โค้ด Go ที่ generate ไว้ตรงกับ schema ล่าสุด:

```bash
go run github.com/99designs/gqlgen generate
```

`gqlgen` จะ:

1. อ่านไฟล์ `.graphqls` ทั้งหมดตามที่ config ไว้ใน `gqlgen.yml`
2. อัปเดต `graph/model/models_gen.go` ให้ตรงกับ type ล่าสุด
3. อัปเดต `graph/generated.go` (executable schema — ตัว engine ที่แปลง GraphQL query จริงมาเรียก resolver ของเรา)
4. **merge** resolver ใหม่ที่ยังไม่มีเข้ากับ `schema.resolvers.go` โดย **ไม่ลบ logic ที่เราเขียนไว้แล้วในฟังก์ชันเดิม** — นี่คือเหตุผลที่ไฟล์นี้ปลอดภัยที่จะแก้ไขด้วยมือ ต่างจาก `generated.go` และ `models_gen.go` ที่มีคอมเมนต์ `DO NOT EDIT` กำกับไว้ชัดเจนและจะถูกเขียนทับทุกครั้ง

---

## 8. Implement Resolver จริง

ตอนนี้เติม logic จริงลงใน `graph/resolver.go` (เก็บ state/dependency) และ `graph/schema.resolvers.go` (logic การทำงาน) โดยใช้ slice ในหน่วยความจำแทนฐานข้อมูลจริงเพื่อความกระชับ (ในระบบจริงตรงนี้จะเรียก repository ที่ต่อฐานข้อมูล ตามที่จะเรียนใน**ภาคที่ 6: Database**):

`graph/resolver.go`:

```go
package graph

import (
	"sync"

	"graphql-example/graph/model"
)

// Resolver คือจุดที่เก็บ dependency ทั้งหมดที่ resolver ต้องใช้ (เช่น database
// connection, cache, repository ในของจริง) สำหรับตัวอย่างนี้เราใช้ slice
// ในหน่วยความจำแทนฐานข้อมูลเพื่อให้ตัวอย่างสั้นและรันได้ทันทีโดยไม่ต้องต่อ DB
type Resolver struct {
	mu         sync.Mutex
	todos      []*model.Todo
	users      map[string]*model.User
	nextTodoID int
}

func NewResolver() *Resolver {
	return &Resolver{
		users: map[string]*model.User{
			"1": {ID: "1", Name: "นัฐพงษ์"},
			"2": {ID: "2", Name: "สมศรี"},
		},
		nextTodoID: 1,
	}
}
```

`graph/schema.resolvers.go` (แก้เฉพาะ 2 ฟังก์ชันที่ gqlgen สร้างโครงไว้ให้):

```go
func (r *mutationResolver) CreateTodo(ctx context.Context, input model.NewTodo) (*model.Todo, error) {
	user, ok := r.users[input.UserID]
	if !ok {
		return nil, fmt.Errorf("ไม่พบผู้ใช้ userId=%s", input.UserID)
	}

	r.mu.Lock()
	defer r.mu.Unlock()

	todo := &model.Todo{
		ID:   strconv.Itoa(r.nextTodoID),
		Text: input.Text,
		Done: false,
		User: user,
	}
	r.nextTodoID++
	r.todos = append(r.todos, todo)
	return todo, nil
}

func (r *queryResolver) Todos(ctx context.Context) ([]*model.Todo, error) {
	r.mu.Lock()
	defer r.mu.Unlock()
	return r.todos, nil
}
```

สังเกตว่า **เราไม่ต้อง implement resolver แยกสำหรับ field `Todo.user`** เลย — เพราะ `model.Todo` มี field `User *model.User` ตรงตามชื่อใน schema พอดีอยู่แล้ว (ที่เราใส่ค่าไว้ตอนสร้าง `Todo` ใน `CreateTodo`) `gqlgen` จึงดึงค่าจาก struct field ตรงๆ ได้เลยโดยไม่ต้อง query อะไรเพิ่ม นี่คือตัวอย่างเล็กๆ ของสิ่งที่เรียกว่า **N+1 query problem** ในโลก GraphQL จริง — ถ้า `User` ต้อง query จากฐานข้อมูลแยกต่างหากสำหรับ Todo แต่ละตัว (ไม่ได้ preload มาด้วยตอน query todos) การขอ `todos { user { name } }` จะยิง query ไปฐานข้อมูล 1 ครั้งสำหรับ todos บวกอีก N ครั้งสำหรับ user ของแต่ละ todo ปัญหานี้แก้ด้วยเทคนิคที่เรียกว่า **dataloader** (batch และ cache การเรียก resolver ย่อยภายใน request เดียว) ซึ่งเป็นหัวข้อขั้นสูงที่ควรศึกษาต่อเมื่อเริ่มต่อฐานข้อมูลจริงตามภาคที่ 6

สุดท้ายแก้ `server.go` ให้ใช้ `NewResolver()` แทนการสร้าง `&graph.Resolver{}` เปล่าๆ:

```go
srv := handler.New(graph.NewExecutableSchema(graph.Config{Resolvers: graph.NewResolver()}))
```

---

## 9. รันเซิร์ฟเวอร์และทดสอบผ่าน Playground/curl

```bash
go run server.go
```

```
connect to http://localhost:8080/ for GraphQL playground
```

`gqlgen init` เตรียม **GraphQL Playground** ไว้ให้แล้วที่หน้าแรก (`/`) — เป็นหน้าเว็บ interactive ที่เขียน query ทดลองได้ทันทีพร้อม autocomplete จาก schema (คล้าย Postman แต่ออกแบบมาสำหรับ GraphQL โดยเฉพาะ) ส่วน endpoint จริงที่รับ query อยู่ที่ `/query`

### ทดสอบผ่าน curl (จำลองสิ่งที่ Playground ทำเบื้องหลัง)

Query — ตอนแรกยังไม่มี todo เลย:

```bash
curl -X POST http://localhost:8080/query \
  -H "Content-Type: application/json" \
  -d '{"query": "{ todos { id text done } }"}'
```

```json
{"data":{"todos":[]}}
```

Mutation — สร้าง todo ใหม่ โดยขอกลับมาแค่บาง field ที่ต้องการ (จุดเด่นของ GraphQL ที่เห็นชัดที่สุด):

```bash
curl -X POST http://localhost:8080/query \
  -H "Content-Type: application/json" \
  -d '{"query": "mutation { createTodo(input: {text: \"เรียน Go ให้จบ\", userId: \"1\"}) { id text user { name } } }"}'
```

```json
{"data":{"createTodo":{"id":"1","text":"เรียน Go ให้จบ","user":{"name":"นัฐพงษ์"}}}}
```

สังเกตว่า response มีแค่ `id`, `text`, และ `user.name` ตรงตาม field ที่ query ระบุไว้เป๊ะๆ ไม่มี `done` หรือ field อื่นปนมาเลย แม้ resolver จะคำนวณค่าทั้งหมดของ `Todo` ไว้ครบก็ตาม — **engine ของ `gqlgen` เป็นคนกรอง field ตาม query ให้เองโดยอัตโนมัติ** เราไม่ต้องเขียน logic กรองเองเลย

Query ซ้ำอีกครั้งหลังสร้างสำเร็จ:

```bash
curl -X POST http://localhost:8080/query \
  -H "Content-Type: application/json" \
  -d '{"query": "{ todos { id text done } }"}'
```

```json
{"data":{"todos":[{"id":"1","text":"เรียน Go ให้จบ","done":false}]}}
```

ทดสอบกรณี error — ส่ง `userId` ที่ไม่มีอยู่จริง:

```bash
curl -X POST http://localhost:8080/query \
  -H "Content-Type: application/json" \
  -d '{"query": "mutation { createTodo(input: {text: \"x\", userId: \"999\"}) { id } }"}'
```

```json
{"errors":[{"message":"ไม่พบผู้ใช้ userId=999","path":["createTodo"]}]}
```

สังเกตว่า **error ของ GraphQL ไม่ได้ตอบด้วย HTTP status code ผิดปกติ** (ยัง HTTP 200 อยู่) แต่ใส่รายละเอียด error ไว้ใน field `errors` ของ JSON response แทน พร้อมระบุ `path` ว่า error เกิดที่จุดไหนของ query — เป็นรูปแบบมาตรฐานของ GraphQL spec ที่ต่างจาก REST ที่มักใช้ HTTP status code (4xx, 5xx) สื่อความหมาย error

โค้ดทั้งหมดในบทนี้ผ่านการทดสอบจริงด้วย `httptest` ครบทุกกรณีข้างต้น (query ว่าง, mutation สำเร็จพร้อม field selection บางส่วน, query หลัง mutation, และกรณี error) รวมถึงหน้า playground เองก็ยืนยันแล้วว่าโหลดได้ถูกต้องผ่าน endpoint `/`

---

## 10. GraphQL Subscriptions: ข้อมูล Real-time ผ่าน WebSocket

นอกจาก `Query` (อ่านข้อมูล) และ `Mutation` (แก้ไขข้อมูล) แล้ว GraphQL ยังมี operation ประเภทที่สามคือ **`Subscription`** — ใช้สำหรับ **รับข้อมูลใหม่แบบ real-time ทุกครั้งที่มี event เกิดขึ้น** โดยไม่ต้อง poll ถามซ้ำๆ นี่คือจุดที่ GraphQL เชื่อมกับสิ่งที่เรียนไปใน **Part 069 (WebSocket)** โดยตรง: `gqlgen` ใช้ WebSocket เป็น transport เบื้องหลังของ subscription

เพิ่ม subscription ลงใน schema ได้แบบนี้:

```graphql
type Subscription {
  todoAdded: Todo!
}
```

หลังรัน `gqlgen generate` จะได้ resolver โครงร่างที่ต้อง implement คืนค่าเป็น **channel** แทนที่จะเป็นค่าเดียวแบบ Query/Mutation:

```go
func (r *subscriptionResolver) TodoAdded(ctx context.Context) (<-chan *model.Todo, error) {
	ch := make(chan *model.Todo, 1)

	r.mu.Lock()
	r.todoAddedSubscribers = append(r.todoAddedSubscribers, ch)
	r.mu.Unlock()

	// เมื่อ client ยกเลิกการเชื่อมต่อ (ปิด browser tab, network หลุด)
	// context จะถูก cancel — ต้องเคลียร์ subscriber ออกจาก list ด้วย ไม่งั้น goroutine ค้างตลอดไป
	go func() {
		<-ctx.Done()
		r.removeSubscriber(ch)
	}()

	return ch, nil
}
```

แล้วในจุดที่ `CreateTodo` สร้าง todo สำเร็จ ให้ส่งค่าเข้าไปในทุก channel ของ subscriber ที่รอฟังอยู่:

```go
func (r *mutationResolver) CreateTodo(ctx context.Context, input model.NewTodo) (*model.Todo, error) {
	// ... logic สร้าง todo เหมือนหัวข้อที่ 8 ...

	r.mu.Lock()
	for _, ch := range r.todoAddedSubscribers {
		select {
		case ch <- todo: // ส่งให้ subscriber ที่รอฟังอยู่ทันที
		default: // subscriber รับไม่ทัน ข้ามไปไม่บล็อก mutation หลัก
		}
	}
	r.mu.Unlock()

	return todo, nil
}
```

จากมุมมองของ client เวลา subscribe จะเห็นข้อความ `todoAdded` ทุกครั้งที่มีใครก็ตามสร้าง todo ใหม่ผ่าน mutation แบบ real-time โดยไม่ต้อง query ซ้ำเอง — คล้ายกับ WebSocket broadcast ใน Part 069 มาก เพียงแต่ห่อด้วย type system และ syntax ของ GraphQL ให้แล้ว

> **ข้อควรรู้**: Subscription เพิ่มความซับซ้อนขึ้นมาอีกระดับ (ต้องจัดการ connection lifecycle, cleanup subscriber, backpressure เหมือนที่เจอใน WebSocket hub ของ Part 069 ทุกประการ) จึงควรใช้เฉพาะ field ที่ต้องการ real-time จริงๆ เท่านั้น ไม่ใช่ทุก field ควรเป็น subscription

---

## 11. เมื่อไหร่ GraphQL คุ้มค่ากับความซับซ้อน เมื่อไหร่มันเกินความจำเป็น

หลังจากเห็นทั้งพลังและความซับซ้อนของ GraphQL แล้ว คำถามสำคัญคือ **"ควรใช้เมื่อไหร่"** — คำตอบตรงไปตรงมา: **ไม่ใช่ทุกโปรเจกต์ควรใช้ GraphQL**

### GraphQL คุ้มค่าเมื่อ:

- **มีหลาย client ที่ต้องการข้อมูลรูปแบบต่างกันมาก** — เช่น mobile app ที่ต้องประหยัด bandwidth, หน้าเว็บ dashboard ที่ต้องการข้อมูลละเอียดครบทุก field, widget เล็กๆ ที่ต้องการแค่ 2-3 field เป็น use case คลาสสิกที่ Facebook สร้าง GraphQL ขึ้นมาแก้ปัญหานี้โดยตรง
- **โครงสร้างข้อมูลซับซ้อนและเชื่อมโยงกันลึก** — เช่น social network ที่ post มี comment ซึ่งมี author ซึ่งมี follower ฯลฯ การ query ข้อมูลที่ซ้อนกันหลายชั้นแบบนี้ด้วย REST มักต้องยิงหลาย request หรือสร้าง endpoint เฉพาะกิจเยอะมาก
- **ทีม frontend หลายทีมใช้ backend เดียวกัน** — schema ที่ชัดเจนช่วยให้แต่ละทีมพัฒนาอิสระต่อกันได้โดยรู้ล่วงหน้าว่า API มีอะไรบ้าง

### GraphQL เกินความจำเป็นเมื่อ:

- **CRUD ธรรมดาที่ client มีแบบเดียว** — ถ้าทำแค่เว็บแอปเดียวที่ทุกหน้าใช้ข้อมูลคล้ายๆ กัน REST ธรรมดาก็เพียงพอและง่ายกว่ามาก
- **ทีมพัฒนาเล็ก ไม่มีเวลาดูแลความซับซ้อนเพิ่ม** — ต้องเรียนรู้ schema language, resolver pattern, N+1 query problem, dataloader, error handling ที่ต่างจาก REST — เป็นต้นทุนที่แท้จริง
- **ต้องการ HTTP caching แบบมาตรฐาน** — GraphQL ทำผ่าน POST ไป endpoint เดียวกันเสมอ ทำให้ cache ด้วย CDN/reverse proxy แบบที่ REST ทำได้ง่ายๆ (cache ตาม URL) นั้นยากกว่ามาก

พูดให้ตรงและซื่อสัตย์ที่สุด: **REST API ที่เราสร้างกันมาตลอด Part 056-069 (รวมถึง static files, JWT auth, OAuth2, WebSocket) แทบทั้งหมดไม่จำเป็นต้องใช้ GraphQL เลย** เพราะเป็นระบบที่มี client ไม่กี่แบบและความสัมพันธ์ข้อมูลไม่ซับซ้อนมาก GraphQL เหมาะกับสถานการณ์เฉพาะที่ปัญหา over-fetching/under-fetching เป็นปัญหาจริงจังที่วัดผลได้ (เช่นวัด bandwidth ที่ mobile app ใช้จริง) ไม่ใช่เทคโนโลยีที่ควรเลือกเพราะ "ดูทันสมัยกว่า"

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- GraphQL แก้ปัญหา **over-fetching** (ได้ข้อมูลเกิน) และ **under-fetching** (ได้ข้อมูลไม่พอ ต้องยิงหลาย request) ที่ REST มักเจอ ด้วยการให้ client เลือก field เองผ่าน endpoint เดียว
- GraphQL มี type system และ schema ที่บังคับชัดเจน เป็นสัญญาระหว่าง frontend กับ backend
- `99designs/gqlgen` ใช้แนวทาง **schema-first**: เขียน `.graphqls` ก่อน รัน `gqlgen generate` ให้สร้างโค้ด type-safe อัตโนมัติ แล้วเติมแค่ logic ใน resolver
- `type Query`/`type Mutation` ใน schema เทียบเท่า `GET`/`POST` ของ REST ตามลำดับ ส่วน `input` type ใช้เป็น parameter
- ไฟล์ `generated.go` และ `models_gen.go` **ห้ามแก้มือ** (ถูกเขียนทับทุกครั้งที่ generate) ส่วน `schema.resolvers.go` ปลอดภัยที่จะแก้ไข logic เอง เพราะ gqlgen จะ merge ของใหม่เข้ากับของเดิมโดยไม่ลบทิ้ง
- Field ที่ตรงกับชื่อ struct field พอดี (`Todo.User`) ไม่ต้องเขียน resolver แยก — แต่ในระบบจริงที่ query จากฐานข้อมูล อาจเจอ **N+1 query problem** ต้องแก้ด้วยเทคนิค dataloader
- Error ของ GraphQL ตอบผ่าน field `errors` ใน JSON response ไม่ใช่ HTTP status code แบบ REST
- **Subscription** คือ operation ประเภทที่สามของ GraphQL สำหรับข้อมูล real-time ผ่าน WebSocket (Part 069) resolver คืนค่าเป็น channel แทนค่าเดียว และต้องจัดการ lifecycle ของ subscriber เองเหมือน WebSocket hub ทุกประการ
- GraphQL คุ้มค่าเมื่อมีหลาย client ที่ต้องการข้อมูลต่างกันมากและโครงสร้างข้อมูลซับซ้อน แต่เกินความจำเป็นสำหรับ CRUD ธรรมดา — REST ที่เรียนมาตลอดภาคนี้ส่วนใหญ่ไม่จำเป็นต้องใช้ GraphQL เลย

## แบบฝึกหัดท้ายบท

1. ทำตามหัวข้อที่ 4-9 ให้ครบด้วยตัวเอง ตั้งแต่ `gqlgen init` จนถึงทดสอบ query/mutation ผ่าน curl หรือ Playground จริง
2. เพิ่ม field `createdAt: String!` ให้ type `Todo` ใน schema แล้วรัน `gqlgen generate` ใหม่ สังเกตว่า `models_gen.go` เปลี่ยนไปอย่างไร และแก้ resolver ให้ใส่ค่าเวลาปัจจุบันตอนสร้าง todo
3. เพิ่ม query ใหม่ `todo(id: ID!): Todo` สำหรับดึง todo รายการเดียวตาม ID (ถ้าไม่พบให้คืน error ที่มีข้อความชัดเจน)
4. เพิ่ม mutation `updateTodoDone(id: ID!, done: Boolean!): Todo!` สำหรับติ๊กว่า todo ทำเสร็จแล้วหรือยัง
5. ทดลองใช้ Playground ที่ `http://localhost:8080/` เขียน query ที่ขอ field ต่างกัน 3 แบบกับ mutation เดียวกัน แล้วสังเกตว่า response กลับมาแตกต่างกันตาม field ที่เลือกจริงหรือไม่
6. ทำตามหัวข้อที่ 10 เพิ่ม subscription `todoAdded` ให้ครบ แล้วทดลองเปิด Playground สองแท็บ แท็บหนึ่ง subscribe ไว้ อีกแท็บหนึ่งยิง mutation สร้าง todo ใหม่ สังเกตว่าแท็บแรกได้รับข้อมูลใหม่ทันทีโดยไม่ต้อง refresh
7. เขียนย่อหน้าอธิบาย (ไม่ต้องเขียนโค้ด) เปรียบเทียบว่าถ้าจะสร้างระบบ e-commerce ที่มีทั้งเว็บแอปและมือถือ (ซึ่งเราจะสร้างจริงใน **Part 103: E-Commerce REST API**) ท่านคิดว่าควรใช้ REST หรือ GraphQL เพราะเหตุใด โดยอ้างอิงเกณฑ์จากหัวข้อที่ 11

---

**ต่อไป**: [Part 071 — `database/sql` พื้นฐาน](./071-database-sql-basics.md)
