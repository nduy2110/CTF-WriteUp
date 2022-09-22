# Phar Deserialization
## 1. Stream trong PHP
Stream lần đầu được được giới thiệu ở bản PHP 4.3.0 như một cách để khái quát hóa file, network, data compression và các tiến trình khác,... những thứ mà dùng chung một tập hợp functions hoặc cách sử dụng. [Tham khảo](https://www.php.net/manual/en/intro.stream.php)\
Stream cung cấp cho ta quyền truy cập dữ liệu theo yêu cầu. Nghĩa là ta sẽ không cần download toàn bộ nội dung file vào bộ nhớ trước khi quá trình xử lý bắt đầu. Stream sẽ đọc file theo các gói dưữ liệu (chunks) và đọc theo 1 cách tuyến tính. Điều này cho phép ta tương tác với các file lớn một cách hiệu quả\
Mỗi stream đều sẽ có một ``implementation wrapper``, các wrapper này cung cấp code bổ sung cần thiết để xử lý các giao thức đặc biệt hoặc phục vụ việc encode. PHP cung cấp cho ta một số wrapper dựng sẳn như:
```php
file://
http://
ftp://
php://
zlib://
data://
glob://
phar://
```
Ngoài ra ta cũng có thể khai báo wrapper cho riêng mình\
Trong bài này ta tập trung vào ``phar://`` để khai thác lổ hỏng Phar Deserialization. Có thể hình dung ``phar://`` dùng để truy cập file Phar và khi truy cập, metadata của Phar file sẽ được unserialized.
## 2. Deserialization vulnerability
Serialization là quá trình chuyển đổi dữ liệu của object thành các stream bytes , hay dễ hiểu hơn là quá trình chuyển dữ liệu thành các chuỗi bits tuần tự để lưu trữ hoặc truyền đi qua mạng. Còn deserialization là quá trình ngược lại/
Ví dụ, ta có đoạn object như sau:
```php
class Student {
        private $name;
        private $age;

        public function __construct($name, $age) {
            $this->name = $name;
            $this->age = $age;
        }
    };

    $nhatduy = new Student('endy', 19);
```
Thự hiện serialize bằng hàm ``serialize()`` cho ta kết quả:
```php
O:7:"Student":2:{s:13:"Studentname";s:5:"enduy";s:12:"Studentage";i:19;}
```
Lỗ hổng Deserialization trong PHP hay còn gọi là PHP Object Injection, lợi dụng cơ chế deserialize của ứng dụng từ đó attakcer sẽ inject một object chứa payload và khi object được deserialize thì payload sẽ được trigger, có thể giúp attacker thực hiện nhiều loại tấn công khác nhau như:  Code Injection, SQL Injection, DoS,… tùy vào từng trường hợp cụ thể\
Để thực hiện khai thác thành công lỗ hổng này trên nền tảng PHP ta cần 2 điều kiện sau:
1. Đối tượng cần tấn công phải có class sử dụng ``Magic method``
2. Tìm được ``POP chain``, hay chính là có thể tùy chỉnh được các đoạn code trong quá trình hàm unserialize() được gọi

## 3. Magic method
Magic method là các function đặc biệt trong các class của PHP, tên của các function này có hai dấu gạch dưới đứng trước, nó sẽ được gọi ngầm ở một sự kiện cụ thể, ví dụ như: __sleep(), __toString(), __construc(), …. Phần lớn trong số các function này sẽ không làm gì nếu không có sự khai báo, thiết lập của người lập trình. Ở đây có hai Magic method có thể trigger được lỗi Phar Deserialization mà ta cần quan tâm đến là:
- ``__wakeup()``:  Được gọi khi một object được deserialize
- ``_destruct()``: Được gọi khi một kịch bản PHP kết thúc hoặc một object không còn được dùng trong code nữa và bị hủy bỏ\
Ví dụ, ta có đoạn code sau:
```php
class Demo {
        public function __destruct() {
            echo "Đang gọi tới destruct method\n";
        }

        public function __wakeup() {
            echo "Đang gọi tới wakeup method\n";
        }
    }

    $demo = new Demo;
    $serial = serialize($demo);
    $unserial = unserialize($serial);
    echo "-------------*----------\n";
```
Output:
```bash
Đang gọi tới wakeup method
-------------*----------
Đang gọi tới destruct method
Đang gọi tới destruct method
```

## 4. POP chain
POP (Property Oriented Programming) nghĩa đen là lập trình hướng thuộc tính, và cái tên này xuất phát từ thực tế, việc kẻ tấn công có thể kiểm soát tất cả các thuộc tính của deserialized object.\
POP chain hay chuỗi các POP hoạt động bằng cách nối các ``gadgets`` lại với nhau để thực hiện một mục đích nào đó của attacker
> "Gadgets": là những đoạn code mượn từ codebase của ứng dụng, được attacker lợi dụng để viết payloads
Ví dụ POP chain:
```php
    class Render {
        protected $file;

        //some code

        public function __construct($file) {
            $this->$file = $file 
        }

        public function __wakeup() {
            $this->$file->open()
        }
    }

    class File {

        //some code

        public function open() {
            fuction_to_open_file() 
        }
    }
```
Ở ví dụ này là đoạn code có chức năng mở file một khi obj Render được deserialized, object ``Render`` sẽ nhận tham số là một object ``File`` và khi ``Render`` được deserialized thì hàm ``__wakeup()`` được gọi, sau đó hàm ``__wakeup()`` gọi tới hàm ``open()`` của ``File``.\
Ta có thể lợi dụng logic này, inject bất kỳ thứ gì ta muốn vào hàm ``open()`` của object ``File``, sau đó serialize nó và gửi cho ứng dụng. Khi ứng dụng deserialize object chứa payload thì đoạn code ta inject vào sẽ được thực thi.

Nói tóm lại thì POP chain sẽ sử dụng magic methods ban đầu để gọi tới một gadget. Gadget này có thể gọi tới một gadget khác. Và ta sẽ thực hiện inject payload vào gadget được gọi tới cuối cùng

## 5. Phar Deserialization
Phar là một file trong PHP, nó tương tự như jar file trong java là một package format cho phép ta gói nhiều các tập code, các thư viện, hình ảnh,… vào một tệp\
Cấu trúc một Phar file gồm có:
- Stub: đơn giản chỉ là một file PHP và ít nhất phải chứa đoạn code sau: <?php __HALT_COMPILER();
- A manifest (bảng kê khai): miêu tả khái quát nội dung sẽ có trong file
- Nội dung chính của file
- Chữ ký: để kiểm tra tính toàn vẹn (cái này là optional, có hay không cũng được)

Điểm ta cần quan tâm là mainfest, là nơi chứa metadata của phar. Nó bao gồm thông tin về archive và từng file bên trong phar. Quan trọng nhất là metadata này được lưu dưới định dạng serialize.

![This is an image](./img//metadata.png)

Và khi một filesystem function gọi đến một phar file thì tất cả các metadata trên sẽ được tự động unserialize. Đây chính là logic ta có thể lợi dụng để thực hiện Phar Deserialization

## Nguồn tham khảo:
- [https://sec.vnpt.vn/2019/08/ky-thuat-khai-thac-lo-hong-phar-deserialization/](https://sec.vnpt.vn/2019/08/ky-thuat-khai-thac-lo-hong-phar-deserialization/)
- [https://i.blackhat.com/us-18/Thu-August-9/us-18-Thomas-Its-A-PHP-Unserialization-Vulnerability-Jim-But-Not-As-We-Know-It-wp.pdf](https://i.blackhat.com/us-18/Thu-August-9/us-18-Thomas-Its-A-PHP-Unserialization-Vulnerability-Jim-But-Not-As-We-Know-It-wp.pdf)
- [https://stackify.com/a-guide-to-streams-in-php-in-depth-tutorial-with-examples/](https://stackify.com/a-guide-to-streams-in-php-in-depth-tutorial-with-examples/)
- [https://www.php.net/manual/en/intro.stream.php](https://www.php.net/manual/en/intro.stream.php)
- [https://pentest-tools.com/blog/exploit-phar-deserialization-vulnerability](https://pentest-tools.com/blog/exploit-phar-deserialization-vulnerability)
