# 3x17

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
![file-elf](http://blog.k3170makan.com/2018/10/introduction-to-elf-format-part-v.html)

# FLAG{Its_just_a_b4by_c4ll_0riented_Pr0gramm1ng_in_3xit}