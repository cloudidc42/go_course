# Part 051: Cryptography พื้นฐาน: Hashing และ Encryption

> ภาคที่ 4: Standard Library เชิงลึก — ตอนที่ 6 จาก 10 (Part 46–55)

## สารบัญของบทนี้

1. Hashing กับ Encryption ต่างกันอย่างไร
2. Hashing ด้วย `crypto/sha256` และ `crypto/md5`
3. ทำไม MD5/SHA-1 ถึง "แตก" แล้ว แต่ยังมีคนใช้อยู่
4. Message Authentication ด้วย `crypto/hmac`
5. ทำไมห้าม hash รหัสผ่านด้วย SHA-256 เปล่าๆ
6. Password Hashing ให้ถูกวิธีด้วย `bcrypt`
7. ตัวอย่างเต็มรูปแบบ: ระบบ Signup/Login
8. Symmetric Encryption ด้วย `crypto/aes` โหมด GCM
9. `crypto/rand` vs `math/rand`: ทำไมต้องเลือกให้ถูก
10. สร้าง Secure Token ด้วย `crypto/rand`
11. TLS/HTTPS: `net/http` จัดการให้อัตโนมัติ
12. สรุปสิ่งที่ได้เรียนในบทนี้
13. แบบฝึกหัดท้ายบท

---

## 1. Hashing กับ Encryption ต่างกันอย่างไร

ก่อนเข้าเนื้อหา ต้องแยกแนวคิดพื้นฐานสองอย่างที่คนเริ่มต้นมักสับสนให้ชัดเจนก่อน เพราะทั้งคู่อยู่ใน package ตระกูล `crypto/*` เหมือนกัน แต่ใช้แก้ปัญหาคนละแบบ:

| | Hashing | Encryption |
|---|---|---|
| ทิศทาง | **ทางเดียว (one-way)** — แปลงกลับไม่ได้ | **สองทาง (two-way)** — เข้ารหัสแล้วถอดกลับได้ |
| จุดประสงค์ | ตรวจสอบความถูกต้อง/ตัวตน (integrity, identity) | ปกปิดเนื้อหา (confidentiality) |
| ต้องใช้ key ไหม | ปกติไม่ต้อง (hash เฉยๆ) หรือใช้ secret key (HMAC) | ต้องใช้ key เสมอ (symmetric หรือ asymmetric) |
| ตัวอย่างการใช้งาน | เก็บรหัสผ่าน, checksum ไฟล์, ลายเซ็นดิจิทัล | เข้ารหัสไฟล์ลับ, เข้ารหัสข้อมูลก่อนส่งผ่าน network |
| package ใน Go | `crypto/sha256`, `crypto/md5`, `crypto/hmac` | `crypto/aes`, `crypto/rsa` |

จำง่ายๆ ว่า **hash คือการ "ย่อย" ข้อมูลให้เป็นลายนิ้วมือขนาดคงที่ที่ไม่มีทางย้อนกลับเป็นข้อมูลเดิมได้** ส่วน **encryption คือการ "ล็อกกุญแจ" ข้อมูล ที่ใครมี key ที่ถูกต้องก็ปลดล็อกกลับมาได้**

ตลอดบทนี้เราจะใช้ทั้งสองแนวคิดผ่านตัวอย่างที่ทำงานได้จริง ทดสอบรันแล้วทุกตัวอย่างด้วย Go 1.24

---

## 2. Hashing ด้วย `crypto/sha256` และ `crypto/md5`

Go มี package แฮชมาให้ในกลุ่ม `crypto/*` เกือบทุกอัลกอริทึมที่ใช้กันในอุตสาหกรรม ที่พบบ่อยที่สุดคือ `crypto/sha256` และ `crypto/md5` โครงสร้าง API ของทั้งคู่คล้ายกันมาก

```go
package main

import (
	"crypto/md5"
	"crypto/sha256"
	"encoding/hex"
	"fmt"
)

func main() {
	data := []byte("hello, go course")

	// วิธีที่ 1: ใช้ Sum256 แบบ one-shot เมื่อมีข้อมูลอยู่ใน []byte ก้อนเดียวแล้ว
	sum256 := sha256.Sum256(data)
	fmt.Println("SHA-256:", hex.EncodeToString(sum256[:]))

	sumMD5 := md5.Sum(data)
	fmt.Println("MD5:", hex.EncodeToString(sumMD5[:]))

	// วิธีที่ 2: ใช้แบบ streaming ผ่าน hash.Hash interface
	// เหมาะกับข้อมูลขนาดใหญ่ที่ไม่อยากโหลดเข้า memory ทั้งก้อน (เช่น อ่านจากไฟล์ทีละส่วน)
	h := sha256.New()
	h.Write([]byte("hello, "))
	h.Write([]byte("go course"))
	fmt.Println("streamed SHA-256:", hex.EncodeToString(h.Sum(nil)))
}
```

ผลลัพธ์ (deterministic เสมอ — input เดิมได้ hash เดิมทุกครั้ง):

```
SHA-256: 5a7f8c0652e45ccacdf2376e0f0723b3a8a64f5fedba40c45b45f15dd78e07ce
MD5: fadb1e99ef3fb0481eccb0e315dfb54b
streamed SHA-256: 5a7f8c0652e45ccacdf2376e0f0723b3a8a64f5fedba40c45b45f15dd78e07ce
```

### จุดสำคัญที่ต้องสังเกต

- `sha256.Sum256(data)` คืนค่าเป็น `[32]byte` (array ขนาดคงที่ ไม่ใช่ slice) ต้อง slice ด้วย `[:]` ก่อนแปลงเป็น string ผ่าน `hex.EncodeToString`
- `md5.Sum(data)` คืน `[16]byte` เพราะ MD5 ให้ output สั้นกว่า SHA-256 (128 bit เทียบกับ 256 bit)
- API แบบ streaming (`h := sha256.New()` แล้ว `h.Write(...)` ซ้ำๆ) implement interface `io.Writer` ทำให้ใช้ร่วมกับ `io.Copy` ได้ตรงๆ เช่น การ hash ไฟล์ขนาดใหญ่โดยไม่ต้องอ่านทั้งไฟล์เข้า memory ก่อน — แนวคิดนี้ต่อยอดมาจาก interface composition ที่เราเรียนไปในบทเรื่อง `io.Reader`/`io.Writer`
- Hash เป็น **deterministic** เสมอ: input เดียวกันได้ output เดียวกันทุกครั้งทุกเครื่อง และเปลี่ยน input แม้แค่ 1 บิตก็ทำให้ output เปลี่ยนไปทั้งหมด (เรียกว่า avalanche effect)

