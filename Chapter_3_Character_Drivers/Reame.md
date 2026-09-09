# Chapter 3: Char drivers

Throughout the chapter, we present code fragments extracted from a real device driver: scull (Simple Character Utility for Loading Localities). scull is a char driver that acts on a memory area as though it were a device. In this chapter, because of that peculiarity of scull, we use the word device interchangeably with “the memory area used by scull.”

The advantage of scull is that it isn’t hardware dependent. scull just acts on some memory, allocated from the kernel. Anyone can compile and run scull, and scull is portable across the computer architectures on which Linux runs. On the other hand, the device doesn’t do anything “useful” other than demonstrate the interface between the kernel and char drivers and allow the user to run some tests.

:heavy_exclamation_mark: **The Design of scull**  
The first step of driver writing is defining the capabilities (the mechanism) the driver will offer to user programs. Since our “device” is part of the computer’s memory, we’re free to do what we want with it. It can be a sequential or random-access device, one device or many, and so on.  
The scull source implements the following devices. Each kind of device implemented
by the module is referred to as a type.
- scull0 to scull3: Four devices, each consisting of a memory area that is both global and persistent. Global means that if the device is opened multiple times, the data contained within the device is shared by all the file descriptors that opened it. Persistent means that if the device is closed and reopened, data isn’t lost. This device can be fun to work with, because it can be accessed and tested using conventional commands, such as cp, cat, and shell I/O redirection.
- scullpipe0 to scullpipe3: Four FIFO (first-in-first-out) devices, which act like pipes. One process reads what another process writes. If multiple processes read the same device, they contend for data. The internals of scullpipe will show how blocking and non- blocking read and write can be implemented without having to resort to interrupts. Although real drivers synchronize with their devices using hardware interrupts, the topic of blocking and nonblocking operations is an important one and is separate from interrupt handling (covered in Chapter 10).
- scullsingle
- scullpriv
- sculluid
- scullwuid: These devices are similar to scull0 but with some limitations on when an open is permitted. The first (scullsingle) allows only one process at a time to use the driver, whereas scullpriv is private to each virtual console (or X terminal ses-
sion), because processes on each console/terminal get different memory areas. sculluid and scullwuid can be opened multiple times, but only by one user at a time; the former returns an error of “Device Busy” if another user is locking the device, whereas the latter implements blocking open. These variations of scull would appear to be confusing policy and mechanism, but they are worth look- ing at, because some real-life devices require this sort of management.

:heavy_exclamation_mark:**Major and minor numbers**  
Char devices are accessed through names in the filesystem. Those names are called special files or device files or simply nodes of the filesystem tree; they are conventionally located in the /dev directory. Special files for char drivers are identified by a “c” in the first column of the output of ls –l. Block devices appear in /dev as well, but they are identified by a “b.” The focus of this chapter is on char devices, but much of the following information applies to block devices as well.

If you issue the ls –l command, you’ll see two numbers (separated by a comma) in the device file entries before the date of the last modification, where the file length normally appears. These numbers are the major and minor device number for the particular device. The following listing shows a few devices as they appear on a typical system. Their major numbers are 1, 4, 7, and 10, while the minors are 1, 3, 5, 64, 65, and 129.
```
crw-rw-rw- 1 root root 1, 3 Apr 11 2002 null
crw------- 1 root root 10, 1 Apr 11 2002 psaux
crw------- 1 root root 4, 1 Oct 28 03:04 tty1
crw-rw-rw- 1 root tty 4, 64 Apr 11 2002 ttys0
crw-rw---- 1 root uucp 4, 65 Apr 11 2002 ttyS1
crw--w---- 1 vcsa tty 7, 1 Apr 11 2002 vcs1
crw--w---- 1 vcsa tty 7, 129 Apr 11 2002 vcsa1
crw-rw-rw- 1 root root 1, 5 Apr 11 2002 zero
```
Traditionally, the major number identifies the driver associated with the device. For example, /dev/null and /dev/zero are both managed by driver 1, whereas virtual con- soles and serial terminals are managed by driver 4; similarly, both vcs1 and vcsa1 devices are managed by driver 7. Modern Linux kernels allow multiple drivers to share major numbers, but most devices that you will see are still organized on the one-major-one-driver principle. 

