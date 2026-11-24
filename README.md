# Halloy-but-Remember-the-history
## Halloy + soju on CachyOS

**A reusable, backup-first IRC setup for CachyOS / Arch Linux**  
**ภาษา:** คู่มือนี้เขียนภาษาไทยเป็นหลัก  
**รูปแบบ:** Halloy native package + soju system service + Libera.Chat + SQLite message store

> This is a community setup guide, not an official Halloy, soju, CachyOS, or Libera.Chat project.

## เป้าหมาย

- เปิด Halloy เพื่ออ่านและส่งข้อความตามปกติ
- ให้ soju เชื่อมต่อ IRC และอยู่ในห้องต่อ แม้ปิด Halloy
- บันทึกข้อความที่ soju ได้รับไว้ใน SQLite
- เปิด Halloy กลับมาแล้วขอประวัติย้อนหลังผ่าน IRCv3 CHATHISTORY
- ปิด/เปิดเครื่องแล้ว soju เริ่มเองและเชื่อมต่อเครือข่าย/ห้องเดิมอีกครั้ง
- มีสคริปต์ตรวจสุขภาพ สำรองข้อมูลตามเวลา และกู้คืนจาก backup
- อัปเดต CachyOS แบบ full upgrade พร้อม backup ก่อนทำ

## ขอบเขตและข้อจำกัดที่ต้องเข้าใจก่อน

1. **คอมพิวเตอร์ปิด = soju หยุดรับข้อความ** หากต้องการรับข้อความระหว่างปิดเครื่อง ต้องย้าย soju ไป VPS/เครื่อง server ที่ออนไลน์ตลอดเวลา
2. ประวัติที่เก็บได้คือข้อความที่ Halloy หรือ soju ได้รับจริง ข้อความที่เกิดระหว่างไม่มี client/bouncer เชื่อมต่อ ไม่สามารถรับประกันว่าจะกู้ได้
3. SQLite ของ Halloy (`history.sqlite3`) และ SQLite ของ soju (`main.db`) เป็นคนละฐานข้อมูล ทำหน้าที่ต่างกัน ต้อง backup ทั้งคู่
4. การใช้ `listen irc+insecure://127.0.0.1:6667` ปลอดภัยเฉพาะการเชื่อมต่อภายในเครื่องเดียว เพราะ bind เฉพาะ loopback และไม่มี TLS อย่าเปลี่ยนเป็น `0.0.0.0` หรือเปิด port 6667 ออกอินเทอร์เน็ตด้วย config นี้
5. ชุดสคริปต์นี้ตั้งเป้า **Halloy แบบ native Linux** ไม่ใช่ Flatpak หากใช้ Flatpak ตำแหน่ง config/data ต่างออกไป ต้องปรับ path ก่อนใช้
6. การ backup ไม่ใช่การ sync และ backup ที่อยู่บนดิสก์เดียวกับเครื่องหลักไม่ช่วยเมื่อดิสก์เสีย ควรคัดลอก backup ไป external drive อีกชุด
7. อย่าใส่รหัสผ่าน, SQLite, log, export ฐานข้อมูล, หรือไฟล์ config ที่มี secret ลง public GitHub

## ค่าที่ต้องแก้ก่อนใช้

| ตัวแปร | ตัวอย่าง | ความหมาย |
|---|---|---|
| `SOJU_USER` | `youruser` | ชื่อผู้ใช้ Bouncer (ไม่ใช่ Linux user) |
| `IRC_NICK` | `YourNick` | IRC nickname ที่ใช้บน Libera |
| `NETWORK` | `liberachat` | ชื่อ network ที่บันทึกใน soju |
| `SOJU_DB` | `/var/lib/soju/main.db` | ไฟล์ฐานข้อมูล soju |
| `HALLoy_CONFIG` | `~/.config/halloy/config.toml` | config ของ Halloy (native) |
| `HALLoy_DATA` | `~/.local/share/halloy` | data/history ของ Halloy (native) |

