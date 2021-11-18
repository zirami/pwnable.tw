# 3x17
```sh
*/
cre: https://0xfeebe.medium.com/3x17-pwnable-tw-7713f7d41ffa
/*
```
# Reverse

Xem các thông số file 3x17

```sh
thanhnt@THANGNT-ZIR:/mnt/c/Users/n18dc/OneDrive/Desktop$ file 3x17
3x17: ELF 64-bit LSB executable, x86-64, version 1 (GNU/Linux), statically linked, for GNU/Linux 3.2.0, BuildID[sha1]=a9f43736cc372b3d1682efa57f19a4d5c70e41d3, stripped
```

Kiểm tra một số cơ chế bảo mật

```sh
[*] '/mnt/c/Users/n18dc/OneDrive/Desktop/3x17'
    Arch:     amd64-64-little
    RELRO:    Partial RELRO
    Stack:    No canary found
    NX:       NX enabled
    PIE:      No PIE (0x400000)
```

Dùng IDA xem pseudo code của file 3x17, do file được stripped nên các tên hàm bên dưới đã được mình modify sau khi reverse và fuzzing bằng 1 chương trình c tương tự với các hàm tương ứng để xác nhận các hàm này.


```sh
__int64 main_func()
{
  __int64 result; // rax
  char *v1; // [rsp+8h] [rbp-28h]
  char buf[24]; // [rsp+10h] [rbp-20h] BYREF
  unsigned __int64 v3; // [rsp+28h] [rbp-8h]

  v3 = __readfsqword(0x28u);
  result = (unsigned __int8)++byte_4B9330;
  if ( byte_4B9330 == 1 )
  {
    write_func(1u, "addr:", 5uLL);
    read_func(0, buf, 0x18uLL);
    v1 = (char *)(int)atoi_func((__int64)buf);
    write_func(1u, "data:", 5uLL);
    read_func(0, v1, 0x18uLL);
    result = 0LL;
  }
  if ( __readfsqword(0x28u) != v3 )
    stack_smashing_detected();
  return result;
}
```

Chương trình cho phép nhập địa chỉ và giá trị gán cho địa chỉ này. Hàm atoi_func để chuyển chuỗi thành số nguyên được ép kiểu thành địa chỉ và sau đó gán cho v1. Sau đó nhập vào v1 0x18 byte. 
Bên cạnh đó, file được build với statically, nên phần đè mục GOT sẽ không khả thi, do khi build thì chương trình sẽ tham chiếu từ thư viện và tạo ra một bản sao, lưu trữ theo file lúc build, phương pháp build static sẽ giúp chương trình thực thi nhanh hơn là dynamic vì không cần tham chiếu vào thư viện mỗi lần thực thi, nhưng hạn chế là sẽ tốn bộ nhớ hơn các dynamic.

# Exploit

Nếu chương trình build static thì vấn đề Write-What-Where vẫn chưa được xử lý! Vậy phải giải quyết như thế nào.
Để giải quyết vấn đề này, cần hiểu thêm 1 chút về file ELF, là cách hoạt động của file, mọi người có thể tìm đọc và hiểu tại đây.
```sh
http://blog.k3170makan.com/2018/10/introduction-to-elf-format-part-v.html
```

Nói một cách đơn giản, thì khi chương trình bắt đầu, hàm _start sẽ được thực thi trước chứ không phải hàm main và nó gọi hàm __libc_start_main() với các tham số là hàm main và các hàm khác, trong đó có 2 phần là `init` và `fini`.

Dưới đây là hàm start của file 3x17 đã được stripped
```sh
void __fastcall __noreturn start(__int64 a1, __int64 a2, __int64 a3)
{
  __int64 v3; // rax
  unsigned int v4; // esi
  __int64 v5; // [rsp-8h] [rbp-8h] BYREF
  void *retaddr; // [rsp+0h] [rbp+0h] BYREF

  v4 = v5;
  v5 = v3;
  sub_401EB0(
    (__int64 (__fastcall *)(_QWORD, __int64, __int64))main_func,
    v4,
    (__int64)&retaddr,
    (void (__fastcall *)(_QWORD, __int64, __int64))sub_4028D0,
    (__int64)sub_402960,
    a3,
    (__int64)&v5);
}
```
Đây là hàm start của file test_no_stripped được mình tạo 
```sh
void __fastcall __noreturn start(__int64 a1, __int64 a2, void (*a3)(void))
{
  __int64 v3; // rax
  int v4; // esi
  __int64 v5; // [rsp-8h] [rbp-8h] BYREF
  char *retaddr; // [rsp+0h] [rbp+0h] BYREF

  v4 = v5;
  v5 = v3;
  _libc_start_main(
    (int (__fastcall *)(int, char **, char **))main,
    v4,
    &retaddr,
    _libc_csu_init,
    _libc_csu_fini,
    a3,
    &v5);
  __halt();
}
```

Các tham số sẽ được định nghĩa như sau, theo link phía trên
![libc_start_main](https://github.com/zirami/pwnable.tw/blob/main/3x17/images/__libc_start_main.png)

Và chương trình sau khi kết thúc hàm main, thì mục `fini` được thực thi, cho nên mục tiêu chúng ta sẽ nhắm để để ghi là mục `fini` này với địa chỉ của main để tạo 1 vòng lặp vô hạn.

Sau đó thực hiện ROP 64-bit đơn giản, vi static nên gadget sẽ rất dễ tìm.
Một số gadget
```sh
POP_RDI = 0x401696
POP_RSI =0x406c30
POP_RAX = 0x41e4af
POP_RDX = 0x446e35
SYSCALL = 0x4022b4
RET = 0x401016
LEAVE_RET = 0x401c4b
```
Và sau khi thực hiện ghi đè xong, thì phải giải quyết vấn đề vòng lặp để có thể Ret được. Dùng gadget LEAVE_RET phía trên để giải quyết vấn đề cuối này và get shell.
# FLAG{Its_just_a_b4by_c4ll_0riented_Pr0gramm1ng_in_3xit}