### ใช้ hash ทำอะไรได้บ้าง

- **Checksum ไฟล์**: ตรวจสอบว่าไฟล์ที่ดาวน์โหลดมาไม่ถูกแก้ไขระหว่างทาง (เทียบ SHA-256 ที่ผู้เผยแพร่ประกาศไว้กับไฟล์ที่ได้จริง)
- **Deduplication**: ใช้ hash เป็น key ตรวจสอบว่าไฟล์/ข้อมูลซ้ำกันหรือไม่โดยไม่ต้องเทียบเนื้อหาทั้งหมด
- **Git commit hash**: Git ใช้ SHA-1 (กำลังทยอยเปลี่ยนเป็น SHA-256) เป็น identifier ของทุก commit/object

---

## 3. ทำไม MD5/SHA-1 ถึง "แตก" แล้ว แต่ยังมีคนใช้อยู่

นี่คือจุดที่ทำให้มือใหม่สับสนบ่อยที่สุด: ถ้า MD5 "แตก" (broken) แล้ว ทำไมยังเห็นโค้ดจำนวนมากใช้ `crypto/md5` อยู่?

คำตอบคือ **"แตก" ในที่นี้หมายถึงแตกในบริบทของความปลอดภัย (cryptographic security) เท่านั้น ไม่ได้แปลว่าใช้งานไม่ได้เลย**

### MD5 และ SHA-1 แตกอย่างไร

คุณสมบัติที่ hash function ที่ปลอดภัยต้องมีคือ **collision resistance** — แทบเป็นไปไม่ได้ในทางปฏิบัติที่จะหา input สองตัวที่ต่างกันแต่ได้ hash เดียวกัน (เรียกว่า collision)

- **MD5**: นักวิจัยพบวิธีสร้าง collision ได้ตั้งแต่ปี 2004 และปัจจุบันสร้างได้ในเวลาไม่กี่วินาทีบนคอมพิวเตอร์ทั่วไป
- **SHA-1**: Google และ CWI Amsterdam สาธิตการโจมตี "SHAttered" สำเร็จในปี 2017 สร้างไฟล์ PDF สองไฟล์ที่เนื้อหาต่างกันแต่ SHA-1 hash เหมือนกันเป๊ะ

เมื่อ collision สร้างได้จริง ระบบที่พึ่งพา hash เพื่อความปลอดภัย เช่น ลายเซ็นดิจิทัลหรือใบรับรอง SSL จะถูกโจมตีได้ — ผู้ไม่หวังดีสามารถสร้างไฟล์ปลอมที่มี hash เดียวกับไฟล์จริง แล้วสวมรอยได้

### แล้วทำไมยังใช้ MD5/SHA-1 อยู่

เพราะงานหลายอย่าง **ไม่ได้ต้องการความปลอดภัยระดับ cryptographic** แค่ต้องการ "ลายนิ้วมือ" ที่คำนวณเร็วและชนกันยาก (แต่ไม่ต้องยากถึงขั้นทนต่อผู้โจมตีที่ตั้งใจ):

- **ETag ของ HTTP response**: เว็บเซิร์ฟเวอร์จำนวนมากใช้ MD5 คำนวณ ETag เพื่อบอก client ว่าเนื้อหาเปลี่ยนหรือไม่ ไม่มีใครพยายามโจมตี ETag เพื่อสวมรอย
- **Checksum ตรวจไฟล์เสียหายจากการส่งผ่านเครือข่าย** (ไม่ใช่จากผู้โจมตี): MD5 เร็วและเพียงพอ
- **Deduplication ใน storage system**: ต้องการความเร็วสูงมากกว่าความปลอดภัยระดับ cryptographic
- **Legacy system**: ระบบเก่าจำนวนมากยังผูกกับ MD5/SHA-1 การเปลี่ยนอาจกระทบ backward compatibility

### กฎที่ต้องจำ

> **ห้ามใช้ MD5 หรือ SHA-1 ในบริบทที่เกี่ยวกับความปลอดภัย** เช่น การเก็บรหัสผ่าน, ลายเซ็นดิจิทัล, ใบรับรอง, หรือสิ่งที่ผู้โจมตีมีแรงจูงใจจะปลอมแปลงให้ hash ตรงกัน สำหรับงานเหล่านี้ให้ใช้ตระกูล **SHA-256 ขึ้นไป** (`crypto/sha256`, `crypto/sha512`) หรือกรณีรหัสผ่านให้ใช้ `bcrypt`/`argon2` โดยเฉพาะ (จะเรียนในหัวข้อถัดไป)

Go ยังคง export `crypto/md5` และ `crypto/sha1` ให้ใช้งานได้ตามปกติ เพราะยอมรับว่ามี use case ที่ไม่เกี่ยวกับความปลอดภัยอยู่จริง แต่ compiler และ linter บางตัว (เช่น `gosec`) จะเตือนถ้าเห็นว่านำไปใช้เก็บรหัสผ่านหรือทำ digital signature

---

## 4. Message Authentication ด้วย `crypto/hmac`

Hash ธรรมดาบอกได้แค่ว่า "ข้อมูลนี้ตรงกับต้นฉบับหรือไม่" แต่บอกไม่ได้ว่า **ใครเป็นคนสร้างข้อมูลนี้** เพราะใครก็คำนวณ SHA-256 ของข้อมูลอะไรก็ได้ ถ้าผู้โจมตีแก้ไขข้อมูลแล้วคำนวณ hash ใหม่แนบไปด้วย ผู้รับก็ตรวจไม่พบความผิดปกติ

