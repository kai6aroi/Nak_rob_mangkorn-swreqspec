# Plan: ยื่นคำร้องขอฝึกงาน

## 1. สรุปแนวทาง
ฟีเจอร์นี้ให้นักศึกษาสามารถเลือกตำแหน่งงานและกรอกข้อมูลคำร้องฝึกงานพร้อมแนบไฟล์ PDF ได้ครบถ้วนก่อนส่งคำร้อง ระบบจะตรวจสิทธิ์ผ่านระบบทะเบียนกลางและบันทึกคำร้องไว้ในสถานะรอการอนุมัติ เมื่อส่งคำร้องสำเร็จจะส่งแจ้งเตือนไปยังอาจารย์ที่ปรึกษาหลักเพียง 1 คน โดยมีการตรวจสอบไฟล์และการ retry การแจ้งเตือนอย่างมีเหตุผล การออกแบบจะเน้นความชัดเจนของข้อมูลและการป้องกันการยื่นคำร้องที่ไม่สมบูรณ์หรือไม่ปลอดภัย

## 2. เทคโนโลยีที่ใช้

| สิ่งที่เลือก | มาจาก | หมายเหตุ |
|---|---|---|
| React + Vite สำหรับ frontend | ทีมเลือกเอง ไม่ได้มาจาก spec | ใช้สร้างหน้าจอยื่นคำร้องและแสดงสถานะคำร้อง |
| Python FastAPI สำหรับ backend | ทีมเลือกเอง ไม่ได้มาจาก spec | ใช้จัดการฟอร์มคำร้อง API, validation และ workflow |
| PostgreSQL สำหรับฐานข้อมูล | ทีมเลือกเอง ไม่ได้มาจาก spec | จัดเก็บคำร้อง สถานประกอบการ ไฟล์ metadata และการแจ้งเตือน |
| API เชื่อมต่อระบบทะเบียนกลาง | PRE-01 | ใช้ตรวจเงื่อนไขผ่านรายวิชาบังคับก่อนและสถานะทางการศึกษาปกติ |
| Virus scan + encrypted storage | NFR-SEC-01 | ใช้เป็นเงื่อนไขบังคับก่อนบันทึกคำร้องสำเร็จ |

## 3. โมเดลข้อมูล

| Entity | ฟิลด์หลัก | รองรับ FR |
|---|---|---|
| StudentProfile | student_id, advisor_id, registration_status, prerequisite_status | FR-UC02-01, FR-UC02-02, FR-UC02-06, PRE-01 |
| InternshipOpportunity | opportunity_id, title, company_id, status, semester | FR-UC02-01 |
| Company | company_id, name, contact_name, address, is_draft | FR-UC02-03, FR-UC02-04 |
| InternshipApplication | application_id, student_id, opportunity_id, company_id, status, created_at, submitted_at, agreement_accepted | FR-UC02-03, FR-UC02-05, FR-UC02-06 |
| Attachment | attachment_id, application_id, type, file_name, storage_path, file_size, virus_scan_status, is_encrypted | FR-UC02-03, FR-UC02-07, NFR-SEC-01 |
| NotificationLog | notification_id, application_id, recipient_id, status, retries, next_retry_at, error_message | FR-UC02-06, FR-UC02-08 |
| ApplicationDraft | draft_id, application_id, company_draft_flag, created_by, created_at, reviewed_by | FR-UC02-04 |

> ข้อมูลที่เป็นความลับหรือข้อมูลส่วนบุคคลที่ไม่ระบุใน spec จะไม่ถูกเก็บเพิ่ม เช่น เลขบัตรประชาชน หรือข้อมูลที่ไม่เกี่ยวข้องกับคำร้องฝึกงาน เพื่อให้สอดคล้องกับ Constraints และ Scope

## 4. API / หน้าจอ

| รายการ | รายละเอียด | รองรับ FR |
|---|---|---|
| GET /internship-opportunities | ดึงตำแหน่งฝึกงานที่เปิดให้ยื่นคำร้อง โดยแสดงข้อมูลสั้นและคัดกรองตามภาคการศึกษา | FR-UC02-01 |
| GET /students/:id/eligibility | ตรวจสอบสถานะพร้อมลงทะเบียนผ่านระบบทะเบียนกลาง และคืนผล prerequisite + active status | PRE-01 |
| GET /companies | ดึงรายชื่อสถานประกอบการที่มีอยู่ในระบบ สำหรับเลือกในฟอร์ม | FR-UC02-03 |
| POST /companies | เพิ่มสถานประกอบการใหม่ใน Draft State และคืน company_id สำหรับใช้ต่อในฟอร์มคำร้อง | FR-UC02-04 |
| POST /applications | สร้างคำร้องฝึกงานพร้อมข้อมูลนักศึกษาและสถานประกอบการ | FR-UC02-02, FR-UC02-03 |
| POST /applications/:id/attachments | รับไฟล์ Resume/Transcript ในรูปแบบ PDF และตรวจสแกนไวรัสก่อนบันทึก | FR-UC02-03, FR-UC02-07, NFR-SEC-01 |
| POST /applications/:id/submit | ตรวจเงื่อนไขก่อนส่ง, บังคับ Checkbox, บันทึกคำร้องและเปลี่ยนสถานะเป็น รอการอนุมัติ | FR-UC02-05, FR-UC02-06 |
| GET /applications/:id | ดูหน้าแสดงสรุปข้อมูลคำร้องและผลลัพธ์การยืนยัน/แจ้งเตือน | FR-UC02-05, FR-UC02-06 |
| POST /notifications/retry | เริ่ม retry ส่งข้อความแจ้งเตือนไปยังอาจารย์ที่ปรึกษาแบบอัตโนมัติ | FR-UC02-08 |

