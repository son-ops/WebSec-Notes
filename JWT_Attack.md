# JWT

Gồm 2 biến thể chính: `JWS` và `JWE`

* **JWS (JSON Web Signature)**: là dạng JWT phổ biến nhất, dữ liệu gồm 3 phần (header.payload.signature). Dữ liệu công khai, ai cũng có thể đọc được nhưng không thể sửa đổi.
* **JWE (JSON Web Encryption)**: Dạng JWT nâng cao (5 phần). Dữ liệu được mã hóa thành các ký tự rác, giúp bảo mật tuyệt đối các thông tin nhạy cảm, chỉ người có khóa giải mã mới xem được.

Chọn `JWE` khi cần bảo mật tuyệt đối (trong dữ liệu có chứa thông tin nhạy cảm - mật khẩu, số thẻ tín dụng chẳng hạn), tuy nhiên đổi lại quá trình mã hóa và giải mã tốn nhiều tài nguyên CPU và thời gian xử lý hơn. Ngược lại, ta nên ưu tiên sử dụng `JWS`.

# JWT Attack

## Missing Signature Verification

Thử thực hiện thay đổi 1 số kí tự trong phần signature và gửi lại request xem JWT này còn được chấp nhận không? -> nếu vẫn chấp nhận -> nó không thực hiện kiểm tra signature -> ta có thể tùy ý chỉnh sửa phần payload để khai thác.

## None Algorithm Attack

Trong trường hợp server check alg được gửi lên từ server và bỏ qua verification nếu như giá trị của nó là none -> Vulnerability

```js
...
function verfyToken(jwtToken){
    try {
        const [headerB64, payloadB64, signatureB64] = jwtToken.split('.');
        const header = JSON.parse(Buffer.from(headerB64, 'base64url').toString());
        const payload = JSON.parse(Buffer.from(payloadB64, 'base64url').toString());
        if(header.alg === 'none'){
            return payload; // Vulnerability
        }
        ...
    }
}
...
```

Exploit
```
Sửa alg trong header thành none
-> sign lại token
-> lúc này nó thành dạng header.payload. 
```

## Brute-force secret key

Trong trường hợp server sử dụng các thuật toán đối xứng (HS256, ...) ta có thể thử sử dụng hashcat để brute-force secret key.

``` bash
echo -n <jwt token> > jwt.txt
hashcat -m 16500 jwt.txt /opt/SecLists/Passwords/Leaked-Databases/rockyou.txt
```

Nếu brute thành công -> lấy secret đó để sign payload tùy ý (có thể sử dụng jwt.io)

## Exploit jwk header

```
The "jwk" (JSON Web Key) Header Parameter is the public key that corresponds to the key used to digitally sign the JWS.
This key is represented as a JSON Web Key. Use of this Header Parameter is OPTIONAL.
```
Nếu ứng dụng tin tưởng và sử dụng khóa do client gửi lên nằm trong trường jwk. Attacker có thể tận dụng điều đó để sign JWT 1 cách tùy ý.

### Trường hợp 1: Symetric algorithm

```bash
# jwk format với sysmetric algorithm
# Giá trị của khóa được base64 và đặt vào k
{
  "kty": "oct",
  "alg": "HS256",
  "use": "sig",
  "kid": "key_symmetric_01",
  "k": "c3VwZXJfc2VjcmV0X2tleV8xMjNfbm90X2Vhc3lfdG9fYnJ1dGVmb3JjZQ"
}

Để khai thác ta sử dụng khóa này để sign JWT một cách tùy ý
```


### Trường hợp 2: Assymetric algorithm

```bash
# jwk format với assymetric algorithm
{
  "kty": "RSA",
  "alg": "RS256",
  "use": "sig",
  "kid": "key_rsa_public_2026",
  "n": "u1W1to9jXvBR...[Chuỗi ký tự Base64Url rất dài đại diện cho Modulus]...",
  "e": "AQAB"
}

Để khai thác ta cần tạo 1 cặp rsa key -> sau đó sign với private key -> và gửi jwk với public key của private key đó
Câu lệnh tạo rsa key:
openssl genpkey -algorithm RSA -out exploit_private.pem -pkeyopt rsa_keygen_bits:2048
openssl rsa -pubout -in exploit_private.pem -out exploit_public.pem

Chuyển đổi qua lại giữa PEM và JWK: https://www-authgear-com.translate.goog/tools/jwk-generator/?_x_tr_sl=en&_x_tr_tl=vi&_x_tr_hl=vi&_x_tr_pto=tc
```

## Expoit jku header

```
The "jku" (JWK Set URL) Header Parameter is a URI that refers to a
resource for a set of JSON-encoded public keys,
one of which corresponds to the key used to digitally sign the JWS.
The keys MUST be encoded as a JWK Set. The protocol used to acquire the
resource MUST provide integrity protection;
an HTTP GET request to retrieve the JWK Set MUST use Transport Layer
Security (TLS),
and the identity of the server MUST be validated,
as per [Section 6 of RFC 6125]. Also, see [Section 8] on TLS requirements.
Use of this Header Parameter is OPTIONAL.
```

Kĩ thuật khai thác tương tự như jwk header nhưng thay vào đó ta cần host một server để nó trả về jwk sau đó truyền url tới server đó.

Ngoài ra, tham số này cũng có thể dẫn đến blind SSRF.

## Exploit x5c, x5u header

* `x5c` tương tự như `jwk`: chỉ khác ở chỗ nó nhúng trực tiếp chứng chỉ dưới dạng `X.509` được mã hóa base64 vào header
* `x5u` tương tự như `jku`: chỉ khác ở chỗ đường dẫn của nó trỏ tới 1 chứng chỉ dạng `X.509`

