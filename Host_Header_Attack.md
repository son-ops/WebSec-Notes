# HTTP Host Header 

Khi mà nhiều web applications khác nhau được chạy trên cùng một host, để phân biệt giữa các máy chủ ta cần sử dụng `Host Header`.

Các CDNs như Akamai, Cloudflared hay các hệ thống trung gian như reverse proxy, load balance, ... cũng sử dụng host để xác định phục vụ web application nào. 1 ví dụ đơn giản về cấu hình của apache để phục vụ 2 virtual hosts khác nhau:

```text
<VirtualHost *:80>
    DocumentRoot "/var/www/testapp"
    ServerName testapp.xyz
</VirtualHost>

<VirtualHost *:80>
    DocumentRoot "/var/www/anotherapp"
    ServerName anotherapp.xyz
</VitualHost>
```

Ví dụ khi ta bắt đầu truy cập `http://testapp.xyz`. Bắt đầu với DNS, `testapp.xyz` được phân giải ra thành địa chỉ IP `200.191.186.10` (ví dụ IP này là của proxy, server đứng sau proxy) chẳng hạn. Sau khi có được IP, client sẽ thực hiện bắt tay 3 bước, tạo kết nối `TCP` tới IP này với port 80. Sau khi tạo kết nối TCP, client thực hiện gửi request 

```http
GET / HTTP/1.1
Host: testapp.xyz
...
```

Proxy lúc này chưa biết sẽ phải chuyển tiếp nó đến backend nào do có nhiều web server đứng phía sau nó. Lúc này nó sẽ dựa vào `Host Header`. Nó thấy Host là `testapp.xyz` khớp với rule 

```
<VirtualHost *:80>
    DocumentRoot "/var/www/testapp"
    ServerName testapp.xyz
</VirtualHost>
``` 

nên sẽ thực hiện phục vụ với DocumentRoot là `/var/www/testapp`

# Host Header Vulnerability

Lỗ hổng này bắt nguồn từ việc xử lí `Host Header` không an toàn bởi web application (backend). Trong thực tế, lỗ hổng này thường khó để detect và exploit do ta không thể chèn giá trị này một cách tùy ý.  Bởi backend thường được đứng sau CDNs hay proxy, và nó sẽ route tới backend dựa trên `Host Header`. Nếu chèn giá trị này tùy ý, request có thể không tới được backend mà bị drop ngay tại các thành phần trung gian này.

1 số web server hỗ trợ `override host headers` qua 1 số header khác, phổ biến như:

* X-Forwarded-Host
* X-HTTP-Host-Override
* Forwarded
* X-Host
* X-Forwarded-Server

Nếu server hỗ trợ ovveride này, attacker có thể lợi dụng nó để khai thác `Host header attack`. 

Để khai thác được lỗ hổng này, ta cũng cần biết cách mà server xử lí nó như thế nào. Để đoán được điều này ta cần xem xét kĩ response của một normal request xem có dấu hiệu gì không. Sau đó có thể thực hiện thay đổi giá trị trong trường Host hoặc các override header trên (nếu hỗ trợ) và quan sát phản hồi trả về, ta có thể dự đoán phần nào cách mà server xử lí chúng. Từ đó đưa ra hướng phù hợp để khai thác.

Nếu request thường bị drop ngay khi ta thay đổi giá trị Host thì có lẽ CDN hay Proxy đã drop nó ngay ở tầng trung gian rồi, trong trường hợp này gần như ta không thể sử dụng lỗ hổng này để khai thác được.

**Note**: mặc định host header sẽ không thể chứa các kí tự đặc biệt, nên trong các trường hợp khai thác cần sử dụng kí tự đặc biệt ta chỉ có thể chèn nó qua các override header (nếu được hỗ trợ )

## Authentication Bypass

Ví dụ một server, thực hiện authen admin panel theo cách sau:

```php
$headers = getallheaders();
$host_header = $headers['Host'];
if ($host_header === 'localhost') {
echo "Welcome Admin!";
} else {
echo "Unauthorized! The admin area can only be accessed locally!";
die();
}
```

Attacker lúc này chỉ cần set giá trị của trường `Host` thành `localhost` là có thể truy cập admin panel này.

## Password Reset Poisoning

Một trường hợp tiếp theo mà server hay sai lầm khi code đó là tạo `password reset link` dựa trên `Host header`. Ví dụ như đoạn code bên dưới:

```php
function create_reset_link($user, $host_header) {
    $token = generate_reset_token($user);
    return "http://" . $host_header . "/pw_reset.php?token=" . $token;
}
```
Luồng thông thường của `password reset functionality` sẽ là  nhập `username` hay `gmail`, sau đó nó sẽ tạo ra 1 `password reset link` và gửi qua gmail cho user. Khi user clink vào link này nó sẽ được điều hướng tới trang nhập password mới.

Trong trường hợp này ta có thể khai thác bằng cách thực hiện reset password `username=admin` và `Host` trỏ tới 1 domain mà attacker có thể kiểm soát. Sau đó, reset link sẽ được gửi tới mail của admin. Khi admin click vào link này, token sẽ được gửi qua attacker server. Attacker sau đó thực hiện sử dụng token này để reset password.

## Combining with Web Cache Poisoning

Một trường hợp khác, ví dụ 1 server hỗ trợ `X-Forwarded-Host` và giá trị này sau đó được sử dụng để tạo absolute link cho các thẻ html. Ví dụ:

```
<img src='<X-Forwarded-Host value>/hihi.png'>
```

Nếu server không lọc đầu vào cũng như escape đầu ra cẩn thận rất có thể sẽ dẫn tới XSS, ta có thể thử chèn 

```
X-Forwarded-Host: '><script>alert(1)</script>
```

Tuy nhiên, kể cả có XSS trong trường hợp này ta vẫn không thể khai thác được do dữ liệu được chèn qua header. Lúc này ta cần chain nó với web cache poisoning. Ta sử dụng kĩ thuật web cache poisoning như thông thường để khai thác.

## Bypass flawed validation

Trong bất kể trường hợp nào mà web server sử dụng `Host header` và thực hiện validate nó không an toàn ta có thể bypass bằng cách tương tự như với cách bypass SSRF.

Trong 1 số trường hợp, Host Header cũng có thể chỉ được check ở request đầu trong 1 connection. Ta có thể test trường hợp này bằng cách set `Connection: keep-alive` và sử dụng group của Repeater của Burp và thực hiện send với mode single connection. Request đầu ta sử dụng host đúng và request thứ 2 giá trị của host là payload.

## Prevention

* Không nên sử dụng `Host Header` để thực hiện authentication, thay vào đó nên triển khai cơ chế login
* Nếu thực sự cần sử dụng `Host Header` trong 1 số trường hợp nào đó, nên whitelist nó chỉ nằm trong các giá trị an toàn
* Ưu tiên sử dụng relative URL
* Nên cấu hình không cho phép override headers











