# คำอธิบาย ipc_udp_client.cpp

ครับ นี่คือโค้ดฝั่ง **Client** สำหรับเชื่อมต่อกับ UDP Server (`ipc_udp_server.cpp`) ที่อธิบายไปก่อนหน้านี้

การทำงานของ Client ในโหมด UDP (`SOCK_DGRAM`) จะแตกต่างจาก Client แบบ TCP อย่างชัดเจน โดยเฉพาะอย่างยิ่งคือมัน **ไม่ต้องใช้คำสั่ง `connect()`** เพื่อสร้างการเชื่อมต่อก่อนส่งข้อมูล

-----

## จุดแตกต่างที่สำคัญจาก Client แบบ TCP

1.  **`SOCK_DGRAM`**: สร้าง Socket เป็นประเภท `SOCK_DGRAM` เพื่อให้ทำงานในโหมด UDP เหมือนกับ Server
2.  **การ `bind()` ของ Client เป็นสิ่งจำเป็น**: ในเวอร์ชัน TCP Client ไม่จำเป็นต้อง `bind` ก็ได้ แต่สำหรับ UDP Client ที่ต้องการรับข้อมูลตอบกลับ **จำเป็นต้อง `bind`** เพื่อสร้าง "ที่อยู่สำหรับส่งกลับ" (Return Address) ให้ตัวเอง ไม่อย่างนั้น Server จะไม่รู้ว่าจะต้องส่งข้อมูลตอบกลับไปที่ไหน
3.  **ไม่ต้องใช้ `connect()`**: เนื่องจากเป็นการสื่อสารแบบไม่ต้องเชื่อมต่อ Client จึงสามารถส่งข้อมูลได้เลยโดยไม่ต้อง `connect` ไปยัง Server ก่อน
4.  **ใช้ `sendto()` และ `recvfrom()`**:
      * `sendto()`: ใช้สำหรับส่งข้อมูล โดย **ต้องระบุที่อยู่ของ Server ปลายทาง** ทุกครั้งที่ส่ง
      * `recvfrom()`: ใช้สำหรับรอรับข้อมูลตอบกลับจาก Server

-----

## อธิบายการทำงานของโค้ด

### 1\. การเตรียมการ (Setup)

```cpp
const char* server_file = "/tmp/ipc_udp_server.sock";
const char* client_file = "/tmp/ipc_udp_client.sock";

int socket_fd = socket(AF_UNIX, SOCK_DGRAM, 0);
// ... error handling ...

struct sockaddr_un client_addr;
// ... setup client address with client_file ...

if(bind(socket_fd, (sockaddr*)&client_addr, sizeof(client_addr)) < 0) {
    // ... error handling ...
}
```

  * `socket()`: สร้าง Socket แบบ `SOCK_DGRAM`
  * `bind()`: **ขั้นตอนนี้สำคัญมาก** Client ทำการ `bind` ตัวเองเข้ากับไฟล์ `/tmp/ipc_udp_client.sock` เพื่อสร้าง **"ที่อยู่"** ของตัวเองสำหรับให้ Server ส่งข้อมูลตอบกลับมาหา

-----

### 2\. วนลูปสื่อสาร (Communication Loop) 🔄

```cpp
struct sockaddr_un serveraddr;
// ... setup server address with server_file ...

while(1) {
    strcpy(msg_buf, "How are you !!!");
    int ssize = sendto(socket_fd, msg_buf, sizeof msg_buf, 0, (sockaddr*)&serveraddr, addrlen);
    // ... error handling ...

    int rsize = recvfrom(socket_fd, msg_buf, sizeof(msg_buf), 0, (sockaddr*)&serveraddr, &addrlen);
    // ... error handling ...

    cout << "I'm Unix socket(TCP) client，receive a msg :" << msg_buf << endl;
    sleep(1);
}
```

  * `struct sockaddr_un serveraddr`: สร้างโครงสร้างข้อมูลเพื่อเก็บที่อยู่ของ Server ที่จะส่งข้อมูลไปหา
  * `while(1)`: เข้าสู่ลูปเพื่อเริ่มส่ง-รับข้อมูล
  * `sendto()`: ส่งข้อความ "How are you \!\!\!" **ไปยังที่อยู่ของ Server (`serveraddr`)** ที่ระบุไว้โดยตรง เปรียบเสมือนการส่งจดหมายไปหา Server
  * `recvfrom()`: **รอรับข้อมูล** ที่จะถูกส่งกลับมายัง "ที่อยู่" ของตัวเองที่ได้ `bind` ไว้ตอนแรก
  * `cout`: แสดงข้อความที่ได้รับจาก Server

-----

## สรุป 📝

Client แบบ UDP จะ **สร้าง Socket และ `bind` ที่อยู่ให้ตัวเองก่อน** เพื่อใช้เป็นที่อยู่สำหรับรับข้อมูลตอบกลับ จากนั้นจึงเข้าสู่ลูปเพื่อ **`sendto`** (ส่งข้อมูลไปยัง Server) และ **`recvfrom`** (รอรับข้อมูลตอบกลับ) โดยไม่มีขั้นตอนการ `connect` ใดๆ ทั้งสิ้นครับ เป็นการสื่อสารแบบส่งแพ็กเก็ตโต้ตอบกันไปมาโดยตรง