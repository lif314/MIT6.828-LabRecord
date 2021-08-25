[TOC]

# Lab Guide

- JOS Environment
- GDB
- QEMU



## Debugging tips

### debug kernel

`qemu-gdb` or `qemu-gdb-nox`  ----  使qemu等待gdb连接

`-d` ---- 如果QEMU发生意外中断、异常、故障等，使用`-d`生成中断日志

- debug virtual memory --- 仅显示当前页表

`info mem` --- 使用QEMU监视器查看详细信息 

`info pg ` --- 详细信息

- debug multiple CPUs

`thread`   or `info threads`  ---- 查看线程相关信息



### user environments （用户环境）

- debug user env

gdb无法识别多个用户环境之间和用户与内核之间的关系

`make run-name` ---  以特定用户环境启动JOS  === 编辑 `kern/init.c`

`run-name-gdb` --- 等待gdb连接

- debug user code  == debug kernel code

**symbol table**  ---  gdb debugging information   ????

`symbol-file` --- 告诉gdb使用的符号表，因为gdb一次只能识别一种符号表

`.gdbinit` --- 加载内核符号表 `obj/kern/kernel`

`symbol-file obj/user/name` --- 用户环境的符号表位于ELF二进制文件中，使用该命令加载它



注意！！ --- *on't* load symbols from any `.o` files, as those haven't been relocated by the linker (libraries are statically linked into JOS user binaries, so those symbols are already included in each user binary). Make sure you get the *right* user binary; library functions will be linked at different EIPs in different binaries and GDB won't know any better!

```
不要从任何 .o 文件加载符号，因为这些符号尚未被链接器重新定位（库静态链接到 JOS 用户二进制文件中，因此这些符号已包含在每个用户二进制文件中）。确保您获得正确的用户二进制文件；库函数将在不同二进制文件中的不同 EIP 处链接，GDB 不会知道任何更好的！
```



- gdb breakpoint  ????? 

```
由于GDB作为一个整体连接到虚拟机，因此它将时钟中断(clock interrupts)视为另一个控制传输(control transfer)。这使得基本上不可能单步执行用户代码，因为在您让VM再次运行的那一刻实际上保证了时钟中断。 stepi命令有效是因为它抑制中断(suppresses interrupts)，但它只执行一条汇编指令。断点通常可以工作，但要小心，因为您可以在不同的环境中访问相同的 EIP（实际上，完全不同的二进制文件！）。
```



## JOS Makefile

`.gdbinit` --- qemu与gdb连接，在启动QEMU后运行`gdb`。 该文件的作用是自动将GDB指向QEMU,加载内核符号文件，并在16位和32位模式之间切换。退出GDB将关闭QEMU。

**常用命令**

`make qemu` --- 构建所有内容并使用新窗口中的 VGA 控制台和终端中的串行控制台启动 QEMU。要退出，请关闭 VGA 窗口或在终端中按 Ctrl-c 或 Ctrl-a x。

`make qemu-nox` --- 与 make qemu 类似，但仅使用串行控制台(serial console)运行。要退出，请按 Ctrl-a x。这在通过 SSH 连接到 Athena 拨号时特别有用，因为 VGA 窗口会消耗大量带宽。

`make qemu-gdb` --- 与 make qemu 类似，但不是在任何时候被动接受 GDB 连接，而是在第一条机器指令处暂停并等待 GDB 连接。

`make qemu-nox-gdb` ---- qemu-nox与qemu-gdb的组合。

`make run-name` --- 运行用户程序名称。例如，make run-hello 运行 user/hello.c。

`make run-name-nox` `run-name-gdb` `run-name-gdb-nox` --- 结合上述说明。 

`make V=1...` ---- 详细模式(Verbose mode)。打印出正在执行的每个命令，包括参数。

`make V=1 grade` --- 在任何失败的等级测试后停止，并将 QEMU 输出留在 jos.out 中以供检查。

`make QEMUEXTRA='args'...` ----  指定要传递给 QEMU 的其他参数。

