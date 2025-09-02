# MakeFile

### **Makefile: พิมพ์เขียวของการคอมไพล์**

``Makefile`` เป็นไฟล์ข้อความธรรมดาที่อยู่ในไดเรกทอรีของซอร์สโค้ด มีหน้าที่หลักคือการกำหนด "กฎ (Rules)" สำหรับการสร้างโปรเจกต์

โครงสร้างพื้นฐานของกฎใน Makefile คือ:

```
target: dependencies
    commands
```
- **Target (เป้าหมาย):** คือสิ่งที่เราต้องการสร้าง เช่น ไฟล์เคอร์เนลหลัก ``(vmlinuz)``, ``โมดูล (.ko)``, หรือแม้แต่การทำงานบางอย่างเช่น install หรือ clean

- **ependencies (สิ่งที่ต้องใช้):** คือไฟล์หรือเป้าหมายอื่นๆ ที่ต้องมีหรือต้องทำให้เสร็จก่อนที่จะสร้าง ``target`` ได้ เช่น ไฟล์ซอร์สโค้ด (.c) หรือไฟล์ object (.o)

- **Commands (คำสั่ง):**  คือชุดคำสั่ง Shell Script (เช่น ``gcc``, ``cp``, ``rm``) ที่จะถูกรันเพื่อสร้าง target ขึ้นมาจาก ``dependencies``

ใน Linux Kernel นั้น ``Makefile`` หลักจะมีขนาดใหญ่มาก และจะมีการเรียกใช้ ``Makefile`` ย่อยๆ ที่กระจายอยู่ในแต่ละโฟลเดอร์ของซอร์สโค้ดอีกทอดหนึ่ง

---

### **make: โปรแกรมสร้าง (Builder)**

make เป็นโปรแกรมที่เราเรียกใช้ผ่าน Command Line มันจะวิ่งเข้ามาอ่านไฟล์ Makefile ในไดเรกทอรีนั้นๆ แล้วทำงานตามกฎที่เขียนไว้

กระบวนการทำงานของ ``make`` เป็นดังนี้:

1. เมื่อคุณรันคำสั่ง เช่น make -j$(nproc) หรือ make modules_install

1. โปรแกรม make จะมองหาไฟล์ชื่อ Makefile ในโฟลเดอร์ปัจจุบัน

1. make จะไปดูกฎของ target ที่คุณระบุ (ถ้าไม่ระบุจะใช้ target แรกสุด ซึ่งมักจะเป็น all)

1. make จะตรวจสอบเวลาของไฟล์ ``(Timestamp)`` มันจะเปรียบเทียบว่า dependencies มีไฟล์ไหนใหม่กว่า target หรือไม่

1. ถ้า dependencies ใหม่กว่า หรือ target ยังไม่มีอยู่จริง make ก็จะรัน commands ที่กำหนดไว้เพื่อสร้าง target ขึ้นมาใหม่

1. ถ้า target ใหม่กว่า dependencies ทั้งหมด make จะถือว่าไฟล์นั้นทันสมัยแล้วและจะ "ไม่ทำอะไรเลย" ซึ่งนี่คือหัวใจสำคัญที่ทำให้การคอมไพล์ครั้งถัดๆ ไปรวดเร็วขึ้นมาก เพราะมันจะทำใหม่เฉพาะส่วนที่มีการเปลี่ยนแปลงเท่านั้น

!!! info
    ``Makefile`` หลักสามารถเรียกใช้ Makefile อื่นๆ ที่อยู่ในโฟลเดอร์ย่อย (subdirectories) ได้ และนี่คือวิธีการมาตรฐานสำหรับจัดการโปรเจกต์ขนาดใหญ่ที่มีซอร์สโค้ดหลายส่วน เช่น Linux Kernel


## **Makefile เรียก Makefile ในโฟลเดอร์ย่อยได้อย่างไร**

หลักการนี้เรียกว่า **"Recursive Make"** ซึ่งเป็นหัวใจสำคัญที่ทำให้สามารถจัดการโปรเจกต์ที่ซับซ้อนได้ โดยการแบ่งงานสร้าง (build) ออกเป็นส่วนๆ ตามโครงสร้างของโฟลเดอร์

### **วิธีการทำงาน**

`Makefile` ที่อยู่ชั้นบนสุด (Top-level Makefile) จะไม่เก็บรายละเอียดการคอมไพล์ของทุกไฟล์ แต่จะมอบหมายหน้าที่ต่อไปยัง `Makefile` ที่อยู่ในโฟลเดอร์ย่อยๆ แทน โดยใช้คำสั่งพิเศษ

คำสั่งที่ใช้คือ:
```makefile
$(MAKE) -C <subdirectory> <target>
```

**ตัวอย่าง ง่ายๆ**
สมมุติว่า เรามีโครงสร้างโปรเจต์ ดังนี้
```bash
project/
├── Makefile        # Makefile หลัก
├── src/
│   ├── main.c
│   └── Makefile    # Makefile สำหรับ src
└── lib/
    ├── utils.c
    └── Makefile    # Makefile สำหรับ lib
```

- จะได้โครงสร้างของ Makefile หลัก โครงสร้างประมาณนี้ 
```
# กำหนดโฟลเดอร์ย่อยที่จะเข้าไปทำงาน
SUBDIRS = src lib

# เป้าหมายหลัก 'all' จะไปสั่งให้ 'make all' ในโฟลเดอร์ย่อยทำงาน
all:
    @echo "Starting build from top-level Makefile..."
    @for dir in $(SUBDIRS); do \
        $(MAKE) -C $$dir; \
    done
    @echo "Top-level build finished."

# เป้าหมาย 'clean' ก็จะไปสั่ง 'make clean' ในโฟลเดอร์ย่อยเช่นกัน
clean:
    @echo "Starting clean from top-level Makefile..."
    @for dir in $(SUBDIRS); do \
        $(MAKE) -C $$dir clean; \
    done
    @echo "Top-level clean finished."
```

- ส่วนในไฟล์ Folder ย่อยจะมีโครงสร้าง ตัวอย่างดังนี้
```
# Makefile นี้รู้แค่เรื่องการ build ไฟล์ในโฟลเดอร์ตัวเอง
all:
    @echo "Building in src directory..."
    gcc -c main.c -o main.o

clean:
    @echo "Cleaning in src directory..."
    rm -f *.o
```