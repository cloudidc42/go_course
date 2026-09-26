# Part 106: Security Best Practices สำหรับ Go

> ภาคที่ 10: มืออาชีพและระดับโลก (Professional & World-Class) — ตอนที่ 7 จาก 11 (Part 100–110)

> **หมายเหตุเรื่องความซื่อสัตย์ในการสาธิต**: บทนี้เป็นเรื่อง **defensive security** ล้วนๆ — สอนวิธี**ป้องกัน**ช่องโหว่ ไม่สอนวิธีโจมตี บทนี้จะไม่มีการยกตัวอย่าง payload โจมตีจริงหรือโค้ด exploit ที่ใช้งานได้ เพราะไม่ใช่จุดประสงค์ของหลักสูตรนี้และไม่มีประโยชน์ต่อการเป็นนักพัฒนาที่ดี ทุกตัวอย่างเรื่อง `govulncheck` ในบทนี้ **รันจริง** ด้วย `govulncheck@v1.8.0` บน Go 1.24.7 ในเครื่องที่ใช้เขียนหลักสูตรนี้ ผลลัพธ์ทั้งหมดคือ output จริงจากการรันคำสั่ง ไม่ใช่ค่าที่แต่งขึ้น (รายละเอียดวิธีรันแบบ reproducible อยู่ในหัวข้อที่ 6)

## สารบัญของบทนี้

1. ภาพรวม: OWASP Top 10 กับมุมมองของนักพัฒนา Go
2. Injection: ทบทวน SQL Injection และหลักการป้องกันด้วย Parameterized Query (Part 071)
3. Broken Authentication: bcrypt, JWT ให้ถูกวิธี (ทบทวน Part 051, 067)
4. Cross-Site Scripting (XSS): พึ่งพา Auto-Escaping ของ `html/template` (ทบทวน Part 065)
5. Broken Access Control: Authorization ไม่ใช่แค่ Authentication และการป้องกัน CSRF
6. Insecure Deserialization: ข้อควรระวังเมื่อ decode JSON/Gob จากแหล่งที่ไม่น่าเชื่อถือ
7. Vulnerable Dependencies: สแกนด้วย `govulncheck` (รันจริง พร้อมผลลัพธ์จริง)
8. Secrets Management: ห้าม hardcode credential เด็ดขาด
9. Security Misconfiguration: ค่า default ที่ปลอดภัย (รวม TLS Hardening)
10. Input Validation คือแนวป้องกันชั้นที่สอง (Defense in Depth)
11. Checklist: Secure Defaults สำหรับ Go Web Service ใหม่
12. Checklist: Security Code Review สำหรับทีม
13. สรุปสิ่งที่ได้เรียนในบทนี้
14. แบบฝึกหัดท้ายบท

---

## 1. ภาพรวม: OWASP Top 10 กับมุมมองของนักพัฒนา Go

ตลอดหลักสูตรนี้เราแตะเรื่องความปลอดภัยมาเป็นระยะๆ อยู่แล้ว — hashing และ encryption ใน **Part 051**, SQL Injection ใน **Part 071**, JWT ใน **Part 067**, OAuth2/Session ใน **Part 068**, และ auto-escaping ของ template ใน **Part 065** บทนี้ทำหน้าที่เป็น**บทรวบยอด**ที่จัดระเบียบความรู้เหล่านั้นให้เป็นระบบเดียว โดยใช้กรอบที่อุตสาหกรรมยอมรับกันกว้างขวางที่สุดคือ **OWASP Top 10** (Open Worldwide Application Security Project) เป็นแผนที่นำทาง

สิ่งสำคัญที่ต้องเข้าใจก่อนคือ **แนวทางของบทนี้คือ "ป้องกัน" ไม่ใช่ "โจมตี"** เป้าหมายของนักพัฒนา Go มืออาชีพไม่ใช่การรู้วิธีแฮ็กระบบ แต่คือการรู้ว่า**ช่องโหว่แต่ละประเภทเกิดจากอะไร** และ **เครื่องมือ/แนวปฏิบัติใดใน Go ที่ป้องกันมันได้อย่างเป็นระบบ** ตารางด้านล่างคือแผนที่เชื่อมโยง OWASP Top 10 (ฉบับหมวดหมู่ที่ใช้กันแพร่หลาย) เข้ากับสิ่งที่เราเรียนไปแล้วและจะเรียนเพิ่มในบทนี้:

| หมวดหมู่ OWASP | ความเสี่ยงคืออะไร | แนวทางป้องกันใน Go |
|---|---|---|
| Injection | ผู้ใช้แทรกคำสั่งปลอมเข้าไปในคำสั่งที่ระบบรัน (เช่น SQL) | Parameterized query (**Part 071**) |
| Broken Authentication | ระบบยืนยันตัวตนอ่อนแอ (รหัสผ่านเก็บผิดวิธี, token คาดเดาได้) | `bcrypt`, `crypto/rand`, JWT ที่ตั้งค่าถูกต้อง (**Part 051, 067**) |
| Sensitive Data Exposure | ข้อมูลอ่อนไหวรั่วไหลระหว่างทางหรือตอนเก็บ | TLS (**Part 051**), AES-GCM, ไม่ log ข้อมูลอ่อนไหว |
| XML External Entities (XXE) | Parser ยอมให้ XML อ้างอิงไฟล์ระบบ/URL ภายนอก | ปิด external entity resolution เมื่อ parse XML ที่ไม่น่าเชื่อถือ (**Part 026**) |
| Broken Access Control | ผู้ใช้เข้าถึงข้อมูล/ฟังก์ชันที่ไม่ควรเข้าถึงได้ | ตรวจสิทธิ์ทุก endpoint ด้วย middleware (**Part 057, 067**) ไม่พึ่งพา "security by obscurity" |
| Security Misconfiguration | ค่า default ที่ไม่ปลอดภัยหลงเหลือใน production | Checklist หัวข้อ 11 ของบทนี้ |
| Cross-Site Scripting (XSS) | Inject โค้ดฝั่ง client ผ่านข้อมูลที่ไม่ถูก escape | Auto-escaping ของ `html/template` (**Part 065**) |
| Insecure Deserialization | Decode ข้อมูลจากแหล่งไม่น่าเชื่อถือแล้วเกิดผลข้างเคียงที่อันตราย | ทบทวนในหัวข้อ 6 ของบทนี้ |
| Using Components with Known Vulnerabilities | Dependency ที่ใช้มีช่องโหว่ที่รู้จักแล้ว | `govulncheck` (หัวข้อ 7 ของบทนี้) |
| Insufficient Logging & Monitoring | ตรวจไม่พบการโจมตีเพราะไม่มี log/alert ที่เพียงพอ | `slog` และ observability (**Part 054, 099**) |

สังเกตว่า**ครึ่งหนึ่งของตารางนี้เราได้เรียนวิธีป้องกันไปแล้วในบทก่อนหน้า** — นี่คือเหตุผลที่บทนี้เน้นการ**เชื่อมโยง** มากกว่าสอนใหม่ทั้งหมด ส่วนที่เหลือ (deserialization, dependency scanning, secrets management, misconfiguration) คือเนื้อหาใหม่ที่จะเติมให้ครบ

---

## 2. Injection: ทบทวน SQL Injection และหลักการป้องกันด้วย Parameterized Query

