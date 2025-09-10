# 02 Basic Linux memory Management


## หัวใจของการจัดการหน่วยความจำใน Kernel

ก่อนจะไปดูโค้ดตัวอย่าง เราต้องเข้าใจภาพใหญ่ก่อนครับ Kernel มีหน้าที่จัดสรรทรัพยากรที่มีจำกัดที่สุดอย่างหนึ่ง นั่นคือ **RAM** ให้กับทุกส่วนของระบบได้อย่างมีประสิทธิภาพและปลอดภัยที่สุด

การจัดการหน่วยความจำใน Kernel แบ่งเป็น 2 ระดับหลักๆ:

1.  **Physical Page Allocator:** เป็นผู้จัดการ "ที่ดินเปล่า" ทั้งหมดในระบบ จัดการหน่วยความจำในหน่วยที่เรียกว่า **Page** (ปกติขนาด 4KB บน x86-64) ทำหน้าที่หา "บล็อก" ของหน่วยความจำกายภาพ (Physical Memory) ที่อยู่ติดกันจริงๆ
2.  **Slab Allocator (SLUB/SLAB/SLOB):** เป็นผู้จัดการ "ร้านค้าปลีก" ที่มาขอซื้อที่ดิน (Pages) จาก Buddy System มาทีละเยอะๆ แล้วซอยแบ่งเป็น "ห้องเล็กๆ" หรืออ็อบเจกต์ขนาดเท่าๆ กัน เพื่อให้ Kernel ส่วนอื่นๆ มาเบิกใช้ได้อย่างรวดเร็ว ไม่ต้องไปรบกวน Buddy System บ่อยๆ ฟังก์ชันอย่าง `kmalloc` ทำงานอยู่บนชั้นนี้ครับ

ทีนี้ เรามาเริ่มกันทีละขั้นตอน จากง่ายไปซับซ้อน ผ่าน 10 ตัวอย่างนี้ครับ

-----

### 1. การจองหน่วยความจำพื้นฐานที่สุด: `kmalloc`

`kmalloc` คือฟังก์ชันที่คุณจะได้ใช้บ่อยที่สุด มันเป็นการขอหน่วยความจำขนาดเล็กถึงปานกลาง (ไม่กี่ไบต์ถึงสองสามเพจ) จาก Slab Allocator

  * **คุณสมบัติเด่น:** หน่วยความจำที่ได้ **รับประกันว่าอยู่ติดกันในหน่วยความจำกายภาพ (Physically Contiguous)** ซึ่งจำเป็นมากสำหรับอุปกรณ์ฮาร์ดแวร์ (DMA)

**ตัวอย่าง:**

```c
#include <linux/slab.h> // สำหรับ kmalloc

char *my_buffer;
size_t buffer_size = 128;

// GFP_KERNEL คือ flag มาตรฐาน บอกให้ allocator รอ (sleep) ได้ถ้าหน่วยความจำไม่ว่าง
my_buffer = kmalloc(buffer_size, GFP_KERNEL);
if (!my_buffer) {
    // การจัดการ error สำคัญเสมอ!
    pr_err("Failed to allocate memory\n");
    return -ENOMEM;
}

// ตอนนี้ my_buffer ชี้ไปยังหน่วยความจำ 128 bytes ที่ใช้งานได้
strcpy(my_buffer, "Hello from the kernel!");
pr_info("Allocated buffer contains: %s\n", my_buffer);
```

**ข้อควรรู้:** `kmalloc` คือเครื่องมือหลักสำหรับงานส่วนใหญ่ แต่มีขนาดจำกัด (ปกติไม่เกิน 4MB)

-----

### \#\#\# 2. การคืนหน่วยความจำ: `kfree`

กฎเหล็กของ Kernel คือ **"จองอะไรไว้ ต้องคืนเสมอ"** การลืมคืนหน่วยความจำ (Memory Leak) ใน Kernel เป็นเรื่องร้ายแรงกว่าใน Userspace มาก เพราะมันจะอยู่กับระบบไปจนกว่าจะรีบูต

**ตัวอย่าง:**

```c
// ... หลังจากใช้ my_buffer เสร็จแล้ว
kfree(my_buffer);
my_buffer = NULL; // เป็น practice ที่ดี ที่จะเซ็ตเป็น NULL หลัง free
```

**ข้อควรรู้:** `kmalloc` คู่กับ `kfree` เสมอ ห้ามใช้ `kfree` กับหน่วยความจำที่จองมาจากฟังก์ชันอื่น

-----

### \#\#\# 3. โลกแห่ง Context: `GFP_` Flags

