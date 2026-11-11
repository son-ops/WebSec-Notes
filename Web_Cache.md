# Cache Struture

```text
User
  |
  v
Browser
  |
  v
Client-side Cache
  Lưu tài nguyên (HTML, CSS, JS, hình ảnh...) trên thiết bị
  người dùng, thường trong RAM hoặc ổ đĩa. Giúp trình duyệt
  tái sử dụng dữ liệu khi truy cập lại mà không cần tải lại.
  |
  v
Shared Cache (CDN / Proxy)
  Lưu các HTTP response có thể chia sẻ giữa nhiều người dùng.
  Khi Cache HIT, trả dữ liệu trực tiếp mà không cần chuyển
  request đến Origin Server.
  |
  v
Server-side Cache
  Lưu dữ liệu phía Server như kết quả truy vấn Database,
  session hoặc dữ liệu nghiệp vụ (thường dùng Redis,
  Memcached), giúp giảm truy vấn và xử lý lặp lại.
  |
  v
Backend (Application / Database)
  Xử lý logic nghiệp vụ, xác thực, truy vấn hoặc cập nhật
  Database khi các tầng cache không có dữ liệu phù hợp.
```

# Shared Cache (Web Cache)

Cơ chế của tầng Cache này dựa trên `Cache Key` và `Cache Rule`.

`Cache key`: xác định danh tính của một cache entry, tham số này quyết định 1 request có được phục vụ trực tiếp từ cache không hay tiếp tục gửi nó đến backend. `Cache key` thường bao gồm `schema`, `host`, `uri path` và `query string`. Hai request có cùng cache key sẽ được truy cập vào cùng 1 cache entry.

`Cache rule`: quyết định 1 request có được cache không. Thường dựa vào các extension như `.js`, `.css` hay là các path như `/resource`, `/static` hoặc có thể là fixed file name `/index.js`, `style.css`.

# Cache Buster

Khi mà thực hiện khai thác 2 kĩ thuật bên dưới, để tránh ta làm việc trực tiếp với cache response ta cần sử dụng cache key khác nhau ở mỗi request. Vì url path và any query parameter thường được sử dụng làm cache key, khi test ta chỉ cần thêm unique param ở mỗi request.

# Detect cached response

Ta có thể detect dựa vào header response được trả về:

* **X-Cache**: hit, miss, dynamic (server generated content), refresh (cached content was outdated và cần cache lại)
* **Cache-Control**: public (cho phép cả Browser Cache và Shared Cache lưu response), private (chỉ cho phép Private Cache như Browser Cache lưu response, không cho phép Shared Cache như CDN/Proxy lưu response), no-store (không cho phép bất kỳ HTTP Cache nào lưu response), no-cache (cho phép lưu response nhưng phải xác thực lại với Origin Server trước khi sử dụng), response header này có thể bị ghi đè bởi CDN và Proxy, và Cache vẫn có thể từ chối lưu vì các chính sách khác.

* Hoặc có sự khác nhau lớn về response time cho same request, vì rất có thể response được trả về nhanh hơn đang được phục vụ trực tiếp từ cache.

# Web Cache Deception

Là kĩ thuật lợi dụng `cache rule` để lừa nạn nhân cache những thông tin nhạy cảm. Sau khi nó được cache, thông tin này có thể được truy cập bởi attacker.

Quy trình để tìm bao gồm:

* **Bước 1**: tìm target endpoint mà trả về thông tin nhạy cảm. Thường tập trung vào các `GET`, `HEAD` or `OPTIONS` request, vì các request mà thay đổi origin server's state thường không được cache
* **Bước 2**: Tìm sự khác biệt giữa cách cache và origin server parse the URL path (Map URLs to resource, process delimiter charaters, normalize paths)
* **Bước 3**: craft a malicious URL mà sử dụng sự khác biệt để đánh lừa cache server tự động cache response. Gửi malicious URL này cho victim, khi victim truy cập, their response được lưu vào cache. Attacker truy cập lại vào malicious URL và đọc sensitive information của victim.

## Static extension cache rule