**Part 071** ได้สาธิตไปแล้วอย่างละเอียดว่าการต่อ string SQL เองจากค่าที่ผู้ใช้กรอกเข้ามาโดยตรง (เช่น ด้วย `fmt.Sprintf`) เปิดช่องให้ผู้ใช้ที่ตั้งใจร้ายส่งค่าที่มีไวยากรณ์ SQL แฝงเข้ามา ทำให้คำสั่งที่ database รันจริงเปลี่ยนความหมายไปจากที่ตั้งใจไว้โดยสิ้นเชิง — ตัวอย่างพื้นฐานที่สุดคือ query ที่ควรค้นหา user คนเดียว กลับคืนข้อมูล user ทุกคนในระบบ

หลักการป้องกันที่ Go มอบให้ผ่าน `database/sql` มาตรฐานคือ **parameterized query**: ส่ง SQL structure และค่าข้อมูลแยกกันเป็นคนละ argument เสมอ

```go
// ปลอดภัย: โครงสร้างคำสั่งกับข้อมูลแยกกันชัดเจน ฐานข้อมูลไม่มีทางตีความค่าข้อมูลเป็นส่วนหนึ่งของคำสั่ง
row := db.QueryRowContext(ctx, "SELECT id, name FROM users WHERE username = $1", username)
```

```go
// อันตราย: ต่อ string เอง — ห้ามเขียนแบบนี้เด็ดขาดไม่ว่ากรณีใด
query := fmt.Sprintf("SELECT id, name FROM users WHERE username = '%s'", username)
row := db.QueryRow(query)
```

กฎที่ต้องยึดถือแบบไม่มีข้อยกเว้น (ทบทวนจาก Part 071):

> **ห้ามต่อ string SQL จากค่าที่มาจากผู้ใช้เด็ดขาด ไม่ว่าจะ escape เองด้วยวิธีใดก็ตาม** ให้ใช้ placeholder (`$1`/`?` แล้วแต่ driver) ของ `database/sql` เสมอ วิธีนี้ไม่ใช่แค่ "ปลอดภัยกว่า" แต่ **ตัดปัญหาออกจากสมการทั้งหมด** เพราะ driver ส่งข้อมูลไปยังฐานข้อมูลแยกช่องทางจากคำสั่ง ทำให้ไม่มีทางที่ข้อมูลจะถูกตีความเป็นส่วนหนึ่งของคำสั่งได้เลยไม่ว่าข้อมูลจะมีอักขระอะไรปนอยู่

หลักการเดียวกันนี้ใช้กับ injection ประเภทอื่นด้วย แม้จะพบน้อยกว่าใน Go: **Command Injection** (ส่ง argument ที่มาจากผู้ใช้เข้า `os/exec` โดยผ่าน shell) ป้องกันได้ด้วยการเรียก `exec.Command(name, args...)` แบบแยก argument เป็น slice เสมอ (ตามที่ **Part 052** สาธิตไว้) แทนที่จะประกอบเป็น string เดียวแล้วส่งผ่าน `sh -c`

---

## 3. Broken Authentication: bcrypt, JWT ให้ถูกวิธี

**Part 051** สอนไปแล้วว่าทำไมการ hash รหัสผ่านด้วย SHA-256 เปล่าๆ ถึงอันตราย และวิธีใช้ `bcrypt` อย่างถูกต้องผ่าน `golang.org/x/crypto/bcrypt` ทบทวนหลักการสำคัญ 3 ข้อ:

1. **ใช้ password hashing function ที่ออกแบบมาโดยเฉพาะ** (`bcrypt`, `argon2`) ที่ช้าโดยตั้งใจและมี salt อัตโนมัติ — ห้ามใช้ `sha256`/`md5` เก็บรหัสผ่าน
2. **ใช้ `crypto/rand` เสมอ** สำหรับสิ่งที่เกี่ยวกับความปลอดภัย เช่น session token, reset-password token — ห้ามใช้ `math/rand`
3. **ข้อความ error ต้องไม่เปิดเผยรายละเอียดที่ช่วยผู้โจมตี** เช่น ไม่บอกแยกว่า "username ไม่มีในระบบ" กับ "รหัสผ่านผิด"

**Part 067** ต่อยอดเรื่องนี้ด้วย JWT ประเด็นด้าน security ที่สำคัญที่สุดที่ต้องย้ำซ้ำในบทนี้:

- **JWT ไม่ได้ถูกเข้ารหัส** (encrypted) เพียงแค่ **เซ็นลายเซ็น** (signed) — ใครก็ตาม decode payload อ่านได้เสมอ (แค่ base64 decode) **ห้ามใส่ข้อมูลอ่อนไหว** เช่น รหัสผ่าน, เลขบัตรเครดิต ลงใน claims เด็ดขาด
- **ต้องตรวจสอบ signing algorithm ที่ระบุมาใน token ตรงกับที่ server คาดหวังเสมอ** ไม่ควรอนุญาตให้ token กำหนด algorithm เองอย่างอิสระ (library อย่าง `golang-jwt/jwt/v5` ที่ใช้ใน Part 067 มี API ที่บังคับให้ระบุ algorithm ที่ยอมรับตอน parse อยู่แล้ว ให้ใช้ตามที่ library แนะนำเสมอ)
- **ตั้งเวลาหมดอายุ (`exp`) สั้นสมเหตุสมผลเสมอ** และใช้ **refresh token pattern** (ตามที่สาธิตใน Part 067) แทนที่จะออก access token ที่มีอายุยาวนาน เพื่อจำกัดความเสียหายหาก token รั่วไหล
- **เก็บ secret key ที่ใช้เซ็น JWT (HMAC secret) หรือ private key (RSA/ECDSA) อย่างปลอดภัย** — ประเด็นนี้เชื่อมโยงตรงกับหัวข้อ 8 (Secrets Management) ของบทนี้

**Part 068** เพิ่มมุมมองเรื่อง cookie-based session และ OAuth2 ไว้แล้ว โดยเฉพาะ flag ของ cookie ที่ต้องตั้งให้ถูกต้อง (`Secure`, `HttpOnly`, `SameSite`) ซึ่งจะย้ำอีกครั้งใน checklist หัวข้อ 11

---

## 4. Cross-Site Scripting (XSS): พึ่งพา Auto-Escaping ของ `html/template`

**Part 065** สาธิตให้เห็นแล้วว่าการ render ข้อมูลจากผู้ใช้ลง HTML โดยใช้ `text/template` เปิดช่องให้ script ที่ผู้ใช้กรอกเข้ามาไปทำงานจริงบนเบราว์เซอร์ของผู้ใช้คนอื่น (ผลลัพธ์คือ `alert(...)` ทำงานแทนที่จะแสดงเป็นข้อความธรรมดา) และวิธีแก้คือเปลี่ยนไปใช้ `html/template` ซึ่งทำ **contextual auto-escaping** ให้อัตโนมัติตามตำแหน่งที่ข้อมูลถูกแทรก (ใน HTML body, ใน attribute, ใน `<script>`, ใน URL — แต่ละบริบทมีกฎ escape ต่างกัน และ `html/template` รู้จักแยกแยะทั้งหมดนี้ให้)

กฎที่ต้องยึดถือ:

