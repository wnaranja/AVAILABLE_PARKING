# Parking Display

หน้าจอแสดง **จำนวนที่จอดรถคงเหลือ** สำหรับขึ้นจอทีวีที่ลานจอดรถ มี 2 หน้า แต่ละหน้าเป็นไฟล์ HTML เดียว (HTML + CSS + JS อยู่ในไฟล์เดียวกัน ไม่ใช้ framework ไม่โหลดอะไรจากอินเทอร์เน็ต)

| ไฟล์ | แสดงอะไร | หัวข้อบนจอ |
|---|---|---|
| `parking_available.html` | ที่จอดรถคงเหลือ (รถยนต์เท่านั้น) | 剩余车位数量 / Available Parking Spaces |
| `member_available.html` | ช่องจอด member คงเหลือ | 会员剩余车位数量 / Available Member Spaces |

ตอนนี้ข้อมูลเป็น **mock คงที่** (ช่องจอดทั้งหมด 500) หน้าที่ของคนเขียน JS คือเปลี่ยนไปดึงข้อมูลจริงจาก backend

---

## รันดูในเครื่อง

ไม่ต้องติดตั้งอะไร ดับเบิลคลิกเปิดไฟล์ในเบราว์เซอร์ได้เลย (กด F11 เพื่อดูเต็มจอ)

ถ้าต้องการเปิดผ่าน web server:

```bash
python -m http.server 5500
# เปิด http://localhost:5500/parking_available.html
# เปิด http://localhost:5500/member_available.html
```

> ถ้าตั้ง `MOCK: false` แล้วต้องเรียก API ให้เปิดผ่าน server (ไม่ใช่ `file://`) เพราะเบราว์เซอร์มักบล็อก `fetch` จากหน้า `file://`

---

## สูตรคำนวณ

คำนวณฝั่งหน้าเว็บ (ในฟังก์ชัน `refresh`)

| หน้า | สูตร |
|---|---|
| `parking_available.html` | `total_slots - parked_cars` |
| `member_available.html` | `total_slots - parked_members` |

- `total_slots` คือจำนวนช่องจอดทั้งหมด
- `parked_cars` คือจำนวนรายการที่จอดอยู่ในระบบ **นับเฉพาะรถยนต์**
- `parked_members` คือจำนวน member ที่จอดอยู่ในระบบ

---

## สิ่งที่ต้องทำเพื่อเชื่อม backend

แก้ใน `<script>` ของแต่ละไฟล์ มี 3 จุด

### 1) ปิด mock

```js
const CONFIG = {
  MOCK: false,        // เปลี่ยนจาก true เป็น false
  POLL_MS: 3000,      // ความถี่ในการดึงข้อมูล (ms)
  LOW_THRESHOLD: 10   // ต่ำกว่าหรือเท่ากับค่านี้ ตัวเลขจะเป็นสีเหลือง
};
```

### 2) แก้ endpoint ใน `fetchParkingStats()`

```js
async function fetchParkingStats() {
  if (CONFIG.MOCK) return { ...MOCK_DATA };

  const res = await fetch('/api/parking/stats', { cache: 'no-store' }); // TODO: ใส่ endpoint จริง
  if (!res.ok) throw new Error('HTTP ' + res.status);
  return res.json();
}
```

ฟังก์ชันนี้ต้อง resolve เป็น object หน้าตาแบบนี้

**หน้า `parking_available.html`**

```json
{ "total_slots": 500, "parked_cars": 380 }
```

**หน้า `member_available.html`**

```json
{ "total_slots": 500, "parked_members": 150 }
```

ถ้า backend ทำ endpoint เดียวที่ส่งมาทั้ง 3 ค่า ใช้ร่วมกันได้ทั้งสองหน้า (field ที่ไม่ได้ใช้จะถูกข้ามไป)

```json
{ "total_slots": 500, "parked_cars": 380, "parked_members": 150 }
```

### 3) ลบ mock ทิ้ง (ไม่บังคับ)

