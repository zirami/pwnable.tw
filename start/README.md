# start

# REVERSE

Sử dụng IDA để xem pseudo code
```sh
public _start
_start proc near
push    esp
push    offset _exit
xor     eax, eax
xor     ebx, ebx
xor     ecx, ecx
xor     edx, edx
push    3A465443h
push    20656874h
push    20747261h
push    74732073h
push    2774654Ch
mov     ecx, esp        ; addr
mov     dl, 14h         ; len
mov     bl, 1           ; fd
mov     al, 4
int     80h             ; LINUX - sys_write
xor     ebx, ebx
mov     dl, 3Ch ; '<'
mov     al, 3
int     80h             ; LINUX -
add     esp, 14h
retn
```

Chương trình sẽ thực hiện xor cho 4 thanh ghi: eax,ebx,ecx,edx lần lượt bằng 0, sau đó push chuỗi `Let's start the CTF:` vào stack, in ra màn hình, và nhập vào stack với len = 0x3C.

# EXPLOIT

Chương trình có push esp vào stack trước khi push đoạn text vào stack, cho nên trên stack sẽ có địa chỉ của esp lúc đầu và đó cũng là địa chỉ nằm trên stack.
Khi kết thúc chương trình add esp, 14h, để thực thi _exit.
Lệnh ngắt 0x80 read đọc với len = 0x3c, nên hướng giải quyết sẽ là ret về chỗ mov ecx,esp, để leak địa chỉ stack. tính toán và nhảy về shellcode.
# FLAG{Pwn4bl3_tW_1s_y0ur_st4rt}
