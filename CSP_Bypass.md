# Content Security Policy (CSP)

## 1. Tổng quan về CSP

**Content Security Policy (CSP)** là một cơ chế bảo mật được trình duyệt thực thi nhằm kiểm soát những tài nguyên mà một trang web được phép tải và thực thi, chẳng hạn như JavaScript, CSS, hình ảnh, font chữ và các tài nguyên bên ngoài.

CSP thường được cấu hình thông qua HTTP response header:

```http
Content-Security-Policy: default-src 'self'; script-src 'self'
```

Ngoài ra, CSP có thể được khai báo thông qua thẻ `<meta>` trong HTML:

```html
<meta http-equiv="Content-Security-Policy" content="default-src 'self'">
```

Tuy nhiên, cấu hình qua HTTP response header được khuyến nghị hơn vì hỗ trợ đầy đủ các directive và được áp dụng sớm hơn trong quá trình xử lý tài liệu.

CSP đóng vai trò là một lớp bảo vệ bổ sung (**Defense-in-Depth**) nhằm giảm thiểu nguy cơ khai thác các lỗ hổng như Cross-Site Scripting (XSS). CSP không thay thế việc xử lý XSS đúng cách, mà giúp hạn chế khả năng thực thi mã độc ngay cả khi ứng dụng tồn tại điểm chèn nội dung không an toàn.

## 2. Cách cấu hình CSP

### 2.1. Cấu trúc CSP

Một CSP policy bao gồm nhiều directive, mỗi directive xác định loại tài nguyên hoặc hành vi được kiểm soát và các nguồn được phép tương ứng.

Cấu trúc:

```http
Content-Security-Policy: directive1 value1; directive2 value2
```

Ví dụ:

```http
Content-Security-Policy: default-src 'self'; script-src 'self'; img-src *
```

Trong đó:

- `default-src`: Directive mặc định, được sử dụng làm fallback cho những directive có cơ chế kế thừa từ nó.
- `script-src`: Kiểm soát nguồn tải và thực thi JavaScript (\<script\>, event onerror, onload, ...)
- `style-src`: Kiểm soát nguồn tải CSS. (\<style\>)
- `img-src`: Kiểm soát nguồn tải hình ảnh. (\<img\>)
- `media-src`: Kiểm soát nguồn tải audio và video. (\<media\>)
- `frame-src`: Kiểm soát nguồn nội dung được tải bên trong iframe. (\<frame\>, \<iframe\>)
- `connect-src`: Kiểm soát các kết nối được khởi tạo từ JavaScript, chẳng hạn như Fetch, XMLHttpRequest và WebSocket.
- `font-src`: Kiểm soát nguồn tải font chữ.
- `object-src`: Kiểm soát các tài nguyên được tải thông qua thẻ `<object>` hoặc `<embed>`.
- `frame-ancestors`: Kiểm soát những website được phép nhúng trang hiện tại vào iframe.
- `form-action`: Kiểm soát những URL được phép nhận dữ liệu từ form.
- `base-uri`: Kiểm soát các URL được phép sử dụng trong thẻ `<base>`.

Lưu ý: `default-src` không phải directive fallback cho tất cả các directive. Ví dụ, `frame-ancestors`, `form-action` và `base-uri` không kế thừa giá trị từ `default-src`.

### 2.2. Các giá trị nguồn phổ biến

Một directive có thể chứa nhiều giá trị nguồn để xác định tài nguyên nào được phép sử dụng.

Các giá trị phổ biến:

- `'self'`: Cho phép tài nguyên có cùng origin với trang hiện tại (bao gồm scheme, hostname và port).
- `'none'`: Không cho phép bất kỳ nguồn nào.
- `*`: Cho phép nhiều nguồn theo quy tắc wildcard của CSP.
- `'unsafe-inline'`: Cho phép thực thi JavaScript hoặc CSS được khai báo trực tiếp trong HTML, tùy directive.
- `'unsafe-eval'`: Cho phép thực thi mã JavaScript được tạo động thông qua các hàm như `eval()`, `Function()` hoặc `setTimeout()` với tham số dạng chuỗi.
- `https://example.com`: Cho phép tải tài nguyên từ hostname được chỉ định.
- `*.example.com`: Cho phép tải tài nguyên từ các subdomain của `example.com`, nhưng không bao gồm chính `example.com`.

Ví dụ:

```http
Content-Security-Policy: script-src 'self' https://cdn.example.com
```

Cấu hình trên cho phép tải JavaScript từ cùng origin và `cdn.example.com`. Những nguồn không nằm trong danh sách cho phép sẽ bị trình duyệt chặn.

Lưu ý: `example.com` và `www.example.com` là hai hostname khác nhau. Nếu muốn cho phép cả domain gốc và các subdomain, cần khai báo phù hợp:

```http
Content-Security-Policy: script-src https://example.com https://*.example.com
```

### 2.3. Kiểm soát inline script và eval

Mặc định, khi `script-src` hoặc directive fallback tương ứng được áp dụng, CSP sẽ chặn các đoạn JavaScript inline, bao gồm:

```html
<script>
    alert(1);
</script>

<img src="x" onerror="alert(1)">

<a href="javascript:alert(1)">Click</a>
```

Để cho phép thực thi inline JavaScript, có thể sử dụng:

```http
Content-Security-Policy: script-src 'unsafe-inline'
```

Tuy nhiên, cấu hình này làm suy yếu khả năng chống XSS vì trình duyệt sẽ cho phép thực thi các đoạn JavaScript inline, bao gồm cả mã độc được chèn vào HTML.

Tương tự, `'unsafe-eval'` cho phép thực thi JavaScript được tạo động:

```http
Content-Security-Policy: script-src 'unsafe-eval'
```

Ví dụ:

```javascript
eval("alert(1)");
```

Với cấu hình trên, `eval()` được phép thực thi. Tuy nhiên, điều này không đồng nghĩa với việc trình duyệt cho phép tải JavaScript từ mọi domain hoặc thực thi mọi đoạn script inline.

### 2.4. Nonce-based CSP

Một phương pháp an toàn hơn để cho phép inline script là sử dụng nonce.

Nonce là một giá trị ngẫu nhiên, khó dự đoán, được máy chủ tạo ra cho mỗi HTTP response. Giá trị này được khai báo đồng thời trong CSP header và thuộc tính `nonce` của những thẻ script hợp lệ.

Ví dụ:

```http
Content-Security-Policy: script-src 'nonce-a8f3d91c'
```

```html
<script nonce="a8f3d91c">
    alert("Hello");
</script>
```

Trình duyệt sẽ so sánh nonce trong CSP với nonce được khai báo trên thẻ `<script>`. Nếu hai giá trị khớp nhau, script được phép thực thi.

Ngược lại, những đoạn script inline không có nonce hợp lệ sẽ bị chặn.

**Lưu ý:**

- Nonce phải được tạo bằng bộ sinh số ngẫu nhiên an toàn về mặt mật mã (CSPRNG).
- Mỗi HTTP response cần sử dụng một nonce mới.
- Không sử dụng nonce cố định hoặc có thể dự đoán.
- Nonce không phải chữ ký số và không xác minh nội dung của script. Nó chỉ xác định những script element được phép thực thi.

Nếu attacker có thể lấy được nonce hợp lệ của response và chèn một thẻ `<script>` mang nonce đó, họ có thể vượt qua cơ chế kiểm soát này.

### 2.5. Hash-based CSP

Một phương pháp khác là sử dụng hash để cho phép thực thi những đoạn inline script có nội dung xác định.

CSP hỗ trợ các thuật toán SHA-256, SHA-384 và SHA-512.

Ví dụ:

```http
Content-Security-Policy: script-src 'sha256-<base64-hash>'
```

Trình duyệt sẽ tính toán hash của nội dung script và so sánh với giá trị được khai báo trong CSP. Nếu hash khớp, script được phép thực thi.

Ví dụ:

```html
<script>
    alert("Hello");
</script>
```

Nếu nội dung script thay đổi, kể cả một số thay đổi về khoảng trắng hoặc xuống dòng, hash cũng có thể thay đổi và cần được tính toán lại.


**Một lưu ý quan trọng:** Khi `script-src` chứa nonce-source hoặc hash-source, trình duyệt hiện đại sẽ bỏ qua `'unsafe-inline'` trong directive đó. Điều này ngăn việc sử dụng `'unsafe-inline'` để vô hiệu hóa mục đích bảo vệ của nonce hoặc hash.

Ví dụ:

```http
Content-Security-Policy: script-src 'unsafe-inline' 'nonce-abc123'
```

Cấu hình trên không đồng nghĩa với việc mọi inline script đều được phép. Những script inline cần có nonce hợp lệ mới được thực thi.


## 3. CSP Bypass

Mặc dù CSP giúp giảm thiểu nguy cơ XSS, việc cấu hình không chặt chẽ hoặc tin tưởng sai nguồn tài nguyên có thể khiến chính sách bị vượt qua.

### 3.1. Sử dụng unsafe-inline

Nếu CSP cho phép inline JavaScript:

```http
Content-Security-Policy: script-src 'unsafe-inline'
```

Attacker có thể lợi dụng điểm chèn HTML để thực thi mã JavaScript:

```html
<script>
    alert(document.domain);
</script>
```

Hoặc sử dụng inline event handler:

```html
<img src="x" onerror="alert(document.domain)">
```

Do đó, việc sử dụng `'unsafe-inline'` làm giảm đáng kể hiệu quả bảo vệ XSS của CSP.

### 3.2. Upload JavaScript trên cùng origin

Giả sử ứng dụng có chính sách:

```http
Content-Security-Policy: default-src 'none'; img-src 'self'; style-src *; script-src 'self'
```

Cấu hình `script-src 'self'` cho phép trình duyệt tải JavaScript từ cùng origin với ứng dụng.

