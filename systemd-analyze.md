## systemd-analyze 程序分析

### 预备知识
-----

- #### ACPI([Advanced Configuration and Power Interface](https://en.wikipedia.org/wiki/Advanced_Configuration_and_Power_Interface))

>高级配置与电源接口（英文：Advanced Configuration and Power Interface，缩写：ACPI），是1997年由英特尔、微软、东芝公司共同提出、制定提供操作系统应用程序管理所有电源管理接口，是一种工业标准，包括了软件和硬件方面的规范。2000年8月康柏和凤凰科技加入，推出 ACPI 2.0规格。2004年9月惠普取代康柏，推出 ACPI 3.0规格。2009年6月16日则推出 ACPI 4.0规格。2011年11月23日推出ACPI 5.0规格。由于ACPI技术正被多个操作系统和处理器架构采用，该规格的管理模式需要与时俱进。2013年10月，ACPI的推广者们一致同意将ACPI的属有归到UEFI论坛。今后新的ACPI规格将由UEFI论坛制定。

- #### ACPI Tables

>ACPI Tables就是BIOS提供给OS的硬件配置数据，包括系统硬件的电源管理和配置管理。

- #### D-Bus([wiki](https://zh.wikipedia.org/wiki/D-Bus))

>D-Bus是一个行程间通信及远程过程调用机制，可以让多个不同的计算机程序（即行程）在同一台计算机上同时进行通信[4]。D-Bus作为freedesktop.org项目的一部分，其设计目的是使Linux桌面环境（如GNOME与KDE等）提供的服务标准化。

### systemd-analyze 实现分析
-----

#### 函数调用流程：

   1 run() (analyze.c)//主函数
   
   2 dispatch_verb()->dispatch()//得到对应参数的对应处理函数地址
   
   3 time 对应的函数是 `analyze_time()`
   
   4 `analyze_time()->pretty_boot_time()->acquire_boot_times()` //boot获取时间
 
##### Func1 `acquire_boot_time()`(analyze.c)

 
 ``` C
 
 static int acquire_boot_times(sd_bus *bus, struct boot_times **bt) {
      .......
      
        r = bus_map_all_properties(
                        bus,
                        "org.freedesktop.systemd1",
                        "/org/freedesktop/systemd1",
                        property_map,
                        BUS_MAP_STRDUP,
                        &error,
                        NULL,
                        &times);
    
    ......
    
finish:
        *bt = &times;
        return 0;
}

 ```

此函数通过 DBus 与 `systemd` 通信，获取时间信息，保存在 `times` 中

##### Func2：`bus_map_all_properties()`(shared/bus-util.c)

```C
int bus_map_all_properties(
                sd_bus *bus,
                const char *destination,
                const char *path,
                const struct bus_properties_map *map,
                unsigned flags,
                sd_bus_error *error,
                sd_bus_message **reply,
                void *userdata) {

      .........
        r = sd_bus_call_method(
                        bus,
                        destination,
                        path,
                        "org.freedesktop.DBus.Properties",
                        "GetAll",
                        error,
                        &m,
                        "s", "");
      ..........

        r = bus_message_map_all_properties(m, map, flags, error, userdata);
        if (r < 0)
                return r;
     ..........
}
```

调用`GetAll()`接口 获取时间信息

**Note：**

- 关于此结论的验证：可通过相应的 `dbus-send` 命令完成。如下：

`	# dbus-send --system --print-reply --dest=org.freedesktop.systemd1 /org/freedesktop/systemd1 org.freedesktop.DBus.Properties.GetAll string:""`

