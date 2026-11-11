# Session

Như ta đã biết, HTTP là giao thức stateless. Hay nói cách khác, mỗi request HTTP sẽ được coi là độc lập với nhau. Vậy nếu một web bán hàng có chức năng thêm vào giỏ hàng sau đó thanh toán 1 cách độc lập thì server sẽ làm cách nào để biết user nào đang thực hiện thanh toán, thực hiện thanh toán giỏ hàng nào hay trong giỏ hàng đang chứa những gì? Đó là lúc cần sử dụng đến Session. 

Session được sử dụng để cung cấp context cần thiết để thực hiện những hành động liên quan đến nhau. Session cũng có 2 loại là stateless và statefull. 

**Stateless session**: là session token mà chứa tất cả thông tin cần thiết trong chính token rồi. Ví dụ như JWT

**Statefull session**: token được server tạo, sau đó các thông tin cần thiết sẽ được linking với token này. Nó bao gồm: `Session ID`, `Session Varibles`.

* **Session ID**: là token định danh duy nhất, server sử dụng giá trị này để tìm session tương ứng trong bộ nhớ hoặc hệ thống lưu trữ
* **Session Varibles**: là các biến chứa dữ liệu trạng thái của người dùng được lưu trữ phía server. Những biến này giúp server ghi nhớ các thông tin, trạng thái của user.

```php
# Ví dụ

$_SESSION['user_id'] = 123;
$_SESSION['username'] = 'son';
$_SESSION['cart'] = ['product_A', 'product_B'];
$_SESSION['step'] = 2;
```

# Session Puzzling

Là lỗ hổng bắt nguồn từ việc xử lí `Session Varibles` không chính xác. Thường xảy ra ở các ứng dụng mà có nhiều chức năng cùng sử dụng chung 1 session, ví dụ như:

* **Authentication:** Sau khi đăng nhập thành công, ứng dụng thiết lập `authenticated = true`. Tuy nhiên, khi logout, ứng dụng chỉ xóa `user_id` mà không reset `authenticated`. Nếu chức năng khác chỉ kiểm tra biến này để xác định trạng thái đăng nhập, attacker có thể lợi dụng trạng thái còn sót lại để truy cập trái phép.

* **Password Reset:** Sau khi xác minh danh tính trong quy trình Reset Password, ứng dụng thiết lập `verified = true` nhưng không xóa biến này sau khi hoàn tất. Nếu chức năng Change Password cũng sử dụng biến `verified` để kiểm tra xác thực, attacker có thể tận dụng trạng thái cũ để bỏ qua bước xác minh.

* **Multi-step Forms:** Trong quy trình đăng ký, sau khi xác minh email, ứng dụng thiết lập `verified = true`. Tuy nhiên, nếu quy trình thanh toán cũng sử dụng chung biến này để kiểm tra trạng thái xác minh, attacker có thể lợi dụng trạng thái từ quy trình đăng ký để bỏ qua bước xác minh trong thanh toán.

* **Account Management:** Khi thay đổi số điện thoại, ứng dụng yêu cầu xác minh OTP và thiết lập `verified = true`. Nếu chức năng thay đổi email cũng sử dụng biến này mà không phân biệt mục đích xác minh, attacker có thể tận dụng trạng thái xác minh OTP trước đó để thay đổi email mà không cần xác minh tương ứng.

## Authentication Bypass

### Via logout

Giả sử server xử lí `login` và `logout` như sau:

```php
<?php
reqiure_once('db.php');
session_start();

// login
if(check_password($_POST['username'], $_POST['password'])) {
    $_SESSION['user_id'] = get_user_id($username);
    header("Location: profile.php");
    die();
} else {
    echo "Unauthorized";
}

// logout
if(isset($_POST['logout'])) {
    $_SESSION['user_id'] = 0;
}
?>
```

Trong thì có vẻ an toàn, nhưng hãy nhìn vào đoạn mã khi `logout`. Server không thực hiện hủy session, thay vào đó lại set nó = 0. Mà đây lại thường là id của user đầu tiên - admin. Như vậy ta attacker truy cập truy cập vào logout để server set `$_SESSION['user_id'] = 0` sau đó thực hiện truy cập thẳng các trang post-login. Lúc này nó sẽ được truy cập dưới user có id = 0.

