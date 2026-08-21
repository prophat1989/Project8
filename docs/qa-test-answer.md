# QA Test Assignment: Advanced Table Reservation

เอกสารนี้ถอดและจัดโครงสร้างคำตอบจาก `test_exam101.xlsx` สำหรับการทดสอบฟีเจอร์ Advanced Table Reservation ของระบบจัดการร้านอาหาร (OMS)

## 1. Overall Detail

| หัวข้อ | รายละเอียด |
|---|---|
| ชื่องาน | การทดสอบฟีเจอร์ Advanced Table Reservation สำหรับระบบจัดการร้านอาหาร (OMS) |
| Inventory | Indoor 10 โต๊ะ, Outdoor 5 โต๊ะ |
| Logic หลัก | การจอง 1 ครั้งต้องระบุ Timeslot ละ 1 ชั่วโมงเท่านั้น |
| Constraint หลัก | หากโต๊ะประเภทนั้นเต็มใน Timeslot ที่เลือก ระบบต้องไม่อนุญาตให้จอง เพื่อป้องกัน Overbooking |
| Integration | เมื่อจองสำเร็จ ระบบต้องส่งข้อมูลไปยัง CRM API เพื่อสะสมแต้ม |

## 2. Task 1: Test Strategy & Design

| No. | Type | Test Scenario | รายละเอียด / เหตุผล | Priority |
|---:|---|---|---|---|
| 1 | Happy Path | จอง Indoor สำเร็จเมื่อมีโต๊ะว่าง | ตรวจสอบ flow ตั้งแต่เลือกโต๊ะ ยืนยัน จนได้รับ Confirmation | High |
| 2 | Happy Path | จอง Outdoor สำเร็จเมื่อมีโต๊ะว่าง | ตรวจสอบการนับ stock Indoor/Outdoor แยกกัน | High |
| 3 | Happy Path | Booking สำเร็จและส่งข้อมูลไป CRM | ตรวจสอบ integration เพื่อสะสมแต้ม | High |
| 4 | Edge Case | จองประเภทโต๊ะที่เต็มแล้ว | ปฏิเสธเมื่อ Indoor=10 หรือ Outdoor=5 และต้องไม่เกิด Overbooking | High |
| 5 | Edge Case | จองคาบเกี่ยวเวลา เช่น 18:00-19:00 กับ 18:30-19:30 | ยืนยันว่าใช้เฉพาะ Timeslot มาตรฐานแบบชั่วโมงเต็ม | High |
| 6 | Edge Case | Cancel แล้ว Rebook ทันที | ยืนยันว่าระบบคืน inventory หลังยกเลิกทันที | High |
| 7 | Edge Case | ผู้ใช้ 2 คนจองโต๊ะสุดท้ายพร้อมกัน | ตรวจสอบ locking/concurrency control | High |
| 8 | Edge Case | จองช่วงเวลาไม่เต็ม 1 ชั่วโมง | Validation ต้องรับเฉพาะ Timeslot ที่กำหนดไว้ | Medium |
| 9 | Edge Case | OMS สำเร็จแต่ CRM timeout/down | ตรวจสอบ retry, queue หรือการ sync แต้มภายหลัง โดยไม่ทำให้ booking ล้มเหลว | High |
| 10 | Negative | ระบุ `table_type` ที่ไม่มี เช่น `VIP` | ปฏิเสธค่าที่ไม่อยู่ใน enum `Indoor`/`Outdoor` | Medium |
| 11 | Negative | ใช้ `customer_id` ที่ไม่มีจริง | ตรวจสอบลูกค้าก่อนสร้าง booking หรือเรียก CRM | Medium |
| 12 | Edge Case | จองวันที่ผ่านมาแล้วหรือเกิน 90 วัน | ตรวจสอบ business rule ของช่วงวันที่ | Low |
| 13 | Edge Case | Timeslot คร่อมเที่ยงคืนหรือเวลาปิดร้าน | จัดการหรือปฏิเสธช่วงเวลาที่อยู่นอกเวลาทำการอย่างเหมาะสม | Low |

### Detailed Test Case: Concurrent Booking / Race Condition

- **Test Case ID:** `TC-007`
- **Scenario:** No. 7 ผู้ใช้ 2 คนพยายามจองโต๊ะตัวสุดท้ายพร้อมกัน
- **Objective:** ยืนยันว่าไม่เกิด Overbooking เมื่อ request สำหรับประเภทโต๊ะและ Timeslot เดียวกันเข้าพร้อมกัน
- **Priority:** High
- **Type:** Functional / Concurrency / Non-happy Path