**HMAC (Hash-based Message Authentication Code)** แก้ปัญหานี้ด้วยการผสม **secret key** เข้าไปในการคำนวณ hash ทำให้มีแต่ผู้ที่รู้ key เท่านั้นที่สร้างหรือตรวจสอบลายเซ็นได้ถูกต้อง

```go
package main

import (
	"crypto/hmac"
	"crypto/sha256"
	"encoding/hex"
	"fmt"
)

func main() {
	secret := []byte("super-secret-key")
	message := []byte(`{"user":"alice","amount":1000}`)

	// ฝั่งผู้ส่ง: คำนวณลายเซ็นด้วย secret key
	mac := hmac.New(sha256.New, secret)
	mac.Write(message)
	signature := mac.Sum(nil)
	fmt.Println("HMAC:", hex.EncodeToString(signature))

	// ฝั่งผู้รับ: คำนวณลายเซ็นใหม่จากข้อความที่ได้รับ แล้วเทียบกับที่แนบมา
	verify := hmac.New(sha256.New, secret)
	verify.Write(message)
	expected := verify.Sum(nil)
	fmt.Println("valid signature:", hmac.Equal(signature, expected))

	// ถ้าข้อความถูกแก้ไข (แม้แค่ตัวเลขเดียว) ลายเซ็นจะไม่ตรงกันทันที
	tampered := hmac.New(sha256.New, secret)
	tampered.Write([]byte(`{"user":"alice","amount":9999999}`))
	fmt.Println("tampered signature valid:", hmac.Equal(signature, tampered.Sum(nil)))
}
```

ผลลัพธ์:

```
HMAC: 2d37f23e782cd94112ac56e9d273834661b71f61f1d7b299eeac1876af967fc6
valid signature: true
tampered signature valid: false
```

### จุดสำคัญ

- `hmac.New(sha256.New, secret)` รับ **constructor function** ของ hash algorithm (`sha256.New`) เป็นพารามิเตอร์ ไม่ใช่ตัว hash ที่คำนวณแล้ว — ทำให้ใช้ HMAC ร่วมกับ hash algorithm ไหนก็ได้ (SHA-256, SHA-512, ฯลฯ)
- **ห้ามเทียบ signature ด้วย `==` หรือ `bytes.Equal` ตรงๆ** ต้องใช้ `hmac.Equal` เท่านั้น เพราะฟังก์ชันนี้ใช้ **constant-time comparison** — เทียบทุกไบต์โดยใช้เวลาเท่ากันเสมอไม่ว่าจะตรงกันกี่ไบต์แรก การเทียบแบบปกติ (`==`) จะหยุดทันทีที่เจอไบต์แรกที่ไม่ตรง ทำให้เวลาที่ใช้ต่างกันเล็กน้อยตามตำแหน่งที่ไม่ตรง ผู้โจมตีสามารถวัดเวลาตอบสนองนี้ (เรียกว่า **timing attack**) เพื่อเดา signature ทีละไบต์ได้ในทางทฤษฎี
- HMAC ใช้กันแพร่หลายมากใน: JWT (JSON Web Token แบบ HS256), การเซ็น webhook payload (เช่น Stripe, GitHub ส่ง header `X-Hub-Signature` มาให้ตรวจสอบด้วย HMAC), API request signing

---

## 5. ทำไมห้าม hash รหัสผ่านด้วย SHA-256 เปล่าๆ

มือใหม่จำนวนมากเข้าใจผิดว่า "hash รหัสผ่านด้วย SHA-256 ก่อนเก็บลง database ก็ปลอดภัยแล้ว" — **ความเข้าใจนี้ผิดและอันตรายมาก** ในระบบจริง

### ปัญหาของ SHA-256/MD5 กับรหัสผ่าน

1. **เร็วเกินไป**: SHA-256 ถูกออกแบบมาให้คำนวณเร็วที่สุดเท่าที่จะทำได้ (เหมาะกับ checksum) การ์ดจอรุ่นใหม่คำนวณ SHA-256 ได้หลาย **พันล้าน** ครั้งต่อวินาที ทำให้ผู้โจมตีที่ขโมย database ไปสามารถ **brute-force** (ลองรหัสผ่านที่เป็นไปได้ทั้งหมด) หรือใช้ **rainbow table** (ตารางสำเร็จรูปที่คำนวณ hash ของรหัสผ่านยอดนิยมไว้ล่วงหน้า) เพื่อถอดรหัสผ่านคืนได้ในเวลาอันสั้น
2. **ไม่มี salt ในตัว**: ถ้าผู้ใช้สองคนตั้งรหัสผ่านเดียวกัน (`123456`) จะได้ SHA-256 hash เหมือนกันเป๊ะ ทำให้ผู้โจมตีที่เจาะ database ได้เห็นรูปแบบและถอดรหัสได้ง่ายขึ้นมากด้วย rainbow table ที่คำนวณไว้ล่วงหน้า

### สิ่งที่ password hashing function ที่ถูกต้องต้องมี

- **ช้าโดยตั้งใจ (deliberately slow)**: ออกแบบให้คำนวณช้าและปรับความช้าได้ (cost factor) เพื่อให้ตามทันฮาร์ดแวร์ที่เร็วขึ้นเรื่อยๆ
- **Salt อัตโนมัติ**: สุ่มค่า salt ที่ไม่ซ้ำกันสำหรับผู้ใช้แต่ละคน แล้วฝัง salt นั้นไว้ในผลลัพธ์เอง ทำให้รหัสผ่านเดียวกันได้ hash ต่างกันเสมอ และป้องกัน rainbow table ได้โดยสมบูรณ์
- **ต้านทาน hardware ที่ทำงานขนานสูง (GPU/ASIC)**: อัลกอริทึมสมัยใหม่อย่าง Argon2 ยังออกแบบให้ใช้ memory เยอะเพื่อทำให้การโจมตีด้วย GPU จำนวนมากทำได้ยากขึ้น (memory-hard function)

