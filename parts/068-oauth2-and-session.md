# Part 068: Authentication ด้วย OAuth2 และ Session

> ภาคที่ 5: Web Development — ตอนที่ 13 จาก 15 (Part 56–70)

## สารบัญของบทนี้

1. OAuth2 คืออะไร แก้ปัญหาอะไร
2. Authorization Code Flow อธิบายทีละขั้นตอน
3. ติดตั้ง `golang.org/x/oauth2`
4. ทำ "Login with Google" ด้วย Go
5. หมายเหตุสำคัญ: สิ่งที่ทดสอบได้และทดสอบไม่ได้ในสภาพแวดล้อมนี้
6. Token-based Auth (JWT) เทียบกับ Session แบบดั้งเดิม
7. Cookie-based Session ด้วย `net/http`
8. In-Memory Session Store
9. เชื่อม OAuth2 เข้ากับ Session: flow ที่สมบูรณ์
10. PKCE: การป้องกันเพิ่มเติมสำหรับ Public Client
11. จะเลือก Session หรือ JWT ดี
12. สรุปสิ่งที่ได้เรียนในบทนี้
13. แบบฝึกหัดท้ายบท

---

## 1. OAuth2 คืออะไร แก้ปัญหาอะไร

ลองนึกภาพปุ่ม "Sign in with Google" หรือ "Login with GitHub" ที่เห็นได้ทั่วไปตามเว็บไซต์ต่างๆ นี่คือการใช้ **OAuth2** — มาตรฐานเปิด (open standard) สำหรับการ **มอบสิทธิ์ (authorization)** ให้แอปหนึ่งเข้าถึงข้อมูลบางส่วนของผู้ใช้ที่อยู่บนอีกระบบหนึ่ง โดยที่**ผู้ใช้ไม่ต้องบอกรหัสผ่านของบัญชี Google/GitHub ให้แอปของเรารู้เลย**

ปัญหาที่ OAuth2 แก้:

- **ก่อนมี OAuth2**: ถ้าอยากให้แอป A เข้าถึงรายชื่อ contact ใน Gmail ของผู้ใช้ วิธีเดียวคือให้ผู้ใช้กรอก username/password ของ Gmail ให้แอป A โดยตรง — อันตรายมาก เพราะแอป A ได้รหัสผ่านเต็มไปเลย เอาไปทำอะไรก็ได้ ไม่ใช่แค่สิ่งที่ผู้ใช้ตั้งใจอนุญาต
- **ด้วย OAuth2**: ผู้ใช้ล็อกอินที่ Google โดยตรง (ไม่ผ่านแอป A เลย) แล้ว Google ออก "token" ที่มีสิทธิ์จำกัดเฉพาะสิ่งที่ผู้ใช้กดยินยอม (เช่น "อ่านอีเมลได้อย่างเดียว") ให้แอป A ไปใช้แทน แอป A ไม่มีทางรู้รหัสผ่านจริงของผู้ใช้เลย

สาม role หลักในระบบ OAuth2:

| Role | คือใคร | ตัวอย่าง |
|---|---|---|
| **Resource Owner** | เจ้าของข้อมูล | ผู้ใช้ที่กำลังล็อกอิน |
| **Client** | แอปที่ต้องการเข้าถึงข้อมูล | เว็บแอปของเรา |
| **Authorization Server** | ระบบที่ออก token | Google, GitHub, Facebook |
| **Resource Server** | ระบบที่เก็บข้อมูลจริงและตรวจสอบ token | Google API (เช่น userinfo endpoint) |

---

## 2. Authorization Code Flow อธิบายทีละขั้นตอน

OAuth2 มีหลาย "flow" (เรียกว่า grant type) แต่แบบที่ใช้กันมากที่สุดสำหรับเว็บแอปคือ **Authorization Code Flow** ลองไล่ทีละขั้นตอนเหมือนเห็นภาพจริง:

```
ผู้ใช้                    เว็บแอปของเรา (Client)              Google (Authorization Server)
  │                              │                                      │
  │  1. กดปุ่ม "Login with       │                                      │
  │     Google"                 │                                      │
  │─────────────────────────────>                                      │
  │                              │  2. Redirect ผู้ใช้ไปหน้า consent    │
  │                              │     ของ Google พร้อม client_id,      │
  │                              │     redirect_uri, scope, state       │
  │<─────────────────────────────│──────────────────────────────────────>
  │                              │                                      │
  │  3. ผู้ใช้ล็อกอินที่ Google  │                                      │
  │     โดยตรง (ไม่ผ่านแอปเรา)   │                                      │
  │     แล้วกด "อนุญาต"          │                                      │
  │─────────────────────────────────────────────────────────────────────>
  │                              │                                      │
  │                              │  4. Google redirect กลับมาที่        │
  │                              │     redirect_uri พร้อม ?code=xxx     │
  │<──────────────────────────────────────────────────────────────────── │
  │─────────────────────────────>│                                      │
  │                              │  5. Client เอา code ไปแลก            │
  │                              │     access token (เรียกตรงๆ           │
  │                              │     server-to-server ไม่ผ่าน browser) │
  │                              │─────────────────────────────────────>│
  │                              │  6. Google ตอบ access token กลับมา   │
  │                              │<─────────────────────────────────────│
  │                              │                                      │
  │                              │  7. Client เอา access token ไปเรียก   │
  │                              │     userinfo endpoint เพื่อดึงข้อมูล   │
  │                              │     ผู้ใช้ (email, ชื่อ, รูปโปรไฟล์)   │
  │                              │─────────────────────────────────────>│
  │                              │<─────────────────────────────────────│
  │  8. Client สร้าง session/JWT │                                      │
  │     ของตัวเองให้ผู้ใช้        │                                      │
  │<─────────────────────────────│                                      │
```