Tuy nhiên, nếu ứng dụng cho phép người dùng upload file JavaScript và file này được phục vụ trực tiếp từ cùng origin, attacker có thể lợi dụng chức năng upload để đưa mã độc vào một nguồn mà CSP đã tin tưởng.

Ví dụ:

```html
<script src="/uploads/attack.js"></script>
```

Do `/uploads/attack.js` thuộc cùng origin, trình duyệt có thể cho phép thực thi script này.

Tuy nhiên, khả năng khai thác còn phụ thuộc vào cách máy chủ phục vụ file, MIME type và các cơ chế bảo vệ khác như `X-Content-Type-Options: nosniff`.

**Phòng tránh:**

- Không phục vụ nội dung do người dùng upload dưới dạng tài nguyên có thể thực thi trên cùng origin.
- Lưu trữ file upload trên một domain riêng biệt.
- Kiểm tra loại file và nội dung trước khi lưu trữ.
- Cấu hình MIME type phù hợp và sử dụng `X-Content-Type-Options: nosniff`.

### 3.3. JSONP-based CSP Bypass

JSONP (JSON with Padding) là một kỹ thuật cho phép lấy dữ liệu cross-origin thông qua thẻ `<script>`.

Ví dụ, một JSONP endpoint có thể trả về:

```javascript
handleData({
    "username": "admin"
});
```

Khi đó, trình duyệt sẽ thực thi nội dung response như JavaScript.

Giả sử CSP cho phép tải script từ một domain đáng tin cậy:

```http
Content-Security-Policy: script-src 'self' https://trusted.example.com
```

Nếu domain này có một JSONP endpoint cho phép attacker kiểm soát callback, attacker có thể lợi dụng endpoint đó để thực thi JavaScript tùy ý.

Ví dụ về mặt nguyên lý:

```html
<script src="https://trusted.example.com/jsonp?callback=alert(1)"></script>
```

Nếu endpoint trả về:

```javascript
alert(1)({...});
```

thì JavaScript sẽ được thực thi trong context của trang hiện tại.

**Lưu ý:** Không phải mọi endpoint trên domain được whitelist đều hỗ trợ JSONP. Khả năng khai thác phụ thuộc vào endpoint cụ thể, cách xử lý callback và response thực tế.

### 3.4. Whitelist quá rộng

Một số ứng dụng sử dụng wildcard hoặc whitelist quá nhiều domain:

```http
Content-Security-Policy: script-src *
```

Hoặc:

```http
Content-Security-Policy: script-src https://*.example.com
```

Điều này có thể khiến chính sách trở nên kém hiệu quả nếu các domain được tin tưởng chứa endpoint không an toàn, JSONP hoặc nội dung do người dùng kiểm soát.

Ví dụ, nếu một subdomain cho phép upload file JavaScript hoặc cung cấp JSONP endpoint có thể kiểm soát callback, attacker có thể lợi dụng nguồn tài nguyên này để vượt qua CSP.

Do đó, việc whitelist một domain đồng nghĩa với việc ứng dụng đang tin tưởng các nguồn script phù hợp trên domain đó. Cần đánh giá kỹ từng domain trước khi đưa vào chính sách.

## 4. Phòng tránh CSP Bypass

Để CSP phát huy hiệu quả, cần chú ý các nguyên tắc sau:

- **Sử dụng chính sách tối thiểu:** Chỉ cho phép những nguồn tài nguyên thực sự cần thiết. Tránh sử dụng wildcard hoặc whitelist quá rộng.
- **Ưu tiên Nonce hoặc Hash:** Sử dụng nonce hoặc hash để kiểm soát inline script thay vì `'unsafe-inline'`.
- **Hạn chế unsafe-eval:** Không cho phép `eval()` và các cơ chế thực thi mã động nếu ứng dụng không thực sự cần thiết.
- **Kiểm soát file upload:** Không để nội dung do người dùng upload có thể được thực thi như JavaScript trên cùng origin với ứng dụng.
- **Kiểm tra domain được whitelist:** Đánh giá các endpoint có thể chứa JSONP, nội dung do người dùng kiểm soát hoặc các chức năng có khả năng trở thành script gadget.
- **Sử dụng các directive bổ sung:** Có thể cấu hình `object-src 'none'`, `base-uri 'none'`, `frame-ancestors` và `form-action` để hạn chế thêm các bề mặt tấn công.
- **Kiểm thử trước khi áp dụng:** Sử dụng `Content-Security-Policy-Report-Only` để ghi nhận các vi phạm chính sách mà chưa chặn tài nguyên, từ đó điều chỉnh cấu hình trước khi chuyển sang chế độ thực thi.

Ví dụ một chính sách CSP tương đối chặt chẽ:

```http
Content-Security-Policy: default-src 'none'; script-src 'self'; connect-src 'self'; img-src 'self'; style-src 'self'; frame-ancestors 'self'; form-action 'self'; object-src 'none'; base-uri 'none'
```

Chính sách trên chỉ là ví dụ. Khi triển khai thực tế, cần điều chỉnh theo nhu cầu của ứng dụng để tránh chặn nhầm các tài nguyên hợp lệ.
