# LDAP Injection

LDAP is protocol used to access directory servers such as AD. Web applications may use LDAP for integration with AD or other directory service for authentication or data retrieval.

## LDAP terminology

```
# Kiến trúc tổng quan DS

                     [ Directory Server (DS) ]
                                |
             +------------------+------------------+

             |                                     |
  [ ou=users,dc=htb,dc=com ]             [ ou=groups,dc=htb,dc=com ]
             |
             +---------> [ LDAP Entry: uid=kaylie,ou=users,dc=htb,dc=com ]
```

```
# Kiến trúc chi tiết của 1 LDAP entry

┌────────────────────────────────────────────────────────────────────────┐
│  DISTINGUISHED NAME (DN)                                               │
│  dn: uid=kaylie , ou=users , dc=hackthebox , dc=com                    │
│      └── RDN 1 ┘   └── RDN 2 ┘   └── RDN 3 ───┘   └── RDN 4 ┘          │
├────────────────────────────────────────────────────────────────────────┤
│  OBJECT CLASSES                                                        │
│  - objectClass: top                                                    │
│  - objectClass: person                                                 │
│  - objectClass: inetOrgPerson                                          │
├────────────────────────────────────────────────────────────────────────┤
│  ATTRIBUTES                                                            │
│  - uid: kaylie                                                         │
│  - mail: kaylie@hackthebox.com                                         │
│  - userPassword: {SSHA}secret...                                       │
└────────────────────────────────────────────────────────────────────────┘
```
* **Directory Server (DS):** Hệ quản trị cơ sở dữ liệu phân cấp dạng cây.
* **LDAP Entry:** Một bản ghi dữ liệu duy nhất trong cây thư mục.
* **Distinguished Name (DN):** Định danh duy nhất (Khóa chính tuyệt đối) của một Entry.
* **Relative Distinguished Name (RDN):** Các thành phần cấu tạo nên DN, phân tách bằng dấu phẩy.
* **Object Classes:** Khuôn mẫu định nghĩa cấu trúc dữ liệu và các thuộc tính bắt buộc của Entry.
* **Attributes:** Cặp thuộc tính dữ liệu (`Key: Value`) lưu trữ thông tin thực tế bên trong Entry.

### LDAP Operations: hệ thống LDAP định nghĩa các thao tác (hành động) sau từ phía client:

* **Bind Operation** (Xác thực hệ thống): Thao tác xác thực tài khoản. Client cung cấp một `DN` cụ thể và mật khẩu tương ứng. Thay vì quét tuần tự toàn bộ hệ thống, Directory Server (DS) sử dụng cấu trúc phân cấp để di chuyển trực tiếp đến Entry sở hữu `DN` đó và đối chiếu giá trị trong thuộc tính mật khẩu (`userPassword`). Nếu khớp, kết nối được cấp quyền hoạt động.

* **Search Operation** (Tìm kiếm bản ghi): Thao tác truy vấn dữ liệu. Client gửi yêu cầu tìm kiếm gồm ba thành phần: vị trí bắt đầu quét (`Search Base`), phạm vi quét (`Scope`), và bộ lọc (`Search Filter`). DS sẽ đối chiếu bộ lọc này với các **Attributes** (thuộc tính dữ liệu), **Object Classes** (lớp đối tượng), hoặc chính thông tin **DN/RDN** của các Entry trong phạm vi chỉ định thông qua hệ thống bảng chỉ mục (index) để trả về kết quả phù hợp.

* **Unbind Operation** (Ngắt kết nối an toàn)
* **Add Operation** (Tạo Entry mới)
* **Delete Operation** (Xóa Entry)
* **Modify Operation** (Chỉnh sửa thuộc tính của Entry)

### LDAP Search Filter Syntax

Search filter có thể bao gồm nhiều components, và mỗi component phải được đặt trong ().
Component example: (attribute=value)

