# 4.1 生成可执行文件

## 4.1.1 样例代码生成main-lib
![编译链接生成main-lib可执行文件](images\编译链接生成main-lib可执行文件.PNG)

```commandline
# 编译addvec.c, multvec.c生成对应addvec.o和multvec.o
# -Og: 在保持代码易读性的前提下，尽可能提供较高的优化级别。如剔除未使用的变量，内联函数、删除冗余的代码等，但不会进行一些可能会降低代码可读性的优化，如函数调用的内联展开
# -g: 选项用于gdb调试
$ gcc -g -Og -c addvec.c multvec.c

# 可重定位(relocatable)文件包含了相对地址的指令和数据，可以在内存中进行重定位。
$ file addvec.o
addvec.o: ELF 64-bit LSB relocatable, x86-64, version 1 (SYSV), with debug_info, not stripped

# 创建静态库libvector.a
$ ar rcs libvector.a addvec.o multvec.o

$ gcc -g -Og -c main-lib.c

# 默认生成动态链接代码
$ gcc main-lib.o libvector.a -o main-lib

# 观察到这里是LSB shared object(动态链接文件)，而非期待的静态链接文件
$ file main-lib
main-lib: ELF 64-bit LSB shared object, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=0716f539226a93ecde0838d807f5a53478268109, for GNU/Linux 3.2.0, with debug_info, not stripped

# 检查gcc默认配置参数，发现配置--enable-default-pie
$ gcc -v
Using built-in specs.
COLLECT_GCC=gcc
COLLECT_LTO_WRAPPER=/usr/lib/gcc/x86_64-linux-gnu/9/lto-wrapper
OFFLOAD_TARGET_NAMES=nvptx-none:hsa
OFFLOAD_TARGET_DEFAULT=1
Target: x86_64-linux-gnu
Configured with: ../src/configure -v --with-pkgversion='Ubuntu 9.4.0-1ubuntu1~20.04.2' --with-bugurl=file:///usr/share/doc/gcc-9/README.Bugs --enable-languages=c,ada,c++,go,brig,d,fortran,objc,obj-c++,gm2 --prefix=/usr --with-gcc-major-version-only --program-suffix=-9 --program-prefix=x86_64-linux-gnu- --enable-shared --enable-linker-build-id --libexecdir=/usr/lib --without-included-gettext --enable-threads=posix --libdir=/usr/lib --enable-nls --enable-clocale=gnu --enable-libstdcxx-debug --enable-libstdcxx-time=yes --with-default-libstdcxx-abi=new --enable-gnu-unique-object --disable-vtable-verify --enable-plugin --enable-default-pie --with-system-zlib --with-target-system-zlib=auto --enable-objc-gc=auto --enable-multiarch --disable-werror --with-arch-32=i686 --with-abi=m64 --with-multilib-list=m32,m64,mx32 --enable-multilib --with-tune=generic --enable-offload-targets=nvptx-none=/build/gcc-9-9QDOt0/gcc-9-9.4.0/debian/tmp-nvptx/usr,hsa --without-cuda-driver --enable-checking=release --build=x86_64-linux-gnu --host=x86_64-linux-gnu --target=x86_64-linux-gnu
Thread model: posix
gcc version 9.4.0 (Ubuntu 9.4.0-1ubuntu1~20.04.2) 

# 生成静态链接文件
$ gcc -no-pie main-lib.o libvector.a -o main-lib

# 观察main-lib文件的汇编代码格式，addvec和printf的重定位地址
$ objdump -d main-lib
#  -d, --disassemble        Display assembler contents of executable sections
    401158:       e8 2c 00 00 00          callq  401189 <addvec>
    40117a:       e8 c1 fe ff ff          callq  401040 <__printf_chk@plt>

$ gdb main-lib
```

## 4.1.2 进程影像
### GDB观察

- address: This is the starting and ending address of the region in the process's address space
- offset: This is the offset in the file where the mapping begins.
- dev: If the region was mapped from a file, this is the major and minor device number.
- inode: If the region was mapped from a file, this is the file number.

 0x1000 =  2 ^ (4 * 3) = 4096 Byte = 4kB，不满一个页(4kB)也按照一个页来写
 00401000-00402000: Code Segment
 00402000-00403000-00404000: Only-Readable Segment
 00404000-00405000: Readable-Writable Segment