Major number xác định driver chịu trách nhiệm quản lý thiết bị.

The minor number is used by the kernel to determine exactly which device is being referred to. Depending on how your driver is written (as we will see below), you can either get a direct pointer to your device from the kernel, or you can use the minor number yourself as an index into a local array of devices. Either way, the kernel itself knows almost nothing about minor numbers beyond the fact that they refer to devices implemented by your driver.

Minor number xác định thiết bị cụ thể hoặc kênh cụ thể do driver đó quản lý (ví dụ: một driver quản lý 4 cổng nối tiếp thì có 1 Major number và 4 Minor number từ 0 đến 3).

:heavy_exclamation_mark:**The Internal Representation of Device Numbers**  
Within the kernel, the dev_t type (defined in <linux/types.h>) is used to hold device numbers—both the major and minor parts. As of Version 2.6.0 of the kernel, dev_t is a 32-bit quantity with 12 bits set aside for the major number and 20 for the minor number. Your code should, of course, never make any assumptions about the internal organization of device numbers; it should, instead, make use of a set of macros found in <linux/kdev_t.h>. To obtain the major or minor parts of a dev_t, use:
```
MAJOR(dev_t dev);
MINOR(dev_t dev);
```

If, instead, you have the major and minor numbers and need to turn them into a dev_t,use:
```
MKDEV(int major, int minor);
```

**Allocating and Freeing Device Numbers**  

**Phương pháp 1: Cấp phát Tĩnh (Static Allocation) — register_chrdev_region**  

Dùng khi bạn đã biết chính xác số Major Number nào mình muốn xin.
```
int register_chrdev_region(dev_t first, unsigned int count, char *name);
```
Tham số:
```
first: Số hiệu thiết bị bắt đầu xin cấp (chứa cả Major và Minor mong muốn, dùng macro MKDEV(major, minor) để tạo). Thường minor bắt đầu từ 0.

count: Số lượng Minor number liên tiếp muốn xin.

name: Tên của thiết bị. Tên này sẽ xuất hiện trong tập tin /proc/devices và hệ thống /sys.
```
Trả về: 0 nếu thành công; trả về số âm (mã lỗi như -EBUSY) nếu dải số đó đã bị driver khác chiếm mất.

Hạn chế: Dễ gây xung đột (conflict) nếu dải số bạn chọn trùng với một driver đã nạp trước đó trong hệ thống.

**Phương pháp 2: Cấp phát Động (Dynamic Allocation) — alloc_chrdev_region (Khuyên dùng)**

Dùng khi bạn chưa có Major Number và muốn Kernel tự động tìm và cấp cho bạn một Major Number còn trống. Đây là phương pháp chuẩn và an toàn nhất trong lập trình Linux Kernel hiện đại.
```
int alloc_chrdev_region(dev_t *dev, unsigned int firstminor, unsigned int count, char *name);
```
Tham số:
```
dev: Tham số đầu ra (Output-only). Sau khi hàm chạy thành công, biến chỉ số dev_t này sẽ lưu số hiệu thiết bị đầu tiên được cấp (chứa Major tự động + Minor đầu tiên).

firstminor: Minor number đầu tiên bạn muốn bắt đầu sử dụng (thường truyền 0).

count: Số lượng Minor number liên tiếp muốn xin cấp.

name: Tên thiết bị đăng ký trong /proc/devices.
```
Trả về: 0 nếu thành công; số âm nếu thất bại.

**Giải phóng Số hiệu Thiết bị — unregister_chrdev_region**

Dù cấp phát theo phương pháp tĩnh hay động, khi gỡ bỏ driver (unmount module), bạn bắt buộc phải trả lại dải số đó cho Kernel để các driver khác có thể tái sử dụng.
```
void unregister_chrdev_region(dev_t first, unsigned int count);
```
Tham số: 
```
first: Số hiệu thiết bị đầu tiên đã xin cấp trước đó.

count: Số lượng thiết bị cần trả lại (phải đúng bằng số count đã xin lúc đăng ký).
```
Vị trí gọi: Thường nằm trong hàm dọn dẹp/thoát của module (module_exit hay hàm cleanup_module).