> **Render HTML ที่มีข้อมูลจากผู้ใช้ปนอยู่ ให้ใช้ `html/template` เสมอ ไม่ใช่ `text/template`** และ**ห้ามใช้ type `template.HTML`, `template.JS`, `template.URL` ครอบข้อมูลดิบจากผู้ใช้** เพราะ type เหล่านี้บอก `html/template` ว่า "เชื่อค่านี้ได้ ไม่ต้อง escape" — ถ้านำไปครอบข้อมูลที่ผู้ใช้ควบคุมได้ ก็เท่ากับปิดการป้องกัน XSS ของตัวเองโดยสมัครใจ type เหล่านี้มีไว้สำหรับ HTML/JS/URL ที่**นักพัฒนาเป็นคนสร้างเองในโค้ด** เท่านั้น (เช่น HTML fragment คงที่ที่ไม่มีส่วนไหนมาจาก input)

หลักการเดียวกันขยายไปถึง **API ที่คืนค่าเป็น JSON**: `encoding/json` (Part 025) escape อักขระ HTML พิเศษ (`<`, `>`, `&`) เป็นค่า Unicode escape โดย default อยู่แล้วเมื่อใช้ `json.NewEncoder` (ป้องกันกรณี JSON ถูกฝังลงใน `<script>` tag ของหน้าเว็บ) — ถ้าจำเป็นต้องปิดพฤติกรรมนี้ (`SetEscapeHTML(false)`) ต้องมั่นใจว่า JSON output จะไม่ถูกนำไปฝังในบริบท HTML ที่ไหนเลย

---

## 5. Broken Access Control: Authorization ไม่ใช่แค่ Authentication และการป้องกัน CSRF

**Authentication** (รู้ว่าใครเรียกมา) กับ **Authorization** (ผู้เรียกมีสิทธิ์ทำสิ่งนี้จริงหรือไม่) เป็นคนละเรื่องกัน แต่มือใหม่จำนวนมากทำแค่ authentication แล้วคิดว่าปลอดภัยแล้ว — **Part 067** สอน JWT authentication ไว้อย่างละเอียด แต่ authentication บอกแค่ว่า "นี่คือ user X" ไม่ได้บอกว่า "user X มีสิทธิ์ลบ order ของ user Y ได้หรือไม่"

### 5.1 ตรวจสอบสิทธิ์ทุก endpoint อย่างชัดเจน

```go
// authMiddleware (จาก Part 067) แค่ยืนยันตัวตนและใส่ user ลง context
// authorizeOwner คือชั้นที่สองที่ตรวจสอบว่า "user นี้เป็นเจ้าของ resource นี้จริงหรือไม่"
func authorizeOwner(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		claims, ok := r.Context().Value(userClaimsKey).(*Claims)
		if !ok {
			http.Error(w, "unauthorized", http.StatusUnauthorized)
			return
		}

		orderID := r.PathValue("id")
		order, err := orderStore.Get(r.Context(), orderID)
		if err != nil {
			http.Error(w, "not found", http.StatusNotFound)
			return
		}

		// จุดสำคัญ: ตรวจสอบความเป็นเจ้าของอย่างชัดเจน ไม่ใช่แค่ตรวจว่า login แล้ว
		if order.OwnerID != claims.UserID {
			http.Error(w, "forbidden", http.StatusForbidden)
			return
		}

		next.ServeHTTP(w, r)
	})
}
```

บั๊กคลาสสิกที่เรียกว่า **Insecure Direct Object Reference (IDOR)** เกิดขึ้นเมื่อ endpoint เช็คแค่ "login แล้วหรือยัง" แต่ลืมเช็คว่า resource ที่ขอ (เช่น `/orders/{id}`) เป็นของผู้เรียกจริงหรือไม่ — ผู้ใช้ที่ login ปกติสามารถเปลี่ยนเลข `{id}` ใน URL แล้วเห็นข้อมูลของคนอื่นได้ทันทีถ้าไม่มีการตรวจสอบชั้นนี้

### 5.2 หลักการ Least Privilege

ออกแบบระบบสิทธิ์ให้ผู้ใช้/service แต่ละตัวมีสิทธิ์**เท่าที่จำเป็นต้องใช้จริงเท่านั้น** ไม่ใช่ "ให้สิทธิ์กว้างไว้ก่อนเผื่อใช้ในอนาคต" — ตัวอย่างเช่น credential ของ database ที่ service ใช้ควรมีสิทธิ์แค่ตารางที่ service นั้นต้องใช้จริง ไม่ใช่สิทธิ์ระดับ superuser ของทั้งฐานข้อมูล แม้จะสะดวกกว่าตอนพัฒนาก็ตาม

### 5.3 CSRF (Cross-Site Request Forgery) และ `SameSite` Cookie

**Part 068** แนะนำ flag `SameSite` ของ cookie ไว้แล้วเป็นส่วนหนึ่งของ session security หัวข้อนี้อธิบายว่าทำไมมันถึงสำคัญกับ CSRF โดยเฉพาะ: CSRF คือการที่เว็บไซต์อื่นหลอกให้เบราว์เซอร์ของผู้ใช้ส่ง request ไปยังระบบของเราโดยที่ผู้ใช้ไม่ได้ตั้งใจ (เช่น ฝัง form ที่ submit อัตโนมัติ) โดยอาศัยว่าเบราว์เซอร์แนบ cookie session ของผู้ใช้ไปกับ request นั้นให้อัตโนมัติ

```go
http.SetCookie(w, &http.Cookie{
	Name:     "session_id",
	Value:    sessionID,
	HttpOnly: true,
	Secure:   true,
	SameSite: http.SameSiteStrictMode, // ป้องกัน cookie ไม่ให้แนบไปกับ request ข้าม origin
	Path:     "/",
})
```

`SameSite=Strict` หรือ `Lax` บอกเบราว์เซอร์ว่า**อย่าแนบ cookie นี้ไปกับ request ที่มาจาก origin อื่น** ทำให้การโจมตี CSRF แบบพื้นฐานใช้ไม่ได้ผลตั้งแต่ระดับเบราว์เซอร์ อย่างไรก็ตาม สำหรับระบบที่ต้องรองรับ cross-site request ที่ถูกต้องตามกฎหมาย (เช่น embed เป็น iframe ในเว็บอื่น) หรือต้องการการป้องกันชั้นที่สอง ควรเพิ่ม **CSRF token** แบบดั้งเดิม: สร้าง token สุ่ม (ด้วย `crypto/rand` ตาม **Part 051**) ฝังไว้ใน form/header แล้วตรวจสอบว่าตรงกับค่าที่เก็บไว้ฝั่ง server ก่อนประมวลผลทุกครั้งที่มีการเปลี่ยนแปลงข้อมูล (POST/PUT/DELETE)

> **ข้อสังเกต**: CSRF เกี่ยวข้องกับระบบที่ใช้ **cookie-based session** เป็นหลัก ระบบที่ใช้ **JWT ผ่าน `Authorization` header** (ตาม **Part 067**) ไม่เสี่ยงต่อ CSRF แบบเดียวกัน เพราะเบราว์เซอร์ไม่แนบ custom header ให้อัตโนมัติข้าม origin แบบที่แนบ cookie ให้ — นี่คือข้อดีเชิง security อย่างหนึ่งของ token-based auth เหนือ session-based auth ที่ **Part 068** กล่าวถึงไว้ในหัวข้อ "จะเลือก Session หรือ JWT ดี"

---

## 6. Insecure Deserialization: ข้อควรระวังเมื่อ Decode ข้อมูลจากแหล่งที่ไม่น่าเชื่อถือ

