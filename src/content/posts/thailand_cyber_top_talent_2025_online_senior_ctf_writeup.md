---
author: Kings (Hex / K1GOD)
description: Write-up ฉบับมือใหม่ ปีแรก
draft: false
published: 2025-09-01
series: CTF
tags:
- ctf
title: Thailand Cyber Top Talent 2025 (Online-Senior) CTF Write-up
---

![thctt25](https://img2.pic.in.th/pic/thctt-2025-post.jpg)

ปีแรกที่พวกผมได้เข้าร่วมการแข่งขันด้านความมั่นคงปลอดภัยไซเบอร์ ที่ใหญ่ที่สุดในไทย 
- ครั้งแรกนี้ได้ถือกำเนิดทีม Lincox
- write-up ข้อที่ทำได้ในรอบ Online-Senior 2025\
(บันทึกสไตล์มือใหม่ ลองผิดลองถูก ปีแรก ✌️)

------------------------------------------------------------------------

## Network  ข้อที่ 2: Whispers in the Wire (200)
![Whispers in the Wire (200)](https://img2.pic.in.th/pic/Screenshot-2025-08-30-140723.png)
โจทย์ให้ไฟล์มาสองตัว

-   `thctt2025_open_netsec3_whisper-in-the-wire.zip`
-   (แตกออกมา มีไฟล์น่าสงสัยคือ thctt2025_open_netsec3_whisper-in-the-wire.pcapng)


------------------------------------------------------------------------

### 🔍 Step 1 --- เปิดไฟล์เช็ค

-   เปิด `.pcapng` ใน Wireshark\
-   กรองดู traffic ก็ไม่เจอ flag ตรง ๆ\
-   เริ่มคิดว่า data ข้างในน่าจะเป็น **compressed stream**
-   เลยลองโยนไปถาม LLM มันบอกว่าอาจจะอยู่ใน zlib

------------------------------------------------------------------------

### 📦 Step 2 --- ทำไมต้องดู zlib?

-   gRPC/HTTP2 ชอบบีบด้วย zlib\
-   zlib มี header ขึ้นต้นชัดเจน เช่น `0x78 0x9C`\
-   เลยลองหาว่ามีส่วนนั้นซ่อนอยู่ในไฟล์

------------------------------------------------------------------------

### 🛠️ Step 3 --- เขียนสคริปต์มาแกะสะหน่อย

ใช้ LLM เขียน python ง่ายๆสำหรับแกะออกมา

``` python
import zlib

with open("thctt2025_open_netsec3_whisper-in-the-wire.pcapng", "rb") as f:
    data = f.read()

count = 0
for header in [b"\x78\x01", b"\x78\x5e", b"\x78\x9c", b"\x78\xda"]:
    start = 0
    while True:
        idx = data.find(header, start)
        if idx == -1:
            break
        try:
            out = zlib.decompress(data[idx:])
            text = out.decode(errors="ignore")
            print(f"\n=== Stream {count} at offset {idx} ===")
            print(text)
            count += 1
        except Exception:
            pass
        start = idx + 1

if count == 0:
    print("❌ ไม่เจอ stream ที่แตกได้")

```

------------------------------------------------------------------------

### 🎯 Step 4 --- เจอ flag

พอแตกออกมาก็เจอเลยว่า

    flag{fd3ca5a5acce56f112a725ce433c96e4}

------------------------------------------------------------------------

------------------------------------------------------------------------

## ข้อที่ 2: \<ชื่อโจทย์\>

(ว่างไว้ก่อน รอเติม)


------------------------------------------------------------------------

## Network ข้อที่ 3: Custom Protocol v2 (300)
![Custom Protocol v2 (300)](https://img2.pic.in.th/pic/Screenshot-2025-08-30-140802-1.png)


> ไฟล์ที่ให้: `thctt2025_open_netsec4_custom-protocol-v2.pcap` (+ มีสเปคโปรโตคอล v2 แนบ)\
> รอไร อ่านไฟล์สเปคเหมือนเดิม จากนั้นก็อ๋อ อ่านไม่ออก555 แต่จับจุดได\
> เป้าหมาย: ถอดข้อความจากโปรโตคอล **STH v2** แล้วอ่าน flag 

---

### 🔎 Step 1 — เปิดเลยยยยย  Wireshark

1. เปิดไฟล์ `.pcap` ใน **Wireshark**  
2. **filter**:
   ```
   udp && data[0:3]==53:54:48 && data[3]==02
   ```
   - `53:54:48` = ASCII `"STH"`
   - `data[3]==02` = เวอร์ชัน 0x02  
3. จะเห็นลำดับแพ็กเก็ตประมาณ: **HELLO (type=0x01)** → **WELCOME (0x02)** → **DATA (0x10)** หลายเฟรม → **BYE (0x20)**


---

### 🧩 Step 2 — คลิกดูค่าใน WELCOME (ตามในไฟล์สเปค)

1. คลิกเฟรมที่เป็น **WELCOME** (`data[4]==02`)  
2. เฮดเดอร์ STH v2 ยาว 23 ไบต์ แล้วต่อด้วย **payload** ของ WELCOME  
3. ลากเลือกช่วงไบต์ใน **Packet Bytes**
4. โครงสร้าง **WELCOME payload**:
   ```
   server_nonce[8] | salt[8] | chunk_size[2] | total_chunks[2] |
   a[4] | c[4] | seed[4] | hint[2] | flags2[1] | reserved[1]
   ```
5. โครงสร้าง **HELLO payload**: `client_nonce[8] | name_len | name[...]`

**ค่าที่ได้สำหรับไฟล์นี้ :**
- `session_id` (จากเฮดเดอร์ WELCOME): `0xCB795043`
- `client_nonce`: `12 69 52 3B CC 32 C9 03`
- `server_nonce`: `02 41 EF A8 B2 36 89 9E`
- `salt`: `8A 0A 13 ED BD D1 2D D7`
- `chunk_size = 70`, `total_chunks = 5`
- LCG: `a = 261603731`, `c = 1`, `seed = 1976845399`
- `flags2 = 0x03` ⇒ มี **Base64 + Reverse**

---

### 🗃️ Step 3 — ดู DATA แล้ว “เรียงชิ้น” ตามไฟล์สเปคที่มีให้ 

- เฟรม **DATA** แต่ละเฟรมมี `seq` (ดูในเฮดเดอร์ STH v2 ตำแหน่งไบต์ 11–14, big-endian) ซึ่งเป็น index **หลัง** ถูกสับ  
- ใช้พารามิเตอร์จาก WELCOME ทำลำดับ **LCG**:
  ```text
  n = total_chunks
  x0 = seed mod n
  x_{k+1} = (a*x_k + c) mod n
  order = [x0, x1, ..., x_{n-1}]
  ```
- ของไฟล์นี้ได้ `order = [4, 0, 1, 2, 3]` -- 

---

### 🧪 Step 4 — อ่านสเปคที่มีให้เหมือนเดิมละทำตาม (ส่วนนี้ผมโยนใน LLM ทำเลยละกัน555)

1. **Base64-decode**   
2. **Reverse** (กลับลำดับไบต์) — ตาม `flags2`  
3. **XOR** กับคีย์สตรีมที่ได้จากการต่อบล็อก `SHA256`:
   ```text
   SHA256( session_id || client_nonce || server_nonce || salt || counter_be32 )
   ```
   สร้างทีละบล็อกด้วย `counter = 0,1,2,...` แล้วต่อจนยาวพอ  
4. จะได้ zlib stream → **zlib.decompress()** → ได้ plaintext

---

### 🎯 งานนี้จบที่ LLM 😂

```
flag{48e64c539d58ac64f574b3c9abd9b6b1}
```

---


------------------------------------------------------------------------
