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

// === MEDICATION STORAGE ===
// ใช้ PropertiesService เก็บข้อมูลยาแทน localStorage
function getMedications() {
  const stored = PropertiesService.getScriptProperties().getProperty('medications');
  return stored ? JSON.parse(stored) : [];
}

function saveMedications(medications) {
  PropertiesService.getScriptProperties().setProperty('medications', JSON.stringify(medications));
}

// === MAIN ===
function doPost(e) {
  const data = JSON.parse(e.postData.contents);
  const message = data.events[0].message.text.trim();

  const calendar = CalendarApp.getCalendarById(CALENDAR_ID);

  // 🆘 คำสั่งช่วยเหลือ
  if (/^help$|^ช่วยเหลือ$/i.test(message)) {
    const helpMsg = `
🆘 วิธีใช้งาน Google Calendar ผ่าน LINE:

📅 CALENDAR:
1️⃣ สร้าง Event:
สร้าง : ประชุม
(ทั้งวันวันนี้)

หรือระบุเวลา:
สร้าง : ประชุม
เวลา : 2025-06-01

หรือ:
สร้าง : ประชุม
เวลาเริ่ม : 2025-06-01 14:00
เวลาจบ : 2025-06-01 15:30

2️⃣ ลบ Event:
ลบ : ประชุม

3️⃣ ดู Event:
วันนี้
พรุ่งนี้
สัปดาห์นี้

💊 MEDICATION:
4️⃣ เพิ่มยา:
เพิ่มยา : ยาแก้ปวด
เวลา : 08:00,14:00,20:00
(หลายเวลาคั่นด้วย ,)

5️⃣ ดูรายการยา:
รายการยา

6️⃣ ลบยา:
ลบยา : ยาแก้ปวด

7️⃣ เช็คยาวันนี้:
ยาวันนี้

📌 ใช้รูปแบบวันที่: yyyy-mm-dd
📌 ใช้รูปแบบเวลา: HH:mm (24 ชั่วโมง)
    `;
    replyToLine(data, helpMsg.trim());
    return;
  }

  // 💊 คำสั่งเกี่ยวกับยา
  if (/^เพิ่มยา/i.test(message)) {
    handleAddMedication(data, message);
    return;
  }
  
  if (/^รายการยา$/i.test(message)) {
    handleListMedications(data);
    return;
  }
  
  if (/^ลบยา/i.test(message)) {
    handleDeleteMedication(data, message);
    return;
  }
  
  if (/^ยาวันนี้$/i.test(message)) {
    handleTodayMedications(data);
    return;
  }

  // 🔍 ตรวจสอบคำสั่งรายงาน
  if (/^วันนี้/i.test(message)) {
    sendEventSummary(data, 0);
    return;
  }
  if (/^พรุ่งนี้/i.test(message)) {
    sendEventSummary(data, 1);
    return;
  }
  if (/อาทิตย์นี้|สัปดาห์นี้/i.test(message)) {
    sendWeeklySummary(data);
    return;
  }

  // 🗑 ลบ Event
  if (message.startsWith("ลบ :")) {
    const name = message.match(/ลบ ?: ?(.*)/)?.[1]?.trim();
    if (!name) return;
    const events = calendar.getEvents(new Date(), new Date(Date.now() + 7 * 24 * 60 * 60 * 1000));
    const match = events.find(ev => ev.getTitle() === name);
    if (match) {
      match.deleteEvent();
      replyToLine(data, `🗑️ ลบ "${name}" แล้ว`);
    } else {
      replyToLine(data, `❌ ไม่พบ "${name}" ใน 7 วัน`);
    }
    return;
  }

  // ✅ สร้าง Event
  if (message.startsWith("สร้าง :")) {
    const name = message.match(/สร้าง ?: ?(.*)/)?.[1]?.trim();
    const startStr = message.match(/เวลาเริ่ม ?: ?(.+)/)?.[1]?.trim();
    const endStr = message.match(/เวลาจบ ?: ?(.+)/)?.[1]?.trim();
    const dateOnly = message.match(/เวลา ?: ?([0-9]{4}-[0-9]{2}-[0-9]{2})/)?.[1];

    if (!name) {
      replyToLine(data, "⚠️ ระบุชื่อ Event เช่น: สร้าง : ประชุม");
      return;
    }

    if (startStr && endStr) {
      const start = new Date(startStr);
      const end = new Date(endStr);
      if (isNaN(start) || isNaN(end)) {
        replyToLine(data, "⚠️ รูปแบบเวลาไม่ถูกต้อง (ต้องเป็น yyyy-mm-dd HH:mm)");
        return;
      }
      calendar.createEvent(name, start, end);
      replyToLine(data, `✅ สร้าง "${name}" เวลา ${formatTime(start)} - ${formatTime(end)}`);
      return;
    }

    if (dateOnly) {
      const date = new Date(dateOnly);
      if (isNaN(date)) {
        replyToLine(data, "⚠️ รูปแบบวันที่ไม่ถูกต้อง (ต้องเป็น yyyy-mm-dd)");
        return;
      }
      calendar.createAllDayEvent(name, date);
      replyToLine(data, `✅ สร้าง "${name}" ทั้งวันวันที่ ${formatDate(date)}`);
      return;
    }

    calendar.createAllDayEvent(name, new Date());
    replyToLine(data, `✅ สร้าง "${name}" ทั้งวันวันนี้`);
  }
}

