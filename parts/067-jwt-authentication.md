# Part 067: Authentication ด้วย JWT

> ภาคที่ 5: Web Development — ตอนที่ 12 จาก 15

## สารบัญของบทนี้

1. JWT คืออะไร และแก้ปัญหาอะไร
2. โครงสร้าง JWT: header.payload.signature
3. ความเข้าใจผิดที่พบบ่อยที่สุด: JWT ไม่ได้ถูกเข้ารหัส
4. ติดตั้ง `github.com/golang-jwt/jwt/v5`
5. สร้างและเซ็น token พร้อม claims
6. ตรวจสอบและ parse token, การหมดอายุ
7. Auth Middleware: อ่าน `Authorization: Bearer <token>`
8. Login Endpoint แบบสมบูรณ์: bcrypt + JWT
9. Refresh Token Pattern
10. ข้อควรระวังด้านความปลอดภัย
11. สรุปสิ่งที่ได้เรียนในบทนี้
12. แบบฝึกหัดท้ายบท

---

## 1. JWT คืออะไร และแก้ปัญหาอะไร

HTTP เป็น protocol แบบ **stateless** — เซิร์ฟเวอร์ไม่จำอะไรเกี่ยวกับ request ก่อนหน้าเลย ทุก request คือคนละเรื่องกันโดยสิ้นเชิง ปัญหาคือเว็บแอปส่วนใหญ่ต้องรู้ว่า "ใครเป็นคนเรียก request นี้" เช่นตอนเรียก `/profile` หรือ `/orders` ต้องรู้ว่าเป็น request ของผู้ใช้คนไหนที่ล็อกอินอยู่

**JWT (JSON Web Token)** คือมาตรฐาน (RFC 7519) สำหรับสร้าง "token" ที่บรรจุข้อมูลผู้ใช้ในรูปแบบที่ **ตรวจสอบได้ว่าไม่ถูกแก้ไข (tamper-proof)** โดยไม่ต้องให้เซิร์ฟเวอร์เก็บ state ใดๆ ไว้เลย client ส่ง token กลับมาแนบทุก request แทนการต้อง login ซ้ำ และเซิร์ฟเวอร์ตรวจสอบ token นั้นได้ทันทีโดยไม่ต้อง query ฐานข้อมูลหรือ session store ใดๆ

จุดเด่นที่ทำให้ JWT ได้รับความนิยมมากในระบบสมัยใหม่ โดยเฉพาะ REST API และ microservices:

- **Stateless** — เซิร์ฟเวอร์ไม่ต้องเก็บ session ไว้ที่ใดเลย ทำให้ scale ระบบแนวนอน (horizontal scaling) ได้ง่ายกว่า เพราะ request ไปตกที่ server instance ไหนก็ตรวจสอบ token ได้เหมือนกันหมด
- **Self-contained** — ข้อมูลที่จำเป็น (เช่น user ID, role) อยู่ใน token เอง ไม่ต้อง query DB ทุกครั้งเพื่อรู้ว่า "นี่ใคร"
- **ใช้ข้าม domain/service ได้** — เหมาะกับสถาปัตยกรรม microservices ที่ต้องส่งต่อ "ตัวตนผู้ใช้" ระหว่างหลาย service (เราจะเจอแนวคิดนี้อีกครั้งใน **ภาคที่ 8: Microservices**)

เปรียบเทียบกับ session-based authentication แบบดั้งเดิม (ที่เซิร์ฟเวอร์เก็บ session ไว้และส่งแค่ session ID ให้ client) จะอธิบายเจาะลึกใน **Part 068** ถัดไป

---

## 2. โครงสร้าง JWT: header.payload.signature

