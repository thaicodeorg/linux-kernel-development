# Ref 01 Struct  netlink_kernel_cfg

แน่นอนครับ

`{ .input = nl_recv_msg }` คือ  **designated initializer** ที่มีในภาษา C (ตั้งแต่ C99) สำหรับการกำหนดค่าเริ่มต้นให้กับสมาชิก (member) ของ `struct` โดยเฉพาะเจาะจง

นี่คือโครงสร้างของ `struct netlink_kernel_cfg` 


-----

## โครงสร้างของ `netlink_kernel_cfg`

```c
struct netlink_kernel_cfg {
    unsigned int    groups;
    unsigned int    flags;
    void            (*input)(struct sk_buff *skb);
    struct mutex    *cb_mutex;
    int             (*bind)(struct net *net, int group);
    void            (*unbind)(struct net *net, int group);
    bool            (*compare)(struct net *net, struct sock *sk);
};
```

### คำอธิบายแต่ละ Member

  * **`groups`**:

      * **ประเภท**: `unsigned int`
      * **หน้าที่**: ใช้ระบุจำนวนสูงสุดของ multicast groups ที่ socket นี้จะรองรับได้ หากคุณไม่ได้ใช้ multicast ก็ไม่ต้องกำหนดค่านี้

  * **`flags`**:

      * **ประเภท**: `unsigned int`
      * **หน้าที่**: ใช้กำหนดค่าสถานะ (flags) เพิ่มเติม แต่โดยทั่วไปมักจะไม่ได้ใช้และปล่อยเป็น 0

  * **`input`**:

      * **ประเภท**: `void (*)(struct sk_buff *skb)`
      * **หน้าที่**: นี่คือสมาชิกที่ **สำคัญที่สุด** 💯 มันคือ **function pointer** ที่จะชี้ไปยังฟังก์ชัน **callback** ของเรา เมื่อมีข้อมูล Netlink ถูกส่งมาถึง Kernel Socket นี้ Kernel จะเรียกใช้ฟังก์ชันที่ `input` ชี้ไปโดยอัตโนมัติ พร้อมส่งข้อมูลที่ได้รับ (ในรูปของ `struct sk_buff *skb`) มาให้

  * **`cb_mutex`**:

      * **ประเภท**: `struct mutex *`
      * **หน้าที่**: เป็น mutex ที่ใช้ป้องกันการเข้าถึง callback พร้อมกัน (concurrency) หากกำหนดค่านี้ Netlink subsystem จะทำการ lock mutex นี้ก่อนเรียกฟังก์ชัน `input` และจะ unlock หลังจาก `input` ทำงานเสร็จ เพื่อให้มั่นใจว่า callback จะทำงานทีละครั้งเท่านั้น

  * **`bind` / `unbind`**:

      * **ประเภท**: `int (*)(...)` / `void (*)(...)`
      * **หน้าที่**: เป็น callback ที่จะถูกเรียกเมื่อ userspace ทำการ bind หรือ unbind กับ multicast group เหมาะสำหรับกรณีที่ต้องมีการจัดการสถานะบางอย่างเมื่อมีคนเริ่มหรือหยุดฟัง event

  * **`compare`**:

      * **ประเภท**: `bool (*)(...)`
      * **หน้าที่**: เป็นฟังก์ชัน callback สำหรับเปรียบเทียบ socket ใช้ในกรณีที่ซับซ้อนขึ้นเพื่อค้นหา socket ที่ต้องการ มักจะไม่จำเป็นต้องใช้ในการใช้งานทั่วไป

ดังนั้น การเขียนโค้ดว่า:

```c
struct netlink_kernel_cfg cfg = {
    .input = nl_recv_msg,
};
```

คือการบอก Kernel ว่า "ฉันกำลังจะสร้าง Netlink socket โดยมีข้อกำหนดว่า เมื่อไหร่ก็ตามที่มีข้อมูลเข้ามา ให้เรียกใช้ฟังก์ชัน `nl_recv_msg` นะ" ส่วนสมาชิกตัวอื่นๆ ที่เราไม่ได้กำหนดค่า จะถูกกำหนดค่าเริ่มต้นเป็นศูนย์ (zero-initialized) โดยอัตโนมัติครับ


### designated initializers

```c
struct Date {
    int year;
    int month;
    int day;
};

Date inFuture { .year = 2050, .month = 4, .day = 10 };
```

!!! info "โครงสร้างพื้นฐาน"
    Type obj = { .designator = val, .designator = val2, ... };  
    // หรือ  
    Type obj = { .designator { val }, .designator { val2 }, ... };

```c title="example"
struct Point { 
    double x { 0.0 };
    double y { 0.0 };
};

const Point p { .x = 10.0, .y = 20.0 };
 
const Point offset { .x { 100.0 }, .y { -100.0 } };

// mix also possible:
const Point translation { .x = 50.0, .y { -40.0 } };
```

- Example initialization
```c "point.c"
#include <stdio.h>
// C program to demonstrate designated
// initializers with structures
struct Point
{
    int a, b, c;
};
int main()
{
    // Examples of initialization using
    // designated initialization
    struct Point p1 = {.b = 0, .c = 1, .a = 2};
    struct Point p2 = {.a = 20};
    printf ("p1.a = %d, p1.b = %d, p1.c = %d\n", p1.a, p1.b, p1.c);
    printf ("p2.a = %d", p2.a);
    return 0;
}
```

- Compile
```
gcc -o program point.c -Wall
./program
---

- Output
```
p1.a = 2, p1.b = 0, p1.c = 1
p2.a = 20
``
