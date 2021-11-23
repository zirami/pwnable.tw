# dubblesort

# Reverse

Xem các thông số của file: kiến trúc 32-bit, dynamic và stripped
![file](https://github.com/zirami/pwnable.tw/blob/main/dubblesort/images/file-dubblesort.png)
Xem một số cơ chế bảo mật của file: hầu như các cơ chế đều được dùng.
![checksec](https://github.com/zirami/pwnable.tw/blob/main/dubblesort/images/checksec.png)

Dùng IDA để xem pseudo code của file.
Hàm main của chương trình gọi 1 số hàm khác như setup_(đã được rename), sub_931, và sub_BA0.
![main](https://github.com/zirami/pwnable.tw/blob/main/dubblesort/images/main.png)

Hàm setup_() đóng vai trò thiết lập signal(14..) và alarm(0x3c) để giới hạn thời gian thực hiện chương trình.
![setup_](https://github.com/zirami/pwnable.tw/blob/main/dubblesort/images/setup_.png)

Hàm sub_931(a1,a2), đóng vai trò là hàm sort tăng dần, các giá trị sẽ so sánh đôi một với nhau và swap giá trị lớn ra sau cùng.
![sub_931](https://github.com/zirami/pwnable.tw/blob/main/dubblesort/images/sub_931.png)

Hàm sub_BA0() khi debug thì nó sẽ gọi **stack_smashing_detected** khi canary chương trình bị thay đổi.
![sub_BA0](https://github.com/zirami/pwnable.tw/blob/main/dubblesort/images/sub_BA0.png)

Chương trình sẽ thực hiện nhập tên vào biến buf với độ dài là 0x40, sau đó in ra chuỗi "Hello %s,How many numbers do you what to sort :" với %s là chuỗi vừa nhập cho đến khi gặp byte \x00. Và tiếp tục nhập số lượng các số cần sort vào biến v8 với tham số %u. Nếu nhập số âm thì thì nó sẽ lấy 0xffffffff - số vừa nhập.
Sau đó thực hiện nhập từng số theo số lượng đã nhập ban đầu vào biến v4 (v4 = v9), và tăng v4 += 4.
Tiếp theo gọi hàm sub_931 để sort các số vừa nhập theo thứ tự từ bé đến lớn. Và in kết quả ra, cuối cùng kiểm tra canary, nếu ko bằng thì gọi hàm sub_BA0 để báo lỗi **stack smashing detected**.


# Exploit

Bài này có canary với PIE, ban đầu ý tưởng mình leak địa chỉ trên bss, tính địa chỉ hàm main để leak được địa chỉ trên libc, thực hiện ret2libc để lên shell.
![ret2libc_fail_ebx](https://github.com/zirami/pwnable.tw/blob/main/dubblesort/images/ret2libc_fail_ebx.png)

Nhưng có một số vấn đề với ebx khi nó kết thúc hàm main nó sẽ pop ebx ra khác với tham số lúc đầu. Và không thực hiện ret2libc thành công.
![debug_fail](https://github.com/zirami/pwnable.tw/blob/main/dubblesort/images/debug_solution1.png)

Mình đã thử tinh địa chỉ của ebx lúc chưa pop, và đưa vào vùng stack tương ứng nhưng không thành công do địa chỉ của vùng này lớn hơn hàm main, và puts_plt và các giá trị đưa vào được hàm sub_931 sort lại từ bé đến lớn. Nên các này mình đã dừng lại tại đây.
![main_1](https://github.com/zirami/pwnable.tw/blob/main/dubblesort/images/main_1.png)
Sau đó mình nhận ra có thể leak thẳng địa chỉ trên libc để tính base, hướng này khắc phục một số khuyết điểm của cách leak địa chỉ bss, không cần quan tâm đến hàm sub_931 dùng để sort các tham số đầu vào.

Địa chỉ ở libc này có kết thúc là 0x...000 mà lệnh
` __printf_chk(1, "Hello %s,How many numbers do you what to sort :");`
in chuỗi nhập vào đến khi gặp \x00, vì vậy, nhập tràn qua \x00 với \x0a, sao đó thay thế \x0a bằng \x00. 
```sh
s.sendline("A"*24)
s.recvuntil("\n")
leak = u32("\x00"+s.recv(3))
```
Sau đó tính libc base bằng cách trừ địa chỉ vừa leak cho Addr của libc với phân vùng got.plt xem như là offset từ libc base đến địa chỉ này.
```sh
Section Headers:
  [Nr] Name              Type            Addr     Off    Size   ES Flg Lk Inf Al
  ...
  [31] .got.plt          PROGBITS        001b0000 1af000 000030 04  WA  0   0  4
```

Sau đó thực hiện ret về system("/bin/sh") với libc base vừa tính được.
Vấn đề cuối là vùng canary, chưa biết canary là gì với không thể brute được. Nhưng rất may do chương trình sử dụng
`__isoc99_scanf("%u", v4);` để nhập giá trị đầu vào cho v4. mà đặc tả %u thường được sử dụng cho địa chỉ và số nguyên. nên khi nhập giá trị khác, ví dụ '+' thì return value sẽ không được match 
![return_value](https://github.com/zirami/pwnable.tw/blob/main/dubblesort/images/return_value.png)
![poc_scanf1](https://github.com/zirami/pwnable.tw/blob/main/dubblesort/images/poc_scanf1.png)
Sau khi nhập '+' thì 0xffffcc9c trong EDI vẫn không thay đổi.
![poc_scanf2](https://github.com/zirami/pwnable.tw/blob/main/dubblesort/images/poc_scanf2.png)
Vì vậy ngay tại chỗ canary, mình sẽ nhập '+' thì canary vẫn được giữ nguyên.

# FLAG{Dubo_duBo_dub0_s0rttttttt}