---
layout: post
title: "kernel build"
category: kernel
tags: [kernel,kbuild]
---
{% include JB/setup %}


根据kernel-5.19的源码进行分析，探索kernel build的过程。

## makefiles

makefiles分为如下几类：

- 顶层目录下的Makefile: $(srctree)/makefile。定义全局的内容，称之为src Makefile。
- arch目录下的Makefile: arch/$(SRCARCH)/makefile。定义和arch相关的内容，称之为arch Makefile。
- kbuild系统的makefiles，存放于scripts/目录下，最重要的是Makefile.build。
- 子目录下的Makefile，比如fs/ext4/Makefile、arch/x86/boot/Makefile。

makefiles间的关系：

- make始终运行在顶层目录下，使用src Makefile。所有sub make通过make -f {sub_makefile}执行，所以所有的make的cwd都是顶层目录。
- src Makefile包含arch Makefile。
- src Makefile包含kbuild系统的makefiles，比如scripts/Makefile.build、scripts/Kbuild.include。
- scripts/Makefile.build是kbuild系统的入口，*make -f scripts/Makefile.build* 进行kbuild的构建。
- 子目录的Makefile是Kbuild系统的一部分，他们被kbuild系统使用，子目录下的构建由kbuild完成。



## kbuild

kbuild是基于makefile实现的一个构建系统。kbuild系统的makefiles保存在scripts/，最重要的文件是Makefile.build。

他是kbuild的入口。kbuild的构建命令如下：

​					`make -f scripts/Makefile.build obj=<subdir> [target]`

scripts/Kbuild.include中定义了build变量									

```
build := -f $(srctree)/scripts/Makefile.build obj                                 
```

对于包含了Kbuild.include的Makefile(比如src Makefile)，简洁地使用如下命令构建：

​				         `$(MAKE) $(build)=<subdir> [target]`

对于kbuild，obj变量指定构建的生成目录，src变量指定构建的源目录(源文件存放的目录)。Makefile.build默认src等于obj：

```
src := $(obj)
```

所以obj、src通常都指向了子目录。

Makefile.build使用如下命令包含子目录的Makefile:

```
kbuild-dir := $(if $(filter /%,$(src)),$(src),$(srctree)/$(src))
include $(or $(wildcard $(kbuild-dir)/Kbuild),$(kbuild-dir)/Makefile)
```

kbuild-dir是对相对路径进行了处理。

因为子目录下的makefile也是Kbuild系统中的一部分，按规定其文件名应为kbuild，只不过人们出于习惯还是会命令为makefile。Kbuild为此做了兼容。

### 子目录的构建

#### 钩子变量

Makefile.build中定义了一系列的钩子变量：

```
obj-y :=
obj-m :=
lib-y :=
lib-m :=
always-y :=
always-m :=
.....
```

**子目录的makefile正是利用这些钩子变量加入kbuild系统。**

下面通过两个例子来具体分析子目录的构建：

#### ext4的构建

在顶层*make*时，对于需要构建的子目录进行构建，比如fs/，构建规则如下：

```
fs: prepare
    $(Q)$(MAKE) $(build)=$@ \
    single-build=$(if $(filter-out $@/, $(filter $@/%, $(KBUILD_SINGLE_TARGETS))),1) \
    need-builtin=1 need-modorder=1
```

使用kbuild构建子目录fs，下面具体分析：

查看Makefile.build的default goal:

```
PHONY := __build
__build:      #first rule

ifdef single-build
......
else
__build: $(if $(KBUILD_BUILTIN), $(targets-for-builtin)) \
     $(if $(KBUILD_MODULES), $(targets-for-modules)) \
     $(subdir-ym) $(always-y)
    @:
endif
```

default goal是\_\_build。查看与之相关的变量：

