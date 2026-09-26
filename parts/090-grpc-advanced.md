# Part 090: gRPC ขั้นสูง: Streaming, Interceptors

> ภาคที่ 8: Microservices, gRPC, Message Queue — ตอนที่ 3 จาก 7 (Part 88–94)

## สารบัญของบทนี้

1. ทบทวน: unary RPC ที่เรียนไปแล้วใน Part 089
2. สี่รูปแบบของ RPC ใน gRPC
3. ออกแบบ `.proto` ให้รองรับทั้ง 4 รูปแบบ
4. Server Streaming: ทยอยส่งผลลัพธ์กลับทีละชิ้น
5. Client Streaming: รับข้อมูลหลายชิ้นแล้วสรุปผลครั้งเดียว
6. Bidirectional Streaming: สื่อสารสองทางพร้อมกัน
7. Interceptor: Middleware เวอร์ชัน gRPC
8. Error Handling: `status.Error` และ gRPC Status Codes
9. Deadline และ Cancellation ผ่าน `context` — ข้อได้เปรียบที่ REST ทำได้ยากกว่า
10. รันจริงทั้งระบบ: พิสูจน์ทั้ง 4 รูปแบบ RPC พร้อม Interceptor และ Deadline
11. gRPC-Gateway: เปิด REST/JSON ให้ client ภายนอกที่ไม่รู้จัก gRPC
12. สรุปสิ่งที่ได้เรียนในบทนี้
13. แบบฝึกหัดท้ายบท

---

## 1. ทบทวน: unary RPC ที่เรียนไปแล้วใน Part 089

ใน Part 089 เราสร้าง `GreeterService.SayHello` ซึ่งเป็น **unary RPC**: client ส่ง request หนึ่งชิ้น รอ server ตอบกลับหนึ่งชิ้น แล้วจบ — รูปแบบนี้ครอบคลุมการใช้งานส่วนใหญ่ได้ แต่มีหลายสถานการณ์ที่ unary RPC ไม่เหมาะ เช่น:

- **ผลลัพธ์มีจำนวนมาก** และอยากให้ client เริ่มประมวลผลได้ทันทีที่ผลลัพธ์แรกมาถึง แทนที่จะรอผลลัพธ์ทั้งหมดเสร็จก่อน (เช่น ค้นหาสินค้าแล้วทยอยแสดงผล)
- **Client มีข้อมูลจำนวนมากที่จะส่ง** และอยากส่งทีละก้อนแทนที่จะรวมเป็นก้อนเดียวก้อนใหญ่ (เช่น อัปโหลด log จำนวนมาก)
- **ทั้งสองฝั่งต้องคุยกันต่อเนื่องแบบ real-time** (เช่น chat, การอัปเดตสถานะแบบสด)

นี่คือเหตุผลที่ gRPC ออกแบบให้รองรับ **streaming** เป็นความสามารถหลักของ protocol ตั้งแต่ต้น ไม่ใช่ส่วนเสริมที่ต้องพึ่งเทคโนโลยีอื่น (ต่างจาก REST ที่ต้องพึ่ง Server-Sent Events หรือ WebSocket แยกต่างหากซึ่งเราเรียนไปแล้วใน Part 069)

---

## 2. สี่รูปแบบของ RPC ใน gRPC

gRPC รองรับ RPC 4 รูปแบบ แบ่งตามทิศทางของ stream:

| รูปแบบ | Client ส่ง | Server ตอบ | ตัวอย่างการใช้งาน |
|---|---|---|---|
| **Unary** | 1 request | 1 response | เรียก API ทั่วไป เช่น `GetUser(id)` |
| **Server streaming** | 1 request | stream ของ response | ค้นหาข้อมูลแล้วทยอยส่งผล, subscribe การอัปเดตราคาหุ้น |
| **Client streaming** | stream ของ request | 1 response | อัปโหลดไฟล์เป็นก้อนๆ, ส่ง log จำนวนมากแล้วรอสรุปผล |
| **Bidirectional streaming** | stream ของ request | stream ของ response (อิสระจากกัน) | Chat แบบ real-time, การแปลภาษาสด |

ทั้ง 4 รูปแบบเกิดจากการผสมกันของ **client-side streaming** และ **server-side streaming** โดย unary คือกรณีที่ไม่มี streaming ทั้งสองฝั่งเลย (ทั้งสองฝั่งส่งแค่ 1 ครั้ง) ส่วน bidirectional คือกรณีที่ streaming ทั้งสองฝั่งพร้อมกัน

---

## 3. ออกแบบ `.proto` ให้รองรับทั้ง 4 รูปแบบ

ต่อยอดจากโปรเจกต์ `grpc-basics` ในบทที่แล้ว สร้างโปรเจกต์ใหม่ `grpc-advanced` และไฟล์ `search.proto` ที่ประกอบด้วย RPC ทั้ง 4 รูปแบบในไฟล์เดียว:

```protobuf
syntax = "proto3";

package search;

option go_package = "grpc-advanced/searchpb";

// SearchService สาธิตทั้ง 4 รูปแบบของ gRPC RPC
service SearchService {
  // 1. Unary: request เดียว, response เดียว (เหมือน REST ปกติ)
  rpc Ping (PingRequest) returns (PingResponse);

  // 2. Server streaming: request เดียว, server ทยอยส่งผลลัพธ์กลับมาเป็น stream
  rpc Search (SearchRequest) returns (stream SearchResult);

  // 3. Client streaming: client ทยอยส่งข้อมูลหลายชิ้น, server ตอบกลับครั้งเดียวตอนจบ
  rpc UploadEvents (stream EventItem) returns (UploadSummary);

  // 4. Bidirectional streaming: ทั้งสองฝั่งส่งข้อมูลเป็น stream พร้อมกันได้อิสระ
  rpc Chat (stream ChatMessage) returns (stream ChatMessage);
}

message PingRequest {
  string sender = 1;
}

message PingResponse {
  string message = 1;
}

message SearchRequest {
  string query = 1;
  int32 limit = 2;
}

message SearchResult {
  string item = 1;
  int32 rank = 2;
}

message EventItem {
  string name = 1;
  int32 value = 2;
}

message UploadSummary {
  int32 total_events = 1;
  int32 sum_value = 2;
}

message ChatMessage {
  string from = 1;
  string text = 2;
}
```

สังเกตวิธีเขียน `.proto`: การเติมคำว่า **`stream`** ไว้หน้า type ของ request และ/หรือ response คือสิ่งเดียวที่บอก `protoc` ว่า RPC ตัวนี้เป็นรูปแบบไหน — ไม่มี syntax พิเศษอื่นเลย generate โค้ดด้วยคำสั่งเดียวกับที่เรียนไปแล้วใน Part 089:

```bash
protoc --go_out=. --go_opt=paths=source_relative \
  --go-grpc_out=. --go-grpc_opt=paths=source_relative \
  search.proto
```

รันจริงในเครื่องที่เขียนบทเรียนนี้ คำสั่งจบสำเร็จ (`exit code 0`) ได้โค้ดที่ generate มาให้ signature ที่ต่างกันตามรูปแบบ RPC — ตัวอย่างจาก `search_grpc.pb.go` ที่ generate จริง:

```go
type SearchServiceClient interface {
	Ping(ctx context.Context, in *PingRequest, opts ...grpc.CallOption) (*PingResponse, error)
	Search(ctx context.Context, in *SearchRequest, opts ...grpc.CallOption) (grpc.ServerStreamingClient[SearchResult], error)
	UploadEvents(ctx context.Context, opts ...grpc.CallOption) (grpc.ClientStreamingClient[EventItem, UploadSummary], error)
	Chat(ctx context.Context, opts ...grpc.CallOption) (grpc.BidiStreamingClient[ChatMessage, ChatMessage], error)
}
```

สังเกตว่า `Search` คืนค่าเป็น `ServerStreamingClient[SearchResult]` (มีแต่ `Recv()`), `UploadEvents` ไม่มี `in` parameter แต่คืน `ClientStreamingClient[EventItem, UploadSummary]` (มีทั้ง `Send()` และ `CloseAndRecv()`), ส่วน `Chat` คืน `BidiStreamingClient[ChatMessage, ChatMessage]` (มีทั้ง `Send()` และ `Recv()` อิสระจากกัน) — โค้ดที่ generate มาบังคับให้ใช้ API ที่ถูกต้องตาม RPC แต่ละแบบโดยอัตโนมัติผ่านระบบ type ของ Go

---

## 4. Server Streaming: ทยอยส่งผลลัพธ์กลับทีละชิ้น

Implement ฝั่ง server สำหรับ `Search`:

```go
func (s *server) Search(req *pb.SearchRequest, stream pb.SearchService_SearchServer) error {
	if req.GetQuery() == "" {
		return status.Error(codes.InvalidArgument, "query must not be empty")
	}

	limit := req.GetLimit()
	if limit <= 0 {
		limit = 5
	}

	for i := int32(1); i <= limit; i++ {
		select {
		case <-stream.Context().Done():
			log.Printf("client cancelled or deadline exceeded: %v", stream.Context().Err())
			return status.FromContextError(stream.Context().Err()).Err()
		default:
		}

		result := &pb.SearchResult{
			Item: fmt.Sprintf("%s-result-%d", req.GetQuery(), i),
			Rank: i,
		}
		if err := stream.Send(result); err != nil {
			return err
		}
		time.Sleep(300 * time.Millisecond)
	}
	return nil
}
```

จุดสำคัญ:

- ลายเซ็นของฟังก์ชันไม่คืนค่า `(*Response, error)` แบบ unary แต่คืน **แค่ `error`** เท่านั้น — ผลลัพธ์แต่ละชิ้นถูกส่งออกไปทันทีผ่าน `stream.Send(result)` แทน
- `stream.Context()` คือ context ของการเชื่อมต่อฝั่ง server สำหรับ streaming RPC — ควรเช็ค `stream.Context().Done()` เป็นระยะระหว่าง loop ยาวๆ เพื่อหยุดทำงานทันทีถ้า client ยกเลิกหรือ deadline หมดอายุแล้ว (จะอธิบายเจาะลึกในหัวข้อ 9)
- `time.Sleep(300 * time.Millisecond)` จำลอง latency ของแต่ละผลลัพธ์ (เช่น query ฐานข้อมูลทีละหน้า) เพื่อให้เห็นชัดว่า client ได้รับผลลัพธ์เป็นชุดๆ ไม่ใช่รอทั้งหมดพร้อมกัน

ฝั่ง client อ่าน stream ด้วย loop `Recv()` จนกว่าจะเจอ `io.EOF`:

```go
func streamAll(client pb.SearchServiceClient) {
	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()

	stream, err := client.Search(ctx, &pb.SearchRequest{Query: "golang", Limit: 3})
	if err != nil {
		log.Fatalf("search failed: %v", err)
	}

	for {
		result, err := stream.Recv()
		if err == io.EOF {
			break
		}
		if err != nil {
			log.Fatalf("stream error: %v", err)
		}
		log.Printf("received: %s (rank=%d)", result.GetItem(), result.GetRank())
	}
}
```

รูปแบบนี้เหมือนกับการอ่านจาก channel ที่เรียนในภาคที่ 3 (Concurrency): loop จนกว่า stream จะปิด (`io.EOF` เทียบเท่ากับ channel ที่ถูกปิดและระบายค่าจนหมด)

---

## 5. Client Streaming: รับข้อมูลหลายชิ้นแล้วสรุปผลครั้งเดียว

Implement ฝั่ง server สำหรับ `UploadEvents`:

```go
func (s *server) UploadEvents(stream pb.SearchService_UploadEventsServer) error {
	var total, sum int32
	for {
		event, err := stream.Recv()
		if err == io.EOF {
			// client ส่งครบแล้ว ปิดฝั่งส่ง -> เราตอบกลับสรุปผลครั้งเดียว
			return stream.SendAndClose(&pb.UploadSummary{
				TotalEvents: total,
				SumValue:    sum,
			})
		}
		if err != nil {
			return err
		}
		total++
		sum += event.GetValue()
		log.Printf("upload event received: name=%s value=%d", event.GetName(), event.GetValue())
	}
}
```

สังเกตว่ารูปแบบนี้ **กลับด้านกับ server streaming**: server เป็นฝ่าย `Recv()` วนไปเรื่อยๆ จนกว่า client จะปิด stream (ได้ `io.EOF`) แล้วค่อยตอบกลับครั้งเดียวด้วย `stream.SendAndClose(...)`

ฝั่ง client ส่งข้อมูลหลายชิ้นแล้วปิด stream ด้วย `CloseAndRecv()`:

```go
func clientStreamingUpload(client pb.SearchServiceClient) {
	ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
	defer cancel()

	stream, err := client.UploadEvents(ctx)
	if err != nil {
		log.Fatalf("upload events failed: %v", err)
	}

	events := []*pb.EventItem{
		{Name: "click", Value: 1},
		{Name: "click", Value: 1},
		{Name: "purchase", Value: 50},
	}
	for _, e := range events {
		if err := stream.Send(e); err != nil {
			log.Fatalf("send event failed: %v", err)
		}
	}

	summary, err := stream.CloseAndRecv()
	if err != nil {
		log.Fatalf("close and recv failed: %v", err)
	}
	log.Printf("upload summary: total=%d sum=%d", summary.GetTotalEvents(), summary.GetSumValue())
}
```

`CloseAndRecv()` ทำสองอย่างในคำสั่งเดียว: บอก server ว่า "ฝั่ง client ส่งครบแล้ว" (เทียบเท่าปิดฝั่งเขียนของ stream) แล้วรอรับ response สุดท้ายกลับมา

---

## 6. Bidirectional Streaming: สื่อสารสองทางพร้อมกัน

Implement ฝั่ง server สำหรับ `Chat` — echo ข้อความกลับไปทันทีที่ได้รับแต่ละข้อความ:

```go
func (s *server) Chat(stream pb.SearchService_ChatServer) error {
	for {
		msg, err := stream.Recv()
		if err == io.EOF {
			return nil
		}
		if err != nil {
			return err
		}
		log.Printf("chat message from %s: %s", msg.GetFrom(), msg.GetText())
		reply := &pb.ChatMessage{From: "server", Text: "echo: " + msg.GetText()}
		if err := stream.Send(reply); err != nil {
			return err
		}
	}
}
```

ฝั่ง client ต้อง **ส่งและรับพร้อมกัน** ซึ่งจำเป็นต้องแยก goroutine สำหรับอ่าน response ออกจาก goroutine หลักที่ส่ง request (ตามหลัก concurrency ที่เรียนในภาคที่ 3):

