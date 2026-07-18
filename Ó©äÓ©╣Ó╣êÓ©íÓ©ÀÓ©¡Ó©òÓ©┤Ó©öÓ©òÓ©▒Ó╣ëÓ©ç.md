# คู่มือติดตั้ง RSU Asset Control (Cloud) — ตั้งแต่ศูนย์จนใช้งานจริง

ระบบประกอบด้วย 3 ส่วน และติดตั้งตามลำดับนี้:

| ส่วน | ทำหน้าที่ | จำเป็นไหม |
|---|---|---|
| A. Firebase (Firestore + Auth) | ฐานข้อมูลกลาง ทุกเครื่องเห็นข้อมูลเดียวกันเรียลไทม์ + ใช้ออฟไลน์ได้ | จำเป็น |
| B. GitHub Pages | โฮสต์หน้าแอป เปิดจากมือถือเครื่องไหนก็ได้ | จำเป็น |
| C. Cloud Function (aiCount) | ตัวกลางเรียก AI นับภาพ ซ่อน API key ฝั่งเซิร์ฟเวอร์ | ไม่บังคับ — ข้ามได้ แอปยังใช้ได้ครบยกเว้นปุ่มนับ AI |

**สิ่งที่ต้องมีก่อนเริ่ม:** บัญชี Google, บัญชี GitHub, และถ้าจะทำส่วน C ต้องมี Node.js 18+ บนคอม, บัญชี Anthropic API (console.anthropic.com), และอัปเกรด Firebase เป็นแผน Blaze (ผูกบัตร — มี free quota สูง งานสเกลนี้แทบไม่เสียเงินส่วน Firebase, เสียจริงเฉพาะค่า AI ต่อรูป)

---

## ส่วน A: สร้างฐานข้อมูล Firebase (~15 นาที)

### A1. สร้างโปรเจกต์
1. ไป **console.firebase.google.com** → **Add project**
2. ตั้งชื่อ เช่น `rsu-asset-control` → ปิด Google Analytics ก็ได้ → **Create**

### A2. เปิด Firestore
1. เมนูซ้าย **Build → Firestore Database** → **Create database**
2. Location เลือก **asia-southeast1 (Singapore)** — *เลือกแล้วเปลี่ยนไม่ได้ อย่าพลาดข้อนี้*
3. เลือก **Start in production mode** → Create

### A3. วาง Security Rules
1. ในหน้า Firestore → แท็บ **Rules**
2. ลบของเดิมทั้งหมด → คัดลอกเนื้อหาไฟล์ `firestore.rules` ไปวาง → **Publish**

> Rules ชุดนี้กำหนดว่า: ต้องล็อกอินจึงใช้ได้, รายการเคลื่อนไหว (tx) **แก้ไขย้อนหลังไม่ได้** (กันแอบแก้ยอด), เคส Incident แก้ได้เฉพาะสถานะเปิด/ปิด

### A4. เปิดระบบล็อกอิน
1. **Build → Authentication** → **Get started**
2. แท็บ **Sign-in method** → เลือก **Anonymous** → Enable → Save

### A5. ลงทะเบียนเว็บแอป + เอา config
1. หน้า Project Overview → กดไอคอน **`</>`** (Web)
2. ตั้งชื่อ app เช่น `rsu-web` → **Register** (ไม่ต้องติ๊ก Hosting)
3. จะได้ก้อน `firebaseConfig = { apiKey: "...", ... }` — **คัดลอกเก็บไว้**
4. เปิดไฟล์ `index.html` ด้วย Notepad/VS Code → แทนที่บล็อก `firebaseConfig` ตรงหัวไฟล์ (มีคอมเมนต์ ⚙️ CONFIG กำกับ) ด้วยของจริง → Save

---

## ส่วน B: โฮสต์แอปบน GitHub Pages (~10 นาที)

1. ไป **github.com** → **New repository** → ตั้งชื่อ เช่น `rsu-asset` → **Private ไม่ได้** (Pages ฟรีต้องเป็น Public — ไม่เป็นไร เพราะในไฟล์ไม่มี secret ใด ๆ, `firebaseConfig` เป็นข้อมูลสาธารณะโดยออกแบบ ความปลอดภัยอยู่ที่ Rules)
2. **Add file → Upload files** → ลาก `index.html` (ที่ใส่ config แล้ว) → **Commit**
3. **Settings → Pages** → Source: **Deploy from a branch** → Branch: `main` / root → **Save**
4. รอ 1–2 นาที จะได้ URL: `https://<ชื่อคุณ>.github.io/rsu-asset/`
5. เปิด URL บนมือถือ → ควรเห็นหน้า "ใส่ชื่อผู้ปฏิบัติงาน" และสถานะ "เชื่อมต่อสำเร็จ"

**ทดสอบ multi-user ทันที:** เปิด URL บนมือถือ 2 เครื่อง → เครื่องแรกเพิ่มคนขับ → เครื่องที่สองต้องเห็นภายใน 1–2 วินาที ถ้าเห็น = ระบบกลางทำงานแล้ว

**ติดตั้งเป็นแอปบนมือถือ:** Chrome/Safari → เมนู → **Add to Home Screen** — ได้ไอคอนเหมือนแอปจริง

---

## ส่วน C: เปิดใช้ AI นับภาพ (~30 นาที, ข้ามได้)

### C1. เตรียม
1. อัปเกรด Firebase เป็น **Blaze**: Console → ไอคอนเฟือง → Usage and billing → Modify plan
2. สร้าง API key ที่ **console.anthropic.com** → API Keys → Create Key → คัดลอกเก็บ (ขึ้นต้น `sk-ant-...`)
3. ลง Firebase CLI บนคอม: เปิด Terminal/CMD พิมพ์
   ```
   npm install -g firebase-tools
   firebase login
   ```