JWT ที่สมบูรณ์หนึ่งตัวคือ string ยาวๆ ที่มี 3 ส่วนคั่นด้วยจุด (`.`):

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VybmFtZSI6Im5hdHRhcG9uZyIsInJvbGUiOiJhZG1pbiIsImV4cCI6MTc5MDM5MTk3NX0.kamh1sZlLKPnaZFdi320JWolD3HD_HTP0LF8wRw4rRk
└──────────── header ────────────┘ └────────────────── payload ──────────────────┘ └──────────── signature ──────────┘
```

แต่ละส่วนคือข้อมูลที่ผ่านการเข้ารหัสแบบ **Base64URL** (Base64 เวอร์ชันที่ใช้อักขระปลอดภัยสำหรับ URL) **ไม่ใช่การเข้ารหัสแบบปกปิดข้อมูล (encryption)**:

### Header — บอกว่าใช้ algorithm อะไรเซ็น

```json
{"alg": "HS256", "typ": "JWT"}
```

### Payload — ข้อมูล "claims" ที่เราต้องการฝากไปกับ token

```json
{"username": "nattapong", "role": "admin", "exp": 1790391975}
```

Claims มาตรฐานที่ JWT กำหนดไว้ (registered claims) ที่ควรรู้จัก:

| Claim | ความหมาย |
|---|---|
| `sub` | Subject — ปกติคือ user ID หรือ username |
| `exp` | Expiration Time — เวลาหมดอายุ (Unix timestamp) |
| `iat` | Issued At — เวลาที่ออก token |
| `iss` | Issuer — ใครเป็นคนออก token นี้ |
| `aud` | Audience — token นี้มีไว้ใช้กับใคร/service ไหน |

นอกจากนี้ยังใส่ **custom claims** ของเราเองได้ เช่น `role`, `email` ตามต้องการ

### Signature — ลายเซ็นที่ยืนยันว่า header+payload ไม่ถูกแก้ไข

```
HMACSHA256(base64UrlEncode(header) + "." + base64UrlEncode(payload), secretKey)
```

เซิร์ฟเวอร์เก็บ `secretKey` ไว้เป็นความลับ เมื่อได้รับ token กลับมาจะคำนวณ signature ใหม่จาก header+payload ที่ได้รับ แล้วเทียบกับ signature ที่แนบมา ถ้าไม่ตรงกัน แปลว่า token ถูกแก้ไข (หรือไม่ได้เซ็นด้วย secret key เดียวกัน) — ต้องปฏิเสธทันที

---

## 3. ความเข้าใจผิดที่พบบ่อยที่สุด: JWT ไม่ได้ถูกเข้ารหัส

นี่คือจุดที่ผู้เริ่มต้นเข้าใจผิดกันบ่อยมาก: **"เซ็น (sign)" ไม่ใช่ "เข้ารหัส (encrypt)"**

Header และ Payload ของ JWT ถูกเข้ารหัสด้วย **Base64URL เท่านั้น** ซึ่งเป็นการเข้ารหัสเพื่อให้ส่งผ่าน text-based protocol ได้อย่างปลอดภัย (คล้ายการ zip แล้วไม่ใส่รหัสผ่าน) **ไม่ใช่การปกปิดเนื้อหา** — ใครก็ตามที่ได้ token ไปสามารถ decode payload อ่านเนื้อหาข้างในได้ทันทีโดยไม่ต้องรู้ secret key เลย

ลองพิสูจน์ด้วยโค้ดจริง:

```go
package main

import (
	"encoding/base64"
	"fmt"
	"strings"
)

func main() {
	token := "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VybmFtZSI6Im5hdHRhcG9uZyIsInJvbGUiOiJhZG1pbiIsImV4cCI6MTc5MDM5MTk3NX0.kamh1sZlLKPnaZFdi320JWolD3HD_HTP0LF8wRw4rRk"

	parts := strings.Split(token, ".")
	payloadJSON, _ := base64.RawURLEncoding.DecodeString(parts[1])
	fmt.Println(string(payloadJSON))
}
```

รันแล้วได้ผลลัพธ์:

```
{"username":"nattapong","role":"admin","exp":1790391975}
```

**ไม่ต้องมี secret key ใดๆ เลยก็อ่านเนื้อหาได้** — ทดลองจริงด้วยตัวเองได้ที่ [jwt.io](https://jwt.io) วาง token ลงไปแล้วจะเห็น payload ทันที

สิ่งที่ secret key ปกป้องคือ **signature** เท่านั้น นั่นคือมันป้องกันไม่ให้ใครปลอมแปลง/แก้ไขเนื้อหาแล้วสร้าง token ปลอมที่ผ่านการตรวจสอบได้ — ไม่ได้ป้องกันการอ่านเนื้อหา

> **ข้อสรุปสำคัญที่ต้องจำ**: **ห้ามใส่ข้อมูลอ่อนไหว (sensitive data) เช่น รหัสผ่าน, เลขบัตรเครดิต, หรือข้อมูลส่วนตัวที่ไม่ควรเปิดเผย ลงใน payload ของ JWT เด็ดขาด** เพราะใครก็ตามที่ได้ token (เช่นดักจับผ่าน network, อ่านจาก browser dev tools, หรือดูจาก log) จะอ่านได้ทันที

ถ้าต้องการปกปิดเนื้อหาจริงๆ ต้องใช้ **JWE (JSON Web Encryption)** ซึ่งเป็นมาตรฐานแยกต่างหาก ไม่ใช่ JWS (JSON Web Signature — รูปแบบที่เราใช้กันทั่วไปและเรียกสั้นๆ ว่า "JWT" ในบทนี้)

---

## 4. ติดตั้ง `github.com/golang-jwt/jwt/v5`

Go standard library ไม่มี JWT ให้ในตัว (ต่างจาก `net/http`, `encoding/json` ที่มีมาให้) เราจึงต้องพึ่ง library ภายนอก ตัวที่นิยมและดูแลต่อเนื่องที่สุดในชุมชน Go คือ [`golang-jwt/jwt`](https://github.com/golang-jwt/jwt) (fork ต่อจาก `dgrijalva/jwt-go` เดิมที่เลิกดูแลแล้ว)

```bash
mkdir jwt-demo && cd jwt-demo
go mod init jwt-demo
go get github.com/golang-jwt/jwt/v5
```

ตรวจสอบว่าติดตั้งสำเร็จใน `go.mod`:

```
module jwt-demo