ตัวอย่างค่าของผู้เขียนเดิมไม่ถูกฝังไว้ใน template เพื่อไม่เผยชื่อบัญชีส่วนตัวใน public repo

**ชื่อ Bouncer user เปลี่ยนภายหลังไม่ได้** ตามข้อกำหนดของ soju; ถ้าต้องเปลี่ยน `SOJU_USER` ต้องสร้าง user ใหม่และย้าย/ตั้งค่า network ใหม่ อย่าแก้เฉพาะใน `config.toml` แล้วคาดว่าจะเป็น user เดิม ส่วน IRC nickname เปลี่ยนได้โดยแก้ nick ของ network ใน soju และ Halloy ให้ตรงกัน

---

# A. ติดตั้งใหม่บน CachyOS

## A1. ติดตั้ง soju และเครื่องมือฐานข้อมูล

```fish
sudo pacman -S --needed halloy soju sqlite python
```

คำสั่งด้านบนติดตั้ง Halloy และ soju จาก repository ของ CachyOS (หาก repository ในเครื่องไม่มีแพ็กเกจดังกล่าว ให้ตรวจ package search ของ CachyOS ก่อนเลือกช่องทางอื่น) และตรวจให้แน่ใจว่าใช้ native data directory ไม่ใช่ Flatpak. หลังติดตั้ง ตรวจเวอร์ชัน:

```fish
halloy --version
soju --version
sojuctl -help
```

หาก `soju --version` ไม่รองรับในแพ็กเกจที่ติดตั้ง ให้ใช้ `pacman -Qi soju` แทน

## A2. ตั้งค่า soju แบบ local-only

สำรองไฟล์ config เดิมก่อนแก้:

```fish
sudo test ! -e /etc/soju/config; or sudo cp -a /etc/soju/config /etc/soju/config.before-halloy-soju
```

เขียน `/etc/soju/config` ด้วยค่า local-only นี้:

```fish
printf '%s\n' \
  'listen irc+insecure://127.0.0.1:6667' \
  'listen unix+admin://' \
  'hostname localhost' \
  'db sqlite3 /var/lib/soju/main.db' \
  'message-store db' \
  | sudo tee /etc/soju/config >/dev/null
```

สิ่งที่ config นี้ทำ:

- IRC client ฟังเฉพาะ `127.0.0.1:6667` ไม่เปิดรับจาก LAN/Internet
- เปิด Unix admin socket สำหรับ `sojuctl`
- เก็บ user/network/channel state ใน `/var/lib/soju/main.db`
- ใช้ `message-store db` เพื่อเก็บประวัติที่ soju รับได้และรองรับ CHATHISTORY
- ไม่ได้ตั้ง `message-expiry` ดังนั้น soju จะไม่กำหนดวันหมดอายุของข้อความเอง (ค่าปริยายคือเก็บไว้ไม่มีกำหนด) แต่ควรเฝ้าดูพื้นที่ดิสก์

**ห้ามใช้ config นี้เพื่อเปิดให้เชื่อมต่อจากมือถือ/อินเทอร์เน็ตโดยตรง** ต้องตั้ง TLS และ firewall แยกต่างหากก่อน

## A3. สร้าง user soju ครั้งแรกเท่านั้น

บนเครื่องใหม่ที่ยังไม่มีฐานข้อมูล/user ให้ตรวจว่า service ยังไม่ทำงาน (`sudo systemctl stop soju`) แล้วสร้าง admin user ก่อน start service ตามคู่มือ soju:

```fish
sudo sojudb create-user YOUR_SOJU_USERNAME -admin
```

แทน `YOUR_SOJU_USERNAME` ด้วยชื่อผู้ใช้ Bouncer ที่ต้องการ แล้วทำตาม prompt ของโปรแกรมเพื่อกำหนดรหัสผ่าน

> **อย่ารันคำสั่งนี้ซ้ำบนเครื่องที่มี user หรือฐานข้อมูลเดิมอยู่แล้ว** ห้ามลบ `/var/lib/soju/main.db` เพื่อเริ่มใหม่หากยังต้องการข้อมูลเดิม