Đăng ký dev_t mới chỉ là bước "xí chỗ": Các hàm trên chỉ đơn thuần báo cho Kernel biết: "Tôi chiếm giữ dải số Major/Minor này, đừng cấp cho ai khác".

Chưa liên kết với hàm xử lý (File Operations): Lúc này, User-space chưa thể thao tác (như open, read, write) với thiết bị được.

Bước tiếp theo: Driver cần khởi tạo cấu trúc cdev (Character Device) và liên kết nó với số hiệu dev_t đã xin thông qua hàm cdev_add().

Example:
```
#include <linux/init.h>
#include <linux/module.h>
#include <linux/fs.h>

static dev_t dev_num; // Biến lưu Major/Minor
static int count = 1; // Xin cấp 1 thiết bị

static int __init my_driver_init(void)
{
    int ret;

    // 1. Cấp phát động device number (Khuyên dùng)
    ret = alloc_chrdev_region(&dev_num, 0, count, "my_custom_device");
    if (ret < 0) {
        pr_err("Failed to allocate major number\n");
        return ret;
    }

    // In ra Major và Minor đã xin được
    pr_info("Driver registered: Major = %d, Minor = %d\n", MAJOR(dev_num), MINOR(dev_num));

    /* 2. Tiếp theo sẽ là khởi tạo cdev và cdev_add (Trình bày ở phần sau của sách) */

    return 0;
}

static void __exit my_driver_exit(void)
{
    // 3. Giải phóng device number khi gỡ driver
    unregister_chrdev_region(dev_num, count);
    pr_info("Driver unregistered successfully\n");
}

module_init(my_driver_init);
module_exit(my_driver_exit);

MODULE_LICENSE("GPL");
```

**Dynamic Allocation of Major Numbers**  
Cấp phát tĩnh (Static Number):
- Trước đây, một số Major Number được gán cố định cho các thiết bị phổ biến (được liệt kê trong Documentation/devices.txt).
- Ngày nay, việc chọn ngẫu nhiên một số Major rảnh rỗi chỉ hoạt động tốt trên máy cá nhân của bạn. Khi chia sẻ driver cho người khác, số ngẫu nhiên này rất dễ bị xung đột (conflict) với driver khác đã chiếm trước đó.

Cấp phát động (Dynamic Allocation):
- Khuyên dùng hàm alloc_chrdev_region. Kernel sẽ tự tìm một Major Number chưa bị chiếm để cấp cho driver.

**Vấn đề của Cấp phát Động và Giải pháp sử dụng Shell Script**  
Nếu Major Number được cấp tự động lúc nạp module (insmod), bạn không thể biết trước số Major để tạo sẵn file thiết bị (/dev/scull0, /dev/scull1...) trong hệ thống.

=> Viết một Shell script (như scull_load) để thực hiện chuỗi thao tác tự động ngay sau khi nạp driver:
```
Gọi insmod để nạp module vào Kernel.

Kernel cấp một Major Number động và ghi thông tin vào tập tin ảo /proc/devices.

Script dùng công cụ như awk đọc /proc/devices để lấy giá trị Major Number vừa được cấp.

Script tự động gọi lệnh mknod để tạo các file thiết bị trong /dev/ với đúng số Major vừa tìm được.

Phân quyền truy cập (owner/permissions) bằng chgrp và chmod để người dùng không phải root cũng có thể sử dụng.
```