Cache rule thường nhắm tới static resources bằng cách matching với file extensions phổ biến như là `.css` hay `.js`.

### Path mapping discrepancies

Có rất nhiều cách khác nhau được sử dụng để mapping. Nếu 2 cơ chế này giữa cache và origin server mismatch, attacker có thể lợi dụng để exploit web cache deception. Ví dụ: một RESTful URL mapping như sau:

```
http://example.com/path/resource/param1
```

RESTful này thực hiện mapping dựa trên param1 và param2. Ví dụ 

```
http://example.com/path/resource/user
```
sẽ map tới thông tin của user đang truy cập

Tuy nhiên:

```
http://example.com/path/resource/user/abc.js
```

cũng mapping tới thông tin của user do `abc.js` là 1 param không dùng đến từ đó dẫn đến sự mismatch giữa cache và origin server.

Ta có thể khai thác trong trường hợp này bằng cách gửi `http://example.com/path/resource/user/abc.js` cho victim, khi victim truy cập nó thực hiện cached response, attacker sau đó chỉ cần truy cập lại url trên và đọc thông tin của victim.

### Delimiter discrepancies

Ví dụ, trong java phần đằng sau ; sẽ được omit: `http://example.xyz/profile;hihi.js`. Lúc này, origin sẽ map tới `http://example.xyz/profile` và bỏ qua `;hihi.js`. Tuy nhiên, cache server thì lại không xử lí như vậy, nó coi path sẽ là cả `/profile;hihi.js`. Để khai thác trong trường hợp này ta chỉ cần gửi url `http://example.xyz/profile;hihi.js` này cho victim.