```
targets-for-builtin := $(extra-y)

ifneq ($(strip $(lib-y) $(lib-m) $(lib-)),)
targets-for-builtin += $(obj)/lib.a
endif

ifdef need-builtin
targets-for-builtin += $(obj)/built-in.a
endif

targets-for-modules := $(foreach x, o mod $(if $(CONFIG_TRIM_UNUSED_KSYMS), usyms), \
                $(patsubst %.o, %.$x, $(filter %.o, $(obj-m))))

ifdef need-modorder
targets-for-modules += $(obj)/modules.order
endif

$(subdir-ym):
    $(Q)$(MAKE) $(build)=$@ \
    $(if $(filter $@/, $(KBUILD_SINGLE_TARGETS)),single-build=) \
    need-builtin=$(if $(filter $@/built-in.a, $(subdir-builtin)),1) \
    need-modorder=$(if $(filter $@/modules.order, $(subdir-modorder)),1)


# from scripts/Makefile.lib
subdir-ym := $(sort $(subdir-y) $(subdir-m) \
            $(patsubst %/,%, $(filter %/, $(obj-y) $(obj-m))))
subdir-ym   := $(addprefix $(obj)/,$(subdir-ym))

```

可以看出这些变量都是由钩子变量extra-y、lib-y、lib-m、lib-、obj-y、obj-m、subdir-y、subdir-m生成。makefile.build包含子目录的makefile(上文提过):

```
include $(or $(wildcard $(kbuild-dir)/Kbuild),$(kbuild-dir)/Makefile)
```

这里子目录的makefile是fs/makefile，该文件对钩子函数进行赋值，其中有如下内容：

```
obj-$(CONFIG_EXT4_FS)       += ext4/
```

这个最终会导致subdir-ym中包含fs/ext4，从而触发子目录fs/ext4/的构建。fs/ext4/Makefile的内容如下：

```
 # fs/ext4/Makefile
 obj-$(CONFIG_EXT4_FS) += ext4.o
  
 ext4-y  := balloc.o bitmap.o dir.o file.o fsync.o ialloc.o inode.o page-io.o \
         ioctl.o namei.o super.o symlink.o hash.o resize.o extents.o \
         ext4_jbd2.o migrate.o mballoc.o block_validity.o move_extent.o \
         mmp.o indirect.o extents_status.o xattr.o xattr_user.o \
         xattr_trusted.o inline.o
  
 ext4-$(CONFIG_EXT4_FS_POSIX_ACL)    += acl.o
 ext4-$(CONFIG_EXT4_FS_SECURITY)     += xattr_security.o
```

这决定了__build的最终规则如下：

```
__build: fs/ext4/built-in.a fs/ext4/modules.order
    @:
```

我们这里只分析fs/ext4/built-in.a，他的规则由Makefile.build定义：

```
$(obj)/built-in.a: $(real-obj-y) FORCE
    $(call if_changed,ar_builtin)
```

real-obj-y由scripts/Makefile.lib定义，也是由钩子变量生成:

```
real-obj-y := $(call real-search, $(obj-y), .o, -objs -y)
real-obj-y  := $(addprefix $(obj)/,$(real-obj-y))
```

real-obj-y值最终如下，**注意子目录Makefile定义了ext4-y，所以ext4.o被替换了ext4-y**：

```
real-obj-y := fs/ext4/balloc.o fs/ext4/bitmap.o fs/ext4/block_validity.o fs/ext4/dir.o fs/ext4/ext4_jbd2.o fs/ext4/extents.o fs/ext4/extents_status.o fs/ext4/file.o fs/ext4/fsmap.o fs/ext4/fsync.o fs/ext4/hash.o fs/ext4/ialloc.o fs/ext4/indirect.o fs/ext4/inline.o fs/ext4/inode.o fs/ext4/ioctl.o fs/ext4/mballoc.o fs/ext4/migrate.o fs/ext4/mmp.o fs/ext4/move_extent.o fs/ext4/namei.o fs/ext4/page-io.o fs/ext4/readpage.o fs/ext4/resize.o fs/ext4/super.o fs/ext4/symlink.o fs/ext4/sysfs.o fs/ext4/xattr.o fs/ext4/xattr_hurd.o fs/ext4/xattr_trusted.o fs/ext4/xattr_user.o fs/ext4/fast_commit.o fs/ext4/orphan.o fs/ext4/acl.o fs/ext4/xattr_security.o fs/ext4/verity.o fs/ext4/crypto.o
```