ลบก้อน `MOCK_DATA` ได้เมื่อเชื่อมจริงเสร็จ

ในโค้ดมีคอมเมนต์ `[BACKEND-1]` และ `[BACKEND-2]` ติดไว้ตรงจุดที่ต้องใส่ค่า (`total_slots` และ `parked_cars` / `parked_members`)

---

## พฤติกรรมของหน้าจอ

| สถานะ | เงื่อนไข | แสดงผล |
|---|---|---|
| `ok` | คงเหลือ > `LOW_THRESHOLD` | ตัวเลขสีเขียว |
| `low` | คงเหลือ ≤ `LOW_THRESHOLD` | ตัวเลขสีเหลือง |
| `full` | คงเหลือ = 0 (หรือติดลบ) | `已满 FULL` สีแดง |
| `error` | `fetchParkingStats()` throw | `--` สีเทา แล้วลองใหม่รอบถัดไป |

- ค่าติดลบจะถูกปัดเป็น 0 เสมอ (`Math.max(0, remaining)`)
- ดึงข้อมูลใหม่ทุก `POLL_MS` ด้วย `setInterval` โดยอัปเดตตัวเลขในหน้าเดิม **ไม่ได้รีโหลดทั้งหน้า** จึงไม่กะพริบ
- สถานะแสดงผลถูกกำหนดผ่าน attribute `data-state` ของ element ตัวเลข สีกำหนดใน CSS (`.number[data-state="..."]`)

---

## โครงสร้างโค้ดในแต่ละไฟล์

```
<style>    CSS (ใช้หน่วย vw/vh ทั้งหมด ปรับตามขนาดจออัตโนมัติ)
<body>     .sign > .label-cn, .label-en, .number
<script>
  CONFIG             ตั้งค่า
  MOCK_DATA          ข้อมูล mock (ลบได้)
  fetchParkingStats  จุดเชื่อม backend
  render(remaining)  ใส่ตัวเลข + สถานะสีลง DOM
  refresh()          ดึงข้อมูล -> คำนวณ -> render (จับ error แล้วแสดง "--")
  setInterval        เรียก refresh ทุก POLL_MS
```

element ที่ JS อ้างถึง

| หน้า | id ของตัวเลข |
|---|---|
| `parking_available.html` | `available_slots` |
| `member_available.html` | `available_member_slots` |

---

## หมายเหตุสำหรับการใช้งานจริง

- **CORS**: ถ้า API อยู่คนละ origin กับที่เสิร์ฟหน้านี้ ต้องเปิด CORS ที่ฝั่ง backend หรือเสิร์ฟหน้านี้จาก origin เดียวกับ API (เช่น วางในโฟลเดอร์ `static/` ของเซิร์ฟเวอร์เดียวกัน แล้วเรียก `/api/...`)
- **อย่าฝัง key หรือ token ในไฟล์ HTML** ใครเปิด View Source ก็เห็นได้
- **เปิดบนทีวีผ่าน mini pc**: เปิดด้วย Chrome แบบ kiosk
  ```bash
  chrome.exe --kiosk "http://127.0.0.1:8000/static/parking_available.html"
  ```
  ตั้ง Windows scaling เป็น 100% และปิด overscan ในเมนูทีวี (Just Scan / Screen Fit) เพื่อไม่ให้ขอบจอถูกตัด
- **ภาษา/ฟอนต์**: ใช้ฟอนต์ระบบ (Microsoft YaHei / Noto Sans SC / PingFang SC) เพื่อให้ทำงานได้แม้ออฟไลน์ ถ้าเครื่องไม่มีฟอนต์จีน ตัวอักษรจีนอาจแสดงผิด ให้ติดตั้ง Noto Sans SC
- เบราว์เซอร์ที่ต้องการ: รุ่นใหม่ที่รองรับ CSS `min()` และ `fetch` (Chrome/Edge ปัจจุบันใช้ได้) Smart TV รุ่นเก่าอาจไม่รองรับ แนะนำให้ใช้ mini pc ตามที่วางแผนไว้
