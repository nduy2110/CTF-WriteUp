# Labs File Upload Vulnerability Portswigger

## 1. Giới thiệu
Upload file là một chức năng phổ biến và thông dụng trong các ứng dụng web. Tuy nhiên nếu không có giải pháp validate file upload đúng cách, thì attacker có thể lợi dụng để upload các file shell lên sever để thực hiện bất kỳ đoạn nào.

## 2. Cách sever xử lý request của một file
Đầu tiên sever sẽ check path của request để xác định extension của file. Sau đó sever sẽ so sánh extension với một list mapping giữa extension và MIME type để xác định file thuộc định dạng nào. Tiếp theo tùy từng loại file mà sẽ được xử lý.

- Nếu là file không thể thực thi (non-executable) như file ảnh, file HTML tĩnh thì sever chỉ việc gửi nội dung file cho response
- Nếu là file có thể thực thi (executable), như file PHP, và sever được config để có thể thực thi được loại file này, thì file sẽ được thực thi và output sẽ được trả về cho response
- Nếu là file có thể thực thi nhưng sever không được config để có thể chạy được loại file này, thì trả về error message cho response. Tuy nhiên một vài trường hợp thì nội dung file cũng có thể được trả về cho response dưới dạng plaintext

## Lab1 Portswigger
Ở labs này sẽ cho ta một trang blog và yêu cầu ta đọc nội dung file ``/home/carlos/secret`` của sever, ta có thể đăng nhập bằng account ``wiener:peter``, và có thể thực hiện upload ảnh avatar. Tuy nhiên labs không thực hiện bất kỳ hình thức validate nào nên ta có thể upload bất kỳ file nào kể cả file shell

Ta thử upload file ``shell.php`` với nội dung:
```php
<?php echo file_get_contents('/home/carlos/secret'); ?>
```

Upload shell lên labs và ta thấy vị trí của file shell tại tag img của avatar:

![lab1](./img/lab1.png)

Ta thực hiện truy cập đén ``/files/avatars/shell.php`` và đọc được nội dung của file /home/carlos/secret

![lab1](./img/lab1-result.png)

## Lab2 Portswigger
Ở bài này thì cũng tương tự như lab1 nhưng đã có thêm 1 lớp bảo mật khi nếu upload file shell thì nó sẽ hiển thông báo lỗi:

![lab2](./img/lab2-error.png)

Để bypass ta chỉ cần thay ``Content-Type`` của file shell từ application/octet-stream thành image/png hoặc image/jpeg

![lab2](./img/lab2-burp.png)

Tương tự lab1 chỉ cần truy cập đến ``files/avatars/shell.php`` để trigger file shell

## Lab3 Portswigger
Ở labs này cho ta thông tin là tại thư mục upload file của người dùng đã được config sao cho không thể execute được file shell. Và yêu cầu của lab vẫn là đọc file ``/home/carlos/secret``

Ta có ý tưởng là sẽ lợi dụng path traversal để thoát khỏi thư mục hiện tại, và thực hiện execute code tại thư mục khác mà không bị sever chặn 

Ta thử upload một file shell thông thường thì vẫn upload thành công tuy nhiên ta không thể trigger nó được:

![lab3](./img/lab3-test.png)

Ta có thể thấy sever chỉ xem file shell nằm trong thư mục ``files/avatars`` như một planitext

Ta thử upload một file shell, nhưng thay ``filename`` trong request thành ``../shell.php``

![lab3](./img/lab3-test-pt.png)

Ta có thể thấy ở response thông báo thì file vẫn được lưu ở ``avatars/shell.php`` nên em nghĩ dấu ``/`` đã bị filter. Em thử encode dấu ``/`` thì exploit thành công

![lab3](./img/lab3-encode.png)

Upload thành công ta chỉ cần truy cập ``files/shell.php`` để trigger file shell:

![lab3](./img/lab3-result.png)





