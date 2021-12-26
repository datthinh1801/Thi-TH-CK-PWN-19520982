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
Đầu tiên là `check2`. Ở `func1`, để `check2 = 1`, ta sẽ truyền chuỗi `I\'m sorry, don\'t leave me. I want you here with me ~~` vào vị trí nào đó trong buffer để biến `local_58` trong hàm `func1` sẽ bằng với chuỗi trên. Lúc này, ta sẽ có `check2 == 1` vì `strcmp` sẽ trả về `0` nếu 2 chuỗi bằng nhau. Vậy nếu ta gọi lại `func1` lần nữa, `check2` lúc này sẽ có giá trị là `1`. Tiếp theo, ta cần truyền giá trị `0x20010508` vào vị trí nào đó trên buffer để khi hàm truy xuất đến `param_1` sẽ lấy được giá trị `0x20010508` và làm thoả mãn điều kiện.  

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

#### Ghi các giá trị vào buffer để thoả mãn các điều kiện trong `func1`
Đầu tiên, ta sẽ chèn chuỗi `I\'m sorry, don\'t leave me. I want you here with me ~~` vào buffer. Khi `disass` hàm `func1`, ta thấy được rằng `local_58` nằm tại `ebp - 0x54` (vì `local_58` là đối số thứ 1 của hàm `strcmp` nên sẽ được push vào stack sau).  

![image](https://user-images.githubusercontent.com/44528004/147410799-3ed87810-c625-42f8-9849-14dfd62710ec.png)  

Đồng thời, ta cũng biết được là `param_1` sẽ nằm tại vị trí `ebp + 0x8`.  

![image](https://user-images.githubusercontent.com/44528004/147410911-5fffc63b-078c-4196-a8d6-8d1a4b83410c.png)


Ta debug, xem các địa chỉ và lập được bảng sau:

| Đối tượng | Địa chỉ trên stack | Khoảng cách so với buffer (byte) |
|---|---|---|
| `ebp` của hàm `func1` | `0xffffd07c` | - |
| `local_58` | `ebp - 0x54 = 0xffffd07c - 0x54 = 0xffffd028` | 0x4 |
| `param_1` | `ebp + 0x8 = 0xffffd084` | 0x60 |
| `return address` của hàm `func1` | `0xffffd080` | 0x5c |

Vậy payload của ta lúc này sẽ là:  
```python
e = ELF('./stack_architect')
func1_addr = e.symbols['func1']
func2_addr = e.symbols['func2']

payload = b'A' * 0x4 + b'I\'m sorry, don\'t leave me, I want you here with me ~~\x00'
payload += (0x58 - len(payload)) * b'A' + p32(func1_addr) + p32(func1_addr)
```