Shell script mẫu:
```
#!/bin/sh
module="scull"
device="scull"
mode="664"

# 1. Nạp module vào Kernel
/sbin/insmod ./$module.ko $* || exit 1

# 2. Xóa các file thiết bị cũ trong /dev/ (nếu có)
rm -f /dev/${device}[0-3]

# 3. Đọc /proc/devices bằng awk để lấy Major Number của "scull"
major=$(awk "\\$2==\"$module\" {print \\$1}" /proc/devices)

# 4. Tạo 4 file thiết bị từ /dev/scull0 đến /dev/scull3 tương ứng các Minor Number 0->3
mknod /dev/${device}0 c $major 0
mknod /dev/${device}1 c $major 1
mknod /dev/${device}2 c $major 2
mknod /dev/${device}3 c $major 3

# 5. Phân quyền truy cập cho nhóm "staff" hoặc "wheel"
group="staff"
grep -q '^staff:' /etc/group || group="wheel"
chgrp $group /dev/${device}[0-3]
chmod $mode /dev/${device}[0-3]
```
Lưu ý về quyền hạn (chgrp/chmod): Mặc định khi script chạy dưới quyền root, các file /dev tạo ra sẽ thuộc sở hữu của root. Việc đổi group và quyền giúp các chương trình chạy ở User-space thông thường vẫn truy cập được thiết bị.

Mẹo trong quá trình phát triển (Development Trick): Nếu chỉ gỡ/nạp lại duy nhất 1 driver để debug (rmmod rồi insmod), Major Number động thường không thay đổi giữa các lần nạp. Do đó bạn không cần tạo lại file /dev liên tục.

Giải pháp tối ưu nhất khi viết driver là mặc định dùng cấp phát động, nhưng vẫn cho phép người dùng truyền Major Number tĩnh nếu muốn (qua dòng lệnh insmod hoặc tham số biên dịch).
```
// Nếu scull_major = 0 -> Dùng cấp phát động (Dynamic)
// Nếu scull_major > 0 -> Dùng cấp phát tĩnh (Static)
if (scull_major) {
    // 1. Cấp phát tĩnh nếu người dùng chỉ định sẵn scull_major
    dev = MKDEV(scull_major, scull_minor);
    result = register_chrdev_region(dev, scull_nr_devs, "scull");
} else {
    // 2. Mặc định cấp phát động nếu scull_major == 0
    result = alloc_chrdev_region(&dev, scull_minor, scull_nr_devs, "scull");
    scull_major = MAJOR(dev); // Cập nhật lại Major Number vừa được cấp
}

if (result < 0) {
    printk(KERN_WARNING "scull: can't get major %d\n", scull_major);
    return result;
}
```


**Some Important Data Structures**  

Most of the fundamental driver opera-
tions involve three important kernel data structures, called file_operations, file, and inode. A basic familiarity with these structures is required to be able to do much of anything interesting.

**File operation**  
Ý nghĩa của struct file_operations (fops):
- Cầu nối hệ thống: Sau khi xin cấp số hiệu thiết bị (dev_t), Kernel vẫn chưa biết khi ứng dụng gọi read(), write(), hay open() thì code nào trong driver sẽ chạy. Cấu trúc file_operations chính là tập hợp các con trỏ hàm (function pointers) đảm nhận nhiệm vụ này.
- Tư duy Hướng đối tượng (OOP in C): Trong Linux Kernel, file đại diện cho "đối tượng" (object), còn các hàm trong file_operations đóng vai trò là "phương thức" (methods) thao tác lên đối tượng đó.
- Quy ước đặt tên: Cấu trúc này hoặc con trỏ trỏ tới nó thường được gọi tắt là fops.
- Giá trị NULL: Nếu driver không cài đặt một hàm nào đó, con trỏ hàm tương ứng sẽ để NULL. Khi ứng dụng gọi system call tương ứng, Kernel sẽ tự xử lý mặc định (thường là trả về lỗi hoặc bỏ qua tùy hàm).
- Chú thích __user: Xuất hiện ở các tham số con trỏ (ví dụ: char __user *buf). Đây là đánh dấu chỉ ra rằng đây là địa chỉ thuộc bộ nhớ User-space, không được phép giải con trỏ (dereference) trực tiếp trong Kernel mà phải dùng các hàm hỗ trợ như copy_to_user() hay copy_from_user().

**Chi tiết các trường quan trọng trong file_operations**  
<figure align="center">
    <img src="../asset/Chapter_3/fops_1.png" alt="fd" width="600" height="500">