- Ubuntu2004 Execute Result(P180)
```commandline
$ cat /proc/3552/maps
address                offset   dev   inode                              pathname
00400000-00401000 r--p 00000000 08:05 1049265                            /home/darrenzhou/Code/C_Practice/main-lib
00401000-00402000 r-xp 00001000 08:05 1049265                            /home/darrenzhou/Code/C_Practice/main-lib
00402000-00403000 r--p 00002000 08:05 1049265                            /home/darrenzhou/Code/C_Practice/main-lib
00403000-00404000 r--p 00002000 08:05 1049265                            /home/darrenzhou/Code/C_Practice/main-lib
00404000-00405000 rw-p 00003000 08:05 1049265                            /home/darrenzhou/Code/C_Practice/main-lib
00405000-00426000 rw-p 00000000 00:00 0                                  [heap]

```



### 4.1.3 ELF文件与装入

ELF File Header描述文件整体信息：如文件类型、硬件平台等信息,并用指针指出程序头表和节头表所在的位置.

- Executable File File Header
```
$ readelf -h main-lib
ELF Header:
  Magic:   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00 
  Class:                             ELF64
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0

  # 可执行文件
  Type:                              EXEC (Executable file)

  # 不同系统上的ELF格式文件并不意味着二进制兼容,例如,x86FreeBSD上的ELF可执行文件并不能直接在运行虽然它们格式相同但存在系统调用机制和语义上的差异问题
  Machine:                           Advanced Micro Devices X86-64
  Version:                           0x1

  # 指向代码<_start>
  Entry point address:               0x401050
  Start of program headers:          64 (bytes into file)
  Start of section headers:          18160 (bytes into file)
  Flags:                             0x0
  Size of this header:               64 (bytes)
  Size of program headers:           56 (bytes)
  Number of program headers:         13
  Size of section headers:           64 (bytes)
  Number of section headers:         38
  Section header string table index: 37
```

- Object File File Header
关注section header

```
$ readelf -h main-lib.o
ELF Header:
  Magic:   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00 
  Class:                             ELF64
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              REL (Relocatable file)
  Machine:                           Advanced Micro Devices X86-64
  Version:                           0x1

  # invalid：入口地址为0，不是可执行文件
  Entry point address:               0x0

  # program header is invalid due to Relocatable
  Start of program headers:          0 (bytes into file)
  Start of section headers:          6136 (bytes into file)
  Flags:                             0x0
  Size of this header:               64 (bytes)

  # program header is invalid due to Relocatable
  Size of program headers:           0 (bytes)
  Number of program headers:         0

  Size of section headers:           64 (bytes)
  Number of section headers:         26
  Section header string table index: 25
```

- Program Header