#### Preconditions

1. มีโต๊ะ Indoor ทั้งหมด 10 โต๊ะ
2. วันที่ `2024-05-20` เวลา `18:00-19:00` ถูกจองแล้ว 9 โต๊ะ เหลือ 1 โต๊ะ
3. `CUST-001` และ `CUST-002` มีข้อมูลถูกต้องใน CRM

#### Test Data

- User A: `CUST-001`, `Indoor`, `2024-05-20`, `18:00-19:00`
- User B: `CUST-002`, `Indoor`, `2024-05-20`, `18:00-19:00`
- ทุกฟิลด์เหมือนกัน ยกเว้น `customer_id`

#### Steps

1. เตรียมข้อมูลให้เหลือ Indoor 1 โต๊ะใน Timeslot ดังกล่าว
2. ยิง `POST /api/v1/reservations` จาก User A และ User B แบบ parallel ห่างกันไม่เกิน 50-100 ms
3. บันทึก HTTP status และ response body ของทั้งสอง request
4. ตรวจสอบจำนวน record ใน `bookings`
5. ตรวจสอบ log หรือ Mock Server ว่า CRM API ถูกเรียกเฉพาะรายการที่สำเร็จ
6. ทำซ้ำอย่างน้อย 5-10 รอบ

#### Expected Result

1. มีเพียง 1 request ที่ได้ `HTTP 201 Created`
2. อีก request ได้ `HTTP 409 Conflict` พร้อมข้อความว่าโต๊ะเต็ม
3. จำนวน booking ของ Indoor/วันที่/Timeslot นี้ไม่เกิน 10 รายการทุกครั้ง
4. CRM API ถูกเรียกเพียง 1 ครั้งสำหรับลูกค้าที่จองสำเร็จ
5. ไม่เกิด `HTTP 500` หรือ Deadlock

## 3. Task 2: API Negative Testing

Endpoint: `POST /api/v1/reservations`

| No. | Negative Test Case | ตัวอย่าง Input | Expected HTTP | Expected Behavior |
|---:|---|---|---|---|
| 1 | Missing required field | ไม่มี `customer_id` เช่น `{"table_type":"Indoor","date":"2024-05-20","timeslot":"18:00-19:00"}` | 400 | Validate ฟิลด์ที่จำเป็นและแจ้งชื่อฟิลด์ที่ขาด |
| 2 | Invalid enum | `table_type: "VIP"` | 400 | ปฏิเสธค่าที่ไม่อยู่ใน `Indoor`/`Outdoor` |
| 3 | Invalid date | `20-05-2024` หรือ `2024-13-45` | 400 | ตรวจรูปแบบ `YYYY-MM-DD` และวันที่จริง |
| 4 | Invalid Timeslot | `18:00-18:30` หรือ `18:15-19:15` | 400 | รับเฉพาะ Timeslot มาตรฐานชั่วโมงเต็ม |
| 5 | Past date | `date: "2023-01-01"` | 400 | ไม่อนุญาตให้จองย้อนหลัง |
| 6 | Unknown customer | `customer_id: "CUST-00000"` | 404 หรือ 400 พร้อม error code | ตรวจสอบสมาชิกก่อนสร้าง booking |
| 7 | Inventory full | Indoor วันที่ `2024-05-20` เวลา `18:00-19:00` เต็ม 10 โต๊ะ | 409 | ปฏิเสธเพื่อป้องกัน Overbooking |
| 8 | Invalid/empty JSON | Body ว่าง หรือ `{table_type: Indoor,}` | 400 | จัดการ parsing error โดยไม่ crash หรือคืน 500 |

### สาเหตุที่เป็นไปได้ของ HTTP 409 Conflict

1. **Inventory Exhausted:** จำนวน Indoor/Outdoor ในวันและ Timeslot นั้นถึงขีดจำกัด (`10`/`5`)
2. **Concurrency Conflict:** ผู้ใช้หลายคนจองโต๊ะสุดท้ายพร้อมกัน และ resource ถูกจองไปก่อนหน้าในช่วงเสี้ยววินาที
3. **Duplicate Booking:** ผู้ใช้เดิมส่ง request ซ้ำ หรือ client retry request เดิมโดยไม่มี idempotency ที่เหมาะสม
4. **State Conflict:** สถานะ resource เปลี่ยนระหว่างประมวลผล เช่น booking อยู่ระหว่างแก้ไข/ยกเลิกหรือ sync