</figure>
<figure align="center">
    <img src="../asset/Chapter_3/fops_2.png" alt="fd" width="600" height="500">
</figure>
<figure align="center">
    <img src="../asset/Chapter_3/fops_3.png" alt="fd" width="600" height="150">
</figure>

Ví dụ khởi tạo instance cho driver scull:
```
struct file_operations scull_fops = {
    .owner   = THIS_MODULE,
    .llseek  = scull_llseek,
    .read    = scull_read,
    .write   = scull_write,
    .ioctl   = scull_ioctl,
    .open    = scull_open,
    .release = scull_release,
}
```

**The file structure**   
Bản chất của struct file: 
- Đại diện cho một File đang mở (Open File): Mỗi khi một tiến trình trong User-space gọi lệnh open() để mở một file (hoặc file thiết bị trong /dev), Kernel sẽ tạo ra một instance của struct file trong Kernel-space.
- Vòng đời: Cấu trúc này tồn tại từ lúc file được mở cho đến khi tất cả các bản sao của nó bị đóng hoàn toàn (close()). Khi không còn tiến trình nào dùng tới, Kernel sẽ giải phóng cấu trúc này.
- Phân biệt struct file (Kernel) và FILE (User-space): FILE (viết hoa): Là con trỏ do thư viện C tiêu chuẩn (stdio.h) quản lý ở User-space, hoàn toàn không xuất hiện trong Kernel code, struct file: Là cấu trúc nội bộ của Kernel, không bao giờ xuất hiện trực tiếp ở User-space.
- Quy ước đặt tên con trỏ: Trong Kernel source code, con trỏ trỏ tới struct file thường được gọi là filp (File Pointer) để tránh nhầm lẫn với chính bản thân cấu trúc file.

**Chi tiết các trường quan trọng trong struct file**  
<figure align="center">
    <img src="../asset/Chapter_3/file_1.png" alt="fd" width="600" height="500">
</figure>
<figure align="center">
    <img src="../asset/Chapter_3/file_2.png" alt="fd" width="600" height="500">
</figure>
<figure align="center">
    <img src="../asset/Chapter_3/file_3.png" alt="fd" width="600" height="500">
</figure>

Chỉ đọc và ghi đúng mục đích: Bạn không tạo ra struct file, Kernel tạo nó cho bạn. Bạn chỉ nhận con trỏ filp qua các tham số hàm (open, read, write, release...).

Khai thác private_data: Đây là nơi lý tưởng nhất để lưu trữ trạng thái của thiết bị giữa các lần gọi system call khác nhau (ví dụ: gán con trỏ thiết bị scull_dev vào filp->private_data ở hàm open, sau đó hàm read/write chỉ cần lấy lại ra để sử dụng).

**Struct inode**   
struct inode là gì và Sự khác biệt cốt lõi với struct file: 
- struct inode (Index Node): Đại diện cho một tập tin thực tế trên hệ thống (file vật lý trên đĩa hoặc file thiết bị trong /dev). Mỗi file trên hệ thống chỉ có duy nhất một struct inode, bất kể có bao nhiêu chương trình đang mở nó.
- struct file: Đại diện cho một phiên mở file (Open File Descriptor).

Ví dụ thực tế: Nếu 10 tiến trình cùng gọi lệnh open("/dev/scull0", ...) đồng thời:
- Kernel sẽ tạo ra 10 struct file riêng biệt (mỗi tiến trình giữ một phiên làm việc với cờ f_flags, vị trí f_pos riêng).

- Nhưng cả 10 struct file đó đều trỏ về duy nhất 1 struct inode đại diện cho file /dev/scull0.

**Hai trường (Fields) quan trọng nhất đối với Kỹ sư lập trình Driver**
<figure align="center">
    <img src="../asset/Chapter_3/inode.png" alt="fd" width="600" height="500">
</figure>

Để viết code có khả năng tương thích cao (portable) và không bị ảnh hưởng bởi các thay đổi trong tương lai của Kernel, lập trình viên không nên đọc trực tiếp inode->i_rdev, mà phải sử dụng 2 macro được Kernel cung cấp sẵn:
```
unsigned int imajor(struct inode *inode); // Trích xuất Major Number từ inode
unsigned int iminor(struct inode *inode); // Trích xuất Minor Number từ inode
```