```
$ readelf -l main-lib

Elf file type is EXEC (Executable file)
Entry point 0x401050
There are 13 program headers, starting at offset 64

Program Headers:
  Type           Offset             VirtAddr           PhysAddr
                 FileSiz            MemSiz              Flags  Align
  PHDR           0x0000000000000040 0x0000000000400040 0x0000000000400040
                 0x00000000000002d8 0x00000000000002d8  R      0x8
  INTERP         0x0000000000000318 0x0000000000400318 0x0000000000400318
                 0x000000000000001c 0x000000000000001c  R      0x1
      [Requesting program interpreter: /lib64/ld-linux-x86-64.so.2]
  LOAD           0x0000000000000000 0x0000000000400000 0x0000000000400000
                 0x00000000000004f8 0x00000000000004f8  R      0x1000

  # 对应03: 
  # .init contains start_up code
  # .text contains program code
  # .plt: This is used when calling functions whose addresses can not be known at linking time
  # .fini: This section holds executable instructions that contribute to the process termination code.
  LOAD           0x0000000000001000 0x0000000000401000 0x0000000000401000
                 0x0000000000000235 0x0000000000000235  R E    0x1000

  # 对应于04：
  # rodata: read-only Data 常量不一定放在rodata中；且是多个进程共享
  # eh_frame_hdr: access eh_frame
  # eh_frame: contains exception unwinding and source language information.
  LOAD           0x0000000000002000 0x0000000000402000 0x0000000000402000
                 0x0000000000000170 0x0000000000000170  R      0x1000

  # 对应05:
  # .init_array: This section holds an array of function pointers that contributes to a single initialization array for the executable or shared object containing the section.
  # .fini_array: This section holds an array of function pointers that contributes to a single termination array for the executable or shared object containing the section.
  # .dynamic: This section holds dynamic linking information.
  # .got: 
  # .data: This section holds initialized data that contribute to the program's memory image.
  # .bss: This section holds data that contributes to the program's memory image. The program may treat this data as uninitialized. However, the system shall initialize this data with zeroes when the program begins to run. The section occupies no file space, as indicated by the section type, SHT_NOBITS
  LOAD           0x0000000000002e10 0x0000000000403e10 0x0000000000403e10
                 0x0000000000000230 0x0000000000000240  RW     0x1000
  DYNAMIC        0x0000000000002e20 0x0000000000403e20 0x0000000000403e20
                 0x00000000000001d0 0x00000000000001d0  RW     0x8
  NOTE           0x0000000000000338 0x0000000000400338 0x0000000000400338
                 0x0000000000000020 0x0000000000000020  R      0x8
  NOTE           0x0000000000000358 0x0000000000400358 0x0000000000400358
                 0x0000000000000044 0x0000000000000044  R      0x4
  GNU_PROPERTY   0x0000000000000338 0x0000000000400338 0x0000000000400338
                 0x0000000000000020 0x0000000000000020  R      0x8
  GNU_EH_FRAME   0x0000000000002014 0x0000000000402014 0x0000000000402014
                 0x000000000000004c 0x000000000000004c  R      0x4
  GNU_STACK      0x0000000000000000 0x0000000000000000 0x0000000000000000
                 0x0000000000000000 0x0000000000000000  RW     0x10
  GNU_RELRO      0x0000000000002e10 0x0000000000403e10 0x0000000000403e10
                 0x00000000000001f0 0x00000000000001f0  R      0x1

 Section to Segment mapping:
  Segment Sections...
   00     
   01     .interp 
   02     .interp .note.gnu.property .note.gnu.build-id .note.ABI-tag .gnu.hash .dynsym .dynstr .gnu.version .gnu.version_r .rela.dyn .rela.plt 
   03     .init .plt .plt.sec .text .fini 
   04     .rodata .eh_frame_hdr .eh_frame 
   05     .init_array .fini_array .dynamic .got .got.plt .data .bss 
   06     .dynamic 
   07     .note.gnu.property 
   08     .note.gnu.build-id .note.ABI-tag 
   09     .note.gnu.property 
   10     .eh_frame_hdr 
   11     
   12     .init_array .fini_array .dynamic .got 
```
#### 内存分析
![段节映射关系_与实际不符合](images\段节映射关系.PNG)

##### Program Header Analysis
- File Header Size: 64 Bytes = 0x40 Bytes
- Program Header Size: 56(Size of Program Headers) * 13(Number of Program Headers) = 728 Bytes = 0x2D8 Bytes
- 03 Segment: [READ & EXEC: Code] 从文件main-lib中Offset = 0x0000000000001000 开始的 FileSize = 0x0000000000000235 注入到虚存空间VirtAddr = 0x0000000000401000 开始的 MemSize = 0x0000000000000235的内存空间，并按照0x1000进行对齐。
- 04 Segment: [READ Data] 部分Read-only Data存放于04Segment(00402000-00403000)
- 05 Segment: [READ & WRITE Data] 从文件main-lib中Offset = 0x0000000000002e10 开始的 FileSize = 0x0000000000000230 注入到虚存空间VirtAddr = 0x0000000000403e10开始的MemSize = 0x0000000000000240中，注意文件中FileSize小于虚存空间的MemSize. 由于.bss变量不需要初始化，只需要占据虚存空间而不需要占据文件空间。
- 12 Segment: [动态重定位] 和05 Segment分割成Read-only Data(00403e10 - 00404000)和Read-Write(00404000 - 00405000)两部分；