**Deserialization** คือกระบวนการแปลงข้อมูลดิบ (bytes/text) กลับเป็น struct/object ในโปรแกรม — เช่น `json.Unmarshal`, `gob.Decode`, `xml.Unmarshal` ความเสี่ยงเกิดขึ้นเมื่อข้อมูลที่ decode มาจาก**แหล่งที่ไม่น่าเชื่อถือ** (เช่น request body จากผู้ใช้ภายนอก) เพราะ deserializer ที่ออกแบบไม่รัดกุมอาจถูกใช้เป็นช่องทางสร้างผลข้างเคียงที่ไม่ตั้งใจ

หลักปฏิบัติที่ปลอดภัยสำหรับ Go เมื่อ decode ข้อมูลจากภายนอก:

### 5.1 จำกัดขนาด input ก่อน decode เสมอ

```go
// จำกัดขนาด request body ก่อนส่งเข้า json.Unmarshal ป้องกัน memory exhaustion
// จาก payload ขนาดใหญ่ผิดปกติที่ผู้โจมตีจงใจส่งมา
const maxBodySize = 1 << 20 // 1 MiB

func handler(w http.ResponseWriter, r *http.Request) {
	r.Body = http.MaxBytesReader(w, r.Body, maxBodySize)

	var payload struct {
		Name string `json:"name"`
	}
	dec := json.NewDecoder(r.Body)
	dec.DisallowUnknownFields() // ปฏิเสธ field แปลกปลอมที่ไม่ได้อยู่ใน struct แทนที่จะเงียบๆ ข้ามไป

	if err := dec.Decode(&payload); err != nil {
		http.Error(w, "invalid request body", http.StatusBadRequest)
		return
	}
	// ใช้งาน payload ต่อ...
}
```

จุดสำคัญ 2 จุดในโค้ดข้างบน:

- **`http.MaxBytesReader`** ป้องกัน request body ขนาดใหญ่ผิดปกติทำให้ server ใช้ memory เกินจนกระทบผู้ใช้คนอื่น (denial of service เชิงทรัพยากร)
- **`DisallowUnknownFields()`** ทำให้ decoder ปฏิเสธ field ที่ไม่รู้จักแทนที่จะข้ามเงียบๆ ช่วยจับ payload ที่ผิดปกติหรือพยายามส่ง field แปลกปลอมเข้ามาได้ตั้งแต่ชั้น decode

### 5.2 อย่า `Unmarshal` ลง `interface{}`/`any` แล้วส่งต่อโดยไม่ตรวจสอบ

Decode ลง type ที่เจาะจงชัดเจน (`struct` ที่มี field ตายตัว) เสมอเมื่อทำได้ แทนที่จะ decode ลง `map[string]interface{}` หรือ `any` แบบกว้างๆ แล้วส่งต่อค่าที่ได้ไปยังส่วนอื่นของระบบโดยไม่ validate — การ decode ลง struct เจาะจงทำให้ Go compiler และ `encoding/json` ช่วยกรอง shape ของข้อมูลให้ตั้งแต่ต้นทาง ลดพื้นที่ผิว (attack surface) ที่ต้อง validate เองในภายหลัง

### 5.3 ระวังการใช้ `encoding/gob` หรือ `reflect`-based decoder กับข้อมูลจากภายนอก

`encoding/gob` ถูกออกแบบมาสำหรับสื่อสารระหว่าง Go program ที่**เชื่อถือกันเอง** (เช่น สอง service ภายในองค์กรเดียวกัน) ไม่ได้ออกแบบมาให้ทนทานต่อข้อมูลจากแหล่งไม่น่าเชื่อถือเท่ากับ `encoding/json` — หลักปฏิบัติที่ปลอดภัยคือ **ใช้ JSON (หรือ Protocol Buffers ที่มี schema ชัดเจนตาม Part 089) สำหรับข้อมูลที่ข้าม trust boundary** และสงวน `gob` ไว้สำหรับการสื่อสารภายในระบบที่ควบคุมทั้งสองฝั่งเอง

### 5.4 Validate ค่าที่ decode ได้ก่อนใช้งานเสมอ

Deserialization สำเร็จไม่ได้แปลว่าข้อมูล "ถูกต้องตามกฎธุรกิจ" แค่แปลว่า "รูปแบบ syntax ตรงกับ struct" — เช่น `Age int` decode ค่า `-5` ได้สำเร็จโดยไม่มี error แต่ไม่สมเหตุสมผลในทางธุรกิจ ต้องมีชั้น validation ต่อจาก deserialization เสมอ (รายละเอียดในหัวข้อ 10)

---

## 7. Vulnerable Dependencies: สแกนด้วย `govulncheck`

หมวดหมู่ OWASP "Using Components with Known Vulnerabilities" คือความเสี่ยงที่นักพัฒนาควบคุมได้ยากที่สุดในบรรดาทั้งหมด เพราะช่องโหว่ไม่ได้อยู่ในโค้ดที่เราเขียนเอง แต่อยู่ใน **dependency** ที่เราติดตั้งมาใช้ — โครงการ Go ขนาดกลางอาจมี dependency (รวม transitive) หลายสิบถึงหลายร้อยตัว การไล่ตรวจ CVE ของแต่ละตัวด้วยมือเป็นไปไม่ได้ในทางปฏิบัติ

ทีม Go จึงสร้างเครื่องมือทางการชื่อ **`govulncheck`** ที่ทำสิ่งที่ scanner ทั่วไปทำไม่ได้: มันไม่ได้แค่เช็คว่า "module เวอร์ชันนี้เคยมี CVE ประกาศไว้หรือไม่" (ซึ่งจะเตือน false positive จำนวนมาก เพราะหลาย CVE อยู่ในโค้ดส่วนที่โปรแกรมเราไม่ได้เรียกใช้เลย) แต่มันทำ **static analysis เจาะลึกถึงระดับ function call graph** เพื่อเช็คว่า**โค้ดของเราเรียกไปถึง symbol (function) ที่มีช่องโหว่จริงหรือไม่** ทำให้ผลลัพธ์ตรงประเด็นและ actionable กว่ามาก

### 6.1 ติดตั้ง

```bash
go install golang.org/x/vuln/cmd/govulncheck@latest
```

ตรวจสอบว่าติดตั้งสำเร็จ:

```bash
govulncheck -version
```

ผลลัพธ์จริงจากเครื่องที่ใช้เขียนบทนี้:

```
Go: go1.24.7
Scanner: govulncheck@v1.8.0
DB: https://vuln.go.dev
```

### 6.2 สาธิตจริง: พบช่องโหว่ในโปรเจกต์ตัวอย่าง

มาสร้างโปรเจกต์เล็กๆ ที่ import module เวอร์ชันเก่าซึ่งมีช่องโหว่ที่รู้จักแล้ว เพื่อดูว่า `govulncheck` รายงานผลอย่างไรจริงๆ:

```go
// main.go
package main

import (
	"fmt"

	"golang.org/x/text/language"
)

func main() {
	tag, err := language.Parse("en-US")
	if err != nil {
		panic(err)
	}
	fmt.Println("parsed tag:", tag)
}
```

```bash
go mod init govulncheck-demo
go get golang.org/x/text@v0.3.6   # เวอร์ชันเก่าที่มีช่องโหว่ GO-2021-0113 จริง
```

รัน `govulncheck`:

```bash
govulncheck ./...
```