go 1.24.7

require github.com/golang-jwt/jwt/v5 v5.3.1
```

---

## 5. สร้างและเซ็น token พร้อม claims

```go
package main

import (
	"fmt"
	"time"

	"github.com/golang-jwt/jwt/v5"
)

var jwtSecret = []byte("super-secret-key-change-me-in-production")

// AppClaims รวม registered claims มาตรฐาน (jwt.RegisteredClaims)
// เข้ากับ custom claims ของเราเอง (Username, Role) ผ่าน struct embedding
// (แนวคิดเดียวกับที่เรียนใน Part 030 — Struct Embedding)
type AppClaims struct {
	Username string `json:"username"`
	Role     string `json:"role"`
	jwt.RegisteredClaims
}

func createToken(username, role string) (string, error) {
	claims := AppClaims{
		Username: username,
		Role:     role,
		RegisteredClaims: jwt.RegisteredClaims{
			Subject:   username,
			ExpiresAt: jwt.NewNumericDate(time.Now().Add(15 * time.Minute)),
			IssuedAt:  jwt.NewNumericDate(time.Now()),
			Issuer:    "go-course-app",
		},
	}
	token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
	return token.SignedString(jwtSecret)
}

func main() {
	token, err := createToken("nattapong", "admin")
	if err != nil {
		panic(err)
	}
	fmt.Println("Token:", token)
}
```

อธิบายทีละส่วน:

- **`jwt.RegisteredClaims`** เป็น struct ที่ library เตรียมไว้ให้แล้ว มี field มาตรฐานอย่าง `ExpiresAt`, `IssuedAt`, `Subject`, `Issuer` ครบตามสเปก RFC 7519 เรา embed มันเข้ากับ struct ของเราเองเพื่อได้ทั้ง claims มาตรฐานและ custom claims ในตัวเดียว
- **`jwt.SigningMethodHS256`** คือ algorithm HMAC-SHA256 — ใช้ secret key ตัวเดียวกันทั้งเซ็นและตรวจสอบ (เรียกว่า **symmetric signing**) เหมาะกับกรณีที่มีแค่เซิร์ฟเวอร์เดียว (หรือกลุ่มเซิร์ฟเวอร์ที่ไว้ใจกัน) เป็นทั้งคนออกและคนตรวจ token ถ้าต้องการให้หลาย service ตรวจสอบ token ได้โดยไม่ต้องรู้ secret (เช่นเปิด public key ให้ service อื่น verify ได้แต่เซ็นได้เฉพาะเจ้าของ private key) ให้ใช้ **RS256** (RSA) หรือ **ES256** (ECDSA) แทน ซึ่งเป็น **asymmetric signing**
- **`token.SignedString(jwtSecret)`** คำนวณ signature และประกอบร่างเป็น string รูปแบบ `header.payload.signature` ที่พร้อมส่งให้ client

รันแล้วได้ token จริง (ตัวอย่างจากการทดสอบจริง):

```
Token: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VybmFtZSI6Im5hdHRhcG9uZyIsInJvbGUiOiJhZG1pbiIsImlzcyI6ImdvLWNvdXJzZS1hcHAiLCJzdWIiOiJuYXR0YXBvbmciLCJleHAiOjE3OTAzOTE5NzUsImlhdCI6MTc5MDM5MTA3NX0.kamh1sZlLKPnaZFdi320JWolD3HD_HTP0LF8wRw4rRk
```

---

## 6. ตรวจสอบและ parse token, การหมดอายุ

```go
func parseToken(tokenString string) (*AppClaims, error) {
	claims := &AppClaims{}
	token, err := jwt.ParseWithClaims(tokenString, claims, func(t *jwt.Token) (interface{}, error) {
		// สำคัญมาก: ต้องตรวจสอบ algorithm ที่ token ใช้จริง ก่อนคืน secret key ไปให้
		if _, ok := t.Method.(*jwt.SigningMethodHMAC); !ok {
			return nil, fmt.Errorf("unexpected signing method: %v", t.Header["alg"])
		}
		return jwtSecret, nil
	})
	if err != nil {
		return nil, err
	}
	if !token.Valid {
		return nil, fmt.Errorf("invalid token")
	}
	return claims, nil
}
```

### ทำไมต้องเช็ค signing method ก่อน

เหตุผลนี้สำคัญมากในทางความปลอดภัย: header ของ JWT ที่ client ส่งมาระบุ algorithm ของตัวเอง (`{"alg": "HS256"}`) ถ้าเราไม่ตรวจสอบก่อนว่า algorithm ที่ token อ้างตรงกับที่เราคาดหวังจริงๆ ผู้โจมตีอาจส่ง token ที่ปลอม header ให้ `alg` เป็น `none` (JWT รองรับ "ไม่เซ็นเลย" ได้ตามสเปก!) หรือพยายามหลอกให้เราตรวจสอบด้วย public key แทน secret key ในกรณีระบบใช้ RS256 (ช่องโหว่ classic ที่เรียกว่า **algorithm confusion attack**) การเช็ค type ของ `t.Method` ก่อนเสมอคือแนวป้องกันมาตรฐานที่ library แนะนำ

### ทดสอบ token ที่หมดอายุและ token ที่ถูกปลอมแปลง

```go
// token หมดอายุ
expiredClaims := AppClaims{
	RegisteredClaims: jwt.RegisteredClaims{
		ExpiresAt: jwt.NewNumericDate(time.Now().Add(-1 * time.Hour)),
	},
}
expiredToken, _ := jwt.NewWithClaims(jwt.SigningMethodHS256, expiredClaims).SignedString(jwtSecret)
_, err := parseToken(expiredToken)
fmt.Println(err) // token has invalid claims: token is expired

