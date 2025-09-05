# 第八章：*第八章*：服务静态资源和模板

网络应用程序的常见功能之一是服务静态文件，例如 **层叠样式表**（**CSS**）或 **JavaScript**（**JS**）文件。在本章中，我们将学习如何从 Rocket 应用程序中服务静态资源。

对于网络框架的一个常见任务是渲染模板到 HTML 文件。我们将学习如何使用 Tera 模板从 Rocket 应用程序中渲染 HTML。

在本章中，我们将涵盖以下主要主题：

+   服务静态资源

+   介绍 Tera 模板

+   展示用户

+   处理表单

+   保护 HTML 表单免受 CSRF 攻击

# 技术要求

对于本章，我们与上一章有相同的技术要求。我们需要一个 Rust 编译器、一个文本编辑器、一个 HTTP 客户端和一个 PostgreSQL 数据库服务器。

对于文本编辑器，您可以尝试添加一个支持 Tera 模板的扩展。如果没有 Tera 扩展，请尝试添加一个支持 Jinja2 或 Django 模板的扩展，并将文件关联设置为包含 `*.tera` 文件。

我们将向我们的应用程序添加 CSS，并使用来自 [`minicss.org/`](https://minicss.org/) 的样式表，因为它体积小且开源。请随意使用并修改示例 HTML 文件以使用其他样式表。

您可以在 [`github.com/PacktPublishing/Rust-Web-Development-with-Rocket/tree/main/Chapter08`](https://github.com/PacktPublishing/Rust-Web-Development-with-Rocket/tree/main/Chapter08) 找到本章的源代码。

# 服务静态资源

服务静态资源（如 HTML 文件、JS 文件或 CSS 文件）是网络应用程序的一个非常常见的任务。我们也可以让 Rocket 服务文件。让我们创建第一个服务 favicon 的函数。之前您可能已经注意到，一些网络浏览器从我们的服务器请求 favicon 文件，尽管我们在提供的 HTML 页面上没有明确提及它。让我们看看步骤：

1.  在应用程序根目录中，创建一个名为 `static` 的文件夹。在 `static` 文件夹内，添加一个名为 `favicon.png` 的文件。您可以在互联网上找到样本 `favicon.png` 文件，或者使用本章示例源代码中的文件。

1.  在 `src/main.rs` 中添加一个新的路由：

    ```rs
    routes![
        ...
        routes::favicon,
    ],
    ```

1.  在 `src/routes/mod.rs` 中添加一个新的路由处理函数来服务 `favicon.png`：

    ```rs
    use rocket::fs::{relative, NamedFile};
    use std::path::Path;
    ...
    #[get("/favicon.png")]
    pub async fn favicon() -> NamedFile {
        NamedFile::open(Path::new(relative!("static/
        favicon.png")))
            .await
            .ok()
            .unwrap()
    }
    ```

在这里，`relative!` 是一个宏，它生成一个相对于 crate 的路径版本。这意味着该宏指的是源文件或生成的二进制文件的文件夹。例如，我们在这个应用程序中的源文件位于 `/some/source`，通过说 `relative!("static/favicon.png")`，这意味着路径是 `/some/source/static/favicon.png`。

每次我们想要服务特定的文件时，我们都可以创建一个路由处理函数，返回 `NamedFile`，并将路由挂载到 Rocket 上。但显然，这种方法并不好；我们可以创建一个函数来动态返回静态文件。

1.  让我们重用我们在创建应用程序骨架时创建的`assets`函数。之前，在*第三章*，*Rocket 请求和响应*中，我们了解到我们可以在路由中使用多个段。我们可以利用这一点，为具有与请求多个段相同的文件名的文件提供服务。

删除我们之前创建的 favicon 功能，并从`src/main.rs`中移除对该功能的引用。在`src/routes/mod.rs`中，修改`use`声明：

```rs
use rocket::fs::{relative, NamedFile};
use std::path::{Path, PathBuf};
```

1.  如果应用程序找不到请求的文件，则应返回 HTTP `404`状态码。我们可以通过将`NamedFile`包裹在`Option`中轻松地返回`404`状态码。如果`NamedFile`是`None`，则响应将自动具有`404`状态。修改`src/routes/mod.rs`中的`assets`函数签名：

    ```rs
    #[get("/<filename..>")]
    pub async fn assets(filename: PathBuf) -> Option<NamedFile> {}
    ```

1.  然后，我们可以实现`assets`函数：

    ```rs
    let mut filename = Path::new(relative!("static")).join(filename);
    NamedFile::open(filename).await.ok()
    ```

1.  不幸的是，Rocket 如果文件名是目录，会返回 HTTP `200`状态码，因此攻击者可以尝试攻击并映射`static`文件夹内的文件夹结构。让我们通过添加以下这些行来处理这种情况：

    ```rs
    let mut filename = Path::new(relative!("static")).join(filename);
    if filename.is_dir() {
        filename.push("index.html");
    }
    NamedFile::open(filename).await.ok()
    ```

如果攻击者尝试系统地检查静态文件内的路径，攻击者将收到 HTTP `404`状态码，并且无法推断出`static`文件夹内的文件夹结构。

1.  还有另一种提供静态文件的方法：使用内置的`rocket::fs::FileServer`结构。在`src/routes/mod.rs`中删除处理静态资源的函数，并在`src/main.rs`中追加以下行：

    ```rs
    use rocket::fs::relative;
    use rocket::fs::FileServer;
    ...
    .mount("/assets", FileServer::from(
     relative!("static")))
    ```

尽管像 Rocket 这样的 Web 框架可以提供静态文件，但在 Web 服务器（如 Apache Web 服务器或 NGINX）后面提供静态文件更为常见。在更高级的设置中，人们还利用云存储，如 Amazon Web Services S3 或 Google Cloud Storage，与**内容分发网络**（**CDN**）结合使用。

在下一节中，我们将对*第六章*，*实现用户 CRUD*中创建的 HTML 进行细化。

# 介绍 Tera 模板

在 Web 应用程序中，通常有一个作为 Web 模板系统的部分。Web 设计师和 Web 开发者可以创建 Web 模板，Web 应用程序从模板生成 HTML 页面。

有不同类型的 Web 模板：服务器端 Web 模板（其中模板在服务器端渲染），客户端 Web 模板（客户端应用程序渲染模板），或混合 Web 模板。

Rust 中有几个模板引擎。我们可以在[`crates.io`](https://crates.io)或 https:/lib.rs 找到用于 Web 开发的模板引擎（例如**Handlebars**、**Tera**、**Askama**或**Liquid**）。

Rocket web 框架以`rocket_dyn_templates`crate 的形式内置了对模板的支持。目前，crate 只支持两个引擎：**Handlebars**和**Tera**。在这本书中，我们将使用 Tera 作为模板引擎以简化开发，但也欢迎尝试 Handlebars 引擎。

Tera 是一个受**Jinja2**和**Django**模板启发的模板引擎。您可以在 https://tera.netlify.app/docs/找到 Tera 的文档。Tera 模板是一个包含表达式、语句和注释的文本文件。当模板被渲染时，表达式、语句和注释被替换为变量和表达式。

例如，假设我们有一个名为`hello.txt.tera`的文件，其内容如下：

```rs
Hello {{ name }}!
```

如果我们的程序有一个名为`name`的变量，其值为`"Robert"`，我们可以创建一个包含以下内容的`hello.txt`文件：

```rs
Hello Robert!
```

如您所见，我们可以轻松地使用 Tera 模板创建 HTML 页面。在 Tera 中，我们可以使用三个分隔符：

+   `{{ }}`用于**表达式**

+   `{% %}`用于**语句**

+   `{# #}`用于**注释**

假设我们有一个名为`hello.html.tera`的模板，其内容如下：

```rs
<div>
```

```rs
    {# we are setting a variable 'name' with the value 
```

```rs
    "Robert" #}
```

```rs
    {% set name = "Robert" %}
```

```rs
    Hello {{ name }}!
```

```rs
</div>
```

我们可以用以下内容的模板将其渲染成`hello.html`文件：

```rs
<div>
```

```rs
    Hello Robert!
```

```rs
</div>
```

Tera 还具有其他功能，如在一个模板中嵌入其他模板、基本数据操作、控制结构和函数。基本数据操作包括基本的数学运算、基本的比较函数和字符串连接。控制结构包括`if`分支、`for`循环和其他模板。函数是返回一些用于模板的文本的定义过程。

我们将通过将`OurApplication`响应更改为使用 Tera 模板引擎来学习一些这些功能。让我们设置`OurApplication`以使用 Tera 模板引擎：

1.  在`Cargo.toml`文件中，添加依赖项。我们需要`rocket_dyn_templates`crate 和`serde`crate 来序列化实例：

    ```rs
    chrono = {version = "0.4", features = ["serde"]}
    rocket_dyn_templates = {path = "../../../rocket/contrib/dyn_templates/", features = ["tera"]}
    serde = "1.0.130"
    ```

1.  接下来，在`Rocket.toml`中添加一个新的配置以指定放置模板文件的文件夹：

    ```rs
    [default]
    ...
    template_dir = "src/views"
    ```

1.  在`src/main.rs`中，添加以下行以将`rocket_dyn_templates::Template`fairing 附加到应用程序：

    ```rs
    use rocket_dyn_templates::Template;
    ...
    #[launch]
    async fn rocket() -> Rocket<Build> {
    ...
    rocket::build()
    ...
        .attach(Template::fairing())
    ...
    }
    ```

添加`Template`fairing 很简单。我们将在下一节通过将`RawHtml`替换为`Template`来学习`Template`。

# 展示用户

我们将修改`/users/<uuid>`和`/users/`的路线，采取以下步骤：

1.  我们需要做的第一件事是创建一个模板。我们已经在`/src/views`中配置了模板文件夹，所以请在`src`文件夹中创建一个名为`views`的文件夹，然后在`views`文件夹内部创建一个名为`template.html.tera`的模板文件。

1.  我们将使用该文件作为所有 HTML 文件的基 HTML 模板。在`src/views/template.html.tera`内部添加以下 HTML 标签：

    ```rs
    <!DOCTYPE html>
    <html lang="en">
    <head>
      <meta charset="utf-8" />
      <title>Our Application User</title>
      <link href="/assets/mini-default.css" 
      rel="stylesheet">
      <link rel="icon" type="image/png" href="/assets/
      favicon.png">
      <meta name="viewport" content="width=device-width, 
      initial-scale=1">
    </head>
    <body>
      <div class="container"></div>
    </body>
    </html>
    ```

注意我们在 HTML 文件中包含了一个 CSS 文件。您可以从 [`minicss.org/`](https://minicss.org/) 下载开源 CSS 文件并将其放入 `static` 文件夹中。由于我们已经在 `/assets/<filename..>` 路径上创建了一个用于服务静态文件的路由，我们可以在 HTML 文件中直接使用该路由。

1.  对于下一步，我们需要包含一个可以放置我们想要渲染的 HTML 文本的部分，以及一个可以插入 `flash` 消息的部分。修改 `src/views/template.html.tera` 如下：

    ```rs
    <div class="container">
      {% if flash %}
        <div class="toast" onclick="this.remove()">
          {{ flash | safe }}
        </div>
      {% endif %}
      {% block body %}{% endblock body %}
    </div>
    ```

默认情况下，所有变量都会被转义渲染以避免 XSS（跨站脚本）攻击。我们添加了条件，如果存在 `flash` 变量，我们将变量放在 `{{ flash }}` 表达式内。为了使 HTML 标签以原始形式渲染而不被转义，我们可以使用 `| safe` 过滤器。

Tera 还有其他内置过滤器，如 `lower`、`upper`、`capitalize` 等。对于内容，我们使用 `block` 语句。`block` 语句意味着我们将在语句中包含另一个模板。您也可以看到 `block` 以 `endblock` 语句结束。

1.  Tera 可以渲染实现 Serde 的 `Serializable` 特性的任何类型。让我们修改 `User` 和相关类型以实现 `Serializable` 特性。在 `src/models/user.rs` 文件中，按如下修改文件：

    ```rs
    use rocket::serde::Serialize;
    ...
    #[derive(Debug, FromRow, FromForm, Serialize)]
    pub struct User {
    ...
    ```

1.  修改 `src/models/user_status.rs` 如下：

    ```rs
    use rocket::serde::Serialize;
    ...
    #[derive(sqlx::Type, Debug, FromFormField, Serialize)]
    #[repr(i32)]
    pub enum UserStatus {
    ...
    ```

1.  此外，修改 `src/models/our_date_time.rs` 如下：

    ```rs
    use rocket::serde::Serialize;
    #[derive(Debug, sqlx::Type, Clone, Serialize)]
    #[sqlx(transparent)]
    pub struct OurDateTime(pub DateTime<Utc>);
    ...
    ```

1.  对于 `Pagination`，我们可以直接派生 `Serialize`，但使用 URL 中的时间戳看起来不太好，例如，`/users?pagination.next=``&pagination.limit=2`。我们可以通过将 `OurDateTime` 转换为 `i64` 和反之亦然来使分页 URL 看起来更好。在 `src/models/pagination.rs` 文件中，追加以下行：

    ```rs
    use rocket::serde::Serialize;
    ...
    #[derive(Serialize)]
    pub struct PaginationContext {
        pub next: i64,
        pub limit: usize,
    }
    impl Pagination {
        pub fn to_context(&self) -> PaginationContext {
            PaginationContext {
                next: self.next.0.timestamp_nanos(),
                limit: self.limit,
            }
        }
    }
    ```

1.  由于我们不再使用 `RawHtml`，请从 `src/routes/mod.rs` 文件中删除 `use rocket::response::content::RawHtml;` 指令，并按如下修改文件：

    ```rs
    use rocket_dyn_templates::Template;
    ...
    type HtmlResponse = Result<Template, Status>;
    ```

1.  在 `src/routes/user.rs` 文件中，删除 `use rocket::response::content::RawHtml` 指令。我们将添加 `Template` 指令以使响应返回 `Template`，但我们还需要 Serde 的 `Serialize` 和 `context!` 宏的帮助来将对象转换为 Tera 变量。追加以下行：

    ```rs
    use rocket::serde::Serialize;
    ...
    use rocket_dyn_templates::{context, Template};
    ```

1.  然后，我们可以修改 `get_user` 函数。在 `src/routes/user.rs` 文件中的 `get_user` 函数内，删除所有与 `RawHtml` 相关的内容。从 `let mut html_string = String::from(USER_HTML_PREFIX);` 到 `Ok(RawHtml(html_string))` 的所有行都应删除。

1.  然后，将删除的行内容替换为以下行：

    ```rs
    #[derive(Serialize)]
    struct GetUser {
        user: User,
        flash: Option<String>,
    }
    let flash_message = flash.map(|fm| String::from(fm.message()));
    let context = GetUser {
        user,
        flash: flash_message,
    };
    Ok(Template::render("users/show", &context))
    ```

记住 Tera 模板可以使用实现 `Serializable` 特性的任何 Rust 类型。我们定义了派生 `Serializable` 特性的 `GetUser` 结构体。由于 `User` 结构体已经实现了 `Serializable`，我们可以在 `GetUser` 结构体中使用它作为字段。创建 `GetUser` 的新实例后，我们告诉应用程序渲染 `"users/show"` 模板文件。

1.  由于我们已经告诉应用程序模板名称是`"users/show"`，所以在`src/views`内部创建一个名为`users`的新文件夹。在`src/views/users`文件夹内部，创建一个新文件，`src/views/users/show.html.tera`。之后，在文件中添加以下行：

    ```rs
    {% extends "template" %}
    {% block body %}
      {% include "users/_user" %}
      <a href="/users/edit/{{user.uuid}}" class="
      button">Edit User</a>
      <form accept-charset="UTF-8" action="/users/
      delete/{{user.uuid}}" autocomplete="off" 
      method="POST" id="deleteUser"
          class="hidden"></form>
      <button type="submit" value="Submit" 
      form="deleteUser">Delete</button>
      <a href="/users" class="button">User List</a>
    {% endblock body %}
    ```

第一条语句`{% extends "template" %}`表示我们正在扩展我们之前创建的`src/views/template.html.tera`。父`src/views/template.html.tera`有一个语句`{% block body %}{% endblock body %}`，我们告诉 Tera 引擎用`src/views/users/show.html.tera`中相同块的内容覆盖该块。

1.  在这段代码中，我们还可以看到`{% include "users/_user" %}`，所以让我们创建一个`src/views/users/_user.html.tera`文件，并添加以下行：

    ```rs
    <div class="row">
      <div class="col-sm-3"><mark>UUID:</mark></div>
      <div class="col-sm-9"> {{ user.uuid }}</div>
    </div>
    <div class="row">
      <div class="col-sm-3"><mark>Username:</mark></div>
      <div class="col-sm-9"> {{ user.username }}</div>
    </div>
    <div class="row">
      <div class="col-sm-3"><mark>Email:</mark></div>
      <div class="col-sm-9"> {{ user.email }}</div>
    </div>
    <div class="row">
      <div class="col-sm-3"><mark>
      Description:</mark></div>
      <div class="col-sm-9"> {{ user.description }}</div>
    </div>
    <div class="row">
      <div class="col-sm-3"><mark>Status:</mark></div>
      <div class="col-sm-9"> {{ user.status }}</div>
    </div>
    <div class="row">
      <div class="col-sm-3"><mark>Created At:</mark></div>
      <div class="col-sm-9"> {{ user.created_at }}</div>
    </div>
    <div class="row">
      <div class="col-sm-3"><mark>Updated At:</mark></div>
      <div class="col-sm-9"> {{ user.updated_at }}</div>
    </div>
    ```

在这两个文件中，您将看到许多表达式，例如`{{ user.username }}`。这些表达式使用我们之前定义的变量：`let context = GetUser { user, flash: flash_message,};`。然后，我们告诉应用程序渲染模板：`Ok(Template::render("users/show", &context))`。

你可能想知道为什么我们要将`show.html.tera`和`_user.html.tera`分开。使用模板系统的一个好处是我们可以重用模板。我们希望在`get_users`函数中重用相同的用户 HTML。

1.  让我们修改`src/routes/user.rs`文件中的`get_users`函数。删除从`let mut html_string = String::from(USER_HTML_PREFIX);`到`Ok(RawHtml(html_string))`的行。将这些行替换为以下行：

    ```rs
    let context = context! {users: users, pagination: new_pagination.map(|pg|pg.to_context())};
    Ok(Template::render("users/index", context))
    ```

1.  我们不是定义一个新的结构体，例如`GetUser`，而是使用`context!`宏。通过使用`context!`宏，我们不需要创建一个新的类型传递给模板。现在，创建一个名为`src/views/users/index.html.tera`的新文件，并将以下行添加到文件中：

    ```rs
    {% extends "template" %}
    {% block body %}
      {% for user in users %}
        <div class="container">
          <div><mark class="tag">{{loop.
          index}}</mark></div>
          {% include "users/_user" %}
          <a href="/users/{{ user.uuid }}" class="
          button">See User</a>
          <a href="/users/edit/{{ user.uuid }}" 
          class="button">Edit User</a>
        </div>
      {% endfor %}
      {% if pagination %}
        <a href="/users?pagination.next={{
        pagination.next}}&pagination.limit={{
        pagination.limit}}" class="button">
          Next
        </a>
      {% endif %}
      <a href="/users/new" class="button">New user</a>
    {% endblock %}
    ```

在这里我们看到了两个新事物：一个`{% for user in users %}...{% endfor %}`语句，它可以用来迭代数组，以及`{{loop.index}}`，用于获取`for`循环中的当前迭代。

1.  我们还想修改`new_user`和`edit_user`函数，但在那之前，我们想看看`get_user`和`get_users`的实际应用。由于我们已经将`HtmlResponse`别名改为`Result<Template, Status>`，我们需要在`new_user`和`edit_user`中将`Ok(RawHtml(html_string))`转换为使用模板。将`new_user`和`edit_user`函数中的`Ok(RawHtml(html_string))`改为`Ok(Template::render("users/tmp", context!()))`，并创建一个空的`src/views/users/tmp.html.tera`文件。

1.  现在，我们可以运行应用程序并检查我们用 CSS 改进的页面：

![图 8.1 – get_user()渲染

![图 8.1 – get_user()渲染

图 8.1 – get_user()渲染

我们可以看到模板正在与应用程序提供的正确 CSS 文件一起工作。在下一节中，我们还将修改表单以使用模板。

# 使用表单

如果我们查看`new_user`和`edit_user`表单的结构，我们可以看到这两个表单几乎相同，只有一些细微的差别。例如，表单的`action`端点不同，因为`edit_user`有两个额外的字段：`_METHOD`和`old_password`。为了简化，我们可以制作一个模板供这两个函数使用。让我们看看步骤：

1.  创建一个名为`src/views/users/form.html.tera`的模板，并插入以下行：

    ```rs
    {% extends "template" %}
    {% block body %}
      <form accept-charset="UTF-8" action="{{ form_url }}" 
      autocomplete="off" method="POST">
        <fieldset>
        </fieldset>
      </form>
    {% endblock %}
    ```

1.  接下来，让我们通过添加一个`legend`标签来给表单添加标题。将此放在`fieldset`标签内：

    ```rs
    <legend>{{ legend }}</legend>
    ```

1.  在`legend`标签下，如果我们正在编辑用户，可以添加一个额外的字段：

    ```rs
    {% if edit %}
      <input type="hidden" name="_METHOD" value="PUT" />
    {% endif %}
    ```

1.  继续添加字段，按照以下方式添加`username`和`email`字段：

    ```rs
    <div class="row">
      <div class="col-sm-12 col-md-3">
        <label for="username">Username:</label>
      </div>
      <div class="col-sm-12 col-md">
        <input name="username" type="text" value="{{ 
        user.username }}"/>
      </div>
    </div>
    <div class="row">
      <div class="col-sm-12 col-md-3">
        <label for="email">Email:</label>
      </div>
      <div class="col-sm-12 col-md">
        <input name="email" type="email" value="{{ 
        user.email }}"/>
      </div>
    </div>
    ```

1.  在`email`字段之后添加一个条件`old_password`字段：

    ```rs
    {% if edit %}
      <div class="row">
        <div class="col-sm-12 col-md-3">
          <label for="old_password">Old password:</label>
        </div>
        <div class="col-sm-12 col-md">
          <input name="old_password" type="password" />
        </div>
      </div>
    {% endif %}
    ```

1.  添加其余的字段：

    ```rs
    <div class="row">
      <div class="col-sm-12 col-md-3">
        <label for="password">Password:</label>
      </div>
      <div class="col-sm-12 col-md">
        <input name="password" type="password" />
      </div>
    </div>
    <div class="row">
      <div class="col-sm-12 col-md-3">
        <label for="password_confirmation">Password 
        Confirmation:</label>
      </div>
      <div class="col-sm-12 col-md">
        <input name="password_confirmation" type=
        "password" />
      </div>
    </div>
    <div class="row">
      <div class="col-sm-12 col-md-3">
        <label for="description">Tell us a little bit more 
        about yourself:</label>
      </div>
      <div class="col-sm-12 col-md">
        <textarea name="description">{{ user.description 
        }}</textarea>
      </div>
    </div>
    <button type="submit" value="Submit">Submit</button>
    ```

1.  然后，将标签和字段更改为显示值（如果有值）：

    ```rs
    <input name="username" type="text" {% if user %}value="{{ user.username }}"{% endif %} />
    ...
    <input name="email" type="email" {% if user %}value="{{ user.email }}"{% endif %} />
    ...
    <label for="password">{% if edit %}New Password:{% else %}Password:{% endif %}</label>
    ...
    <textarea name="description">{% if user %}{{ user.description }}{% endif %}</textarea>
    ```

1.  在我们创建了表单模板之后，我们可以修改`new_user`和`edit_user`函数。在`new_user`中，从`let mut html_string = String::from(USER_HTML_PREFIX);`到`Ok(Template::render("users/tmp", context!()))`删除这些行以创建`RawHtml`。

在`form.html.tera`中，我们添加了这些变量：`form_url`、`edit`和`legend`。我们还需要将`Option<FlashMessage<'_>>`转换为`String`，因为`FlashMessage`的`Serializable`特质的默认实现不是人类可读的。在`new_user`函数中添加这些变量并渲染模板，如下所示：

```rs
let flash_string = flash
    .map(|fl| format!("{}", fl.message()))
    .unwrap_or_else(|| "".to_string());
let context = context! {
    edit: false,
    form_url: "/users",
    legend: "New User",
    flash: flash_string,
};
Ok(Template::render("users/form", context))
```

1.  对于`edit_user`，我们可以创建相同的变量，但这次我们知道`user`的数据，因此我们可以将`user`包含在上下文中。在`src/routes/user.rs`的`edit_user`函数中删除从`let mut html_string = String::from(USER_HTML_PREFIX);`到`Ok(Template::render("users/tmp", context!()))`的行。

将这些行替换为以下代码：

```rs
let flash_string = flash
    .map(|fl| format!("{}", fl.message()))
    .unwrap_or_else(|| "".to_string());
let context = context! {
    form_url: format!("/users/{}",&user.uuid ),
    edit: true,
    legend: "Edit User",
    flash: flash_string,
    user,
};
Ok(Template::render("users/form", context))
```

1.  作为最后的润色，我们可以从`src/routes/user.rs`中删除`USER_HTML_PREFIX`和`USER_HTML_SUFFIX`常量。我们还应该删除`src/views/users/tmp.html.tera`文件，因为再也没有函数使用该文件了。而且，因为我们已经在模板中用`<div></div>`标签包围了闪存消息，所以我们可以从闪存消息中删除`div`的使用。例如，在`src/routes/user.rs`中，我们可以修改以下行：

    ```rs
    Ok(Flash::success(
        Redirect::to(format!("/users/{}", user.uuid)),
        "<div>Successfully created user</div>",
    ))
    ```

我们可以将它们修改为以下行：

```rs
Ok(Flash::success(
    Redirect::to(format!("/users/{}", user.uuid)),
    "Successfully created user",
))
```

我们还可以改进表单的一个方面，那就是添加一个令牌来保护应用程序免受**跨站请求伪造**（**CSRF**）攻击。我们将在下一节学习如何保护我们的表单。

# 保护 HTML 表单免受 CSRF 攻击

最常见的安全攻击之一是 CSRF，恶意第三方诱使用户发送一个具有与预期不同的值的 Web 表单。减轻这种攻击的一种方法是在表单内容中发送一个一次性令牌。然后，Web 服务器检查令牌的有效性，以确保请求来自正确的 Web 浏览器。

我们可以在 Rocket 应用程序中通过创建一个将生成令牌并检查发送回的表单值的公平性来创建这样的令牌。让我们看看步骤：

1.  首先，我们需要添加这个依赖项。我们将需要一个`base64`crate 来将二进制值编码和解码为字符串。我们还需要 Rocket 的`secrets`功能来存储和检索私有 cookie。私有 cookie 就像常规 cookie 一样，但它们被我们在`Rocket.toml`文件中配置的`secret_key`加密。

对于依赖项，我们还需要添加`time`作为依赖项。在`Cargo.toml`中添加以下行：

```rs
base64 = {version = "0.13.0"}
...
rocket = {path = "../../../rocket/core/lib/", features = ["uuid", "json", "secrets"]}
...
time = {version = "0.3", features = ["std"]}
```

防止 CSRF 的步骤是生成随机字节，将随机字节存储在私有 cookie 中，将随机字节作为字符串进行哈希处理，并渲染带有令牌字符串的表单模板。当用户发送令牌回来时，我们可以从 cookie 中检索令牌并比较两者。

1.  要创建一个 CSRF 公平性，添加一个新的模块。在`src/fairings/mod.rs`中添加新的模块：

    ```rs
    pub mod csrf;
    ```

1.  之后，创建一个名为`src/fairings/csrf.rs`的文件，并添加存储随机字节的 cookie 默认值的依赖项和常量：

    ```rs
    use argon2::{
        password_hash::{
            rand_core::{OsRng, RngCore},
            PasswordHash, PasswordHasher, 
            PasswordVerifier, SaltString,
        },
        Argon2,
    };
    use rocket::fairing::{self, Fairing, Info, Kind};
    use rocket::http::{Cookie, Status};
    use rocket::request::{FromRequest, Outcome, Request};
    use rocket::serde::Serialize;
    use rocket::{Build, Data, Rocket};
    use time::{Duration, OffsetDateTime};
    const CSRF_NAME: &str = "csrf_cookie";
    const CSRF_LENGTH: usize = 32;
    const CSRF_DURATION: Duration = Duration::hours(1);
    ```

然后，我们可以扩展 Rocket 的`Request`以添加一个新的方法来检索 CSRF 令牌。因为`Request`是一个外部 crate，我们无法添加另一个方法，但我们可以通过添加一个 trait 并使外部 crate 类型扩展此 trait 来克服这一点。我们不能使用外部 trait 扩展外部 crate，但使用内部 trait 扩展外部 crate 是允许的。

1.  我们想要创建一个方法来从私有 cookie 中检索 CSRF 令牌。继续在`src/fairings/csrf.rs`中添加以下行：

    ```rs
    trait RequestCsrf {
        fn get_csrf_token(&self) -> Option<Vec<u8>>;
    }
    impl RequestCsrf for Request<'_> {
        fn get_csrf_token(&self) -> Option<Vec<u8>> {
            self.cookies()
                .get_private(CSRF_NAME)
                .and_then(|cookie| base64::
                decode(cookie.value()).ok())
                .and_then(|raw| {
                    if raw.len() >= CSRF_LENGTH {
                        Some(raw)
                    } else {
                        None
                    }
                })
        }
    }
    ```

1.  之后，我们想要添加一个检索或生成并存储随机字节（如果 cookie 不存在）的公平性。添加一个新的结构体来作为公平性管理：

    ```rs
    #[derive(Debug, Clone)]
    pub struct Csrf {}
    impl Csrf {
        pub fn new() -> Self {
            Self {}
        }
    }
    #[rocket::async_trait]
    impl Fairing for Csrf {
        fn info(&self) -> Info {
            Info {
                name: "CSRF Fairing",
                kind: Kind::Ignite | Kind::Request,
            }
        }
        async fn on_ignite(&self, rocket: Rocket<Build>) –
        > fairing::Result {
            Ok(rocket.manage(self.clone()))
        }
    }
    ```

1.  我们想要首先检索令牌，如果令牌不存在，则生成随机字节并将字节添加到私有令牌中。在`impl Fairing`块中，添加`on_request`函数：

    ```rs
    async fn on_request(&self, request: &mut Request<'_>, _: &mut Data<'_>) {
        if let Some(_) = request.get_csrf_token() {
            return;
        }
        let mut key = vec![0; CSRF_LENGTH];
        OsRng.fill_bytes(&mut key);
        let encoded = base64::encode(&key[..]);
        let expires = OffsetDateTime::now_utc() + CSRF_
        DURATION;
        let mut csrf_cookie = Cookie::new(
        String::from(CSRF_NAME), encoded);
        csrf_cookie.set_expires(expires);
        request.cookies().add_private(csrf_cookie);
    }
    ```

1.  我们需要一个请求保护者来从请求中检索令牌字符串。添加以下行：

    ```rs
    #[derive(Debug, Serialize)]
    pub struct Token(String);
    #[rocket::async_trait]
    impl<'r> FromRequest<'r> for Token {
        type Error = ();
        async fn from_request(request: &'r Request<'_>) -> 
        Outcome<Self, Self::Error> {
            match request.get_csrf_token() {
                None => Outcome::Failure((Status::
                Forbidden, ())),
                Some(token) => Outcome::
                Success(Self(base64::encode(token))),
            }
        }
    }
    ```

1.  如果找不到令牌，我们返回 HTTP `403`状态码。我们还需要两个额外的函数：生成哈希和比较令牌哈希与其他字符串。由于我们已经在密码哈希中使用`argon2`，我们可以重用`argon2`crate 来执行这些函数。添加以下行：

    ```rs
    impl Token {
        pub fn generate_hash(&self) -> Result<String, 
        String> {
            let salt = SaltString::generate(&mut OsRng);
            Argon2::default()
                .hash_password(self.0.as_bytes(), &salt)
                .map(|hp| hp.to_string())
                .map_err(|_| String::from("cannot hash 
                authenticity token"))
        }
        pub fn verify(&self, form_authenticity_token: 
        &str) -> Result<(), String> {
            let old_password_hash = self.generate_hash()?;
            let parsed_hash = PasswordHash::new(&old_
            password_hash)
                .map_err(|_| String::from("cannot verify 
                authenticity token"))?;
            Ok(Argon2::default()
                .verify_password(form_authenticity_
                token.as_bytes(), &parsed_hash)
                .map_err(|_| String::from("cannot verify 
                authenticity token"))?)
        }
    }
    ```

1.  在我们设置了`Csrf`公平性之后，我们可以在应用程序中使用它。在`src/main.rs`中，将公平性附加到 Rocket 应用程序：

    ```rs
    use our_application::fairings::{csrf::Csrf, db::DBConnection};
    ...
    async fn rocket() -> Rocket<Build> {
    ...
            .attach(Csrf::new())
    ...
    }
    ```

1.  在`src/models/user.rs`中添加一个新字段来包含从表单发送的令牌：

    ```rs
    pub struct NewUser<'r> {
    ...
        pub authenticity_token: &'r str,
    }
    ...
    pub struct EditedUser<'r> {
    ...
        pub authenticity_token: &'r str,
    }
    ```

1.  在`src/views/users/form.html.tera`中添加一个字段来存储令牌字符串：

    ```rs
    <form accept-charset="UTF-8" action="{{ form_url }}" autocomplete="off" method="POST">
      <input type="hidden" name="authenticity_token" 
      value="{{ csrf_token }}"/>
    ...
    ```

1.  最后，我们可以修改`src/routes/user.rs`。添加`Token`依赖项：

    ```rs
    use crate::fairings::csrf::Token as CsrfToken;
    ```

1.  我们可以将`CsrfToken`用作请求保护者，将令牌传递到模板中，并将模板作为 HTML 渲染：

    ```rs
    pub async fn new_user(flash: Option<FlashMessage<'_>>, csrf_token: CsrfToken) -> HtmlResponse {
    ...
        let context = context! {
            ...
            csrf_token: csrf_token,
        };
        ...
    }
    ...
    pub async fn edit_user(
        mut db: Connection<DBConnection>, uuid: &str, 
        flash: Option<FlashMessage<'_>>, csrf_token: 
        CsrfToken) -> HtmlResponse {
    ...
        let context = context! {
            ...
            csrf_token: csrf_token,
        };
        ...
    }
    ```

1.  修改 `create_user` 函数以验证令牌，如果哈希值不匹配则返回：

    ```rs
    pub async fn create_user<'r>(
        ...
        csrf_token: CsrfToken,
    ) -> Result<Flash<Redirect>, Flash<Redirect>> {
        …
        let new_user = user_context.value.as_ref().
        unwrap();
        csrf_token
            .verify(&new_user.authenticity_token)
            .map_err(|_| {
                Flash::error(
                    Redirect::to("/users/new"),
                    "Something went wrong when creating 
                    user",
                )
            })?;
        ...
    }
    ```

1.  同样对 `update_user`、`put_user` 和 `patch_user` 函数进行操作：

    ```rs
    pub async fn update_user<'r>(
        ...
        csrf_token: CsrfToken,
    ) -> Result<Flash<Redirect>, Flash<Redirect>> {
        ...
        match user_value.method {
            "PUT" => put_user(db, uuid, user_context, 
            csrf_token).await,
            "PATCH" => patch_user(db, uuid, user_context, 
            csrf_token).await,
            ...
        }
    }
    ...
    pub async fn put_user<'r>(
        ...
        csrf_token: CsrfToken,
    ) -> Result<Flash<Redirect>, Flash<Redirect>> {
        let user_value = user_context.value.as_ref().
        unwrap();
        csrf_token
            .verify(&user_value.authenticity_token)
            .map_err(|_| {
                Flash::error(
                    Redirect::to(format!("/users/edit/{}",
                    uuid)),
                    "Something went wrong when updating 
                    user",
                )
            })?;
        …
    }
    …
    pub async fn patch_user<'r>(
        ...
        csrf_token: CsrfToken,
    ) -> Result<Flash<Redirect>, Flash<Redirect>> {
        put_user(db, uuid, user_context, csrf_token).await
    }
    ```

然后，尝试重新启动应用程序并发送不带令牌的表单。我们应该看到应用程序返回 HTTP `403` 状态码。CSRF 是最常见的网络攻击之一，但我们已经学习了如何通过使用 Rocket 功能来减轻这种攻击。

# 摘要

在本章中，我们学习了三个对于网络应用程序来说是常见的事情。第一点是学习如何通过使用 `PathBuf` 或 `FileServer` 结构来让 Rocket 应用程序服务静态文件。

另一件事是我们已经学到的，那就是如何使用 `rocket_dyn_templates` 将模板转换为客户端的响应。我们还了解了模板引擎 Tera 以及 Tera 模板引擎的各种功能。

通过利用静态资源和模板，我们可以轻松地创建现代网络应用程序。在下一章中，我们将学习关于用户帖子：文本、图片和视频的内容。
