# 01 Struct concept

![](./images/)

`struct` (ย่อมาจาก structure) คือ **โครงสร้างข้อมูล** ที่ให้เราสร้าง "ชนิดข้อมูล" (Data Type) ขึ้นมาใหม่ได้เอง เปรียบเสมือน 'แม่แบบ' หรือ 'พิมพ์เขียว' ที่เราสร้างขึ้นเพื่อเก็บข้อมูลหลายๆ อย่างที่เกี่ยวข้องกันไว้ในที่เดียว 🧩

ลองนึกถึง **"บัตรนักเรียน"** 📇 หนึ่งใบ จะมีข้อมูลหลายอย่างประกอบกัน เช่น:

  * รหัสนักเรียน (เป็นตัวเลข)
  * ชื่อ-สกุล (เป็นข้อความ)
  * เกรดเฉลี่ย (เป็นเลขทศนิยม)

ข้อมูลเหล่านี้มีชนิดที่แตกต่างกัน แต่ทั้งหมดนี้รวมกันเป็นข้อมูลของ "นักเรียน" หนึ่งคน `struct` ช่วยให้เรามัดรวมข้อมูลเหล่านี้ไว้ด้วยกันได้อย่างเป็นระเบียบ

-----

## ทำไมถึงต้องใช้ Struct?

เพื่อ **จัดระเบียบข้อมูล** ครับ ลองดูตัวอย่างนี้

**แบบไม่ใช้ Struct:**

```c
int student1_id = 6601001;
char student1_name[] = "สมชาย ใจดี";
float student1_gpa = 3.55;

int student2_id = 6601002;
char student2_name[] = "สมหญิง เก่งมาก";
float student2_gpa = 3.89;
```

จะเห็นว่าข้อมูลของนักเรียนแต่ละคนกระจัดกระจาย ถ้ามีนักเรียน 100 คน การจัดการจะวุ่นวายมาก

**แบบที่ใช้ Struct:**

```c
// สร้างแม่แบบ (Struct) ขึ้นมาก่อน
struct student {
    int id;
    char name[50];
    float gpa;
};

// นำแม่แบบมาสร้างเป็นตัวแปรจริงๆ
struct student s1;
s1.id = 6601001;
// ... (ดูตัวอย่างเต็มด้านล่าง)
```

โค้ดจะดูสะอาดและเข้าใจง่ายกว่า เพราะเรารู้ว่า `s1` คือข้อมูลของนักเรียนทั้งก้อน

-----

## วิธีการประกาศและใช้งาน

### 1\. การสร้างแม่แบบ (Define a Struct)

เราใช้คีย์เวิร์ด `struct` ตามด้วยชื่อแม่แบบที่เราต้องการ และภายในวงเล็บปีกกา `{}` คือสมาชิก (members) ที่อยู่ในโครงสร้างนั้น

```c
struct student {
    int   id;       // สมาชิกสำหรับเก็บรหัส (Integer)
    char  name[50]; // สมาชิกสำหรับเก็บชื่อ (Array of Characters)
    float gpa;      // สมาชิกสำหรับเก็บเกรดเฉลี่ย (Float)
}; // อย่าลืมเครื่องหมาย ; ปิดท้ายตรงนี้
```

### 2\. การนำแม่แบบไปใช้งาน (Declaring and Using a Struct Variable)

เมื่อเรามีแม่แบบแล้ว เราสามารถสร้างตัวแปรจากแม่แบบนั้นได้

การเข้าถึงข้อมูลข้างใน (members) จะใช้ **เครื่องหมายจุด `.` (Dot Operator)** 👉

```c
// ประกาศตัวแปรชื่อ s1 โดยใช้แม่แบบ struct student
struct student s1;

// กำหนดค่าให้กับสมาชิกข้างใน s1 โดยใช้ .
s1.id = 6601001;
s1.gpa = 3.55;

// สำหรับ string (char array) เราจะใช้ฟังก์ชัน strcpy
// ซึ่งต้อง #include <string.h> ก่อน
strcpy(s1.name, "สมชาย ใจดี");
```

**ข้อควรระวัง:** ในภาษา C เราไม่สามารถกำหนดค่าสตริงด้วยเครื่องหมาย `=` ตรงๆ ได้ ต้องใช้ฟังก์ชัน `strcpy()` (string copy) ซึ่งอยู่ในไลบรารี `string.h`

-----

## ตัวอย่างโค้ดฉบับสมบูรณ์

นี่คือโปรแกรมที่สมบูรณ์ซึ่งสร้าง `struct`, กำหนดค่า และแสดงผลข้อมูลออกมาทางหน้าจอ

