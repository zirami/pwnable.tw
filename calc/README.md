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
Hàm main setup alarm(60), sau 60s thì chương trình sẽ bị ngắt, sau đó gọi hàm calc.
![main](https://github.com/zirami/pwnable.tw/blob/main/calc/images/main.png)
Hàm calc sẽ là hàm xử lý chính trong chương trình và nằm trong vòng lặp while(1)
![calc](https://github.com/zirami/pwnable.tw/blob/main/calc/images/calc.png)
Trước khi thực hiện gọi hàm get_expr() thì gọi hàm bzero(s,0x400), theo mình hiểu thì nó sẽ re_init lại biến s
![bzero](https://github.com/zirami/pwnable.tw/blob/main/calc/images/bzero.png)
Tiếp theo gọi hàm get_expr(s,1024) để nhận các giá trị input được filter theo các ký tự đã được định nghĩa sẵn như: 0-9, '+', '-', '*', '/', '%'. Các ký tự còn lại sẽ ko được ghi nhận.
![get_expr](https://github.com/zirami/pwnable.tw/blob/main/calc/images/get_expr.png)
Tiếp theo sẽ gọi hàm init_pool(v1) để re_init(v1) với giá trị bằng 0
![init_pool](https://github.com/zirami/pwnable.tw/blob/main/calc/images/init_pool.png)
Tiếp đến là hàm parse_expr() để phân tích các expression. 
* Filter 1 số dùng trong phép tính bằng 0, hoặc 2 expression liền kề.
###  Đoạn s[v6] được hiểu như sau:
* Khi s[v6] = 0, thì lúc này chương trình gặp expression đầu tiên, thì giá trị chưa được tính mà được đưa vào mảng a2[v3+1].
* Khi s[v6] đã có giá trị, thì khi gặp expression tiếp theo là: %,/,* nhưng khác '-', '+' sẽ nhảy đến LABEL_14, nếu bằng thì s[v6] sẽ bằng thì gán s[v6] cho bằng expression hiên tại rồi break khỏi switch-case. Ý nghĩa cho đoạn code này là dùng để thực thi các phép toán theo quy tắc: `Nhân chia trước, cộng trừ sau`.
* Khi thoát khỏi vòng lặp for, nếu v6 >= 0 thì sẽ thực hiện tính toán cho đến khi v6 < 0.

![parse_expr](https://github.com/zirami/pwnable.tw/blob/main/calc/images/parse_expr.png)
![parse_expr2](https://github.com/zirami/pwnable.tw/blob/main/calc/images/parse_expr2.png)

Hàm thực hiện tính toán sẽ là hàm eval() thực hiện `+`, `-`, `*`, `/`, không có thực hiện `%`, và cuối cùng sẽ --*a1 (giảm giá trị được trỏ tới bởi địa chỉ a1 xuống 1 đơn vị.)

![eval](https://github.com/zirami/pwnable.tw/blob/main/calc/images/eval.png)


# Exploit

Để exploit chương trình calc cần phải trả lời 2 câu hỏi là:
* Chương trình calc như trên bị lỗi chỗ nào?
* Khai thác nó như thế nào?

### Chương trình calc như trên bị lỗi chỗ nào?

Trong hàm parse_expr() có 1 lỗi logic, khi chương trình xét theo dấu express, thì nếu họ nhập -1. thì lệnh sau đây không được check vì lúc này v9=0
```sh
if ( v9 > 0 )
      {
        v3 = (*a2)++;
        a2[v3 + 1] = v9;
      }
```
do khi gặp expression đầu tiên thì v7 = i + a1 - v4 (v4=a1,i=0) 
```sh
s1 = malloc(v7 + 1);
memcpy(s1, v4, v7);
s1[v7] = 0;  -- s1[0] = 0
...
v9 = atoi(s1); -- v9 = 0
```
Thì lỗi chỗ này sẽ xảy ra như sau: khi người dùng nhập:
`-1+66` chương trình sẽ lấy 0-1 = -1, và do lệnh if ở trên ko được check, nên khi thực hiện hàm eval() biến *(a2) sẽ giảm xuống 1 đơn vị, và (*a2) lúc này sẽ trỏ đến a2[-1] sẽ nhân được giá trị là 66 do a2[v3+1] = v9 (a2[0]=v9)

thì lúc này chúng ta sẽ lợi dụng a2[v3+1]=v9 để Write-What-Where.

### Khai thác nó như thế nào?

Dùng `-1+offset+value` để ghi value vào offset bắt đầu từ *(a2) để ghi các giá trị. Vậy ghi cái gì và ghi ở đâu.

Do chương trình build static, nên sẽ không có libc, nhưng bù lại, chúng ta có ropchain, dùng ropchain có sẵn bằng lệnh.
`ROPgadget --binary calc --ropchain`

Và cuối cùng là ghi vào nơi ret của hàm main(). Vì chúng ta phải dùng hàm parse_expr() để ghi, nên hàm main() sẽ là lựa chọn tốt.

# FLAG{C:\Windows\System32\calc.exe}