// signature ถูกแก้ไข
parts := strings.Split(token, ".")
tampered := parts[0] + "." + parts[1] + ".AAAAtamperedAAAA"
_, err = parseToken(tampered)
fmt.Println(err) // token signature is invalid: signature is invalid
```

ผลลัพธ์จากการรันจริง:

```
ผลลัพธ์เมื่อ token หมดอายุ: token has invalid claims: token is expired
ผลลัพธ์เมื่อ signature ไม่ถูกต้อง: token signature is invalid: signature is invalid
```

`jwt.ParseWithClaims` ตรวจสอบ `exp` ให้อัตโนมัติ (เทียบกับเวลาปัจจุบัน) เราไม่ต้องเขียน logic เช็คหมดอายุเองเลย — ถ้า token หมดอายุหรือ signature ไม่ตรง ฟังก์ชันจะคืน error ทันที และ `token.Valid` จะเป็น `false`

---

## 7. Auth Middleware: อ่าน `Authorization: Bearer <token>`

จำรูปแบบ middleware ที่เรียนไปใน **Part 057** ได้ไหม — ฟังก์ชันที่รับ handler เข้ามาแล้วคืน handler ใหม่ที่ทำงานบางอย่างก่อน/หลังเรียก handler เดิม เราจะนำรูปแบบเดียวกันมาสร้าง **auth middleware** ที่ตรวจสอบ JWT ก่อนปล่อยให้ request เข้าถึง handler จริง:

```go
type ctxKey string

const claimsKey ctxKey = "claims"