```go
func bidiChat(client pb.SearchServiceClient) {
	ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
	defer cancel()

	stream, err := client.Chat(ctx)
	if err != nil {
		log.Fatalf("chat failed: %v", err)
	}

	messages := []string{"hello", "how are you?", "bye"}

	done := make(chan struct{})
	go func() {
		defer close(done)
		for {
			reply, err := stream.Recv()
			if err == io.EOF {
				return
			}
			if err != nil {
				log.Printf("chat recv error: %v", err)
				return
			}
			log.Printf("chat received: %s", reply.GetText())
		}
	}()

	for _, m := range messages {
		if err := stream.Send(&pb.ChatMessage{From: "client", Text: m}); err != nil {
			log.Fatalf("chat send failed: %v", err)
		}
	}
	stream.CloseSend()
	<-done
}
```

จุดสำคัญ: ต้องเปิด **goroutine แยก** สำหรับอ่าน (`stream.Recv()`) เพราะการอ่านและเขียนบน bidirectional stream เป็นอิสระจากกันโดยสมบูรณ์ — ถ้าอ่านและเขียนใน goroutine เดียวกันแบบลำดับ (ส่ง 1 ครั้ง รอรับ 1 ครั้ง สลับกัน) จะไม่ใช่ bidirectional streaming ที่แท้จริงอีกต่อไป (กลายเป็นเหมือน unary ที่เรียกซ้ำหลายครั้งในทางปฏิบัติ) `stream.CloseSend()` บอกฝั่ง server ว่าจะไม่ส่งอะไรเพิ่มแล้ว (ปิดฝั่งเขียน) แต่ยังรอรับ response ที่เหลือได้ตามปกติ ก่อนจะ `<-done` รอให้ goroutine อ่านจบสนิท

---

## 7. Interceptor: Middleware เวอร์ชัน gRPC

ใน Part 057 เราเรียน **Middleware Pattern** สำหรับ HTTP server ในรูปแบบ `func(http.Handler) http.Handler` — gRPC มีแนวคิดเดียวกันเป๊ะ เรียกว่า **Interceptor** ทำหน้าที่ครอบ (wrap) การเรียก RPC ทุกตัวเพื่อทำงานร่วม เช่น logging, authentication, metrics โดยไม่ต้องเขียนซ้ำในทุกเมธอด

gRPC แยก interceptor เป็น 2 ชนิดตามประเภทของ RPC: **Unary interceptor** (สำหรับ unary RPC) และ **Stream interceptor** (สำหรับ 3 รูปแบบที่เหลือทั้งหมด)

### Unary Interceptor

```go
func loggingUnaryInterceptor(ctx context.Context, req interface{}, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (interface{}, error) {
	start := time.Now()
	resp, err := handler(ctx, req)
	log.Printf("[unary] method=%s duration=%s err=%v", info.FullMethod, time.Since(start), err)
	return resp, err
}
```

โครงสร้างนี้เหมือนกับ HTTP middleware มาก: รับ `handler` (คือ RPC ตัวจริงที่จะถูกเรียก) มาเป็น parameter, ทำงานอะไรก่อนเรียก (`start := time.Now()`), เรียก `handler(ctx, req)`, แล้วทำงานอะไรหลังเรียกเสร็จ (log ผลลัพธ์) — `info.FullMethod` บอกชื่อเต็มของ RPC ที่กำลังถูกเรียก เช่น `/search.SearchService/Ping`

### Stream Interceptor

```go
func loggingStreamInterceptor(srv interface{}, ss grpc.ServerStream, info *grpc.StreamServerInfo, handler grpc.StreamHandler) error {
	start := time.Now()
	err := handler(srv, ss)
	log.Printf("[stream] method=%s duration=%s err=%v", info.FullMethod, time.Since(start), err)
	return err
}
```

หลักการเดียวกัน แต่ครอบทั้ง stream ทั้งก้อน (`duration` ที่วัดได้คือเวลารวมของทั้ง stream ตั้งแต่เปิดจนปิด ไม่ใช่แค่ request เดียว)

### ตัวอย่าง Auth Interceptor (โครงร่าง)

```go
func authUnaryInterceptor(ctx context.Context, req interface{}, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (interface{}, error) {
	// ในระบบจริง: อ่าน token จาก metadata.FromIncomingContext(ctx)
	// แล้วตรวจสอบสิทธิ์ก่อนอนุญาตให้ handler ทำงานต่อ เช่นเดียวกับ JWT middleware ใน Part 067
	return handler(ctx, req)
}
```

gRPC ส่ง metadata (เทียบเท่า HTTP header) ผ่าน `context` โดยใช้ package `google.golang.org/grpc/metadata` — วิธีอ่าน token จาก metadata มีรูปแบบเดียวกับการอ่าน `Authorization` header ที่เรียนใน Part 067 (JWT Authentication) เพียงแค่เปลี่ยนจาก `r.Header.Get(...)` เป็น `metadata.FromIncomingContext(ctx)`

### รวม Interceptor หลายตัวด้วย Chain