อัลกอริทึมที่ออกแบบมาสำหรับรหัสผ่านโดยเฉพาะ ได้แก่ **bcrypt**, **scrypt**, และ **Argon2** (ผู้ชนะการแข่งขัน Password Hashing Competition ปี 2015 และเป็นตัวเลือกที่แนะนำที่สุดในปัจจุบันสำหรับระบบใหม่) ในบทนี้เราจะใช้ **bcrypt** เพราะเสถียร ใช้กันแพร่หลายที่สุด และมี library คุณภาพสูงพร้อมใช้ใน Go ecosystem

---

## 6. Password Hashing ให้ถูกวิธีด้วย `bcrypt`

`bcrypt` ไม่ได้อยู่ใน standard library ของ Go โดยตรง แต่อยู่ใน **`golang.org/x/crypto`** ซึ่งเป็น package กลุ่ม "extended" ที่ทีม Go ดูแลอย่างเป็นทางการ (ต่างจาก third-party library ทั่วไป) และถือเป็นตัวเลือกมาตรฐานที่ชุมชน Go แนะนำสำหรับงานนี้

ติดตั้งด้วยคำสั่ง:

```bash
go get golang.org/x/crypto/bcrypt
```

### ตัวอย่างการใช้งานพื้นฐาน

```go
package main

import (
	"fmt"
	"log"

	"golang.org/x/crypto/bcrypt"
)

func hashPassword(password string) (string, error) {
	// bcrypt.DefaultCost คือ 10 ในปัจจุบัน — ยิ่งค่าสูง ยิ่งช้าและปลอดภัยขึ้น
	// ค่าที่สูงขึ้นแต่ละหน่วยทำให้เวลาคำนวณเพิ่มขึ้นเป็นสองเท่า (exponential)
	hash, err := bcrypt.GenerateFromPassword([]byte(password), bcrypt.DefaultCost)
	if err != nil {
		return "", err
	}
	return string(hash), nil
}

func checkPassword(password, hash string) bool {
	err := bcrypt.CompareHashAndPassword([]byte(hash), []byte(password))
	return err == nil
}

func main() {
	hash, err := hashPassword("s3cr3t-password")
	if err != nil {
		log.Fatal(err)
	}
	fmt.Println("hash:", hash)
	fmt.Println("len:", len(hash))
	fmt.Println("correct password:", checkPassword("s3cr3t-password", hash))
	fmt.Println("wrong password:", checkPassword("wrong", hash))

	// hash คนละครั้งของรหัสผ่านเดียวกัน ได้ผลลัพธ์ไม่เหมือนกัน (เพราะ salt สุ่มใหม่ทุกครั้ง)
	hash2, _ := hashPassword("s3cr3t-password")
	fmt.Println("same password, different hash each time:", hash != hash2)
}
```

ผลลัพธ์ (ค่า hash จะสุ่มไม่เหมือนกันทุกครั้งที่รัน แต่ความยาวและรูปแบบคงที่):

```
hash: $2a$10$YyTLq/O9ONd3FFY8M04oJ.j3voy29sC5dNQlkzYqwFbFB8Q014O5q
len: 60
correct password: true
wrong password: false
same password, different hash each time: true
```

### อ่านโครงสร้างของ bcrypt hash

สตริง `$2a$10$YyTLq/O9ONd3FFY8M04oJ.j3voy29sC5dNQlkzYqwFbFB8Q014O5q` ไม่ใช่ค่าสุ่มไร้ความหมาย แต่แบ่งเป็นส่วนตามรูปแบบมาตรฐานของ bcrypt:

| ส่วน | ความหมาย |
|---|---|
| `$2a$` | เวอร์ชันของ algorithm bcrypt |
| `10$` | cost factor (จำนวนรอบการคำนวณ = 2^10) |
| 22 ตัวอักษรถัดมา | salt ที่สุ่มมา (เข้ารหัส base64 แบบพิเศษของ bcrypt) |
| ส่วนที่เหลือ | ผลลัพธ์ hash จริง |

จุดสำคัญคือ **salt ถูกฝังอยู่ในผลลัพธ์เดียวกันเลย** ทำให้เราไม่ต้องสร้างตาราง/คอลัมน์แยกเก็บ salt เอง — แค่เก็บสตริงผลลัพธ์จาก `GenerateFromPassword` ทั้งก้อนลง database ก็เพียงพอ และตอน `CompareHashAndPassword` ฟังก์ชันจะแกะ salt ออกมาจาก hash ที่เก็บไว้เองโดยอัตโนมัติ

### เลือกค่า cost อย่างไร

`bcrypt.DefaultCost` (ปัจจุบันคือ 10) เหมาะกับการใช้งานทั่วไป ยิ่งเพิ่มค่า (สูงสุดคือ 31) ยิ่งปลอดภัยขึ้นแต่ก็ยิ่งใช้เวลาคำนวณนานขึ้นแบบทวีคูณ — หลักปฏิบัติทั่วไปคือปรับค่าให้การ hash ใช้เวลาประมาณ 100-300 มิลลิวินาทีบน hardware ที่ใช้งานจริง (ไม่เร็วเกินจนผู้โจมตี brute-force ได้ง่าย แต่ไม่ช้าเกินจนกระทบ user experience ตอน signup/login)

---

## 7. ตัวอย่างเต็มรูปแบบ: ระบบ Signup/Login

มาประกอบทุกอย่างเข้าด้วยกันเป็นระบบ signup/login แบบง่าย ที่ทำตามหลักปฏิบัติที่ถูกต้องทุกข้อ:

```go
package main

import (
	"errors"
	"fmt"

	"golang.org/x/crypto/bcrypt"
)

// User เก็บเฉพาะ hash ของรหัสผ่าน ไม่เก็บรหัสผ่านจริงเด็ดขาด
type User struct {
	Username     string
	PasswordHash string
}

// userStore จำลองฐานข้อมูลแบบง่ายด้วย map ในหน่วยความจำ
type userStore struct {
	users map[string]User
}

func newUserStore() *userStore {
	return &userStore{users: make(map[string]User)}
}

var ErrUserExists = errors.New("username already taken")
var ErrInvalidCredentials = errors.New("invalid username or password")

// SignUp รับรหัสผ่านตัวเปล่า (plaintext) จากผู้ใช้ แล้ว hash ก่อนเก็บเสมอ
func (s *userStore) SignUp(username, password string) error {
	if _, exists := s.users[username]; exists {
		return ErrUserExists
	}

	hash, err := bcrypt.GenerateFromPassword([]byte(password), bcrypt.DefaultCost)
	if err != nil {
		return fmt.Errorf("failed to hash password: %w", err)
	}

	s.users[username] = User{
		Username:     username,
		PasswordHash: string(hash),
	}
	return nil
}

// Login ตรวจสอบรหัสผ่านโดยไม่เคย decrypt hash กลับมาเป็น plaintext เลย
func (s *userStore) Login(username, password string) error {
	user, exists := s.users[username]
	if !exists {
		// สำคัญ: ไม่บอกแยกว่า "username ไม่มี" กับ "รหัสผ่านผิด"
		// เพื่อป้องกันผู้โจมตี enumerate ว่า username ใดมีอยู่จริงในระบบ
		return ErrInvalidCredentials
	}

	err := bcrypt.CompareHashAndPassword([]byte(user.PasswordHash), []byte(password))
	if err != nil {
		return ErrInvalidCredentials
	}
	return nil
}

func main() {
	store := newUserStore()

	if err := store.SignUp("alice", "correct-horse-battery-staple"); err != nil {
		fmt.Println("signup error:", err)
	} else {
		fmt.Println("signup ok for alice")
	}

	if err := store.SignUp("alice", "another-password"); err != nil {
		fmt.Println("signup error (expected):", err)
	}

	fmt.Println("stored hash:", store.users["alice"].PasswordHash)

	if err := store.Login("alice", "correct-horse-battery-staple"); err != nil {
		fmt.Println("login error:", err)
	} else {
		fmt.Println("login ok for alice")
	}

	if err := store.Login("alice", "wrong-password"); err != nil {
		fmt.Println("login error (expected):", err)
	}

	if err := store.Login("bob", "anything"); err != nil {
		fmt.Println("login error (expected, unknown user):", err)
	}
}
```

ผลลัพธ์:

```
signup ok for alice
signup error (expected): username already taken
stored hash: $2a$10$/a48IveE18z1NgmamFwa7eoVaX9YKGowzu6YfSGhFtupopvyHWiDS
login ok for alice
login error (expected): invalid username or password
login error (expected, unknown user): invalid username or password
```

สังเกตหลักปฏิบัติที่ดี 3 ข้อในตัวอย่างนี้:

1. **ไม่มีที่ไหนในโค้ดที่เก็บหรือ log รหัสผ่านตัวเปล่าเลย** — รับเข้ามาแล้ว hash ทันที
2. **ข้อความ error ไม่บอกรายละเอียดว่าผิดจากอะไร** (username ไม่มี vs รหัสผ่านผิด) เพื่อไม่ให้ผู้โจมตีใช้ error message เป็นเครื่องมือเดา username ที่มีอยู่จริง
3. **struct `User` ไม่มี field เก็บรหัสผ่านตัวเปล่าเลยแม้แต่ field เดียว** — ทำให้ไม่มีทางเผลอ log หรือ serialize รหัสผ่านออกไปโดยไม่ได้ตั้งใจ

ในระบบจริงยังต้องเพิ่มเรื่อง rate limiting (จำกัดจำนวนครั้งที่ login ผิดได้), password complexity requirement, และการส่งผ่าน HTTPS เท่านั้น ซึ่งจะกลับมาพูดถึงอีกครั้งในภาคที่ 5 (Web Development) ตอน Authentication และภาคที่ 10 ตอน Security Best Practices

---

## 8. Symmetric Encryption ด้วย `crypto/aes` โหมด GCM

Hashing เหมาะกับการเก็บรหัสผ่าน แต่ถ้าต้องการ **เข้ารหัสข้อมูลแล้วถอดกลับมาได้** เช่น เข้ารหัสไฟล์ลับ หรือเข้ารหัสข้อมูลก่อนเก็บลง database ต้องใช้ **encryption** แทน

**AES (Advanced Encryption Standard)** เป็นอัลกอริทึม symmetric encryption ที่ใช้กันแพร่หลายที่สุดในโลก (symmetric แปลว่าใช้ **key เดียวกัน** ทั้งเข้ารหัสและถอดรหัส) โหมดที่แนะนำให้ใช้ในปัจจุบันคือ **GCM (Galois/Counter Mode)** เพราะนอกจากเข้ารหัสข้อมูลแล้ว ยังตรวจสอบ **integrity** ให้ในตัว (คล้าย HMAC ในตัวเดียวกัน) ทำให้รู้ทันทีถ้ามีใครแก้ไข ciphertext ระหว่างทาง