这些目标文件的规则比较复杂，以fs/ext4/balloc.o为例，他的recipe来自Makefile.build：

```
$(obj)/%.o: $(src)/%.c $(recordmcount_source) FORCE
    $(call if_changed_rule,cc_o_c)
    $(call cmd,force_checksrc)
```

还有一部分依赖来自fs/ext4/.balloc.o.cmd文件，这个文件是自动生成的，目前暂不清楚怎么生成的，Makefile.build包含这些文件：

```
-include $(foreach f,$(existing-targets),$(dir $(f)).$(notdir $(f)).cmd)
```

对于fs/ext4/balloc.o，他的规则最终如下：

```
fs/ext4/balloc.o: fs/ext4/balloc.c FORCE include/linux/compiler-version.h ......(很多.h的依赖)
    $(call if_changed_rule,cc_o_c)
    $(call cmd,force_checksrc)
    
cmd_cc_o_c = $(CC) $(c_flags) -c -o $@ $< $(cmd_ld_single_m) $(cmd_objtool)
```

可以看出fs/ext4/balloc.o就是由源文件CC而来。

回到fs/ext4/built-in.a规则，他的构建为如下命令：

```
cmd_ar_builtin = rm -f $@; echo $(patsubst $(obj)/%,%,$(real-prereqs)) | sed -E 's:([^ ]+):$(obj)/\1:g' | xargs $(AR) cDPrST $@
```

可以看出fs/ext4/built-in.a就是对众多目标文件进行ar而生成。

#### bzImage的构建

bzImage构成是通过如下kbuild命令构建：

​                                  `make -f scripts/Makefile.build obj=arch/x86/boot arch/x86/boot/bzImage`

该命令指定了goal，而不再是使用默认的__build。

具体分析见下面。

## kernel build

下面详细分析在x86_64平台下对5.19内核运行*make* 进行kernel build的过程：

### default goal

```
PHONY := __all
__all:  

ifeq ($(KBUILD_EXTMOD),)   
__all: all                
else
__all: modules
endif

all: vmlinux
all: bzImage      #arch/x86/Makefile
all: modules
ifdef  	
all: scripts_gdb    
endif
```

第一个规则的目标为__all，即为 default goal。依次查找规则至all，查看database。

```
.DEFAULT_GOAL := __all
KBUILD_EXTMOD :=
__all: all
CONFIG_GDB_SCRIPTS = y   # from .config
all: vmlinux bzImage modules scripts_gdb
```

我们依次分析vmlinux bzImage modules scripts_gdb

### vmlinux

```
vmlinux: scripts/link-vmlinux.sh autoksyms_recursive $(vmlinux-deps) FORCE
    +$(call if_changed_dep,link-vmlinux)

# from Kbuild.include
if_changed_dep = $(if $(if-changed-cond),$(cmd_and_fixdep),@:)
cmd_and_fixdep =                                                             \
    $(cmd);                                                              \
    scripts/basic/fixdep $(depfile) $@ '$(make-cmd)' > $(dot-target).cmd;\
    rm -f $(depfile)
cmd = @set -e; $(echo-cmd) $($(quiet)redirect) $(cmd_$(1))

# from src Makefile
cmd_link-vmlinux =                                                 \
    $(CONFIG_SHELL) $< "$(LD)" "$(KBUILD_LDFLAGS)" "$(LDFLAGS_vmlinux)";    \
    $(if $(ARCH_POSTLINK), $(MAKE) -f $(ARCH_POSTLINK) $@, true)

```

$(cmd_$(1))展开后为cmd_link-vmlinux，在cmd_link-vmlinux中，$<为scripts/link-vmlinux.sh。vmlinux的正是由这个脚本生成的。

查看scripts/link-vmlinux.sh：