func authMiddleware(next http.HandlerFunc) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		authHeader := r.Header.Get("Authorization")
		if authHeader == "" {
			http.Error(w, "ไม่พบ Authorization header", http.StatusUnauthorized)
			return
		}

		const prefix = "Bearer "
		if !strings.HasPrefix(authHeader, prefix) {
			http.Error(w, "รูปแบบ Authorization header ต้องเป็น 'Bearer <token>'", http.StatusUnauthorized)
			return
		}
		tokenString := strings.TrimPrefix(authHeader, prefix)

		claims, err := parseAccessToken(tokenString)
		if err != nil {
			http.Error(w, "token ไม่ถูกต้องหรือหมดอายุ: "+err.Error(), http.StatusUnauthorized)
			return
		}

		// ฝัง claims ไว้ใน context (แนวคิดจาก Part 032) เพื่อให้ handler ปลายทางดึงไปใช้ได้
		r = r.WithContext(context.WithValue(r.Context(), claimsKey, claims))
		next(w, r)
	}
}
```

รูปแบบ `Authorization: Bearer <token>` เป็นมาตรฐาน (RFC 6750) ที่ระบบส่วนใหญ่ใช้กัน คำว่า "Bearer" แปลว่า "ผู้ถือ" — ใครก็ตามที่ถือ token นี้อยู่ (bearer) จะถูกเชื่อว่าเป็นเจ้าของสิทธิ์นั้น ดังนั้นการป้องกันไม่ให้ token รั่วไหลจึงสำคัญมาก (ดูหัวข้อที่ 10)

การใช้งาน middleware นี้ทำได้เหมือนที่เรียนไปแล้วใน Part 057:

```go
mux.HandleFunc("/profile", authMiddleware(profileHandler))
```

---

## 8. Login Endpoint แบบสมบูรณ์: bcrypt + JWT

ย้อนกลับไป **Part 051 (Cryptography พื้นฐาน)** เราเรียนไปแล้วว่า **ห้ามเก็บรหัสผ่านแบบ plaintext เด็ดขาด** ต้อง hash ด้วย bcrypt ก่อนเก็บลงฐานข้อมูลเสมอ บทนี้จะเอาแนวคิดนั้นมาต่อกับ JWT เพื่อสร้าง flow การ login ที่สมบูรณ์:

```
1. ผู้ใช้ส่ง username + password มาที่ /login
2. เซิร์ฟเวอร์ค้นหา user แล้วเทียบ password กับ hash ที่เก็บไว้ด้วย bcrypt.CompareHashAndPassword
3. ถ้าถูกต้อง → สร้าง JWT ที่มี claims เช่น username, role พร้อมเวลาหมดอายุสั้นๆ
4. ส่ง JWT กลับไปให้ client เก็บไว้ (เช่นใน memory ของ frontend app)
5. ทุก request ถัดไปที่ต้อง authenticate ผู้ใช้แนบ header "Authorization: Bearer <token>" มาด้วย
```

ติดตั้ง bcrypt เพิ่ม (จาก `golang.org/x/crypto` ตามที่ใช้ใน Part 051):

```bash
go get golang.org/x/crypto/bcrypt
```

โค้ดสมบูรณ์:

```go
package main

import (
	"encoding/json"
	"errors"
	"fmt"
	"net/http"
	"strings"
	"sync"
	"time"

	"github.com/golang-jwt/jwt/v5"
	"golang.org/x/crypto/bcrypt"
)

var jwtSecret = []byte("super-secret-key-change-me-in-production")

// ---------- "database" ผู้ใช้แบบง่ายๆ ในหน่วยความจำ ----------

type user struct {
	Username     string
	PasswordHash string // เก็บ hash ของรหัสผ่านเท่านั้น ไม่เก็บ plaintext เด็ดขาด (ดู Part 051)
	Role         string
}

var userStore = struct {
	sync.RWMutex
	users map[string]user
}{users: map[string]user{}}

func registerUser(username, plainPassword, role string) error {
	hash, err := bcrypt.GenerateFromPassword([]byte(plainPassword), bcrypt.DefaultCost)
	if err != nil {
		return err
	}
	userStore.Lock()
	defer userStore.Unlock()
	userStore.users[username] = user{Username: username, PasswordHash: string(hash), Role: role}
	return nil
}

func checkPassword(username, plainPassword string) (user, error) {
	userStore.RLock()
	u, ok := userStore.users[username]
	userStore.RUnlock()
	if !ok {
		return user{}, errors.New("ไม่พบผู้ใช้")
	}
	if err := bcrypt.CompareHashAndPassword([]byte(u.PasswordHash), []byte(plainPassword)); err != nil {
		return user{}, errors.New("รหัสผ่านไม่ถูกต้อง")
	}
	return u, nil
}

// ---------- JWT ----------

type AppClaims struct {
	Role string `json:"role"`
	jwt.RegisteredClaims
}

func createAccessToken(u user) (string, error) {
	claims := AppClaims{
		Role: u.Role,
		RegisteredClaims: jwt.RegisteredClaims{
			Subject:   u.Username,
			ExpiresAt: jwt.NewNumericDate(time.Now().Add(15 * time.Minute)),
			IssuedAt:  jwt.NewNumericDate(time.Now()),
		},
	}
	token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
	return token.SignedString(jwtSecret)
}

func parseAccessToken(tokenString string) (*AppClaims, error) {
	claims := &AppClaims{}
	token, err := jwt.ParseWithClaims(tokenString, claims, func(t *jwt.Token) (interface{}, error) {
		if _, ok := t.Method.(*jwt.SigningMethodHMAC); !ok {
			return nil, fmt.Errorf("unexpected signing method: %v", t.Header["alg"])
		}
		return jwtSecret, nil
	})
	if err != nil || !token.Valid {
		return nil, fmt.Errorf("invalid token: %w", err)
	}
	return claims, nil
}

