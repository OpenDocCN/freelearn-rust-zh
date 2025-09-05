# *第十一章*：安全性和添加 API 及 JSON

网络应用的两个最重要的方面是认证和授权。在本章中，我们将学习如何实现简单的认证和授权系统。在创建了这些系统之后，我们将学习如何创建一个简单的**应用程序编程接口**（**API**）以及如何使用**JSON Web Token**（**JWT**）保护 API 端点。

在本章结束时，你将能够创建一个认证系统，包括登录和登出以及为已登录用户设置访问权限的功能。你还将能够创建一个 API 服务器，并了解如何保护 API 端点。

在本章中，我们将涵盖以下主要主题：

+   用户认证

+   用户授权

+   处理 JSON

+   使用 JWT 保护 API

# 技术要求

对于本章，我们有通常的要求：一个 Rust 编译器、一个文本编辑器、一个网络浏览器和一个 PostgreSQL 数据库服务器，以及 FFmpeg 命令行。在本章中，我们将学习关于 JSON 和 API 的知识。安装 cURL 或其他任何 HTTP 测试客户端。

你可以在[`github.com/PacktPublishing/Rust-Web-Development-with-Rocket/tree/main/Chapter11`](https://github.com/PacktPublishing/Rust-Web-Development-with-Rocket/tree/main/Chapter11)找到本章的源代码。

# 用户认证

网络应用中最常见的任务之一是处理注册和登录。通过登录，用户可以向网络服务器表明他们确实是他们所说的那个人。

当我们为用户模型实现 CRUD 时，我们已经创建了一个注册系统。现在，让我们使用现有的用户模型来实现登录系统。

登录的想法很简单：用户可以填写他们的用户名和密码。然后，应用程序会验证用户名和密码是否有效。之后，应用程序可以生成包含用户信息的 cookie 并将其返回给网络浏览器。每当浏览器有请求时，cookie 都会从浏览器发送回服务器，我们验证 cookie 的内容。

为了确保我们不必为每个请求实现 cookie，如果我们使用请求保护器在路由处理函数中自动验证 cookie，我们可以创建一个请求保护器。

让我们按照以下步骤开始实现用户登录系统：

1.  创建一个请求保护器来处理用户认证 cookie。如果我们想在路由处理函数中添加新的请求保护器，我们可以将请求保护器组织在同一个地方，这样会更方便。在`src/lib.rs`中添加一个新的模块：

    ```rs
    pub mod guards;
    ```

1.  然后，创建一个名为`src/guards`的文件夹。在`src/guards`内部，添加一个名为`src/guards/mod.rs`的文件。在这个新文件中添加一个新的模块：

    ```rs
    pub mod auth;
    ```

1.  然后，创建一个名为`src/guards/auth.rs`的新文件。

1.  创建一个结构体来处理用户认证 cookie。让我们将这个结构体命名为`CurrentUser`。在`src/guards/auth.rs`中，添加一个结构体来存储`User`信息：

    ```rs
    use crate::fairings::db::DBConnection;
    use crate::models::user::User;
    use rocket::http::Status;
    use rocket::request::{FromRequest, Outcome, Request};
    use rocket::serde::Serialize;
    use rocket_db_pools::{sqlx::Acquire, Connection};
    #[derive(Serialize)]
    pub struct CurrentUser {
        pub user: User,
    }
    ```

1.  定义一个常量，该常量将用作 cookie 的键来存储用户的**全局唯一标识符**（**UUID**）：

    ```rs
    pub const LOGIN_COOKIE_NAME: &str = "user_uuid";
    ```

1.  为`CurrentUser`实现`FromRequest`特质，使结构体成为一个请求守卫。添加以下实现框架：

    ```rs
    #[rocket::async_trait]
    impl<'r> FromRequest<'r> for CurrentUser {
        type Error = ();
        async fn from_request(req: &'r Request<'_>) -> 
        Outcome<Self, Self::Error> {
        }
    }
    ```

1.  在`from_request`函数内部，定义一个错误，如果发生错误则返回：

    ```rs
    let error = Outcome::Failure((Status::Unauthorized, ()));
    ```

1.  从请求中获取 cookie，并按如下方式提取 cookie 中的 UUID：

    ```rs
    let parsed_cookie = req.cookies().get_private(LOGIN_COOKIE_NAME);
    if parsed_cookie.is_none() {
        return error;
    }
    let cookie = parsed_cookie.unwrap();
    let uuid = cookie.value();
    ```

1.  我们想要获取数据库的连接以查找用户信息。我们可以在请求守卫实现内部获取另一个请求守卫（例如`Connection<DBConnection>`）。添加以下行：

    ```rs
    let parsed_db = req.guard::<Connection<DBConnection>>().await;
    if !parsed_db.is_success() {
        return error;
    }
    let mut db = parsed_db.unwrap();
    let parsed_connection = db.acquire().await;
    if parsed_connection.is_err() {
        return error;
    }
    let connection = parsed_connection.unwrap();
    ```

1.  查找并返回用户。添加以下行：

    ```rs
    let found_user = User::find(connection, uuid).await;
    if found_user.is_err() {
        return error;
    }
    let user = found_user.unwrap();
    Outcome::Success(CurrentUser { user })
    ```

1.  接下来，我们想要实现登录本身。我们将创建一个`like sessions/new`路由来获取登录页面，一个`sessions/create`路由来发送登录的用户名和密码，以及一个`sessions/delete`路由用于登出。在实现这些路由之前，让我们为登录创建一个模板。在`src/views`中添加一个名为`sessions`的新文件夹。然后，创建一个名为`src/views/sessions/new.html.tera`的文件。将以下行追加到该文件中：

    ```rs
    {% extends "template" %}
    {% block body %}
      <form accept-charset="UTF-8" action="login" 
      autocomplete="off" method="POST">
        <input type="hidden" name="authenticity_token" 
        value="{{ csrf_token }}"/>
        <fieldset>
          <legend>Login</legend>
          <div class="row">
            <div class="col-sm-12 col-md-3">
              <label for="username">Username:</label>
            </div>
            <div class="col-sm-12 col-md">
              <input name="username" type="text" value="" 
              />
            </div>
          </div>
          <div class="row">
            <div class="col-sm-12 col-md-3">
              <label for="password">Password:</label>
            </div>
            <div class="col-sm-12 col-md">
              <input name="password" type="password" />
            </div>
          </div>
          <button type="submit" value="Submit">
          Submit</button>
        </fieldset>
      </form>
    {% endblock %}
    ```

1.  在`src/models/user.rs`中，添加一个用于登录信息的结构体：

    ```rs
    #[derive(FromForm)]
    pub struct Login<'r> {
        pub username: &'r str,
        pub password: &'r str,
        pub authenticity_token: &'r str,
    }
    ```

1.  在同一文件中，我们想要为`User`结构体创建一个方法，以便能够根据登录用户名信息从数据库中查找用户，并验证登录密码是否正确。在通过`update`方法验证密码正确后，是时候重构了。创建一个新的函数来验证密码：

    ```rs
    fn verify_password(ag: &Argon2, reference: &str, password: &str) -> Result<(), OurError> {
        let reference_hash = PasswordHash::new(
        reference).map_err(|e| {
            OurError::new_internal_server_error(
            String::from("Input error"), Some(
            Box::new(e)))
        })?;
        Ok(ag
            .verify_password(password.as_bytes(), 
            &reference_hash)
            .map_err(|e| {
                OurError::new_internal_server_error(
                    String::from("Cannot verify 
                    password"),
                    Some(Box::new(e)),
                )
            })?)
    }
    ```

1.  将`update`方法从以下行更改：

    ```rs
    let old_password_hash = PasswordHash::new(&old_user.password_hash).map_err(|e| {
        OurError::new_internal_server_error(
        String::from("Input error"), Some(Box::new(e)))
    })?;
    let argon2 = Argon2::default();
    argon2
        .verify_password(user.old_password.as_bytes(), 
        &old_password_hash)
        .map_err(|e| {
            OurError::new_internal_server_error(
                String::from("Cannot confirm old 
                password"),
                Some(Box::new(e)),
            )
        })?;
    ```

然后，将其更改为以下行：

```rs
let argon2 = Argon2::default();
verify_password(&argon2, &old_user.password_hash, user.old_password)?;
```

1.  创建一个基于登录用户名查找用户的方法。在`impl User`块中，添加以下方法：

    ```rs
    pub async fn find_by_login<'r>(
        connection: &mut PgConnection,
        login: &'r Login<'r>,
    ) -> Result<Self, OurError> {
        let query_str = "SELECT * FROM users WHERE 
        username = $1";
        let user = sqlx::query_as::<_, Self>(query_str)
            .bind(&login.username)
            .fetch_one(connection)
            .await
            .map_err(OurError::from_sqlx_error)?;
        let argon2 = Argon2::default();
        verify_password(&argon2, &user.password_hash, 
        &login.password)?;
        Ok(user)
    }
    ```

1.  现在，实现处理登录的路由。在`src/routes/mod.rs`中创建一个新的`mod`：

    ```rs
    pub mod session;
    ```

然后，创建一个名为`src/routes/session.rs`的新文件。

1.  在`src/routes/session.rs`中，创建一个名为`new`的路由处理函数。我们希望该函数为我们之前创建的登录模板提供渲染后的模板。添加以下行：

    ```rs
    use super::HtmlResponse;
    use crate::fairings::csrf::Token as CsrfToken;
    use rocket::request::FlashMessage;
    use rocket_dyn_templates::{context, Template};
    #[get("/login", format = "text/html")]
    pub async fn new<'r>(flash: Option<FlashMessage<'_>>, csrf_token: CsrfToken) -> HtmlResponse {
        let flash_string = flash
            .map(|fl| format!("{}", fl.message()))
            .unwrap_or_else(|| "".to_string());
        let context = context! {
            flash: flash_string,
            csrf_token: csrf_token,
        };
        Ok(Template::render("sessions/new", context))
    }
    ```

1.  然后，创建一个名为`create`的新函数。在这个函数中，我们想要找到用户并验证与数据库中的密码散列匹配的密码。如果一切顺利，设置包含用户信息的 cookie。追加以下行：

    ```rs
    use crate::fairings::db::DBConnection;
    use crate::guards::auth::LOGIN_COOKIE_NAME;
    use crate::models::user::{Login, User};
    use rocket::form::{Contextual, Form};
    use rocket::http::{Cookie, CookieJar};
    use rocket::response::{Flash, Redirect};
    use rocket_db_pools::{sqlx::Acquire, Connection};
    ...
    #[post("/login", format = "application/x-www-form-urlencoded", data = "<login_context>")]
    pub async fn create<'r>(
        mut db: Connection<DBConnection>,
        login_context: Form<Contextual<'r, Login<'r>>>,
        csrf_token: CsrfToken,
        cookies: &CookieJar<'_>,
    ) -> Result<Flash<Redirect>, Flash<Redirect>> {
        let login_error = || Flash::error(
        Redirect::to("/login"), "Cannot login");
        if login_context.value.is_none() {
            return Err(login_error());
        }
        let login = login_context.value.as_ref().unwrap();
        csrf_token
            .verify(&login.authenticity_token)
            .map_err(|_| login_error())?;
        let connection = db.acquire().await.map_err(|_| 
        login_error())?;
        let user = User::find_by_login(connection, login)
            .await
            .map_err(|_| login_error())?;
        cookies.add_private(Cookie::new(LOGIN_COOKIE_NAME, 
        user.uuid.to_string()));
        Ok(Flash::success(Redirect::to("/users"), "Login 
        successfully"))
    }
    ```

1.  最后，创建一个名为`delete`的函数。我们将使用此函数作为登出路由。追加以下行：

    ```rs
    #[post("/logout", format = "application/x-www-form-urlencoded")]
    pub async fn delete(cookies: &CookieJar<'_>) -> Flash<Redirect> {
        cookies.remove_private(
        Cookie::named(LOGIN_COOKIE_NAME));
        Flash::success(Redirect::to("/users"), "Logout 
        successfully")
    }
    ```

1.  将`session::new`、`session::create`和`session::delete`添加到`src/main.rs`中：

    ```rs
    use our_application::routes::{self, post, session, user};
    ...
    async fn rocket() -> Rocket<Build> {
        ...
        routes![
        ...
            session::new,
            session::create,
            session::delete,
        ]
        ...
    }
    ```

1.  现在，我们可以使用`CurrentUser`来确保只有登录用户才能访问我们应用程序中的一些端点。在`src/routes/user.rs`中，删除`edit`端点中查找用户的例程。删除以下行：

    ```rs
    pub async fn edit_user(
        mut db: Connection<DBConnection>,
        ...
    ) -> HtmlResponse {
        let connection = db
            .acquire()
            .await
            .map_err(|_| Status::InternalServerError)?;
    let user = User::find(connection, 
        uuid).await.map_err(|e| e.status)?;
        ...
    }
    ```

1.  然后，将`CurrentUser`添加到需要登录用户的路由中，如下所示：

    ```rs
    use crate::guards::auth::CurrentUser;
    ...
    pub async fn edit_user(...
        current_user: CurrentUser,
    ) -> HtmlResponse {
        ...
        let context = context! {
            form_url: format!("/users/{}", uuid),
            ...
            user: &current_user.user,
            current_user: &current_user,
            ...
        };
        ...
    }
    ...
    pub async fn update_user<'r>(...
        current_user: CurrentUser,
    ) -> Result<Flash<Redirect>, Flash<Redirect>> {
        ...
        match user_value.method {
            "PUT" => put_user(db, uuid, user_context, 
            csrf_token, current_user).await,
            "PATCH" => patch_user(db, uuid, user_context, 
            csrf_token, current_user).await,
            ...
        }
    }
    ...
    pub async fn put_user<'r>(...
        _current_user: CurrentUser,
    ) -> Result<Flash<Redirect>, Flash<Redirect>> {...}
    ...
    pub async fn patch_user<'r>(...
        current_user: CurrentUser,
    ) -> Result<Flash<Redirect>, Flash<Redirect>> {
        put_user(db, uuid, user_context, csrf_token, 
        current_user).await
    }
    ...
    pub async fn delete_user_entry_point(...
        current_user: CurrentUser,
    ) -> Result<Flash<Redirect>, Flash<Redirect>> {
        delete_user(db, uuid, current_user).await
    }
    ...
    pub async fn delete_user(...
        _current_user: CurrentUser,
    ) -> Result<Flash<Redirect>, Flash<Redirect>> {...}
    ```

1.  最后，在`src/routes/post.rs`的端点上也进行保护。只有已登录用户可以上传和删除帖子，因此将代码修改为以下内容：

    ```rs
    crate::guards::auth::CurrentUser;
    ...
    pub async fn create_post<'r>(...
        _current_user: CurrentUser,
    ) -> Result<Flash<Redirect>, Flash<Redirect>> {...}
    ...
    pub async fn delete_post(...
        _current_user: CurrentUser,
    ) -> Result<Flash<Redirect>, Flash<Redirect>> {...}
    ```

在我们实现认证之前，我们可以编辑和删除任何用户或帖子。现在尝试在不登录的情况下编辑或删除某些内容。然后，尝试登录并删除和编辑。

仍然存在一个问题：登录后，用户可以编辑和删除其他用户的资料。我们将在下一节中通过实现授权来学习如何防止这个问题。

# 授权用户

认证和授权是信息安全中的两个主要概念。如果认证是一种证明实体是其所称实体的方式，那么授权就是一种赋予实体权利的方式。一个实体可能能够修改某些资源，一个实体可能能够修改所有资源，一个实体可能只能查看有限资源，等等。

在上一节中，我们实现了登录和`CurrentUser`等认证概念；现在是时候实现授权了。想法是确保已登录用户只能修改他们自己的信息和帖子。

请记住，这个例子非常简单。在更高级的信息安全中，有更高级的概念，例如基于角色的访问控制。例如，我们可以创建一个名为`admin`的角色，我们可以将某个用户设置为`admin`，而`admin`可以无限制地做任何事情。

让我们按照以下步骤尝试实现简单的授权：

1.  为`CurrentUser`添加一个简单的方法来比较其实例与 UUID。在`src/guards/auth.rs`中追加以下行：

    ```rs
    impl CurrentUser {
        pub fn is(&self, uuid: &str) -> bool {
            self.user.uuid.to_string() == uuid
        }
        pub fn is_not(&self, uuid: &str) -> bool {
            !self.is(uuid)
        }
    }
    ```

1.  添加一种新的错误类型。在`src/errors/our_error.rs`文件中的`impl OurError {}`块中添加一个`new`方法：

    ```rs
    pub fn new_unauthorized_error(debug: Option<Box<dyn Error>>) -> Self {
        Self::new_error_with_status(Status::Unauthorized, 
        String::from("unauthorized"), debug)
    }
    ```

1.  我们可以在模板中检查`CurrentUser`实例来控制应用程序的流程。例如，如果没有`CurrentUser`实例，我们显示注册和登录的链接。如果有`CurrentUser`实例，我们显示注销的链接。让我们修改 Tera 模板。编辑`src/views/template.html.tera`文件并追加以下行：

    ```rs
    <body>
      <header>
        <a href="/users" class="button">Home</a>
        {% if current_user %}
    <form accept-charset="UTF-8" action="/logout" 
          autocomplete="off" method="POST" id="logout"  
          class="hidden"></form>
          <button type="submit" value="Submit" form="
          logout">Logout</button>
        {% else %}
          <a href="/login" class="button">Login</a>
          <a href="/users/new" class="button">Signup</a>
        {% endif %}
      </header>
      <div class="container">
    ```

1.  编辑`src/views/users/index.html.tera`文件并删除以下行：

    ```rs
    <a href="/users/new" class="button">New user</a>
    ```

找到这一行：

```rs
<a href="/users/edit/{{ user.uuid }}" class="button">Edit User</a>
```

将其修改为以下行：

```rs
{% if current_user and current_user.user.uuid == user.uuid %}
    <a href="/users/edit/{{user.uuid}}" class="
    button">Edit User</a>
{% endif %}
```

1.  编辑`src/views/users/show.html.tera`文件并找到这些行：

    ```rs
    <a href="/users/edit/{{user.uuid}}" class="button">Edit User</a>
    <form accept-charset="UTF-8" action="/users/delete/{{user.uuid}}" autocomplete="off" method="POST" id="deleteUser" class="hidden"></form>
    <button type="submit" value="Submit" form="deleteUser">Delete</button>
    ```

然后，用以下条件检查包围这些行：

```rs
{% if current_user and current_user.user.uuid == user.uuid %}
     <a href... 
    ... 
    </button>
{% endif %}
```

1.  接下来，我们希望只允许已登录用户上传。在`src/views/posts/index.html.tera`文件中找到表单行：

    ```rs
    <form action="/users/{{ user.uuid }}/posts" enctype="multipart/form-data" method="POST">
    ...
    </form>
    ```

用以下条件包围表单行：

```rs
{% if current_user %}
     <form action="/users/{{ user.uuid }}/posts" enctype="multipart/form-data" method="POST"> 
    ... 
    </form>
{% endif %}
```

1.  现在对模板进行最后的修改。我们希望只有帖子的所有者才能删除帖子。在`src/views/posts/show.html.tera`文件中找到这些行：

    ```rs
    <form accept-charset="UTF-8" action="/users/{{user.uuid}}/posts/delete/{{post.uuid}}" autocomplete="off" method="POST" id="deletePost" class="hidden"></form>
    <button type="submit" value="Submit" form="deletePost">Delete</button>
    ```

用以下行包围它们：

```rs
{% if current_user and current_user.user.uuid == user.uuid %}
    <form... 
    ... 
    </button>
{% endif %}
```

1.  修改路由处理函数以获取 `current_user` 的值。记住，我们可以将请求保护器包装在 `Option` 中，例如 `Option<CurrentUser>`。当一个路由处理函数无法获取 `CurrentUser` 实例（例如，没有已登录的用户）时，它将生成 `Option` 的 `None` 变体。然后我们可以将实例传递给模板。

让我们从 `src/routes/post.rs` 开始转换路由处理函数。按照以下方式修改 `get_post()` 函数：

```rs
pub async fn get_post(...
    current_user: Option<CurrentUser>,
) -> HtmlResponse {
    ...
    let context = context! {user, current_user, post: 
    &(post.to_show_post())};
    Ok(Template::render("posts/show", context))
}
```

1.  让我们对 `get_posts()` 函数做同样的事情。按照以下方式修改函数：

    ```rs
    pub async fn get_posts(...
        current_user: Option<CurrentUser>,
    ) -> HtmlResponse {
        let context = context! {
            ...
            current_user,
        };
        Ok(Template::render("posts/index", context))
    }
    ```

1.  为了确保 `create_post()` 函数的安全性，我们可以检查上传文件的用户的 UUID 是否与 URL 上的 `user_uuid` 相同。这个检查是为了防止已登录的攻击者篡改请求并发送虚假请求。在我们进行文件操作之前，将检查放入 `create_post()` 函数中，如下所示：

    ```rs
    pub async fn create_post<'r>(...
        current_user: CurrentUser,
    ) -> Result<Flash<Redirect>, Flash<Redirect>> {
        ...
        if current_user.is_not(user_uuid) {
            return Err(create_err());
        }
        ...
    }
    ```

1.  我们可以在 `src/routes/post.rs` 中的 `delete_post()` 函数进行相同的检查。我们希望阻止未经授权的用户发送篡改的请求并删除他人的帖子。按照以下方式修改 `delete_post()` 函数：

    ```rs
    pub async fn delete_post(...
        current_user: CurrentUser,
    ) -> Result<Flash<Redirect>, Flash<Redirect>> {
        ...
        if current_user.is_not(user_uuid) {
            return Err(delete_err());
        }
        ...
    }
    ```

1.  尝试重新启动应用程序，登录，并查看您是否可以删除他人的帖子。还尝试通过应用相同的原理修改 `src/routes/user.rs`：获取 `CurrentUser` 实例并应用必要的检查，或将 `CurrentUser` 实例传递给模板。您可以在 [`github.com/PacktPublishing/Rust-Web-Development-with-Rocket/tree/main/Chapter11/02Authorization`](https://github.com/PacktPublishing/Rust-Web-Development-with-Rocket/tree/main/Chapter11/02Authorization) 找到完整的代码，包括保护用户相关路由。

1.  网络服务器最常见的任务之一是提供 API，并且一些 API 必须防止不受欢迎的使用。在下一节中，我们将学习如何提供 API 并保护 API 端点。

# 处理 JSON

网络应用程序的常见任务之一是处理 API。API 可以返回很多不同的格式，但现代 API 已经收敛为两种常见的格式：JSON 和 XML。

在 Rocket 网络框架中，构建返回 JSON 的端点相当简单。对于处理 JSON 格式的请求体，我们可以使用 `rocket::serde::json::Json<T>` 作为数据保护器。泛型 `T` 类型必须实现 `serde::Deserialize` 特性，否则 Rust 编译器将拒绝编译。

对于响应，我们可以通过返回 `rocket::serde::json::Json<T>` 来做同样的事情。泛型 `T` 类型在用作响应时必须仅实现 `serde::Serialize` 特性。

让我们看看如何处理 JSON 请求和响应的示例。我们希望创建一个单独的 API 端点 `/api/users`。此端点可以接收类似于 `our_application::models::pagination::Pagination` 结构的 JSON 主体，如下所示：

```rs
{"next":"2022-02-22T22:22:22.222222Z","limit":10}
```

按照以下步骤实现 API 端点：

1.  为 `OurError` 实现 `serde::Serialize`。将这些行添加到 `src/errors/our_error.rs`：

    ```rs
    use rocket::serde::{Serialize, Serializer};
    use serde::ser::SerializeStruct;
    ...
    impl Serialize for OurError {
        fn serialize<S>(&self, serializer: S) -> 
        Result<S::Ok, S::Error>
        where
            S: Serializer,
        {
            let mut state = serializer.
            serialize_struct("OurError", 2)?;
            state.serialize_field("status", &self
            .status.code)?;
            state.serialize_field("message", &self
            .message)?;
            state.end()
        }
    }
    ```

1.  我们希望 `Pagination` 继承 `Deserialize` 并自动实现 `Deserialize` 特性，因为 `Pagination` 将会在 JSON 数据保护器 `Json<Pagination>` 中使用。由于 `Pagination` 包含 `OurDateTime` 成员，因此 `OurDateTime` 也必须实现 `Deserialize` 特性。修改 `src/models/our_date_time.rs` 并添加 `Deserialize` derive 宏：

    ```rs
    use rocket::serde::{Deserialize, Serialize};
    ...
    #[derive(Debug, sqlx::Type, Clone, Serialize, Deserialize)]
    #[sqlx(transparent)]
    pub struct OurDateTime(pub DateTime<Utc>);
    ```

1.  为 `Pagination` 实现 `Serialize` 和 `Deserialize`。我们还想实现 `Serialize`，因为我们想将 `Pagination` 作为 `/api/users` 端点的响应的一部分使用。按照以下方式修改 `src/models/pagination.rs`：

    ```rs
    use rocket::serde::{Deserialize, Serialize};
    ...
    #[derive(FromForm, Serialize, Deserialize)]
    pub struct Pagination {...}
    ```

1.  对于 `User` 结构体，它已经自动继承了 `Serialize`，因此我们可以在 `User` 的向量中使用它。需要修复的一个问题是，我们不希望密码包含在生成的 JSON 中。Serde 有许多宏可以控制如何从结构体生成序列化数据。添加一个宏来跳过 `password_hash` 字段。修改 `src/models/user.rs`：

    ```rs
    pub struct User {
        ...
        #[serde(skip_serializing)]
        pub password_hash: String,
        ...
    }
    ```

1.  我们希望返回 `User` 和 `Pagination` 的向量作为生成的 JSON。我们可以创建一个新的结构体来将这些封装在一个字段中。在 `src/models/user.rs` 中添加以下行：

    ```rs
    #[derive(Serialize)]
    pub struct UsersWrapper {
        pub users: Vec<User>,
        #[serde(skip_serializing_if = "Option::is_none")]
        #[serde(default)]
        pub pagination: Option<Pagination>,
    }
    ```

注意，如果 `pagination` 字段为 `None`，我们将跳过它。

1.  在 `src/routes/mod.rs` 中添加一个新的模块：

    ```rs
    pub mod api;
    ```

然后，在 `src/routes/api.rs` 中创建一个新文件。

1.  在 `src/routes/api.rs` 中添加通常的 `use` 声明、模型、错误和数据库连接：

    ```rs
    use crate::errors::our_error::OurError;
    use crate::fairings::db::DBConnection;
    use crate::models::{
        pagination::Pagination,
        user::{User, UsersWrapper},
    };
    use rocket_db_pools::Connection;
    ```

1.  添加 `use` 声明 `rocket::serde::json::Json`：

    ```rs
    use rocket::serde::json::Json;
    ```

1.  添加一个处理函数定义以获取用户：

    ```rs
    #[get("/users", format = "json", data = "<pagination>")]
    pub async fn users(
        mut db: Connection<DBConnection>,
        pagination: Option<Json<Pagination>>,
    ) -> Result<Json<UsersWrapper>, Json<OurError>> {}
    ```

1.  实现函数。在函数中，我们可以使用 `into_inner()` 方法获取 JSON 的内容，如下所示：

    ```rs
    let parsed_pagination = pagination.map(|p| p.into_inner());
    ```

1.  查找用户。添加以下行：

    ```rs
    let (users, new_pagination) = User::find_all(&mut db, parsed_pagination)
        .await
        .map_err(|_| OurError::new_internal_server_
        error(String::from("Internal Error"), None))?;
    ```

由于我们已经为 `OurError` 实现了 `Serialize` 特性，我们可以自动返回该类型。

1.  现在，是时候返回 `UsersWrapper`。添加以下行：

    ```rs
    Ok(Json(UsersWrapper {
        users,
        pagination: new_pagination,
    }))
    ```

1.  最后要做的事情是将路由添加到 `src/main.rs`：

    ```rs
    use our_application::routes::{self, api, post, session, user};
    ...
    .mount("/", ...)
    .mount("/assets", FileServer::from(relative!("static")))
    .mount("/api", routes![api::users])
    ```

1.  尝试运行应用程序并向 `http://127.0.0.1:8000/api/users` 发送请求。我们可以使用任何 HTTP 客户端，但如果使用 cURL，它将如下所示：

    ```rs
    curl -X GET -H "Content-Type: application/json" -d "{\"next\":\"2022-02-22T22:22:22.222222Z\",\"limit\":1}" http://127.0.0.1:8000/api/users
    ```

应用程序应该返回类似以下输出的内容：

```rs
{"users":[{"uuid":"8faa59d6-1079-424a-8eb9-09ceef1969c8","username":"example","email":"example@example.com","description":"example","status":"Inactive","created_at":"2021-11-06T06:09:09.534864Z","updated_at":"2021-11-06T06:09:09.534864Z"}],"pagination":{"next":"2021-11-06T06:09:09.534864Z","limit":1}}
```

现在我们已经完成了 API 端点的创建，接下来让我们在下一节尝试保护这个端点。

# 使用 JWT 保护 API

我们想要执行的一个常见任务是保护 API 端点免受未经授权的访问。有许多原因需要保护 API 端点，例如保护敏感数据、进行金融服务或提供订阅服务。

在网络浏览器中，我们可以通过创建会话、分配一个 cookie 给会话并将会话返回给网络浏览器来保护服务器端点，但 API 客户端不总是网络浏览器。API 客户端可以是移动应用程序、其他 Web 应用程序、硬件监控器等等。这引发了一个问题，*我们如何保护 API 端点？*

保护 API 端点有很多方法，但一个行业标准是使用 JWT。根据*IETF RFC7519*，JWT 是一种紧凑、URL 安全的表示声明的方式，用于在双方之间传输。JWT 中的声明可以是 JSON 对象或这些 JSON 对象的特殊纯文本表示。

使用 JWT 的一个流程如下：

1.  客户端向服务器发送认证请求。

1.  服务器响应 JWT。

1.  客户端存储 JWT。

1.  客户端使用存储的 JWT 发送 API 请求。

1.  服务器验证 JWT 并根据情况响应。

让我们尝试按照以下步骤实现 API 端点保护：

1.  在`Cargo.toml`依赖项部分添加所需的库：

    ```rs
    hmac = "0.12.1"
    jwt = "0.16.0"
    sha2 = "0.10.2"
    ```

1.  我们想要使用一个秘密令牌来签名 JWT 令牌。在`Rocket.toml`中添加一个新的条目如下：

    ```rs
    jwt_secret = "fill with your own secret"
    ```

1.  添加一个新的状态来存储令牌的秘密。当应用程序创建或验证 JWT 时，我们想要检索这个秘密。在`src/states/mod.rs`中添加以下行：

    ```rs
    pub struct JWToken {
        pub secret: String,
    }
    ```

1.  修改`src/main.rs`以使应用程序从配置中检索秘密并管理状态：

    ```rs
    use our_application::states::JWToken;
    ...
    struct Config {...
        jwt_secret: String,
    }
    ...
    async fn rocket() -> Rocket<Build> {
        ...
        let config: Config = our_rocket...
        let jwt_secret = JWToken {
            secret: String::from(config.jwt_
            secret.clone()),
        };
        let final_rocket = our_rocket.manage(jwt_secret);
        ...
        final_rocket
    }
    ```

1.  创建一个结构体来存储用于认证发送的 JSON 数据，另一个结构体来存储包含要返回给客户端的令牌的 JSON 数据。在`src/models/user.rs`中添加以下`use`声明：

    ```rs
    use rocket::serde::{Deserialize, Serialize};
    ```

添加以下结构体：

```rs
#[derive(Deserialize)]
pub struct JWTLogin<'r> {
    pub username: &'r str,
    pub password: &'r str,
}
#[derive(Serialize)]
pub struct Auth {
    pub token: String,
}
```

1.  为`JWTLogin`实现一个验证用户名和密码的方法。添加`impl`块和方法：

    ```rs
    impl<'r> JWTLogin<'r> {
        pub async fn authenticate(
            &self,
            connection: &mut PgConnection,
            secret: &'r str,
        ) -> Result<Auth, OurError> {}
    }
    ```

1.  在`authenticate()`方法内部，添加`error`闭包：

    ```rs
    let auth_error =
        || OurError::new_bad_request_error(
        String::from("Cannot verify password"), None);
    ```

1.  然后，根据用户名查找用户并验证密码：

    ```rs
    let user = User::find_by_login(
        connection,
        &Login {
            username: self.username,
            password: self.password,
            authenticity_token: "",
        },
    )
    .await
    .map_err(|_| auth_error())?;
    verify_password(&Argon2::default(), &user.password_hash, self.password)?;
    ```

1.  添加以下`use`声明：

    ```rs
    use hmac::{Hmac, Mac};
    use jwt::{SignWithKey};
    use sha2::Sha256;
    use std::collections::BTreeMap; 
    ```

在`authenticate`中继续以下操作以从用户的 UUID 生成令牌并返回令牌：

```rs
let user_uuid = &user.uuid.to_string();
let key: Hmac<Sha256> =
    Hmac::new_from_slice(secret.as_bytes()
    ).map_err(|_| auth_error())?;
let mut claims = BTreeMap::new();
claims.insert("user_uuid", user_uuid);
let token = claims.sign_with_key(&key).map_err(|_| auth_error())?;
Ok(Auth {
    token: token.as_str().to_string(),
})
```

1.  创建一个用于认证的函数。让我们称这个函数为`login()`。在`src/routes/api.rs`中添加所需的`use`声明：

    ```rs
    use crate::models::user::{Auth, JWTLogin, User, UsersWrapper};
    use crate::states::JWToken;
    use rocket::State;
    use rocket_db_pools::{sqlx::Acquire, Connection};
    ```

1.  然后，按照以下方式添加`login()`函数：

    ```rs
    #[post("/login", format = "json", data = "<jwt_login>")]
    pub async fn login<'r>(
        mut db: Connection<DBConnection>,
        jwt_login: Option<Json<JWTLogin<'r>>>,
        jwt_secret: &State<JWToken>,
    ) -> Result<Json<Auth>, Json<OurError>> {
        let connection = db
            .acquire()
            .await
            .map_err(|_| OurError::new_internal_server_
            error(String::from("Cannot login"), None))?;
        let parsed_jwt_login = jwt_login
            .map(|p| p.into_inner())
            .ok_or_else(|| OurError::new_bad_request_
            error(String::from("Cannot login"), None))?;
        Ok(Json(
            parsed_jwt_login
                .authenticate(connection, &jwt_secret
                .secret)
                .await
                .map_err(|_| OurError::new_internal_
                server_error(String::from("Cannot login"), 
                None))?,
        ))
    }
    ```

1.  现在我们已经创建了登录功能，下一步是创建一个请求保护器来处理请求头中的授权令牌。在`src/guards/auth.rs`中添加以下`use`声明：

    ```rs
    use crate::states::JWToken;
    use hmac::{Hmac, Mac};
    use jwt::{Header, Token, VerifyWithKey};
    use sha2::Sha256;
    use std::collections::BTreeMap;
    ```

1.  为请求保护器添加一个新的结构体`APIUser`：

    ```rs
    pub struct APIUser {
        pub user: User,
    }
    ```

1.  为`APIUser`实现`FromRequest`。添加以下代码块：

    ```rs
    #[rocket::async_trait]
    impl<'r> FromRequest<'r> for APIUser {
        type Error = ();
        async fn from_request(req: &'r Request<'_>) -> 
        Outcome<Self, Self::Error> {}
    }
    ```

1.  在`from_request()`中，添加返回错误的闭包：

    ```rs
    let error = || Outcome::Failure ((Status::Unauthorized, ()));
    ```

1.  从请求头中获取令牌：

    ```rs
    let parsed_header = req.headers().get_one("Authorization");
    if parsed_header.is_none() {
        return error();
    }
    let token_str = parsed_header.unwrap();
    ```

1.  从状态中获取秘密：

    ```rs
    let parsed_secret = req.rocket().state::<JWToken>();
    if parsed_secret.is_none() {
        return error();
    }
    let secret = &parsed_secret.unwrap().secret;
    ```

1.  验证令牌并获取用户的 UUID：

    ```rs
    let parsed_key: Result<Hmac<Sha256>, _> = Hmac::new_from_slice(secret.as_bytes());
    if parsed_key.is_err() {
        return error();
    }
    let key = parsed_key.unwrap();
    let parsed_token: Result<Token<Header, BTreeMap<String, String>, _>, _> = token_str.verify_with_key(&key);
    if parsed_token.is_err() {
        return error();
    }
    let token = parsed_token.unwrap();
    let claims = token.claims();
    let parsed_user_uuid = claims.get("user_uuid");
    if parsed_user_uuid.is_none() {
        return error();
    }
    let user_uuid = parsed_user_uuid.unwrap();
    ```

1.  查找用户并返回用户数据：

    ```rs
    let parsed_db = req.guard::<Connection<DBConnection>>().await;
    if !parsed_db.is_success() {
        return error();
    }
    let mut db = parsed_db.unwrap();
    let parsed_connection = db.acquire().await;
    if parsed_connection.is_err() {
        return error();
    }
    let connection = parsed_connection.unwrap();
    let found_user = User::find(connection, &user_uuid).await;
    if found_user.is_err() {
        return error();
    }
    let user = found_user.unwrap();
    Outcome::Success(APIUser { user })
    ```

1.  最后，在`src/routes/api.rs`中添加一个新的受保护 API 端点：

    ```rs
    use crate::guards::auth::APIUser;
    ...
    #[get("/protected_users", format = "json", data = "<pagination>")]
    pub async fn authenticated_users(
        db: Connection<DBConnection>,
        pagination: Option<Json<Pagination>>,
        _authorized_user: APIUser,
    ) -> Result<Json<UsersWrapper>, Json<OurError>> {
        users(db, pagination).await
    }
    ```

1.  在`src/main.rs`中添加到 Rocket 的路由：

    ```rs
    ...
    .mount("/api", routes![api::users, api::login, 
     api::authenticated_users])
    ...
    ```

现在，尝试访问新的端点。以下是一个使用 cURL 命令行的示例：

```rs
curl -X GET -H "Content-Type: application/json" \
 http://127.0.0.1:8000/api/protected_users
```

响应将会是一个错误。现在尝试发送一个请求来获取访问令牌。以下是一个示例：

```rs
curl -X POST -H "Content-Type: application/json" \
  -d "{\"username\":\"example\", \"password\": \"password\"}" \
 http://127.0.0.1:8000/api/login
```

如此示例所示，返回了一个令牌：

```rs
{"token":"eyJhbGciOiJIUzI1NiJ9.eyJ1c2VyX3V1aWQiOiJmMGMyZDM4Yy0zNjQ5LTRkOWQtYWQ4My0wZGE4ZmZlY2 E2MDgifQ.XJIaKlIfrBEUw_Ho2HTxd7hQkowTzHkx2q_xKy8HMKA"}
```

使用令牌发送请求，如本示例所示：

```rs
curl -X GET -H "Content-Type: application/json" \T -H "Content-Type: application/json" \
 -H "Authorization: eyJhbGciOiJIUzI1NiJ9.eyJ1c2VyX3V1aWQiOiJmMGMyZDM4Yy0zNjQ5LTRkOWQtYWQ4My0wZGE4ZmZlY2 E2MDgifQ.XJIaKlIfrBEUw_Ho2HTxd7hQkowTzHkx2q_xKy8HMKA" \
 http://127.0.0.1:8000/api/protected_users
```

然后，将返回正确的响应。JWT 是保护 API 端点的好方法，所以当需要时使用我们学到的技术。

# 摘要

在本章中，我们学习了如何验证用户并创建一个 cookie 来存储已登录用户的信息。我们还介绍了`CurrentUser`作为一个请求守卫，它在应用程序的某些部分充当授权。

在创建认证和授权系统之后，我们还学习了关于 API 端点的内容。我们将传入的请求体解析为 API 中的请求守卫，然后创建了一个 API 响应。

最后，我们了解了一些关于 JWT 以及如何用它来保护 API 端点的内容。

在下一章中，我们将学习如何测试我们创建的代码。
