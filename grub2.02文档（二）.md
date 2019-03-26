### grub2.02 源码分析——时间方面
-----

-  **Structure1：** `boottime` 对应的时间结构体

```C
#if BOOT_TIME_STATS // configure 选项 ---enable-boot-time
struct grub_boot_time
{
  struct grub_boot_time *next;
  grub_uint64_t tp;//时间
  const char *file;//文件名
  int line;//行数
  char *msg;//时间描述信息
};

```

可见，grub 使用一个链表记录时间

-  **Func1：** `grub_boot_time(...)` 添加链表节点的函数 

```C
#define grub_boot_time(...) grub_real_boot_time(GRUB_FILE, __LINE__, __VA_ARGS__)

struct grub_boot_time *grub_boot_time_head;
static struct grub_boot_time **boot_time_last = &grub_boot_time_head;

void
grub_real_boot_time (const char *file,
		     const int line,
		     const char *fmt, ...)
{
  struct grub_boot_time *n;
  .......

  va_start (args, fmt);
  n->msg = grub_xvasprintf (fmt, args);    
  va_end (args);
   grub_dprintf ("boot-time",n->tp); //添加函数,新增 boot-time 调试 tag
  *boot_time_last = n;
  boot_time_last = &n->next;

  .......
}
```

在函数体内，添加 `grub_dprintf()` 函数， 使调用 `grub_boot_time（）` 函数时可以同时打印出 debug 信息


-  **Func2：** `grub_dprintf()` 调试信息输出函数

```C
#define grub_dprintf(condition, ...) grub_real_dprintf(GRUB_FILE, __LINE__, condition, __VA_ARGS__)


void
grub_real_dprintf (const char *file, const int line, const char *condition,
		   const char *fmt, ...)
{
  va_list args;
  const char *debug = grub_env_get ("debug");

  if (! debug)
    return;

  if (grub_strword (debug, "all") || grub_strword (debug, condition))
    {
      grub_printf ("%s:%d: ", file, line);
      va_start (args, fmt);
      grub_vprintf (fmt, args);
      va_end (args);
      grub_refresh ();
    }
}

```

此处的 condition  即为 调试 tag


-  **Func3：** `grub_efi_init ()` efi 初始化（`grub-core/kern/i386/efi/init.c`）

```C
void
grub_efi_init (void)
{
........

  efi_call_4 (grub_efi_system_table->boot_services->set_watchdog_timer,
	      0, 0, 0, NULL); //关闭 watchdogtimer 

........
}
```

- 时间获取的方式

	- x86_64 使用汇编命令 `rdtsc` 获取 TSC（[Time Stamp Counter](https://en.wikipedia.org/wiki/Time_Stamp_Counter)）
	- i386-pc 使用 BIOS 中断 `int 0x1a` 获取 RTC（[Real-time clock](https://en.wikipedia.org/wiki/Real-time_clock)）


### 参考文献
-----

[1] <https://uefi.org/sites/default/files/resources/UEFI_Spec_2_7.pdf>