### Via Password Reset

Giả sử server có chức năng `Password Reset` với flow như sau:

```
Request 1: Nhập gmail hoặc username
Request 2: Answer security question
Request 3: Reset password
```
Vì những request này được xử lí một cách độc lập, đồng thời ở bước `answer security question` cũng không gửi thêm thông tin nào để cung cấp question này trả lời cho user nào. Vì thể ta có thể đoạn được server đang lưu trạng thái thông qua Session Varibles.

Ở đây sẽ sinh ra 2 hướng khai thác:

**Hướng 1**: nhập username để lấy token ở bước 1 (có thể chứa thông tin về user), sau đó sử dụng token này nhảy trực tiếp tới request 3 để reset password mà không cần request 2.

**Hướng 2**: Nhập username để lấy token ở bước 1, sau đó truy cập trực tiếp `profile` 

Example code vul hướng 2:

```php
# Reset_1.php
<SNIP>
if(isset($_POST['Submit'])){
    $_SESSION['Username'] = $_POST['Username'];
    header("Location: reset_2.php");
    exit;
}
<SNIP>

# Profile.php
<SNIP>
    if(!isset($_SESSION['Username'])){
    header("Location: login.php");
    exit;
}
<SNIP>
```

### Via premature session population

Một hướng khai thác khác của lỗ hổng này đó là premature session population. Nó bắt nguồn từ việc premature population of session variables. Có nghĩa là server thực hiện lưu data vào trong session varibles quá sớm trước khi xử lí xong toàn bộ process hay trước khi biết được kết quả của process. 

Ví dụ một trang web thực hiện chức năng đăng nhập như sau:

```php
<SNIP>
if(isset($_POST['Submit'])){

    $_SESSION['Username'] = $_POST['Username'];
    $_SESSION['Active'] = true;
    
    // check user credentials
    if(login($Username, $_POST['Password'])) {
    header("Location: profile.php");
    exit;
} else {
    header("Location: login.php?failed=1");
    exit;
}
}
if (isset($_GET['failed'])) {
    session_destroy();
    session_start();
}
<SNIP>
```

Ta thấy session varibles được khởi tọa ngay sau khi nhận submit. Trong trường hợp failed nó mới thực hiện hủy biến phiên này. Tuy nhiên, trong trường hợp này attacker có thể submit sau đó, không follow redirect, sau đó thực hiện dùng token được trả về và truy cập trực tiếp vào `/profile.php`. Dẫn đến Auth Bypass

### Via combined different process

Giả sử một ứng dụng triển khai 2 chức năng registration và reset password. Luồng của 2 chức năng đó như sau:

* **Registration**: phase1 nhập thông tin -> phase2 nhập ... -> phase3 register
* **Reset Password**: phase1 nhập -> phase2 answer security question -> phase3 reset password

Khi test với các hướng ở trên đều không thành công, lúc này ta có thể nghĩ tới 1 hướng khác đó là cả 2 chức năng này đều sử dụng chung 1 biến để lưu trạng thái phase. Lúc này ta có thể test bằng cách:

* Đầu tiên, thực hiện phase 1 của reset password với username `admin`
* Tiếp theo, thực hiện phase 1 và phase 2 với tài khoản bất kì ở bước đăng kí
* Sử dụng token sesion trả về ở bước đăng kí để thực hiện phase 3 của reset password

Example vul code:

```php
# register_1.php
<SNIP>
if (isset($_POST['Submit'])) {
    $_SESSION['reg_username'] = $_POST['Username'];
    $_SESSION['reg_desc'] = $_POST['Description'];
    $_SESSION['reg_pw'] = $_POST['Password'];
    $_SESSION['reg_question'] = $_POST['Question'];
    $_SESSION['reg_answer'] = $_POST['Answer'];
    $_SESSION['Phase'] = 2;

    header("Location: register_2.php");
    exit;
}
<SNIP>


# register_2.php
<SNIP>
if ($_SESSION['Phase'] !== 2) {
    header("Location: login.php?msg=Please complete Phase 1 first");
    exit;
}

if (isset($_POST['Submit'])) {
    // Registration Phase 2 completed
    $_SESSION['Phase'] = 3;

    header("Location: register_3.php");
    exit;
}
<SNIP>


# register_3.php
<SNIP>
if ($_SESSION['Phase'] !== 3) {
    header("Location: login.php?msg=Please complete Phase 2 first");
    exit;
}

// Create account using registration data
create_account(
    $_SESSION['reg_username'],
    $_SESSION['reg_pw'],
    $_SESSION['reg_question'],
    $_SESSION['reg_answer']
);
<SNIP>


# reset_1.php
<SNIP>
if (isset($_POST['Submit'])) {
    $user_data = fetch_user_data($_POST['Username']);

    if ($user_data) {
        $_SESSION['reset_username'] = $user_data['username'];
        $_SESSION['Phase'] = 2;

        header("Location: reset_2.php");
        exit;
    }
}
<SNIP>


# reset_2.php
<SNIP>
if ($_SESSION['Phase'] !== 2) {
    header("Location: login.php");
    exit;
}

if (isset($_POST['Submit'])) {
    // Verify security question
    if (check_security_answer(
        $_SESSION['reset_username'],
        $_POST['Answer']
    )) {
        $_SESSION['Phase'] = 3;

        header("Location: reset_3.php");
        exit;
    }
}
<SNIP>


# reset_3.php
<SNIP>
if ($_SESSION['Phase'] !== 3) {
    header("Location: login.php");
    exit;
}

// Reset password
update_password(
    $_SESSION['reset_username'],
    $_POST['Password']
);
<SNIP>
```

## Weak Session ID

Theo OWASP, độ dài tối thiểu của session là `16 bytes`. Nếu session có độ dài quá ngắn, attacker hoàn toàn có thể brute-force để thực hiện có được session của user khác.

Ngoài việc có độ dài đủ lớn, session token cũng cần có tính ngẫu nhiên đủ lớn. Tính ngẫu nhiên thường được đánh giá dựa trên `entropy`. Theo OWASP, tính ngẫu nhiên phải được cung cấp tối thiểu `64 bits of entropy`. Để kiểm tra tính ngẫu nhiên của một token session ta có thể sử dụng `Burp Sequencer`. Thực hiện gửi request mà khởi tạo session tới `Burp Sequencer` bằng cách right-click và chọn `Send to Sequencer` (ví dụ login request). Tại tab sequencer, tích chọn `Token Location Within Response` và `Cookie`. Chúng ta cũng có thể chỉ định custom location, tùy thuộc vào vị trí mà server trả về token. Sau đó thực hiện ấn `start the live capture`. Lúc này burp sẽ thực hiện gửi 1 lượng lớn request và captures token để analyze. Chờ cho đến khi Burp thu thập được tối thiểu 1000 token , ta click vào nút `Analyze` để có được kết quả.

## Prevention

* Thực hiện hủy session varibles đúng cách nếu không dùng đến nữa. Ví dụ:

```php
<SNIP>
// logout
if(isset($_POST['logout'])) {
session_start(); // Khởi tạo hoặc tiếp tục session hiện tại
session_unset(); // xóa toàn bộ session varibles của session hiện tại
session_destroy(); // hủy dữ liệu session tương ứng trên server
}
```

* Không nên tái sử dụng session variables trong những process khác nhau, nên chia tách biệt. Ví dụ:

```php
// reset password
$_SESSION['reset']['verified'] = true;
$_SESSION['reset']['step'] = 3;

// change email
$_SESSION['change_email']['verified'] = false;
$_SESSION['change_email']['step'] = 1;
```
* Chỉ thực hiện khởi tạo, lưu data cho session variables khi đã hoàn tất process hoặc có được kết quả của process đấy
* Sử dụng session token có độ dài và độ ngẫu nhiên đủ lớn