### C2. ตั้งโปรเจกต์ Functions
สร้างโฟลเดอร์ว่าง เช่น `rsu-functions` แล้วใน Terminal:
```
cd rsu-functions
firebase init functions
```
- เลือก **Use an existing project** → เลือกโปรเจกต์ที่สร้างไว้
- Language: **JavaScript** → ESLint: No → Install dependencies: Yes

จากนั้น **แทนที่** ไฟล์ `functions/index.js` และ `functions/package.json` ด้วยไฟล์ที่ให้มา แล้วรัน:
```
cd functions
npm install
cd ..
```

### C3. เก็บ API key เป็น Secret (ห้ามใส่ key ในโค้ดเด็ดขาด)
```
firebase functions:secrets:set ANTHROPIC_API_KEY
```
ระบบจะถามค่า → วาง `sk-ant-...` → Enter

### C4. Deploy
```
firebase deploy --only functions
```
เสร็จแล้วจะโชว์ URL ประมาณ:
`https://aicount-xxxxxxxx-as.a.run.app`
**คัดลอก URL นี้** → เปิด `index.html` → ใส่ในบรรทัด `const AI_ENDPOINT = "..."` → อัปโหลดไฟล์ทับใน GitHub อีกครั้ง

### C5. ล็อกโดเมน (ทำหลังทดสอบผ่าน)
ใน `functions/index.js` เปลี่ยน `cors: true` เป็น:
```js
cors: ["https://<ชื่อคุณ>.github.io"]
```
แล้ว deploy ซ้ำ — กันคนนอกยิง endpoint คุณไปใช้ AI ฟรีด้วยเงินคุณ

---

## Checklist ทดสอบก่อนใช้จริง

| # | ทดสอบ | ผ่านเมื่อ |
|---|---|---|
| 1 | เปิด URL บน 2 เครื่อง เพิ่มคนขับเครื่องเดียว | อีกเครื่องเห็นใน 1–2 วิ |
| 2 | เปิดโหมดเครื่องบิน → บันทึกรายการ → เปิดเน็ตกลับ | รายการโผล่บนเครื่องอื่นเองหลังเน็ตกลับ |
| 3 | จ่ายออกให้คนขับ A แล้วรอเกินวันที่ตั้ง (หรือตั้ง overdue = 0 ชั่วคราว) | ขึ้นป้าย 🔒 ล็อก |
| 4 | ถ่ายรูปพาเลทจริง | AI เติมตัวเลข + บอก confidence |
| 5 | กด Export | ได้ Excel 3 ชีต เปิดอ่านภาษาไทยถูกต้อง |
| 6 | ลองแก้เอกสารใน collection `tx` ผ่าน Console ด้วยบัญชีแอป | Rules ปฏิเสธ (update: false) |

---

## เรื่องที่ต้องรู้ตรง ๆ (อ่านก่อนประกาศใช้)

1. **Anonymous Auth = ใครมี URL ก็เขียนข้อมูลได้** เพียงพอสำหรับ pilot ภายในถ้าไม่แชร์ลิงก์ออกนอกทีม แต่ถ้าจะใช้ยาว ให้อัปเกรดเป็น Email/Password ต่อกะ (เพิ่มใน Authentication แล้วแก้โค้ด login ~20 บรรทัด) หรือเปิด **App Check** เสริม
2. **ค่าใช้จ่ายจริง:** Firestore ระดับ 1 ไซต์อยู่ใน free quota เกือบแน่นอน ที่เสียคือ AI — ประมาณ 0.3–1 บาท/รูป ถ่ายวันละ 100 รูปก็หลักร้อยบาท/เดือน ตั้ง `maxInstances: 3` กันบิลบานไว้แล้ว
3. **อย่า commit ไฟล์ที่มี API key ลง GitHub เด็ดขาด** — key อยู่ใน Secret Manager ฝั่ง Firebase เท่านั้น ในโค้ดที่ให้มาไม่มีจุดไหนต้องใส่ key ในไฟล์เลย ถ้าเผลอทำหลุด ให้ revoke key ทันทีที่ console.anthropic.com
4. **การลบคนขับลบประวัติ tx ของเขาด้วย** — ใช้ให้ถูก: ลบเฉพาะกรณีเพิ่มผิด ถ้าคนขับลาออกแต่มีประวัติ ให้เก็บไว้ อย่าลบ (ประวัติคือหลักฐาน)
5. **จำกัด listener ที่ 2,000 รายการล่าสุด** — เกินหลักหมื่นรายการค่อยคุยเรื่อง pagination/summary docs ระดับ pilot ไม่ต้องสน

## ลำดับ Go-Live ที่แนะนำ

วันที่ 1: ติดตั้ง A+B → ทดสอบ 2 เครื่อง → ลงทะเบียนคนขับนำร่อง 5–10 คน
วันที่ 2–3: ซ้อมกับทีม 1 กะ (ยังไม่ประกาศกติกาล็อก)
วันที่ 4: ติดตั้งส่วน C (AI) → ซ้อมถ่ายรูปที่จุดจ่ายจริง หามุม/แสงที่ AI นับแม่น แล้วตีเส้นจุดถ่ายถาวร
สัปดาห์ที่ 2: ผู้บริหารประกาศกติกา Wallet + ล็อก → เริ่มนับจริง
สิ้นเดือน: Export Excel เทียบกับ RAMS → ตัวเลขชุดนี้คือกระสุนขอไฟเขียวขยายทั้งไซต์
