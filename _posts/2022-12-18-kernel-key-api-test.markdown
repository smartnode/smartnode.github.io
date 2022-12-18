---
layout: post
title:  "Kernel Key API Test: Load custom key"
date:   2022-12-18
description: This blog post provides instructions and code for installing and implementing a kernel module on a Debian machine. It includes a Makefile for building and cleaning the module, and a C file for the actual code of the module. The code involves creating and manipulating a keyring and key in the Linux kernel. The module can be tested by inserting and removing it from the kernel. The post also includes a description of the variables and functions used in the code.
---

### Install Kernel Headers

To build kernel module you need to install kernel headers package. On debian machines you may install kernel headers with following command.

```sh
sudo apt install linux-headers-$(uname -r)
```

### Implement sample module

Here is `Makefile` for building sample module.

```make
PROJECT    := epk
MODULE_DIR := $(PWD)
KERNEL_DIR := /lib/modules/$(shell uname -r)/build
MOD_EXISTS := $(shell lsmod | grep -o $(PROJECT))
obj-m += $(PROJECT).o

all:
	@make -C $(KERNEL_DIR) M=$(MODULE_DIR) modules

clean:
	@make -C $(KERNEL_DIR) M=$(MODULE_DIR) clean

test:
	@if [ ! -z "$(MOD_EXISTS)" ]; then \
		sudo rmmod $(PROJECT); \
	fi
	@sudo insmod $(PROJECT).ko
```

This Makefile is used to build and clean a Linux kernel module with the name `epk`. The `PROJECT` variable is set to `epk`, which is the name of the kernel module. The `MODULE_DIR` variable is set to the current working directory (`PWD`), which is the directory where the Makefile and kernel module source files are located. The `KERNEL_DIR` variable is set to the directory of the Linux kernel source code, which is typically located at `/lib/modules/[kernel version]/build`.

The `obj-m` line specifies that the `epk` kernel module should be built as a loadable module.

The `all` target is used to build the kernel module. It runs the `make` command with the `-C` option, which specifies the directory to run make in (in this case, the Linux kernel source code directory). The `M` option specifies the directory of the kernel module source files (the `MODULE_DIR`). The `modules` target builds the kernel module.

The `clean` target is used to clean up the files created during the build process. It runs the `make` command in the same way as the `all` target, but with the clean target instead of `modules`.

The `test` target is used to test the kernel module. It first checks if the kernel module is already loaded in the kernel using the `lsmod` command and the `grep` command. If the module is loaded, it removes the module from the kernel using the `rmmod` command. It then inserts the module into the kernel using the `insmod` command.

Now we implement actual code in `epk.c` file.