จุดสำคัญที่ต้องสังเกต:

- **ขั้นตอนที่ 3 (การล็อกอินจริง) เกิดขึ้นที่ Google โดยตรง** เว็บแอปของเราไม่เห็นรหัสผ่านของผู้ใช้เลยแม้แต่ตัวอักษรเดียว
- **`code` ที่ได้ในขั้นตอนที่ 4 ใช้ได้ครั้งเดียวและหมดอายุเร็วมาก** (มักไม่กี่นาที) มันไม่ใช่ access token ตัวจริง แค่เป็น "ตั๋วแลก" ที่ต้องเอาไปแลกอีกทีในขั้นตอนที่ 5
- **`state`** คือค่าสุ่มที่เราสร้างเองในขั้นตอนที่ 2 แล้วตรวจสอบซ้ำในขั้นตอนที่ 4 ว่าตรงกัน เพื่อป้องกัน **CSRF attack** — ถ้าไม่มี state ผู้โจมตีอาจหลอกให้เหยื่อกด link ที่มี code ของผู้โจมตีเอง ทำให้เหยื่อ "ล็อกอินเข้าบัญชีของผู้โจมตี" โดยไม่รู้ตัว (เรียกว่า **session fixation / login CSRF**)
- **ขั้นตอนที่ 8 คือส่วนที่เราต้องทำเอง**: หลังจากรู้ว่าผู้ใช้คือใคร (จาก userinfo) เว็บแอปของเราต้องสร้าง "ตัวตนในระบบของเราเอง" ขึ้นมา ไม่ว่าจะเป็น JWT (Part 067) หรือ session cookie (หัวข้อที่ 7 ในบทนี้)

---

## 3. ติดตั้ง `golang.org/x/oauth2`

Go มี package กึ่งทางการ (จากทีม Go เอง แต่แยกจาก standard library หลัก) ชื่อ `golang.org/x/oauth2` ที่ implement OAuth2 client ให้ครบ พร้อม endpoint สำเร็จรูปของผู้ให้บริการรายใหญ่ (`golang.org/x/oauth2/google`, `.../github`, `.../facebook` ฯลฯ)

```bash
mkdir oauth-demo && cd oauth-demo
go mod init oauth-demo
go get golang.org/x/oauth2
go get golang.org/x/oauth2/google
```

---

## 4. ทำ "Login with Google" ด้วย Go