`GFP` (Get Free Page) flags คือตัวกำหนด "พฤติกรรม" ของ Allocator ตอนที่คุณเรียก `kmalloc` หรือฟังก์ชันอื่นๆ สองตัวที่สำคัญที่สุดคือ:

  * **`GFP_KERNEL`**: ใช้ใน **Process Context** (เช่น ตอนทำงานใน System Call) อนุญาตให้ Kernel "หลับ" (sleep) หรือพักการทำงานของ Process ปัจจุบัน เพื่อรอให้มีหน่วยความจำว่าง
  * **`GFP_ATOMIC`**: ใช้ใน **Interrupt Context** หรือในโค้ดที่ห้ามหลับเด็ดขาด (เช่น ขณะที่ถือ spinlock อยู่) การจองแบบนี้ **ห้ามหลับ** และจะพยายามจองจากหน่วยความจำสำรองฉุกเฉิน ถ้าไม่มีก็ล้มเหลวทันที

**ตัวอย่าง:**

```c
// ใน Interrupt Handler หรือโค้ดที่ถือ spinlock
void *atomic_buffer;
atomic_buffer = kmalloc(64, GFP_ATOMIC);
if (!atomic_buffer) {
    // ไม่สามารถรอได้ ต้องจัดการ error ทันที
    pr_warn("Atomic allocation failed. Dropping packet.\n");
    return;
}
// ... ใช้งาน buffer ...
kfree(atomic_buffer);
```

**ข้อควรรู้:** การเลือกใช้ GFP Flag ผิดประเภทเป็นสาเหตุของบั๊กที่อันตรายมาก ถ้าไม่แน่ใจ ให้เริ่มจาก `GFP_KERNEL`

-----

### \#\#\# 4. การจองหน่วยความจำขนาดใหญ่ระดับ Page: `alloc_pages`

เมื่อ `kmalloc` ไม่พอ (ต้องการหน่วยความจำเยอะๆ) เราต้องลงไปคุยกับ Buddy System โดยตรงผ่าน `alloc_pages` ซึ่งจะจองหน่วยความจำเป็นจำนวนยกกำลังสองของ Page

**ตัวอย่าง:**

```c
#include <linux/gfp.h> // สำหรับ alloc_pages

struct page *my_pages;
// ขอ 4 เพจ (2^2 = 4)
unsigned int order = 2; // order คือ log base 2 ของจำนวนเพจ

// ได้ผลลัพธ์เป็น struct page* ไม่ใช่ virtual address
my_pages = alloc_pages(GFP_KERNEL, order);
if (!my_pages) {
    return -ENOMEM;
}

// แปลง struct page* เป็น virtual address เพื่อให้ CPU ใช้งานได้
void *my_large_buffer = page_address(my_pages);

// ตอนนี้ my_large_buffer คือ buffer ขนาด 4 * 4096 = 16384 bytes
memset(my_large_buffer, 0, (1 << order) * PAGE_SIZE);
```

**ข้อควรรู้:** `alloc_pages` เหมาะกับการจองหน่วยความจำขนาดใหญ่ที่ต้องอยู่ติดกันจริงๆ (Physically Contiguous)

-----

### \#\#\# 5. การคืนหน่วยความจำระดับ Page: `free_pages`

เช่นเดียวกับ `kmalloc`/`kfree`, `alloc_pages` ก็มีคู่ของมัน

**ตัวอย่าง:**

```c
// เราส่งคืนโดยใช้ struct page* และ order เดิม
free_pages(my_pages, order);
```

**ข้อควรรู้:** ต้องใช้ `struct page*` และ `order` ที่ถูกต้องในการคืนหน่วยความจำ

-----

### \#\#\# 6. หน่วยความจำเสมือนที่อยู่ติดกัน: `vmalloc`

บางครั้งเราต้องการหน่วยความจำขนาดใหญ่มาก แต่**ไม่จำเป็น**ต้องอยู่ติดกันใน Physical RAM ก็ได้ `vmalloc` คือคำตอบ

  * **การทำงาน:** `vmalloc` จะไปขอ Page จาก Buddy System มาทีละหน้า (ซึ่งอาจจะกระจัดกระจายกันอยู่ใน RAM) แล้วนำมาเรียงต่อกันใน **Virtual Address Space** ของ Kernel ให้ดูเหมือนว่ามันอยู่ติดกัน

**ตัวอย่าง:**

```c
#include <linux/vmalloc.h>

char *v_buffer;
size_t huge_size = 32 * 1024 * 1024; // 32 MB

v_buffer = vmalloc(huge_size);
if (!v_buffer) {
    return -ENOMEM;
}
// ... ใช้งาน v_buffer ได้เหมือน buffer ทั่วไป ...
```

**ข้อควรรู้:** `vmalloc` ช้ากว่า `kmalloc` และไม่เหมาะกับงานที่ต้องใช้ DMA แต่เหมาะกับการสร้าง buffer ขนาดใหญ่มากๆ สำหรับซอฟต์แวร์

-----

### \#\#\# 7. การคืนหน่วยความจำเสมือน: `vfree`

`vmalloc` ก็ต้องคู่กับ `vfree`

**ตัวอย่าง:**

```c
// คืนหน่วยความจำที่จองโดย vmalloc
vfree(v_buffer);
```

**ข้อควรรู้:** ห้ามใช้ `kfree` กับ `vmalloc` และห้ามใช้ `vfree` กับ `kmalloc` เด็ดขาด\!