```c
#include <stdio.h>
#include <string.h> // ต้อง include ไฟล์นี้เพื่อใช้ฟังก์ชัน strcpy

// 1. สร้างแม่แบบ (Define Struct)
struct student {
    int id;
    char name[50];
    float gpa;
};

int main() {
    // 2. ประกาศตัวแปรจากแม่แบบ
    struct student s1;

    // 3. กำหนดค่าให้สมาชิกของ s1
    s1.id = 6601001;
    strcpy(s1.name, "สมชาย ใจดี"); // ใช้ strcpy สำหรับ string
    s1.gpa = 3.55;

    // 4. แสดงผลข้อมูลโดยการเข้าถึงสมาชิกผ่าน .
    printf("--- ข้อมูลนักเรียน ---\n");
    printf("รหัสนักเรียน: %d\n", s1.id);
    printf("ชื่อ-สกุล: %s\n", s1.name);
    printf("เกรดเฉลี่ย: %.2f\n", s1.gpa); // %.2f คือแสดงทศนิยม 2 ตำแหน่ง

    return 0;
}
```

**ผลลัพธ์ที่ได้:**

```
--- ข้อมูลนักเรียน ---
รหัสนักเรียน: 6601001
ชื่อ-สกุล: สมชาย ใจดี
เกรดเฉลี่ย: 3.55
```

## Struct initialization

You can initialize a `struct` in C using curly braces `{}` in two main ways: by the order of its members or by explicitly naming the members. This method, often called **aggregate initialization**, provides a concise way to set the initial values of a `struct` variable when you declare it.

-----

### \#\# Methods of Struct Initialization

Let's use the following `struct` definition for all examples:

```c
struct network_device {
    int id;
    char name[20];
    char ip_address[16];
    int status; // 0 = Down, 1 = Up
};
```

#### \#\#\# 1. Initialization by Order

This is the classic method where you provide values in the **same order** as the members are defined in the `struct`. You must provide a value for each member.

```c
struct network_device eth0 = { 1, "eth0", "192.168.1.1", 1 };
```

In this case:

  * `1` is assigned to `id`.
  * `"eth0"` is assigned to `name`.
  * `"192.168.1.1"` is assigned to `ip_address`.
  * `1` is assigned to `status`.

#### \#\#\# 2. Designated Initialization (Recommended)

Introduced in the C99 standard, this method is more flexible and readable. You explicitly name which member you are initializing using a dot (`.`) before the member name. The order doesn't matter, and you can skip members, which will then be initialized to zero or NULL automatically.

```c
struct network_device wlan0 = {
    .name = "wlan0",
    .ip_address = "10.0.0.15",
    .status = 1,
    .id = 2
};
```

  * **Readability:** It's immediately clear which value corresponds to which member.
  * **Flexibility:** If the `struct` definition changes (e.g., members are reordered), this code will still work correctly.
  * **Partial Initialization:** You can easily initialize only the members you care about. For example, if you omit `.id`, it will be automatically initialized to `0`.

-----

### \#\# Complete Code Example

This program demonstrates both methods.

```c
#include <stdio.h>
#include <string.h>

// Define the structure for a network device
struct network_device {
    int id;
    char name[20];
    char ip_address[16];
    int status; // 0 = Down, 1 = Up
};

// A function to print device info
void print_device_info(struct network_device dev) {
    printf("ID: %d\n", dev.id);
    printf("Name: %s\n", dev.name);
    printf("IP Address: %s\n", dev.ip_address);
    printf("Status: %s\n", dev.status == 1 ? "Up" : "Down");
    printf("------------------------\n");
}

int main() {
    // 1. Initialization by Order
    // Values must match the order in the struct definition.
    struct network_device eth0 = { 1, "eth0", "192.168.1.1", 1 };

    // 2. Designated Initialization
    // Order doesn't matter, more readable and robust.
    struct network_device wlan0 = {
        .name = "wlan0",
        .ip_address = "10.0.0.15",
        .id = 2,
        .status = 1
    };
    
    // 3. Designated Initialization (Partial)
    // The 'id' and 'status' members will be auto-initialized to 0.
    struct network_device lo = {
        .name = "lo",
        .ip_address = "127.0.0.1"
    };


    printf("--- Device 1 (Initialized by Order) ---\n");
    print_device_info(eth0);

    printf("--- Device 2 (Designated Initializer) ---\n");
    print_device_info(wlan0);
    
    printf("--- Device 3 (Partial Initializer) ---\n");
    print_device_info(lo);

    return 0;
}
```

#### \#\#\# Program Output

```
--- Device 1 (Initialized by Order) ---
ID: 1
Name: eth0
IP Address: 192.168.1.1
Status: Up
------------------------
--- Device 2 (Designated Initializer) ---
ID: 2
Name: wlan0
IP Address: 10.0.0.15
Status: Up
------------------------
--- Device 3 (Partial Initializer) ---
ID: 0
Name: lo
IP Address: 127.0.0.1
Status: Down
------------------------
```