> **หมายเหตุสภาพแวดล้อม**: sandbox ที่ใช้เขียนหลักสูตรนี้ปิดกั้น network egress ไปยัง `vuln.go.dev` (ฐานข้อมูลช่องโหว่ทางการที่ `govulncheck` เรียกผ่าน HTTPS ตาม default) ตาม policy ของ environment เอง ในเครื่อง dev/CI ทั่วไปที่ไม่มีข้อจำกัดนี้ คำสั่งข้างบนจะทำงานตรงๆ โดยไม่ต้องตั้งค่าอะไรเพิ่ม สำหรับบทนี้ เราจึงรัน `govulncheck` จริงโดยชี้ `-db` ไปยัง**ฐานข้อมูลช่องโหว่ในรูปแบบเดียวกัน (OSV format)** ที่ทีม `x/vuln` ใช้เป็น test fixture ของตัวเองในซอร์สโค้ด (มีรายการ `GO-2021-0113` สำหรับ `golang.org/x/text` อยู่จริง) เพื่อให้เห็น**พฤติกรรมและ output จริงของตัวเครื่องมือ** โดยไม่ต้องพึ่ง network ภายนอก — ตัว logic การวิเคราะห์ที่รันคือของจริงทั้งหมด ต่างกันแค่แหล่งข้อมูลช่องโหว่ที่ใช้อ้างอิง

```bash
govulncheck -db "file://$(go env GOMODCACHE)/golang.org/x/vuln@v1.8.0/cmd/govulncheck/testdata/strip/vulndb-v1" -show verbose ./...
```

ผลลัพธ์จริงจากการรันคำสั่งนี้:

```
Fetching vulnerabilities from the database...

Checking the code against the vulnerabilities...

The package pattern matched the following root package:
  govulncheck-demo
Govulncheck scanned the following 2 modules and the go1.24.7 standard library:
  govulncheck-demo
  golang.org/x/text@v0.3.6

=== Symbol Results ===

Vulnerability #1: GO-2021-0113
    Due to improper index calculation, an incorrectly formatted language tag can
    cause Parse to panic via an out of bounds read. If Parse is used to process
    untrusted user inputs, this may be used as a vector for a denial of service
    attack.
  More info: https://pkg.go.dev/vuln/GO-2021-0113
  Module: golang.org/x/text
    Found in: golang.org/x/text@v0.3.6
    Fixed in: golang.org/x/text@v0.3.7
    Example traces found:
      #1: main.go:10:28: govulncheck.main calls language.Parse
      #2: main.go:14:13: govulncheck.main calls fmt.Println, which eventually calls language.Tag.String

=== Package Results ===

No other vulnerabilities found.

=== Module Results ===

No other vulnerabilities found.

Your code is affected by 1 vulnerability from 1 module.
This scan found no other vulnerabilities in packages you import or modules you
require.
```

สังเกตจุดสำคัญในผลลัพธ์:

- **"Example traces found"** แสดง **call chain จริง** ในโค้ดของเราที่นำไปสู่ symbol ที่มีช่องโหว่ (`main.go:10:28` เรียก `language.Parse` โดยตรง) — นี่คือสิ่งที่ทำให้ `govulncheck` ต่างจาก dependency scanner ทั่วไปที่แค่เทียบเลขเวอร์ชัน
- **"Fixed in: golang.org/x/text@v0.3.7"** บอกเวอร์ชันขั้นต่ำที่ต้องอัปเกรดไปเพื่อแก้ปัญหานี้โดยเฉพาะ
- Exit code ของ `govulncheck` เป็น `3` เมื่อพบช่องโหว่ที่โค้ดเรียกใช้จริง — ทำให้นำไปเป็นเงื่อนไข fail ใน CI pipeline ได้ตรงๆ (เชื่อมโยงกับ **Part 098**)

### 6.3 แก้ไขแล้วสแกนซ้ำ

```bash
go get golang.org/x/text@v0.3.7
go mod tidy
govulncheck -db "file://.../vulndb-v1" ./...
```

ผลลัพธ์จริงหลังอัปเกรด:

```
No vulnerabilities found.
```

Exit code เปลี่ยนเป็น `0` — นี่คือ workflow มาตรฐาน: **สแกน → เห็น call trace → อัปเกรด dependency ไปเวอร์ชันที่แก้แล้ว → สแกนซ้ำจนสะอาด**

### 6.4 ใช้ `govulncheck` เป็นส่วนหนึ่งของ CI

ต่อยอดจาก pipeline ใน **Part 098** เพิ่ม stage สแกนช่องโหว่แบบง่ายที่สุด:

```yaml
# .github/workflows/ci.yml (เพิ่มเป็น step ใหม่ในงานเดิม)
- name: Install govulncheck
  run: go install golang.org/x/vuln/cmd/govulncheck@latest

- name: Scan for known vulnerabilities
  run: govulncheck ./...
```

เพราะ exit code ไม่ใช่ศูนย์เมื่อพบช่องโหว่ที่ใช้งานจริง step นี้จะทำให้ **build fail โดยอัตโนมัติ** เมื่อมี dependency ที่มีช่องโหว่ถูกเรียกใช้จริงในโค้ด — ควรวางไว้เป็นด่านมาตรฐานคู่กับ `go vet` และ `golangci-lint` ตาม pattern "Fail Fast" ที่ Part 098 สอนไว้

### 6.5 `govulncheck` vs `go list -m -u all` / Dependabot / Renovate

เครื่องมืออย่าง **Dependabot** (GitHub) หรือ **Renovate** ทำงานคนละชั้นกับ `govulncheck`: มันสแกนว่า "มี dependency เวอร์ชันใหม่กว่าที่ประกาศแก้ CVE ไว้หรือไม่" โดยดูจากเลขเวอร์ชันอย่างเดียว (**ไม่วิเคราะห์ call graph**) จึงมักแจ้งเตือนถี่กว่าความจำเป็น (รวมถึง CVE ในโค้ดส่วนที่เราไม่เคยเรียกใช้เลย) แนวทางที่แนะนำสำหรับทีมคือ **ใช้ทั้งสองร่วมกัน**: ให้ Dependabot/Renovate เปิด PR อัปเดต dependency อัตโนมัติเป็นประจำ ส่วน `govulncheck` เป็นด่านตัดสินใน CI ว่า PR ไหน "จำเป็นเร่งด่วน" เพราะกระทบโค้ดที่ใช้งานจริง

---

## 8. Secrets Management: ห้าม Hardcode Credential เด็ดขาด

**Part 096** สอนการใช้ไฟล์ `.env` ร่วมกับ Docker Compose ไปแล้ว หลักการด้าน security ที่ต้องย้ำให้ชัดเจนในบทนี้คือ:

> **API key, database password, JWT signing secret, TLS private key ต้องไม่ปรากฏเป็นค่าคงที่ (hardcoded) อยู่ในซอร์สโค้ดเด็ดขาด ไม่ว่ากรณีใดก็ตาม** รวมถึงห้าม commit ไฟล์ `.env` ที่มีค่าจริงเข้า git repository

### 7.1 ทำไม hardcode ถึงอันตรายกว่าที่คิด