```go
package main

import (
	"crypto/aes"
	"crypto/cipher"
	"crypto/rand"
	"encoding/base64"
	"errors"
	"fmt"
	"io"
)

// encrypt เข้ารหัส plaintext ด้วย AES-GCM แล้วคืนผลลัพธ์เป็น base64 string
// นำ nonce ไปแปะไว้ข้างหน้า ciphertext เพื่อให้ decrypt แกะออกมาใช้ได้ทีหลัง
func encrypt(plaintext, key []byte) (string, error) {
	block, err := aes.NewCipher(key)
	if err != nil {
		return "", err
	}
	gcm, err := cipher.NewGCM(block)
	if err != nil {
		return "", err
	}

	// nonce ต้อง "ไม่ซ้ำกัน" ทุกครั้งที่เข้ารหัสด้วย key เดียวกัน (ห้ามนำ nonce กลับมาใช้ซ้ำเด็ดขาด)
	// จึงต้องสุ่มด้วย crypto/rand เสมอ ไม่ใช่ math/rand
	nonce := make([]byte, gcm.NonceSize())
	if _, err := io.ReadFull(rand.Reader, nonce); err != nil {
		return "", err
	}

	// Seal จะ "แปะ" nonce ไว้ข้างหน้าผลลัพธ์ให้อัตโนมัติผ่าน argument แรก (dst)
	ciphertext := gcm.Seal(nonce, nonce, plaintext, nil)
	return base64.StdEncoding.EncodeToString(ciphertext), nil
}

// decrypt แกะ nonce ออกจากหน้า ciphertext แล้วถอดรหัสและตรวจสอบ integrity ไปพร้อมกัน
func decrypt(encoded string, key []byte) ([]byte, error) {
	data, err := base64.StdEncoding.DecodeString(encoded)
	if err != nil {
		return nil, err
	}
	block, err := aes.NewCipher(key)
	if err != nil {
		return nil, err
	}
	gcm, err := cipher.NewGCM(block)
	if err != nil {
		return nil, err
	}

	nonceSize := gcm.NonceSize()
	if len(data) < nonceSize {
		return nil, errors.New("ciphertext too short")
	}
	nonce, ciphertext := data[:nonceSize], data[nonceSize:]

	// Open จะ error ทันทีถ้า ciphertext ถูกแก้ไข หรือ key ไม่ถูกต้อง
	return gcm.Open(nil, nonce, ciphertext, nil)
}

func main() {
	// AES-256 ต้องใช้ key ยาว 32 ไบต์ (16 = AES-128, 24 = AES-192, 32 = AES-256)
	key := make([]byte, 32)
	if _, err := io.ReadFull(rand.Reader, key); err != nil {
		panic(err)
	}

	plaintext := []byte("this is a top secret message")
	encoded, err := encrypt(plaintext, key)
	if err != nil {
		panic(err)
	}
	fmt.Println("encrypted (base64):", encoded)

	decoded, err := decrypt(encoded, key)
	if err != nil {
		panic(err)
	}
	fmt.Println("decrypted:", string(decoded))

	// ทดสอบ tamper: แก้ไข ciphertext แล้วลอง decrypt ควรได้ error ทันที
	tamperedInput := encoded[:len(encoded)-4] + "abcd"
	_, err = decrypt(tamperedInput, key)
	fmt.Println("tampered decrypt error:", err != nil)
}
```

ผลลัพธ์ (ค่า encrypted จะไม่เหมือนเดิมทุกครั้งที่รัน เพราะ nonce สุ่มใหม่เสมอ):

```
encrypted (base64): lVz671RXeNpzOctv3eMH5hMhMsnv5HmQUHVi62+NkLuc5QrPAwTF/ZjcpTEK2kFN1B9yELFZM3k=
decrypted: this is a top secret message
tampered decrypt error: true
```

### จุดสำคัญที่ห้ามพลาดเรื่อง nonce

- **Nonce (Number used ONCE)** ต้องไม่ซ้ำกันเลยตลอดอายุการใช้งานของ key เดียวกัน ถ้า nonce ซ้ำกันสองครั้งกับ key เดียวกัน ผู้โจมตีสามารถวิเคราะห์เพื่อกู้ plaintext กลับมาได้บางส่วนหรือทั้งหมด — นี่คือจุดที่มือใหม่พลาดบ่อยที่สุดเวลาเขียนโค้ด AES-GCM เอง
- วิธีที่ปลอดภัยที่สุดคือ**สุ่ม nonce ใหม่ทุกครั้งด้วย `crypto/rand`** ตามตัวอย่างข้างบน ขนาดมาตรฐานของ GCM nonce คือ 12 ไบต์ (`gcm.NonceSize()` จะคืนค่านี้ให้)
- เทคนิคทั่วไปคือแปะ nonce ไว้ข้างหน้า ciphertext (ตามตัวอย่าง) เพราะ nonce ไม่ใช่ความลับ — ไม่จำเป็นต้องเข้ารหัส แค่ต้องไม่ซ้ำเท่านั้น ฝั่ง decrypt จะแกะ nonce ที่แปะไว้ออกมาใช้ได้ทันที
- **ห้ามใช้ AES โหมด ECB เด็ดขาด** เพราะ ECB เข้ารหัสแต่ละ block แยกกันโดยไม่ผสมกับ block ก่อนหน้า ทำให้ pattern ของข้อมูลต้นฉบับรั่วไหลออกมาใน ciphertext ได้ (ตัวอย่างคลาสสิกคือภาพ ECB Penguin ที่ยังเห็นรูปเพนกวินได้แม้เข้ารหัสแล้ว) Go standard library ไม่มี helper สำหรับ ECB มาให้ตรงๆ ด้วยเหตุผลนี้โดยเจตนา — ให้ใช้ **GCM** เสมอสำหรับงานทั่วไป

---

## 9. `crypto/rand` vs `math/rand`: ทำไมต้องเลือกให้ถูก

Go มี package สุ่มเลขสองตัวที่ชื่อคล้ายกันแต่ใช้แทนกันไม่ได้เด็ดขาดในบริบทที่เกี่ยวกับความปลอดภัย:

| | `math/rand` (และ `math/rand/v2`) | `crypto/rand` |
|---|---|---|
| ประเภท | Pseudo-random (PRNG) | Cryptographically secure (CSPRNG) |
| ความเร็ว | เร็วมาก | ช้ากว่า (ต้องอ่านจาก entropy source ของ OS) |
| คาดเดาได้ไหม | **คาดเดาได้** ถ้ารู้ seed และ algorithm | คาดเดาไม่ได้ในทางปฏิบัติ |
| แหล่งที่มาของความสุ่ม | สูตรคณิตศาสตร์ (deterministic algorithm) | Operating system (`/dev/urandom` บน Linux, `CryptGenRandom` บน Windows) |
| ใช้ทำอะไร | เกม, การจำลอง (simulation), สุ่มลำดับที่ไม่กระทบความปลอดภัย | token, session key, encryption key, nonce, salt, password reset link |

### ทำไม `math/rand` ถึงคาดเดาได้

`math/rand` ใช้ **algorithm ที่กำหนดแน่นอน (deterministic)** โดยเริ่มจากค่า **seed** ตัวหนึ่ง แล้วสร้างลำดับตัวเลขที่ "ดูเหมือนสุ่ม" ออกมาต่อเนื่องกัน — ถ้าผู้โจมตีรู้ seed (หรือเดาได้ เช่น seed มาจาก `time.Now().UnixNano()` ซึ่งเดาช่วงเวลาได้ไม่ยาก) ก็สามารถคำนวณลำดับตัวเลขทั้งหมดที่จะสุ่มออกมาได้ล่วงหน้าทั้งหมด