```
objs="${KBUILD_VMLINUX_OBJS}"
libs="${KBUILD_VMLINUX_LIBS}"
 
${ld} ${ldflags} -o ${output}                   \
	${wl}--whole-archive ${objs} ${wl}--no-whole-archive    \
	${wl}--start-group ${libs} ${wl}--end-group     \
	$@ ${ldlibs}
```

变量KBUILD_VMLINUX_OBJS、KBUILD_VMLINUX_LIBS是ld的源。这两个变量源自make。 

```
# Externally visible symbols (used by link-vmlinux.sh)
KBUILD_VMLINUX_OBJS := $(head-y) $(patsubst %/,%/built-in.a, $(core-y))
KBUILD_VMLINUX_OBJS += $(addsuffix built-in.a, $(filter %/, $(libs-y)))
ifdef CONFIG_MODULES
KBUILD_VMLINUX_OBJS += $(patsubst %/, %/lib.a, $(filter %/, $(libs-y)))
KBUILD_VMLINUX_LIBS := $(filter-out %/, $(libs-y))
else
KBUILD_VMLINUX_LIBS := $(patsubst %/,%/lib.a, $(libs-y))
endif
KBUILD_VMLINUX_OBJS += $(patsubst %/,%/built-in.a, $(drivers-y))

export KBUILD_VMLINUX_OBJS KBUILD_VMLINUX_LIBS
export KBUILD_LDS          := arch/$(SRCARCH)/kernel/vmlinux.lds
```

这两个变量源于**head-y、core-y、libs-y、drivers-y**，这些变量的定义源于src Makefile、arch Makefile。查看database：

```
CONFIG_MODULES = y   #from .config

head-y := arch/x86/kernel/head_64.o arch/x86/kernel/head64.o arch/x86/kernel/ebda.o arch/x86/kernel/platform-quirks.o
core-y := init/ usr/ arch/x86/ kernel/ certs/ mm/ fs/ ipc/ security/ crypto/ block/
libs-y := lib/ arch/x86/lib/
drivers-y := drivers/ sound/ samples/ net/ virt/ arch/x86/pci/ arch/x86/power/ arch/x86/video/

KBUILD_VMLINUX_OBJS := arch/x86/kernel/head_64.o arch/x86/kernel/head64.o arch/x86/kernel/ebda.o arch/x86/kernel/platform-quirks.o init/built-in.a usr/built-in.a arch/x86/built-in.a kernel/built-in.a certs/built-in.a mm/built-in.a fs/built-in.a ipc/built-in.a security/built-in.a crypto/built-in.a          block/built-in.a lib/built-in.a arch/x86/lib/built-in.a  lib/lib.a  arch/x86/lib/lib.a drivers/built-in.a sound/built-in.a samples/built-in.a net/built-in.a virt/built-in.a arch/x86/pci/built-in.a arch/x86/power/built-in.a arch/x86/video/built-in.a

KBUILD_VMLINUX_LIBS :=
```

KBUILD_VMLINUX_OBJS、KBUILD_VMLINUX_LIBS是怎么生成的呢，我们可以看到vmlinux依赖$(vmlinux-deps)：

```
vmlinux-deps := $(KBUILD_LDS) $(KBUILD_VMLINUX_OBJS) $(KBUILD_VMLINUX_LIBS)

$(sort $(vmlinux-deps) $(subdir-modorder)): descend ;

descend: $(build-dirs)
$(build-dirs): prepare
    $(Q)$(MAKE) $(build)=$@ \
    single-build=$(if $(filter-out $@/, $(filter $@/%, $(KBUILD_SINGLE_TARGETS))),1) \
    need-builtin=1 need-modorder=1

build-dirs := $(foreach d, $(build-dirs), \
            $(if $(filter $(d)/%, $(KBUILD_SINGLE_TARGETS)), $(d)))
```

可以发现，KBUILD_VMLINUX_OBJS、KBUILD_VMLINUX_LIBS的都通过kbuild生成，查看database：