**Lưu ý**: Cơ chế `x5c` / `x5u` chỉ hỗ trợ và hoạt động với các thuật toán bất đối xứng (như RS256).

**Kịch bản tấn công**: 

```bash
Bước 1: Tạo private key và tạo chứng chỉ tự ký
# Tạo Private Key (Dùng để KÝ số - Sign Token)
openssl genpkey -algorithm RSA -out exploit_private.pem -pkeyopt rsa_keygen_bits:2048

# Tạo Chứng chỉ X.509 tự ký tương ứng (Dùng để nhúng vào Header)
openssl req -new -x509 -key exploit_private.pem -out exploit_cert.crt -days 365

Bước 2: sign token bằng file exploit_private.pem và base64 exploit_cert.crt ( bỏ các dòng BEGIN/END và xuống dòng) nhét vào trường x5c

{
  "alg": "RS256",
  "typ": "JWT",
  "x5c": ["<Chuỗi_Base64_của_exploit_cert.crt>"]
}
```

## Exploit kid header

Một header claim tùy chọn có tên kid được dùng để xác định một key cụ thể. Claim này đặc biệt quan trọng trong các môi trường có nhiều key dùng để xác minh chữ ký token. Claim này giúp chọn key phù hợp để xác minh chữ ký của token.

Server thường lấy giá trị của kid để thực hiện các câu lệnh tìm kiếm khóa. Nếu Server lập trình lỏng lẻo, kid có thể bị dính các lỗ hổng sau:

* **SQLi**: `SELECT * FROM keys WHERE id = 'INPUT'`
* **Path Traversal**: Nếu Server lưu các file khóa trên ổ cứng và đọc khóa theo kiểu `"path/to/keys/" + kid`, hacker có thể truyền vào `"kid": "../../../../../dev/null"` để ép Server đọc một file trống (tương đương với việc biến khóa thành chuỗi rỗng hoặc rác, giúp hacker dễ brute-force hoặc bypass chữ ký). Sau đó, ta sẽ sign JWT với private key là rỗng ''
* **Cmdi**: `/root/res/keys/secret7.key; cd /root/res/keys/ && python -m SimpleHTTPServer 1337&`

## Algorithm Confusion

Algorithm confusion is a attack that forces the web application to use a different algorithm to verify the JWT's signature.

```js
// Example vul code

const jwt = require('jsonwebtoken');
const fs = require('fs');

const PUBLIC_KEY = fs.readFileSync('public_key.pem', 'utf8'); 

function verifyToken(token) {
        // Lập trình viên không ép buộc thuật toán cố định trong cấu hình.
        // Hàm jwt.verify() sẽ tự động đọc trường "alg" trong Header của Token gửi lên.
        // Nếu "alg" là HS256, thư viện sẽ coi tham số PUBLIC_KEY là một chuỗi Secret Key đối xứng thô!
        
        const decoded = jwt.verify(token, PUBLIC_KEY); 
        return decoded;
    } catch (err) {
        return null;
    }
}
```

Nếu server sử dụng thuật toán bất đối xứng để sign và verify. Thì private key được dùng để sign và public key được sử dụng để verify. Nếu server không forces thuật toán để verify, đồng thời tin vào thuật toán qua header `alg`, kẻ tấn công có thể đổi thuật toán sang mã hóa đối xứng và sử dụng public key để sign. Lúc này server vẫn sử dụng public key để verify -> bypass.

**Kịch bản khai thác**:

* Bước 1: lấy được public key

Trong 1 số trường hợp, public key được lưu trên server như ở x5c, jwk header. Hay nó nằm ở 1 số path phổ biến như `/jwks.json`, `/.well-known`, ...

Tuy nhiên không phải lúc nào nó cũng được cung cấp bởi server. Trong trường hợp này ta có thể sử dụng https://github.com/silentsignal/rsa_sign2n để compute the public key.

```bash
git clone https://github.com/silentsignal/rsa_sign2n
cd rsa_sign2n/standalone/
docker build . -t sig2n
docker run -it sig2n /bin/bash

Đầu vào của công cụ sẽ cần JWT token hợp lệ dùng chung 1 public key
python3 jwt_forgery.py <token-1> <token-2>

Kết quả public key trả về sẽ tự động được lưu ở file trong container 
```

* Bước 2: Sử dụng public key đó để sign với `alg=HS256` (cyberchef).

Chú ý: cần add thêm newline at the end of public key 

## Reusing JWT secret

Nếu 1 doanh nghiệp hosts nhiều ứng dụng mà sử dụng JWTs cho việc xác thực, nhưng lại sử dụng chung 1 bộ key. Attacker có thể tận dụng điều này để sử dụng JWT ở 1 trang web để xác thực 1 trang web khác. Nếu như 1 trong 2 trang web cấp quyền cho user này cao hơn -> escalate privilege.

## Tools: jwt_tools

```bash
git clone https://github.com/ticarpi/jwt_tool
pip3 install -r requirements.txt
```

## Preventing

* Không nên tự viết hàm sign, verify chữ kí. Hãy dùng version mới nhất của các thư viện chuẩn, nổi tiếng (`jsonwebtoken` của nodejs hay `PyJWT` của python)
* Ép chặt thuật toán được sử dụng, không tin tưởng vào các trường trong header JWT.
* Nếu cho phép tải chứng chỉ từ `jku`, `x5u` thì cần phải `whitelist hosts` nghiêm nghặt để tránh SSRF.
* Thêm trường `exp` vào payload để đảm bảo token không có giá trị vĩnh viễn, ngăn chặn tấn công phát lại