```go
grpcServer := grpc.NewServer(
	grpc.ChainUnaryInterceptor(loggingUnaryInterceptor, authUnaryInterceptor),
	grpc.ChainStreamInterceptor(loggingStreamInterceptor),
)
```

`grpc.ChainUnaryInterceptor` ทำหน้าที่เดียวกับฟังก์ชัน `Chain` ที่เราเขียนเองใน Part 057 สำหรับ HTTP middleware — เรียง interceptor ต่อกันเป็นชั้นๆ ตามลำดับที่ระบุ (`loggingUnaryInterceptor` ทำงานก่อน แล้วค่อยเรียก `authUnaryInterceptor` แล้วค่อยถึง handler จริง) ข้อแตกต่างสำคัญคือ **ต้องแยกลงทะเบียน unary กับ stream interceptor คนละ chain** เพราะ signature ของทั้งสองไม่เหมือนกัน

---

## 8. Error Handling: `status.Error` และ gRPC Status Codes

ใน Part 015-016 เราเรียนเรื่อง error handling ของ Go ด้วย `error` interface ธรรมดา — สำหรับ gRPC การคืน error แบบ Go เปล่าๆ (`errors.New("something wrong")`) จะถูกส่งกลับไปหา client เป็น status code `codes.Unknown` เสมอ ซึ่งไม่มีประโยชน์กับ client ในการตัดสินใจว่าควร retry หรือไม่ ควรแสดง error อะไรให้ผู้ใช้เห็น

gRPC มีชุด **status codes มาตรฐาน** (คล้าย HTTP status code ที่เรียนใน Part 059) ให้ใช้แทน:

| Code | ความหมาย | เทียบเท่า HTTP |
|---|---|---|
| `OK` | สำเร็จ | 200 |
| `InvalidArgument` | ข้อมูลที่ client ส่งมาไม่ถูกต้อง | 400 |
| `NotFound` | ไม่พบข้อมูลที่ร้องขอ | 404 |
| `AlreadyExists` | ข้อมูลมีอยู่แล้ว (เช่น สร้างซ้ำ) | 409 |
| `PermissionDenied` | ไม่มีสิทธิ์เข้าถึง | 403 |
| `Unauthenticated` | ยังไม่ได้ยืนยันตัวตน | 401 |
| `DeadlineExceeded` | หมดเวลาที่กำหนด | 504 |
| `Canceled` | ถูกยกเลิกโดย client | 499 |
| `ResourceExhausted` | เกิน rate limit / quota | 429 |
| `Internal` | ข้อผิดพลาดภายใน server | 500 |
| `Unavailable` | service ไม่พร้อมให้บริการชั่วคราว | 503 |

การคืน error ที่มี status code ที่ถูกต้องทำผ่าน package `google.golang.org/grpc/status`:

```go
import (
	"google.golang.org/grpc/codes"
	"google.golang.org/grpc/status"
)

func (s *server) Search(req *pb.SearchRequest, stream pb.SearchService_SearchServer) error {
	if req.GetQuery() == "" {
		return status.Error(codes.InvalidArgument, "query must not be empty")
	}
	// ...
}
```

ฝั่ง client ตรวจสอบ error ที่ได้รับด้วย `status.FromError`:

```go
if s, ok := status.FromError(err); ok {
	log.Printf("got expected error: code=%s message=%s", s.Code(), s.Message())
} else {
	log.Printf("unexpected error type: %v", err)
}
```

รันจริงในเครื่องที่เขียนบทเรียนนี้ ได้ผลลัพธ์:

```
=== case 3: invalid argument ===
got expected error: code=InvalidArgument message=query must not be empty
```

`status.FromError` คืนค่า `ok = false` ถ้า error ที่ได้รับไม่ใช่ gRPC status error (เช่น เป็น network error ทั่วไป) — การเช็คแบบนี้เทียบเท่ากับการเช็ค `errors.As` กับ custom error type ที่เรียนใน Part 016 เพียงแต่เป็นกลไกเฉพาะของ gRPC

---

## 9. Deadline และ Cancellation ผ่าน `context` — ข้อได้เปรียบที่ REST ทำได้ยากกว่า

ใน Part 032 เราเรียนว่า `context.WithTimeout`/`WithDeadline` ใช้ยกเลิกงานที่ใช้เวลานานเกินไปได้ — ข้อได้เปรียบสำคัญของ gRPC เหนือ REST ทั่วไปคือ **deadline ที่ตั้งฝั่ง client จะถูกส่งไปกับ request จริงๆ ผ่าน network (เป็นส่วนหนึ่งของ gRPC metadata)** ทำให้ฝั่ง server รู้ทันทีว่าเหลือเวลาอีกเท่าไหร่ก่อนที่ client จะเลิกรอ และสามารถหยุดทำงานได้ทันทีแทนที่จะทำงานต่อไปโดยไม่มีประโยชน์ (เพราะ client ไม่รอผลลัพธ์แล้ว)

