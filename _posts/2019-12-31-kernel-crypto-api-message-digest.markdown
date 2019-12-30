---
layout: post
title:  "Kernel Crypto API : Message Digest"
date:   2019-12-31
description: Simple kernel module example to generated SHA256 digest of input.
---

<p class="intro"><span class="dropcap">T</span>he kernel crypto API offers a rich set of cryptographic ciphers as well as other data transformation mechanisms and methods to invoke these</p>

The kernel crypto API refers to all algorithms as “transformations”. Therefore, a cipher handle variable usually has the name “tfm”. Besides cryptographic operations, the kernel crypto API also knows compression transformations and handles them the same way as ciphers.

The kernel crypto API serves the following entity types:

* consumers requesting cryptographic services
* data transformation implementations (typically ciphers) that can be called by consumers using the kernel crypto API

## Terminology

The transformation implementation is an actual code or interface to hardware which implements a certain transformation with precisely defined behavior.

The transformation object (TFM) is an instance of a transformation implementation. There can be multiple transformation objects associated with a single transformation implementation. Each of those transformation objects is held by a crypto API consumer or another transformation. Transformation object is allocated when a crypto API consumer requests a transformation implementation. The consumer is then provided with a structure, which contains a transformation object (TFM).

The structure that contains transformation objects may also be referred to as a “cipher handle”. Such a cipher handle is always subject to the following phases that are reflected in the API calls applicable to such a cipher handle:

1. Initialization of a cipher handle.
2. Execution of all intended cipher operations applicable for the handle where the cipher handle must be furnished to every API call.
3. Destruction of a cipher handle.

When using the initialization API calls, a cipher handle is created and returned to the consumer. The initialization API calls have all the same naming conventions of crypto_alloc*.

## Sample Module
We are going to write a simple kernel module that gets input from sysfs (``/sys/kernel/kcv/value``) and generates sha256 hash of input upon reading.
Here is ``Makefile`` for the module, which we call Kernl Crypto Verification (KCV).
```mak
obj-m += kcv.o
KDIR := /lib/modules/$(shell uname -r)/build

all:
	make -C $(KDIR) M=$(PWD) modules

clean:
	make -C $(KDIR) M=$(PWD) clean

test:
	-sudo rmmod kcv
	sudo insmod kcv.ko
```

### Create directory in ``sysfs``

Let's first create sysfs with ``kobject_create_and_add`` function. This function creates a kobject structure dynamically and registers it with sysfs. If the kobject was not able to be created, NULL will be returned. When you are finished with this structure, call kobject_put and the structure will be dynamically freed when it is no longer being used.

```c
struct kobject *kobject_create_and_add(const char *name, struct kobject *parent);
```

Where, ``name`` is the name for the kobject, ``parent`` is the parent kobject of this kobject, if any.

If we pass ``kernel_kobj`` to the second argument, it will create the directory under ``/sys/kernel/``, and under ``/sys/firmware/`` for ``firmware_kobj``, under ``/sys/fs`` for ``fs_kobj`` respectively. To create directly under ``/sys/`` we may pass NULL.

### Create ``sysfs`` file
Now we need to create sysfs file, which is used to interact user space with kernel space through sysfs. So we can create the sysfs file using sysfs attributes.

Attributes are represented as regular files in sysfs with one value per file. There are loads of helper function that can be used to create the kobject attributes. ``kobj_attribute`` is defined as,

```c
struct kobj_attribute {
    struct attribute attr;
    ssize_t (*show)(struct kobject *kobj, struct kobj_attribute *attr, char *buf);
    ssize_t (*store)(struct kobject *kobj, struct kobj_attribute *attr, const char *buf, size_t count);
};
```

Where, ``attr`` is the attribute representing the file to be created, ``show`` is the pointer to the function that will be called when the file is read in sysfs, ``store`` is the pointer to the function which will be called when the file is written in sysfs. We can create attribute using ``__ATTR`` macro.

```c
__ATTR(name, permission, show_ptr, store_ptr);
```

Here is how our implementation would look like, without actual implementation.
```c
static struct kobject *kcv_kobject;

static ssize_t kcv_show(struct kobject *kobj, struct kobj_attribute *attr, char *buf)
{
	...
	return 0;
}

static ssize_t kcv_store(struct kobject *kobj, struct kobj_attribute *attr, const char *buf, size_t count)
{
	...
	return count;
}

static struct kobj_attribute kcv_attribute = __ATTR(value, 0644, kcv_show, kcv_store);
```

To create a single file attribute we are going to use ‘sysfs_create_file’.

```c
int sysfs_create_file (struct kobject * kobj, const struct attribute *attr);
```

