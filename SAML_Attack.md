# SAML

SAML là giao thức được sử dụng để trao đổi thông tin xác thực và ủy quyền giữa 2 bên, thường được sử để triển khai SSO. SAML gồm các thành phần chính sau:

* **Identity Provider (IdP)**: nơi xác thực người dùng và cung cấp issues XML assertions
* **Service Provider (SP)**: nơi mà cung cấp dịch vụ, tài nguyên cho user (web server). Nó dựa vào XML assertions để xác thực
* **XML assertions**: XML based data chứa ttin về user

## SAML Flow

Khi người dùng click vào đăng nhập qua SSO:

```GET /sso/login 
Host: <SP_server>
```

**Bước 1**: Người dùng truy cập SP

Hệ thống thấy người dùng chưa có phiên đăng nhập hợp lệ

**Bước 2**: SP redirect user tới IdP

SP redirect user tới IdP và gửi kèm yêu cầu xác thực SAML (`SAML Request`)

```http
HTTP/1.1 302 Found
...
Location: http://<IdP_server>/<endpoint...>?SAMLRequest=<base64+urlencode SAML request>
```

Example SAMLRequest:
```
fVNNj9owFLzzK1DukA/SbWNBJAr9QKIQEbqHXirHfhRLie36OV3239dxsru02sUHR3qemTdjv8
yRNrUmy9ae5QF+t4B2NHbr0tQSiT9cBK2RRFEUSCRtAIllpFx+25JkGhFtlFVM1cF/tNssigjG
CiV72ma9CPa7T9v9l83uJ884Te8ynqZJNKuqiMazE/2QZXcV0DR9H7/LkjiKq6yn3oNBp7MInG
ww6tUQW9hItFRaV4+SdBLNJkl2jCOSJCTNfvTUtQsrJLWefrZWkzBEVNOzrUIUja6hCxJ2WxIK
rsOy3Jdg/ggGU33WvUgx5P8oJBfy1+3YVQ9C8vV4LCbFvjz2Isun61gpiW0DZmjz/bB9dkYZ5d
A8eneUoXeQe/a8c0h8apO/gp6H14AXiiY753CzLlQt2KOvd+uzMg21bweJp7GvCD45eShpJWpg
4iSAB88yy7pWDysD1MIisKaFYBz+03wYN+B++FxyCxc7XqlGUyOwexO4UGaHkC9Br+Gr2k3SAU
75zWFjhHU4Vy7c50EZ3r0aMNf7aKgzr4wdLulV8d51eMN2Pno6vv6T8r8=
```
Decode
```xml
< AuthnRequest
samlp="urn:oasis:names:tc:SAML:2.0:protocol"
saml="urn:oasis:names:tc:SAML:2.0:assertion"
ID="ONELOGIN_d9da469d44203bb0a13fa8996bea4471592101b9"
Version="2.0"
IssueInstant="2024-03-29T10:22:49Z"
Destination="http://sso.htb/simplesaml/saml2/idp/SSOService.php"
ProtocolBinding="urn:oasis:names:tc:SAML:2.0:bindings:HTTP-POST"
AssertionConsumerServiceURL="http://academy.htb/acs.php">
< Issuer>http://academy.htb/</ Issuer>
< NameIDPolicy
Format="urn:oasis:names:tc:SAML:1.1:nameid-format:unspecified"
AllowCreate="true" />
< RequestedAuthnContext Comparison="exact">
< AuthnContextClassRef>urn:oasis:names:tc:SAML:2.0:ac:classes:Password
ProtectedTransport</ AuthnContextClassRef>
</ RequestedAuthnContext>
</ AuthnRequest>
```

SAML Request chứa 3 trường cần quan tâm:

* **Destination**: endpoint của IdP, nơi nhận vào SAML Request
* **AssertionConsumerServiceURL**: endpoint của SP mà IdP sẽ gọi về sau khi xác thực thành công và gửi kèm `XML assertion`
* **Isuer**: SAML request's issuer

**Bước 3**: IdP xác thực người dùng

IdP yêu cầu user đăng nhập nếu chưa có phiên đăng nhập hợp lệ, sau đó nó sẽ xác thực thông tin.

**Bước 4**: Assertion Generation

Nếu thông tin xác thực hợp lệ, IdP thực hiện khởi tạo 1 XML Assertion cho user đó.

Ví dụ 1 assertion được tạo:

```xml
<saml:Assertion
    ID="_abc123"
    IssueInstant="2026-11-01T10:00:00Z"
    Version="2.0">

    <saml:Issuer>
        https://sso.example.com
    </saml:Issuer>

    <saml:Subject>
        <saml:NameID>user@example.com</saml:NameID>
    </saml:Subject>

    <saml:Conditions
        NotBefore="2026-11-01T09:59:00Z"
        NotOnOrAfter="2026-11-01T10:05:00Z">

        <saml:AudienceRestriction>
            <saml:Audience>
                https://app.example.com
            </saml:Audience>
        </saml:AudienceRestriction>

    </saml:Conditions>

</saml:Assertion>
```