可得到与 `systemd-analyze`相同的时间信息。更多 `dbus-send` 命令的用法，详见 [man dbus-send](https://dbus.freedesktop.org/doc/dbus-send.1.html)

- systemd 中的时间结构体

```C
	typedef struct dual_timestamp {
        usec_t realtime;
        usec_t monotonic;//主要对比的时间
} dual_timestamp;

```

#### Manager 的初始化

排查代码可知，上述函数 `GetAll()` 获取的 Manager 的属性

```C
	//Manager 结构体
    struct Manager {
 		 ........

        dual_timestamp timestamps[_MANAGER_TIMESTAMP_MAX]; //保存时间信息
		........
	};

```
 ##### Func1：`boot_timestamps（）`(shared/boot-timestamps.c) 初始化 Manager 时间信息
 
 ```C
 	int boot_timestamps(const dual_timestamp *n, dual_timestamp *firmware, dual_timestamp *loader) {
      .........

        r = acpi_get_boot_usec(&x, &y);//通过 ACPI 获取
        if (r < 0) {
                r = efi_loader_get_boot_usec(&x, &y);//通过 efivars 获取
                if (r < 0)
                        return r;
        }

        /* Let's convert this to timestamps where the firmware
         * began/loader began working. To make this more confusing:
         * since usec_t is unsigned and the kernel's monotonic clock
         * begins at kernel initialization we'll actually initialize
         * the monotonic timestamps here as negative of the actual
         * value. */

        firmware->monotonic = y;
        loader->monotonic = y - x;

        a = n->monotonic + firmware->monotonic;
        firmware->realtime = n->realtime > a ? n->realtime - a : 0;

        a = n->monotonic + loader->monotonic;
        loader->realtime = n->realtime > a ? n->realtime - a : 0;

        return 0;
}

 ```
 
 **Note：**
 
 通过添加 linux 内核启动参数 `acpi=off` ,发现 `systemd-analyze` 不再能获取到 firmware、loader 的时间。由此可断定在本系统上使用的是 ACPI 来获取时间
 
 ##### Func2：acpi_get_boot_usec`(shared/acpi-fpdt.c)  ACPI 时间获取函数
 
 ```C
 
 	int acpi_get_boot_usec(usec_t *loader_start, usec_t *loader_exit) {
        ........
        
        r = read_full_file("/sys/firmware/acpi/tables/FPDT", &buf, &l);
        ........
        
        tbl = (struct acpi_table_header *)buf;
        ........
        
        /* find Firmware Basic Boot Performance Pointer Record */
        for (rec = (struct acpi_fpdt_header *)(buf + sizeof(struct acpi_table_header));
             (char *)rec < buf + l;
             rec = (struct acpi_fpdt_header *)((char *)rec + rec->length)) {
                if (rec->length <= 0)
                        break;
                if (rec->type != ACPI_FPDT_TYPE_BOOT)
                        continue;
                if (rec->length != sizeof(struct acpi_fpdt_header))
                        continue;

                ptr = rec->ptr;
                break;
        }

        ........

        /* read Firmware Basic Boot Performance Data Record */
        fd = open("/dev/mem", O_CLOEXEC|O_RDONLY);
        ........
        
        l = pread(fd, &hbrec, sizeof(struct acpi_fpdt_boot_header), ptr);
        ........

        if (memcmp(hbrec.signature, "FBPT", 4) != 0)
                return -EINVAL;

        ........
        
        l = pread(fd, &brec, sizeof(struct acpi_fpdt_boot), ptr + sizeof(struct acpi_fpdt_boot_header));
        if (l != sizeof(struct acpi_fpdt_boot))
                return -EINVAL;

        .........
        
        if (loader_start)
                *loader_start = brec.startup_start / 1000;
        if (loader_exit)
                *loader_exit = brec.exit_services_exit / 1000;

        return 0;
}

 ```

此函数主要是读取 FPDT ，然后从 FPDT 表中找出 FBPT 表的指针。然后在 `/dev/mem` 相应的内存位置读取 FBPT 表。从该表中读取 `startup_start`,`exit_services_exit` 时间戳


**Note：**

- 可配合使用工具 `acpidump`,`acpixtract`,`iasl` 工具查看 ACPI Table 的内容
- 关于 FBPT 表内容的读取验证，可仿照 `acpi_get_boot_usec()` 函数编程实现
- ACPI 各个表的结构可在参考文献[1] ACPI 手册中找到


   
### 参考文献
-----

[1] <https://uefi.org/sites/default/files/resources/ACPI_6_3_final_Jan30.pdf>