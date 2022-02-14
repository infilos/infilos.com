---
type: docs 
title: "认证方法"
linkTitle: "认证方法"
weight: 1
---

{{% pageinfo color="primary" %}}
[原文连接：https://www.infoq.cn/article/xeirmzbscwxjoyc3hflv](https://www.infoq.cn/article/xeirmzbscwxjoyc3hflv)
{{% /pageinfo %}}

## 身份验证与授权

身份验证（Authentication）是具备权限的系统验证尝试访问系统的用户或设备所用凭据的过程。相比之下，授权（Authorization）是给定系统验证是否允许用户或设备在系统上执行某些任务的过程。

简单地说：

1. 身份验证：你是谁？
2. 授权：你能做什么？

身份验证先于授权。也就是说，用户必须先处于合法状态，然后才能根据其授权级别被授予对资源的访问权限。验证用户身份的最常见方法是用户名和密码的组合。用户通过身份验证后，系统将为他们分配不同的角色，例如管理员、主持人等，从而为他们授予一些特殊的系统权限。

接下来，我们来看一下用于用户身份验证的各种方法。

## HTTP 基本验证

HTTP 协议中内置的基本身份验证（Basic auth）是最基本的身份验证形式。使用它时，登录凭据随每个请求一起发送到请求标头中：

```
"Authorization: Basic dXNlcm5hbWU6cGFzc3dvcmQ=" your-website.com
```

这里的用户名和密码未加密，而是使用一个`:`符号将用户名和密码串联在一起，形成单个字符串：`username:password`，再使用 base64 编码这个字符串。

```
>>> import base64
>>>
>>> auth = "username:password"
>>> auth_bytes = auth.encode('ascii') # convert to bytes
>>> auth_bytes
b'username:password'
>>>
>>> encoded = base64.b64encode(auth_bytes) # base64 encode
>>> encoded
b'dXNlcm5hbWU6cGFzc3dvcmQ='
>>> base64.b64decode(encoded) # base64 decode
b'username:password'
```

这种方法是无状态的，因此客户端必须为每个请求提供凭据。它适用于 API 调用以及不需要持久会话的简单身份验证工作流。

### 流程

1. 未经身份验证的客户端请求受限制的资源
2. 返回的 HTTP401Unauthorized 带有标头`WWW-Authenticate`，其值为 Basic。
3. `WWW-Authenticate:Basic`标头使浏览器显示用户名和密码输入框
4. 输入你的凭据后，它们随每个请求一起发送到标头中：`Authorization: Basic dcdvcmQ=`

<div><img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20220214215049.png" style="display:block;margin-left:auto;margin-right:auto;width:60%;" alt="20220214215049" /></div>

### 优点

- 由于执行的操作不多，因此使用该方法可以快速完成身份验证。
- 易于实现。
- 所有主要浏览器均支持。

### 缺点

- Base64 不是加密。这只是表示数据的另一种方式。由于 base64 编码的字符串以纯文本格式发送，因此可以轻松解码。这么差的安全性很容易招致多种类型的攻击。因此，HTTPS/SSL 是绝对必要的。
- 凭据必须随每个请求一起发送。
- 只能使用无效的凭据重写凭据来注销用户。

### 代码

使用 Flask-HTTP 包，可以轻松地在 Flask 中完成基本的 HTTP 身份验证。

```python
from flask import Flask
from flask_httpauth import HTTPBasicAuth
from werkzeug.security import generate_password_hash, check_password_hash

app = Flask(__name__)
auth = HTTPBasicAuth()

users = {
    "username": generate_password_hash("password"),
}


@auth.verify_password
def verify_password(username, password):
    if username in users and check_password_hash(users.get("username"), password):
        return username


@app.route("/")
@auth.login_required
def index():
    return f"You have successfully logged in, {auth.current_user()}"


if __name__ == "__main__":
    app.run()
```

## HTTP 摘要验证

HTTP Digest Auth（或 Digest Access Auth）是 HTTP 基本验证的一种更安全的形式。主要区别在于 HTTP 摘要验证的密码是以 MD5 哈希形式代替纯文本形式发送的，因此它比基本身份验证更安全。

### 流程

1. 未经身份验证的客户端请求受限制的资源
2. 服务器生成一个随机值（称为随机数，nonce），并发回一个 HTTP 401 未验证状态，带有一个`WWW-Authenticate`标头（其值为`Digest`）以及随机数：`WWW-Authenticate:Digestnonce="44f0437004157342f50f935906ad46fc"`
3. `WWW-Authenticate:Basic`标头使浏览器显示用户名和密码输入框
4. 输入你的凭据后，系统将对密码进行哈希处理，然后与每个请求的随机数一起在标头中发送：`Authorization: Digest username="username",` `nonce="16e30069e45a7f47b4e2606aeeb7ab62", response="89549b93e13d438cd0946c6d93321c52"`
5. 服务器使用用户名获取密码，将其与随机数一起哈希，然后验证哈希是否相同

<div><img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20220214215134.png" style="display:block;margin-left:auto;margin-right:auto;width:60%;" alt="20220214215134" /></div>

### 优点

- 由于密码不是以纯文本形式发送的，因此比基本身份验证更安全。
- 易于实现。
- 所有主要浏览器均支持。

### 缺点

- 凭据必须随每个请求一起发送。
- 只能使用无效的凭据重写凭据来注销用户。
- 与基本身份验证相比，由于无法使用 bcrypt，因此密码在服务器上的安全性较低。
- 容易受到中间人攻击。

### 代码

Flask-HTTP 包也支持摘要 HTTP 验证。

```python
from flask import Flask
from flask_httpauth import HTTPDigestAuth

app = Flask(__name__)
app.config["SECRET_KEY"] = "change me"
auth = HTTPDigestAuth()

users = {
    "username": "password"
}


@auth.get_password
def get_user(username):
    if username in users:
        return users.get(username)


@app.route("/")
@auth.login_required
def index():
    return f"You have successfully logged in, {auth.current_user()}"


if __name__ == "__main__":
    app.run()
```

## 基于会话的验证

使用基于会话的身份验证（或称会话 cookie 验证、基于 cookie 的验证）时，用户状态存储在服务器上。它不需要用户在每个请求中提供用户名或密码，而是在登录后由服务器验证凭据。如果凭据有效，它将生成一个会话，并将其存储在一个会话存储中，然后将其会话 ID 发送回浏览器。浏览器将这个会话 ID 存储为 cookie，该 cookie 可以在向服务器发出请求时随时发送。

基于会话的身份验证是有状态的。每次客户端请求服务器时，服务器必须将会话放在内存中，以便将会话 ID 绑定到关联的用户。

### 流程

<div><img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20220214215255.png" style="display:block;margin-left:auto;margin-right:auto;width:60%;" alt="20220214215255" /></div>

### 优点

- 后续登录速度更快，因为不需要凭据。
- 改善用户体验。
- 相当容易实现。许多框架（例如 Django）都是开箱即用的。

### 缺点

- 它是有状态的。服务器要在服务端跟踪每个会话。用于存储用户会话信息的会话存储需要在多个服务之间共享以启用身份验证。因此，由于 REST 是无状态协议，它不适用于 RESTful 服务。
- 即使不需要验证，Cookie 也会随每个请求一起发送
- 易受 CSRF 攻击。在[这里](https://testdriven.io/blog/csrf-flask/)阅读更多关于 CSRF 以及如何在 Flask 中防御它的信息。

### 代码

Flask-Login 非常适合基于会话的身份验证。这个包负责登录和注销，并可以在一段时间内记住用户。

```python
from flask import Flask, request
from flask_login import (
    LoginManager,
    UserMixin,
    current_user,
    login_required,
    login_user,
)
from werkzeug.security import generate_password_hash, check_password_hash


app = Flask(__name__)
app.config.update(
    SECRET_KEY="change_this_key",
)

login_manager = LoginManager()
login_manager.init_app(app)


users = {
    "username": generate_password_hash("password"),
}


class User(UserMixin):
    ...


@login_manager.user_loader
def user_loader(username: str):
    if username in users:
        user_model = User()
        user_model.id = username
        return user_model
    return None


@app.route("/login", methods=["POST"])
def login_page():
    data = request.get_json()
    username = data.get("username")
    password = data.get("password")

    if username in users:
        if check_password_hash(users.get(username), password):
            user_model = User()
            user_model.id = username
            login_user(user_model)
        else:
            return "Wrong credentials"
    return "logged in"


@app.route("/")
@login_required
def protected():
    return f"Current user: {current_user.id}"


if __name__ == "__main__":
    app.run()
```

## 基于令牌的身份验证

这种方法使用令牌而不是 cookie 来验证用户。用户使用有效的凭据验证身份，服务器返回签名的令牌。这个令牌可用于后续请求。

最常用的令牌是 JSON Web Token（JWT）。JWT 包含三个部分：

- 标头（包括令牌类型和使用的哈希算法）
- 负载（包括声明，是关于主题的陈述）
- 签名（用于验证消息在此过程中未被更改）

这三部分都是 base64 编码的，并使用一个`.`串联并做哈希。由于它们已编码，因此任何人都可以解码和读取消息。但是，只有验证的用户才能生成有效的签名令牌。令牌使用签名来验证，签名用的是一个私钥。

> JSON Web Token（JWT）是一种紧凑的、URL 安全的方法，用于表示要在两方之间转移的声明。JWT 中的声明被编码为一个 JSON 对象，用作一个 JSON Web Signature（JWS）结构的负载，或一个 JSON Web Encryption（JWE）结构的纯文本，从而使声明可以进行数字签名，或使用一个消息验证码 Message Authentication Code（MAC）来做完整性保护和/或加密。——IETF

令牌不必保存在服务端。只需使用它们的签名即可验证它们。近年来，由于 RESTfulAPI 和单页应用（SPA）的出现，令牌的使用量有所增加。

### 流程

<div><img src="https://infi-img.oss-cn-hangzhou.aliyuncs.com/img/20220214215353.png" style="display:block;margin-left:auto;margin-right:auto;width:60%;" alt="20220214215353" /></div>

### 优点

- 它是无状态的。服务器不需要存储令牌，因为可以使用签名对其进行验证。由于不需要数据库查找，因此可以让请求更快。
- 适用于微服务架构，其中有多个服务需要验证。我们只需在每一端配置如何处理令牌和令牌密钥即可。

### 缺点

- 根据令牌在客户端上的保存方式，它可能导致 XSS（通过 localStorage）或 CSRF（通过 cookie）攻击。
- 令牌无法被删除。它们只能过期。这意味着如果令牌泄漏，则攻击者可以滥用令牌直到其到期。因此，将令牌过期时间设置为非常小的值（例如 15 分钟）是非常重要的。
- 需要设置令牌刷新以在到期时自动发行令牌。
- 删除令牌的一种方法是创建一个将令牌列入黑名单的数据库。这为微服务架构增加了额外的开销并引入了状态。

### 代码

Flask-JWT-Extended 包为处理 JWT 提供了很多可能性。

```python
from flask import Flask, request, jsonify
from flask_jwt_extended import (
    JWTManager,
    jwt_required,
    create_access_token,
    get_jwt_identity,
)
from werkzeug.security import check_password_hash, generate_password_hash

app = Flask(__name__)
app.config.update(
    JWT_SECRET_KEY="please_change_this",
)

jwt = JWTManager(app)

users = {
    "username": generate_password_hash("password"),
}


@app.route("/login", methods=["POST"])
def login_page():
    username = request.json.get("username")
    password = request.json.get("password")

    if username in users:
        if check_password_hash(users.get(username), password):
            access_token = create_access_token(identity=username)
            return jsonify(access_token=access_token), 200

    return "Wrong credentials", 400


@app.route("/")
@jwt_required
def protected():
    return jsonify(logged_in_as=get_jwt_identity()), 200


if __name__ == "__main__":
    app.run()
```

## 一次性密码

一次性密码（One Time Password，OTP）通常用作身份验证的确认。OTP 是随机生成的代码，可用于验证用户是否是他们声称的身份。它通常用在启用双因素身份验证的应用中，在用户凭据确认后使用。

要使用 OTP，必须存在一个受信任的系统。这个受信任的系统可以是经过验证的电子邮件或手机号码。

现代 OTP 是无状态的。可以使用多种方法来验证它们。尽管有几种不同类型的 OTP，但基于时间的 OTP（TOTP）可以说是最常见的类型。它们生成后会在一段时间后过期。

由于 OTP 让你获得了额外的一层安全保护，因此建议将 OTP 用于涉及高度敏感数据的应用，例如在线银行和其他金融服务。

### 流程

实现 OTP 的传统方式：

- 客户端发送用户名和密码
- 经过凭据验证后，服务器会生成一个随机代码，将其存储在服务端，然后将代码发送到受信任的系统
- 用户在受信任的系统上获取代码，然后在 Web 应用上重新输入它
- 服务器对照存储的代码验证输入的代码，并相应地授予访问权限

TOTP 如何工作：

- 客户端发送用户名和密码
- 经过凭据验证后，服务器会使用随机生成的种子生成随机代码，并将种子存储在服务端，然后将代码发送到受信任的系统
- 用户在受信任的系统上获取代码，然后将其输入回 Web 应用
- 服务器使用存储的种子验证代码，确保其未过期，并相应地授予访问权限

谷歌身份验证器、微软身份验证器和 FreeOTP 等 OTP 代理如何工作：

- 注册双因素身份验证（2FA）后，服务器会生成一个随机种子值，并将该种子以唯一 QR 码的形式发送给用户
- 用户使用其 2FA 应用程序扫描 QR 码以验证受信任的设备
- 每当需要 OTP 时，用户都会在其设备上检查代码，然后在 Web 应用中输入该代码
- 服务器验证代码并相应地授予访问权限

### 优点

- 添加了一层额外的保护
- 不会有被盗密码在实现 OTP 的多个站点或服务上通过验证的危险

### 缺点

- 你需要存储用于生成 OTP 的种子。
- 像谷歌验证器这样的 OTP 代理中，如果你丢失了恢复代码，则很难再次设置 OTP 代理
- 当受信任的设备不可用时（电池耗尽，网络错误等）会出现问题。因此通常需要一个备用设备，这个设备会引入一个额外的攻击媒介。

### 代码

PyOTP 包提供了基于时间和基于计数器的 OTP。

```python
from time import sleep

import pyotp

if __name__ == "__main__":
    otp = pyotp.TOTP(pyotp.random_base32())
    code = otp.now()
    print(f"OTP generated: {code}")
    print(f"Verify OTP: {otp.verify(code)}")
    sleep(30)
    print(f"Verify after 30s: {otp.verify(code)}")
```

示例：

```
OTP generated: 474771
Verify OTP: True
Verify after 30s: False
```

## OAuth 和 OpenID

OAuth/OAuth2 和 OpenID 分别是授权和身份验证的流行形式。它们用于实现社交登录，一种单点登录（SSO）形式。社交登录使用来自诸如 Facebook、Twitter 或谷歌等社交网络服务的现有信息登录到第三方网站，而不是创建一个专用于该网站的新登录帐户。

当你需要高度安全的身份验证时，可以使用这种身份验证和授权方法。这些提供者中有一些拥有足够的资源来增强身份验证能力。利用经过反复考验的身份验证系统，可以让你的应用程序更加安全。

这种方法通常与基于会话的身份验证结合使用。

### 流程

你访问的网站需要登录。你转到登录页面，然后看到一个名为“使用谷歌登录”的按钮。单击该按钮，它将带你到谷歌登录页面。通过身份验证后，你将被重定向回自动登录的网站。这是使用 OpenID 进行身份验证的示例。它让你可以使用现有帐户（通过一个 OpenID 提供程序）进行身份验证，而无需创建新帐户。

最著名的 OpenID 提供方有谷歌、Facebook、Twitter 和 GitHub。

登录后，你可以转到网站上的下载服务，该服务可让你直接将大文件下载到谷歌云端硬盘。网站如何访问你的 Google 云端硬盘？这里就会用到 OAuth。你可以授予访问另一个网站上资源的权限。在这里，你授予的就是写入谷歌云端硬盘的访问权限。

### 优点

- 提高安全性。
- 由于无需创建和记住用户名或密码，因此登录流程更加轻松快捷。
- 如果发生安全漏洞，由于身份验证是无密码的，因此不会对第三方造成损害。

### 缺点

- 现在，你的应用程序依赖于你无法控制的另一个应用。如果 OpenID 系统关闭，则用户将无法登录。
- 人们通常倾向于忽略 OAuth 应用程序请求的权限。
- 在你配置的 OpenID 提供方上没有帐户的用户将无法访问你的应用程序。最好的方法是同时实现多种途径。例如用户名和密码以及 OpenID，并让用户自行选择。

### 代码

你可以使用 Flask-Dance 实现 GitHub 社交身份验证。

```python
from flask import Flask, url_for, redirect
from flask_dance.contrib.github import make_github_blueprint, github

app = Flask(__name__)
app.secret_key = "change me"
app.config["GITHUB_OAUTH_CLIENT_ID"] = "1aaf1bf583d5e425dc8b"
app.config["GITHUB_OAUTH_CLIENT_SECRET"] = "dee0c5bc7e0acfb71791b21ca459c008be992d7c"

github_blueprint = make_github_blueprint()
app.register_blueprint(github_blueprint, url_prefix="/login")


@app.route("/")
def index():
    if not github.authorized:
        return redirect(url_for("github.login"))
    resp = github.get("/user")
    assert resp.ok
    return f"You have successfully logged in, {resp.json()['login']}"


if __name__ == "__main__":
    app.run()
```

## 总结

在本文中，我们研究了许多不同的 Web 身份验证方法，它们都有各自的优缺点。

你什么时候应该使用哪种方法？具体情况要具体分析。一些基本的经验法则：

1. 对于利用服务端模板的 Web 应用程序，通过用户名和密码进行基于会话的身份验证通常是最合适的。你也可以添加 OAuth 和 OpenID。
2. 对于 RESTful API，建议使用基于令牌的身份验证，因为它是无状态的。
3. 如果必须处理高度敏感的数据，则你可能需要将 OTP 添加到身份验证流中。

最后请记住，本文的示例仅仅是简单的演示。生产环境需要进一步的配置。