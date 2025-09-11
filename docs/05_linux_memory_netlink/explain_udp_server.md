# คำอธิบาย ipc_udp_server.cpp

โค้ดนี้คือโปรแกรม Server ที่ใช้ **Unix Domain Socket** เหมือนเดิม แต่ทำงานในโหมด **UDP (`SOCK_DGRAM`)** ซึ่งมีความแตกต่างที่สำคัญจากโหมด TCP (`SOCK_STREAM`) ในตัวอย่างก่อนหน้านี้ครับ

ความแตกต่างหลักคือการสื่อสารแบบ UDP นั้นเป็นแบบ **"ไม่ต้องสร้างการเชื่อมต่อ" (Connectionless)** เปรียบเสมือนการส่งจดหมายหรือไปรษณียบัตร ที่แค่จ่าหน้าซองแล้วส่งไปเลย โดยไม่ต้องโทรไปนัดหมายก่อน

-----

## จุดแตกต่างที่สำคัญจากเวอร์ชัน TCP

1.  **`SOCK_DGRAM`**: ตอนสร้าง Socket จะใช้ `SOCK_DGRAM` (Datagram) แทน `SOCK_STREAM` เพื่อบอกว่าเป็น Socket แบบที่ไม่ต้องสร้างการเชื่อมต่อ
2.  **ไม่ต้องใช้ `listen()` และ `accept()`**: เนื่องจากไม่มีแนวคิดเรื่อง "การเชื่อมต่อ" Server จึงไม่จำเป็นต้องรอฟัง (`listen`) หรือยอมรับ (`accept`) การเชื่อมต่อจาก Client แค่ `bind` ตัวเองเข้ากับไฟล์ Socket แล้วก็พร้อมรอรับข้อมูลได้ทันที
3.  **ใช้ `recvfrom()` และ `sendto()`**:
      * `recvfrom()`: ใช้สำหรับ **รับข้อมูล** ฟังก์ชันนี้จะรอรับแพ็กเก็ตข้อมูล และที่สำคัญคือมันจะบอกด้วยว่าข้อมูลนั้นถูกส่ง **มาจากใคร** (โดยการเก็บที่อยู่ของ Client ไว้ในตัวแปร `clientaddr`)
      * `sendto()`: ใช้สำหรับ **ส่งข้อมูล** เวลาจะส่งข้อมูลตอบกลับ เราต้องระบุที่อยู่ปลายทาง (`clientaddr`) ไปด้วยทุกครั้ง เพื่อให้ข้อมูลถูกส่งกลับไปหา Client ที่ถูกต้อง

-----

## อธิบายการทำงานของโค้ด

### 1\. การเตรียมการ (Setup)

```cpp
const char* server_file = "/tmp/ipc_udp_server.sock";

int socket_fd = socket(AF_UNIX,SOCK_DGRAM,0);
// ... error handling ...

struct sockaddr_un addr;
// ... setup server address ...

if (bind(socket_fd,(sockaddr*)&addr,sizeof(addr)) < 0) {
    // ... error handling ...
}
```

  * `socket(AF_UNIX, SOCK_DGRAM, 0)`: สร้าง Socket แบบ Unix Domain แต่ระบุโหมดการทำงานเป็น `SOCK_DGRAM`
  * `bind()`: ทำหน้าที่เหมือนเดิม คือผูก Socket ของ Server เข้ากับไฟล์ `/tmp/ipc_udp_server.sock` เพื่อให้ Server มี "ที่อยู่" ที่ Client สามารถส่งข้อมูลมาหาได้

-----

### 2\. วนลูปสื่อสาร (Communication Loop) 🔄

```cpp
while (1) {
    // ...
    int rsize = recvfrom(socket_fd, msg_buf, sizeof(msg_buf), 0, (sockaddr*)&clientaddr, &addrlen);
    // ... error handling ...

    cout << "I'm Unix socket(UDP) server, recv a msg: " << msg_buf  << " from: " << clientaddr.sun_path << endl;

    strcpy(msg_buf, "OK,I got it!");
    int ssize = sendto(socket_fd, msg_buf, sizeof msg_buf, 0, (sockaddr*)&clientaddr, addrlen);
    // ... error handling ...
}
```

  * `while(1)`: เข้าสู่ลูปเพื่อรอรับ-ส่งข้อมูล
  * `recvfrom()`: โปรแกรมจะ **หยุดรอ** อยู่ที่บรรทัดนี้จนกว่าจะมีข้อมูลส่งมาถึง เมื่อมี Client ส่งข้อมูลมา ฟังก์ชันนี้จะเก็บข้อมูลไว้ใน `msg_buf` และเก็บที่อยู่ของผู้ส่งไว้ใน `clientaddr`
  * `cout`: แสดงข้อความที่ได้รับ พร้อมกับแสดงที่อยู่ของ Client ที่ส่งมา
  * `sendto()`: ส่งข้อความ `"OK,I got it!"` **กลับไปยังที่อยู่ของ Client (`clientaddr`)** ที่เพิ่งได้รับมา

-----

## สรุปเปรียบเทียบ ⚖️

  * **TCP Server (SOCK\_STREAM)**: เหมือนการ **โทรศัพท์** 📞 ต้องมีการต่อสาย (`connect`/`accept`) ให้สำเร็จก่อน แล้วจึงเริ่มสนทนากันผ่านสายนั้น
  * **UDP Server (SOCK\_DGRAM)**: เหมือน **ตู้ไปรษณีย์** 📬 แค่ตั้งตู้รอไว้ที่บ้าน (bind) ใครอยากส่งอะไรมาก็ส่งมาได้เลย เมื่อได้รับจดหมาย (recvfrom) ก็ดูชื่อที่อยู่ผู้ส่ง แล้วเขียนจดหมายตอบกลับ (sendto) ไปตามที่อยู่นั้น