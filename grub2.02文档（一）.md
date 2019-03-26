### grub2.02 的下载、编译和安装
-------
- 下载（在[官网](ftp://ftp.gnu.org/gnu/grub)下载）


```shell
	$ wget ftp://ftp.gnu.org/grub/grub-2.02.tar.gz
```

- 编译


```shell
	$ mkdir grub #建立工作目录
	$ cd grub
	$ tar xzvf grub-2.02.tar.gz #解压缩，解包
	$ cd grub-2.02
	$ ./configure --target=x86_64  --with-platform=efi --enable-boot-time
	$ make -j4
```

在运行`configure`脚本时，`target`和`with-platform`用于指定编译平台为**x86_64-efi** .`enable-boot-time`选项用来打开 grub 本身的记录时间引导时间的特性。以方便在后续时间分析中，使用。（通过在 *grub shell* 通过 `boottime`查看）

- 安装

```shell
	$ make install
	$ grub-install --efi-directory=/boot/efi/ --bootloader-id=RainOS --target=x86_64-efi
```

`grub-install`会将新生成的 `*.mod` 文件安装到 `/boot/grub/x86_64-efi`中，并会将 `grubx64.efi` 安装到 `/boot/efi/EFI/RainOS`文件夹中

**Note:**

1. `--efi-directory` 选项指定 `esp` 在 RainOS 中即为 `boot/efi`
2. `--bootloader-id` 选项指定在`esp/EFI`目录下生成或使用的目录名.
3. `--target`  选项指定目标平台

### grub2.02 的 debug 设置

```shell
	$ set timeout=-1
	$ set pager=1
	$ set debug=tags
```

在 `boot/grub/grub.cfg`中的末尾（确保此时的变量值覆盖所有前置值）添加以上三行。分别用于显示 grub 菜单、分页显示debug信息、设置显示那些 debug 信息

**Note：**

1. 变量的设置， `=` 的两边必须紧挨
2. 在更改该文件时，可能需要 `chmod`
3. 在使用 `update-grub` 后，这些配置将丢失。也可以在 `/boot/grub/custom.cfg` 中修改后，再 `update-grub`
4. 变量 `debug` 中的 tags 由源代码中 `grub_dprinf()` 函数的第一个参数确定。 `all` 为打印所有 debug 信息，也可以使用 `linux,scripting` 用逗号隔离多个的形式


### 参考文献

[1]  <https://wiki.archlinux.org/index.php/GRUB#Installation>