```makefile
#
# This makefile system follows the structuring conventions
# recommended by Peter Miller in his excellent paper:
#
#	Recursive Make Considered Harmful
#	http://aegis.sourceforge.net/auug97.pdf
#
OBJDIR := obj

# Run 'make V=1' to turn on verbose commands, or 'make V=0' to turn them off.
ifeq ($(V),1)
override V =
endif
ifeq ($(V),0)
override V = @
endif

-include conf/lab.mk

-include conf/env.mk

LABSETUP ?= ./

TOP = .

# Cross-compiler jos toolchain
#
# This Makefile will automatically use the cross-compiler toolchain
# installed as 'i386-jos-elf-*', if one exists.  If the host tools ('gcc',
# 'objdump', and so forth) compile for a 32-bit x86 ELF target, that will
# be detected as well.  If you have the right compiler toolchain installed
# using a different name, set GCCPREFIX explicitly in conf/env.mk

# try to infer the correct GCCPREFIX
ifndef GCCPREFIX
GCCPREFIX := $(shell if i386-jos-elf-objdump -i 2>&1 | grep '^elf32-i386$$' >/dev/null 2>&1; \
	then echo 'i386-jos-elf-'; \
	elif objdump -i 2>&1 | grep 'elf32-i386' >/dev/null 2>&1; \
	then echo ''; \
	else echo "***" 1>&2; \
	echo "*** Error: Couldn't find an i386-*-elf version of GCC/binutils." 1>&2; \
	echo "*** Is the directory with i386-jos-elf-gcc in your PATH?" 1>&2; \
	echo "*** If your i386-*-elf toolchain is installed with a command" 1>&2; \
	echo "*** prefix other than 'i386-jos-elf-', set your GCCPREFIX" 1>&2; \
	echo "*** environment variable to that prefix and run 'make' again." 1>&2; \
	echo "*** To turn off this error, run 'gmake GCCPREFIX= ...'." 1>&2; \
	echo "***" 1>&2; exit 1; fi)
endif

# try to infer the correct QEMU
ifndef QEMU
QEMU := $(shell if which qemu >/dev/null 2>&1; \
	then echo qemu; exit; \
        elif which qemu-system-i386 >/dev/null 2>&1; \
        then echo qemu-system-i386; exit; \
	else \
	qemu=/Applications/Q.app/Contents/MacOS/i386-softmmu.app/Contents/MacOS/i386-softmmu; \
	if test -x $$qemu; then echo $$qemu; exit; fi; fi; \
	echo "***" 1>&2; \
	echo "*** Error: Couldn't find a working QEMU executable." 1>&2; \
	echo "*** Is the directory containing the qemu binary in your PATH" 1>&2; \
	echo "*** or have you tried setting the QEMU variable in conf/env.mk?" 1>&2; \
	echo "***" 1>&2; exit 1)
endif

# try to generate a unique GDB port
GDBPORT	:= $(shell expr `id -u` % 5000 + 25000)

CC	:= $(GCCPREFIX)gcc -pipe
AS	:= $(GCCPREFIX)as
AR	:= $(GCCPREFIX)ar
LD	:= $(GCCPREFIX)ld
OBJCOPY	:= $(GCCPREFIX)objcopy
OBJDUMP	:= $(GCCPREFIX)objdump
NM	:= $(GCCPREFIX)nm

# Native commands
NCC	:= gcc $(CC_VER) -pipe
NATIVE_CFLAGS := $(CFLAGS) $(DEFS) $(LABDEFS) -I$(TOP) -MD -Wall
TAR	:= gtar
PERL	:= perl

# Compiler flags
# -fno-builtin is required to avoid refs to undefined functions in the kernel.
# Only optimize to -O1 to discourage inlining, which complicates backtraces.
CFLAGS := $(CFLAGS) $(DEFS) $(LABDEFS) -O1 -fno-builtin -I$(TOP) -MD
CFLAGS += -fno-omit-frame-pointer
CFLAGS += -std=gnu99
CFLAGS += -static
CFLAGS += -Wall -Wno-format -Wno-unused -Werror -gstabs -m32
# -fno-tree-ch prevented gcc from sometimes reordering read_ebp() before
# mon_backtrace()'s function prologue on gcc version: (Debian 4.7.2-5) 4.7.2
CFLAGS += -fno-tree-ch

# Add -fno-stack-protector if the option exists.
CFLAGS += $(shell $(CC) -fno-stack-protector -E -x c /dev/null >/dev/null 2>&1 && echo -fno-stack-protector)

# Common linker flags
LDFLAGS := -m elf_i386

# Linker flags for JOS user programs
ULDFLAGS := -T user/user.ld

GCC_LIB := $(shell $(CC) $(CFLAGS) -print-libgcc-file-name)

# Lists that the */Makefrag makefile fragments will add to
OBJDIRS :=

# Make sure that 'all' is the first target
all:

# Eliminate default suffix rules
.SUFFIXES:

# Delete target files if there is an error (or make is interrupted)
.DELETE_ON_ERROR:

# make it so that no intermediate .o files are ever deleted
.PRECIOUS: %.o $(OBJDIR)/boot/%.o $(OBJDIR)/kern/%.o \
	   $(OBJDIR)/lib/%.o $(OBJDIR)/fs/%.o $(OBJDIR)/net/%.o \
	   $(OBJDIR)/user/%.o

KERN_CFLAGS := $(CFLAGS) -DJOS_KERNEL -gstabs
USER_CFLAGS := $(CFLAGS) -DJOS_USER -gstabs

# Update .vars.X if variable X has changed since the last make run.
#
# Rules that use variable X should depend on $(OBJDIR)/.vars.X.  If
# the variable's value has changed, this will update the vars file and
# force a rebuild of the rule that depends on it.
$(OBJDIR)/.vars.%: FORCE
	$(V)echo "$($*)" | cmp -s $@ || echo "$($*)" > $@
.PRECIOUS: $(OBJDIR)/.vars.%
.PHONY: FORCE


# Include Makefrags for subdirectories
include boot/Makefrag
include kern/Makefrag


QEMUOPTS = -drive file=$(OBJDIR)/kern/kernel.img,index=0,media=disk,format=raw -serial mon:stdio -gdb tcp::$(GDBPORT)
QEMUOPTS += $(shell if $(QEMU) -nographic -help | grep -q '^-D '; then echo '-D qemu.log'; fi)
IMAGES = $(OBJDIR)/kern/kernel.img
QEMUOPTS += $(QEMUEXTRA)

.gdbinit: .gdbinit.tmpl
	sed "s/localhost:1234/localhost:$(GDBPORT)/" < $^ > $@

gdb:
	gdb -n -x .gdbinit

pre-qemu: .gdbinit

qemu: $(IMAGES) pre-qemu
	$(QEMU) $(QEMUOPTS)

qemu-nox: $(IMAGES) pre-qemu
	@echo "***"
	@echo "*** Use Ctrl-a x to exit qemu"
	@echo "***"
	$(QEMU) -nographic $(QEMUOPTS)

qemu-gdb: $(IMAGES) pre-qemu
	@echo "***"
	@echo "*** Now run 'make gdb'." 1>&2
	@echo "***"
	$(QEMU) $(QEMUOPTS) -S

qemu-nox-gdb: $(IMAGES) pre-qemu
	@echo "***"
	@echo "*** Now run 'make gdb'." 1>&2
	@echo "***"
	$(QEMU) -nographic $(QEMUOPTS) -S

print-qemu:
	@echo $(QEMU)

print-gdbport:
	@echo $(GDBPORT)

# For deleting the build
clean:
	rm -rf $(OBJDIR) .gdbinit jos.in qemu.log

realclean: clean
	rm -rf lab$(LAB).tar.gz \
		jos.out $(wildcard jos.out.*) \
		qemu.pcap $(wildcard qemu.pcap.*) \
		myapi.key

distclean: realclean
	rm -rf conf/gcc.mk

ifneq ($(V),@)
GRADEFLAGS += -v
endif

grade:
	@echo $(MAKE) clean
	@$(MAKE) clean || \
	  (echo "'make clean' failed.  HINT: Do you have another running instance of JOS?" && exit 1)
	./grade-lab$(LAB) $(GRADEFLAGS)

git-handin: handin-check
	@if test -n "`git config remote.handin.url`"; then \
		echo "Hand in to remote repository using 'git push handin HEAD' ..."; \
		if ! git push -f handin HEAD; then \
            echo ; \
			echo "Hand in failed."; \
			echo "As an alternative, please run 'make tarball'"; \
			echo "and visit http://pdos.csail.mit.edu/6.828/submit/"; \
			echo "to upload lab$(LAB)-handin.tar.gz.  Thanks!"; \
			false; \
		fi; \
    else \
		echo "Hand-in repository is not configured."; \
		echo "Please run 'make handin-prep' first.  Thanks!"; \
		false; \
	fi

WEBSUB := https://6828.scripts.mit.edu/2018/handin.py

handin: tarball-pref myapi.key
	@SUF=$(LAB); \
	test -f .suf && SUF=`cat .suf`; \
	curl -f -F file=@lab$$SUF-handin.tar.gz -F key=\<myapi.key $(WEBSUB)/upload \
	    > /dev/null || { \
		echo ; \
		echo Submit seems to have failed.; \
		echo Please go to $(WEBSUB)/ and upload the tarball manually.; }

handin-check:
	@if ! test -d .git; then \
		echo No .git directory, is this a git repository?; \
		false; \
	fi
	@if test "$$(git symbolic-ref HEAD)" != refs/heads/lab$(LAB); then \
		git branch; \
		read -p "You are not on the lab$(LAB) branch.  Hand-in the current branch? [y/N] " r; \
		test "$$r" = y; \
	fi
	@if ! git diff-files --quiet || ! git diff-index --quiet --cached HEAD; then \
		git status -s; \
		echo; \
		echo "You have uncomitted changes.  Please commit or stash them."; \
		false; \
	fi
	@if test -n "`git status -s`"; then \
		git status -s; \
		read -p "Untracked files will not be handed in.  Continue? [y/N] " r; \
		test "$$r" = y; \
	fi

UPSTREAM := $(shell git remote -v | grep "pdos.csail.mit.edu/6.828/2018/jos.git (fetch)" | awk '{split($$0,a," "); print a[1]}')

tarball-pref: handin-check
	@SUF=$(LAB); \
	if test $(LAB) -eq 3 -o $(LAB) -eq 4; then \
		read -p "Which part would you like to submit? [a, b, c (c for lab 4 only)]" p; \
		if test "$$p" != a -a "$$p" != b; then \
			if test ! $(LAB) -eq 4 -o ! "$$p" = c; then \
				echo "Bad part \"$$p\""; \
				exit 1; \
			fi; \
		fi; \
		SUF="$(LAB)$$p"; \
		echo $$SUF > .suf; \
	else \
		rm -f .suf; \
	fi; \
	git archive --format=tar HEAD > lab$$SUF-handin.tar; \
	git diff $(UPSTREAM)/lab$(LAB) > /tmp/lab$$SUF-diff.patch; \
	tar -rf lab$$SUF-handin.tar /tmp/lab$$SUF-diff.patch; \
	gzip -c lab$$SUF-handin.tar > lab$$SUF-handin.tar.gz; \
	rm lab$$SUF-handin.tar; \
	rm /tmp/lab$$SUF-diff.patch; \

myapi.key:
	@echo Get an API key for yourself by visiting $(WEBSUB)/
	@read -p "Please enter your API key: " k; \
	if test `echo "$$k" |tr -d '\n' |wc -c` = 32 ; then \
		TF=`mktemp -t tmp.XXXXXX`; \
		if test "x$$TF" != "x" ; then \
			echo "$$k" |tr -d '\n' > $$TF; \
			mv -f $$TF $@; \
		else \
			echo mktemp failed; \
			false; \
		fi; \
	else \
		echo Bad API key: $$k; \
		echo An API key should be 32 characters long.; \
		false; \
	fi;

#handin-prep:
#	@./handin-prep


# This magic automatically generates makefile dependencies
# for header files included from C source files we compile,
# and keeps those dependencies up-to-date every time we recompile.
# See 'mergedep.pl' for more information.
$(OBJDIR)/.deps: $(foreach dir, $(OBJDIRS), $(wildcard $(OBJDIR)/$(dir)/*.d))
	@mkdir -p $(@D)
	@$(PERL) mergedep.pl $@ $^

-include $(OBJDIR)/.deps

always:
	@:

.PHONY: all always \
	handin git-handin tarball tarball-pref clean realclean distclean grade handin-prep handin-check
```