REST แบบทั่วไปทำแบบนี้ไม่ได้โดยอัตโนมัติ — HTTP header ไม่มีแนวคิดเรื่อง "deadline ที่เหลือ" ในตัว โดย default ถ้า client ตั้ง timeout ฝั่งตัวเอง แล้ว timeout หมดอายุ, client แค่หยุดรอ response แต่ server (ถ้าไม่ได้เขียนโค้ดพิเศษเช็ค connection ที่ปิดไปแล้วเอง) จะยังคงทำงานต่อไปจนเสร็จโดยไม่รู้ตัวเลยว่าไม่มีใครรอผลอยู่แล้ว เปลืองทรัพยากรฟรีๆ

มาดูการทดลองจริงที่พิสูจน์เรื่องนี้: ฝั่ง client ตั้ง deadline สั้นกว่าที่ server ต้องใช้:

```go
func streamWithShortDeadline(client pb.SearchServiceClient) {
	// server ใช้เวลา ~300ms ต่อผลลัพธ์ และขอ limit=5 (รวม ~1.5s) แต่เราตั้ง deadline แค่ 500ms
	ctx, cancel := context.WithTimeout(context.Background(), 500*time.Millisecond)
	defer cancel()

	stream, err := client.Search(ctx, &pb.SearchRequest{Query: "deadline-test", Limit: 5})
	// ...
	for {
		result, err := stream.Recv()
		if err != nil {
			st, ok := status.FromError(err)
			if ok && st.Code() == codes.DeadlineExceeded {
				log.Printf("expected: deadline exceeded — %v", st.Message())
			}
			break
		}
		log.Printf("received before deadline: %s", result.GetItem())
	}
}
```

ฝั่ง server เช็ค `stream.Context().Done()` ในทุกรอบ loop (โค้ดในหัวข้อ 4):

```go
select {
case <-stream.Context().Done():
	log.Printf("client cancelled or deadline exceeded: %v", stream.Context().Err())
	return status.FromContextError(stream.Context().Err()).Err()
default:
}
```

รันจริงได้ผลลัพธ์ที่ยืนยันว่า deadline ถูกส่งผ่าน wire จริงและ server รับรู้ได้:

**ฝั่ง client:**

```
=== case 2: short deadline ===
received before deadline: deadline-test-result-1
received before deadline: deadline-test-result-2
expected: deadline exceeded — context deadline exceeded
```

**ฝั่ง server (log พร้อมกัน):**

```
client cancelled or deadline exceeded: context canceled
[stream] method=/search.SearchService/Search duration=602.451442ms err=rpc error: code = Canceled desc = context canceled
```

Client ได้รับผลลัพธ์แค่ 2 ชิ้นจาก 5 ชิ้นที่ขอ (เพราะแต่ละชิ้นใช้เวลา ~300ms และ deadline คือ 500ms) แล้ว `Recv()` คืน error `DeadlineExceeded` ทันที **และ** server หยุดทำงาน loop ต่อ (สังเกตว่า duration ของ stream บน server คือ ~602ms ไม่ใช่ ~1.5s เต็มที่ควรใช้ถ้าทำครบ 5 ชิ้น) — นี่คือหลักฐานว่า deadline ถูกส่งผ่าน network จริง ไม่ใช่แค่ timeout ที่ทำงานฝั่ง client เท่านั้น server รับรู้และหยุดทำงานให้เองโดยอัตโนมัติผ่านกลไกของ `context` ที่ผูกกับ gRPC connection โดยตรง — สิ่งนี้ประหยัดทรัพยากรของระบบได้มากในสถานการณ์ที่ client จำนวนมากยกเลิก request พร้อมกัน (เช่น ผู้ใช้ปิดแอปกลางทาง)

---

## 10. รันจริงทั้งระบบ: พิสูจน์ทั้ง 4 รูปแบบ RPC พร้อม Interceptor และ Deadline

รวมทุกอย่างเข้าด้วยกันในไฟล์ `server/main.go`:

```go
func main() {
	lis, err := net.Listen("tcp", ":50052")
	if err != nil {
		log.Fatalf("failed to listen: %v", err)
	}

	grpcServer := grpc.NewServer(
		grpc.ChainUnaryInterceptor(loggingUnaryInterceptor, authUnaryInterceptor),
		grpc.ChainStreamInterceptor(loggingStreamInterceptor),
	)
	pb.RegisterSearchServiceServer(grpcServer, &server{})

	log.Println("gRPC server (streaming + interceptor demo) listening on :50052")
	if err := grpcServer.Serve(lis); err != nil {
		log.Fatalf("failed to serve: %v", err)
	}
}
```

และไฟล์ `client/main.go` เรียกทดสอบทั้ง 6 เคส (unary, server streaming ปกติ, server streaming ที่โดน deadline, error handling, client streaming, bidirectional streaming) เรียงกัน

Build และรันจริงในเครื่องที่เขียนบทเรียนนี้:

```bash
go build ./...
go run ./server &
go run ./client
```

ผลลัพธ์เต็มที่ได้จากฝั่ง client:

```
=== case 0: unary RPC ===
ping response: pong to client
=== case 1: full stream ===
received: golang-result-1 (rank=1)
received: golang-result-2 (rank=2)
received: golang-result-3 (rank=3)
=== case 2: short deadline ===
received before deadline: deadline-test-result-1
received before deadline: deadline-test-result-2
expected: deadline exceeded — context deadline exceeded
=== case 3: invalid argument ===
got expected error: code=InvalidArgument message=query must not be empty
=== case 4: client streaming ===
upload summary: total=3 sum=52
=== case 5: bidirectional streaming ===
chat received: echo: hello
chat received: echo: how are you?
chat received: echo: bye
```

และฝั่ง server แสดง log จาก interceptor ที่ครอบทุก RPC ไว้ ยืนยันว่า logging interceptor ทำงานกับทุกรูปแบบ RPC จริง:

```
gRPC server (streaming + interceptor demo) listening on :50052
[unary] method=/search.SearchService/Ping duration=2.022µs err=<nil>
[stream] method=/search.SearchService/Search duration=902.409091ms err=<nil>
[stream] method=/search.SearchService/Search duration=43.655µs err=rpc error: code = InvalidArgument desc = query must not be empty
upload event received: name=click value=1
upload event received: name=click value=1
upload event received: name=purchase value=50
[stream] method=/search.SearchService/UploadEvents duration=167.217µs err=<nil>
chat message from client: hello
chat message from client: how are you?
chat message from client: bye
[stream] method=/search.SearchService/Chat duration=391.323µs err=<nil>
client cancelled or deadline exceeded: context canceled
[stream] method=/search.SearchService/Search duration=601.473812ms err=rpc error: code = Canceled desc = context canceled
```

การรันจริงนี้ยืนยันครบทุกประเด็นสำคัญของบทนี้ในคราวเดียว: RPC ทั้ง 4 รูปแบบทำงานถูกต้อง, interceptor ครอบทุก method (ทั้ง unary และ stream) และ log duration/error ได้แม่นยำ, error handling คืน status code ที่ถูกต้อง, และ deadline ถูกส่งผ่าน wire จริงจน server หยุดทำงานได้เองโดยไม่ต้องรอ timeout ฝั่ง client เท่านั้น

---

## 11. gRPC-Gateway: เปิด REST/JSON ให้ client ภายนอกที่ไม่รู้จัก gRPC

ย้อนกลับไปที่ Part 088 หัวข้อ 8 (API Gateway) — ปัญหาหนึ่งของการเลือกใช้ gRPC ภายในระบบคือ **เบราว์เซอร์และ client ภายนอกจำนวนมากเรียก gRPC ตรงๆ ไม่ได้** (เพราะ gRPC ต้องการ HTTP/2 และ binary framing ที่ JavaScript `fetch` มาตรฐานไม่รองรับโดยตรง)

**gRPC-Gateway** (โปรเจกต์โอเพนซอร์สแยกต่างหาก `grpc-ecosystem/grpc-gateway`) แก้ปัญหานี้โดยการ generate **reverse-proxy server** จากไฟล์ `.proto` เดิม (เพิ่ม annotation พิเศษที่ map แต่ละ RPC เข้ากับ REST endpoint) reverse-proxy ตัวนี้รับ HTTP/JSON request แบบ REST ปกติจากภายนอก แล้วแปลงเป็น gRPC call ไปยัง service จริงภายใน แล้วแปลง response กลับเป็น JSON ส่งคืน

```
Browser/Mobile (REST+JSON) → [gRPC-Gateway] → gRPC (binary) → [Internal gRPC Service]
```

วิธีใช้งานคร่าวๆ คือเพิ่ม annotation ใน `.proto`:

```protobuf
import "google/api/annotations.proto";

service SearchService {
  rpc Search (SearchRequest) returns (stream SearchResult) {
    option (google.api.http) = {
      get: "/v1/search"
    };
  }
}
```

แล้วใช้ปลั๊กอิน `protoc-gen-grpc-gateway` generate reverse-proxy handler เพิ่มเติมจากปลั๊กอินเดิม (`protoc-gen-go`, `protoc-gen-go-grpc`) ที่เรียนไปแล้วใน Part 089

ผลลัพธ์คือ **ระบบเดียวรองรับทั้งสองแบบพร้อมกัน**: service ภายในคุยกันด้วย gRPC (เร็ว, type-safe) ในขณะที่ client ภายนอก (เว็บ, มือถือ, นักพัฒนาภายนอกที่คุ้นเคยกับ REST) เรียกผ่าน endpoint REST ปกติที่ gRPC-Gateway แปลงให้อัตโนมัติ โดยไม่ต้องเขียน handler REST แยกต่างหากซ้ำซ้อนกับ logic ที่มีอยู่แล้วใน gRPC service