```
vmlinux-deps := arch/x86/kernel/vmlinux.lds arch/x86/kernel/head_64.o arch/x86/kernel/head64.o arch/x86/kernel/ebda.o arch/x86/kernel/platform-quirks.o init/built-in.a usr/built-in.a arch/x86/built-in.a kernel/built-in.a certs/built-in.a mm/built-in.a fs/built-in.a ipc/built-in.a security/built-in.a crypto/built-in.a block/built-in.a lib/built-in.a arch/x86/lib/built-in.a  lib/lib.a  arch/x86/lib/lib.a drivers/built-in.a sound/built-in.a samples/built-in.a net/built-in.a virt/built-in.a arch/x86/pci/built-in.a arch/x86/power/built-in.a arch/x86/video/built-in.a

arch/x86/kernel/head_64.o: descend

descend: init usr arch/x86 kernel certs mm fs ipc security crypto block drivers sound samples net virt arch/x86/pci arch/x86/power arch/x86/video lib  arch/x86/lib

build-dirs := init usr arch/x86 kernel certs mm fs ipc security crypto block drivers sound samples net virt arch/x86/pci arch/x86/power arch/x86/video lib arch/x86/lib

init: prepare
    $(Q)$(MAKE) $(build)=$@ \
    single-build=$(if $(filter-out $@/, $(filter $@/%, $(KBUILD_SINGLE_TARGETS))),1) \
    need-builtin=1 need-modorder=1
```

build-dirs记录的就是需要构建的子目录，最后构建被展开为对kbuild对每个子目录的构建，就比如上面展示的init的规则。

回到scripts/link-vmlinux.sh，vmlinux最终由对KBUILD_VMLINUX_OBJS、KBUILD_VMLINUX_LIBS进行ld生成。

### bzImage

bzImage定义在arch Makefile

```
bzImage: vmlinux
ifeq ($(CONFIG_X86_DECODER_SELFTEST),y)
    $(Q)$(MAKE) $(build)=arch/x86/tools posttest
endif
    $(Q)$(MAKE) $(build)=$(boot) $(KBUILD_IMAGE)
    $(Q)mkdir -p $(objtree)/arch/$(UTS_MACHINE)/boot
    $(Q)ln -fsn ../../x86/boot/bzImage $(objtree)/arch/$(UTS_MACHINE)/boot/$@
    
boot := arch/x86/boot
KBUILD_IMAGE := $(boot)/bzImage

# from src Makefile
objtree     := .
ARCH        ?= $(SUBARCH)
UTS_MACHINE     := $(ARCH)

# from scripts/subarch.include
SUBARCH := $(shell uname -m | sed -e s/i.86/x86/ -e s/x86_64/x86/ \
                  -e s/sun4u/sparc64/ \
                  -e s/arm.*/arm/ -e s/sa110/arm/ \
                  -e s/s390x/s390/ \
                  -e s/ppc.*/powerpc/ -e s/mips.*/mips/ \
                  -e s/sh[234].*/sh/ -e s/aarch64.*/arm64/ \
                  -e s/riscv.*/riscv/ -e s/loongarch.*/loongarch/)
```

最终使用kbuild构建：

​					 `make -f scripts/Makefile.build obj=arch/x86/boot arch/x86/boot/bzImage`

子目录的makefile为arch/x86/boot/Makefile，定义了和bzImage相关的规则：

```
$(obj)/bzImage: $(obj)/setup.bin $(obj)/vmlinux.bin $(obj)/tools/build FORCE
    $(call if_changed,image)
    @$(kecho) 'Kernel: $@ is ready' ' (#'`cat .version`')'

$(obj)/vmlinux.bin: $(obj)/compressed/vmlinux FORCE
    $(call if_changed,objcopy)

$(obj)/setup.bin: $(obj)/setup.elf FORCE
    $(call if_changed,objcopy)
    
cmd_image = $(obj)/tools/build $(obj)/setup.bin $(obj)/vmlinux.bin \
                   $(obj)/zoffset.h $@ $($(quiet)redirect_image)
```

bzimage由vmlinux.bin和setup.bin通过tools/build生成。下面解析vmlinux.bin和setup.bin。