Ứng dụng thực tế trong hàm open của Driver:

Khi ứng dụng mở file thiết bị, hàm open trong driver nhận vào tham số (struct inode *inode, struct file *filp). Bạn có thể dùng iminor(inode) để biết chính xác người dùng đang mở thiết bị phụ (Minor) nào:
```
static int scull_open(struct inode *inode, struct file *filp)
{
    unsigned int minor = iminor(inode);
    
    // Kiểm tra xem người dùng đang mở /dev/scull0, /dev/scull1 hay /dev/scull2...
    pr_info("Opening scull device with Minor number: %d\n", minor);

    return 0;
}
```

**Char device registration**  
Kernel sử dụng cấu trúc struct cdev (định nghĩa trong <linux/cdev.h>) để quản lý các thiết bị ký tự ở bộ nhớ nội bộ. Trước khi Kernel có thể gọi bất kỳ hàm thao tác nào (read, write, open...) của driver, bạn phải khởi tạo và đăng ký cấu trúc cdev này.

**Hai cách Cấp phát và Khởi tạo struct cdev**  
- Cách 1: Dùng khi bạn chỉ muốn cấp phát động một con trỏ cdev riêng lẻ:
```
struct cdev *my_cdev = cdev_alloc();
my_cdev->ops = &my_fops;
my_cdev->owner = THIS_MODULE;
```

- Cách 2: Nhúng cdev vào cấu trúc dữ liệu riêng của Driver (Embedded cdev) — Khuyên dùng
```
void cdev_init(struct cdev *cdev, struct file_operations *fops);
```
Ví dụ: 
```
struct my_device_struct {
    int dev_data;
    struct cdev cdev; // Nhúng cdev vào bên trong
};

struct my_device_struct my_dev;

// Khởi tạo
cdev_init(&my_dev.cdev, &my_fops);
my_dev.cdev.owner = THIS_MODULE;
```
Lưu ý: Dù chọn cách nào, bạn luôn phải gán trường owner của cdev bằng THIS_MODULE.

**Kích hoạt thiết bị với Kernel: cdev_add**  
Sau khi đã cài đặt xong cdev, bước quyết định là gọi hàm cdev_add để báo cho Kernel biết thiết bị đã sẵn sàng hoạt động.
```
int cdev_add(struct cdev *dev, dev_t num, unsigned int count);
```
Tham số:
```
dev: Con trỏ trỏ tới cấu trúc cdev đã khởi tạo.

num: Số hiệu thiết bị đầu tiên (dev_t) mà thiết bị này phản hồi.

count: Số lượng Minor number liên quan gắn với cdev này (thường là 1).
```
Kiểm tra lỗi: cdev_add có thể thất bại. Nếu trả về lỗi âm, thiết bị chưa được thêm vào hệ thống.  

Thiết bị "SỐNG" ngay lập tức (Live Device): Ngay khi cdev_add trả về thành công, Kernel có thể lập tức gọi các hàm open, read, write của driver nếu có yêu cầu từ User-space. Do đó, chỉ gọi cdev_add khi driver và phần cứng đã được chuẩn bị hoàn toàn xong xuôi.

**Hủy đăng ký thiết bị: cdev_del**  
Khi gỡ bỏ driver (trong hàm cleanup/exit), bạn cần gỡ thiết bị khỏi Kernel bằng hàm:
```
void cdev_del(struct cdev *dev);
```

**Đoạn code tổng hợp luồng đăng ký chuẩn**  
```
#include <linux/cdev.h>

struct cdev my_cdev;
dev_t dev_num; // Đã được xin cấp phát từ trước bằng alloc_chrdev_region

int init_my_character_device(void)
{
    int result;

    // 1. Khởi tạo cdev và gắn với file_operations (my_fops)
    cdev_init(&my_cdev, &my_fops);
    my_cdev.owner = THIS_MODULE;

    // 2. Đăng ký cdev với Kernel (Kích hoạt thiết bị)
    result = cdev_add(&my_cdev, dev_num, 1);
    if (result < 0) {
        pr_notice("Error %d adding my_cdev", result);
        return result;
    }

    return 0;
}

void cleanup_my_character_device(void)
{
    // 3. Gỡ bỏ cdev khi thoát module
    cdev_del(&my_cdev);
}
```