## 4. Task 3: Data Integrity & SQL

### 4.1 Verify Stock

ตรวจจำนวน Indoor ที่ active ในวันที่ `2024-05-20` เวลา `18:00-19:00` โดยไม่นับรายการ `CANCELLED`:

```sql
SELECT
  table_type,
  COUNT(*) AS booked_count
FROM bookings
WHERE booking_date = '2024-05-20'
  AND timeslot = '18:00-19:00'
  AND table_type = 'Indoor'
  AND status IN ('CONFIRMED', 'BOOKED')
GROUP BY table_type;
```

### 4.2 Data Mismatch

ค้นหา `customer_id` ที่มี booking แต่ไม่มีสมาชิกใน `members`:

```sql
SELECT DISTINCT b.customer_id
FROM bookings b
LEFT JOIN members m
  ON b.customer_id = m.customer_id
WHERE m.customer_id IS NULL;
```

**Assumption:** `bookings` มี `booking_id`, `customer_id`, `table_type`, `booking_date`, `timeslot`, `status` และ `members` มี `customer_id`, `name`, `points` เป็นต้น หาก schema จริงใช้ชื่ออื่น เช่น `reservation_date` ต้องปรับ query ให้ตรงกับฐานข้อมูลจริง

## 5. Task 4: Bug Report

- **Bug ID:** `BUG-2024-0520-01`
- **Title:** `[Mobile App] ระบบอนุญาตให้จอง Outdoor เกินจำนวนจริง: สำเร็จ 6 รายการจาก stock 5 โต๊ะ`
- **Severity:** High
- **Priority:** High
- **Environment:** พบใน Mobile App เช่น iOS 17.x / Android 14 และ App Version ที่ใช้ทดสอบต้องระบุให้ครบ; Web Application ทำงานถูกต้องและปฏิเสธรายการที่ 6 ด้วย `409 Conflict`

### Preconditions

1. Timeslot ที่ทดสอบมี Outdoor ว่างพอดี 5 โต๊ะ
2. มีบัญชีทดสอบอย่างน้อย 6 บัญชี หรือจำลอง concurrent request 6 รายการผ่าน Mobile App

### Steps to Reproduce

1. Login เข้า Mobile App
2. เลือก Outdoor ในวันที่และ Timeslot ที่เหลือ 5 โต๊ะ
3. Submit booking 6 ครั้งต่อเนื่องหรือพร้อมกันสำหรับ Timeslot เดียวกัน
4. ตรวจผลลัพธ์ทั้ง 6 รายการและจำนวน booking ใน Database/Admin Panel
5. ทำซ้ำบน Web เพื่อเปรียบเทียบ

### Actual Result

Mobile App ยอมรับ booking สำเร็จทั้ง 6 รายการ ทั้งที่ Outdoor มีเพียง 5 โต๊ะ จึงเกิด Overbooking 1 รายการ ส่วน Web ปฏิเสธรายการที่ 6 ด้วย `409 Conflict`

### Expected Result

ระบบต้องอนุญาต Outdoor สูงสุด 5 รายการต่อ Timeslot และปฏิเสธรายการที่ 6 ด้วยข้อความว่าโต๊ะเต็ม โดย Mobile App และ Web ต้องมีพฤติกรรมตรงกัน

### Suspected Root Cause

1. Mobile App เรียก API endpoint หรือ API version คนละตัวกับ Web ซึ่งไม่มี server-side stock validation
2. Mobile App cache จำนวนโต๊ะว่างและไม่ refresh ข้อมูลล่าสุดก่อน confirm
3. Business logic ถูกทำซ้ำที่ client แทนที่จะบังคับตรวจ stock ที่ server เป็นจุดเดียว

### Business Impact

ร้านอาจรับลูกค้าเกินความจุจริง ต้องปฏิเสธลูกค้าหน้างาน และกระทบความน่าเชื่อถือของระบบ

### Attachments / Evidence

- Screenshot หรือ screen recording ของ booking 6 รายการ
- Mobile API request/response ทั้ง 6 รายการ
- Query ผลต่างจำนวน booking ในฐานข้อมูลก่อนและหลังทดสอบ

- **Reporter:** `[ชื่อ QA ผู้รายงาน]`
- **Date Reported:** `[วันที่พบปัญหา]`

## Source

ไฟล์ต้นฉบับ: [`test_exam101.xlsx`](../test_exam101.xlsx)