### vmlinux.bin

```
$(obj)/vmlinux.bin: $(obj)/compressed/vmlinux FORCE
    $(call if_changed,objcopy)
    
$(obj)/compressed/vmlinux: FORCE
    $(Q)$(MAKE) $(build)=$(obj)/compressed $@
```

vmlinux.bin是对compressed/vmlinux进行objcopy生成。而compressed/vmlinux是kubild构建：

构建规则来自子目录的Makefile——arch/x86/boot/compressed/Makefile：

```
$(obj)/vmlinux: $(vmlinux-objs-y) $(efi-obj-y) FORCE
    $(call if_changed,ld)
```

vmlinux-objs-y和efi-obj-y也都是在子目录的Makefile中赋值，最终规则如下:

```
arch/x86/boot/compressed/vmlinux: arch/x86/boot/compressed/vmlinux.lds arch/x86/boot/compressed/kernel_info.o arch/x86/boot/compressed/head_64.o arch/x86/boot/compressed/misc.o arch/x86/boot/compressed/string.o arch/x86/boot/compressed/cmdline.o arch/x86/boot/compressed/error.o arch/x86/boot/compressed/piggy.o arch/x86/boot/compressed/cpuflags.o arch/x86/boot/compressed/early_serial_console.o arch/x86/boot/compressed/kaslr.o arch/x86/boot/compressed/ident_map_64.o arch/x86/boot/compressed/idt_64.o arch/x86/boot/compressed/idt_handlers_64.o arch/x86/boot/compressed/mem_encrypt.o arch/x86/boot/compresse      d/pgtable_64.o arch/x86/boot/compressed/sev.o arch/x86/boot/compressed/acpi.o arch/x86/boot/compressed/tdx.o arch/x86/boot/compressed/tdcall.o arch/x86/boot/compressed/efi_thunk_64.o arch/x86/boot/compressed/efi.o drivers/firmware/efi/libstub/lib.a FORCE
    $(call if_changed,ld)

```

vmlinux就是对这些目标文件进行ld而生存。

arch/x86/boot/compressed/misc.o这些目标文件的构建过程和上面叙述的fs/ext4/balloc.o一样。**这里要特别注意arch/x86/boot/compressed/piggy.o 这个文件**。

他的最终规则如下：

```
arch/x86/boot/compressed/piggy.o: arch/x86/boot/compressed/piggy.S FORCE include/linux/compiler-version.h include/config/CC_VERSION_TEXT include/linux/kconfig.h include/linux/hidden.h
    $(call if_changed_rule,as_o_S)
```

他的源文件是piggy.S，其中有这么一条汇编指令：

```
.incbin "arch/x86/boot/compressed/vmlinux.bin.zst"
```

这条是指把vmlinux.bin.zst加到了section  .rodata..compressed中。

那么vmlinux.bin.zst是什么呢，怎么构建的呢,?直接查看最终规则：

```
arch/x86/boot/compressed/piggy.S: arch/x86/boot/compressed/vmlinux.bin.zst arch/x86/boot/compressed/mkpiggy FORCE
    $(call if_changed,mkpiggy)

arch/x86/boot/compressed/vmlinux.bin.zst: arch/x86/boot/compressed/vmlinux.bin arch/x86/boot/compressed/vmlinux.relocs FORCE
    $(call if_changed,zstd22_with_size)

arch/x86/boot/compressed/vmlinux.bin: vmlinux FORCE
    $(call if_changed,objcopy)
```

总结：

1. 上文描述的vmlinux，经过objdump，生存arch/x86/boot/compressed/vmlinux.bin。
2. arch/x86/boot/compressed/vmlinux.bin经过压缩生成arch/x86/boot/compressed/vmlinux.bin.zst。
3. arch/x86/boot/compressed/piggy.S把arch/x86/boot/compressed/vmlinux.bin.zst添加到源码中，最终存在于arch/x86/boot/compressed/piggy.o的section  .rodata..compressed中。
4. 对arch/x86/boot/compressed/piggy.o等目标文件ld生成arch/x86/boot/vmlinux。
5. 对arch/x86/boot/vmlinux进行objcopy生成arch/x86/boot/vmlinux.bin。