**Device registration in scull**  
Driver scull định nghĩa một cấu trúc dữ liệu tùy chỉnh có tên struct scull_dev để lưu trữ toàn bộ thông tin và trạng thái nội bộ của thiết bị.  
```
struct scull_dev {
    struct scull_qset *data; /* Con trỏ tới tập hợp quantum đầu tiên (vùng nhớ lưu dữ liệu) */
    int quantum;              /* Kích thước của mỗi quantum */
    int qset;                 /* Kích thước của mảng qset */
    unsigned long size;       /* Tổng lượng dữ liệu đang lưu trong thiết bị */
    unsigned int access_key;  /* Dùng cho việc kiểm soát truy cập trong sculluid/scullpriv */
    struct semaphore sem;     /* Semaphore dùng để khóa loại trừ tương hỗ (Mutual Exclusion) */
    struct cdev cdev;         /* Cấu trúc Char Device của Kernel nhúng bên trong */
};
```

**Giải thích Chi tiết Hàm scull_setup_cdev**  
```
static void scull_setup_cdev(struct scull_dev *dev, int index)
{
    int err, devno = MKDEV(scull_major, scull_minor + index);

    // 1. Khởi tạo cdev và gán bảng thao tác hàm scull_fops
    cdev_init(&dev->cdev, &scull_fops);

    // 2. Thiết lập thông tin chủ sở hữu module và gán lạiops
    dev->cdev.owner = THIS_MODULE;
    dev->cdev.ops = &scull_fops;

    // 3. Đăng ký cdev với Kernel
    err = cdev_add(&dev->cdev, devno, 1);

    /* 4. Xử lý lỗi nếu đăng ký thất bại */
    if (err)
        printk(KERN_NOTICE "Error %d adding scull%d", err, index);
}
```

Phân tích từng dòng lệnh:

- MKDEV(scull_major, scull_minor + index): Tạo ra số hiệu thiết bị devno (dev_t) cho thiết bị thứ index.
Ví dụ: Nếu scull_major = 240, scull_minor = 0: Với index = 0 $\rightarrow$ devno đại diện cho scull0 (Major 240, Minor 0), Với index = 1 $\rightarrow$ devno đại diện cho scull1 (Major 240, Minor 1).
- cdev_init(&dev->cdev, &scull_fops): Vì cdev nằm nhúng trong struct scull_dev, ta bắt buộc phải gọi cdev_init để Kernel thiết lập các giá trị mặc định ban đầu cho cdev và liên kết nó với bảng thao tác scull_fops.
- dev->cdev.owner = THIS_MODULE: Báo cho Kernel biết module hiện tại sở hữu thiết bị này, giúp Kernel tự động tăng/giảm đếm số lượt tham chiếu (reference count) để ngăn người dùng gỡ module (rmmod) khi thiết bị đang được sử dụng.
- cdev_add(&dev->cdev, devno, 1): Chính thức báo cho Kernel biết: "Thiết bị devno này đã sẵn sàng hoạt động với các hàm nằm trong scull_fops". Tham số 1 chỉ ra rằng cdev này chỉ quản lý đúng 1 Minor number.
- Xử lý lỗi (if (err)): Nếu cdev_add trả về số khác 0 (thất bại), thông điệp thông báo lỗi sẽ được in ra qua printk.

**Luồng tổng thể khi nạp Module scull**  
Khi driver scull được nạp vào Kernel, nó thường chạy một vòng lặp gọi hàm scull_setup_cdev cho từng thiết bị:
```
for (i = 0; i < scull_nr_devs; i++) {
    scull_setup_cdev(&scull_devices[i], i);
}
```

:heavy_exclamation_mark:


