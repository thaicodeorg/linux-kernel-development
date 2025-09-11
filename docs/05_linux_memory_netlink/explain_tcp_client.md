# คำอธิบาย tcp_client.c

โปรแกรมฝั่ง **Client** ที่จะเชื่อมต่อไปยัง Server จากตัวอย่างก่อนหน้านี้ (`tcp_server.c`) เพื่อทำการสื่อสารกัน

-----

## อธิบายการทำงานของโค้ด Client

### 1\. ส่วนเริ่มต้น (Headers and Constants)

```cpp
#include <sys/types.h>
#include <sys/socket.h>
// ... (includes are the same as the server)

const char* server_file = "/tmp/ipc_tcp_server.sock";
const char* client_file = "/tmp/ipc_tcp_client.sock";
```

  * `#include`: นำเข้าไลบรารีที่จำเป็นเหมือนกับฝั่ง Server
  * `server_file`: **สำคัญมาก** เป็นการระบุที่อยู่ (ไฟล์ Socket) ของ **Server** ที่ Client ต้องการจะเชื่อมต่อไปหา
  * `client_file`: เป็นการกำหนดที่อยู่ (ไฟล์ Socket) ของตัว **Client** เอง ซึ่งในโค้ดนี้ Client จะทำการ `bind` ตัวเองกับไฟล์นี้ก่อนเชื่อมต่อ

### 2\. สร้าง Socket และ Bind (ฝั่ง Client)

```cpp
int socket_fd = socket(AF_UNIX, SOCK_STREAM, 0);
// ... error handling ...

struct sockaddr_un client_addr;
// ... setup client_addr with client_file ...

if (access(client_addr.sun_path, 0) != -1) {
    remove(client_addr.sun_path);
}

if(bind(socket_fd, (sockaddr*)&client_addr, sizeof(client_addr)) < 0) {
    // ... error handling ...
}
```

  * `socket()`: Client สร้าง Socket ของตัวเองขึ้นมา ซึ่งเป็นประเภท `AF_UNIX` และ `SOCK_STREAM` เหมือนกับของ Server
  * `bind()`: ในโค้ดตัวอย่างนี้ Client ทำการ `bind` ตัวเองเข้ากับไฟล์ `/tmp/ipc_tcp_client.sock` ก่อน ซึ่งเปรียบเสมือนการ "ตั้งชื่อ" หรือ "จองที่อยู่" ให้กับตัวเอง (ถึงแม้ในหลายกรณี Client ไม่จำเป็นต้อง `bind` ก็ได้ เพราะระบบปฏิบัติการจะกำหนดที่อยู่ชั่วคราวให้เอง)

### 3\. เชื่อมต่อไปยัง Server (Connect) 🔌

```cpp
struct sockaddr_un serveraddr;
memset(&serveraddr, 0, sizeof(serveraddr));
serveraddr.sun_family = AF_UNIX;
strcpy(serveraddr.sun_path, server_file);

int newcon = -1;
newcon = connect(socket_fd, (sockaddr*)&serveraddr, addrlen);
if (newcon < 0){
    perror("client connect");
}
```

  * `struct sockaddr_un serveraddr`: สร้างโครงสร้างข้อมูลเพื่อเก็บ "ที่อยู่" ของ Server
  * `strcpy(serveraddr.sun_path, server_file)`: นำที่อยู่ของ Server ที่เราทราบ (`/tmp/ipc_tcp_server.sock`) มาใส่ในโครงสร้าง
  * `connect()`: **นี่คือหัวใจของฝั่ง Client** 💖 เป็นคำสั่งที่ใช้ **ร้องขอการเชื่อมต่อ** ไปยังที่อยู่ของ Server ที่ระบุไว้ เมื่อคำสั่งนี้ทำงานสำเร็จ การเชื่อมต่อระหว่าง Client และ Server ก็จะเกิดขึ้น และฟังก์ชัน `accept()` ของฝั่ง Server ก็จะทำงานเสร็จสิ้น

### 4\. วนลูปสื่อสาร (Communication Loop) 🔄

```cpp
while(1) {
    strcpy(msg_buf, "How are you !!!");
    int ssize = send(socket_fd, msg_buf, sizeof msg_buf, 0);
    // ... error handling ...

    int rsize = recv(socket_fd, msg_buf, sizeof(msg_buf), 0);
    // ... error handling ...

    cout << "I'm Unix socket(TCP) client，receive a msg :" << msg_buf << endl;
    sleep(1);
}
```

  * `while(1)`: หลังจากเชื่อมต่อสำเร็จ Client จะเข้าสู่ลูปเพื่อเริ่มการสื่อสาร
  * `send()`: **ส่งข้อความ** "How are you \!\!\!" ไปให้ Server
  * `recv()`: **รอรับข้อความ** ตอบกลับจาก Server
  * `cout`: แสดงข้อความที่ได้รับจาก Server ออกทางหน้าจอ
  * `sleep(1)`: หยุดรอ 1 วินาที แล้วเริ่มส่งข้อความใหม่

### 5\. ปิดการเชื่อมต่อ (Close)

```cpp
if (close(socket_fd) < 0) {
    // ... error handling ...
}
```

  * `close()`: เมื่อโปรแกรมจบการทำงาน (เช่น ผู้ใช้กด Ctrl+C) จะทำการปิด Socket เพื่อยกเลิกการเชื่อมต่อและคืนทรัพยากรให้กับระบบ

-----

## สรุปภาพรวม 📝

โค้ด Client นี้จะ **สร้าง Socket** ของตัวเอง, **เชื่อมต่อ** ไปยังไฟล์ Socket ของ Server ที่กำลังรอ `listen` อยู่, และเมื่อเชื่อมต่อสำเร็จก็จะเข้าสู่ลูป **ส่ง-รับข้อความ** กับ Server ไปเรื่อยๆ ครับ