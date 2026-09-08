# วิธีการสร้าง User ทดสอบใน Database พร้อมวิธียิง radtest ตรวจสอบ

การสร้าง User สำหรับทดสอบจะทำผ่านการเพิ่มข้อมูลลงในตาราง `radcheck` ของ PostgreSQL และใช้คำสั่ง `radtest` จากภายในคอนเทนเนอร์ของ FreeRADIUS เพื่อทดสอบระบบ

1. **เพิ่มข้อมูล User ลงในฐานข้อมูล:** รันคำสั่งบนฝั่ง PostgreSQL.
เข้าไปจัดการฐานข้อมูล PostgreSQL ของคุณ (ผ่านเครื่องมือเช่น DBeaver, pgAdmin หรือ psql) แล้วรันคำสั่ง SQL นี้เพื่อสร้างผู้ใช้งาน `testuser` รหัสผ่าน `testpassword`:

```sql
INSERT INTO radcheck (username, attribute, op, value) 
VALUES ('testuser', 'Cleartext-Password', ':=', 'testpassword');

```


2. **ทดสอบยืนยันตัวตนด้วย radtest:** รันผ่าน Terminal ของ Host.
เนื่องจากคุณใช้งานผ่าน Docker สามารถใช้คำสั่ง `docker exec` เพื่อเรียกใช้ `radtest` จากภายในคอนเทนเนอร์ได้ทันที โดยโครงสร้างคำสั่งคือ `radtest <username> <password> <ip> <port> <secret>`

```bash
docker exec -it freeradius-server radtest testuser testpassword localhost 1812 testing123

```

*(หมายเหตุ: `testing123` คือ Shared Secret ค่าเริ่มต้นสำหรับ localhost ของ FreeRADIUS)*


3. **ตรวจสอบผลลัพธ์:** ดูบรรทัดสุดท้ายของ Log.
เมื่อรันคำสั่ง ระบบจะแสดง Log การคุยกันระหว่าง Client และ Server หากการเชื่อมต่อ Database ถูกต้องและรหัสผ่านตรงกัน บรรทัดสุดท้ายจะต้องแสดงข้อความ **Access-Accept**:

```text
Received Access-Accept Id 1 from 127.0.0.1:1812 to 127.0.0.1:0 length 20

```

หากแสดงเป็น **Access-Reject** ให้เช็คความถูกต้องของรหัสผ่าน, การตั้งค่าการเชื่อมต่อ Database, หรือการตั้งค่า `sites-available`
