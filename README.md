# Cấu hình pre-commit git

```bash
php artisan pre-commit:install
```

### Thêm tất cả các file đã thay đổi vào git stage và chạy thử:

```bash
php artisan pre-commit:check
```

# Quy tắc chung
## Các function phải có docblock và type của param truyền vào
Ví dụ:

```php
/**
 * Store User
 *
 * @param array $data
 * @return User
 */
public function store($data)
{
    return User::create($data);
}
```

```php
/**
 * @return \Illuminate\Database\Eloquent\Builder|\Illuminate\Database\Query\Builder
 */
public function makeNewQuery()
{
    return User::isActive();
}
```

```php
/**
 * Set the relationships that should be eager loaded.
 *
 * @param  string|array  $relations
 * @param  string|\Closure|null  $callback
 * @return $this
 */
public function with($relations, $callback = null)
{
    $this->currentQuery()->with(...func_get_args());

    return $this;
}
```

## Các thuộc tính cần phải có docblock
Ví dụ:

```php
/**
 * @var integer
 */
protected $perPage = 10;
```

```php
/**
 * @var \Illuminate\Database\Eloquent\Builder|\Illuminate\Database\Query\Builder
 */
protected $query;
```

# Luồng code, chi tiết các thành phần

route -> middleware -> request -> controller -> service -> controller -> resource -> return


## route

- Điều hướng request
- route phải được nhóm vào các nhóm gọn gàng
- Đặt tên route phải có ý nghĩa và không quá dài
- Route lấy dữ liệu sẽ có method là get, thay đổi dữ liệu method sẽ là post (create, update, delete)
- CURD sử dụng 5 tên api cơ bản (list, store, detail, update, destroy).

## middleware

- Xác nhận auth
- Chặn phân quyền

## request

- Xử lý validate

## controller

- Viết middleware vào `__construct()` của controller
- Controller sẽ xử lý lấy dữ liệu từ request để gửi vào Service

## service

- Xử lý logic.
- Query DB để lấy ra dữ liệu, cập nhật dữ liệu vào DB.
- Chú ý chỉ lấy ra những dữ liệu cần thiết.

## resource

- Transform dữ liệu trước khi trả về
- Riêng master data sẽ không có resource, phải transform dữ liệu trong service

# Quy tắc viết model

- Mỗi model sẽ tương ứng với 1 bảng
- Mỗi model sẽ có một trait scope tương ứng trong thư mục `Scopes`. Model sẽ `use` trait scope đó.
- Tất cả scope sẽ được viết vào file scope không được viết vào model.
- Thư mục Traits dùng để chứa những code được sử dụng trong nhiều model. Ví dụ: nhiều model có status giống nhau có thể viết vào traits.

# Quy tắc viết controller

- Một controller có thể thực hiện 1 chức năng hoặc nhiều chức năng liên quan đến nhau.
- Xử lý lấy params từ request trước khi truyền vào service ở controller.
- Hạn chế sử dụng `$request->all()`. Sử dụng `$request->only(['pr1', 'pr2'])`.
- Hạn chế truyền cả request vào service.
- Trong 1 controller chỉ gọi 1 service. Trong service sẽ gọi những service khác nếu cần thiết.

# Quy tắc viết service

- Tất cả các service phải được extends abstract class `Service`
- Khi thêm một service thì phải register vào `AppServiceProvider`

```php
$this->app->scoped(UserTableService::class, function ($app) {
    return new UserTableService();
});
```

- Khi sử dụng service thì phải gọi qua hàm static `getInstance()` để lấy đối tượng service để sử dụng.

# Cách sử dụng transaction

- Phải viết transaction khi tính năng thao tác create/update/delete tới nhiều bảng trong DB.
- Transaction phải tuân thủ cấu trúc bên dưới. Dưới `DB::commit()` chỉ được return dữ liệu. Không được gọi hàm khác.
- Nên viết transaction trong service (hàm được controller gọi).
- Không được gọi hàm có transaction trong một transaction khác. Điều này sẽ khiến transaction bị lồng nhau.
- Trong 1 transaction thứ tự update các bảng phải theo một luồng nhất định.

## Cấu trúc của một transaction

```php
try {
    DB::beginTransaction();

    // code here
    // make $data;

    DB::commit();
    return $data;
} catch (Exception $e) {
    DB::rollBack();
    throw $e;
}
```

hoặc

```php
$data = null;
try {
    DB::beginTransaction();

    // code here
    // modify $data;

    DB::commit();
} catch (Exception $e) {
    DB::rollBack();
    throw $e;
}
return $data;
```

## Trong 1 transaction thứ tự update các bảng phải theo một luồng nhất định

- Thứ tự bảng khi update hoặc delete phải giống thứ tự bảng khi create
- Khi sử dụng transaction phải ghi Thứ tự bảng trong transaction đó vào file `transaction-follow.md`.
    - Full Class name: tên class chứa `DB::beginTransaction()`. Tên phải là full name. Ví dụ: `App\Services\User\UserService`.
    - Method name: tên method chứa `DB::beginTransaction()`.
    - Nội dung là thứ tự bảng được sử dụng

Ví dụ: Một sản phẩm có nhiều ảnh.

```php
// Create product: thứ tự bảng products -> images
$product = Product::create(['name' => 'Sản phẩm 1']);
$product->images()->create(['ulr' => 'https://img.example/img.jpg']);

// Delete product: thứ tự bảng products -> images
Product::where('id', 1)->delete();
Image::where('product_id', 1)->delete();
```

# Query log
```php
DB::enableQueryLog();
$queries = DB::getQueryLog();
```