// === 💊 MEDICATION FUNCTIONS ===
function handleAddMedication(data, message) {
  const nameMatch = message.match(/เพิ่มยา ?: ?(.*?)(?=\n|เวลา|$)/)?.[1]?.trim();
  const timeMatch = message.match(/เวลา ?: ?(.+)/)?.[1]?.trim();
  
  if (!nameMatch) {
    replyToLine(data, "⚠️ ระบุชื่อยา เช่น: เพิ่มยา : ยาแก้ปวด");
    return;
  }
  
  if (!timeMatch) {
    replyToLine(data, "⚠️ ระบุเวลากินยา เช่น: เวลา : 08:00,14:00,20:00");
    return;
  }
  
  const times = timeMatch.split(',').map(t => t.trim()).filter(t => /^\d{2}:\d{2}$/.test(t));
  
  if (times.length === 0) {
    replyToLine(data, "⚠️ รูปแบบเวลาไม่ถูกต้อง ใช้ HH:MM เช่น 08:00");
    return;
  }
  
  const medications = getMedications();
  const existing = medications.find(med => med.name === nameMatch);
  
  if (existing) {
    existing.times = times;
  } else {
    medications.push({
      name: nameMatch,
      times: times,
      createdAt: new Date().toISOString()
    });
  }
  
  saveMedications(medications);
  
  // สร้าง triggers สำหรับแจ้งเตือน
  createMedicationTriggers(nameMatch, times);
  
  replyToLine(data, `💊 เพิ่มยา "${nameMatch}" แล้ว\n🕒 เวลากิน: ${times.join(', ')}\n⏰ ระบบจะแจ้งเตือนทุกวัน`);
}

function handleListMedications(data) {
  const medications = getMedications();
  
  if (medications.length === 0) {
    replyToLine(data, "💊 ยังไม่มียาในระบบ");
    return;
  }
  
  let msg = "💊 รายการยาของคุณ:\n";
  medications.forEach((med, index) => {
    msg += `\n${index + 1}. ${med.name}\n🕒 เวลา: ${med.times.join(', ')}`;
  });
  
  replyToLine(data, msg);
}

function handleDeleteMedication(data, message) {
  const name = message.match(/ลบยา ?: ?(.*)/)?.[1]?.trim();
  
  if (!name) {
    replyToLine(data, "⚠️ ระบุชื่อยาที่ต้องการลบ");
    return;
  }
  
  const medications = getMedications();
  const index = medications.findIndex(med => med.name === name);
  
  if (index === -1) {
    replyToLine(data, `❌ ไม่พบยา "${name}" ในระบบ`);
    return;
  }
  
  medications.splice(index, 1);
  saveMedications(medications);
  
  // ลบ triggers ที่เกี่ยวข้อง
  deleteMedicationTriggers(name);
  
  replyToLine(data, `🗑️ ลบยา "${name}" แล้ว`);
}

function handleTodayMedications(data) {
  const medications = getMedications();
  
  if (medications.length === 0) {
    replyToLine(data, "💊 ยังไม่มียาในระบบ");
    return;
  }
  
  const now = new Date();
  const currentTime = Utilities.formatDate(now, "Asia/Bangkok", "HH:mm");
  
  let msg = "💊 ยาวันนี้:\n";
  
  medications.forEach(med => {
    msg += `\n📌 ${med.name}`;
    med.times.forEach(time => {
      const status = time <= currentTime ? "✅" : "⏰";
      msg += `\n${status} ${time}`;
    });
  });
  
  replyToLine(data, msg);
}

// === 🔔 MEDICATION TRIGGERS ===
function createMedicationTriggers(medName, times) {
  // ลบ trigger เก่าก่อน (ถ้ามี)
  deleteMedicationTriggers(medName);
  
  times.forEach(time => {
    const [hour, minute] = time.split(':').map(Number);
    
    // สร้าง trigger รายวัน
    ScriptApp.newTrigger('sendMedicationReminder')
      .timeBased()
      .everyDays(1)
      .atHour(hour)
      .nearMinute(minute)
      .create();
    
    // เก็บข้อมูล trigger
    const triggers = getTriggerData();
    triggers.push({
      medName: medName,
      time: time,
      type: 'medication'
    });
    saveTriggerData(triggers);
  });
}

function deleteMedicationTriggers(medName) {
  const triggers = ScriptApp.getProjectTriggers();
  const triggerData = getTriggerData();
  
  triggers.forEach(trigger => {
    if (trigger.getHandlerFunction() === 'sendMedicationReminder') {
      const relatedData = triggerData.find(t => t.medName === medName && t.type === 'medication');
      if (relatedData) {
        ScriptApp.deleteTrigger(trigger);
      }
    }
  });
  
  // ลบข้อมูล trigger
  const updatedTriggers = triggerData.filter(t => !(t.medName === medName && t.type === 'medication'));
  saveTriggerData(updatedTriggers);
}