// ---------- HTTP handlers ----------

type loginRequest struct {
	Username string `json:"username"`
	Password string `json:"password"`
}

type loginResponse struct {
	AccessToken string `json:"access_token"`
	TokenType   string `json:"token_type"`
	ExpiresIn   int    `json:"expires_in"`
}

func loginHandler(w http.ResponseWriter, r *http.Request) {
	var req loginRequest
	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		http.Error(w, "รูปแบบ request ไม่ถูกต้อง", http.StatusBadRequest)
		return
	}

	u, err := checkPassword(req.Username, req.Password)
	if err != nil {
		// ตอบข้อความเดียวกันไม่ว่าจะ "ไม่พบผู้ใช้" หรือ "รหัสผิด"
		// เพื่อไม่ให้แฮกเกอร์รู้ว่า username ไหนมีอยู่จริงในระบบ (user enumeration)
		http.Error(w, "username หรือ password ไม่ถูกต้อง", http.StatusUnauthorized)
		return
	}

	token, err := createAccessToken(u)
	if err != nil {
		http.Error(w, "สร้าง token ไม่สำเร็จ", http.StatusInternalServerError)
		return
	}

	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(loginResponse{
		AccessToken: token,
		TokenType:   "Bearer",
		ExpiresIn:   15 * 60,
	})
}
```

และ handler ที่ต้อง authenticate พร้อม role-based access control ง่ายๆ:

```go
func profileHandler(w http.ResponseWriter, r *http.Request) {
	claims := r.Context().Value(claimsKey).(*AppClaims)
	fmt.Fprintf(w, "สวัสดีคุณ %s (role: %s)\n", claims.Subject, claims.Role)
}

func adminOnlyHandler(w http.ResponseWriter, r *http.Request) {
	claims := r.Context().Value(claimsKey).(*AppClaims)
	if claims.Role != "admin" {
		http.Error(w, "ต้องเป็น admin เท่านั้น", http.StatusForbidden)
		return
	}
	fmt.Fprintf(w, "ยินดีต้อนรับแอดมิน %s\n", claims.Subject)
}

func main() {
	registerUser("nattapong", "s3cret-password", "admin")
	registerUser("somsri", "another-password", "member")

	mux := http.NewServeMux()
	mux.HandleFunc("/login", loginHandler)
	mux.HandleFunc("/profile", authMiddleware(profileHandler))
	mux.HandleFunc("/admin", authMiddleware(adminOnlyHandler))

	http.ListenAndServe(":8080", mux)
}
```

โค้ดชุดนี้ผ่านการทดสอบจริงด้วย `httptest` แล้วครบทุก flow:

```
1. login ด้วย password ผิด             -> 401 Unauthorized
2. login ถูกต้อง                       -> 200 พร้อม access_token
3. เรียก /profile โดยไม่มี token        -> 401 Unauthorized
4. เรียก /profile พร้อม token ที่ถูกต้อง -> 200 "สวัสดีคุณ nattapong (role: admin)"
5. admin เรียก /admin                  -> 200 สำเร็จ
6. member เรียก /admin                 -> 403 Forbidden (role ไม่พอ)
7. token ที่ถูกแก้ไข (tampered)         -> 401 Unauthorized
```

ทดสอบด้วย `curl` จริง:

```bash
curl -X POST http://localhost:8080/login \
  -H "Content-Type: application/json" \
  -d '{"username":"nattapong","password":"s3cret-password"}'
# {"access_token":"eyJhbGci...","token_type":"Bearer","expires_in":900}

curl http://localhost:8080/profile \
  -H "Authorization: Bearer eyJhbGci..."