### setup.bin

```
# from arch/x86/boot/Makefile
$(obj)/setup.bin: $(obj)/setup.elf FORCE
    $(call if_changed,objcopy)

$(obj)/setup.elf: $(src)/setup.ld $(SETUP_OBJS) FORCE
    $(call if_changed,ld)

SETUP_OBJS = $(addprefix $(obj)/,$(setup-y))
```

setup-y由arch/x86/boot/Makefile赋值，最终setup.elf的规则：

```
arch/x86/boot/setup.elf: arch/x86/boot/setup.ld arch/x86/boot/a20.o arch/x86/boot/bioscall.o arch/x86/boot/cmdline.o arch/x86/boot/copy.o arch/x86/boot/cpu.o arch/x86/boot/cpuflags.o arch/x86/boot/cpucheck.o arch/x86/boot/early_serial_console.o arch/x86/boot/edd.o arch/x86/boot/header.o arch/x86/boot/main.o arch/x86/boot/memory.o arch/x86/boot/pm.o arch/x86/boot/pmjump.o arch/x86/boot/printf.o arch/x86/boot/regs.o arch/x86/boot/string.o arch/x86/boot/tty.o arch/x86/boot/video.o arch/x86/boot/video-mode.o arch/x86/boot/version.o arch/x86/boot/video-vga.o arch/x86/boot/video-vesa.o arch/x86/         boot/video-bios.o FORCE
    $(call if_changed,ld)
```

还是类似的构建过程，就不再复述。

### build kernel image

bzImage是最终的kernel image。他的构建过程如下：

1.构建./vmlinux。

2.构建./arch/x86/boot/vmlinux.bin，规则依赖./vmlinux。

3.构建./arch/x86/boot/setup.bin。

4.构建./bzimage，规则依赖./arch/x86/boot/vmlinux.bin和./arch/x86/boot/setup.bin。

### modules

todo

## kernel image 

### 构建过程

上文详细叙述kernel image(bzimage)的构建过程，和如下的参考图是一致的(细节上有些差别)。

<img src="https://github.com/Reventon1993/pictures/blob/master/typora/ba1471e4edc86d8c4e3cc00ac136d314.png" style="zoom: 40%;" />

我们这里探讨一下为什么构建规则这么复杂：

#### 压缩

早期kernel image需要压缩：

- 加载内核时为实模式，可访问的内存空间较小。

- 磁盘存储容量较小。


现在这两个问题都已不存在：加载内核时为保护模式、磁盘空间不再受限制。但是现在依旧保留了压缩内核的操作，主要是因为，解压缩的速度大于读磁盘的速度，所以这样做内核加载得更快。

图中的uncompressed代码包含了解压缩代码。解压缩所需要的信息保存在piggy.o中。

#### 内核重定位

内核可能是需要重定位的。uncompressed代码包含了重定位代码。重定位所需要的信息保存在piggy.o中。

#### setup.bin

setup.bin的作用如下：

* 收集硬件信息，传递数据给内核。

最初，setup.bin把收集到的硬件信息保存在struct boot_params params中(自己的数据段中)。然后把控制权交给vmlinux.bin前，把params数据复制到内核数据段中。

在最新的**32位启动协议(32-bit boot protocol)**规定中，这部分工作由bootloader完成。

* 切换到保护模式。

在最新的**32位启动协议**规定中，这部分工作由bootloader完成。

* 传递内核信息给bootloader。

在最新的**32位启动协议**规定中，这是setup.bin唯一的作用。

现在由最新的64位启动协议，和32位启动协议没有本质的区别。

### kernel image的布局

<img src="https://github.com/Reventon1993/pictures/blob/b123163acaf6dc0aa601476555084cf6379d7814/typora/image-20230710085454330.png" alt="image-20230710085454330" style="zoom:40%;" />