แม้แต่ repository แบบ private ก็มีความเสี่ยง: credential ที่ commit เข้า git **จะยังอยู่ใน git history ตลอดไป** แม้จะลบออกจากไฟล์ในภายหลังและ commit ใหม่ทับ — ต้องใช้เครื่องมือ rewrite history (เช่น `git filter-repo`) เพื่อลบออกจริง และถ้า repository เคยถูก push ไปที่ไหนแล้ว (fork, CI cache, colleague's local clone) ก็ไม่มีทางลบให้หมดจดได้ 100% วิธีที่ถูกต้องเมื่อ credential รั่วไหลเข้า git คือ**หมุนเวียน (rotate) credential นั้นทันที** ไม่ใช่พยายามลบออกจาก history

### 7.2 ลำดับขั้นที่แนะนำ: จาก `.env` สู่ Secret Manager

| ระดับ | วิธีเก็บ secret | เหมาะกับ |
|---|---|---|
| 1 | Environment variable ตั้งตรงๆ ตอน deploy | Dev environment ส่วนตัว |
| 2 | ไฟล์ `.env` + `.gitignore` (ตาม Part 096) | ทีมเล็ก, dev/staging |
| 3 | Kubernetes `Secret` object (ตาม Part 097) | Production บน Kubernetes |
| 4 | Secret Manager เฉพาะทาง (HashiCorp Vault, AWS Secrets Manager, GCP Secret Manager) | Production ระดับองค์กร ที่ต้องการ audit log, auto-rotation |

โค้ด Go ไม่จำเป็นต้องรู้ว่า secret มาจากระดับไหน — รูปแบบที่ดีคืออ่านผ่าน environment variable เสมอในระดับ application code แล้วปล่อยให้ **infrastructure layer** (Kubernetes manifest, deployment script, Vault agent) เป็นคนฉีดค่าเข้า environment variable ให้ ทำให้โค้ดแอปพลิเคชันเรียบง่ายและ portable ข้าม environment:

```go
package main

import (
	"fmt"
	"log"
	"os"
)

func mustGetEnv(key string) string {
	v := os.Getenv(key)
	if v == "" {
		log.Fatalf("required environment variable %q is not set", key)
	}
	return v
}

func main() {
	dbPassword := mustGetEnv("DB_PASSWORD")
	jwtSecret := mustGetEnv("JWT_SIGNING_SECRET")

	// ใช้ dbPassword, jwtSecret ต่อ — ไม่มีค่า secret ใดๆ ปรากฏในซอร์สโค้ดเลย
	fmt.Println("secrets loaded, lengths:", len(dbPassword), len(jwtSecret))
}
```

### 7.3 ป้องกันการรั่วไหลโดยไม่ตั้งใจผ่าน Logging

จุดที่มือใหม่มักพลาดคือ **log struct ทั้งก้อนที่มี field เก็บ secret ปนอยู่** เช่น log ทั้ง config struct ตอน startup โดยไม่ได้กรอง field รหัสผ่านออกก่อน วิธีป้องกันที่ทำได้ตั้งแต่ระดับ type:

```go
type Config struct {
	DBHost     string
	DBPassword string
}

// String กำหนด custom format เพื่อไม่ให้ fmt.Println/log พิมพ์ DBPassword ออกมาตรงๆ
// เมื่อมีใครเผลอ log ค่า Config ทั้งก้อน
func (c Config) String() string {
	return fmt.Sprintf("Config{DBHost: %q, DBPassword: <redacted>}", c.DBHost)
}
```

การ implement `fmt.Stringer` (ทบทวนจาก **Part 013**) แบบนี้ทำให้ **ไม่ว่าใครในทีมจะเผลอ `fmt.Println(cfg)` หรือ `log.Printf("%v", cfg)` ที่ไหนก็ตาม** ค่า secret จะไม่หลุดออกไปใน log โดยอัตโนมัติ — เป็นการป้องกันเชิงโครงสร้างที่ดีกว่าการหวังให้ทุกคนในทีม "จำได้เสมอ" ว่าต้องกรอง field ไหนออก

### 7.4 หมุนเวียน Secret เป็นประจำ (Rotation)

Secret Manager ระดับองค์กร (Vault, AWS/GCP Secrets Manager) รองรับการ **auto-rotate** credential ตามรอบเวลา (เช่น หมุนรหัสผ่าน database ทุก 30 วันอัตโนมัติ) หลักการที่โค้ด Go ต้องรองรับให้ได้คือ **ไม่ cache credential ไว้ถาวรตั้งแต่ startup โดยไม่มีทางรีเฟรช** — connection pool ของ `database/sql` ทำสิ่งนี้ให้อยู่แล้วผ่าน `SetConnMaxLifetime` (**Part 078**) ที่บังคับให้ connection เก่าถูกปิดและสร้างใหม่เป็นระยะ ทำให้ระบบรับ credential ใหม่ที่หมุนเวียนมาได้โดยไม่ต้อง restart แอป

---

## 9. Security Misconfiguration: ค่า Default ที่ปลอดภัย

"Security Misconfiguration" เป็นหมวดหมู่ OWASP ที่กว้างที่สุดและพบบ่อยที่สุดในทางปฏิบัติ เพราะไม่ใช่ bug ในโค้ด แต่เป็น**การตั้งค่าที่หลงเหลือความหละหลวมจากตอนพัฒนาไปจนถึง production** ตัวอย่างที่พบบ่อยใน Go web service:

- **เปิด debug endpoint (`net/http/pprof`) ไว้ใน production โดยไม่จำกัดการเข้าถึง** — `net/http/pprof` (**Part 083**) มีประโยชน์มากตอน debug แต่เปิดให้ทุกคนเข้าถึงได้ทาง public internet เท่ากับเปิดเผยข้อมูลภายในของโปรเซส (memory layout, goroutine stack, source path) ให้ผู้ไม่เกี่ยวข้อง แนวทางที่ถูกต้องคือ mount `pprof` endpoint บน port แยกที่เข้าถึงได้เฉพาะจาก internal network เท่านั้น
- **CORS ตั้งค่า `Access-Control-Allow-Origin: *` ร่วมกับ credential** — อนุญาตทุก origin เรียก API ที่ผูกกับ cookie/session ของผู้ใช้ได้ ซึ่งเปิดช่องให้เว็บไซต์อื่นแอบเรียก API แทนผู้ใช้โดยที่ผู้ใช้ไม่รู้ตัว (ระบุ origin ที่อนุญาตแบบเจาะจงเสมอเมื่อ endpoint นั้นต้องใช้ credential)
- **Error message ที่ตอบกลับ client มีรายละเอียดภายในระบบมากเกินไป** เช่น stack trace เต็ม, connection string ของ database, path ของไฟล์บนเครื่อง server — ควร log รายละเอียดเต็มไว้ฝั่ง server (ผ่าน `slog` ตาม **Part 054**) แต่ตอบกลับ client ด้วยข้อความทั่วไปที่ไม่เปิดเผยโครงสร้างภายใน
- **ไม่บังคับ HTTPS** ปล่อยให้ endpoint สำคัญ (login, API ที่มี token) เข้าถึงได้ผ่าน `http://` เปล่าๆ ทำให้ข้อมูลที่ส่งผ่านเครือข่าย (รวมถึง credential) เดินทางแบบไม่เข้ารหัส
- **ใช้ dependency เวอร์ชัน default/latest แบบไม่ pin** ทำให้ build ที่ได้ไม่ reproducible และเสี่ยงได้รับโค้ดที่เปลี่ยนแปลงโดยไม่รู้ตัว — `go.sum` (**Part 018**) แก้ปัญหานี้อยู่แล้วตราบใดที่ commit เข้า repository เสมอและไม่ถูกลบทิ้ง

หลักการรวมของหัวข้อนี้คือ **"secure by default"**: ออกแบบระบบให้ค่าเริ่มต้น (ก่อนใครมาปรับแต่งอะไรเพิ่ม) เป็นค่าที่ปลอดภัยที่สุดเท่าที่จะเป็นไปได้ ไม่ใช่ค่าที่สะดวกที่สุดสำหรับการพัฒนา แล้วค่อยผ่อนคลายเฉพาะจุดที่จำเป็นจริงๆ พร้อมเหตุผลที่บันทึกไว้ชัดเจน

---

## 10. Input Validation คือแนวป้องกันชั้นที่สอง (Defense in Depth)

ตลอดบทนี้เราเน้นว่า**การป้องกัน injection/deserialization ที่แท้จริงมาจากกลไกที่ถูกต้อง** (parameterized query, auto-escaping) ไม่ใช่จากการ validate input แต่ input validation ยังมีบทบาทสำคัญในฐานะ **แนวป้องกันชั้นที่สอง (defense in depth)** — หลักการที่ว่าระบบควรมีการป้องกันซ้อนกันหลายชั้น เพื่อที่ถ้าชั้นหนึ่งพลาดไป ยังมีอีกชั้นคอยกันไว้

หลักปฏิบัติ input validation ที่ดีใน Go:

### 9.1 Validate โครงสร้างด้วย struct tag + library

ต่อยอดจาก **Part 062** (Gin validation/binding) หลักการเดียวกันใช้ได้กับทุก framework: ประกาศกฎ validation ไว้ที่ struct โดยตรง ทำให้กฎอยู่ใกล้กับ type ที่มันควบคุม และตรวจสอบอัตโนมัติตอน bind request:

```go
type CreateUserRequest struct {
	Username string `json:"username" binding:"required,alphanum,min=3,max=32"`
	Email    string `json:"email" binding:"required,email"`
	Age      int    `json:"age" binding:"required,gte=13,lte=120"`
}
```

### 9.2 Allowlist ดีกว่า Denylist เสมอ

เมื่อต้อง validate ว่า input อยู่ในรูปแบบที่ยอมรับได้ ให้นิยาม**สิ่งที่อนุญาต** (allowlist) แทนที่จะพยายามไล่ห้าม**สิ่งที่ไม่อนุญาต** (denylist) เพราะ denylist ต้องคาดเดาล่วงหน้าให้ครบทุกกรณีที่อันตราย ซึ่งในทางปฏิบัติแทบเป็นไปไม่ได้ ตัวอย่างเช่น validate username ด้วย regex ที่ระบุอักขระที่**อนุญาต**ชัดเจน (`^[a-zA-Z0-9_]{3,32}$`) ปลอดภัยกว่าการพยายามไล่กรองอักขระที่ "อันตราย" ออกทีละตัว

### 9.3 Validate ที่ Boundary ของระบบ ไม่ใช่กระจายทั่วโค้ด

จัดวาง validation ไว้ที่จุดที่ข้อมูลเข้าสู่ระบบ (HTTP handler, message queue consumer) ให้ครบถ้วนตั้งแต่ต้นทาง เพื่อให้โค้ดชั้นในกว่านั้น (service layer, repository layer ตามที่ **Part 100** จะแนะนำเรื่อง Clean Architecture) สามารถ**เชื่อถือ**ข้อมูลที่ได้รับมาได้เต็มที่ โดยไม่ต้อง validate ซ้ำซ้อนกระจัดกระจายไปทั่วทุกชั้นของระบบ

### 9.4 อย่าลืม Validate ขนาดและ Rate ไม่ใช่แค่รูปแบบ

Input validation ไม่ได้มีแค่มิติ "รูปแบบถูกต้องไหม" แต่รวมถึง "ขนาดสมเหตุสมผลไหม" (เชื่อมโยงกับ `http.MaxBytesReader` ในหัวข้อ 6) และ "ความถี่สมเหตุสมผลไหม" — ข้อหลังคือหน้าที่ของ **rate limiting** ซึ่งจะกลับมาในหัวข้อ 11 และ Part 107

---

## 11. Checklist: Secure Defaults สำหรับ Go Web Service ใหม่

เมื่อเริ่มโปรเจกต์ Go web service ใหม่ ใช้ checklist นี้เป็นค่าเริ่มต้นมาตรฐานก่อนขึ้น production:

- [ ] **บังคับ HTTPS เท่านั้น** — redirect `http://` ไป `https://` ทุก request หรือปฏิเสธโดยตรงถ้าเป็น API ที่ไม่ควรมีทาง fallback (TLS จัดการอัตโนมัติผ่าน `net/http` + `crypto/tls` ตาม **Part 051**)
- [ ] **ตั้งค่า cookie flag ให้ครบ**: `Secure` (ส่งผ่าน HTTPS เท่านั้น), `HttpOnly` (JavaScript อ่านไม่ได้ ป้องกันการขโมย cookie ผ่าน XSS), `SameSite=Strict` หรือ `Lax` (ป้องกัน CSRF บางส่วน) ตามที่สาธิตใน **Part 068**
- [ ] **ตั้งค่า CORS แบบเจาะจง origin** ไม่ใช้ `*` ร่วมกับ endpoint ที่ผูก credential/cookie
- [ ] **ใส่ rate limiting ที่ endpoint สำคัญ** โดยเฉพาะ login/signup/password-reset เพื่อชะลอการเดารหัสผ่านแบบ brute-force ต่อยอดจากแนวคิด retry/backoff ใน **Part 094** (rate limiter ฝั่ง server ทำงานตรงข้ามกับ retry ฝั่ง client แต่ใช้หลักการ "จำกัดอัตรา" เดียวกัน)
- [ ] **ตั้ง timeout ให้ทุก `http.Server`** (`ReadTimeout`, `WriteTimeout`, `IdleTimeout`) ป้องกัน connection ที่ค้างนานผิดปกติดึง resource ของ server ไปเรื่อยๆ
- [ ] **จำกัดขนาด request body** ด้วย `http.MaxBytesReader` ทุก endpoint ที่รับข้อมูลจากผู้ใช้ภายนอก
- [ ] **ปิด debug endpoint (`net/http/pprof`) จาก public internet** — เปิดเฉพาะ internal network หรือ port แยกที่ไม่ expose ออกนอก
- [ ] **ไม่เปิดเผยรายละเอียด error ภายในให้ client** — log เต็มฝั่ง server, ตอบกลับ client แบบทั่วไป
- [ ] **ผ่าน `govulncheck` โดยไม่มีช่องโหว่ที่โค้ดเรียกใช้จริง** ก่อน merge ทุกครั้ง (เชื่อมโยง CI จาก **Part 098**)
- [ ] **ไม่มี secret ใดๆ hardcode ในซอร์สโค้ดหรือ commit เข้า git** — ตรวจสอบด้วย pre-commit hook หรือเครื่องมือสแกน secret ในโค้ด (เช่น `gitleaks`) เป็นด่านมาตรฐาน
- [ ] **Log ผ่าน `slog` แบบมีโครงสร้าง** พร้อม field ที่ช่วยตรวจจับความผิดปกติ (failed login count, unusual request pattern) ตาม **Part 054, 099**

---

## 12. Checklist: Security Code Review สำหรับทีม

นอกจาก checklist สำหรับตัวระบบแล้ว ทีมควรมี checklist สำหรับ**คนรีวิวโค้ด** ใช้ประกอบกับ pull request ทุกครั้งที่แตะเรื่อง input จากภายนอกหรือ authentication:

1. **มีการต่อ string เข้า SQL/shell command จากค่าที่มาจากผู้ใช้หรือไม่?** ถ้ามี ต้องเปลี่ยนเป็น parameterized query หรือ `exec.Command` แบบแยก argument ทันที
2. **มีการ render ข้อมูลจากผู้ใช้ลง HTML ผ่าน `text/template` แทน `html/template` หรือไม่?**
3. **มีการใช้ `template.HTML`/`template.JS`/`template.URL` ครอบข้อมูลที่มาจาก input โดยตรงหรือไม่?**
4. **Password/token ใหม่ที่เพิ่มเข้ามา hash ด้วย `bcrypt`/`argon2` หรือสุ่มด้วย `crypto/rand` หรือไม่** (ไม่ใช่ `sha256` เปล่าๆ หรือ `math/rand`)?
5. **Error message ที่ส่งกลับ client เปิดเผยรายละเอียดภายในระบบมากเกินไปหรือไม่?**
6. **มี field ที่เป็น secret (password, token, API key) ถูก log หรือ serialize ออกไปโดยไม่ตั้งใจหรือไม่** (เช็คทั้ง `fmt.Println`/`log.Printf` และ `json:"..."` tag ที่อาจ serialize field ที่ไม่ควร expose)?
7. **Endpoint ใหม่มีการตรวจสอบสิทธิ์ (authorization) ครบถ้วนหรือไม่** ไม่ใช่แค่ authentication (รู้ว่าใครเรียกมา) แต่รวมถึง authorization (ผู้เรียกมีสิทธิ์ทำสิ่งนี้จริงหรือไม่)?
8. **Dependency ใหม่ที่เพิ่มเข้ามาผ่าน `govulncheck` แล้วหรือยัง** และมาจากแหล่งที่น่าเชื่อถือ (จำนวน star, การดูแลรักษาอย่างต่อเนื่อง, ไม่ใช่ package ที่เพิ่งสร้างโดยไม่มีประวัติ)?
9. **Input จากภายนอกทุกจุดผ่านชั้น validation ก่อนใช้งานจริงหรือไม่** ทั้งรูปแบบ ขนาด และช่วงค่าที่สมเหตุสมผล?

Checklist ทั้งสองชุดนี้ไม่ใช่เอกสารที่เขียนครั้งเดียวแล้วจบ — ทีมที่ดีจะปรับปรุง checklist ต่อเนื่องทุกครั้งที่เจอปัญหาจริงใน production หรือพบจุดที่ checklist เดิมไม่ครอบคลุม

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- OWASP Top 10 เป็นแผนที่จัดระเบียบความเสี่ยงด้านความปลอดภัยที่อุตสาหกรรมยอมรับกันกว้างขวาง และเราได้เชื่อมโยงแต่ละหมวดหมู่เข้ากับสิ่งที่เรียนไปแล้วตลอดหลักสูตร (Part 051, 065, 067, 068, 071, 078, 094, 096, 097, 098)
- **Injection** ป้องกันด้วย parameterized query เสมอ ไม่มีข้อยกเว้น (ทบทวน Part 071)
- **Broken Authentication** ป้องกันด้วย `bcrypt`/`argon2` สำหรับรหัสผ่าน, `crypto/rand` สำหรับ token, และ JWT ที่ตั้งค่า algorithm/expiry ถูกต้อง (ทบทวน Part 051, 067)
- **XSS** ป้องกันด้วย auto-escaping ของ `html/template` และห้ามครอบข้อมูลจากผู้ใช้ด้วย `template.HTML` (ทบทวน Part 065)
- **Insecure Deserialization** ป้องกันด้วยการจำกัดขนาด input, `DisallowUnknownFields`, decode ลง type เจาะจง, และ validate ค่าหลัง decode เสมอ
- **`govulncheck`** วิเคราะห์ call graph จริงเพื่อบอกว่าโค้ดเราเรียกไปถึง symbol ที่มีช่องโหว่หรือไม่ — เราได้รันจริงจนเห็น `GO-2021-0113` ใน `golang.org/x/text@v0.3.6` แล้วอัปเกรดจนสแกนสะอาด (`No vulnerabilities found`)
- **Secrets management** ต้องไม่ hardcode credential ในโค้ดหรือ commit เข้า git เด็ดขาด ใช้ environment variable + secret manager ตามระดับความจำเป็นของทีม
- **Security Misconfiguration** เป็นความเสี่ยงที่พบบ่อยที่สุดในทางปฏิบัติ แก้ด้วยหลัก "secure by default"
- **Input validation** คือแนวป้องกันชั้นที่สอง (defense in depth) เสริมกลไกป้องกันหลัก ไม่ใช่ตัวป้องกันหลักเอง
- Checklist สองชุด (secure defaults + code review) ช่วยให้ทีมนำหลักการทั้งหมดไปใช้อย่างเป็นระบบ ไม่ใช่แค่ความรู้ในหัว

## แบบฝึกหัดท้ายบท

1. เขียนโปรแกรม Go เล็กๆ ที่มี dependency `golang.org/x/text@v0.3.6` เหมือนตัวอย่างในบทนี้ แล้วรัน `govulncheck` ในเครื่องตัวเอง (ถ้ามี network access ปกติไม่ถูกจำกัดแบบ sandbox นี้ ให้รันแบบไม่ต้องระบุ `-db` เอง) สังเกตผลลัพธ์เทียบกับที่บทนี้แสดงไว้
2. หยิบโปรเจกต์ Go ที่เคยเขียนไว้ในบทก่อนๆ ของหลักสูตรนี้ (เช่นจาก Part 060 หรือ Part 088) มารัน `govulncheck ./...` จริง ถ้าพบช่องโหว่ ให้อัปเกรด dependency ที่เกี่ยวข้องแล้วสแกนซ้ำจนสะอาด
3. เขียน middleware สำหรับ Gin/Echo (ทบทวนจาก Part 057, 061, 063) ที่ implement rate limiting แบบง่ายด้วย token bucket จำกัดจำนวน request ต่อ IP ต่อนาที สำหรับ endpoint `/login`
4. ทบทวนโค้ดระบบ signup/login ที่เขียนไว้ใน Part 051 แล้วเพิ่ม custom `String()` method ให้ struct `User`/`Config` ที่เกี่ยวข้อง เพื่อป้องกันไม่ให้ password hash หลุดออกไปใน log โดยไม่ตั้งใจ
5. เขียน checklist security code review ของบทนี้ (หัวข้อ 12) ให้เป็นไฟล์ `SECURITY_CHECKLIST.md` แล้วลองใช้ตรวจ pull request จริงหนึ่งอันจากโปรเจกต์ของตัวเอง (หรือ pull request สาธารณะของ open source project) — บันทึกว่าเจอข้อไหนที่น่าสนใจบ้าง
6. ค้นคว้าเพิ่มเติมเรื่อง `gitleaks` หรือ `trufflehog` (เครื่องมือสแกนหา secret ที่หลุดเข้า git history) ลองติดตั้งและรันกับ repository ของตัวเองเพื่อดูว่ามี credential ใดๆ หลุดเข้าไปโดยไม่ตั้งใจหรือไม่

---

**ต่อไป**: [Part 107 — Performance Tuning ระดับ Production](./107-performance-tuning-production.md)
