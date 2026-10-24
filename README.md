# คู่มือการติดตั้งและตั้งค่า OpenBao บน Ubuntu 24.04 LTS

OpenBao เป็นระบบจัดการความลับ (Secrets Management) แบบ Open-source ที่แยกตัว (Fork) ออกมาจาก HashiCorp Vault คู่มือนี้จะครอบคลุมตั้งแต่การดาวน์โหลด ตั้งค่า ไปจนถึงการเริ่มใช้งานเบื้องต้น

## ข้อกำหนดเบื้องต้น (Prerequisites)

* เครื่องเซิร์ฟเวอร์ที่ติดตั้ง Ubuntu 24.04 LTS

* สิทธิ์การใช้งานระดับ `sudo` หรือ `root`

## ส่วนที่ 1: การติดตั้ง OpenBao

### 1. อัปเดตระบบและติดตั้งแพ็กเกจพื้นฐาน

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install curl wget unzip jq -y

```

### 2. ดาวน์โหลดและติดตั้ง OpenBao Binary

เราจะติดตั้งผ่าน Binary จาก GitHub Releases โดยตรง โดยกำหนดเวอร์ชันและใช้ลิงก์ดาวน์โหลดล่าสุด

```bash
# กำหนดเวอร์ชัน (2.7.1)
BAO_VERSION="2.7.1"

# ดาวน์โหลดไฟล์
wget https://github.com/openbao/openbao/releases/download/v${BAO_VERSION}/openbao_${BAO_VERSION}_linux_amd64.tar.gz

# แตกไฟล์
tar -xvf openbao_${BAO_VERSION}_linux_amd64.tar.gz

# ย้ายไปยัง /usr/local/bin (ถ้าตัว binary ชื่อ bao)
sudo mv bao /usr/local/bin/

# ตรวจสอบการติดตั้ง
bao --version

```

*(ถ้าติดตั้งสำเร็จ จะแสดงเวอร์ชันของ OpenBao ขึ้นมา)*

## ส่วนที่ 2: การตั้งค่าระบบและการสร้าง Systemd Service

### 1. สร้าง System User และ Directories สำหรับ OpenBao

เพื่อความปลอดภัย เราควรให้ OpenBao รันภายใต้ User ของตัวเอง

```bash
# สร้าง User สำหรับ OpenBao
sudo useradd --system --home /etc/openbao.d --shell /bin/false bao

# สร้าง Directory สำหรับเก็บ Configuration และ Data
sudo mkdir -p /etc/openbao.d
sudo mkdir -p /opt/openbao/data

# กำหนดสิทธิ์ให้ User 'bao'
sudo chown -R bao:bao /etc/openbao.d
sudo chown -R bao:bao /opt/openbao/data

```

### 2. สร้างไฟล์ Configuration (HCL)

เราจะสร้างไฟล์คอนฟิกพื้นฐานที่ใช้ Integrated Storage (Raft) ซึ่งเหมาะสำหรับการทำ Cluster ในอนาคต

```bash
sudo nano /etc/openbao.d/openbao.hcl

```

เพิ่มเนื้อหาต่อไปนี้ลงในไฟล์:

```hcl
storage "raft" {
  path    = "/opt/openbao/data"
  node_id = "node1"
}

listener "tcp" {
  address     = "0.0.0.0:8200"
  tls_disable = "true" # ควรเปลี่ยนเป็น false และตั้งค่า SSL/TLS หากใช้งานจริง (Production)
}

api_addr = "http://127.0.0.1:8200"
cluster_addr = "https://127.0.0.1:8201"
ui = true
disable_mlock = true

```

### 3. สร้าง Systemd Service File

เพื่อให้ OpenBao เริ่มทำงานอัตโนมัติเมื่อเปิดเครื่อง

```bash
sudo nano /etc/systemd/system/openbao.service

```

เพิ่มเนื้อหาต่อไปนี้ลงในไฟล์:

```ini
[Unit]
Description="OpenBao secret management tool"
Requires=network-online.target
After=network-online.target
ConditionFileNotEmpty=/etc/openbao.d/openbao.hcl

[Service]
User=bao
Group=bao
ProtectSystem=full
ProtectHome=read-only
PrivateTmp=yes
PrivateDevices=yes
SecureBits=keep-caps
AmbientCapabilities=CAP_IPC_LOCK
Capabilities=CAP_IPC_LOCK+ep
CapabilityBoundingSet=CAP_SYSLOG CAP_IPC_LOCK
NoNewPrivileges=yes
ExecStart=/usr/local/bin/bao server -config=/etc/openbao.d/openbao.hcl
ExecReload=/bin/kill --signal HUP $MAINPID
KillMode=process
KillSignal=SIGINT
Restart=on-failure
RestartSec=5
TimeoutStopSec=30
LimitNOFILE=65536
LimitMEMLOCK=infinity

[Install]
WantedBy=multi-user.target

```

### 4. เปิดใช้งานและเริ่มต้น Service

```bash
sudo systemctl daemon-reload
sudo systemctl enable openbao
sudo systemctl start openbao
sudo systemctl status openbao

```

*(สถานะควรขึ้นเป็น `active (running)`)*

## ส่วนที่ 3: การเริ่มต้น (Initialize) และปลดล็อก (Unseal)

เมื่อรันเซิร์ฟเวอร์ครั้งแรก ระบบจะถูกล็อกไว้ (Sealed) คุณต้องทำการ Initialize เพื่อรับ Unseal Keys และ Root Token

### 1. ตั้งค่า Environment Variable

```bash
export BAO_ADDR='http://127.0.0.1:8200'

```

*(คุณสามารถเพิ่มคำสั่งนี้ลงใน `~/.bashrc` ได้เพื่อไม่ต้องพิมพ์ใหม่ทุกครั้ง)*

### 2. Initialize OpenBao

```bash
bao operator init

```

**คำเตือนที่สำคัญมาก:** คำสั่งนี้จะแสดง **Unseal Keys** (ค่าเริ่มต้นคือ 5 คีย์) และ **Initial Root Token** ออกมาบนหน้าจอ
**กรุณาคัดลอกและบันทึกข้อมูลเหล่านี้ไว้ในที่ปลอดภัยทันที!** หากสูญหาย คุณจะไม่สามารถเข้าถึงข้อมูลใน OpenBao ได้อีก

### 3. ปลดล็อก (Unseal) OpenBao

ค่าเริ่มต้นของ OpenBao ต้องการ 3 ใน 5 คีย์ เพื่อทำการปลดล็อก ให้รันคำสั่งด้านล่างนี้ 3 ครั้ง และใส่ Unseal Key ที่ต่างกัน 3 คีย์

```bash
bao operator unseal
# (ระบบจะให้กรอก Unseal Key ที่บันทึกไว้)

```

### 4. ล็อกอินด้วย Root Token

เมื่อปลดล็อกสำเร็จ (Sealed = false) ให้ทำการล็อกอิน:

```bash
bao login
# (ระบบจะให้กรอก Initial Root Token)

```

## ยินดีด้วย!

คุณได้ติดตั้งและตั้งค่า OpenBao บน Ubuntu 24.04 LTS สำเร็จแล้ว ตอนนี้คุณสามารถใช้งานผ่าน Command Line หรือเข้าถึง Web UI ได้ที่ `http://<IP_เครื่องของคุณ>:8200`
