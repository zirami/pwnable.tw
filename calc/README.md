# calc

# Reverse

Xem các thông số của file, statically linked và not stripped
```sh
thanhnt@THANGNT-ZIR:/mnt/c/Users/n18dc/OneDrive/Desktop$ file calc
calc: ELF 32-bit LSB executable, Intel 80386, version 1 (GNU/Linux), statically linked, for GNU/Linux 2.6.24, BuildID[sha1]=26cd6e85abb708b115d4526bcce2ea6db8a80c64, not stripped
```
Tiếp theo xem phần checksec file, có Canary và Partial RELRO
```sh
pwndbg> checksec
[*] '/mnt/c/Users/n18dc/OneDrive/Desktop/calc'
    Arch:     i386-32-little
    RELRO:    Partial RELRO
    Stack:    Canary found
    NX:       NX enabled
    PIE:      No PIE (0x8048000)
```

Dùng IDA xem pseudo code của file calc


# Exploit