# สวัสดีคุณ nattapong (role: admin)
```

---

## 9. Refresh Token Pattern

สังเกตว่าเราตั้งเวลาหมดอายุของ access token ไว้สั้นมาก (15 นาที) เพื่อความปลอดภัย (ดูเหตุผลในหัวข้อที่ 10) แต่การให้ผู้ใช้ล็อกอินใหม่ทุก 15 นาทีย่อมสร้างประสบการณ์ที่แย่มาก แนวทางมาตรฐานที่แก้ปัญหานี้คือ **refresh token pattern**:

- **Access Token** — อายุสั้น (นาทีถึงชั่วโมง) ใช้แนบไปกับทุก request เพื่อยืนยันตัวตน
- **Refresh Token** — อายุยาว (วันถึงสัปดาห์) เก็บไว้อย่างปลอดภัยกว่า (เช่นใน HttpOnly cookie) ใช้แลก access token ใหม่เมื่อ access token หมดอายุ **โดยไม่ต้องให้ผู้ใช้กรอก username/password ซ้ำ**

Flow คร่าวๆ:

```
1. Login สำเร็จ → เซิร์ฟเวอร์ออกทั้ง access token (อายุสั้น) และ refresh token (อายุยาว)
2. Client ใช้ access token เรียก API ตามปกติ
3. เมื่อ access token หมดอายุ (เซิร์ฟเวอร์ตอบ 401) → client เรียก /refresh พร้อมแนบ refresh token
4. เซิร์ฟเวอร์ตรวจสอบ refresh token ว่ายังไม่หมดอายุและไม่ถูกเพิกถอน → ออก access token ใหม่ให้
5. ถ้า refresh token เองก็หมดอายุ/ถูกเพิกถอนแล้ว → ต้อง login ใหม่จริงๆ
```

ตัวอย่าง endpoint คร่าวๆ (โครงสร้าง ไม่ใช่โค้ดที่รันได้สมบูรณ์ในบทนี้ เพราะต้องมี store สำหรับ refresh token ที่เพิกถอนได้ ซึ่งมักใช้ฐานข้อมูลหรือ Redis — จะเรียนเจาะลึกใน **ภาคที่ 6: Database** และ **Part 077: Redis**):

```go
func refreshHandler(w http.ResponseWriter, r *http.Request) {
	var req struct{ RefreshToken string `json:"refresh_token"` }
	json.NewDecoder(r.Body).Decode(&req)

	// ตรวจสอบ refresh token กับ store (DB/Redis) ว่ายังใช้ได้อยู่ไหม
	// ถ้าผ่าน ให้ออก access token ใหม่ (และอาจหมุน refresh token ใหม่ด้วย
	// เพื่อลดความเสี่ยงถ้า refresh token เดิมรั่วไหล — เรียกว่า "refresh token rotation")
}
```

**สาเหตุที่ refresh token ต้อง "เพิกถอนได้" (revocable)** ต่างจาก access token ที่มักปล่อยให้หมดอายุเองเพราะอายุสั้นอยู่แล้ว: refresh token มีอายุยาวและมีอำนาจมาก (แลก access token ใหม่ได้เรื่อยๆ) จึงจำเป็นต้องมีทางให้เซิร์ฟเวอร์ "ยกเลิก" มันได้ทันทีถ้าสงสัยว่ารั่วไหล (เช่นตอนผู้ใช้กด logout หรือเปลี่ยนรหัสผ่าน) ซึ่งขัดกับธรรมชาติ stateless ของ JWT ล้วนๆ ในทางปฏิบัติ ระบบส่วนใหญ่จึงเก็บ refresh token (หรือ ID ของมัน) ไว้ในฐานข้อมูลเพื่อให้ตรวจสอบ/เพิกถอนได้ ในขณะที่ปล่อยให้ access token เป็น stateless เต็มรูปแบบตามเดิม

---

## 10. ข้อควรระวังด้านความปลอดภัย

1. **Access token ต้องมีอายุสั้น** — เพราะเป็น stateless เมื่อออกไปแล้วเซิร์ฟเวอร์ "เพิกถอน" มันไม่ได้ (นอกจากจะเพิ่มระบบ blacklist ซึ่งขัดกับจุดเด่นเรื่อง stateless) อายุสั้นช่วยจำกัดความเสียหายถ้า token รั่วไหล
2. **ห้ามใส่ข้อมูลอ่อนไหวใน payload** — ตามที่อธิบายในหัวข้อที่ 3 เพราะใครก็อ่านได้โดยไม่ต้องมี secret key
3. **ใช้ HTTPS เสมอ** — ถ้าส่ง token ผ่าน HTTP ธรรมดา ใครก็ตามที่ดักฟัง network (เช่นบน public WiFi) จะเห็น token แบบ plain text ทันทีและนำไปใช้แอบอ้างตัวตนได้เลย (เรียกว่า **token theft**) HTTPS เข้ารหัสทั้ง connection ทำให้ดักฟังไม่ได้
4. **เก็บ secret key อย่างปลอดภัย** — ห้าม hardcode ลงโค้ดแล้ว commit ขึ้น git repository จริง (ตัวอย่างในบทนี้ hardcode ไว้เพื่อความง่ายในการสาธิตเท่านั้น) ควรอ่านจาก environment variable หรือ secret manager เสมอ
5. **ตรวจสอบ signing algorithm ทุกครั้ง** — ป้องกัน algorithm confusion attack ตามที่อธิบายในหัวข้อที่ 6
6. **Client เก็บ token ให้ปลอดภัย** — ถ้าเป็นเว็บแอป หลีกเลี่ยงการเก็บ JWT ใน `localStorage` เพราะเสี่ยงถูกอ่านผ่าน XSS หากมีช่องโหว่ในหน้าเว็บ ทางเลือกที่ปลอดภัยกว่าคือ HttpOnly cookie (แลกกับต้องจัดการเรื่อง CSRF เพิ่ม) — เราจะเจอแนวคิด HttpOnly cookie อีกครั้งใน **Part 068**
7. **ทุก endpoint ที่ authenticate แล้ว ควรตรวจสอบ authorization (สิทธิ์) แยกต่างหากเสมอ** — authentication (ยืนยันว่าเป็นใคร) กับ authorization (ตรวจว่ามีสิทธิ์ทำอะไรได้บ้าง) เป็นคนละเรื่องกัน อย่าลืมเช็ค role/permission อย่างที่ทำใน `adminOnlyHandler`

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- JWT คือ token รูปแบบ `header.payload.signature` ที่เข้ารหัสด้วย Base64URL (ไม่ใช่ encryption) และมี signature ป้องกันการปลอมแปลง
- **ความเข้าใจผิดที่พบบ่อยที่สุด**: JWT ไม่ได้ถูกเข้ารหัส ใครก็ตามที่มี token อ่าน payload ได้เสมอโดยไม่ต้องรู้ secret — ห้ามใส่ข้อมูลอ่อนไหวลงไป
- ใช้ `github.com/golang-jwt/jwt/v5` สร้าง token ด้วย `jwt.NewWithClaims` + `SignedString` และตรวจสอบด้วย `jwt.ParseWithClaims`
- ต้องตรวจสอบ signing algorithm ก่อนคืน secret key เสมอ เพื่อป้องกัน algorithm confusion attack
- Auth middleware อ่าน header `Authorization: Bearer <token>` ตาม pattern เดียวกับที่เรียนใน Part 057 แล้วฝัง claims ไว้ใน context ตาม Part 032
- Login endpoint ที่สมบูรณ์รวม bcrypt (Part 051) สำหรับตรวจรหัสผ่าน เข้ากับ JWT สำหรับออก token
- Refresh token pattern ช่วยให้ access token อายุสั้นได้โดยไม่ต้องให้ผู้ใช้ login ซ้ำบ่อยๆ แต่ refresh token เองต้องเก็บไว้แบบที่เพิกถอนได้
- ต้องใช้ HTTPS เสมอ, ตั้งอายุ token ให้สั้น, และเก็บ secret key อย่างปลอดภัยนอกโค้ด

## แบบฝึกหัดท้ายบท

1. เขียนโปรแกรมสร้าง JWT แล้ว decode payload ด้วยมือ (ใช้ `encoding/base64`) โดยไม่ใช้ library JWT ใดๆ ในการอ่าน เพื่อพิสูจน์ด้วยตัวเองว่า payload อ่านได้จริงโดยไม่ต้องมี secret key
2. เพิ่ม claim `email` ลงใน `AppClaims` แล้วแก้ `createAccessToken`/`profileHandler` ให้ใช้งาน field นี้ได้
3. ทดลองเปลี่ยนเวลาหมดอายุของ token เป็น 2 วินาที แล้วเขียนโค้ดทดสอบว่าหลังรอ 3 วินาที `parseAccessToken` คืน error ตามคาดจริงหรือไม่
4. เพิ่ม endpoint `/refresh` ที่ยังไม่ต้องมี store จริง (ใช้ map ในหน่วยความจำเก็บ refresh token ที่ยังไม่ถูกเพิกถอนก็พอ) ให้ผู้ใช้แลก refresh token เป็น access token ใหม่ได้
5. ลองแก้ `authMiddleware` ให้จงใจไม่ตรวจสอบ signing algorithm (ลบ `if _, ok := t.Method.(*jwt.SigningMethodHMAC); !ok` ออก) แล้วค้นคว้าเพิ่มเติมว่า algorithm confusion attack แบบ `alg: none` โจมตีระบบที่ไม่ตรวจสอบ algorithm ได้อย่างไร
6. เพิ่ม role ที่สามระดับ (`admin`, `editor`, `viewer`) พร้อม middleware ตรวจสอบสิทธิ์แบบยืดหยุ่นที่รับ list ของ role ที่อนุญาตเป็น parameter เช่น `requireRole("admin", "editor")`

---

**ต่อไป**: [Part 068 — Authentication ด้วย OAuth2 และ Session](./068-oauth2-and-session.md)