### JOS obj/

构建JOS时makefile输出的额外文件，可能在调试时有用。

``obj/boot/boot.asm`**,** `obj/kern/kernel.asm`**,** `obj/user/hello.asm`,`etc` 

Assembly code listings for the bootloader, kernel, and user programs.

引导加载程序、内核和用户程序的汇编代码列表。



``obj/kern/kernel.sym`**,** `obj/user/hello.sym`, `etc` -----

Symbol tables for the kernel and user programs.

内核和用户程序的符号表。



``obj/boot/boot.out`**,** `obj/kern/kernel`**,** `obj/user/hello`,` etc` ----

Linked ELF images of the kernel and user programs. These contain symbol information that can be used by GDB.

内核和用户程序的链接 ELF 映像。这些包含 GDB 可以使用的符号信息。





### GDB

- **EIP** 

EIP is a register in x86 architectures (32bit). It holds the "Extended Instruction Pointer" for the stack. In other words, it tells the computer 

x86 架构（32 位）中的寄存器。它保存堆栈的“扩展指令指针”。换句话说，它告诉计算机下一步去哪里执行下一个命令并控制程序的流程。



- [GDB manual](https://sourceware.org/gdb/current/onlinedocs/gdb/)

`Ctrl-c` -- 停止机器并按照当前指令进入 GDB。如果 QEMU 有多个虚拟 CPU，这会停止所有虚拟 CPU。

`c (continue)`  ---- 继续执行直到下一个断点或 `Ctrl-c`。



`si (stepi)` --- 执行一条机器指令。



`b function | b file:line  (breakpoint)` --- 在给定的函数或行上设置断点。



`b *addr (breakpoint)` ---- 在 EIP 地址处设置断点。



`set print pretty` --- Enable pretty-printing of arrays and structs.

`info registers` ----  打印通用寄存器、eip、eflags 和段选择器(segment selectors)。要获得更彻底的机器寄存器状态转储，请参阅 QEMU 自己的 info registers 命令。



`x/Nx addr` ---- 显示从 addr 开始的 N 条汇编指令。使用 `$eip` 作为 addr的值将显示当前指令指针处的所有指令。



`x/Ni addr` --- 显示从虚拟地址 addr 开始的 N 个字的十六进制转储。如果省略 N，则默认为 1。 addr 可以是任何表达式。



`symbol-file file` --- 切换到符号文件`file`。当 GDB 连接到 QEMU 时，它没有虚拟机内进程边界的概念，因此我们必须告诉它使用哪些符号。默认情况下，我们将 GDB 配置为使用内核符号文件 `obj/kern/kernel`。如果机器正在运行用户代码(user code)，比如 hello.c，你可以使用`symbol-file obj/user/hello` 切换到 hello symbol file。



QEMU 将每个虚拟 CPU 表示为 GDB 中的一个线程，因此您可以使用 GDB 的所有线程相关命令来查看或操作 QEMU 的虚拟 CPU。

`thread n` ---- GDB 一次只关注一个线程（即 CPU）。此命令将焦点切换到线程 n，从零开始编号。

`info threads` ---- 列出所有线程（即 CPU），包括它们的状态（活动或停止）以及它们处于什么功能。



### QEMU

[QEMU Monitor](https://qemu-project.gitlab.io/qemu/system/monitor.html)

QEMU是一种通用的开源计算机仿真器和虚拟器。QEMU共有两种操作模式

- 全系统仿真：能够在任意支持的架构上为任何机器运行一个完整的操作系统

- 用户模式仿真：能够在任意支持的架构上为另一个Linux/BSD运行程序

同时当进行虚拟化时，QEMU也可以以接近本机的性能运行KVM或者Xen。

QEMU 包含一个内置监视器，方便检查和修改机器状态。要进入监视器，请在运行 QEMU 的终端中按 Ctrl-a c。再次按 Ctrl-a c 切换回串行控制台。



### 常用命令

`xp/Nx paddr` --- 显示从物理地址 paddr 开始的 N 个字的十六进制转储(a hex dump)。如果省略 N，则默认为 1。这是 GDB 的 x 命令的物理内存模拟。

`info registers` --- 显示机器内部寄存器状态的完整转储(a full dump)。特别是，这包括段选择器(segment selectors)和本地(local)、全局(global)和中断描述符表(interrupt descriptor tables以及任务寄存器(task register)的机器隐藏段状态(hidden segment state)。这种隐藏状态是当加载段选择器时虚拟 CPU 从 GDT/LDT 读取的信息。下面是在lab 1的JOS内核中运行时的CS以及每个字段的含义：

```assembly
CS =0008 10000000 ffffffff 10cf9a00 DPL=0 CS32 [-R-]
```

**CS = 0008** 

--- 代码选择器(code selector)的可见部分。我们正在使用段 0x8。这也告诉我们:我们指的是全局描述符表[global descriptor table]（0x8&4=0），我们的 CPL[current privilege level]（当前权限级别）是 0x8&3=0。

**10000000** 

---- 此段的基址。线性地址 = 逻辑地址 + 0x10000000

**ffffffff**

----- 此段的地址上限。高于0xffffffff 的线性地址将导致段违规异常(越界异常)。

**10cf9a00**    ??????

----- 该段的原始标志(raw flags)，QEMU 在接下来的几个字段中帮助我们解码。

**DPL=0**

--- 此段的权限级别。只有以权限级别0运行的代码才能加载此段。

**CS32**    ????? 

----- 这是一个 32 位代码段。其他值包括数据段的 DS（不要与 DS 寄存器混淆）和本地描述符表的 LDT。

**[-R-]**

---- 该段只允许读操作。



`info mem`  ------ 显示映射的虚拟内存和权限(permissions)。

```assembly
ef7c0000-ef800000 00040000 urw
efbf8000-efc00000 00008000 -rw
```

**ef7c0000-ef800000 00040000 urw**

----- 从 0xef7c0000 到 0xef800000 的 0x00040000 字节内存被映射为读/写和用户可访问.

**efbf8000-efc00000 00008000 -rw**

---- 从 0xefbf8000 到 0xefc00000 的内存被映射为读/写，但只有内核可访问。



`info pg`  ---- 显示当前页表结构(page table structure)。输出类似于 `info mem`，但区分页目录条目(page directory entries)和页表条目(page table entriies)，并分别赋予每个条目的权限。重复的 PTE's 和整个页表折叠成一行。  **??????**

```
VPN range     Entry         Flags        Physical page
[00000-003ff]  PDE[000]     -------UWP
  [00200-00233]  PTE[200-233] -------U-P 00380 0037e 0037d 0037c 0037b 0037a ..
