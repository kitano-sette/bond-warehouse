# SEC Investor Dashboard

Mobile-first web app สำหรับค้นหาบริษัทไทยแล้วดูฐานะการเงิน + Financial Ratio + หุ้นกู้ +
Credit Rating ในหน้าเดียว ข้อมูลทั้งหมดดึงจาก **SEC Open Data API** (https://api.sec.or.th)
ผ่าน Backend Proxy ของตัวเอง (ไม่มีการเรียก SEC API ตรงจาก browser และไม่มี API Key ฝั่ง frontend เลย)

```
Browser (React)
   ↓
Supabase Edge Function "sec-proxy"   ← เก็บ SEC_API_KEY เป็น secret ที่นี่เท่านั้น
   ↓ (cache-first)
Postgres cache tables (Supabase)
   ↓ (cache miss เท่านั้น)
SEC Open API (https://api.sec.or.th)
```

---

## 1. โครงสร้างโปรเจกต์

```
sec-investor-dashboard/
├── src/
│   ├── api/            secApi.js, bondApi.js, financialApi.js — เรียก backend proxy เท่านั้น
│   ├── components/      CompanySearch, CompanyOverview, FinancialTable, RatioTable,
│   │                    BondTable, RatingBadge, RiskCard, BottomNav
│   ├── mapping/          financialMapping.js, ratioMapping.js — set_code → ชื่อไทย
│   ├── utils/            formatter.js, normalize.js, cache.js, errorMessages.js,
│   │                    risk.js, checklist.js
│   ├── pages/            Home.jsx (ค้นหา + Overview/Financial/Bond/Risk),
│   │                    BondDetail.jsx (คลิกการ์ดหุ้นกู้ → หน้านี้)
│   ├── styles/           tokens.css (design tokens), layout.css
│   ├── App.jsx           React Router (HashRouter)
│   └── main.jsx
├── supabase/
│   ├── functions/sec-proxy/   Backend Proxy (Deno / Supabase Edge Function)
│   └── migrations/            schema ตาราง cache ใน Postgres
├── index.html
├── package.json
├── vite.config.js
└── .env.example
```

## 2. วิธีขอ SEC API Key

1. สมัครและยื่นคำขอใช้งาน SEC Open API ที่ SEC Open Data Portal ของสำนักงาน ก.ล.ต.
2. เมื่อได้รับอนุมัติจะได้ Subscription Key มาใช้เป็น header
   `Ocp-Apim-Subscription-Key`.
3. **ห้าม** ใส่ Key นี้ในโค้ด frontend หรือ commit ลง git โดยเด็ดขาด — ใส่เป็น
   Supabase secret เท่านั้น (ดูขั้นตอนด้านล่าง)

## 3. ติดตั้งและรัน Local

### 3.1 ติดตั้ง dependencies (frontend)
```bash
npm install
```

### 3.2 ตั้งค่า Supabase (backend proxy + cache)
ต้องมี [Supabase CLI](https://supabase.com/docs/guides/cli) และโปรเจกต์ Supabase อยู่แล้ว

```bash
# ล็อกอินและผูกกับโปรเจกต์ Supabase ของคุณ
supabase login
supabase link --project-ref YOUR_PROJECT_REF

# รัน migration เพื่อสร้างตาราง cache
supabase db push

# ตั้งค่า SEC API Key เป็น secret (ฝั่งเซิร์ฟเวอร์เท่านั้น)
supabase secrets set SEC_API_KEY=xxxxxxxxxxxxxxxx

# deploy backend proxy
supabase functions deploy sec-proxy --no-verify-jwt
```

### 3.3 ตั้งค่าไฟล์ .env ของ frontend
```bash
cp .env.example .env
# แก้ VITE_SUPABASE_URL และ VITE_SUPABASE_ANON_KEY ให้ตรงกับโปรเจกต์ Supabase ของคุณ
```

### 3.4 รัน dev server
```bash
npm run dev
```
เปิด http://localhost:5173 — แอปจะเปิดมาพร้อมข้อมูลตัวอย่าง **KTB** (ธนาคารกรุงไทย) แล้วพิมพ์
ในช่องค้นหาเพื่อสลับไปบริษัทอื่น เช่น PTT, KBANK, AOT, CPALL

## 4. Deploy เป็น Static Web App

Frontend เป็น static build ล้วน (ไม่มี backend ของตัวเอง — ใช้ Supabase Edge Function
เป็น backend) จึง deploy ได้กับทุกบริการ static hosting เช่น Netlify, Vercel, Cloudflare Pages:

```bash
npm run build      # ได้ไฟล์ static อยู่ใน dist/
```
อัปโหลดโฟลเดอร์ `dist/` ขึ้น static host ที่ต้องการ (ตั้งค่า environment variables
`VITE_SUPABASE_URL` / `VITE_SUPABASE_ANON_KEY` บนแพลตฟอร์มนั้นๆ ก่อน build ด้วย)

ใช้ `HashRouter` ในแอปอยู่แล้ว จึงไม่ต้องตั้งค่า server-side rewrite สำหรับ
client-side routing (URL จะเป็นรูปแบบ `/#/bond/xxxx`)

## 5. Feature ตาม spec ที่ทำแล้ว

| Feature | สถานะ | อยู่ที่ไฟล์ |
|---|---|---|
| ค้นหาบริษัทจาก Symbol/ชื่อไทย/ชื่ออังกฤษ + autocomplete | ✅ | `components/CompanySearch.jsx` |
| Company Overview + KPI cards | ✅ (บางส่วน ดูข้อ 7) | `components/CompanyOverview.jsx` |
| งบการเงินย้อนหลัง 3 ปี + mapping set_code | ✅ | `components/FinancialTable.jsx`, `mapping/financialMapping.js` |
| Financial Ratio + กราฟย้อนหลัง | ✅ | `components/RatioTable.jsx`, `mapping/ratioMapping.js` |
| หุ้นกู้: features/credit-ratings/outstanding/involve-parties | ✅ | `api/bondApi.js` |
| คลิกการ์ดหุ้นกู้ → หน้ารายละเอียด | ✅ | `pages/BondDetail.jsx` |
| Investor Risk Dashboard (Low/Medium/High เฉพาะที่มีสูตรชัดเจน) | ✅ | `utils/risk.js` |
| Bond Investment Checklist พร้อมลิงก์ "ดูข้อมูล" | ✅ (7/11 ข้อ ดูข้อ 7) | `utils/checklist.js` |
| Bottom Navigation: Overview/Financial/Bond/Risk | ✅ | `components/BottomNav.jsx` |
| Pagination (`fetchAllPages`, next_cursor) | ✅ | `supabase/functions/sec-proxy/secClient.ts` |
| Cache 2 ชั้น: Postgres (ทุกผู้ใช้ร่วมกัน) + localStorage (ต่อเบราว์เซอร์) TTL 6 ชม. | ✅ | `cacheClient.ts`, `utils/cache.js` |
| Error handling 401/403/404/429/500 เป็นภาษาไทย | ✅ | `utils/errorMessages.js` |
| Backend Proxy ซ่อน API Key | ✅ | `supabase/functions/sec-proxy/` |
| Responsive / Mobile-first | ✅ | `styles/tokens.css` (`--max-width: 480px`, bottom nav) |

## 6. ข้อจำกัดที่ควรทราบก่อนใช้งานจริง

1. **ยังไม่เคยทดสอบกับ SEC API จริง** — โค้ดนี้เขียนตามสเปกที่ให้มาเท่านั้น ยังไม่มีตัวอย่าง
   raw response จริงมาตรวจสอบ โครงสร้าง field ภายใน `balanceSheet` / `incomeStatement` /
   `cashFlow` / `financialRatio` (สมมติว่าเป็น array ของ `{set_code, amount, unit}`) **ต้อง
   ตรวจสอบกับ response จริงก่อน deploy ใช้งานจริง** แก้ได้ที่ `FinancialTable.jsx` /
   `RatioTable.jsx` เท่านั้น ไม่กระทบส่วนอื่น
2. **mapping ยังไม่ครบ** — `financialMapping.js` ยืนยันไว้แค่ 4 set_code (สินทรัพย์รวม,
   หนี้สินและส่วนของผู้ถือหุ้นรวม, กำไรก่อนภาษี, กำไรสุทธิ) ตามที่ spec ให้มา ส่วน
   `ratioMapping.js` ยังว่างเปล่า (spec ให้แค่ตัวอย่าง code โดยไม่มีความหมายกำกับ) ทำให้:
   - KPI cards หน้า Overview ยังไม่มี หนี้สินรวม/ส่วนของผู้ถือหุ้น/กระแสเงินสด/ROE/ROA แยกต่างหาก
     (309998 คือสินทรัพย์รวม, 319999 คือ "หนี้สินและส่วนของผู้ถือหุ้นรวม" ซึ่งเท่ากับสินทรัพย์รวม
     ตามหลักบัญชี ไม่ใช่หนี้สินรวมเพียงอย่างเดียว — จึงยังนำมาแยกแสดงเป็น "หนี้สินรวม" ไม่ได้)
   - Risk Dashboard เปิดใช้เฉพาะ **Credit Risk** (จาก credit rating) และ **Bond Risk**
     (จากโครงสร้างตราสาร: มีหลักประกัน/ด้อยสิทธิหรือไม่) ส่วน Liquidity/Leverage/
     Profitability/Cash Flow แสดงเป็น "ไม่ระบุ" เพราะยังไม่มีสูตรที่มีข้อมูลรองรับชัดเจน
   - Checklist มี 7 จาก 11 ข้อของ spec (ข้อที่เหลือต้องใช้ set_code ที่ยังไม่ยืนยัน)

   **แก้ไขได้โดยเพิ่ม entry ใน `financialMapping.js` / `ratioMapping.js` เมื่อยืนยัน
   ความหมาย set_code เพิ่มแล้ว** — ไม่ต้องแก้โค้ด UI ใดๆ เพราะ component ทุกตัวอ่านจาก
   mapping layer อัตโนมัติ

3. **`getBondFeaturesByCompany`** ใน `bondApi.js` เรียก credit-rating และ outstanding-value
   ของหุ้นกู้ทุกตัวแบบขนาน (`Promise.all`) — ถ้าบริษัทหนึ่งมีหุ้นกู้จำนวนมากอาจต้องเพิ่ม
   throttling/batch เพื่อไม่ชน rate limit (429) ของ SEC API

## 7. Error handling

| HTTP Status | ข้อความที่แสดง |
|---|---|
| 401 | API Key ไม่ถูกต้อง |
| 403 | ไม่มีสิทธิ์เข้าถึง API |
| 404 | ไม่พบข้อมูลที่ค้นหา |
| 429 | เรียก API มากเกินไป กรุณารอสักครู่ |
| 500 / อื่นๆ | เกิดข้อผิดพลาดของระบบ กรุณาลองใหม่อีกครั้ง |

## 8. Cache

- **ชั้นที่ 1 (ใช้ร่วมกันทุกผู้ใช้):** ตาราง Postgres ใน Supabase (`supabase/migrations/0001_init_sec_cache.sql`)
  TTL ค่าเริ่มต้น 6 ชั่วโมง
- **ชั้นที่ 2 (ต่อเบราว์เซอร์ผู้ใช้):** `localStorage` ผ่าน `src/utils/cache.js` TTL 6 ชั่วโมงเช่นกัน
- ไม่มีปุ่ม "Refresh Data" ในเวอร์ชันนี้ — เพิ่มได้โดยเรียก `cacheClearAll()` จาก `utils/cache.js`
  ก่อนเรียก API ซ้ำ (ฝั่ง backend ต้องเพิ่ม endpoint สำหรับล้าง request cache ด้วยถ้าต้องการบังคับ
  ข้าม cache ชั้น Postgres)

## 9. Demo

เปิดแอปมาแสดง **KTB** (ธนาคารกรุงไทย) เป็นตัวอย่างเสมอ ตามที่ spec กำหนด
พิมพ์ในช่องค้นหาเพื่อสลับไปดู PTT, KBANK, AOT, CPALL หรือบริษัทอื่นที่มีใน SEC Open API

## 10. Responsive UI

ออกแบบ mobile-first: จำกัดความกว้างหน้าจอไว้ที่ 480px แม้เปิดบนจอใหญ่ เพื่อให้ยังรู้สึกเหมือน
แอปมือถือ ใช้ Bottom Navigation แบบ fixed พร้อม `env(safe-area-inset-bottom)` รองรับมือถือที่มี
notch/home indicator และ focus state ที่มองเห็นชัดเจนสำหรับการใช้งานด้วยคีย์บอร์ด