### หน้าจอหลัก
- หน้าเลือกตำแหน่งฝึกงาน: FR-UC02-01
- หน้าแบบฟอร์มคำร้อง: FR-UC02-02, FR-UC02-03
- หน้าเพิ่มสถานประกอบการใหม่: FR-UC02-04
- หน้าสรุปข้อมูลและยืนยัน: FR-UC02-05
- หน้าผลลัพธ์หลังยื่นคำร้อง: FR-UC02-06, FR-UC02-08

## 5. ตารางตรวจ Constraints

| Constraint ID | ถูกนำไปใช้ที่ไหนใน plan | สถานะ |
|---|---|---|
| PRE-01 | GET /students/:id/eligibility และ POST /applications/:id/submit | ใช้แล้ว |
| NFR-SEC-01 | POST /applications/:id/attachments และ model Attachment | ใช้แล้ว |

## 6. แผนทดสอบจาก Acceptance Criteria

| AC ID | ชื่อ test | ทดสอบอย่างไร |
|---|---|---|
| AC-UC02-01 | test_AC_UC02_01_submit_application_success | Mock API ระบบทะเบียนกลางให้สถานะพร้อมฝึกงาน, submit ข้อมูลครบถ้วน, ตรวจว่าคำร้องเก็บสถานะเป็น รอการอนุมัติ และ notification ถูกส่งถึงอาจารย์ที่ปรึกษาหลักภายใน 3 วินาที |
| AC-UC02-02 | test_AC_UC02_02_add_new_company_and_return_to_form | เลือกสถานประกอบการที่ไม่มีในระบบ, บันทึกข้อมูลใหม่, ตรวจว่าระบบกลับสู่ฟอร์มคำร้องและบันทึก draft company อย่างถูกต้อง |
| AC-UC02-03 | test_AC_UC02_03_block_invalid_attachment | อัปโหลดไฟล์ PDF เกิน 10MB หรือไม่ปลอดภัย, ตรวจว่าหน้าจอแสดงสาเหตุและปุ่มยืนยันถูก block อย่างชัดเจน |
| AC-UC02-04 | test_AC_UC02_04_retry_notification_within_5_minutes | ปลอมสถานะส่ง notification ล้มเหลว, ตรวจว่าระบบเก็บคำร้องและ retry ทุก 1 นาที สูงสุด 5 ครั้ง ภายใน 5 นาที |

## 7. ลำดับงาน

1. กำหนดโครงสร้างฟอร์มและหน้า UI สำหรับเลือกตำแหน่งและกรอกข้อมูลคำร้อง (FR-UC02-01, FR-UC02-02, FR-UC02-03)
2. สร้าง backend API สำหรับตรวจสิทธิ์นักศึกษาและเชื่อมต่อระบบทะเบียนกลาง (PRE-01)
3. สร้างโครงสร้างเก็บข้อมูลคำร้อง สถานประกอบการ และไฟล์แนบ พร้อมระดับความปลอดภัย (FR-UC02-03, FR-UC02-04, NFR-SEC-01)
4. สร้างฟังก์ชัน upload PDF และ validation ขนาด/ความปลอดภัย ก่อนอนุญาตให้ยื่น (FR-UC02-07)
5. สร้างหน้าสรุปข้อมูลและ Checkbox บังคับก่อนกดยืนยัน (FR-UC02-05)
6. สร้าง workflow submit คำร้องและการเปลี่ยนสถานะเป็น รอการอนุมัติ พร้อมส่ง notification ต่ออาจารย์ที่ปรึกษา (FR-UC02-06)
7. สร้างระบบ retry notification กับ error log และตรวจสอบประสิทธิภาพภายใน 5 นาที (FR-UC02-08)
8. ทดสอบครบตาม AC ทั้ง 4 ข้อ และปรับปรุงตาม feedback จากทีม (AC-UC02-01 ถึง AC-UC02-04)

## 8. สิ่งที่ยังไม่ทำ

- ไม่มี Open Questions ที่ค้างอยู่ใน spec หลังจากขั้น clarify แล้ว
- ส่วนที่เกี่ยวข้องกับข้อนี้จะยังไม่สร้างจนกว่าจะได้คำตอบ: ไม่มีค่าที่ต้องรอคำตอบเพิ่มเติม
