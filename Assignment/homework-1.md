# รายงานการออกแบบและจำลองระบบเครือข่ายองค์กรด้วย Cisco Packet Tracer

รายงานฉบับนี้จัดทำขึ้นเพื่อแสดงรายละเอียดการออกแบบระบบเครือข่ายสำหรับบริษัท โดยมีเงื่อนไขหลักคือ **"แบ่งแยกเครือข่ายแต่ละแผนกไม่ให้สามารถ Ping ข้ามวงได้"** และ **"ใช้งบประมาณน้อยที่สุด"** 

---

## 1. วิธีการแบ่งส่วนเครือข่าย (Network Segmentation)
เพื่อตอบโจทย์การแยกวงเครือข่ายและจำกัดงบประมาณ ระบบนี้ใช้เทคนิคดังต่อไปนี้:
*   **VLAN (Virtual LAN):** แยกเครือข่ายย่อยทางตรรกะใน Switch ตัวเดียวกันหรือข้าม Switch 
*   **Router-on-a-Stick:** ใช้ Router 1 ตัวสร้าง Sub-interface เพื่อเป็น Gateway ให้กับทุกๆ VLAN
*   **DHCP Server:** ให้ Router ทำหน้าที่แจก IP Address อัตโนมัติให้คอมพิวเตอร์ทุกเครื่องตาม VLAN ของตนเอง
*   **ACL (Access Control List):** สร้างกฎบน Router เพื่อบล็อกไม่ให้เครือข่ายภายใน (Private IP) สามารถสื่อสารข้ามแผนกได้ แต่ยังคงอนุญาตให้ออกอินเทอร์เน็ตได้
*   **Core Switch Optimization:** นำ Switch ของแผนกไอที (IT) มาทำหน้าที่เป็น Main Switch (จุดศูนย์กลาง) เพื่อประหยัดงบประมาณในการซื้อ Switch หลักแยกต่างหาก

---

## 2. การแบ่งวง IP Address และ VLAN
เลือกใช้ IP Address คลาส C (Subnet Mask: `255.255.255.0` หรือ `/24`) ซึ่งรองรับได้ 254 เครื่องต่อแผนก เพียงพอต่อการใช้งาน

| แผนก (ฝ่าย) | VLAN ID | Network Address | Default Gateway | แจก IP ตั้งแต่ (DHCP) |
| :--- | :---: | :--- | :--- | :--- |
| ฝ่ายขาย (Sales) | 10 | `192.168.10.0/24` | `192.168.10.1` | `192.168.10.2 - 254` |
| ฝ่ายผลิต (Production)| 20 | `192.168.20.0/24` | `192.168.20.1` | `192.168.20.2 - 254` |
| ฝ่ายบัญชี (Accounting)| 30 | `192.168.30.0/24` | `192.168.30.1` | `192.168.30.2 - 254` |
| ฝ่ายไอที (IT) | 40 | `192.168.40.0/24` | `192.168.40.1` | `192.168.40.2 - 254` |
| อื่นๆ (Others) | 50 | `192.168.50.0/24` | `192.168.50.1` | `192.168.50.2 - 254` |

---

## 3. รายการอุปกรณ์ที่ใช้และงบประมาณ
ใช้วิธีการจัดสรรพอร์ตโดยดึง Switch ของฝ่ายไอทีมาเป็นศูนย์กลาง (Main Switch) เพื่อประหยัดงบประมาณ
*(หมายเหตุ: ใน Packet Tracer ใช้รุ่น 2960 เป็นตัวแทนจำลองการทำงาน)*

| ฝ่าย | จำนวนเครื่อง | รายการ Switch ที่ต้องใช้ | ราคา (บาท) |
| :--- | :---: | :--- | :---: |
| **ไอที (ทำหน้าที่ Main)**| 10 | 16-Port x 1 ตัว *(พอร์ตพอดีเป๊ะ)* | 3,000 |
| **ขาย** | 58 | 16-Port x 4 ตัว, 8-Port x 1 ตัว | 14,000 |
| **ผลิต** | 37 | 16-Port x 3 ตัว | 9,000 |
| **บัญชี** | 16 | 16-Port x 1 ตัว, 8-Port x 1 ตัว | 5,000 |
| **อื่นๆ** | 61 | 16-Port x 4 ตัว, 8-Port x 1 ตัว | 14,000 |
| **Router (Gateway)**| 1 | Cisco ISR 4331 | - |
| **รวมงบประมาณที่คุ้มค่าที่สุด** | | | **45,000 บาท** |

---

## 4. คำสั่งในการ Config อุปกรณ์ (CLI Configuration)

### 4.1 การตั้งค่า Router (รุ่น ISR 4331)
**จุดประสงค์:** เปิดพอร์ต, สร้าง Sub-interface, ทำ DHCP Server และทำ ACL Block Ping ข้ามแผนก

```text
enable
configure terminal

! 1. เปิดพอร์ตหลัก และทำให้แน่ใจว่าไม่มี IP ซ้อนทับ
interface g0/0/0
no ip address
no shutdown
exit

! 2. สร้าง Sub-Interface และแจก IP สำหรับฝ่ายขาย (VLAN 10)
interface g0/0/0.10
encapsulation dot1Q 10
ip address 192.168.10.1 255.255.255.0
exit
ip dhcp excluded-address 192.168.10.1
ip dhcp pool SALES
network 192.168.10.0 255.255.255.0
default-router 192.168.10.1
dns-server 8.8.8.8
exit

! 3. สร้าง Sub-Interface และแจก IP สำหรับฝ่ายผลิต (VLAN 20)
interface g0/0/0.20
encapsulation dot1Q 20
ip address 192.168.20.1 255.255.255.0
exit
ip dhcp excluded-address 192.168.20.1
ip dhcp pool PRODUCTION
network 192.168.20.0 255.255.255.0
default-router 192.168.20.1
dns-server 8.8.8.8
exit

! 4. สร้าง Sub-Interface และแจก IP สำหรับฝ่ายบัญชี (VLAN 30)
interface g0/0/0.30
encapsulation dot1Q 30
ip address 192.168.30.1 255.255.255.0
exit
ip dhcp excluded-address 192.168.30.1
ip dhcp pool ACCOUNTING
network 192.168.30.0 255.255.255.0
default-router 192.168.30.1
dns-server 8.8.8.8
exit

! 5. สร้าง Sub-Interface และแจก IP สำหรับฝ่ายไอที (VLAN 40)
interface g0/0/0.40
encapsulation dot1Q 40
ip address 192.168.40.1 255.255.255.0
exit
ip dhcp excluded-address 192.168.40.1
ip dhcp pool IT
network 192.168.40.0 255.255.255.0
default-router 192.168.40.1
dns-server 8.8.8.8
exit

! 6. สร้าง Sub-Interface และแจก IP สำหรับฝ่ายอื่นๆ (VLAN 50)
interface g0/0/0.50
encapsulation dot1Q 50
ip address 192.168.50.1 255.255.255.0
exit
ip dhcp excluded-address 192.168.50.1
ip dhcp pool OTHERS
network 192.168.50.0 255.255.255.0
default-router 192.168.50.1
dns-server 8.8.8.8
exit

! 7. ตั้งค่า ACL (Access Control List)
! บล็อก IP 192.168.x.x ไม่ให้คุยกันเอง แต่บรรทัดล่างอนุญาตให้ออกเน็ตได้
access-list 100 deny ip 192.168.0.0 0.0.255.255 192.168.0.0 0.0.255.255
access-list 100 permit ip any any

! 8. นำ ACL ไปผูกกับ Sub-Interface ของทุก VLAN
interface range g0/0/0.10, g0/0/0.20, g0/0/0.30, g0/0/0.40, g0/0/0.50
ip access-group 100 in
exit
```
### 4.2 การตั้งค่า Switch ฝ่ายไอที (ทำหน้าที่ Main Switch)
***จุดประสงค์:*** รู้จักทุก VLAN, ส่งต่อ Trunk ไปหาแผนกอื่น และเป็น Access ให้แผนกไอที

```text
enable
configure terminal

! 1. สร้าง VLAN ฐานข้อมูล
vlan 10
name Sales
vlan 20
name Production
vlan 30
name Accounting
vlan 40
name IT
vlan 50
name Others
exit

! 2. พอร์ต g0/1 ต่อไปหา Router (ทำเป็น Trunk)
interface g0/1
switchport mode trunk
exit

! 3. พอร์ต f0/1-4 ลากไปหา Switch ฝ่ายอื่นๆ (ทำเป็น Trunk)
interface range f0/1-4
switchport mode trunk
exit

! 4. พอร์ต f0/5-14 ลากไปหา PC ฝ่ายไอที 10 เครื่อง (ทำเป็น Access VLAN 40)
interface range f0/5-14
switchport mode access
switchport access vlan 40
exit
```
### 4.3 การตั้งค่า Switch แผนกอื่นๆ (ตัวอย่าง: สวิตช์ฝ่ายขายตัวแรก)
***จุดประสงค์:*** รับสาย Trunk มาจาก Switch ไอที, ส่ง Trunk ต่อให้ Switch ตัวที่ 2 (Cascade) และจ่าย Access ให้ PC ตัวเอง
```text
enable
configure terminal

! 1. สร้าง VLAN ของตัวเอง
vlan 10
name Sales
exit

! 2. พอร์ต g0/1 รับสายประมาจาก Switch ไอที (ทำเป็น Trunk)
interface g0/1
switchport mode trunk
exit

! 3. พอร์ต g0/2 ส่งสายประต่อไปหาสวิตช์ฝ่ายขายตัวที่ 2 (ทำเป็น Trunk)
! *หากเป็น Switch ตัวสุดท้ายของแผนก ไม่ต้องทำข้อนี้*
interface g0/2
switchport mode trunk
exit

! 4. พอร์ต f0/1 ถึง f0/15 ต่อเข้า PC (ทำเป็น Access VLAN 10)
interface range f0/1-15
switchport mode access
switchport access vlan 10
exit
```