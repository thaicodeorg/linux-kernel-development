# อธิบาย tcp_server.c 
โปรแกรมฝั่ง Server ที่ใช้ **Unix Domain Socket** สำหรับการสื่อสารระหว่างโปรเซส (Inter-Process Communication - IPC) บนเครื่องคอมพิวเตอร์เดียวกันครับ

การทำงานของมันจะคล้ายกับ TCP Server ทั่วไป แต่แทนที่จะสื่อสารผ่าน IP Address และ Port มันจะสื่อสารผ่าน **ไฟล์พิเศษ** ในระบบไฟล์แทน ซึ่งทำให้การสื่อสารรวดเร็วกว่าเพราะไม่ต้องผ่าน Network Stack ของระบบปฏิบัติการ


ผมคิดว่า "8nv" อาจจะเป็นการพิมพ์ผิดนะครับ แต่คำว่า **"ไฟล์พิเศษ" (Special File)** นั้นมีอยู่จริงในระบบปฏิบัติการตระกูล Unix (เช่น Linux, macOS) และเป็นแนวคิดที่สำคัญมากครับ

**ไฟล์พิเศษ** คือ ไฟล์ประเภทหนึ่งในระบบไฟล์ที่ **ไม่ได้ใช้เก็บข้อมูลเหมือนไฟล์ทั่วไป** แต่ทำหน้าที่เป็น **"ประตู"** หรือ **"ช่องทางสื่อสาร"** ไปยังส่วนอื่นๆ ของระบบ เช่น อุปกรณ์ฮาร์ดแวร์ หรือช่องทางการสื่อสารระหว่างโปรเซส (IPC)

เราสามารถเห็นประเภทของไฟล์ได้โดยใช้คำสั่ง `ls -l` ซึ่งตัวอักษรตัวแรกของแต่ละบรรทัดจะบอกประเภทของไฟล์นั้นๆ

---

## ประเภทของไฟล์พิเศษที่พบบ่อย

### 1. ไฟล์อุปกรณ์ (Device Files)
เป็นไฟล์ที่ใช้แทนอุปกรณ์ฮาร์ดแวร์ต่างๆ มักจะอยู่ในไดเรกทอรี `/dev`
* **Character Devices (`c`)**: เป็นอุปกรณ์ที่สื่อสารทีละตัวอักษร (character-by-character)
    * **ตัวอย่าง**: `/dev/tty` (หน้าจอ Terminal), `/dev/null` (หลุมดำ รับข้อมูลอะไรไปก็ได้แล้วทิ้ง), `/dev/random` (ตัวสร้างเลขสุ่ม)
* **Block Devices (`b`)**: เป็นอุปกรณ์ที่สื่อสารเป็นบล็อกข้อมูลขนาดใหญ่ (block-by-block)
    * **ตัวอย่าง**: `/dev/sda` (ฮาร์ดดิสก์ลูกแรก), `/dev/sdb1` (พาร์ติชันที่ 1 ของฮาร์ดดิสก์ลูกที่สอง)

### 2. Named Pipes (FIFO)
ตัวอักษรนำหน้าคือ `p` (ย่อมาจาก Pipe) เป็นไฟล์ที่ทำหน้าที่เหมือนท่อส่งข้อมูลแบบทางเดียว (one-way) สำหรับการสื่อสารระหว่างโปรเซส โปรเซสหนึ่งจะเขียนข้อมูลลงในไฟล์นี้ และอีกโปรเซสหนึ่งจะอ่านข้อมูลออกไป

### 3. Sockets
ตัวอักษรนำหน้าคือ `s` เป็นไฟล์ที่ใช้สำหรับการสื่อสารสองทาง (two-way) ระหว่างโปรเซส ซึ่งเป็นประเภทเดียวกับไฟล์ที่ถูกสร้างขึ้นจากโค้ด C++ ในคำถามก่อนหน้านี้ของคุณ
* **ตัวอย่าง**: ไฟล์ `/tmp/ipc_tcp_server.sock` จากโค้ดก่อนหน้านี้ คือ **ไฟล์พิเศษ** ประเภท **Socket** นั่นเองครับ มันไม่ได้เก็บข้อมูลอะไรไว้ข้างใน แต่ทำหน้าที่เป็นจุดเชื่อมต่อให้โปรแกรมอื่น (Client) เข้ามาคุยกับโปรแกรม Server ของเราได้
* 
-----

## อธิบายการทำงานของโค้ดทีละส่วน

### 1\. ส่วนเริ่มต้น (Headers and Constants)

```cpp
#include <sys/types.h>
#include <sys/socket.h>
#include <sys/un.h>
#include <memory.h>
#include <unistd.h>
#include <stdio.h>
#include <iostream>

using namespace std;
const char* server_file = "/tmp/ipc_tcp_server.sock";
```

  * `#include`: เป็นการนำเข้าไลบรารีที่จำเป็นสำหรับการทำงานกับ Socket, การจัดการหน่วยความจำ และการแสดงผล
  * `const char* server_file`: ประกาศตัวแปรเก็บชื่อ **ไฟล์ Socket** ที่จะใช้เป็นช่องทางสื่อสารของ Server ซึ่งในที่นี้คือ `/tmp/ipc_tcp_server.sock`

-----

### 2\. สร้าง Socket และตั้งค่า Address