[00800-00bff]  PDE[002]     ----A--UWP
  [00800-00801]  PTE[000-001] ----A--U-P 0034b 00349
  [00802-00802]  PTE[002]     -------U-P 00348
```

这显示了两个页目录条目，分别跨越虚拟地址 0x00000000 到 0x003fffff 和 0x00800000 到 0x00bfffff。两个 PDE 都存在、可写，并且用户也可以访问第二个 PDE。这些页表中的第二个映射三个页面，跨越虚拟地址 0x00800000 到 0x00802fff，其中前两个是存在、用户和访问的，第三个是仅存在和用户。这些 PTE 的第一个映射物理页面 0x34b。



```
QEMU 还接受一些有用的命令行参数，可以使用 QEMUEXTRA 变量将它们传递到 JOS 生成文件中。
```

`make QEMUEXTRA='-d int' ... ` ---- 将所有中断以及完整的寄存器转储记录到 qemu.log。您可以忽略前两个日志条目“SMM: enter”和“SMM: after RMS”，因为它们是在进入引导加载程序之前生成的。在此之后，日志条目看起来像

```
     4: v=30 e=0000 i=1 cpl=3 IP=001b:00800e2e pc=00800e2e SP=0023:eebfdf28 EAX=00000005
EAX=00000005 EBX=00001002 ECX=00200000 EDX=00000000
ESI=00000805 EDI=00200000 EBP=eebfdf60 ESP=eebfdf28
...
```

第一行描述了中断。

 4：只是一个日志记录计数器。

 v : 以十六进制给出向量编号。

 e : 给出错误代码。

 i=1 :  表示这是由 int 指令产生的（而不是硬件中断）。该行的其余部分应该是不言自明的。查看`info pg`
注意：如果您运行的是 0.15 之前的 QEMU 版本，日志将写入 /tmp 而不是当前目录。





## GDB + QEMU 调试

打开两个终端

- 一个终端在`lab`目录下运行

```shell
make qemu-gdb
```

此时模拟器会卡住

- 另一个终端运行

```shell
gdb

# 进入gdb
target remote localhost:26000   # 连接端口
symbol‐file obj/kern/kernel     # 切换到内核符号文件

# 使用gdb命令进行调试
b main
b exec
c
si
file user/ls    # 调试特定文件
...
```