```c
#include <linux/init.h>
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/vmalloc.h>
#include <linux/key.h>
#include <linux/cred.h>

#define CONFIG_EPK_KEYRING_NAME     ".epk_custom"
#define CONFIG_EPK_KEY_DESCRIPTION  "epk-verification-key"

static struct key *m_epk_keyring;
static key_ref_t m_epk_main_key;

static unsigned char EPK_X509_CERTIFICATE_DATA[] = { /* x509 certificate data */ };
static unsigned int EPK_X509_CERTIFICATE_LEN = 0; /* size of above array */

static int __init epk_init(void)
{
	const struct cred *cred = current_cred();
	key_perm_t perm = (KEY_POS_ALL & ~KEY_POS_SETATTR) | KEY_USR_VIEW | KEY_USR_READ;
	unsigned long flags = KEY_ALLOC_NOT_IN_QUOTA;

	pr_info("epk: Initialize module\n");

	m_epk_keyring = keyring_alloc(CONFIG_EPK_KEYRING_NAME, KUIDT_INIT(0), KGIDT_INIT(0),
								  cred, perm, flags, NULL, NULL);
	if (IS_ERR(m_epk_keyring))
	{
		pr_err("epk: Failed to allocate keyring (%ld)\n", PTR_ERR(m_epk_keyring));
		return PTR_ERR(m_epk_keyring);
	}

	flags |= KEY_ALLOC_BUILT_IN;
	flags |= KEY_ALLOC_BYPASS_RESTRICTION;
	m_epk_main_key = key_create_or_update(make_key_ref(m_epk_keyring, 1), "asymmetric",
										  CONFIG_EPK_KEY_DESCRIPTION,
										  EPK_X509_CERTIFICATE_DATA, EPK_X509_CERTIFICATE_LEN,
										  perm, flags);
	if (IS_ERR(m_epk_main_key))
	{
		pr_err("epk: Failed to load X.509 certificate (%ld)\n", PTR_ERR(m_epk_main_key));
		key_put(m_epk_keyring);
		return PTR_ERR(m_epk_main_key);
	}

	return 0;
}

static void __exit epk_exit(void)
{
	pr_info("epk: Deinitialize module\n");

	if (!IS_ERR(m_epk_main_key))
		key_ref_put(m_epk_main_key);

	if (!IS_ERR(m_epk_keyring))
		key_put(m_epk_keyring);
}

module_init(epk_init);
module_exit(epk_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Elmurod A. Talipov");
MODULE_DESCRIPTION("Kernel Key API Test Module.");
MODULE_VERSION("0.1");
```


This above defines the initialization and exit functions for a Linux kernel module.

The `__init` and `__exit` functions are special functions in the Linux kernel that are executed when the kernel module is loaded into the kernel and unloaded from the kernel, respectively. The `__init` function is executed when the module is loaded, and the `__exit` function is executed when the module is unloaded.

The `epk_init` function is the initialization function for the kernel module. It performs the following tasks:

- It prints a message to the kernel log using the `pr_info` function.
- It allocates a keyring using the `keyring_alloc` function and stores it in the `m_epk_keyring` global variable. A keyring is a special kind of key that can store other keys.
- It creates or updates a key in the keyring using the `key_create_or_update` function and stores it in the `m_epk_main_key` global variable. The key is created with the type "asymmetric", a description specified by the `CONFIG_EPK_KEY_DESCRIPTION` macro, and data specified by the `EPK_X509_CERTIFICATE_DATA` and `EPK_X509_CERTIFICATE_LEN` macros.
- It returns 0 to indicate that the initialization was successful.

The `epk_exit` function is the exit function for the kernel module. It performs the following tasks:

- It prints a message to the kernel log using the `pr_info` function.
- It releases the reference to the `m_epk_main_key` key using the `key_ref_put` function.
- It releases the reference to the `m_epk_keyring` keyring using the `key_put` function.

The `module_init` and `module_exit` macros are used to specify the initialization and exit functions for the kernel module. When the module is loaded into the kernel, the `epk_init` function is executed, and when the module is unloaded from the kernel, the `epk_exit` function is executed.


You can generate x509 certificate data using [this script](https://github.com/smartnode/epk/blob/linux-5.4/data/genkey.sh).

### Building Module

To install module you need to have root access. Run make command to build and verify output.

```sh
[elmurod@smartnode:epk{main}]$ make
make[1]: Entering directory '/usr/src/linux-headers-5.4.0-100-generic'
  Building modules, stage 2.
  MODPOST 1 modules
make[1]: Leaving directory '/usr/src/linux-headers-5.4.0-100-generic'pk/epk.mod.o
  LD [M]  /home/elmurod/tizen/github/e-talipov/epk/epk.ko
make[1]: Leaving directory '/usr/src/linux-headers-5.4.0-100-generic'
```

To install module run `make test`

### Verify Loaded Key

Check the key in key list with following command:

```sh
sudo cat /proc/keys | grep epk
```

Output should be something like below

```sh
22685c40 I------     1 perm 1f030000     0     0 keyring   .epk_custom: 1
38b5aca8 I------     2 perm 1f030000     0     0 asymmetri epk-verification-key: X509.rsa []
```