ก่อนเขียนโค้ด ต้องไปสร้าง **OAuth 2.0 Client ID** ที่ [Google Cloud Console](https://console.cloud.google.com/) ก่อน (เมนู APIs & Services > Credentials > Create Credentials > OAuth client ID) จะได้ `Client ID` และ `Client Secret` มา พร้อมต้องระบุ **Authorized redirect URI** ให้ตรงกับที่โค้ดของเราใช้

```go
package main

import (
	"crypto/rand"
	"encoding/base64"
	"encoding/json"
	"fmt"
	"log"
	"net/http"
	"os"
	"sync"

	"golang.org/x/oauth2"
	"golang.org/x/oauth2/google"
)

// googleOAuthConfig สร้าง config จาก Client ID/Secret ที่ได้จาก Google Cloud Console
// ใน production ควรอ่านค่าจาก environment variable หรือ secret manager ไม่ hardcode ลงโค้ด
func googleOAuthConfig() *oauth2.Config {
	return &oauth2.Config{
		ClientID:     os.Getenv("GOOGLE_CLIENT_ID"),
		ClientSecret: os.Getenv("GOOGLE_CLIENT_SECRET"),
		RedirectURL:  "http://localhost:8080/auth/google/callback",
		Scopes: []string{
			"https://www.googleapis.com/auth/userinfo.email",
			"https://www.googleapis.com/auth/userinfo.profile",
		},
		Endpoint: google.Endpoint, // preset ที่ x/oauth2/google เตรียมไว้ให้แล้ว
	}
}

// เก็บ "state" ชั่วคราวไว้ตรวจสอบตอน callback กัน CSRF (ดูหัวข้อที่ 2)
var (
	stateMu    sync.Mutex
	pendingSet = map[string]bool{}
)

func randomState() (string, error) {
	b := make([]byte, 16)
	if _, err := rand.Read(b); err != nil {
		return "", err
	}
	return base64.URLEncoding.EncodeToString(b), nil
}

// ขั้นที่ 1-2: redirect ผู้ใช้ไปหน้า consent ของ Google พร้อม state แบบสุ่ม
func handleGoogleLogin(w http.ResponseWriter, r *http.Request) {
	cfg := googleOAuthConfig()
	state, err := randomState()
	if err != nil {
		http.Error(w, "สร้าง state ไม่สำเร็จ", http.StatusInternalServerError)
		return
	}
	stateMu.Lock()
	pendingSet[state] = true
	stateMu.Unlock()

	url := cfg.AuthCodeURL(state, oauth2.AccessTypeOffline)
	http.Redirect(w, r, url, http.StatusTemporaryRedirect)
}

type googleUserInfo struct {
	ID      string `json:"id"`
	Email   string `json:"email"`
	Name    string `json:"name"`
	Picture string `json:"picture"`
}

// ขั้นที่ 4-7: Google redirect กลับมาที่นี่พร้อม ?code=...&state=...
// เราเอา code ไปแลก access token แล้วใช้ token นั้นเรียก userinfo endpoint
func handleGoogleCallback(w http.ResponseWriter, r *http.Request) {
	state := r.URL.Query().Get("state")
	stateMu.Lock()
	ok := pendingSet[state]
	delete(pendingSet, state)
	stateMu.Unlock()
	if !ok {
		http.Error(w, "state ไม่ถูกต้อง (อาจถูกโจมตีแบบ CSRF)", http.StatusBadRequest)
		return
	}

	code := r.URL.Query().Get("code")
	if code == "" {
		http.Error(w, "ไม่พบ authorization code", http.StatusBadRequest)
		return
	}

	cfg := googleOAuthConfig()
	token, err := cfg.Exchange(r.Context(), code) // ขั้นที่ 5-6
	if err != nil {
		http.Error(w, "แลก token ไม่สำเร็จ: "+err.Error(), http.StatusInternalServerError)
		return
	}

	client := cfg.Client(r.Context(), token) // http.Client ที่แนบ access token ให้ทุก request อัตโนมัติ
	resp, err := client.Get("https://www.googleapis.com/oauth2/v2/userinfo") // ขั้นที่ 7
	if err != nil {
		http.Error(w, "ดึงข้อมูลผู้ใช้ไม่สำเร็จ: "+err.Error(), http.StatusInternalServerError)
		return
	}
	defer resp.Body.Close()

	var info googleUserInfo
	if err := json.NewDecoder(resp.Body).Decode(&info); err != nil {
		http.Error(w, "แปลงข้อมูลผู้ใช้ไม่สำเร็จ", http.StatusInternalServerError)
		return
	}

	// ขั้นที่ 8: จากตรงนี้สร้าง session หรือ JWT ของแอปเราเอง ผูกกับ info.Email/info.ID
	// (ดูตัวอย่าง session ในหัวข้อที่ 7-8 หรือ JWT ใน Part 067)
	fmt.Fprintf(w, "ล็อกอินสำเร็จ: %s (%s)\n", info.Name, info.Email)
}

func main() {
	mux := http.NewServeMux()
	mux.HandleFunc("/auth/google/login", handleGoogleLogin)
	mux.HandleFunc("/auth/google/callback", handleGoogleCallback)

	log.Println("listening on :8080 (ต้องตั้งค่า GOOGLE_CLIENT_ID/GOOGLE_CLIENT_SECRET จริงก่อนใช้งาน)")
	log.Fatal(http.ListenAndServe(":8080", mux))
}
```

โค้ดชุดนี้ **compile และ vet ผ่านสมบูรณ์** (ยืนยันด้วย `go build`/`go vet` จริงระหว่างเขียนบทเรียนนี้) โครงสร้างทุกจุดถูกต้องตามที่ `golang.org/x/oauth2` คาดหวัง

---

## 5. หมายเหตุสำคัญ: สิ่งที่ทดสอบได้และทดสอบไม่ได้ในสภาพแวดล้อมนี้

**สิ่งที่ต้องเข้าใจให้ชัดเจน**: โค้ด OAuth2 ด้านบนไม่สามารถรันจนจบ flow แบบ end-to-end ได้ในสภาพแวดล้อมทดลอง (เช่น sandbox, CI, หรือเครื่องที่ไม่มี browser) เพราะ **ขั้นตอนที่ 2-4 ต้องการ browser จริงที่ผู้ใช้คนหนึ่งล็อกอินเข้า Google ด้วยบัญชีจริง** และ **ต้องมี Client ID/Secret จริงที่ลงทะเบียนไว้กับ Google Cloud Console** ซึ่งเป็นข้อมูลเฉพาะของแต่ละโปรเจกต์ ไม่มีทาง mock หรือจำลองส่วนนี้ได้อย่างถูกต้อง (การ mock จะทำให้ไม่ใช่การทดสอบ OAuth2 จริงอีกต่อไป)

**สิ่งที่ยืนยันได้และยืนยันแล้วในบทเรียนนี้**:

- โค้ดทั้งหมด compile ผ่าน ไม่มี syntax error หรือ type error
- โครงสร้างการเรียกใช้ `oauth2.Config`, `AuthCodeURL`, `Exchange`, `Client` ตรงตาม API จริงของ library
- `google.Endpoint` คือค่าคงที่ที่ `golang.org/x/oauth2/google` เตรียมไว้ให้ ชี้ไปที่ authorization endpoint และ token endpoint ของ Google จริง

**วิธีทดสอบให้ครบ flow จริงด้วยตัวเอง** (ทำนอกบทเรียนนี้ เมื่อพร้อมทำโปรเจกต์จริง):

1. สร้างโปรเจกต์ใน [Google Cloud Console](https://console.cloud.google.com/) แล้วเปิดใช้ OAuth consent screen
2. สร้าง OAuth 2.0 Client ID ประเภท "Web application" ระบุ Authorized redirect URI เป็น `http://localhost:8080/auth/google/callback`
3. ตั้งค่า environment variable `GOOGLE_CLIENT_ID` และ `GOOGLE_CLIENT_SECRET` จากค่าที่ได้
4. รันโปรแกรม แล้วเปิด browser ไปที่ `http://localhost:8080/auth/google/login`
5. ล็อกอินด้วยบัญชี Google จริง กด "อนุญาต" แล้วสังเกตว่าถูก redirect กลับมาพร้อมข้อมูลผู้ใช้จริงที่ endpoint `/auth/google/callback`

หลักการเขียนโค้ดให้ "ถูกต้องแม้ยังทดสอบ end-to-end ไม่ได้" แบบนี้เป็นทักษะสำคัญที่นักพัฒนา Go มืออาชีพต้องมี — เขียนตาม API contract ให้ถูกต้อง ตรวจสอบด้วย `go build`/`go vet` และเขียน unit test เฉพาะส่วนที่ทดสอบได้จริง (เช่น การสร้าง/ตรวจสอบ `state`, การ parse response JSON) ส่วนที่พึ่งพา external service ค่อยทดสอบแบบ integration test แยกต่างหากเมื่อมี credential จริง

---

## 6. Token-based Auth (JWT) เทียบกับ Session แบบดั้งเดิม

หลังจากได้ตัวตนผู้ใช้มาแล้ว (ไม่ว่าจะจาก OAuth2 หรือ login ฟอร์มธรรมดาแบบ Part 067) เว็บแอปต้อง "จำ" ว่าผู้ใช้คนนี้ล็อกอินอยู่สำหรับ request ถัดๆ ไป มีสองแนวทางหลัก:

| | **JWT (Part 067)** | **Session-based (บทนี้)** |
|---|---|---|
| **State อยู่ที่ไหน** | อยู่ใน token เอง (client ถือ) | อยู่ที่เซิร์ฟเวอร์ (server เก็บ) client ถือแค่ session ID |
| **Stateless?** | ใช่ เซิร์ฟเวอร์ไม่ต้องเก็บอะไร | ไม่ เซิร์ฟเวอร์ต้องมี store เก็บ session |
| **เพิกถอนก่อนหมดอายุ (revoke)** | ทำไม่ได้ตรงๆ (ต้องมี blacklist เพิ่ม) | ทำได้ทันที แค่ลบ session ออกจาก store |
| **Scale หลาย server** | ง่าย เพราะไม่มี state ต้องแชร์ | ต้องใช้ store กลาง (เช่น Redis) ถ้ามีหลาย instance |
| **ขนาดข้อมูลที่ client ส่งทุก request** | ใหญ่กว่า (ทั้ง header+payload+signature) | เล็กมาก (แค่ session ID) |
| **เหมาะกับ** | REST API, mobile app, microservices, SPA ที่แยก frontend/backend ชัดเจน | เว็บแอปแบบดั้งเดิม (server-rendered), ระบบที่ต้อง revoke ได้ทันที |

ข้อแตกต่างที่สำคัญที่สุดคือเรื่อง **การเพิกถอน (revocation)** — session ลบออกจาก store ได้ทันทีเมื่อผู้ใช้ logout หรือแอดมินต้องการบังคับให้ออกจากระบบ ในขณะที่ JWT ที่ออกไปแล้วจะใช้ได้จนกว่าจะหมดอายุเองเสมอ (นี่คือเหตุผลที่ Part 067 เน้นย้ำให้ตั้งอายุ JWT ให้สั้น)

---

## 7. Cookie-based Session ด้วย `net/http`

Session-based authentication ทำงานตามหลักการนี้:

```
1. Login สำเร็จ → เซิร์ฟเวอร์สร้าง session ID แบบสุ่ม เก็บข้อมูลผู้ใช้ไว้ใน store (map/DB/Redis)
   คู่กับ session ID นั้น แล้วส่ง session ID กลับไปให้ client ผ่าน cookie
2. ทุก request ถัดไป browser แนบ cookie นั้นมาอัตโนมัติ (ไม่ต้องเขียนโค้ด client เพิ่มเลย)
3. เซิร์ฟเวอร์อ่าน session ID จาก cookie แล้วค้นหาข้อมูลผู้ใช้จาก store
4. Logout → ลบ session ออกจาก store แล้วสั่ง browser ลบ cookie ทิ้ง
```

Go standard library มี `http.SetCookie` และ `r.Cookie()` ให้ใช้จัดการ cookie โดยตรง ไม่ต้องพึ่ง library ภายนอกเลย:

```go
const sessionCookieName = "session_id"
const sessionTTL = 24 * time.Hour

func loginHandler(w http.ResponseWriter, r *http.Request) {
	username := r.FormValue("username")
	password := r.FormValue("password")

	// ในของจริงต้องตรวจกับฐานข้อมูล + bcrypt เหมือน Part 067
	if username == "" || password != "1234" {
		http.Error(w, "username หรือ password ไม่ถูกต้อง", http.StatusUnauthorized)
		return
	}

	sessionID, err := store.Create(username, sessionTTL)
	if err != nil {
		http.Error(w, "สร้าง session ไม่สำเร็จ", http.StatusInternalServerError)
		return
	}

	http.SetCookie(w, &http.Cookie{
		Name:     sessionCookieName,
		Value:    sessionID,
		Path:     "/",
		Expires:  time.Now().Add(sessionTTL),
		HttpOnly: true,                 // JavaScript อ่าน cookie นี้ไม่ได้ ป้องกัน XSS ขโมย session
		Secure:   true,                 // ส่งผ่าน HTTPS เท่านั้น (ปิดใน localhost dev ได้ถ้าจำเป็น)
		SameSite: http.SameSiteLaxMode, // กัน CSRF ระดับหนึ่งโดยไม่ส่ง cookie ข้าม site แบบ cross-origin POST
	})

	fmt.Fprintf(w, "ล็อกอินสำเร็จ: %s\n", username)
}
```

อธิบาย flag สำคัญของ `http.Cookie`:

- **`HttpOnly: true`** — บอก browser ว่าห้าม JavaScript (`document.cookie`) เข้าถึง cookie นี้ได้ นี่คือแนวป้องกันหลักถ้าเว็บไซต์มีช่องโหว่ XSS: แม้ผู้โจมตีจะแทรก script ลงหน้าเว็บได้สำเร็จ ก็ยังขโมย session cookie ผ่าน JavaScript ไม่ได้
- **`Secure: true`** — cookie จะถูกส่งเฉพาะผ่าน HTTPS เท่านั้น ป้องกันไม่ให้ session ID หลุดไปตอนดักฟัง HTTP ธรรมดา (เหตุผลเดียวกับที่ JWT ต้องใช้ HTTPS ใน Part 067)
- **`SameSite: http.SameSiteLaxMode`** — ควบคุมว่า browser จะส่ง cookie นี้ไปกับ request ที่มาจากเว็บไซต์อื่นหรือไม่ ค่า `Lax` อนุญาตให้ส่งตอนกด link ธรรมดา (navigation) แต่ไม่ส่งตอนมี form POST หรือ request แบบ cross-site อื่นๆ ช่วยลดความเสี่ยง CSRF ได้ในระดับหนึ่งโดยไม่ต้องเขียน CSRF token เพิ่ม (ค่า `Strict` เข้มกว่านี้อีก แต่บางกรณีทำให้ user experience แปลกๆ เช่นกด link จากอีเมลแล้วเหมือนยังไม่ล็อกอิน)

การ logout ก็ทำสองอย่าง: ลบ session ออกจาก store และสั่งให้ browser ลบ cookie ทิ้งด้วยการตั้งเวลาหมดอายุไว้ในอดีต:

```go
func logoutHandler(w http.ResponseWriter, r *http.Request) {
	cookie, err := r.Cookie(sessionCookieName)
	if err == nil {
		store.Delete(cookie.Value)
	}
	http.SetCookie(w, &http.Cookie{
		Name:     sessionCookieName,
		Value:    "",
		Path:     "/",
		Expires:  time.Unix(0, 0), // ตั้งเวลาหมดอายุในอดีต = สั่ง browser ลบ cookie ทิ้ง
		HttpOnly: true,
		Secure:   true,
	})
	fmt.Fprintln(w, "ออกจากระบบแล้ว")
}
```

Middleware สำหรับตรวจสอบ session ก่อนเข้าถึง handler (รูปแบบเดียวกับ auth middleware ของ JWT ใน Part 067):

```go
func requireSession(next http.HandlerFunc) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		cookie, err := r.Cookie(sessionCookieName)
		if err != nil {
			http.Error(w, "กรุณาล็อกอินก่อน", http.StatusUnauthorized)
			return
		}
		sess, ok := store.Get(cookie.Value)
		if !ok {
			http.Error(w, "session หมดอายุหรือไม่ถูกต้อง กรุณาล็อกอินใหม่", http.StatusUnauthorized)
			return
		}
		r = r.WithContext(context.WithValue(r.Context(), usernameKey, sess.Username))
		next(w, r)
	}
}
```

---

## 8. In-Memory Session Store

Session ต้องเก็บไว้ที่ไหนสักแห่งบนเซิร์ฟเวอร์ ตัวอย่างนี้ใช้ map ในหน่วยความจำแบบง่ายที่สุด (เหมาะกับการเรียนรู้และแอปเดียว):

```go
type Session struct {
	Username  string
	CreatedAt time.Time
	ExpiresAt time.Time
}

type MemoryStore struct {
	mu       sync.RWMutex
	sessions map[string]Session
}

func NewMemoryStore() *MemoryStore {
	return &MemoryStore{sessions: map[string]Session{}}
}

func newSessionID() (string, error) {
	b := make([]byte, 32)
	if _, err := rand.Read(b); err != nil {
		return "", err
	}
	return base64.RawURLEncoding.EncodeToString(b), nil
}

func (s *MemoryStore) Create(username string, ttl time.Duration) (string, error) {
	id, err := newSessionID()
	if err != nil {
		return "", err
	}
	now := time.Now()
	s.mu.Lock()
	s.sessions[id] = Session{Username: username, CreatedAt: now, ExpiresAt: now.Add(ttl)}
	s.mu.Unlock()
	return id, nil
}

func (s *MemoryStore) Get(id string) (Session, bool) {
	s.mu.RLock()
	sess, found := s.sessions[id]
	s.mu.RUnlock()
	if !found {
		return Session{}, false
	}
	if time.Now().After(sess.ExpiresAt) {
		s.Delete(id)
		return Session{}, false
	}
	return sess, true
}

func (s *MemoryStore) Delete(id string) {
	s.mu.Lock()
	delete(s.sessions, id)
	s.mu.Unlock()
}
```

จุดสำคัญ: **session ID ต้องสุ่มด้วย `crypto/rand`** (ไม่ใช่ `math/rand`) เพราะต้องเดายากในระดับ cryptographic — ถ้าเดา session ID ของคนอื่นได้ ก็เท่ากับ hijack บัญชีนั้นได้ทันทีโดยไม่ต้องรู้รหัสผ่านเลย

โค้ดชุดนี้ผ่านการทดสอบจริงด้วย `httptest` ครบทุก flow:

```
1. ยังไม่ล็อกอิน เข้า /dashboard        -> 401 Unauthorized
2. ล็อกอินด้วย username/password ถูก   -> 200 พร้อม Set-Cookie: session_id=...
   (ตรวจสอบแล้วว่า cookie มี HttpOnly=true, Secure=true, SameSite=Lax ตามที่ตั้งไว้)
3. ใช้ cookie เรียก /dashboard          -> 200 "แดชบอร์ดของ nattapong"
4. logout แล้วใช้ cookie เดิมเรียกซ้ำ  -> 401 Unauthorized (session ถูกลบไปแล้วจริง)
```

> **ข้อควรระวังสำหรับ production**: `MemoryStore` เก็บข้อมูลไว้ใน RAM ของ process เดียว ถ้าเซิร์ฟเวอร์รีสตาร์ท session ทั้งหมดจะหายไป และถ้ามีเซิร์ฟเวอร์หลาย instance (scale แนวนอน) request ที่ไปตกที่ instance ที่ไม่มี session นั้นจะพบว่าผู้ใช้ "ไม่ได้ล็อกอิน" ทั้งที่จริงล็อกอินอยู่ ทางแก้คือใช้ store กลางที่ทุก instance เข้าถึงร่วมกันได้ เช่น **Redis** ซึ่งจะเรียนเจาะลึกใน **Part 077**

---

## 9. เชื่อม OAuth2 เข้ากับ Session: flow ที่สมบูรณ์

ตอนนี้เรามีทั้งสองชิ้นส่วนแล้ว: OAuth2 login flow (หัวข้อที่ 4) ที่จบด้วยการรู้ email/ชื่อของผู้ใช้จริงจาก Google และ session store (หัวข้อที่ 7-8) ที่จำผู้ใช้ได้ผ่าน cookie มาต่อกันให้ครบวงจร: หลังจากขั้นตอนที่ 7 ของ OAuth2 (ได้ `googleUserInfo` มาแล้ว) แทนที่จะแค่พิมพ์ข้อความตอบกลับเฉยๆ เราควรสร้าง session ให้ผู้ใช้คนนั้นทันที เหมือนกับที่ทำหลัง login ด้วยฟอร์มธรรมดา:

```go
func handleGoogleCallback(w http.ResponseWriter, r *http.Request) {
	// ... ตรวจสอบ state, แลก code เป็น token, ดึง userinfo เหมือนหัวข้อที่ 4 ...

	// map บัญชี Google (info.Email) เข้ากับผู้ใช้ในระบบของเราเอง
	// ถ้ายังไม่เคยมีในระบบ ให้สร้างบัญชีใหม่ให้อัตโนมัติ (เรียกว่า "just-in-time provisioning")
	localUsername := findOrCreateUserByEmail(info.Email, info.Name)

	sessionID, err := store.Create(localUsername, sessionTTL)
	if err != nil {
		http.Error(w, "สร้าง session ไม่สำเร็จ", http.StatusInternalServerError)
		return
	}

	http.SetCookie(w, &http.Cookie{
		Name:     sessionCookieName,
		Value:    sessionID,
		Path:     "/",
		Expires:  time.Now().Add(sessionTTL),
		HttpOnly: true,
		Secure:   true,
		SameSite: http.SameSiteLaxMode,
	})

	http.Redirect(w, r, "/dashboard", http.StatusTemporaryRedirect)
}
```

จุดสำคัญคือ **`findOrCreateUserByEmail`** — ระบบต้องตัดสินใจว่าจะ "จับคู่" บัญชี Google เข้ากับผู้ใช้ในระบบของตัวเองอย่างไร แนวทางทั่วไปคือใช้ email เป็น key เชื่อมโยง (ถ้า email ตรงกับบัญชีที่เคยสมัครด้วยฟอร์มปกติมาก่อน ก็ถือเป็นคนเดียวกัน) และถ้าไม่เคยมีบัญชีมาก่อนเลยก็สร้างบัญชีใหม่ให้อัตโนมัติทันที (แนวคิดนี้เรียกว่า **just-in-time provisioning** — ผู้ใช้ไม่ต้อง "สมัครสมาชิก" แยกต่างหากก่อนเลย กด "Login with Google" ครั้งแรกก็มีบัญชีในระบบทันที)

หลังจากขั้นตอนนี้ **ตัวตนของผู้ใช้ในระบบของเราไม่ได้ผูกกับ OAuth2 อีกต่อไปแล้ว** — request ถัดๆ ไปทั้งหมดตรวจสอบผ่าน session cookie ธรรมดา (หรือจะออกเป็น JWT แทนก็ได้ตาม Part 067) ไม่ต้องติดต่อ Google อีกเลยจนกว่า session จะหมดอายุ

---

## 10. PKCE: การป้องกันเพิ่มเติมสำหรับ Public Client

Flow ในหัวข้อที่ 2 ใช้ได้ดีเมื่อ **client เก็บ `ClientSecret` ไว้อย่างปลอดภัยบนเซิร์ฟเวอร์** (เรียกว่า **confidential client**) แต่ถ้า client เป็น **mobile app หรือ Single Page Application (SPA)** ที่ code ทั้งหมดอยู่บนเครื่องผู้ใช้หรือดาวน์โหลดไปรันในเบราว์เซอร์ (เรียกว่า **public client**) จะไม่มีทางเก็บ `ClientSecret` ให้ปลอดภัยได้เลย — ใครก็ถอด compile หรือเปิด dev tools ดู source แล้วขุดหา secret เจอได้

**PKCE (Proof Key for Code Exchange, อ่านว่า "pixy")** คือส่วนขยายของ OAuth2 ที่แก้ปัญหานี้โดยไม่ต้องใช้ `ClientSecret` เลย หลักการคร่าวๆ:

```
1. Client สุ่มค่าลับ "code_verifier" ขึ้นมาเก็บไว้เอง (ไม่ส่งให้ใคร)
2. Client คำนวณ "code_challenge" = SHA256(code_verifier) แล้วส่งไปกับ AuthCodeURL
3. Authorization Server เก็บ code_challenge ไว้คู่กับ code ที่จะออกให้
4. ตอนแลก code เป็น token (ขั้นตอน Exchange) client ต้องส่ง code_verifier ตัวจริงแนบไปด้วย
5. Authorization Server คำนวณ SHA256(code_verifier) ที่ได้รับ เทียบกับ code_challenge ที่เก็บไว้
   ถ้าไม่ตรงกัน ปฏิเสธทันที
```

ประโยชน์คือ แม้ authorization code จะถูกดักจับระหว่างทาง (เช่นจาก mobile app ที่ redirect URI อาจถูกแอปอื่นในเครื่องเดียวกันดักได้) ผู้ดักจับก็เอา code ไปแลก token ต่อไม่ได้ เพราะไม่มี `code_verifier` ตัวจริงที่ไม่เคยถูกส่งผ่าน network เลย

`golang.org/x/oauth2` รองรับ PKCE ผ่าน `oauth2.S256ChallengeOption`:

```go
import "golang.org/x/oauth2"

verifier := oauth2.GenerateVerifier() // สุ่ม code_verifier ให้อัตโนมัติ

url := cfg.AuthCodeURL(state, oauth2.S256ChallengeOption(verifier))
// ... เก็บ verifier ไว้ (เช่นใน session ชั่วคราวฝั่งเดียวกับที่เก็บ state) ...

token, err := cfg.Exchange(ctx, code, oauth2.VerifierOption(verifier))
```

> **ข้อควรรู้**: ปัจจุบันแนวปฏิบัติที่ดี (best practice) ที่หลายผู้ให้บริการแนะนำคือใช้ PKCE **แม้กับ confidential client ก็ตาม** เพราะเป็นการป้องกันเพิ่มอีกชั้นโดยแทบไม่มีต้นทุนเพิ่มเลย ไม่ใช่ทางเลือกที่ใช้แทนกันได้กับ `ClientSecret` แต่เป็นการป้องกันเสริมที่ใช้ร่วมกันได้

---

## 11. จะเลือก Session หรือ JWT ดี

ไม่มีคำตอบที่ถูกต้องตายตัว ขึ้นกับสถาปัตยกรรมของระบบ:

**เลือก Session เมื่อ:**
- เป็นเว็บแอปแบบดั้งเดิมที่ render HTML ฝั่งเซิร์ฟเวอร์ (server-rendered, ใช้ `html/template` จาก Part 065) ไม่ใช่ SPA/API แยกส่วน
- ต้องการความสามารถ **บังคับ logout ผู้ใช้ทันที** (เช่นแอดมินสั่งระงับบัญชี ต้องมีผลทันทีไม่ต้องรอ token หมดอายุ)
- ระบบมีขนาดไม่ใหญ่มาก มี backend เดียวหรือใช้ store กลาง (Redis) อยู่แล้ว

**เลือก JWT เมื่อ:**
- Backend เป็น REST/GraphQL API ที่ frontend (SPA, mobile app) แยกส่วนชัดเจนออกจากกัน
- ระบบเป็น microservices ที่หลาย service ต้องตรวจสอบตัวตนผู้ใช้โดยไม่อยากให้ทุก service ต้อง query session store กลางตลอดเวลา
- ต้องการ scale แนวนอนโดยไม่อยากพึ่ง shared state ระหว่างเซิร์ฟเวอร์

ในทางปฏิบัติ ระบบจำนวนมากใช้ทั้งสองแบบผสมกัน เช่น ใช้ session cookie สำหรับหน้าเว็บที่ผู้ใช้ทั่วไปเข้าใช้งาน และใช้ JWT สำหรับ public API ที่เปิดให้ third-party หรือ mobile app เรียกใช้

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- OAuth2 คือมาตรฐานสำหรับมอบสิทธิ์ให้แอปเข้าถึงข้อมูลผู้ใช้บนระบบอื่น โดยไม่ต้องเปิดเผยรหัสผ่านให้แอปนั้นรู้
- **Authorization Code Flow** มี 8 ขั้นตอนหลัก: redirect ไป authorization server → ผู้ใช้ล็อกอิน+ยินยอม → redirect กลับพร้อม code → แลก code เป็น access token → ใช้ token ดึงข้อมูลผู้ใช้ → สร้างตัวตนในระบบของเราเอง
- `golang.org/x/oauth2` และ `golang.org/x/oauth2/google` ใช้ implement "Login with Google" ได้ โดยโค้ดต้อง compile และมีโครงสร้างถูกต้องตาม API แม้จะทดสอบ end-to-end ไม่ได้โดยไม่มี credential จริงและ browser
- **`state` parameter สำคัญมาก** ใช้ป้องกัน CSRF ในขั้นตอน callback ของ OAuth2
- Session-based auth เก็บ state ไว้ที่เซิร์ฟเวอร์ (stateful) ต่างจาก JWT ที่เป็น stateless — แลกกับความสามารถเพิกถอนได้ทันที
- Cookie-based session ใช้ `http.SetCookie`/`r.Cookie()` พร้อม flag ความปลอดภัยสำคัญ: `HttpOnly` (กัน XSS อ่าน cookie), `Secure` (บังคับ HTTPS), `SameSite` (ลดความเสี่ยง CSRF)
- Session ID ต้องสุ่มด้วย `crypto/rand` เสมอ ไม่ใช่ `math/rand`
- หลังจากได้ข้อมูลผู้ใช้จาก OAuth2 แล้ว ควร map เข้ากับบัญชีในระบบของเราเองด้วย email (just-in-time provisioning) แล้วสร้าง session/JWT ของแอปเราเองต่อทันที ไม่ต้องพึ่ง Google อีกต่อไปจนกว่าจะหมดอายุ
- **PKCE** ป้องกันการดักจับ authorization code สำหรับ public client (mobile app, SPA) ที่เก็บ `ClientSecret` ให้ปลอดภัยไม่ได้ และปัจจุบันแนะนำให้ใช้แม้กับ confidential client เพื่อความปลอดภัยเพิ่มอีกชั้น
- เลือก session เมื่อเป็นเว็บแอป server-rendered ที่ต้องการเพิกถอนสิทธิ์ได้ทันที เลือก JWT เมื่อเป็น API/microservices ที่ต้องการ stateless และ scale ง่าย

## แบบฝึกหัดท้ายบท

1. สมัคร Google Cloud Console สร้าง OAuth Client ID จริง แล้วทดลองรันโค้ดในหัวข้อที่ 4 ให้ครบ flow จนได้ข้อมูล email จริงของตัวเองกลับมา
2. เพิ่ม endpoint `/auth/github/login` และ `/auth/github/callback` โดยใช้ `golang.org/x/oauth2/github` ตามรูปแบบเดียวกับ Google
3. เขียน unit test สำหรับฟังก์ชัน `randomState()` และ logic ตรวจสอบ `state` ใน `handleGoogleCallback` (ส่วนที่ทดสอบได้โดยไม่ต้องพึ่ง Google จริง)
4. แก้ `MemoryStore` ให้มีฟังก์ชัน cleanup ที่ลบ session ที่หมดอายุแล้วออกเป็นระยะ (ใช้ `time.Ticker` รันเป็น goroutine พื้นหลัง ตามแนวคิดจาก Part 036-037)
5. ทดลองเอา flag `HttpOnly` ออกจาก cookie แล้วเขียนโค้ด JavaScript ตัวอย่าง (สมมติสถานการณ์) ที่แสดงให้เห็นว่า `document.cookie` จะอ่าน session ID ได้ทันทีถ้าไม่ได้ตั้ง flag นี้ไว้
6. เขียนฟังก์ชัน `findOrCreateUserByEmail` แบบง่ายๆ (ใช้ map ในหน่วยความจำแทนฐานข้อมูล) ที่ใช้ในหัวข้อที่ 9 แล้วเขียนทดสอบว่า login ซ้ำด้วย email เดิมสองครั้งได้ user คนเดียวกัน ไม่สร้างบัญชีซ้ำ
7. ค้นคว้าเพิ่มเติมว่า `oauth2.GenerateVerifier()` กับ `oauth2.S256ChallengeOption()` ในหัวข้อที่ 10 ทำงานภายในอย่างไร (ดู source code ของ `golang.org/x/oauth2`) แล้วอธิบายด้วยคำพูดของตัวเองว่าทำไม SHA256 ถึงเพียงพอสำหรับป้องกันการปลอมแปลง `code_verifier`
8. ออกแบบและอธิบายเป็นข้อความ (ไม่ต้องเขียนโค้ด) ว่าถ้าต้องทำระบบที่มีทั้งเว็บแอปสำหรับผู้ใช้ทั่วไปและ public API สำหรับนักพัฒนาภายนอก ท่านจะออกแบบให้ใช้ session, JWT หรือทั้งสองอย่างผสมกันอย่างไร เพราะเหตุใด

---

**ต่อไป**: [Part 069 — WebSocket ด้วย Go](./069-websocket.md)