ในทางกลับกัน `crypto/rand` ไม่มี concept ของ "seed" ที่ผู้ใช้กำหนดเองเลย — มันอ่านค่าความสุ่มจริงจากฮาร์ดแวร์/ระบบปฏิบัติการโดยตรง (entropy จาก interrupt ของ hardware, การเคลื่อนไหวเมาส์, timing ของดิสก์ ฯลฯ) ทำให้คาดเดาไม่ได้แม้จะรู้ algorithm ทั้งหมด

### กฎเหล็ก

> **สิ่งใดก็ตามที่เกี่ยวกับความปลอดภัย ต้องใช้ `crypto/rand` เท่านั้น** — session token, API key, encryption key, nonce, salt, ลิงก์รีเซ็ตรหัสผ่าน, CSRF token ทั้งหมดนี้ถ้าใช้ `math/rand` แล้วถูกคาดเดาได้ ผู้โจมตีสามารถสวมรอยเป็นผู้ใช้คนอื่นได้ทันที ในทางกลับกัน ถ้าแค่ต้องการสุ่มลำดับไพ่ในเกม หรือสุ่มตัวอย่างข้อมูลสำหรับการทดสอบ ใช้ `math/rand` ก็เพียงพอและเร็วกว่ามาก

---

## 10. สร้าง Secure Token ด้วย `crypto/rand`

Use case ที่พบบ่อยที่สุดของ `crypto/rand` ในงาน backend คือการสร้าง token แบบสุ่มที่ปลอดภัย เช่น session token, API key, หรือ password reset token

```go
package main

import (
	"crypto/rand"
	"encoding/base64"
	"fmt"
)

// generateToken สร้าง token ขนาด n ไบต์ที่สุ่มแบบปลอดภัย แล้ว encode เป็น URL-safe base64
func generateToken(n int) (string, error) {
	b := make([]byte, n)
	if _, err := rand.Read(b); err != nil {
		return "", err
	}
	// ใช้ URLEncoding แบบไม่มี padding (=) เพื่อให้นำไปใช้ใน URL ได้ตรงๆ โดยไม่ต้อง escape
	return base64.URLEncoding.WithPadding(base64.NoPadding).EncodeToString(b), nil
}

func main() {
	token, err := generateToken(32)
	if err != nil {
		panic(err)
	}
	fmt.Println("secure token:", token)
	fmt.Println("token length:", len(token))
}
```

ผลลัพธ์ (ค่าจะสุ่มไม่ซ้ำกันทุกครั้งที่รัน):

```
secure token: 1ENMC5VWhr_zU2vZ9k-Qj07V4NDCHkop2zO0SZnixfY
token length: 43
```

### เลือกขนาด token เท่าไรดี

โดยทั่วไปแนะนำให้ใช้ **อย่างน้อย 128 บิต (16 ไบต์) ของ entropy** สำหรับ token ที่ต้องคาดเดาไม่ได้ในทางปฏิบัติ ตัวอย่างข้างบนใช้ 32 ไบต์ (256 บิต) ซึ่งเผื่อเหลือเผื่อขาดมาก เพียงพอสำหรับ session token หรือ API key ระดับ production

`rand.Read(b)` เป็นฟังก์ชัน convenience ที่เทียบเท่ากับการเรียก `io.ReadFull(rand.Reader, b)` — ทั้งสองแบบใช้แทนกันได้ ในตัวอย่างเรื่อง AES ด้านบนใช้ `io.ReadFull(rand.Reader, ...)` เพื่อให้เห็นว่า `rand.Reader` ก็คือค่า `io.Reader` ธรรมดาตัวหนึ่ง (เชื่อมโยงกับ interface `io.Reader` ที่เรียนไปในบทที่ 048) ส่วน `rand.Read` เป็นทางลัดที่สะดวกกว่าสำหรับกรณีทั่วไป

---

## 11. TLS/HTTPS: `net/http` จัดการให้อัตโนมัติ

หลังจากเรียนเรื่อง encryption มาทั้งหมด อาจสงสัยว่าแล้วเวลาส่งข้อมูลผ่าน internet (เช่น เรียก REST API ผ่าน `https://`) เราต้องเขียนโค้ดเข้ารหัสเองไหม — **คำตอบคือไม่ต้อง**

เมื่อใช้ `net/http` (ซึ่งเรียนไปแล้วในภาคที่ 4 ตอน HTTP Client/Server) เรียก URL ที่ขึ้นต้นด้วย `https://` **Go จัดการ TLS handshake, การเข้ารหัสข้อมูลระหว่างทาง, และการตรวจสอบใบรับรอง (certificate) ให้ทั้งหมดโดยอัตโนมัติ** ผ่าน package `crypto/tls` ที่ทำงานอยู่เบื้องหลัง

```go
package main

import (
	"fmt"
	"io"
	"net/http"
)

func main() {
	// เรียก https:// ตรงๆ — Go จัดการ TLS ให้ทั้งหมดโดยที่เราไม่ต้องยุ่งกับ crypto/tls เอง
	resp, err := http.Get("https://go.dev")
	if err != nil {
		panic(err)
	}
	defer resp.Body.Close()

	body, _ := io.ReadAll(resp.Body)
	fmt.Println("status:", resp.Status)
	fmt.Println("TLS version negotiated:", resp.TLS.Version)
	fmt.Println("body length:", len(body))
}
```

สิ่งที่เกิดขึ้นเบื้องหลังโดยที่เราไม่ต้องเขียนเอง ได้แก่:

- **TLS handshake**: การตกลง protocol version, cipher suite ระหว่าง client-server
- **การตรวจสอบใบรับรอง**: เช็คว่า certificate ของ server ออกโดย Certificate Authority (CA) ที่เชื่อถือได้ และยังไม่หมดอายุ (ใช้ค่า CA ที่ระบบปฏิบัติการติดตั้งไว้ให้)
- **การเข้ารหัสข้อมูลทั้งหมด**: ทุก byte ที่ส่ง-รับผ่าน connection นี้ถูกเข้ารหัสด้วย symmetric key ที่ตกลงกันระหว่าง handshake (ใช้หลักการเดียวกับ AES-GCM ที่เรียนไปข้างบนนี่เอง เพียงแต่ negotiate key กันแบบอัตโนมัติผ่าน asymmetric cryptography ก่อน)