function getTriggerData() {
  const stored = PropertiesService.getScriptProperties().getProperty('triggerData');
  return stored ? JSON.parse(stored) : [];
}

function saveTriggerData(data) {
  PropertiesService.getScriptProperties().setProperty('triggerData', JSON.stringify(data));
}

// === 🔔 MEDICATION REMINDER ===
function sendMedicationReminder() {
  const now = new Date();
  const currentTime = Utilities.formatDate(now, "Asia/Bangkok", "HH:mm");
  const medications = getMedications();
  
  medications.forEach(med => {
    if (med.times.includes(currentTime)) {
      const msg = `💊 ถึงเวลากินยาแล้ว!\n\n🔔 ${med.name}\n🕒 เวลา: ${currentTime}\n\n💡 อย่าลืมกินยาตามเวลานะครับ`;
      sendLinePush(msg);
    }
  });
}

// === 📅 รายงาน Event รายวัน/สัปดาห์ ===
function sendEventSummary(data, offsetDay = 0) {
  const date = new Date();
  date.setDate(date.getDate() + offsetDay);
  const events = CalendarApp.getCalendarById(CALENDAR_ID).getEventsForDay(date);

  if (!events.length) {
    replyToLine(data, offsetDay === 0 ? "📅 วันนี้ไม่มี Event" : "📅 พรุ่งนี้ไม่มี Event");
    return;
  }

  let msg = `📅 Event ${offsetDay === 0 ? "วันนี้" : "พรุ่งนี้"}:\n`;
  events.forEach(e => {
    msg += `\n📌 ${e.getTitle()}\n🕒 ${formatTime(e.getStartTime())} - ${formatTime(e.getEndTime())}`;
  });

  replyToLine(data, msg);
}

function sendWeeklySummary(data) {
  const today = new Date();
  const next7 = new Date(today.getTime() + 7 * 24 * 60 * 60 * 1000);
  const events = CalendarApp.getCalendarById(CALENDAR_ID).getEvents(today, next7);

  if (!events.length) {
    replyToLine(data, "📅 สัปดาห์นี้ยังไม่มี Event");
    return;
  }

  let msg = "📅 Event สัปดาห์นี้:\n";
  events.forEach(e => {
    msg += `\n📌 ${e.getTitle()}\n📆 ${formatDate(e.getStartTime())} 🕒 ${formatTime(e.getStartTime())} - ${formatTime(e.getEndTime())}`;
  });

  replyToLine(data, msg);
}

// === 🔁 ส่งกลับ LINE ===
function replyToLine(data, msg) {
  const payload = {
    replyToken: data.events[0].replyToken,
    messages: [{ type: 'text', text: msg }]
  };
  UrlFetchApp.fetch('https://api.line.me/v2/bot/message/reply', {
    method: 'post',
    contentType: 'application/json',
    headers: {
      Authorization: 'Bearer ' + LINE_TOKEN
    },
    payload: JSON.stringify(payload)
  });
}

// === 🕓 ฟอร์แมตเวลา ===
function formatTime(date) {
  return Utilities.formatDate(date, "Asia/Bangkok", "HH:mm");
}

function formatDate(date) {
  return Utilities.formatDate(date, "Asia/Bangkok", "yyyy-MM-dd");
}

function sendDailyEvents() {
  const calendar = CalendarApp.getCalendarById(CALENDAR_ID);
  const today = new Date();
  const events = calendar.getEventsForDay(today);

  if (events.length === 0) {
    sendLinePush("📅 วันนี้ไม่มี Event นะครับ");
    return;
  }

  let msg = "📅 Event วันนี้:\n";
  events.forEach(event => {
    msg += `\n🕒 ${event.getTitle()}\nเวลา: ${formatTime(event.getStartTime())} - ${formatTime(event.getEndTime())}`;
  });

  sendLinePush(msg);
}

function sendLinePush(msg) {
  const payload = {
    to: LINE_USER_ID,
    messages: [{ type: 'text', text: msg }]
  };

  UrlFetchApp.fetch('https://api.line.me/v2/bot/message/push', {
    method: 'post',
    contentType: 'application/json',
    headers: {
      Authorization: 'Bearer ' + LINE_TOKEN
    },
    payload: JSON.stringify(payload)
  });
}

// === 🔄 DAILY MEDICATION SUMMARY ===
function sendDailyMedicationSummary() {
  const medications = getMedications();
  
  if (medications.length === 0) return;
  
  let msg = "💊 สรุปยาวันนี้:\n";
  
  medications.forEach(med => {
    msg += `\n📌 ${med.name}`;
    med.times.forEach(time => {
      msg += `\n⏰ ${time}`;
    });
  });
  
  msg += "\n\n💡 อย่าลืมกินยาตามเวลานะครับ!";
  
  sendLinePush(msg);
}