* **Toán tử &**: thỏa mãn tất cả điều kiện bên trong - (&(name=abc)(title=hi)(id=111))
* **Toán tử |**: thỏa mãn 1 trong các điều kiện bên trong (|(name=abc)(title=hi))
* **Toán tử !**: tìm các entry không thỏa mãn đkien bên trong (!(name=abc))
* **Wildcard**: 
```
(name=*) - khớp với tất cả entry có attribute name
(name=*admin*) - khớp với tất cả entry có attribute name chứa từ khóa admin
```

Đặc biệt: có thể biểu diễn giá trị boolean bằng các phép toán rỗng

    * (&) - True
    * (|) - False
    
Common Attribute Types:
| Attribute Type | Description |
|----------------|------------|
|cn | Full Name|
|givenName| First name|
|sn | Last name |
| uid | User ID |
| objectClass | Object type |
| distinguishedName | Distinguished Name|
| ou | Organizational Unit |
| title | Title of a Person |
| telephoneNumber | Phone Number |
| description | Description |
| mail | Email Address |
| street | Address |
| postalCode | Zip code|
| member | Group Memberships |
| userPassword | User password |

## Exploit

Example vulnerable code:
```php
// ldap connection
const LDAP_HOST = "localhost";
const LDAP_PORT = 389;
const LDAP_DC = "dc=example,dc=htb";
const LDAP_DN = "cn=ldapuser,dc=example,dc=htb";
const LDAP_PASS = "ldappassword";

// connect to server
$conn = ldap_connect(LDAP_HOST, LDAP_PORT);
if (!$conn) {
    exit('LDAP connection failed');
}

// bind operation
ldap_set_option($conn, LDAP_OPT_PROTOCOL_VERSION, 3);
$bind = ldap_bind($conn, LDAP_DN, LDAP_PASS);
if (!$bind) {
    exit('LDAP bind failed');
}

// search operation
$filter = '(&(cn=' . $_POST['username'] . ')(userPassword=' . $_POST['password'] . '))';
$search = ldap_search($conn, LDAP_DC, $filter);
$entries = ldap_get_entries($conn, $search);
if ($entries['count'] > 0) {
    // successful login
    <SNIP>
} else {
    // login failed
    <SNIP>
}
```

```ldap
# Authentication Bypass
username = * và password = * -> trả về tất cả entry -> login với user đầu tiên
username = *admin* và password = * -> login với username đầu tiên có chứa từ khóa admin trong name
username = admin)(|(& và password = xxx) -> bypass without *, nma cần biết username

Data Exfiltration
nếu server có về bản ghi trực tiếp -> sử dụng wildcard để exfiltrate

Blind boolean-based: cần 1 tín hiệu từ server để làm điều kiện
Ví dụ: (&(uid=htb-stdnt)(password=<your_input>))
input = a* -> check kí tự đầu tiên của password có phải a không 
input = pa* -> input=pas* -> ...
hoặc muốn exfiltrate 1 attribute khác:
input = *)(description=a*
hoặc tìm 1 attribute name có tồn tại không:
input = *)(description=*
```

## Prevention

Escape special characters:

* ( -> \28
* ) -> \29
* \* -> \2a
* \\ -> \5c
* null byte -> \00

Trong php có sẵn hàm `ldap_escape` thực hiện nhiệm vụ này

### Best pratices:

* Account used to bind to the DS nên có quyền thấp nhất có thể
* **Anonymous Binds** là cơ chế cho phép Client kết nối và thực hiện các thao tác (như Tìm kiếm) trên DS mà không cần tài khoản/mật khẩu. Cấu hình này **phải bị vô hiệu hóa hoàn toàn**, bắt buộc Client phải thực hiện `Bind Operation` hợp lệ trước nhằm ngăn chặn kẻ tấn công ẩn danh dò quét (Enumerate) toàn bộ sơ đồ tổ chức, danh sách nhân sự và dữ liệu nhạy cảm.

* authentication bằng cách dùng bind để không cần dùng đến search filter

```php
...
$dn = "cn=" . ldap_escape($_POST['username'], "", LDAP_ESCAPE_DN) . ",dc=example,dc=htb";
$pw = $_POST['password'];
...
$bind = ldap_bind($conn, $dn, $pw);
...
```