โดยสรุปแล้ว **ความรู้เรื่อง `crypto/aes`, `crypto/rand`, hashing ในบทนี้ ไม่ได้มีไว้ให้เราสร้าง TLS เอง** (นั่นเป็นงานที่ซับซ้อนมากและควรปล่อยให้ library ที่ผ่านการตรวจสอบอย่างเข้มงวดจัดการ) แต่มีไว้ใช้กับงานระดับ application เช่น การเข้ารหัสข้อมูลก่อนเก็บลง database, การเซ็น payload ของตัวเอง, หรือการสร้าง token — ส่วนการสื่อสารผ่าน network ให้เชื่อใจ `net/http` และ `crypto/tls` ที่ Go เตรียมไว้ให้เสมอ

---

## สรุปสิ่งที่ได้เรียนในบทนี้

- **Hashing** (ทางเดียว) ใช้ตรวจสอบความถูกต้อง/ตัวตน ส่วน **Encryption** (สองทาง) ใช้ปกปิดเนื้อหาแล้วถอดกลับได้
- `crypto/sha256` และ `crypto/md5` มี API คล้ายกัน ใช้ทั้งแบบ one-shot (`Sum256`/`Sum`) และแบบ streaming (`New()` + `Write`)
- MD5/SHA-1 "แตก" ในแง่ collision resistance ทำให้ **ห้ามใช้กับงานความปลอดภัย** แต่ยังใช้ได้กับ checksum/ETag ที่ไม่มีผู้โจมตีตั้งใจปลอมแปลง
- `crypto/hmac` ผสม secret key เข้ากับ hash เพื่อยืนยันทั้งความถูกต้องและตัวตนของผู้ส่ง ต้องเทียบด้วย `hmac.Equal` เท่านั้นเพื่อป้องกัน timing attack
- **ห้าม hash รหัสผ่านด้วย SHA-256 เปล่าๆ** เพราะเร็วเกินไปและไม่มี salt ในตัว ต้องใช้ `bcrypt` (หรือ Argon2/scrypt) ที่ออกแบบมาให้ช้าโดยตั้งใจและมี salt อัตโนมัติ
- `bcrypt.GenerateFromPassword`/`bcrypt.CompareHashAndPassword` จาก `golang.org/x/crypto/bcrypt` คือวิธีมาตรฐานในการจัดการรหัสผ่านใน Go
- `crypto/aes` ร่วมกับโหมด **GCM** ให้ทั้งการเข้ารหัสและตรวจสอบ integrity ในตัวเดียว โดยต้องสุ่ม **nonce ใหม่ทุกครั้ง** ด้วย `crypto/rand` และห้ามใช้ nonce ซ้ำกับ key เดียวกันเด็ดขาด
- `crypto/rand` ให้ความสุ่มที่คาดเดาไม่ได้จาก OS entropy ต่างจาก `math/rand` ที่เป็น pseudo-random คาดเดาได้ถ้ารู้ seed — **สิ่งที่เกี่ยวกับความปลอดภัยต้องใช้ `crypto/rand` เท่านั้น**
- `net/http` จัดการ TLS/HTTPS ให้อัตโนมัติผ่าน `crypto/tls` เบื้องหลัง ไม่ต้องเขียนโค้ดเข้ารหัส network เอง

## แบบฝึกหัดท้ายบท

1. เขียนโปรแกรมที่อ่านไฟล์หนึ่งไฟล์ด้วย `os.Open` แล้วคำนวณ SHA-256 checksum ของไฟล์นั้นโดยใช้ API แบบ streaming (`sha256.New()` + `io.Copy`) แทนการโหลดทั้งไฟล์เข้า memory ก่อน (เชื่อมโยงกับ `io.Reader`/`io.Copy` จากบทที่ 048)
2. เขียนฟังก์ชัน `signPayload(payload []byte, secret []byte) string` ที่คืนค่าลายเซ็น HMAC-SHA256 เป็น hex string แล้วเขียนฟังก์ชัน `verifyPayload(payload []byte, secret []byte, signature string) bool` มาคู่กัน ทดสอบว่าตรวจจับการแก้ไข payload ได้จริง
3. ขยายตัวอย่างระบบ signup/login ในหัวข้อที่ 7 ให้เพิ่มฟีเจอร์ "เปลี่ยนรหัสผ่าน" (`ChangePassword`) ที่ต้องตรวจสอบรหัสผ่านเก่าให้ถูกต้องก่อน แล้วจึง hash รหัสผ่านใหม่ทับของเดิม
4. เขียนโปรแกรมเข้ารหัส/ถอดรหัสไฟล์ข้อความด้วย AES-GCM โดยรับ key จาก argument บรรทัดคำสั่ง (แปลงจาก hex string เป็น `[]byte` ด้วย `encoding/hex`) แล้วอ่าน/เขียนไฟล์จริงด้วย `os.ReadFile`/`os.WriteFile`
5. ลองแก้ไขตัวอย่าง AES-GCM ให้ใช้ nonce คงที่ (hardcoded) แทนการสุ่มใหม่ทุกครั้ง แล้วเข้ารหัสข้อความสองข้อความที่ต่างกันด้วย key และ nonce เดียวกัน ค้นคว้าเพิ่มเติมว่าทำไมนี่ถึงเป็นความผิดพลาดร้ายแรง (คำใบ้: ค้นคำว่า "AES-GCM nonce reuse attack")
6. ค้นคว้าเพิ่มเติมเกี่ยวกับ **Argon2** ผ่าน package `golang.org/x/crypto/argon2` เปรียบเทียบกับ bcrypt ว่าต่างกันอย่างไร และทำไมหลายบริษัทเลือกเปลี่ยนมาใช้ Argon2 สำหรับระบบใหม่ในปัจจุบัน

---

**ต่อไป**: [Part 052 — `os/exec` รัน External Command](./052-os-exec.md)