Where, ``kobj`` – object we’re creating for, ``attr`` – attribute descriptor.
One can use another function ``sysfs_create_group`` to create a group of attributes.
Once you have done with sysfs file, you should delete this file using ``sysfs_remove_file`` which has same prototype.

We are going to create ``sysfs`` directory and file during module initialization and remove then when module is removed.

```c
static int __init kcv_init(void)
{
	int ret = 0;

	pr_info("kcv: Initialize module\n");

	kcv_kobject = kobject_create_and_add("kcv", kernel_kobj);
	if (!kcv_kobject) {
		pr_err("kcv: Failed to create kernel object\n");
		return -ENOMEM;
	}

	ret = sysfs_create_file(kcv_kobject, &kcv_attribute.attr);
	if (ret)
		pr_err("kcv: Failed to create sysfs file\n");

	return ret;
}

static void __exit kcv_exit(void)
{
	pr_info("kcv: Deinitialize module\n");
	sysfs_remove_file(kcv_kobject, &kcv_attribute.attr);
	kobject_put(kcv_kobject);
}
```

### Message digest generation
To generate message digest first we need to create transformation object with ``crypto_alloc_shash(const char *alg, u32 type, u32 mask)``. The function allocates message digest handle, where ``alg`` is name of the message digest cypher, ``type`` is type, ``mask`` is mask of cypher.

```c
struct crypto_shash *tfm;
tfm = crypto_alloc_shash("sha256", 0, 0);
```

We then allocate memory for the operational state defined with struct ``shash_desc`` where the size of that data structure is to be calculated as ``sizeof(struct shash_desc) + crypto_shash_descsize(alg)``.

```c
struct shash_desc *desc;
desc = kmalloc(sizeof(*desc) + crypto_shash_descsize(tfm), GFP_KERNEL);
desc->tfm = tfm;
```

Now we will generate sha256 message digest with following function.
```c
crypto_shash_digest(struct shash_desc *desc, const u8 *data, unsigned int len, u8 *out)
```
Where ``desc`` is operational state handle that is already initialized, ``data`` is input data to be added to the message digest, ``len`` is the length of the input data, ``out`` is output buffer filled with the message digest. This function is a “short-hand” for the function calls of ``crypto_shash_init``, ``crypto_shash_update`` and ``crypto_shash_final``.

Now let's look how full implementation of store and show attirbute functions look.
```c
static ssize_t kcv_store(struct kobject *kobj, struct kobj_attribute *attr, const char *buf, size_t count)
{
	snprintf(kcv_buffer, sizeof(kcv_buffer), "%s", buf);
	kcv_buffer[sizeof(kcv_buffer) - 1] = '\0';
	return count;
}
```
Here, user input is received and stored in ``kcv_buffer`` whose maximum size is 4096 bytes.

```c
static ssize_t kcv_show(struct kobject *kobj, struct kobj_attribute *attr, char *buf)
{
	int ret = 0, i;
	struct crypto_shash *tfm;
	struct shash_desc *desc;
	unsigned char digest[32];
	char *hd, result[128] = {0, };
	tfm = crypto_alloc_shash("sha256", 0, 0);
	if (IS_ERR(tfm)) {
		pr_err("Failed to allocate hash\n");
		ret = -EINVAL;
		goto finish;
	}

	desc = kmalloc(sizeof(*desc) + crypto_shash_descsize(tfm), GFP_KERNEL);
	if (!desc) {
		pr_err("Failed to alocate to memory for shash_desc\n");
		ret = -ENOMEM;
		goto out_free_shash;
	}
	desc->tfm = tfm;

	ret = crypto_shash_digest(desc, kcv_buffer, strlen(kcv_buffer), digest);
	if (ret) {
		pr_err("Failed to digest, err = %d\n", ret);
		goto out_free_desc;
	}

	hd = result;
	for (i = 0; i < sizeof(digest); i++) {
		sprintf(hd, "%02x", digest[i]);
		hd += 2;
	}
	ret = sprintf(buf, "%s\n", result);

out_free_desc:
	kfree(desc);
out_free_shash:
	crypto_free_shash(tfm);
finish:
	return ret;
}
```

When user reads the ``sysfs`` file, sha256 hash of input is calculated and returned as hexadecimal string.

## Build and Test
We have built and tested the module on Raspberry Pi3.
```sh
root@sapi:/home/pi/kcv $ make
make -C /lib/modules/4.19.89-v7+/build M=/home/pi/kcv modules
make[1]: Entering directory '/home/pi/linux'
  CC [M]  /home/pi/kcv/kcv.o
  Building modules, stage 2.
  MODPOST 1 modules
  CC      /home/pi/kcv/kcv.mod.o
  LD [M]  /home/pi/kcv/kcv.ko
make[1]: Leaving directory '/home/pi/linux'
root@sapi:/home/pi/kcv $ make test
sudo insmod kcv.ko
```

