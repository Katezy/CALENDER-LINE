# LINE Bot สำหรับจัดการ Google Calendar และแจ้งเตือนยา

สคริปต์นี้เป็น Google Apps Script ที่ใช้เชื่อม LINE Messaging API กับ Google Calendar และระบบจัดการยา (medication reminder)  
สามารถสร้าง/ลบ/ดู Event ใน Google Calendar ผ่านข้อความ LINE ได้ และแจ้งเตือนเวลาทานยาผ่าน LINE

---

## ฟีเจอร์หลัก

- สร้าง Event (ทั้งวันหรือระบุเวลา) ใน Google Calendar
- ลบ Event ในช่วง 7 วันข้างหน้า
- ดู Event วันนี้, พรุ่งนี้, สัปดาห์นี้
- เพิ่ม/ลบ/ดูรายการยา พร้อมตั้งเวลาทานยา
- แจ้งเตือนเวลาทานยาอัตโนมัติผ่าน LINE Push Message
- ส่งสรุป Event และยาแบบรายวัน

---

## วิธีใช้งาน

### 1. สร้าง Google Apps Script Project

- เข้าไปที่ https://script.google.com/
- สร้างโปรเจคใหม่
- คัดลอกโค้ดทั้งหมดในไฟล์สคริปต์ของคุณ (ตัวอย่างโค้ดที่ให้ไว้)
- วางในไฟล์ `Code.gs` ในโปรเจคใหม่

### 2. ตั้งค่าคอนฟิก

แก้ไขค่าคงที่ในโค้ดให้ตรงกับของคุณ เช่น

```js
const LINE_TOKEN = 'ใส่ LINE Channel Access Token ของคุณ';
const LINE_USER_ID = 'ใส่ LINE User ID สำหรับส่งข้อความแบบ Push';
const CALENDAR_ID = 'ใส่ Google Calendar ID หรือใช้ "primary"';
