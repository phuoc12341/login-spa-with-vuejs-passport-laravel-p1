# login-spa-with-vuejs-passport-laravel-p1

Chào các bạn hôm nay mình sẽ giới thiệu với các bạn cách làm chức năng Login theo hướng SPA sử dụng **Vuejs + Passport** của Laravel.
Luồng xử lý bài toán của mình như sau:
1. Có 1 trang Login thông thường, sau khi user nhập username + password và submit, thì sẽ gửi api thông qua **axios** đến Server
2. Server nhận request đầu vào và xác thực thông tin, nếu định danh được user thì sẽ sẽ trả về 1 **JWT thông qua Passport của Laravel**, nếu có error hoặc validate message thì trả về cho user
3. User nhận response nếu có lỗi thì hiển thị, nếu success thì **lưu lại JWT vào localStorage** sau đó render ra 1 page khác ví dụ Dashboard
4. Nếu sau này user vào trang Login, check xem trong localStorage có biến JWT hay không, nếu có thì redirect sang Dashboard luôn, nếu ko có JWT thì sẽ render ra trang Login và đăng nhập như bình thường
5. Nếu user muốn truy cập đến trang nào check xem user đã có token chưa, nếu có thì gửi kèm JWT vào api request. Bên server ta mình cũng đặt 1 Middleware để check JWT thì mới truy cập được vào route này

Vì có khá nhiều phần nên mình chia làm 2 phần:
Phần 1: Sẽ thực hiện Setup project và thực hiện đến bước User lấy được JWT trong response trả về rồi lưu vào localStorage
Phần 2: Mình sẽ thực hiện nốt các yêu cầu còn lại

[Link pull request: https://github.com/phuoc12341/login-spa/pull/2](https://github.com/phuoc12341/login-spa/pull/2)
***

# 1. Setup Passport cho Laravel
Đầu tiên ta cần setup 1 project Laravel và sau đó cài đặt Passport cho Laravel
```
composer require laravel/passport
```
Tiếp đến ta chạy migration để tạo ra các bảng dùng cho việc lưu các client và token
```
php artisan migrate
```
Sau đó tạo ra các key mã hóa dùng để tạo ra access token
```
php artisan passport:install
```
Ngoài ra câu lệnh trên cũng tạo sẵn cho mình 2 client là: `personal access` và `password grant` luôn
Sau khi chạy xong, vào trong Model/User thêm trait `Laravel\Passport\HasApiTokens`. Trait này cung cấp các function() để ta có thể kiểm tra **token và scope** của User đã được xác thực sau này
```
<?php

namespace App;

use Illuminate\Foundation\Auth\User as Authenticatable;
use Illuminate\Notifications\Notifiable;
use Laravel\Passport\HasApiTokens;

class User extends Authenticatable
{
    use HasApiTokens, Notifiable;
}
```
Tiếp đến ta sẽ gọi `Passport::routes` trong hàm boot() của AuthServiceProvider để cấp hay thu hồi các **access token** hoặc là các **client**
```
<?php

namespace App\Providers;

use Illuminate\Foundation\Support\Providers\AuthServiceProvider as ServiceProvider;
use Illuminate\Support\Facades\Gate;
use Laravel\Passport\Passport;

class AuthServiceProvider extends ServiceProvider
{
    /**
     * The policy mappings for the application.
     *
     * @var array
     */
    protected $policies = [
        'App\Model' => 'App\Policies\ModelPolicy',
    ];

    /**
     * Register any authentication / authorization services.
     *
     * @return void
     */
    public function boot()
    {
        $this->registerPolicies();

        Passport::routes();
    }
}
```
Cuối cùng trong file cấu hình `config/auth.php` ta sẽ thay đổi lại phần driver của api thành `passport`. Điều này sẽ hướng dẫn ứng dụng sử dụng **TokenGuard của Passport** khi xác thực 1 API request đến
```
'guards' => [
    'web' => [
        'driver' => 'session',
        'provider' => 'users',
    ],

    'api' => [
        'driver' => 'passport',
        'provider' => 'users',
    ],
],
```

Ở đây ta sẽ sử dụng grant là **Personal Access Tokens** cho đơn giản.
Đầu tiền muốn có được access token ta phải có được 1 client. Lệnh `passport:install` ở trên ta đã tạo được rồi, nếu như vẫn chưa có thì ta chạy lệnh sau để tạo 1 client theo **grant personal**

```
php artisan passport:client --personal
```
Sau khi có 1 client thì ta có thể **sử dụng client này để issue 1 access token**

# 2. Setup Vuejs
Trước tiên, ta cần cài **Vuejs + axios** thông qua npm:
```
npm install vue
npm install axios
```
Tiếp đến ta khởi tạo 1 **LoginComponent** đơn giản như sau:
```
<template>
    <div>
        <form autocomplete="off" @submit="checkForm" method="post">
            <p v-if="errors.length">
                <b>Please correct the following error(s):</b>
                <ul>
                    <li v-for="error in errors">{{ error }}</li>
                </ul>
            </p>
            <div class="form-group">
                <label for="email">E-mail</label>
                <input type="" id="email" class="form-control" placeholder="user@example.com" v-model="email">
            </div>
            <div class="form-group">
                <label for="password">Password</label>
                <input type="password" id="password" class="form-control" v-model="password">
            </div>
            <button type="submit" class="btn btn-default">Sign in</button>
        </form>
    </div>
</template>

<script>
    export default {
        mounted() {
            console.log('Component mounted.')
        },
        data(){
            return {
                email: null,
                password: null,
                errors: []
            }
        },
        methods: {
            checkForm: function (e) {
            if (this.email && this.password) {
                axios.post('/api/v1/login', {
                    email: this.email,
                    password: this.password
                })
                .then(function (response) {
                    console.log(response);
                    localStorage.setItem('token', response.data.data['token'])
                })
                .catch(function (error) {
                    console.log(error);
                });
            }
            this.errors = [];
            if (!this.email) {
                this.errors.push('Name required.');
            }
            if (!this.password) {
                this.password.push('Age required.');
            }
            e.preventDefault();
            }
        }
    }
</script>
```
Như ở trên có thể thấy, sau khi user submit ta call 1 api `api/v1/login`, như vậy trong file `routes/api.php` ta thêm 1 route như sau:
```
Route::post('login', 'LoginController@login')->name('login');
```
Và như vậy trong Logincotnroller của ta sẽ như sau:
```
    public function login(LoginRequest $request)
    {
        $params = $request->all();
        $loginData = [
            'username' => $params['email'],
            'password' => $params['password'],
        ];

        if ($this->authService->attempt($request)) {
            $user = Auth::user();
            $token = $user->createToken('mytoken')->accessToken;
        }

        return new AuthResource($token, __FUNCTION__);
    }
```
Như vậy ở trên ta nếu user xác thực thành công thì ta đã issue ra 1 access token thông qua Passport và trả về cho user.
Và ở phía user cũng đã nhận được **access token** này, sau đó ở phía client sẽ lưu token này vào **localStorage**

Sau đó ta sẽ rediect user sang trang Dashboard ... phần này thì sang phần 2 mình sẽ triển khai nốt nhé ^.^

[Link pull request: https://github.com/phuoc12341/login-spa/pull/2](https://github.com/phuoc12341/login-spa/pull/2)