> **หมายเหตุความโปร่งใส**: ส่วนนี้เป็นข้อมูลอ้างอิงจากเอกสารทางการของโปรเจกต์ `grpc-ecosystem/grpc-gateway` เพื่อให้เห็นภาพรวมว่ามันมีไว้แก้ปัญหาอะไรและ pattern การใช้งานหน้าตาเป็นอย่างไร — ไม่ได้ติดตั้งและรันจริงในบทนี้ เพราะเป็นเครื่องมือเสริมที่อยู่นอกขอบเขตของ core concept ที่บทนี้ต้องการสอน (streaming, interceptor, deadline) ผู้อ่านที่สนใจนำไปใช้จริงควรศึกษาต่อจากเอกสารทางการที่ [grpc-ecosystem/grpc-gateway](https://github.com/grpc-ecosystem/grpc-gateway)

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- gRPC รองรับ RPC **4 รูปแบบ**: Unary, Server streaming, Client streaming, Bidirectional streaming — กำหนดด้วยการเติม keyword `stream` หน้า type ใน `.proto` เท่านั้น
- **Server streaming**: server ใช้ `stream.Send()` วนส่งผลลัพธ์หลายชิ้น, client ใช้ `stream.Recv()` วนอ่านจนเจอ `io.EOF`
- **Client streaming**: client ใช้ `stream.Send()` วนส่งหลายชิ้นแล้วปิดด้วย `CloseAndRecv()`, server ใช้ `stream.Recv()` วนอ่านจนเจอ `io.EOF` แล้วตอบครั้งเดียวด้วย `SendAndClose()`
- **Bidirectional streaming**: ต้องแยก goroutine สำหรับอ่านและเขียน เพราะทั้งสองทิศทางเป็นอิสระจากกันอย่างแท้จริง
- **Interceptor** คือ middleware เวอร์ชัน gRPC แยกเป็น Unary interceptor และ Stream interceptor เพราะ signature ต่างกัน, รวมหลายตัวด้วย `grpc.ChainUnaryInterceptor`/`grpc.ChainStreamInterceptor`
- ใช้ `status.Error(codes.XXX, "message")` แทน `errors.New(...)` เพื่อส่ง gRPC status code ที่มีความหมายกลับไปหา client, ตรวจสอบด้วย `status.FromError(err)`
- **Deadline ที่ตั้งฝั่ง client (`context.WithTimeout`) ถูกส่งผ่าน wire จริงไปถึง server** — server เช็ค `stream.Context().Done()` เพื่อหยุดทำงานทันทีเมื่อ client ไม่รอผลแล้ว นี่คือข้อได้เปรียบสำคัญของ gRPC เหนือ REST ทั่วไป ซึ่งได้พิสูจน์ด้วยการรันจริงในบทนี้
- **gRPC-Gateway** เปิดทางให้ client ภายนอกที่ใช้ REST/JSON เรียก gRPC service ได้โดยไม่ต้องรู้จัก gRPC โดยตรง เหมาะกับสถานการณ์ที่ระบบภายในใช้ gRPC แต่ต้องเปิด public API ด้วย

## แบบฝึกหัดท้ายบท

1. ทำตามบทเรียนนี้ สร้าง `SearchService` ที่มีครบทั้ง 4 รูปแบบ RPC รันจริงแล้วยืนยันผลลัพธ์เหมือนในหัวข้อ 10
2. เพิ่ม stream interceptor ตัวใหม่ที่ทำหน้าที่นับจำนวนข้อความที่ส่งผ่าน stream ทั้งหมด (ทั้ง client streaming และ bidirectional) แล้ว log สรุปจำนวนออกมาหลัง stream ปิด
3. แก้ไข `Search` (server streaming) ให้คืน error `codes.ResourceExhausted` แทน `codes.InvalidArgument` เมื่อ `req.Limit` มากกว่า 100 แล้วทดสอบว่า client ได้รับ status code ที่ถูกต้อง
4. ทดลองปรับ deadline ในโค้ด client streaming (`clientStreamingUpload`) ให้สั้นกว่าที่ server ใช้ประมวลผลจริง (เพิ่ม `time.Sleep` ใน server) แล้วสังเกตพฤติกรรมของ `CloseAndRecv()` เมื่อ deadline หมดอายุ
5. เขียน bidirectional Chat ให้รองรับหลาย client พร้อมกัน (server เก็บ list ของ stream ที่ active แล้ว broadcast ข้อความไปหาทุกคน แทนที่จะ echo กลับหาแค่ผู้ส่งคนเดียว) — ระวังเรื่อง concurrent access ตามที่เรียนในภาคที่ 3 (ต้องใช้ mutex หรือ channel ป้องกัน race condition)
6. ค้นคว้าเพิ่มเติมเกี่ยวกับ gRPC-Gateway แล้วลองติดตั้งจริงกับ `SearchService` จากแบบฝึกหัดข้อ 1 (ถ้ามี `protoc` และเครื่องมือครบ) เปรียบเทียบกับสิ่งที่เอกสารในหัวข้อ 11 อธิบายไว้

---

**ต่อไป**: [Part 091 — Message Queue: RabbitMQ กับ Go](./091-rabbitmq.md)
