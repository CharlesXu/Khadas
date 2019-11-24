# dev node

早期版本的Linux使用`register_chrdev`就会创建`/dev/`下的节点，貌似现在版本的不会了；

## 参考文档

* [How to create a device node from the init_module code of a Linux kernel module?](https://stackoverflow.com/questions/5970595/how-to-create-a-device-node-from-the-init-module-code-of-a-linux-kernel-module)

## code

```C
#include <linux/cdev.h>
#include <linux/device.h>
#include <linux/fs.h> /* register_chrdev, unregister_chrdev */
#include <linux/module.h>
#include <linux/seq_file.h> /* seq_read, seq_lseek, single_release */

#define NAME "lkmc_character_device_create"

static int major = -1;
static struct cdev mycdev;
static struct class *myclass = NULL;

static int show(struct seq_file *m, void *v)
{
    seq_printf(m, "abcd");
    return 0;
}

static int open(struct inode *inode, struct file *file)
{
    return single_open(file, show, NULL);
}

static const struct file_operations fops = {
    .llseek = seq_lseek,
    .open = open,
    .owner = THIS_MODULE,
    .read = seq_read,
    .release = single_release,
};

static void cleanup(int device_created)
{
    if (device_created) {
        device_destroy(myclass, major);
        cdev_del(&mycdev);
    }
    if (myclass)
        class_destroy(myclass);
    if (major != -1)
        unregister_chrdev_region(major, 1);
}

static int myinit(void)
{
    int device_created = 0;

    /* cat /proc/devices */
    if (alloc_chrdev_region(&major, 0, 1, NAME "_proc") < 0)
        goto error;
    /* ls /sys/class */
    if ((myclass = class_create(THIS_MODULE, NAME "_sys")) == NULL)
        goto error;
    /* ls /dev/ */
    if (device_create(myclass, NULL, major, NULL, NAME "_dev") == NULL)
        goto error;
    device_created = 1;
    cdev_init(&mycdev, &fops);
    if (cdev_add(&mycdev, major, 1) == -1)
        goto error;
    return 0;
error:
    cleanup(device_created);
    return -1;
}

static void myexit(void)
{
    cleanup(1);
}

module_init(myinit)
module_exit(myexit)
MODULE_LICENSE("GPL");
```