เปิด service และให้เริ่มเองเมื่อบูต:

```fish
sudo systemctl enable --now soju
systemctl is-active soju
systemctl is-enabled soju
```

ผลที่ต้องการคือ `active` และ `enabled`

## A4. เพิ่ม Libera.Chat ใน soju

ใช้คำสั่ง `sojuctl user run` โดยแทนชื่อ Bouncer และ nick:

```fish
sudo sojuctl user run YOUR_SOJU_USERNAME network create -addr ircs://irc.libera.chat -name liberachat -nick YOUR_IRC_NICK
```

หากแจ้งว่า network `liberachat` มีอยู่แล้ว **อย่าสร้างซ้ำ** ให้ตรวจแทน:

```fish
sudo sojuctl user run YOUR_SOJU_USERNAME network status
```

ดูห้องที่ soju จำไว้:

```fish
sudo sojuctl user run YOUR_SOJU_USERNAME channel status -network liberachat
```

การเข้า channel ให้ทำจาก Halloy ด้วย `/join #channel` เพื่อให้ soju บันทึกห้องไว้

## A5. ตั้ง Halloy

สร้าง config จากตัวอย่างใน `examples/halloy-config.toml` โดยแก้ชื่อ user และ nickname ให้ตรงกัน หรือใช้สคริปต์:

```fish
bash scripts/setup-halloy.sh
```

หากมี `config.toml` อยู่แล้ว สคริปต์จะสำรองไฟล์เดิมและหยุดโดยไม่เขียนทับ ให้ตรวจไฟล์สำรองก่อน แล้วจึงรันโดยตั้งใจแทนที่:

```fish
bash scripts/setup-halloy.sh --replace
```

การแทนที่นี้จะสร้าง config ขั้นต่ำ จึงไม่ merge theme, extra servers หรือ preferences อื่นจากไฟล์เดิม แต่เก็บสำเนาเดิมไว้ให้กู้/merge เอง

สคริปต์จะ:

- สำรอง config เก่าก่อนเขียน
- ถามชื่อ soju user และ IRC nickname
- สร้าง config แบบ automatic `bouncer-networks`
- ใช้ `password_keyring = true` แทนการเขียนรหัสผ่านลง public config
- เปิด `restore_on_launch`
- ปิด shortcut `Ctrl+Shift+W` (`leave_buffer`) เพื่อช่วยป้องกันการออกจากห้องโดยไม่ตั้งใจ

เมื่อเปิด Halloy ครั้งแรก ให้กรอกรหัสผ่าน **soju** (ไม่ใช่รหัส NickServ) เมื่อ Halloy ขอ และให้ระบบ Secret Service ของ Linux จัดเก็บใน keyring

```fish
halloy
```

Halloy ควรแสดง network `liberachat` ใต้ server `soju` โดยอัตโนมัติ หากไม่เห็น ให้ตรวจ log/config ก่อน อย่ารีบสร้าง network ซ้ำ

## A6. ล็อกอิน NickServ (ถ้า nickname ลงทะเบียนไว้)

รหัสผ่าน Bouncer กับรหัสผ่าน NickServ เป็นคนละรหัสกัน หาก nickname เป็นบัญชี Libera ที่ลงทะเบียนไว้ ให้ตั้ง SASL ที่ soju ด้วยสคริปต์ซึ่งรับรหัสผ่านแบบไม่แสดงบนหน้าจอ:

```fish
bash scripts/set-network-sasl.sh YOUR_SOJU_USERNAME YOUR_IRC_NICK liberachat
```

คำสั่งนี้ใช้รูปแบบ BouncerServ `sasl set-plain -network <network> <username> <password>` ตาม soju manual และไม่เขียนรหัสผ่านลงไฟล์ config หรือ shell history โดยตรง หลังตั้งค่าให้ตรวจ:

```fish
sudo sojuctl user run YOUR_SOJU_USERNAME sasl status -network liberachat
sudo sojuctl user run YOUR_SOJU_USERNAME network status
```

ห้ามใส่รหัส NickServ ในไฟล์ public หรือส่งรหัสผ่านในภาพ Terminal/issue tracker

---

# B. การใช้งานประจำวัน

## คำสั่งในช่องแชต Halloy (ไม่ใช่ Terminal)

| คำสั่ง | ผล |
|---|---|
| `/join #channel` | เข้าห้อง และ soju จะจำการ Join |
| `/detach` | ซ่อน/Detach ห้อง โดยไม่ออกจาก IRC channel; soju ยังอยู่ในห้อง |
| `/part` | ออกจากห้องจริง อย่าใช้ถ้าต้องการให้ soju รับข้อความต่อ |
| `/msg BouncerServ network status` | ดูสถานะ network ของ soju |
| `/msg BouncerServ channel status -network liberachat` | ดูรายการห้องที่ soju บันทึก |
| `/msg BouncerServ help` | ดูคำสั่ง BouncerServ ของเวอร์ชันที่ใช้อยู่ |
| `/search` | เปิดการค้นหาประวัติใน Halloy (ถ้ารุ่นที่ติดตั้งรองรับ) |

`/detach` ไม่ได้แปลว่าออกจากห้อง ส่วน `/part` คือการออกจริง

## ปุ่ม X และ keyboard

Halloy แยก action `close_buffer` (ปิด buffer/pane) กับ `leave_buffer` (ออกจาก channel หรือปิด query) ออกจากกัน

- `Ctrl+W` เป็น shortcut ของ `close_buffer`
- `Ctrl+Shift+W` เป็น shortcut ของ `leave_buffer`
- template นี้ตั้ง `leave_buffer = "noop"` เพื่อปิด shortcut `Ctrl+Shift+W`

ปุ่ม X บนหัว pane ในหน้าต่าง Halloy เป็นปุ่มปิด buffer ตาม UI ที่แสดงในภาพตัวอย่างของผู้ใช้ อย่างไรก็ตามปุ่ม X ของ widget/ตำแหน่งอื่นอาจมี action ต่างกัน อย่าคิดว่าทุกกากบาทเหมือนกัน หากไม่แน่ใจ ให้ใช้ `/detach` แทน

## ปิดโปรแกรม/ปิดเครื่อง

- ปิด Halloy ทั้งโปรแกรมได้: soju เป็น system service แยกและยังทำงานต่อ
- ปิดเครื่องได้: `systemd` จะเริ่ม soju ใหม่หลังบูตเพราะตั้ง `enabled` แล้ว
- ตอนคอมพิวเตอร์ปิด soju ก็หยุดรับข้อความ ไม่มีระบบใดรับข้อความแทนในช่วงนั้น
- เมื่อเปิดเครื่อง ให้ตรวจ `systemctl is-active soju` และ `channel status` ก่อนเปิด Halloy ถ้าต้องการตรวจความผิดปกติ

---

# C. ตรวจสุขภาพระบบ

ใช้สคริปต์:

```fish
bash scripts/healthcheck.sh
```

หรือรันคำสั่งทีละบรรทัด:

```fish
systemctl is-active soju
systemctl is-enabled soju
sudo sojuctl user run YOUR_SOJU_USERNAME network status
sudo sojuctl user run YOUR_SOJU_USERNAME channel status -network liberachat
ss -lnt | grep 6667
```

ผลที่คาดหวัง:

- soju = `active`
- soju = `enabled`
- network `liberachat` = connected
- channel status แสดง channel ที่ต้องการเก็บ
- listener แสดง `127.0.0.1:6667` ไม่ใช่ `0.0.0.0:6667`

Log:

```fish
journalctl -u soju -n 100 --no-pager
```

---

# D. Backup และ Restore

## D1. สำรองทันที

```fish
bash scripts/backup.sh
```

สำรองข้อมูลต่อไปนี้:

- Halloy `history.sqlite3` ผ่าน SQLite Online Backup
- soju `/var/lib/soju/main.db` ผ่าน SQLite Online Backup
- Halloy `config.toml`
- soju `/etc/soju/config`
- เวอร์ชันแพ็กเกจใน manifest

สคริปต์จะเก็บไว้ที่ `~/Backups/halloy-soju/<timestamp>/` และตั้ง permission ให้เป็นส่วนตัว

สามารถกำหนดตำแหน่ง external drive ได้ เช่น:

```fish
set -x IRC_BACKUP_DIR /run/media/$USER/BackupDisk/HalloySoju
bash scripts/backup.sh
set -e IRC_BACKUP_DIR
```

เปลี่ยน `BackupDisk` ให้ตรงกับชื่อ mount ของดิสก์จริง ตรวจ path ให้ถูกก่อนรัน

## D2. ตั้ง Backup อัตโนมัติรายวัน

```fish
sudo bash scripts/install-backup-timer.sh
```

สคริปต์ติดตั้ง systemd timer ที่รัน backup ทุกวันเวลา 03:20 (เวลาท้องถิ่น) และใช้ `Persistent=true` เพื่อให้รันเมื่อกลับมาเปิดเครื่อง หากพลาดเวลาเพราะคอมปิดอยู่

ตรวจ timer:

```fish
systemctl list-timers halloy-soju-backup.timer
systemctl status halloy-soju-backup.timer
```

เรียก backup ด้วยมือผ่าน systemd:

```fish
sudo systemctl start halloy-soju-backup.service
journalctl -u halloy-soju-backup.service -n 80 --no-pager
```

**สำคัญ:** daily backup ในดิสก์ภายในเครื่องยังไม่ใช่ off-device backup ให้คัดลอกโฟลเดอร์ backup ไป external drive เป็นระยะ

## D3. ตรวจ backup

```fish
find ~/Backups/halloy-soju -maxdepth 2 -type f -printf '%p %s bytes\n' | sort
```

ไฟล์ SQLite ในแต่ละชุดควรเปิดตรวจได้:

```fish
sqlite3 ~/Backups/halloy-soju/ชื่อโฟลเดอร์/halloy-history.sqlite3 'PRAGMA integrity_check;'
sqlite3 ~/Backups/halloy-soju/ชื่อโฟลเดอร์/soju-main.sqlite3 'PRAGMA integrity_check;'
```

ผลที่ต้องการคือ `ok`

## D4. Restore

**Restore เขียนทับข้อมูลปัจจุบัน เป็นขั้นตอนอันตราย** ให้ใช้เมื่อจำเป็นจริง ๆ และเลือก backup ที่ตรวจ `integrity_check` ผ่านแล้วเท่านั้น

```fish
bash scripts/restore-from-backup.sh ~/Backups/halloy-soju/ชื่อโฟลเดอร์
```

สคริปต์จะ:

- ปฏิเสธการทำงานถ้า Halloy ยังเปิดอยู่
- สร้าง backup ปัจจุบันก่อน restore
- หยุด soju ก่อนแทนที่ฐานข้อมูล
- เก็บไฟล์ปัจจุบันไว้เป็น `.pre-restore-*`
- ตรวจไฟล์ SQLite ก่อนนำไปใช้
- ใช้การยืนยันคำว่า `RESTORE` ก่อนเขียนทับ
- เริ่ม soju กลับเมื่อจบ

หลัง restore ให้เปิด Halloy และตรวจ history/network/channel status ด้วยตนเอง

---

# E. อัปเดต CachyOS / เปลี่ยนเวอร์ชัน

CachyOS เป็น rolling-release distribution อย่าทำ partial upgrade (`pacman -Sy package`) ให้ทำ full system upgrade แทน

## E1. ก่อนอัปเดตทุกครั้ง

```fish
bash scripts/backup.sh
pacman -Q halloy soju sqlite
systemctl is-active soju
sudo sojuctl user run YOUR_SOJU_USERNAME network status
sudo sojuctl user run YOUR_SOJU_USERNAME channel status -network liberachat
```

ตรวจว่า backup สำเร็จจริงและมีขนาดไม่ใช่ 0 bytes ก่อนอัปเดต

## E2. อัปเดต

กรณีใช้ pacman กับแพ็กเกจจาก official repository:

```fish
sudo pacman -Syu
```

ถ้า Halloy เป็น AUR package และคุณจัดการ AUR ด้วย paru ให้ใช้:

```fish
paru -Syu
```

ถ้า Halloy เป็น Flatpak ให้ใช้ `flatpak update` แต่จำไว้ว่าคู่มือนี้ตั้งค่า path สำหรับ native package เป็นหลัก

**ห้าม** ใช้ `sudo pacman -Sy halloy`, `sudo pacman -Sy soju` หรือ `pacman -Sy` แล้วค่อยติดตั้งแพ็กเกจแยก เพราะเป็น partial upgrade

## E3. หลังอัปเดต

```fish
pacman -Q halloy soju sqlite
sudo systemctl restart soju
systemctl is-active soju
sudo sojuctl user run YOUR_SOJU_USERNAME network status
sudo sojuctl user run YOUR_SOJU_USERNAME channel status -network liberachat
```

เปิด Halloy ตรวจว่า network และประวัติทำงาน แล้วทดสอบค้นข้อความเก่า

**อย่าดาวน์เกรด Halloy/soju หรือ restore database รุ่นใหม่ลง binary รุ่นเก่าโดยเดา** หากอัปเดตทำ schema migration แล้ว ต้องเลือก restore package+database snapshot ที่เป็นชุดเดียวกัน หรือใช้รุ่นใหม่ที่เข้ากันได้

---

# F. ย้ายไปเครื่องใหม่

1. ที่เครื่องเก่า สั่ง backup และตรวจ integrity ให้ผ่าน
2. คัดลอก backup folder ไป external drive หรือเครื่องใหม่ ห้ามใช้ GitHub public repo เป็นที่เก็บฐานข้อมูล/รหัสผ่าน
3. ติดตั้ง `soju`, `sqlite`, `python` และ Halloy native บนเครื่องใหม่
4. สร้าง `/etc/soju/config` จากไฟล์ backup หรือ `examples/soju-config`
5. **ก่อนเปิด Halloy/soju ครั้งแรกบนเครื่องใหม่** หยุด service แล้ว restore `soju-main.sqlite3` ไปที่ `/var/lib/soju/main.db` โดยรักษา owner/group ของ service user ให้ถูกต้อง
6. คืน `history.sqlite3` ไปที่ `~/.local/share/halloy/history.sqlite3` ขณะที่ Halloy ปิดอยู่
7. คืน `config.toml` หรือสร้างใหม่ด้วย `scripts/setup-halloy.sh`
8. เปิด soju แล้วตรวจ network/channel status
9. เปิด Halloy แล้วกรอกรหัส Bouncer ใหม่ใน system keyring หาก keyring ไม่ได้ย้ายมาด้วย
10. อย่าลบเครื่องเก่าหรือ backup ต้นฉบับจนกว่าจะยืนยันว่าข้อความเก่า ห้อง และการเชื่อมต่อทำงานครบ

หากชื่อ Linux user/home directory เปลี่ยน ให้ปรับ path และ ownership ก่อน restore

---

# G. Troubleshooting

## Halloy แจ้ง lock acquisition failed

อย่าลบ `history.lock` ด้วยมือ:

```fish
pgrep -af halloy
```

ปิด Halloy จากเมนู Quit ให้หมดก่อน แล้วเปิดเพียง instance เดียว

## soju ไม่ start

```fish
systemctl status soju --no-pager
journalctl -u soju -n 100 --no-pager
```

## network ไม่ connect

```fish
sudo sojuctl user run YOUR_SOJU_USERNAME network status
journalctl -u soju -n 100 --no-pager
```

ตรวจ Internet/DNS, NickServ/SASL, network enabled และ log upstream

## ห้องไม่กลับมา

```fish
sudo sojuctl user run YOUR_SOJU_USERNAME channel status -network liberachat
```

หากห้องไม่อยู่ในรายการ ต้อง Join ใหม่จาก Halloy ด้วย `/join #channel` จากนั้นตรวจ status ซ้ำ

## Halloy history ดูเหมือนหาย

- อย่าลบ `history.sqlite3`, `history.sqlite3-wal`, `history.sqlite3-shm`, `history.lock`
- อย่าเปิด Halloy หลาย instance
- ตรวจว่าใช้ native หรือ Flatpak data directory ให้ถูก
- สำรองไฟล์ก่อนตรวจ/ซ่อมฐานข้อมูล
- ตรวจด้วย `sqlite3 <db> 'PRAGMA integrity_check;'` เมื่อแน่ใจว่าใช้ path ถูกต้อง

---

# H. GitHub hygiene

ก่อน public repository ให้ตรวจว่าไม่มี:

- NickServ password หรือ soju password
- `config.toml` จริงของเครื่องส่วนตัว
- `/var/lib/soju/main.db`
- `history.sqlite3`, `-wal`, `-shm`
- log ที่อาจมี nick, network, private messages หรือรายละเอียดส่วนตัว
- backup archive

ไฟล์ `.gitignore` ใน repository นี้ตั้งค่าป้องกันชื่อไฟล์เหล่านี้ไว้แล้ว แต่ให้ตรวจ `git status` ก่อน `git push` ทุกครั้ง

```fish
git status --short
```

---

# References

- Halloy configuration: https://halloy.chat/configuration
- Halloy Connect with soju: https://halloy.chat/guides/connect-with-soju
- Halloy Servers / SASL / keyring: https://halloy.chat/configuration/servers
- Halloy Keyboard actions: https://halloy.chat/configuration/keyboard
- Halloy Buffer / chathistory: https://halloy.chat/configuration/buffer
- soju getting started: https://github.com/emersion/soju/blob/master/doc/getting-started.md
- soju manual (Arch package): https://man.archlinux.org/man/extra/soju/soju.1.en
- sojuctl manual: https://man.archlinux.org/man/sojuctl.1.en
- SQLite Online Backup API: https://www.sqlite.org/backup.html
- Arch system maintenance / partial upgrades: https://wiki.archlinux.org/title/System_maintenance
- CachyOS post-install/update docs: https://wiki.cachyos.org/configuration/post_install_setup/

## Publish this guide to GitHub

สร้าง repository ใหม่บน GitHub (แนะนำ public สำหรับคู่มือ แต่ห้ามใส่ข้อมูลส่วนตัว/backup) จากนั้นในโฟลเดอร์โปรเจกต์รัน:

```fish
git init
git add README.md LICENSE SECURITY.md .gitignore examples scripts
git status --short
```

ตรวจ `git status --short` ให้แน่ใจว่าไม่มี `config.toml`, database, log, password หรือ backup ก่อน commit:

```fish
git commit -m "Add Halloy and soju CachyOS guide"
git branch -M main
git remote add origin git@github.com:YOUR_GITHUB_NAME/halloy-soju-cachyos.git
git push -u origin main
```

แทน `YOUR_GITHUB_NAME` และสร้าง repository ชื่อ `halloy-soju-cachyos` ไว้ก่อน หากใช้ HTTPS แทน SSH ให้คัดลอก remote URL จากหน้า repository ของคุณ

## Version notes

- The official Halloy documentation currently exposes the 2026.10 configuration reference. Re-check docs before copying the config into a later major/minor release.
- The soju package and its command flags can change. Run `man soju`, `man sojuctl`, `/msg BouncerServ help`, and `/msg BouncerServ help sasl` on the installed version before using advanced commands.
- This repo intentionally does not pin package versions. On a rolling release, use full system upgrades and backups rather than holding old libraries indefinitely.