-----

### \#\#\# 8. กำแพงกั้นที่ปลอดภัย: `copy_from_user`

Kernel **ห้าม** เข้าถึง Pointer จาก Userspace โดยตรงเด็ดขาด เพราะ Userspace อาจส่ง Pointer ที่เป็น NULL, ชี้ไปยังหน่วยความจำของ Kernel เอง (เพื่อโจมตี) หรือหน้าที่ยังไม่มีอยู่จริง (ทำให้เกิด Page Fault)

`copy_from_user` คือฟังก์ชันที่ทำหน้าที่เป็น "ด่านตรวจ" คัดลอกข้อมูลจาก Userspace มายัง Kernel อย่างปลอดภัย

**ตัวอย่าง (ในฟังก์ชัน `write` ของ Character Device):**

```c
#include <linux/uaccess.h>

char k_buf[32];
size_t len = 32;

// buf คือ pointer ที่ได้มาจาก userspace
// copy_from_user จะตรวจสอบความถูกต้องทั้งหมดก่อนคัดลอก
if (copy_from_user(k_buf, buf, len)) {
    // ถ้าคืนค่าไม่เป็น 0 แสดงว่าผิดพลาด
    pr_err("Failed to copy data from user\n");
    return -EFAULT;
}

// ตอนนี้ข้อมูลใน k_buf ปลอดภัยและใช้งานได้
```

-----

### \#\#\# 9. ส่งข้อมูลกลับอย่างปลอดภัย: `copy_to_user`

ในทางกลับกัน เมื่อ Kernel ต้องการส่งข้อมูลกลับไปให้ Userspace ก็ต้องใช้ `copy_to_user`

**ตัวอย่าง (ในฟังก์ชัน `read` ของ Character Device):**

```c
char k_buf[] = "Data from kernel";
size_t len = sizeof(k_buf);

// to คือ pointer ปลายทางใน userspace
if (copy_to_user(to, k_buf, len)) {
    return -EFAULT;
}
```

**ข้อควรรู้:** `copy_from/to_user` คือกฎที่ต้องปฏิบัติตามอย่างเคร่งครัดเมื่อมีการสื่อสารข้ามกำแพง Kernel/User

-----

### \#\#\# 10. การจัดการอายุของอ็อบเจกต์: `kref` (Reference Counting)

จะเกิดอะไรขึ้นถ้ามีโค้ดหลายส่วนต้องการใช้อ็อบเจกต์เดียวกัน? ใครจะเป็นคน `kfree` มัน? ถ้าคนแรก `free` ไป คนที่สองมาใช้ต่อก็จะเกิด Crash ทันที

`kref` คือกลไกการนับอ้างอิง (Reference Counting) เพื่อแก้ปัญหานี้

  * เมื่อมีคนต้องการใช้อ็อบเจกต์ ให้เรียก `kref_get` (ตัวนับ++)
  * เมื่อใช้เสร็จ ให้เรียก `kref_put` (ตัวนับ--)
  * `kref_put` จะตรวจสอบว่าถ้านับเป็น 0 เมื่อไหร่ ให้เรียกฟังก์ชันทำลายอ็อบเจกต์ (ที่เรากำหนดไว้) โดยอัตโนมัติ

**ตัวอย่าง:**

```c
#include <linux/kref.h>

struct my_object {
    int data;
    struct kref refcount;
};

// ฟังก์ชันที่จะถูกเรียกเมื่อ refcount เป็น 0
void my_object_release(struct kref *ref) {
    struct my_object *obj = container_of(ref, struct my_object, refcount);
    pr_info("Releasing my_object with data %d\n", obj->data);
    kfree(obj);
}

// --- การใช้งาน ---
// 1. สร้างและกำหนดค่าเริ่มต้น
struct my_object *obj = kmalloc(sizeof(*obj), GFP_KERNEL);
obj->data = 123;
kref_init(&obj->refcount); // refcount = 1

// 2. มีคนอื่นมาขอใช้
kref_get(&obj->refcount); // refcount = 2

// 3. คนแรกใช้เสร็จ
kref_put(&obj->refcount, my_object_release); // refcount = 1

// 4. คนที่สองใช้เสร็จ
kref_put(&obj->refcount, my_object_release); // refcount = 0, ฟังก์ชัน my_object_release จะถูกเรียกและ kfree(obj) จะทำงาน
```

**ข้อควรรู้:** `kref` เป็นเครื่องมือสำคัญในการจัดการ Lifetime ของอ็อบเจกต์ที่ซับซ้อนและถูกใช้งานร่วมกันหลายที่

-----

บทเรียน 10 ขั้นตอนนี้จะพาคุณตั้งแต่การจองหน่วยความจำที่ง่ายที่สุด ไปจนถึงการจัดการที่ซับซ้อนและปลอดภัย ซึ่งเป็นทักษะที่นักพัฒนา Kernel ทุกคนต้องมีครับ