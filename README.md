# Lerning Freeradius

การตั้งค่า FreeRADIUS บน Docker ให้เชื่อมต่อกับ PostgreSQL ภายนอก จะต้องใช้การดึงไฟล์ Configuration พื้นฐานออกมา (Volume Mount) เพื่อเปิดใช้งาน Module `sql` และแก้ไขค่าการเชื่อมต่อ

1. **สร้างโฟลเดอร์สำหรับโปรเจกต์:** เตรียมพื้นที่ทำงาน.
สร้างโฟลเดอร์เพื่อเก็บไฟล์ Compose และ Configuration ของตัวอักษร

```bash
mkdir -p freeradius-postgres/raddb
cd freeradius-postgres

```


2. **สร้างไฟล์ Docker Compose:** docker-compose.yml.
สร้างไฟล์ `docker-compose.yml` สำหรับรัน FreeRADIUS

```yaml
services:
  freeradius:
    image: freeradius/freeradius-server:latest
    container_name: freeradius-server
    ports:
      - "1812:1812/udp"
      - "1813:1813/udp"
    volumes:
      - ./raddb:/etc/raddb
    restart: unless-stopped

```


3. **ดึงค่า Config พื้นฐานออกมา:** Extract Configuration.
รัน Container ชั่วคราวเพื่อคัดลอกไฟล์ตั้งค่า (`raddb`) ออกมาไว้บนเครื่อง Host เพื่อให้เราแก้ไขค่าต่างๆ ได้

```bash
docker run --rm -d --name rad-temp freeradius/freeradius-server:latest
docker cp rad-temp:/etc/raddb/. ./raddb/
docker stop rad-temp

```


4. **เปิดใช้งาน SQL Module และเชื่อมต่อ Postgres:** แก้ไขไฟล์ใน raddb.
เข้าไปแก้ไขไฟล์ตั้งค่าเพื่อเปลี่ยนให้ระบบรู้จักกับ PostgreSQL

1. เปิดไฟล์ `raddb/mods-available/sql`
2. ค้นหาและแก้ไขชุดคำสั่งให้ตรงกับ Database ภายนอกของคุณ:

```text
sql {
    driver = "rlm_sql_postgresql"
    dialect = "postgresql"

    # ข้อมูลการเชื่อมต่อ PostgreSQL ภายนอก
    server = "IP_หรือ_โดเมนของ_Postgres"
    port = 5432
    login = "radius_user"
    password = "your_secure_password"
    radius_db = "radius_database"
}

```

3. เปิดใช้งาน Module โดยการทำ Symlink:

```bash
cd raddb/mods-enabled
ln -s ../mods-available/sql sql
cd ../..

```


5. **สร้างตารางใน PostgreSQL ภายนอก:** Import Schema.
นำไฟล์ Schema ไปรันในเซิร์ฟเวอร์ PostgreSQL ของคุณ (ไฟล์จะอยู่ในโฟลเดอร์ที่คุณเพิ่งดึงออกมา)

คำสั่งสำหรับสร้างตารางโครงสร้างของ FreeRADIUS (ตัวอย่างการรันผ่าน psql):

```bash
psql -h IP_หรือ_โดเมนของ_Postgres -U radius_user -d radius_database -f raddb/mods-config/sql/main/postgresql/schema.sql

```


6. **เริ่มการทำงานและตรวจสอบ Log:** Start Container.
รัน Docker Compose เพื่อให้ระบบเริ่มทำงานด้วย Config ใหม่

```bash
docker-compose up -d

```

**การตรวจสอบ:** เช็คว่า FreeRADIUS ต่อ DB สำเร็จหรือไม่ด้วยคำสั่ง `docker logs freeradius-server`