```
# Các delimiter phổ biến

!
"
#
$
%
&
'
(
)
*
+
,
-
.
/
:
;
<
=
>
?
@
[
\
]
^
_
`
{
|
}
~
%21
%22
%23
%24
%25
%26
%27
%28
%29
%2A
%2B
%2C
%2D
%2E
%2F
%3A
%3B
%3C
%3D
%3E
%3F
%40
%5B
%5C
%5D
%5E
%5F
%60
%7B
%7C
%7D
%7E
%00
%0A
%09
```

## Static directory cache rules

### Normalize discrepancies

#### Detect normalization by origin server

Để test, ta thực hiện gửi 1 request `non-cacheable` (ví dụ sử dụng POST) với 1 path traversal sequence bên dưới:

```
GET /aaa/..%2fprofile
```
Nếu server vẫn trả về profile -> server có decode và resolve dot segment và ngược lại

#### Detect normalization by cache server

Để test, ta chọn 1 request với response đã được cache. Ví dụ:

```
GET /static/asset.js
```

Sau đó thực hiện thêm traversal 

```
GET /static/abc/..%2fasset.js
```

Nếu response trả về vẫn được phục vụ từ cache -> cache server có decode và resolve dot segment và ngược lại.

#### Exploit normailization discrepancies

Trường hợp 1: origin server decode và resolve nhưng cache server không

```
GET /static/..%2fprofile
```

Điều kiện phải `static` cần phải là thư mục nằm trong cache rule

Trường hợp 2: cache server decode và resolve nhưng origin server không

```
GET /profile;/..%2fstatic/asset.js
```

Trong trường hơp này ta cần tìm thêm delimiter để khai thác.

## File name cache rules

trong trường hợp này chỉ có thể khai thác thông qua normalization discepancies khi cache server decode và resolve dot-segment còn origin server thì không. Cách thực hiện tương tự như trên

## Prevention

* Cấu hình `Cache-Control: no-store, private` cho những endpoint chứa thông tin nhạy cảm
* Cấu hình CDN, Proxy không ghi đè Cache-Control đồng thời chỉ cache ở những vị trí cần thiết
* Kích hoạt cơ chế bảo vệ Web Cache Deception của CDN, proxy: một số CDN, proxy hỗ trợ tính năng này. Ví dụ nó có cơ chế kiểm tra sự tương thích giữa Content-Type trong http response và phần extension trong URL.
* Đảm bảo không có sự khác biệt giữa cách interpret url path của cache và origin server

# Web Cache Poisoning

Là kĩ thuật lợi dụng `cache key` để chèn malicious content vào cached response, cái mà sau đó được phục vụ cho nhiều users khác. Nó thường được sử dụng để delivery malicious content to a vast number of users. Hoặc nó cũng được dùng tăng mức độ nghiêm trọng của lỗ hổng khác, hoặc tận dụng những lổ hổng dường như không thể khai thác nếu đứng một mình trở thành vũ khí tấn công lợi hại.

Ví dụ: khi mà 1 unkey header được reflected lại trong response và không được escape đầy đủ dẫn đến self XSS. Tự thân nó thì dường như không thể khai thác nhưng nếu ta tận dụng cache key để cache response với unkey header này là 1 payload XSS. Sau đó, payload này sẽ được phục vụ cho tất cả người dùng mà truy cập vào cùng cache entry này.

Quy trình để tìm bao gồm:

* **Bước 1**: tìm kiếm `unkeyed inputs`. Để tìm gía trị này, ta có thể thử gửi 2 request (với unique value) và xem nó có được phục vụ trực tiếp từ cache không. Ví dụ: muốn check param url, ta có thể gửi request đầu tiên với `?url=xyz1`, và request thứ 2 với `?url=xyz2`. Nếu cả 2 request đều cache miss -> đây là keyed input. Vì khác key nên thực hiện cache lại. Nếu request đầu là miss và request sau là hit -> đây là unkeyed input.
* **Bước 2**: tận dụng những `unkeyed input`, xem xét cách mà server xử lí và có thể tận dụng nó để chèn malicious content không? Nếu có thể thì cho nó cache vào các entry cache mà có nhiều user cùng sử dụng chung.
* **Bước 3**: get the response cached

`unkeyed inputs` ở đây ta có thể quan tâm tới các parameter, các header (host header, custom header, ...), cookie. Và ta có thể sử dụng Param Miner extension trong burp để tự động hóa việc tìm kiếm. Ngoài ra 1 số server hỗ trợ hiển thị những thành phần nào được coi như là cache key thông qua header `Pragma: x-get-cache-key`

Ngoài các unkeyed input có thể tìm trực tiếp, ta có 1 số cách khác như Fat GET và parameter Cloaking.

## Fat GET

Đây là HTTP Get request mà chứa phần body. Nếu server misconfigured hoặc triển khai không chính xác, nó có thể parse body of GET request. Lúc này các keyed input khi truyền qua body có thể trở thành unkeyed input, từ đó ta có thể tận dụng để khai thác web cache poisoning tương tự như trên.

## Parameter Cloaking

Đây là kĩ thuật lợi dụng sự khác nhau giữa parse các keyed input. Ví dụ vulnerability được phát hiện trong web framework Bottle (`CVE-2020-28473`). Bottle cho phép `;` để phân tách giữa 2 parameter. Ví dụ: `?url=xyz;ref=abc`. Bottle lúc này sẽ coi server chứa 2 tham số là `url=xyz` và `ref=abc`. Tuy nhiên, cache server thì lại chỉ nhìn thấy 1 tham số duy nhất là `url=xyz;ref=abc`. Giả sử `ref` được server xử lí và reflected lại không đúng cách dẫn đến XSS. Trong trường hợp này ta có thể khai thác bằng cách: `?url=xyz&a=b;ref=<payload_xss_here>`. Payload lúc này sẽ được cache vào cache entry `url=xyz` và tất cả user truy cập entry này sẽ thực thi payload XSS.


## Tool: https://github.com/Hackmanit/Web-Cache-Vulnerability-Scanner

## Prevention

* Không nên hỗ trợ `Fat GET` request
* Nên cho tất cả những tham số có ảnh hưởng đến response vào `keyed input`
* Cập nhật phiên bản của server nên mới nhất để tránh xảy ra mismatch ( ví dụ dẫn đến parameter cloaking )
* Vá mọi XSS kể cả trong trường hợp nó chưa thể tự khai thác được
* Chỉ thực hiện cache những tài nguyên cần thiết