As we have mentioned above ``sysfs`` file is created as ``/sys/kernel/kcv/value``. We can write to and read from the file. We write "Hello world!" string and get its sha256 digest value.
```sh
root@sapi:~$ echo "Hello world!" > /sys/kernel/kcv/value
root@sapi:~$ cat /sys/kernel/kcv/value
0ba904eae8773b70c75333db4de2f3ac45a8ad4ddba1b242f0b3cfc199391dd8
```

Let's verify result with userspace ``sha25sum`` tool. It indeed matches.
```sh
root@sapi:~$ echo "Hello world!" | sha256sum
0ba904eae8773b70c75333db4de2f3ac45a8ad4ddba1b242f0b3cfc199391dd8  -
```

## Full Sample Code

```c
/*
 * Copyright (C) 2019 Elmurod Talipov <elmurod.talipov@gmail.com>
 *
 * This program is free software; you can redistribute it and/or
 * modify it under the terms of the GNU General Public License
 * as published by the Free Software Foundation; either version 2
 * of the License, or (at your option) any later version.
 *
 * This program is distributed in the hope that it will be useful,
 * but WITHOUT ANY WARRANTY; without even the implied warranty of
 * MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
 * GNU General Public License for more details.
 *
 * You should have received a copy of the GNU General Public License
 * along with this program; if not, write to the Free Software
 * Foundation, Inc., 51 Franklin Street, Fifth Floor, Boston,
 * MA  02110-1301, USA.
 *
 */

#include <linux/init.h>
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/kobject.h>
#include <linux/sysfs.h>
#include <linux/string.h>
#include <linux/vmalloc.h>
#include <crypto/hash.h>

static struct kobject *kcv_kobject;
static char kcv_buffer[4096] = {0, };

static ssize_t kcv_show(struct kobject *kobj, struct kobj_attribute *attr, char *buf)
{
	int ret = 0, i;
	struct crypto_shash *tfm;
	struct shash_desc *desc;
	unsigned char digest[32];
	char *hd, result[128] = {0, };
	tfm = crypto_alloc_shash("sha256", 0, 0);
	if (IS_ERR(tfm)) {
		pr_err("Failed to allocate hash\n");
		ret = -EINVAL;
		goto finish;
	}

	desc = kmalloc(sizeof(*desc) + crypto_shash_descsize(tfm), GFP_KERNEL);
	if (!desc) {
		pr_err("Failed to alocate to memory for shash_desc\n");
		ret = -ENOMEM;
		goto out_free_shash;
	}
	desc->tfm = tfm;

	ret = crypto_shash_digest(desc, kcv_buffer, strlen(kcv_buffer), digest);
	if (ret) {
		pr_err("Failed to digest, err = %d\n", ret);
		goto out_free_desc;
	}

	hd = result;
	for (i = 0; i < sizeof(digest); i++) {
		sprintf(hd, "%02x", digest[i]);
		hd += 2;
	}
	ret = sprintf(buf, "%s\n", result);

out_free_desc:
	kfree(desc);
out_free_shash:
	crypto_free_shash(tfm);
finish:
	return ret;
}

static ssize_t kcv_store(struct kobject *kobj, struct kobj_attribute *attr, const char *buf, size_t count)
{
	snprintf(kcv_buffer, sizeof(kcv_buffer), "%s", buf);
	kcv_buffer[sizeof(kcv_buffer) - 1] = '\0';
	return count;
}

static struct kobj_attribute kcv_attribute = __ATTR(value, 0644, kcv_show, kcv_store);

static int __init kcv_init(void)
{
	int ret = 0;

	pr_info("kcv: Initialize module\n");

	kcv_kobject = kobject_create_and_add("kcv", kernel_kobj);
	if (!kcv_kobject) {
		pr_err("kcv: Failed to create kernel object\n");
		return -ENOMEM;
	}

	ret = sysfs_create_file(kcv_kobject, &kcv_attribute.attr);
	if (ret)
		pr_err("kcv: Failed to create sysfs file\n");

	return ret;
}

static void __exit kcv_exit(void)
{
	pr_info("kcv: Deinitialize module\n");
	sysfs_remove_file(kcv_kobject, &kcv_attribute.attr);
	kobject_put(kcv_kobject);
}

module_init(kcv_init);
module_exit(kcv_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Elmurod A. Talipov");
MODULE_DESCRIPTION("Kernel Crypto API Verification Module.");
MODULE_VERSION("0.1");
```
