# การ Update Kernel ubuntu
- Kernel version ล่าสุด  (3 กันยายน 2025)
![](./images/2_updatekernel2.png)
[https://kernel.org/](https://kernel.org/)

## Ubuntu 24.04

### Step 1: Check Current kernel Version
```
$ uname -r
6.8.0-55-generic
```

### Step 2: Update Package list
```
sudo apt update -y
```

### Step 3: Upgrade Package
```
sudo apt upgrade -y

sudo apt  install pkexec -y
uname -r
```

### Step 4: Reboot
```
$ sudo reboot
```

### Step 5: Install mainline kernel in Ubuntu โดย ติดตั้งจาก unofficial PPA
```
$ sudo add-apt-repository ppa:cappelikan/ppa
$ sudo apt update && sudo apt install mainline
$ sudo apt upgrade -y
```

```
$ mainline check
$ sudo mainline install-latest
$ sudo update-grub
```

![](./images/2_updatekernel7.png)
- kernel verion  (6.16.4-061604.202508281650) 3/9/2025

- **Reboot** เพื่อโหลด Kernel
```
sudo reboot
```

### Step6 Check Version
```
$ uname -r
$ sudo apt install linux-headers-$(uname -r)
```

### Step 7 

![](./images/2_updatekernel1.png)

---
## Build Kernel

```
$ sudo apt install -y build-essential libncurses-dev bison flex libssl-dev libelf-dev fakeroot bc
$ sudo apt install -y dwarves
```

### Clone Linux kernel และ บริหารจัดการ Branch จาก Git
```
sudo su -
mkdir /usr/src/build
cd /usr/src/build
git clone  git://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git
cd linux
git branch
git tag -l

```

### Generate Default config
```
# uname -r
6.16.4-061604-generic

# cp -v /boot/config-$(uname -r) .config
'/boot/config-6.16.4-061604-generic' -> '.config'

# yes '' | make oldconfig

# make clean
```

### Disable kernel parameter
[https://wiki.debian.org/BuildADebianKernelPackage](https://wiki.debian.org/BuildADebianKernelPackage)

```
scripts/config --set-str SYSTEM_TRUSTED_KEYS ""
scripts/config --set-str SYSTEM_REVOCATION_KEYS  ""
```

```
# make -j$(nproc)
# make -j$(nproc) modules_install
# make -j$(nproc) install
# make -j$(nproc) bindeb-pkg

apt install ../*.deb
```

### สามารถรวมคำสั่งพร้อมการจับเวลา และ สร้าง log
```
time make -j$(nproc) 2>&1 | tee build-0.log
time make -d modules_install install 2>&1 | tee make-install-0.log
```

### Reboot
```
reboot
```

![](./images/2_updatekernel3.png)



1. ``2>&1`` means "redirect standard error to the same place as standard output." This ensures that both regular progress messages and error messages are treated as a single stream of data.
2. tee build.log: The tee command is named after a T-splitter in plumbing. It takes the input it receives and splits it into two directions:
    1. It prints the output to the screen (standard output), so you can watch the build progress in real time.
    2. It saves a copy of the output to the specified file, build.log.

### update bootloader สำหรับการยอมรับ Kernel ใหม่
```
sudo update-grub
```


สรุปสั้นๆ คือ `modules_install` เป็นการติดตั้ง **"ไดรเวอร์และส่วนประกอบเสริม"** ส่วน `install` เป็นการติดตั้ง **"แกนหลักของเคอร์เนลและตั้งค่าการบูต"** ครับ

---

### **1. `sudo make modules_install` (ติดตั้งส่วนประกอบเสริม)**

คำสั่งนี้มีหน้าที่นำ **โมดูล (Modules)** ทั้งหมดที่คอมไลพ์เสร็จแล้วไปติดตั้งในไดเรกทอรีของระบบ (`/lib/modules/<เวอร์ชันเคอร์เนลใหม่>`)

* **โมดูลคืออะไร?** เปรียบเสมือน **ไดรเวอร์** สำหรับอุปกรณ์ต่างๆ เช่น ไดรเวอร์การ์ดจอ, Wi-Fi, Bluetooth, เสียง หรือระบบไฟล์ต่างๆ
* **ทำไมต้องติดตั้ง?** เพื่อให้เคอร์เนลตัวใหม่ที่คุณกำลังจะใช้งาน สามารถเรียกใช้ไดรเวอร์เหล่านี้เพื่อสื่อสารกับฮาร์ดแวร์ในเครื่องคอมพิวเตอร์ของคุณได้

**สรุปหน้าที่:** คัดลอกไฟล์ไดรเวอร์ทั้งหมดไปไว้ในที่ที่ระบบปฏิบัติการพร้อมเรียกใช้งาน

![](./images/2_updatekernel4.png)

![](./images/2_updatekernel5.pngcl)
---

### **2. `sudo make install` (ติดตั้งแกนหลัก)**

หลังจากติดตั้งส่วนประกอบเสริมแล้ว คำสั่งนี้จะทำการติดตั้ง **แกนหลัก (core) ของเคอร์เนล** และทำให้ระบบพร้อมที่จะบูตเข้าเคอร์เนลตัวใหม่นี้ได้

โดยคำสั่งนี้จะทำงานหลักๆ 2-3 อย่าง:
1.  **คัดลอกไฟล์เคอร์เนลหลัก:** นำไฟล์เคอร์เนลที่คอมไพล์เสร็จแล้ว (เช่น `vmlinuz-...`) ไปไว้ที่ไดเรกทอรี `/boot` ซึ่งเป็นที่เก็บไฟล์สำหรับบูตเครื่อง
2.  **คัดลอกไฟล์ที่เกี่ยวข้อง:** นำไฟล์อื่นๆ ที่จำเป็น เช่น `System.map` และ `.config` ไปไว้ที่ `/boot` ด้วย
3.  **อัปเดต Bootloader:** ขั้นตอนที่สำคัญที่สุดคือ **สั่งอัปเดต GRUB (Bootloader) โดยอัตโนมัติ** เพื่อเพิ่มรายชื่อเคอร์เนลตัวใหม่เข้าไปในเมนูตอนเปิดเครื่อง

**สรุปหน้าที่:** ติดตั้งไฟล์เคอร์เนลตัวหลักและอัปเดตเมนูบูตเพื่อให้คุณสามารถเลือกใช้งานเคอร์เนลใหม่ตอนเปิดเครื่องได้



### **เปรียบเทียบง่ายๆ**

ถ้าการคอมไพล์เคอร์เนลเหมือนการ **"สร้างรถยนต์คันใหม่"**:
* `make modules_install` คือการ **"ติดตั้งอุปกรณ์เสริม"** เช่น ระบบเครื่องเสียง, แอร์, GPS เข้าไปในรถ
* `make install` คือการ **"นำรถไปจอดในโรงรถและทำกุญแจให้"** เพื่อให้คุณพร้อมที่จะสตาร์ทและขับรถคันนั้นได้


### List Kernel Module
```
# lsmod
```

![](./images/2_updatekernel6.png)