Assertion này sẽ chứa các thông tin về user và được ký số để bên nhận xác minh tính toàn vẹn cũng như nguồn phát hành của dữ liệu.

* **Issuer**: assertion's issuer
* **NameID**: unique identifier for user
* **Attribute**: chứa các thông tin về user như email, name, ...

Sau đó, `Assertion` này được đóng gói vào `Auth Respone` và trả về cho user. (ở trang trả về này sẽ chứa 1 đoạn mã để user tự động post assertion này tới `AssertionConsumerServiceURL`)

```http
POST /saml/acs HTTP/1.1
Host: <SP_server>
Content-Type: application/x-www-form-urlencoded

SAMLResponse=<...>&RelayState=original-state
```

**Bước 5**: Assertion Verification

SP nhận Auth response và thực hiện verifies SAML assertion

**Bước 6**: Resource access

Sau khi xác minh SAMLResponse thành công, SP trích xuất thông tin người dùng, tạo phiên đăng nhập (Session) và cho phép người dùng truy cập các dịch vụ, tài nguyên tương ứng với quyền được cấp.

# SAML Attack

## Signature Exclusion

Nếu SP misconfigured và skip signature verifcation nếu response không chứa signature XML element. Điều này sẽ cho phép attack manipulate toàn bộ SAML response.

Để thực hiện tấn công, ta chỉ cần lấy Auth Response, sau đó xóa đi toàn bộ `ds:Signature` và sửa thông tin mà ta muốn thay đổi và thực hiện gửi lại request tới `AssertionConsumerServiceURL` nằm trong Auth Request.

```http
POST /acs.php HTTP/1.1
Host: <SP_server>
...

SAMLResponse=...&RelayState=/acs.php
```

## Signature Wrapping Attack

### Lí thuyết

Như đã đề cập, SAML Assertion sẽ được kí để bảo vệ tính toàn vẹn của dữ liệu. Và `ds:Signature` node sẽ chứa một node con là `ds:Reference` -> tham chiếu tới assertion mà nó kí.

Có nhiều phương thức để kí:

* **envoloped**: phần tử `ds:Signature` nằm trong chính assertion được kí
```xml
<saml:Assertion ID="_abc123">
    <ds:Signature>
        <ds:SignedInfo>
            <ds:Reference URI="#_abc123"/>
        </ds:SignedInfo>
        <ds:SignatureValue>...</ds:SignatureValue>
    </ds:Signature>
    ...
</saml:Assertion>
```
* **enveloping**: phần tử được kí nằm trong signature
```
<ds:Signature>
    <ds:SignedInfo>
        <ds:Reference URI="#_abc123"/>
    </ds:SignedInfo>
    <ds:SignatureValue>...</ds:SignatureValue>
    <ds:Object>
        <saml:Assertion ID="_abc123">
            ...
        </saml:Assertion>
    </ds:Object>
</ds:Signature>
```
* **Detached**: phần tử signature và assertion nằm tách biệt
```
<samlp:Response>
    <ds:Signature>
        <ds:SignedInfo>
            <ds:Reference URI="#_abc123"/>
        </ds:SignedInfo>
        <ds:SignatureValue>...</ds:SignatureValue>
    </ds:Signature>

    <saml:Assertion ID="_abc123">
        ...
    </saml:Assertion>
</samlp:Response>
```
Ở bước 6, Assertion verification, SP thực hiện kiểm tra tính toàn vẹn của dữ liệu dựa trên signature này. Tuy nhiên, nếu xử lí không đúng cách ở bước sau (bước trích xuất thông tin nguời dùng) ứng dụng có thể dính lỗ hổng Signature Wrapping attack.

Cụ thể, ứng dụng bị dính lỗi trong trường hợp logic verification searches the SAML response for the `ds:Signature` node và element referenced in the `ds:Reference` element. Sau đó thực hiện verify mà không check thêm gì (chẳng hạn số lượng bản ghi của assertion chẳng hạn). Sau đó, nó thực hiện trích xuất thông tin xác thực từ bản ghi SAML assertion đầu tiên nó tìm thấy trong SAML response. Attack lúc này có thể chèn thêm 1 assertion vào phía trước node signature để giả mạo -> bypass.

Cách thực hiện: lấy ra 1 assertion mẫu, thực hiện thay đổi gịá trị của trường `ID` trong assertion. Đồng thời thay đổi thông tin theo ý muốn của mình. Chèn nó vào node đầu tiên ngay sau node root. Sau đó thực hiện gửi lại request tương tự như `Signature Exclusion` ở trên.

## Other SAML Vulnerability

Vì nó xử lí dữ liệu XML. Nên có thể dẫn đến XXE hoặc XSLT Injection. Để test ta lấy SAML Response sau đó thêm payload để test rồi gửi lại request tương tự như trên.

# Tool: BurpSute -> Extension > BApp Store > SAML Raider

# Prevention

* Sử dụng phiên bản mới nhất của những thư viện uy tín cho tất cả các bước liên quan đến SAML như verification hay extraction 