查看具体section对应的内存空间：
```
$ readelf -S main-lib
here are 38 section headers, starting at offset 0x46f0:

Section Headers:
  [Nr] Name              Type             Address           Offset
       Size              EntSize          Flags  Link  Info  Align
  [ 0]                   NULL             0000000000000000  00000000
       0000000000000000  0000000000000000           0     0     0
  [ 1] .interp           PROGBITS         0000000000400318  00000318
       000000000000001c  0000000000000000   A       0     0     1
  [ 2] .note.gnu.propert NOTE             0000000000400338  00000338
       0000000000000020  0000000000000000   A       0     0     8
  [ 3] .note.gnu.build-i NOTE             0000000000400358  00000358
       0000000000000024  0000000000000000   A       0     0     4
  [ 4] .note.ABI-tag     NOTE             000000000040037c  0000037c
       0000000000000020  0000000000000000   A       0     0     4
  [ 5] .gnu.hash         GNU_HASH         00000000004003a0  000003a0
       000000000000001c  0000000000000000   A       6     0     8
  [ 6] .dynsym           DYNSYM           00000000004003c0  000003c0
       0000000000000060  0000000000000018   A       7     1     8
  [ 7] .dynstr           STRTAB           0000000000400420  00000420
       0000000000000051  0000000000000000   A       0     0     1
  [ 8] .gnu.version      VERSYM           0000000000400472  00000472
       0000000000000008  0000000000000002   A       6     0     2
  [ 9] .gnu.version_r    VERNEED          0000000000400480  00000480
       0000000000000030  0000000000000000   A       7     1     8
  [10] .rela.dyn         RELA             00000000004004b0  000004b0
       0000000000000030  0000000000000018   A       6     0     8
  [11] .rela.plt         RELA             00000000004004e0  000004e0
       0000000000000018  0000000000000018  AI       6    24     8
  [12] .init             PROGBITS         0000000000401000  00001000
       000000000000001b  0000000000000000  AX       0     0     4
  [13] .plt              PROGBITS         0000000000401020  00001020
       0000000000000020  0000000000000010  AX       0     0     16
  [14] .plt.sec          PROGBITS         0000000000401040  00001040
       0000000000000010  0000000000000010  AX       0     0     16
  [15] .text             PROGBITS         0000000000401050  00001050
       00000000000001d5  0000000000000000  AX       0     0     16
  [16] .fini             PROGBITS         0000000000401228  00001228
       000000000000000d  0000000000000000  AX       0     0     4
  [17] .rodata           PROGBITS         0000000000402000  00002000
       0000000000000011  0000000000000000   A       0     0     4
  [18] .eh_frame_hdr     PROGBITS         0000000000402014  00002014
       000000000000004c  0000000000000000   A       0     0     4
  [19] .eh_frame         PROGBITS         0000000000402060  00002060
       0000000000000110  0000000000000000   A       0     0     8
  [20] .init_array       INIT_ARRAY       0000000000403e10  00002e10
       0000000000000008  0000000000000008  WA       0     0     8
  [21] .fini_array       FINI_ARRAY       0000000000403e18  00002e18
       0000000000000008  0000000000000008  WA       0     0     8
  [22] .dynamic          DYNAMIC          0000000000403e20  00002e20
       00000000000001d0  0000000000000010  WA       7     0     8
  [24] .got.plt          PROGBITS         0000000000404000  00003000
       0000000000000020  0000000000000008  WA       0     0     8
  [25] .data             PROGBITS         0000000000404020  00003020
       0000000000000020  0000000000000000  WA       0     0     8
  [26] .bss              NOBITS           0000000000404040  00003040
       0000000000000010  0000000000000000  WA       0     0     8
  [27] .comment          PROGBITS         0000000000000000  00003040
       000000000000002b  0000000000000001  MS       0     0     1
  [28] .debug_aranges    PROGBITS         0000000000000000  0000306b
       0000000000000060  0000000000000000           0     0     1
  [29] .debug_info       PROGBITS         0000000000000000  000030cb
       00000000000004b3  0000000000000000           0     0     1
  [30] .debug_abbrev     PROGBITS         0000000000000000  0000357e
       00000000000001cc  0000000000000000           0     0     1
  [31] .debug_line       PROGBITS         0000000000000000  0000374a
       00000000000001dd  0000000000000000           0     0     1
  [32] .debug_str        PROGBITS         0000000000000000  00003927
       00000000000002d3  0000000000000001  MS       0     0     1
  [33] .debug_loc        PROGBITS         0000000000000000  00003bfa
       0000000000000069  0000000000000000           0     0     1
  [34] .debug_ranges     PROGBITS         0000000000000000  00003c63
       0000000000000020  0000000000000000           0     0     1
  [35] .symtab           SYMTAB           0000000000000000  00003c88
       0000000000000708  0000000000000018          36    53     8
  [36] .strtab           STRTAB           0000000000000000  00004390
       00000000000001e6  0000000000000000           0     0     1
  [37] .shstrtab         STRTAB           0000000000000000  00004576
       0000000000000178  0000000000000000           0     0     1
Key to Flags:
  W (write), A (alloc), X (execute), M (merge), S (strings), I (info),
  L (link order), O (extra OS processing required), G (group), T (TLS),
  C (compressed), x (unknown), o (OS specific), E (exclude),
  l (large), p (processor specific)
```