```cpp
int socket_fd = socket(AF_UNIX, SOCK_STREAM, 0);
if (socket_fd < 0) { /* ... error handling ... */ }

struct sockaddr_un addr;
memset(&addr, 0, sizeof(addr));
addr.sun_family = AF_UNIX;
strcpy(addr.sun_path, server_file);
```

  * `socket()`: สร้าง Socket ขึ้นมา
      * `AF_UNIX`: ระบุว่าเป็น **Unix Domain Socket** (สื่อสารภายในเครื่อง)
      * `SOCK_STREAM`: ระบุว่าเป็น Socket แบบมีการเชื่อมต่อและรับประกันการส่งข้อมูล (เหมือน TCP)
  * `struct sockaddr_un addr`: สร้างโครงสร้างข้อมูลเพื่อเก็บ "ที่อยู่" ของ Server
  * `addr.sun_family = AF_UNIX`: กำหนดประเภทของที่อยู่เป็น `AF_UNIX`
  * `strcpy(addr.sun_path, server_file)`: คัดลอกที่อยู่ (ชื่อไฟล์ Socket) ไปใส่ในโครงสร้างข้อมูล `addr`

-----

### 3\. Bind (ผูก Socket กับ Address)

```cpp
if (access(addr.sun_path, 0) != -1) {
    remove(addr.sun_path);
}
if (bind(socket_fd, (sockaddr*)&addr, sizeof(addr)) < 0) {
    /* ... error handling ... */
}
```

  * `access()` และ `remove()`: **เป็นขั้นตอนสำคัญ** คือการตรวจสอบว่ามีไฟล์ Socket เก่าค้างอยู่หรือไม่ ถ้ามีให้ลบทิ้งก่อน เพราะฟังก์ชัน `bind` จะทำงานไม่สำเร็จหากไฟล์นั้นมีอยู่แล้ว
  * `bind()`: **ผูก** Socket ที่สร้างไว้ (`socket_fd`) เข้ากับที่อยู่ (ไฟล์ `/tmp/ipc_tcp_server.sock`) ที่เรากำหนดไว้ ตอนนี้ Server ก็จะมี "ที่อยู่" เป็นของตัวเองแล้ว

-----

### 4\. Listen และ Accept (รอรับการเชื่อมต่อ)

```cpp
if (listen(socket_fd, 12) < 0) { /* ... error handling ... */ }

int newcon = -1;
newcon = accept(socket_fd, (sockaddr*)&clientaddr, &addrlen);
if (newcon < 0) { /* ... error handling ... */ }
```

  * `listen()`: สั่งให้ Socket ของ Server เริ่ม **รอฟัง** หรือรอรับการเชื่อมต่อจาก Client
  * `accept()`: เป็นคำสั่งที่โปรแกรมจะ **หยุดรอ** จนกว่าจะมี Client เชื่อมต่อเข้ามา เมื่อมี Client เชื่อมต่อสำเร็จ ฟังก์ชันนี้จะสร้าง **Socket ใหม่** (`newcon`) ขึ้นมาเพื่อใช้สื่อสารกับ Client คนนั้นโดยเฉพาะ ส่วน Socket ตัวเก่า (`socket_fd`) ก็จะยังคงรอรับการเชื่อมต่อจาก Client คนอื่นๆ ต่อไป

-----

### 5\. วนลูปสื่อสาร (Communication Loop)

```cpp
while (1) {
    memset(msg_buf, '\0', 1024);
    int rsize = recv(newcon, msg_buf, sizeof(msg_buf), 0);
    if (rsize < 0) { /* ... error handling ... */ break; }

    cout << "I'm Unix socket(TCP) server, recv a msg:" << msg_buf << " from: " << clientaddr.sun_path << endl;

    strcpy(msg_buf, "OK,I got it!");
    int ssize = send(newcon, msg_buf, sizeof msg_buf, 0);
    if (ssize < 0) { /* ... error handling ... */ break; }

    sleep(1);
}
```

  * `while(1)`: เข้าสู่ลูปไม่รู้จบเพื่อรับ-ส่งข้อมูลกับ Client ที่เชื่อมต่ออยู่
  * `recv()`: **รอรับข้อมูล** จาก Client ผ่าน Socket `newcon` แล้วเก็บไว้ใน `msg_buf`
  * `cout`: แสดงข้อความที่ได้รับออกมาทางหน้าจอ
  * `send()`: **ส่งข้อมูล** ตอบกลับ (`"OK,I got it!"`) ไปยัง Client
  * `sleep(1)`: หยุดการทำงาน 1 วินาที ก่อนจะวนกลับไปรอรับข้อมูลใหม่

-----

### 6\. ปิดการเชื่อมต่อ (Close)

```cpp
if (close(newcon) < 0) { /* ... error handling ... */ }
if (close(socket_fd) < 0) { /* ... error handling ... */ }
```

  * เมื่อลูปสิ้นสุดลง (เช่น Client ตัดการเชื่อมต่อ หรือเกิดข้อผิดพลาด) โปรแกรมจะทำการปิด Socket ทั้งหมด
  * `close(newcon)`: ปิด Socket ที่ใช้คุยกับ Client
  * `close(socket_fd)`: ปิด Socket หลักที่ใช้รอรับการเชื่อมต่อจาก Client คนอื่นๆ

## สรุป 📝

โค้ดนี้คือการสร้าง **Server** ที่ทำงานบนเครื่องเดียว (localhost) โดยใช้ไฟล์เป็นตัวกลางในการสื่อสาร มีขั้นตอนคือ **สร้าง Socket** -\> **ผูกกับไฟล์** -\> **รอรับการเชื่อมต่อ** -\> **เมื่อมี Client มาเชื่อมต่อ ก็จะสร้างช่องทางสื่อสารใหม่เพื่อรับ-ส่งข้อความกัน** ครับ