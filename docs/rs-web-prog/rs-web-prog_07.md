

# 第七章：管理用户会话

到目前为止，我们的应用程序通过在视图中点击按钮来操作数据库中的数据。然而，任何遇到我们的应用程序的人都可以编辑数据。虽然我们的应用程序不需要太多的安全性，但了解和实践这一概念在一般 Web 开发中非常重要。

在本章中，我们将构建一个创建用户的系统。此系统还将通过要求用户在通过前端应用程序修改任何待办事项之前登录来管理用户会话。

在本章中，我们将涵盖以下主题：

+   通过数据库迁移创建具有与其他表相关联的具有某些字段唯一约束的用户数据模型

+   认证我们的用户

+   管理用户会话

+   清理认证要求

+   配置认证令牌的过期时间

+   将认证添加到我们的前端

在阅读本章之后，您将能够理解在 Web 服务器上认证用户的基本知识。您还将能够在我们的 Rust 应用程序的服务器端实现此认证，并在前端 React 应用程序中存储凭证。对本章涵盖的概念和实践的理解也将使您能够通过 React Native 在手机应用程序中集成认证，并通过将我们的 React 应用程序包装在**Electron**中在 Rust 服务器和桌面应用程序中实现。

# 技术要求

在本章中，我们将在前一章构建的代码基础上进行构建。您可以在以下 URL 找到这些代码：[`github.com/PacktPublishing/Rust-Web-Programming-2nd-Edition/tree/main/chapter06/building_a_database_connection_pool`](https://github.com/PacktPublishing/Rust-Web-Programming-2nd-Edition/tree/main/chapter06/building_a_database_connection_pool)。

本章的代码可以在[`github.com/PacktPublishing/Rust-Web-Programming-2nd-Edition/tree/main/chapter07`](https://github.com/PacktPublishing/Rust-Web-Programming-2nd-Edition/tree/main/chapter07)找到。

# 创建我们的用户模型

由于我们在应用程序中管理用户会话，我们需要在允许创建、删除和编辑待办事项之前存储有关用户的信息以检查其凭证。我们将把我们的用户数据存储在**PostgreSQL**数据库中。虽然这不是必需的，但我们还将数据库中的用户与待办事项链接起来。这将使我们了解如何修改现有表并在表之间创建链接。为了创建我们的用户模型，我们将必须执行以下操作：

1.  创建一个`User`数据模型。

1.  创建一个`NewUser`数据模型。

1.  修改待办事项数据模型，以便我们可以将其链接到用户模型。

1.  更新模式文件，包含新的表和修改的字段。

1.  在数据库上创建和运行迁移脚本。

在接下来的几节中，我们将详细查看前面的步骤。

## 创建用户数据模块

在我们开始之前，我们需要更新`Cargo.toml`文件中的依赖项，如下所示：

```rs
[dependencies]
. . .
bcrypt = "0.13.0"
uuid = {version = "1.0.0", features = ["serde", "v4"]}
```

我们将使用`bcrypt`包来散列和检查密码，以及使用`uuid`包为我们用户数据模型生成唯一的 ID。正如我们在*第六章*中讨论的，*使用 PostgreSQL 进行数据持久化*，我们需要为我们的用户数据模型创建两个不同的结构体。

新用户将没有`id`字段，因为这个字段在数据库中尚不存在。当新用户被插入到表中时，数据库将创建这个 ID。然后我们还有一个具有所有相同字段的结构体，包括我们添加的`id`字段，因为我们可能需要在与数据库中现有用户交互时使用这个 ID。ID 数字可以用于引用其他表。它们很短，我们知道它们是唯一的。我们将使用用户 ID 将用户与待办事项链接起来。这些数据模型可以放在以下文件结构中，在`src/models.rs`目录下：

```rs
└── user
    ├── mod.rs
    ├── new_user.rs
    └── user.rs
```

我们将在`new_user.rs`文件中定义数据模型。首先，我们必须定义导入，如下所示：

```rs
use uuid::Uuid;
use diesel::Insertable;
use bcrypt::{DEFAULT_COST, hash};
use crate::schema::users;
```

必须注意，我们尚未在模式中定义用户。在完成所有数据模型之后，我们将解决这个问题。在定义了我们的`users`模式之前，我们将无法编译我们的代码。我们还将导入一个唯一的 ID 包，因为我们将在创建新用户时创建一个唯一的 ID，以及从`diesel`包中导入的`Insertable`特质，因为我们将在数据库中插入新用户。然后我们使用`bcrypt`包中的`hash`函数来散列我们为新用户定义的新密码。我们还可以看到我们导入了来自`bcrypt`包的`DEFAULT_COST`常量。`DEFAULT_COST`仅仅是一个我们将传递给`hash`函数的常量。我们将在下一节中探讨为什么这是这种情况，当我们介绍散列密码时。现在我们已经定义了用户数据模型模块并导入了我们需要的内容，我们可以继续到下一节创建`NewUser`结构体。

## 创建一个 NewUser 数据模型

我们可以使用以下代码定义我们的数据模型：

```rs
#[derive(Insertable, Clone)]
#[table_name="users"]
pub struct NewUser {
    pub username: String,
    pub email: String,
    pub password: String,
    pub unique_id: String,
}
```

在这里，我们可以看到我们允许我们的数据模型可插入。然而，我们不允许它被查询。我们想要确保当从数据库中检索用户时，他们的 ID 是存在的。我们本可以继续定义用户的通用数据模型，但这并不安全。我们需要确保通过加密来保护我们的密码。如果你还记得第二章，*在 Rust 中设计您的 Web 应用程序*，我们利用特性允许某些待办事项结构执行操作。一些结构可以创建，而其他结构可以根据它们实现的特性进行删除。我们在这里通过仅实现`Insertable`特性来锁定`NewUser`结构的函数。然而，我们将通过为`User`结构实现其他特性来启用查询，如图下所示：

![图 7.1 – 使用特性锁定数据模型结构](img/Figure_7.1_B18722.jpg)

图 7.1 – 使用特性锁定数据模型结构

现在我们已经创建了将新用户插入数据库的结构，我们可以探索如何在数据库中存储我们的用户密码。

你可能想知道为什么你不能恢复忘记的密码；你只能重置它们。这是因为密码被*加密*了。加密密码是存储密码时的常见做法。这就是我们使用算法来混淆密码，使其无法被读取的地方。一旦这样做，就无法逆转。

加密后的密码随后被存储在数据库中。为了验证密码，输入的密码会被加密并与数据库中存储的加密密码进行比较。这使我们能够查看输入的加密密码是否与数据库中存储的加密密码相匹配。这有几个优点。首先，它防止了有权访问您数据的员工知道您的密码。如果发生数据泄露，它也防止泄露的数据直接将您的密码暴露给拥有数据的人。

考虑到很多人使用相同的密码进行多项操作（尽管他们不应该这样做），如果你不散列密码并且发生数据泄露，你可以想象这将对使用你应用程序的人造成多大的损害。然而，散列比这更复杂。有一个叫做 *盐值* 的概念，确保当你散列相同的密码时，不会得到相同的散列结果。它是通过在散列之前向密码中添加额外的数据来实现的。这也是我们传递给 `hash` 函数的 `DEFAULT_COST` 常量所在的地方。假设我们获得了数据库中的数据，并想编写代码来猜测数据中的密码。如果我们有足够的计算能力，我们实际上可以猜测密码。因此，我们可以传递一个成本参数。随着成本参数的增加，无论是 CPU 时间还是内存的工作量都会呈指数增长。将成本因子增加一个单位将使计算散列所需的操作数量增加 10,000 或更多。

详细解释密码安全性超出了本书的范围。然而，必须强调的是，在存储密码时，密码散列始终是必须的。幸运的是，所有主流语言中都有一系列模块，只需几行代码即可实现密码的散列和检查。Rust 也不例外。

为了确保我们可以将新的用户以散列密码的形式插入到数据库中，请按照以下步骤操作：

1.  首先，我们必须确保在 `NewUser` 构造函数中散列输入密码，它定义如下：

    ```rs
    impl NewUser {
    ```

    ```rs
        pub fn new(username: String,
    ```

    ```rs
            email: String, password: String) -> NewUser {
    ```

    ```rs
            let hashed_password: String = hash(
    ```

    ```rs
                    password.as_str(), DEFAULT_COST
    ```

    ```rs
                ).unwrap();
    ```

    ```rs
            let uuid = Uuid::new_v4().to_string();
    ```

    ```rs
            return NewUser {
    ```

    ```rs
                username,
    ```

    ```rs
                email,
    ```

    ```rs
                password: hashed_password,
    ```

    ```rs
                unique_id: uuid
    ```

    ```rs
            }
    ```

    ```rs
        }
    ```

    ```rs
    }
    ```

在这里，我们使用了 `bcrypt` 包中的 `hash` 函数来散列我们的密码，其中我们还传递了 `DEFAULT_COST` 常量。我们还使用 `Uuid` 包创建了一个唯一的 ID，然后使用这些属性构造了一个新的 `NewUser` 结构体实例。在我们的应用程序中，实际上并不需要一个唯一的 ID。然而，这些在多个服务器和数据库之间通信时可能会很有用。

1.  现在我们已经定义了我们的 `NewUser` 数据模型，我们可以在 `user.rs` 文件中使用以下代码定义我们的通用用户数据模型。首先，我们必须定义以下导入：

    ```rs
    extern crate bcrypt;
    ```

    ```rs
    use diesel::{Queryable, Identifiable};
    ```

    ```rs
    use bcrypt::verify;
    ```

    ```rs
    use crate::schema::users;
    ```

在这里，我们可以看到我们正在使用 `verify` 函数，并且我们还允许通用用户数据模型结构体可查询和可识别。

1.  在上一步定义的导入之后，我们可以构建我们的 `User` 结构体。记住，这是一个在我们进行数据库查询时将从数据库中加载的结构体。在你继续阅读之前，这是一个尝试自己构建 `User` 结构体的好时机，因为它使用与 `NewUser` 结构体相同的表，但有一个 `id` 字段，并且是查询而不是插入。一旦你构建了你的 `User` 结构体，它应该看起来像以下代码：

    ```rs
    #[derive(Queryable, Clone, Identifiable)]
    ```

    ```rs
    #[table_name="users"]
    ```

    ```rs
    pub struct User {
    ```

    ```rs
        pub id: i32,
    ```

    ```rs
        pub username: String,
    ```

    ```rs
        pub email: String,
    ```

    ```rs
        pub password: String,
    ```

    ```rs
        pub unique_id: String
    ```

    ```rs
    }
    ```

我们可以看到，我们只是添加了 `id` 字段，并推导了 `Queryable` 特性而不是 `Insertable` 特性。

1.  现在我们已经定义了`User`结构体，我们可以构建一个函数来验证输入的密码是否与用户的密码匹配，以下代码所示：

    ```rs
    impl User {
    ```

    ```rs
        pub fn verify(&self, password: String) -> bool {
    ```

    ```rs
        verify(password.as_str(),
    ```

    ```rs
        &self.password).unwrap()
    ```

    ```rs
        }
    ```

    ```rs
    }
    ```

1.  现在我们已经定义了模型，我们必须记住在`models/user/mod.rs`文件中使用以下代码注册它们：

    ```rs
    pub mod new_user;
    ```

    ```rs
    pub mod user;
    ```

1.  此外，我们可以通过在`models/mod.rs`文件中添加以下行来使这些模块对应用程序可用：

    ```rs
    pub mod item;
    ```

    ```rs
    pub mod user;
    ```

因此，我们已经定义了针对用户的数据模型。然而，我们仍然需要将它们与我们的待办事项链接起来。

## 修改待办事项数据模型

要将数据模型链接到我们的待办事项，我们必须修改我们的待办事项数据模型。我们可以通过多种方式来实现这一点。例如，我们可以在项目表中添加一个`user_id`字段，它只是用户表中的`unique_id`字段。当我们创建一个新的项目时，我们就会将用户的唯一 ID 传递给项目构造函数。这很容易实现；然而，它确实存在风险。仅仅将用户的唯一 ID 传递给项目并不能强制执行该 ID 是有效的并且在数据库中。没有任何东西阻止我们将已删除用户的 ID 插入到项目构造函数中，从而将孤立的项目插入到数据库中。这将使得以后很难提取，因为我们没有关于孤立项目关联的用户 ID 的引用。我们还可以创建一个新表，该表通过用户 ID 引用项目 ID，如下所示：

![图 7.2 – 用于记录用户与项目关联的独立数据库表](img/Figure_7.2_B18722.jpg)

图 7.2 – 用于记录用户与项目关联的独立数据库表

这种方法的优点是，只需删除表就可以轻松地将用户与项目解耦。然而，它也没有在创建新条目时对有效用户 ID 或项目 ID 进行强制执行。我们还将不得不进行两次查询，一次是关联表，然后是项目表，以从用户那里获取项目。由于前两种方法（将用户 ID 列附加到项目表或创建包含项目 ID 和用户唯一 ID 的桥梁表）易于实现，我们不会探讨它们；你应该能够在这个阶段自己实现它们。在待办事项应用程序的上下文中，前两种方法将是不够好的，因为它们没有提供任何好处，却在将数据插入我们的数据库时引入了错误的风险。这并不意味着前两种方法永远不会被使用。每个项目的数据需求都是不同的。在我们的项目中，我们将创建一个外键来将我们的用户与项目链接起来，如下所示：

![图 7.3 – 用户与项目之间的外键关联](img/Figure_7.3_B18722.jpg)

图 7.3 – 用户与项目之间的外键关联

这不允许我们通过一个数据库调用访问与用户关联的项目，但我们只能插入具有合法用户 ID 引用的项目。外键也可以触发级联事件，如果在删除用户时，这将自动删除与该用户关联的所有现有项目，以防止创建孤儿项目。我们通过声明与表的链接宏来创建外键。在 `models/item/item.rs` 中，我们可以通过以下方式实现：

```rs
use crate::schema::to_do;
use chrono::NaiveDateTime;
use super::super::user::user::User;
```

我们可以看到，我们必须导入 `User` 结构体，因为我们将在 `belongs_to` 宏中引用它，以声明我们的 `Item` 结构体属于 `User` 结构体，如下面的代码所示：

```rs
#[derive(Queryable, Identifiable, Associations)]
#[belongs_to(User)]
#[table_name="to_do"]
pub struct Item {
    pub id: i32,
    pub title: String,
    pub status: String,
    pub date: NaiveDateTime,
    pub user_id: i32,
}
```

在这里，我们可以看到我们导入了用户数据模型结构体，使用 `belongs_to` 宏进行了定义，并添加了一个 `user_id` 字段来链接结构体。请注意，如果我们没有包含 `Associations` 宏，`belongs_to` 宏将无法调用。

我们最后需要做的是在 `models/item/new_item.rs` 文件中的字段和构造函数中添加 `user_id` 字段。我们需要这样做，以便可以将新的待办事项与创建该项目的用户链接起来。这可以通过以下代码实现：

```rs
use crate::schema::to_do;
use chrono::{NaiveDateTime, Utc};
#[derive(Insertable)]
#[table_name="to_do"]
pub struct NewItem {
    pub title: String,
    pub status: String,
    pub date: NaiveDateTime,
    pub user_id: i32,
}
impl NewItem {
    pub fn new(title: String, user_id: i32) -> NewItem {
        let now = Utc::now().naive_local();
        NewItem{
            title, status: String::from("PENDING"),
            date: now,
            user_id
        }
    }
}
```

因此，总结我们所做的，所有我们的数据模型结构体都已更改，并且我们能够在与数据库交互时，根据需要使用它们在应用程序中。然而，我们尚未更新数据库，也尚未更新连接应用程序和数据库的桥梁。我们将在下一步这样做。

## 更新模式文件

为了确保数据模型结构体到数据库的映射是最新的，我们必须使用这些更改更新我们的模式。这意味着我们必须更改待办事项项表的现有模式，并在 `src/schema.rs` 文件中添加用户模式。这由以下代码表示：

```rs
table! {
    to_do (id) {
        id -> Int4,
        title -> Varchar,
        status -> Varchar,
        date -> Timestamp,
        user_id -> Int4,
    }
}
table! {
    users (id) {
        id -> Int4,
        username -> Varchar,
        email -> Varchar,
        password -> Varchar,
        unique_id -> Varchar,
    }
}
```

必须注意，在模式文件中，我们的字段定义的顺序与 Rust 数据模型相同。这一点很重要，因为如果我们不这样做，当我们连接到数据库时，字段将不匹配。我们可能还会意识到，我们的模式仅仅只是定义了字段及其类型；它并没有涵盖待办事项表和用户表之间的关系。

我们不必担心这个问题，因为当我们创建和运行自己的迁移时，这个模式文件将随着关系的更新而更新。这导致我们创建自己的迁移来完成这个模式文件。

## 在数据库上创建和运行迁移脚本

运行迁移的过程与我们之前在 *第六章* 中讨论的类似，即 *使用 PostgreSQL 进行数据持久化*，该章节介绍了如何安装 Diesel 客户端并连接到数据库。首先，我们必须使用 `docker-compose` 命令运行我们的数据库：

```rs
docker-compose up
```

当我们运行迁移时，我们需要在后台运行这个程序。然后我们可以通过运行以下命令来创建迁移脚本：

```rs
diesel migration generate create_users
```

这在迁移目录中创建了一个目录，目录的名称包含`create_users`。在这个目录内，我们有两个空的*SQL*文件。在这里，我们将手动编写自己的 SQL 脚本用于迁移。最初，你可能会觉得这没有必要，因为其他语言中有库可以自动生成这些迁移，但这样做有一些优点。

首先，它让我们保持对 SQL 的熟悉，SQL 是另一个实用的工具。这使得我们能够在解决日常问题时考虑利用 SQL 的解决方案。它还让我们能够更细致地控制迁移的流程。例如，在我们将要创建的迁移中，我们需要创建用户表和基础用户，这样当我们修改`to_do`表中的列时，我们可以用占位符用户行的 ID 来填充它。我们通过以下表定义在我们的`up.sql`文件中执行这一操作：

```rs
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    username VARCHAR NOT NULL UNIQUE,
    email VARCHAR NOT NULL UNIQUE,
    password VARCHAR NOT NULL,
    unique_id VARCHAR NOT NULL
);
```

这很简单。注意，`email`和`username`字段是唯一的。这是因为我们不希望有重复用户名和电子邮件的用户。在这个级别上放置约束有多个原因。例如，我们可以通过数据库调用用户名和电子邮件来防止这种情况，如果存在重复，则拒绝插入新用户。

然而，代码中可能存在错误，或者有人可能在将来修改我们的代码。可能会引入一个新功能，这个功能没有这个检查，比如编辑功能。可能会有一个迁移修改行或插入新用户。如果你自己编写 SQL，通常最好的做法是确保你使用`;`符号来表示操作已经完成。

这个 SQL 命令被触发，然后紧接着下一个命令被触发。在我们的`up.sql`文件中，下一个命令插入了一个占位符用户行，使用以下命令：

```rs
 INSERT INTO users (username, email, password, unique_id)
VALUES ('placeholder', 'placeholder email',
'placeholder password', 'placeholder unique id');
```

现在我们已经创建了用户，然后我们修改`to_do`表。我们可以使用以下命令来完成这个操作，在刚刚写的命令相同的文件下：

```rs
ALTER TABLE to_do ADD user_id integer default 1
CONSTRAINT user_id REFERENCES users NOT NULL;
```

有了这些，我们的`up.sql`迁移已经被定义。现在，我们必须定义我们的`down.sql`迁移。在向下迁移中，我们基本上必须撤销向上迁移中做的操作。这意味着在`to_do`表中删除`user_id`列，然后完全删除用户表。这可以通过在`down.sql`文件中的以下 SQL 代码来完成：

```rs
ALTER TABLE to_do DROP COLUMN user_id;
DROP TABLE users
```

我们必须记住，Docker 必须运行，以便迁移能够影响数据库。一旦运行了这个迁移，我们可以看到以下代码已经被添加到了`src/schema.rs`文件中：

```rs
joinable!(to_do -> users (user_id));
allow_tables_to_appear_in_same_query!(
    to_do,
    users,
);
```

这使得我们的 Rust 数据模型能够查询用户和待办事项之间的关系。随着这次迁移的完成，我们可以再次运行我们的应用程序。然而，在我们这样做之前，我们只需要在`src/views/to_do/create.rs`文件中进行一个小小的修改，在`create`视图函数中的新项构造函数添加以下行代码：

```rs
let new_post = NewItem::new(title, 1);
```

现在运行我们的应用程序将导致与我们在*第六章*中描述的行为相同，即我们的应用程序正在运行我们已制作的迁移。然而，我们还需要查看我们的新用户构造函数是否正常工作，因为我们正在散列密码并生成唯一的 ID。

要做到这一点，我们需要构建一个创建用户端点。为此，我们必须定义模式，然后定义一个视图，将新用户插入到数据库中。我们可以在`src/json_serialization/new_user.rs`文件中使用以下代码创建我们的模式：

```rs
use serde::Deserialize;
#[derive(Deserialize)]
pub struct NewUserSchema {
    pub name: String,
    pub email: String,
    pub password: String
}
```

在此之后，我们可以在`src/json_serialization/mod.rs`文件中声明新的用户模式`pub mod new_user;`。一旦我们的模式被定义，我们可以创建自己的用户视图模块，其文件结构如下：

```rs
views
...
└── users
    ├── create.rs
    └── mod.rs
```

在我们的`users/create.rs`文件中，我们需要构建一个创建视图函数。首先，导入以下 crate：

```rs
use crate::diesel;
use diesel::prelude::*;
use actix_web::{web, HttpResponse, Responder};
use actix_web::HttpResponseBuilder;
use crate::database::DB;
use crate::json_serialization::new_user::NewUserSchema;
use crate::models::user::new_user::NewUser;
use crate::schema::users;
```

由于我们已经多次构建我们的视图，因此这些导入不应该令人惊讶。我们导入`diesel`宏和 crate，以便我们能够调用数据库。然后我们导入`actix_web`特性和结构体，以使数据能够流入和流出视图。接着我们导入我们的模式和结构体，以结构化我们接收和处理的数据。现在我们已经导入了正确的 crate，我们必须使用以下代码定义`create`视图函数：

```rs
pub async fn create(new_user: web::Json<NewUserSchema>,
                    db: DB) -> impl Responder {
    . . .
}
```

在这里，我们可以看到我们接受加载到`NewUserSchema`结构体的 JSON 数据。我们还使用`DB`结构体从连接池中建立数据库连接。在我们的`create`视图函数内部，我们从`NewUserSchema`结构体中提取所需的数据，使用以下代码创建一个`NewUser`结构体：

```rs
let new_user = NewUser::new(
    new_user.name.clone(),
    new_user.email.clone(),
    new_user.password.clone()
);
```

我们必须克隆要传递给`NewUser`构造函数的字段，因为字符串没有实现`Copy`特性，这意味着我们必须手动完成此操作。然后我们创建数据库的`insert`命令并执行以下代码：

```rs
let insert_result = diesel::insert_into(users::table)
                            .values(&new_user)
                            .execute(&db.connection);
```

这返回一个`Result`结构体。然而，我们并不直接`unwrap`它。可能存在冲突。例如，我们可能正在尝试插入一个已经存在于数据库中的用户名或电子邮件的新用户。然而，我们不想让它仅仅出错。这是一个我们预期作为我们已经实现了唯一的用户名和电子邮件约束的边缘情况。如果在视图执行过程中发生了一个合法的错误，我们需要知道它。因此，我们必须给边缘情况提供响应代码。因此，我们匹配插入的结果，并使用以下代码返回适当的响应代码：

```rs
match insert_result {
    Ok(_) => HttpResponse::Created(),
    Err(_) => HttpResponse::Conflict()
}
```

在这里，我们已经建立了一个数据库连接，从 JSON 体中提取了字段，创建了一个新的`NewUser`结构体，然后将其插入到数据库中。与其它视图相比，这里有一个细微的差别。在返回响应中，我们必须先`await`然后`unwrap`它。这是因为我们不返回 JSON 体。因此，`HttpResponse::Ok()`仅仅是一个构建器结构体。

现在我们已经构建了创建视图，我们需要在`views/users/mod.rs`文件中定义我们的视图工厂，如下所示：

```rs
mod create;
use actix_web::web::{ServiceConfig, post, scope};
pub fn user_views_factory(app: &mut ServiceConfig) {
    app.service(
        scope("v1/user")
        .route("create", post().to(create::create))
    );
}
```

再次，由于我们一直在定期构建视图，这些内容对你来说不应该感到意外。如果确实如此，建议你阅读*第三章*中“使用 Actix Web 框架管理视图”部分，*处理 HTTP 请求*，以获得清晰度。现在，我们的主视图工厂在`views/mod.rs`文件中应该看起来如下：

```rs
mod auth;
mod to_do;
mod app;
mod users;
use auth::auth_views_factory;
use to_do::to_do_views_factory;
use app::app_views_factory;
use users::user_views_factory;
use actix_web::web::ServiceConfig;
pub fn views_factory(app: &mut ServiceConfig) {
    auth_views_factory(app);
    to_do_views_factory(app);
    app_views_factory(app);
    user_views_factory(app);
}
```

现在我们已经注册了我们的用户视图，我们可以运行我们的应用程序，并使用以下 Postman 调用创建我们的用户：

![图 7.4 – 向我们的创建用户端点发送邮递员调用](img/Figure_7.4_B18722.jpg)

图 7.4 – 向我们的创建用户端点发送邮递员调用

因此，我们应该得到一个`201`创建响应。如果我们再次调用完全相同的调用，我们应该得到一个`409`冲突。因此，我们应该期待我们的新用户已经被创建。根据*第六章*中“使用 Diesel 连接 PostgreSQL”部分所涵盖的步骤，*使用 PostgreSQL 进行数据持久化*，我们可以在我们的 Docker 容器中检查数据库，以下是我们得到的结果：

```rs
 id |    name     |       email
----+-------------+-------------------
  1 | placeholder | placeholder email
  2 | maxwell     | test@gmail.com
                           password
-------------------------------------------------------------
 placeholder password
 $2b$12$jlfLwu4AHjrvTpZrB311Y.W0JulQ71WVy2g771xl50e5nS1UfqwQ.
              unique_id
--------------------------------------
 placeholder unique id
 543b7aa8-e563-43e0-8f62-55211960a604
```

在这里，我们可以看到在我们的迁移中创建的初始用户。然而，我们也可以看到我们通过视图创建的用户。这里有一个散列密码和唯一的 ID。从这一点我们可以看出，我们永远不应该直接创建我们的用户；我们只应该通过`NewUser`结构体所属的构造函数创建用户。

在我们应用程序的上下文中，我们实际上并不需要一个唯一的 ID。然而，在更广泛的情况下，当使用多个服务器和数据库时，一个唯一的 ID 可能变得很有用。我们还必须注意，我们第二个冲突响应是正确的；第三个副本创建用户调用，并没有将副本用户插入到数据库中。

使用这个方法，我们的应用程序正在正常运行，因为现在有一个用户表，其中包含与待办事项关联的用户模型。因此，我们可以创建其他具有关系和结构迁移的数据表，以便它们可以无缝升级和降级。我们也已经涵盖了如何验证和创建密码。然而，我们实际上并没有编写任何检查用户是否传递正确凭证的代码。在下一节中，我们将致力于验证用户并拒绝不包含正确凭证的请求。

# 验证我们的用户

当涉及到验证我们的用户时，我们已经构建了一个结构，用于从 HTTP 请求的头部提取消息。我们现在已经到了可以真正利用这种提取，将有关用户的数据存储在头部的时候了。目前，没有任何东西阻止我们在每个 HTTP 请求的头部存储用户名、ID 和密码，以便我们可以验证每一个。然而，这是一个糟糕的做法。如果有人拦截请求或获取存储在浏览器中以方便此操作的数据，那么账户就会受到损害，黑客可以随心所欲地做任何事情。相反，我们将对数据进行混淆，如下面的图所示：

![图 7.5 – 验证请求的步骤](img/Figure_7.5_B18722.jpg)

图 7.5 – 验证请求的步骤

在**图 7.5**中，我们可以看到我们使用一个密钥将有关用户的结构化数据序列化为字节形式的令牌。然后我们将令牌给用户，让他们存储在浏览器中。当用户想要发送授权请求时，用户必须在请求的头部发送令牌。然后我们的服务器使用密钥将令牌反序列化为有关用户的结构化数据。这个过程所使用的算法是任何人都可以获得的标准哈希算法。因此，我们有一个定义的密钥，以保持令牌在野外安全。为了我们的应用程序执行图 7.5 中概述的过程，我们可能需要重写大部分`src/jwt.rs`文件，包括`JwToken`结构体。在我们开始之前，我们需要使用以下代码更新我们的`Cargo.toml`依赖项：

```rs
[dependencies]
. . .
chrono = {version = "0.4.19", features = ["serde"]}
. . .
jsonwebtoken = "8.1.0"
```

我们可以看到，我们已经将`serde`功能添加到`chrono`包中，并添加了`jsonwebtoken`包。为了重建`JwToken`结构体，我们需要在`src/jwt.rs`文件中导入以下内容：

```rs
use actix_web::dev::Payload;
use actix_web::{Error, FromRequest, HttpRequest};
use actix_web::error::ErrorUnauthorized;
use futures::future::{Ready, ok, err};
use serde::{Deserialize, Serialize};
use jsonwebtoken::{encode, decode, Algorithm, Header,
                   EncodingKey, DecodingKey, Validation};
use chrono::{DateTime, Utc};
use chrono::serde::ts_seconds;
use crate::config::Config;
```

我们可以看到，我们导入了`actix_web`特性和结构体，以启用请求和响应的处理。然后我们导入`futures`，以便我们能够在 HTTP 请求击中视图之前拦截它。然后我们导入`serde`和`jsonwebtoken`，以启用数据到和从令牌的序列化和反序列化。然后我们导入`chrono` crate，因为我们想记录这些令牌何时被铸造。我们还需要序列化的密钥，我们从这个配置文件中获取它，这就是为什么我们导入`Config`结构体的原因。现在我们已经导入了所有需要的特性和结构体，我们可以用以下代码编写我们的令牌结构体：

```rs
#[derive(Debug, Serialize, Deserialize)]
pub struct JwToken {
    pub user_id: i32,
    #[serde(with = "ts_seconds")]
    pub minted: DateTime<Utc>
}
```

在这里，我们可以看到我们有用户的 ID，我们还有令牌创建的日期和时间。我们还用`serde`宏装饰我们的`minted`字段，以说明我们将如何序列化`datetime`字段。现在我们已经有了令牌所需的数据，我们可以继续定义序列化函数，以下代码：

```rs
impl JwToken {
    pub fn get_key() -> String {
        . . .
    }
    pub fn encode(self) -> String {
        . . .
    }
    pub fn new(user_id: i32) -> Self {
        . . .
    }
    pub fn from_token(token: String) -> Option<Self> {
        . . .
    }
}
```

我们可以用以下项目符号解释前面每个函数的作用：

+   `get_key`：从`config.yml`文件获取序列化和反序列化的密钥

+   `encode`：将`JwToken`结构体的数据编码为令牌

+   `new`：创建一个新的`JwToken`结构体

+   `from_token`：从令牌创建一个`JwToken`结构体。如果反序列化失败，它返回`None`，因为反序列化可能会失败。

一旦我们构建了前面的函数，我们的`JwToken`结构体将能够根据我们的需要处理令牌。我们用以下代码完善`get_key`函数：

```rs
pub fn get_key() -> String {
    let config = Config::new();
    let key_str = config.map.get("SECRET_KEY")
                            .unwrap().as_str()
                            .unwrap();
    return key_str.to_owned()
}
```

在这里，我们可以看到我们从配置文件中加载密钥。因此，我们需要将密钥添加到`config.yml`文件中，这样我们的文件看起来就像这样：

```rs
DB_URL: postgres://username:password@localhost:5433/to_do
SECRET_KEY: secret
```

如果我们的服务器在生产环境中，我们应该有一个更好的密钥。然而，对于本地开发，这将足够好。现在我们已经从配置文件中提取了密钥，我们可以用以下代码定义我们的`encode`函数：

```rs
pub fn encode(self) -> String {
    let key = EncodingKey::
              from_secret(JwToken::get_key().as_ref());
    let token = encode(&Header::default(), &self,
                       &key).unwrap();
    return token
}
```

在这里，我们可以看到我们使用配置文件中的密钥定义了一个编码密钥。然后我们使用这个密钥将`JwToken`结构体的数据编码成令牌并返回它。现在我们能够编码我们的`JwToken`结构体，当我们需要时，我们将需要创建新的`JwToken`结构体，这可以通过以下`new`函数实现：

```rs
pub fn new(user_id: i32) -> Self {
    let timestamp = Utc::now();
    return JwToken { user_id, minted: timestamp};
}
```

使用构造函数，我们知道我们的`JwToken`何时被铸造。如果我们想的话，这可以帮助我们管理用户会话。例如，如果令牌的年龄超过我们认为合适的阈值，我们可以强制进行另一次登录。

现在，我们只有`from_token`函数，其中我们使用以下代码从令牌中提取数据：

```rs
pub fn from_token(token: String) -> Option<Self> {
    let key = DecodingKey::from_secret(
                JwToken::get_key().as_ref()
              );
    let token_result = decode::<JwToken>(
                        &token, &key,
                        &Validation::new(Algorithm::HS256)
                        );
    match token_result {
        Ok(data) => {
            Some(data.claims)
        },
        Err(_) => {
            return None
        }
    }
}
```

在这里，我们定义了一个解码密钥，然后使用它来解码令牌。然后我们使用`data.claims`返回`JwToken`。现在，我们的`JwToken`结构体可以创建、编码成令牌，并从令牌中提取。现在，我们只需要在视图加载之前从 HTTP 请求的头中提取它，以下是一个概要：

```rs
impl FromRequest for JwToken {
    type Error = Error;
    type Future = Ready<Result<JwToken, Error>>;
    fn from_request(req: &HttpRequest,
                    _: &mut Payload) -> Self::Future {
        . . .
    }
}
```

我们已经多次实现了`FromRequest`特质，用于数据库连接和之前为`JwToken`结构体实现的实现。在`from_request`函数内部，我们使用以下代码从头中提取令牌：

```rs
match req.headers().get("token") {
    Some(data) => {
        . . .
    },
    None => {
        let error = ErrorUnauthorized(
                    "token not in header under key 'token'"
                    );
        return err(error)
    }
}
```

如果令牌不在头中，我们直接返回`ErrorUnauthorized`，完全避免调用视图。如果我们能够从头中提取令牌，我们可以使用以下代码来处理它：

```rs
Some(data) => {
    let raw_token = data.to_str()
                        .unwrap()
                        .to_string();
    let token_result = JwToken::from_token(
                                raw_token
                            );
    match token_result {
        Some(token) => {
            return ok(token)
        },
        None => {
            let error = ErrorUnauthorized(
                            "token can't be decoded"
                        );
            return err(error)
        }
    }
},
```

在这里，我们将从头中提取的原始令牌转换为字符串。然后我们反序列化令牌并将其加载到`JwToken`结构体中。然而，如果由于提供了伪造的令牌而失败，我们返回一个`ErrorUnauthorized`错误。现在我们的认证已经完全工作；然而，我们将无法做任何事情，因为我们没有有效的令牌，如下面的图所示：

![图 7.6 – 认证阻止请求](img/Figure_7.6_B18722.jpg)

图 7.6 – 认证阻止请求

在下一节中，我们将构建登录 API 端点，以便我们能够与受保护的端点交互。

# 管理用户会话

对于我们的用户，我们必须让他们能够登录。这意味着我们必须创建一个端点来验证他们的凭据，然后生成一个 JWT 并将其通过响应头返回给用户。我们的第一步是在`src/json_serialization/login.rs`文件中定义一个登录模式，以下代码如下：

```rs
use serde::Deserialize;
#[derive(Deserialize)]
pub struct Login {
    pub username: String,
    pub password: String
}
```

我们必须记住在`src/json_serialization/mod.rs`文件中使用`pub mod login;`代码行来注册它。一旦我们这样做，我们就可以构建我们的登录端点了。我们可以通过编辑我们在*第三章*中创建的`src/views/auth/login.rs`文件来实现，该文件位于*“使用 Actix Web 框架管理视图”*部分，它声明了我们的基本登录视图。这仅仅返回一个字符串。

现在，我们可以开始重构这个视图，通过定义所需的导入，如下面的代码所示：

```rs
use crate::diesel;
use diesel::prelude::*;
use actix_web::{web, HttpResponse, Responder};
use crate::database::DB;
use crate::models::user::user::User;
use crate::json_serialization::login::Login;
use crate::schema::users;
use crate::jwt::JwToken;
```

在这个阶段，我们可以浏览一下导入，了解我们将要做什么。我们将从请求体中提取用户名和密码。然后我们将连接到数据库以检查用户名和密码，接着使用`JwToken`结构体创建一个将被返回给用户的令牌。我们可以在同一文件中使用以下代码来初步布局视图的轮廓：

```rs
. . .
Use std::collections::HashMap;
pub async fn login(credentials: web::Json<Login>,
                   db: DB) -> impl HttpResponse {
    . . .
}
```

在这里，我们可以看到我们从传入请求的正文接受登录凭证，并为视图从连接池中准备一个数据库连接。然后我们可以从请求体中提取所需的详细信息，并使用以下代码进行数据库调用：

```rs
let password = credentials.password.clone();
let users = users::table
    .filter(users::columns::username.eq(
        credentials.username.clone())
    ).load::<User>(&db.connection).unwrap();
```

现在，我们必须检查我们是否从数据库调用中得到了预期的结果，如下面的代码所示：

```rs
if users.len() == 0 {
    return HttpResponse::NotFound().await.unwrap()
} else if users.len() > 1 {
    return HttpResponse::Conflict().await.unwrap()
}
```

在这里，我们已经进行了一些初步的返回。如果没有用户，我们将返回一个`not found`响应代码。这是我们时不时都会期待的事情。然而，如果有多个用户使用相同的用户名，我们需要返回一个不同的代码。

由于显示的独特约束，有些事情非常不对劲。未来的迁移脚本可能会撤销这些唯一约束，或者用户查询可能会意外地被更改。如果发生这种情况，我们需要立即知道这一点，因为违反我们约束的损坏数据可能会导致我们的应用程序以难以调试的意外方式运行。

现在我们已经确认检索到了正确的用户数量，我们可以有信心地获取索引为零的唯一用户，并检查他们的密码是否可接受，如下所示：

```rs
match users[0].verify(password) {
    true => {
        let token = JwToken::new(users[0].id);
        let raw_token = token.encode();
        let mut body = HashMap::new();
      body.insert("token", raw_token);
        HttpResponse::Ok().json(body)
    },
    false => HttpResponse::Unauthorized()
}
```

在这里，我们可以看到我们使用了`verify`函数。如果密码匹配，我们就使用 ID 生成一个 token，并将其返回给用户作为正文。如果密码不正确，我们则返回一个未授权代码。

在我们的注销方面，我们将采取一种更轻量级的方法。在我们的注销视图中，我们只需运行两行 JavaScript 代码。一行是将用户 token 从本地存储中移除，然后恢复用户到主视图。HTML 可以托管在打开时立即运行的 JavaScript。因此，我们可以通过在`src/views/auth/logout.rs`文件中放置以下代码来实现这一点：

```rs
use actix_web::HttpResponse;
pub async fn logout() -> HttpResponse {
    HttpResponse::Ok()
        .content_type("text/html; charset=utf-8")
        .body("<html>\
                <script>\
                    localStorage.removeItem('user-token'); \
                    window.location.replace(
                        document.location.origin);\
                </script>\
              </html>")
}
```

由于这个视图已经注册，我们可以运行应用程序并使用 Postman 进行调用：

![Figure 7.7 – 使用 Postman 通过登录端点登录我们的应用程序](img/Figure_7.7_B18722.jpg)

Figure 7.7 – 使用 Postman 通过登录端点登录我们的应用程序

修改用户名将给我们一个`404-response`代码，而修改密码将给我们一个`401-response`代码。如果我们有正确的用户名和密码，我们将得到一个`200-response`代码，并且响应头中会有一个*token*，如图*7**.7*所示。然而，如果我们想在响应头中使用我们的*token*，我们将收到一个`token can't be decoded`消息。在下文中，我们将清理我们的认证要求。

# 清理认证要求

在本节中，我们在开始配置前端以处理这些认证过程之前，将清理我们的 Rust 服务器以实现认证。为了保持章节的连贯性，我们并没有定期进行“维护”。现在，我们将更新我们的`to_do`视图。我们可以从更新带有认证要求的`create`视图开始。为此，`src/views/to_do/create.rs`文件中`create`视图的函数签名应如下所示：

```rs
. . .
use crate::jwt::JwToken;
use crate::database::DB
pub async fn create(token: JwToken,
                    req: HttpRequest, db: DB) -> HttpResponse {
    . . .
```

在创建新项目时，我们也必须使用来自令牌的 ID 更新用户 ID，代码如下：

```rs
if items.len() == 0 {
    let new_post = NewItem::new(title, token.user_id);
    let _ = diesel::
            insert_into(to_do::table).values(&new_post)
        .execute(&db.connection);
}
Return HttpResponse::Ok().json(
    ToDoItems::get_state(token.user_id)
)
```

在我们的`delete`视图中，我们必须确保我们正在删除请求用户所属的待办事项。如果我们不使用用户 ID 添加过滤器，待办事项的删除将是随机的。这个过滤器可以添加到我们的`src/views/to_do/delete.rs`文件中，代码如下：

```rs
. . .
Use crate::database::DB;
. . .
pub async fn delete(to_do_item: web::Json<ToDoItem>,
                    token: JwToken, db: DB) -> HttpResponse {
    let items = to_do::table
        .filter(to_do::columns::title.eq(
                    &to_do_item.title.as_str())
                )
        .filter(to_do::columns::user_id.eq(&token.user_id))
        .order(to_do::columns::id.asc())
        .load::<Item>(&db.connection)
        .unwrap();
    let _ = diesel::delete(&items[0]).execute(&db.connection);
    return HttpResponse::Ok().json(ToDoItems::get_state(
        token.user_id
    ))
}
```

我们可以看到，在执行数据库查询时，`filter`函数可以简单地串联。考虑到我们对`delete`视图所做的工作，你认为我们将在`src/views/to_do/edit.rs`文件中如何升级我们的认证要求？在这个阶段，我鼓励你自己尝试更新`edit`视图，因为方法类似于我们的`delete`视图升级。一旦你完成了这个，你的`edit`视图应该如下所示：

```rs
pub async fn edit(to_do_item: web::Json<ToDoItem>,
                  token: JwToken, db: DB) -> HttpResponse {
    let results = to_do::table.filter(to_do::columns::title
                              .eq(&to_do_item.title))
                              .filter(to_do::columns::user_
                                      id
                              .eq(&token.user_id));
    let _ = diesel::update(results)
        .set(to_do::columns::status.eq("DONE"))
        .execute(&db.connection);
    return HttpResponse::Ok().json(ToDoItems::get_state(
                                   token.user_id
    ))
}
```

现在我们已经更新了特定的视图，现在我们可以继续到`get`视图，它也有应用于所有其他视图的`get_state`函数。`src/views/to_do/get.rs`文件中的`get`视图现在如下所示：

```rs
use actix_web::Responder;
use crate::json_serialization::to_do_items::ToDoItems;
use crate::jwt::JwToken;
pub async fn get(token: JwToken) -> impl Responder {
    ToDoItems::get_state(token.user_id)
}
```

现在，前述代码中的所有内容都不应该令人惊讶。我们可以看到，我们将用户 ID 传递给`ToDoItems::get_state`函数。你必须记住在`ToDoItems::get_state`函数实现的所有地方填写用户 ID，这包括所有待办事项视图。然后我们可以在`src/json_serialization/to_do_items.rs`文件中重新定义我们的`ToDoItems::get_state`函数，代码如下：

```rs
. . .
use crate::database::DBCONNECTION;
. . .
impl ToDoItems {
    . . .
    pub fn get_state(user_id: i32) -> ToDoItems {
        let connection = DBCONNECTION.db_connection.get()
                         .unwrap();
        let items = to_do::table
                    .filter(to_do::columns::user_id.eq
                           (&user_id))
                    .order(to_do::columns::id.asc())
                    .load::<Item>(&connection)
                    .unwrap();
        let mut array_buffer = Vec::
                               with_capacity(items.len());
        for item in items {
            let status = TaskStatus::from_string(
            &item.status.as_str().to_string());
            let item = to_do_factory(&item.title, status);
            array_buffer.push(item);
        }
        return ToDoItems::new(array_buffer)
    }
}
```

在这里，我们可以看到我们已经更新了数据库连接和用户 ID 的过滤器。我们现在已经更新了我们的代码以适应不同的用户。还有一个变化我们必须做出。因为我们将在 React 应用程序中编写前端代码，我们将尽量使 React 编码尽可能简单，因为 React 开发本身就是一本大书。为了避免过度复杂化前端开发中的标题提取和`GET`帖子使用 Axios，我们将在登录时添加一个`Post`方法，并通过正文返回令牌。这又是一个尝试自己解决问题的好机会，因为我们已经涵盖了实现这一目标所需的所有概念。

如果你尝试自己解决这个问题，它应该如下所示。首先，我们在`src/json_serialization/login_response.rs`文件中定义一个响应结构体，代码如下：

```rs
use serde::Serialize;
#[derive(Serialize)]
pub struct LoginResponse {
    pub token: String
}
```

我们记得在`src/json_serialization/mod.rs`文件中声明前面的结构体，通过添加`pub mod login_response`。我们现在转到`src/views/auth/login.rs`，在`login`函数中有以下`return`语句：

```rs
match users[0].clone().verify(credentials.password.clone()) {
    true => {
        let user_id = users[0].clone().id;
        let token = JwToken::new(user_id);
        let raw_token = token.encode();
        let response = LoginResponse{token:
                                     raw_token.clone()};
        let body = serde_json::
                   to_string(&response).unwrap();
        HttpResponse::Ok().append_header(("token",
                           raw_token)).json(&body)
    },
    false => HttpResponse::Unauthorized().finish()
}
```

注意

你可能已经注意到我们对未授权的部分做了一些小的改动：

**HttpResponse::Unauthorized().finish()**

这是因为我们将视图函数的`return`类型从`HttpResponse`结构体切换了，从而得到了以下函数签名：

**(credentials: web::Json<Login>, db: DB) ->** **HttpResponse**

我们必须进行切换，因为将`json`函数添加到我们的响应中，将我们的响应从`HttpResponseBuilder`转换为`HttpResponse`。一旦调用了`json`函数，就不能再使用`HttpResponseBuilder`。回到未授权的响应构建器，我们可以推断出`finish`函数将`HttpResponseBuilder`转换为`HttpResponse`。我们也可以通过使用`await`将我们的`HttpResponseBuilder`转换为`HttpResponse`，如下所示：

**HttpResponse::Unauthorized().await.unwrap()**

在这里，我们可以看到我们在头部和体中返回了令牌。这将给我们编写前端代码时的灵活性和便捷性。然而，必须强调的是，这并不是最佳实践。我们正在实施将令牌返回到体和头部的做法，以使前端开发部分保持简单。我们可以在`src/views/auth/mod.rs`文件中为我们的登录视图启用`POST`方法，如下所示：

```rs
mod login;
mod logout;
use actix_web::web::{ServiceConfig, get, post, scope};
pub fn auth_views_factory(app: &mut ServiceConfig) {
    app.service(
            scope("v1/auth")
            .route("login", get().to(login::login))
            .route("login", post().to(login::login))
            .route("logout", get().to(logout::logout))
    );
}
```

我们可以看到我们只是在同一个`login`视图中堆叠了一个`get`函数。现在，`POST`和`GET`都可用于我们的登录视图。我们现在可以进入下一节，配置我们的认证令牌，以便它们可以过期。我们希望令牌过期以提高我们的安全性。如果一个令牌被泄露，恶意行为者获取了令牌，他们可以无限制地做他们想做的事情，而无需登录。然而，如果我们的令牌过期，那么恶意行为者只有有限的时间窗口，在令牌过期之前。

# 配置认证令牌的过期时间

如果我们尝试使用从登录令牌中获取的有效令牌在我们的现在受保护的端点上执行 API 调用，我们将得到一个未授权错误。如果我们插入一些`print`语句，在解码令牌失败时，我们会得到以下错误：

```rs
missing required claim: exp
```

这意味着在我们的`JwToken`结构体中没有名为`exp`的字段。如果我们参考[`docs.rs/jsonwebtoken/latest/jsonwebtoken/fn.encode.html`](https://docs.rs/jsonwebtoken/latest/jsonwebtoken/fn.encode.html)上的`jsonwebtoken`文档，我们可以看到`encode`指令从未提到`exp`：

```rs
use serde::{Deserialize, Serialize};
use jsonwebtoken::{encode, Algorithm, Header, EncodingKey};
#[derive(Debug, Serialize, Deserialize)]
struct Claims {
   sub: String,
   company: String
}
let my_claims = Claims {
    sub: "b@b.com".to_owned(),
    company: "ACME".to_owned()
};
// my_claims is a struct that implements Serialize
// This will create a JWT using HS256 as algorithm
let token = encode(&Header::default(), &my_claims,
&EncodingKey::from_secret("secret".as_ref())).unwrap();
```

在这里，我们可以看到没有提及任何声明。然而，实际上当我们尝试反序列化我们的令牌时，`jsonwebtoken` crate 中的`decode`函数会自动寻找`exp`字段来确定令牌何时应该过期。我们正在探索这一点，因为官方文档和稍微有些令人困惑的错误信息可能会让你浪费数小时试图弄清楚发生了什么。考虑到这一点，我们必须回到我们的`src/jwt.rs`文件进行一些修改，但这是最后一次，我保证，这并不是一个完整的重写。首先，我们确保以下内容与`src/jwt.rs`文件中已有的内容一起导入：

```rs
. . .
use jsonwebtoken::{encode, decode, Header,
                   EncodingKey, DecodingKey,
                   Validation};
use chrono::Utc;
. . .
```

我们可以确保使用以下代码将`exp`字段写入我们的`JwToken`结构：

```rs
#[derive(Debug, Serialize, Deserialize)]
pub struct JwToken {
    pub user_id: i32,
    pub exp: usize,
}
```

我们现在必须重写我们的`JwToken`结构的`new`构造函数。在`new`函数中，我们必须定义新铸造的`JwToken`结构何时过期。这必须变化；作为一个开发者，你可能想要调整超时所需的时间。记住，每次我们更改 Rust 代码时都必须重新编译；因此，在配置文件中定义超时周期是有意义的。考虑到超时周期的变化，我们的`new`函数具有以下形式：

```rs
pub fn new(user_id: i32) -> Self {
    let config = Config::new();
    let minutes = config.map.get("EXPIRE_MINUTES")
                            .unwrap().as_i64().unwrap();
    let expiration = Utc::now()
    .checked_add_signed(chrono::Duration::minutes(minutes))
                            .expect("valid timestamp")
                            .timestamp();
    return JwToken { user_id, exp: expiration as usize };
}
```

我们可以看到我们定义了分钟数。然后我们将过期时间转换为`usize`，然后构建我们的`JwToken`结构。现在我们有了这个，我们需要更具体地返回错误类型，因为这可能是令牌解码错误，或者令牌可能已过期。我们使用以下代码处理解码令牌时的不同类型的错误：

```rs
pub fn from_token(token: String) -> Result<Self, String> {
    let key = DecodingKey::
              from_secret(JwToken::get_key().as_ref());
    let token_result = decode::<JwToken>(&token.as_str(),
                              &key,&Validation::default());
    match token_result {
        Ok(data) => {
            return Ok(data.claims)
        },
        Err(error) => {
            let message = format!("{}", error);
            return Err(message)
        }
    }
}
```

在这里，我们可以看到我们已经从返回`Option`切换到了`Result`。我们切换到`Result`是因为我们在`FromRequest`特质的实现中返回的消息可以被我们的`from_request`函数消化和处理。`from_request`函数中的其余代码保持不变。我们进行更改的地方是检查消息是否存在错误，并使用以下代码向前端返回不同的消息：

```rs
fn from_request(req: &HttpRequest,
                _: &mut Payload) -> Self::Future {
    match req.headers().get("token") {
        Some(data) => {
            let raw_token = data.to_str()
                                .unwrap()
                                .to_string();
            let token_result = JwToken::
                        from_token(raw_token);
            match token_result {
                Ok(token) => {
                    return ok(token)
                },
                Err(message) => {
                    if message == "ExpiredSignature"
                                  .to_owned() {
                        return err(
                        ErrorUnauthorized("token expired"))
                    }
                    return err(
                    ErrorUnauthorized("token can't be decoded"))
                }
            }
        },
        None => {
            return err(
            ErrorUnauthorized(
            "token not in header under key 'token'"))
        }
    }
}
```

带有细微的错误信息，我们的前端代码可以处理和适应，因为我们可以在前端更具体地处理错误。在前端更具体地处理错误可以帮助用户，提示他们出错的地方。然而，当涉及到身份验证时，确保不要透露太多，因为这也可能帮助试图获取未经授权访问的恶意行为者。我们现在有我们的登录和注销端点正在运行；我们还有在需要视图上的令牌授权。然而，如果我们想要标准用户与我们的应用程序交互，这并不很有用，因为他们不太可能使用 Postman。因此，在下一节中，我们必须将我们的登录/注销端点纳入前端。

# 将身份验证添加到我们的前端

我们集成了我们的登录功能。我们必须从在`src/components/LoginForm.js`文件中构建登录表单开始。首先，我们导入以下内容：

```rs
import React, {Component} from 'react';
import axios from 'axios';
import '../css/LoginForm.css';
```

本章的*附录*中提供了导入的 CSS 代码。我们在这里不会讲解它，因为它有很多重复的代码。您也可以从 GitHub 仓库下载 CSS 代码。有了这些导入，我们可以用以下代码构建我们的登录表单框架：

```rs
class LoginForm extends Component {
    state = {
        username: "",
        password: "",
    }
    submitLogin = (e) => {
        . . .
    }
    handlePasswordChange = (e) => {
        this.setState({password: e.target.value})
    }
    handleUsernameChange = (e) => {
        this.setState({username: e.target.value})
    }
    render() {
        . . .
    }
}
export default LoginForm;
```

在这里，我们可以看到我们跟踪`username`和`password`，它们会不断更新状态。记住，当状态更新时，我们执行`render`函数。这很强大，因为我们可以改变我们想要的任何东西。例如，如果`username`的长度超过一定长度，我们可以改变组件的颜色或移除按钮。我们不会自己做出剧烈的改变，因为这超出了本书的范围。现在我们已经定义了我们的框架，我们可以用以下代码来声明我们的`render`函数返回的内容：

```rs
<form className="login" onSubmit={this.submitLogin}>
    <h1 className="login-title">Login</h1>
    <input type="text" className="login-input"
    placeholder="Username"
    autoFocus onChange={this.handleUsernameChange}
           value={this.state.username} />
    <input type="password" className="login-input"
    placeholder="Password"
    onChange={this.handlePasswordChange}
           value={this.state.password} />
    <input type="submit" value="Lets Go"
    className="login-button" />
</form>
```

在这里，我们可以看到表单中有`username`和`password`字段，当有变化时，会执行`handleUsernameChange`和`handlePasswordChange`函数。当我们输入`username`和`password`时，我们需要通过`submitLogin`函数将这些字段提交到后端，我们可以在下面定义这个函数：

```rs
submitLogin = (e) => {
    e.preventDefault();
    axios.post("http://localhost:8000/v1/auth/login",
        {"username": this.state.username,
         "password": this.state.password},
        {headers: {"Access-Control-Allow-Origin": "*"}}
        )
        .then(response => {
            this.setState({username: "", password: ""});
            this.props.handleLogin(response.data["token"]);
        })
        .catch(error => {
            alert(error);
            this.setState({password: "", firstName: ""});
        });
}
```

在这里，我们可以看到我们将登录 API 调用的响应传递给一个我们通过 props 传递的函数。我们将在`src/app.js`文件中定义这个函数。如果有错误，我们将在一个 alert 中打印出来，告诉我们发生了什么。无论如何，我们将清空`username`和`password`字段。

现在我们已经定义了我们的登录表单，当我们需要用户登录时，我们将需要显示它。一旦用户登录，我们需要隐藏登录表单。在我们能够做到这一点之前，我们需要用以下代码将我们的登录表单导入到`src/app.js`文件中：

```rs
import LoginForm from "./components/LoginForm";
```

我们现在需要跟踪登录状态。为此，我们的`App`类的状态需要以下形式：

```rs
state = {
  "pending_items": [],
  "done_items": [],
  "pending_items_count": 0,
  "done_items_count": 0,
  "login_status": false,
}
```

我们正在跟踪我们的项目，但如果`login_status`是`false`，我们可以显示登录表单。一旦用户登录，我们可以将`login_status`设置为`true`，因此我们可以隐藏登录表单。现在我们已经记录了登录状态，我们可以更新`App`类的`getItems`函数：

```rs
getItems() {
  axios.get("http://127.0.0.1:8000/v1/item/get",
  {headers: {"token": localStorage.getItem("user-token")}})
  .then(response => {
      let pending_items = response.data["pending_items"]
      let done_items = response.data["done_items"]
      this.setState({
        "pending_items":
         this.processItemValues(pending_items),
        "done_items": this.processItemValues(done_items),
        "pending_items_count":
         response.data["pending_item_count"],
        "done_items_count":
         response.data["done_item_count"]
        })
  }).catch(error => {
      if (error.response.status === 401) {
        this.logout();
      }
  });
}
```

我们可以看到我们获取了令牌并将其放入头部。如果存在授权代码错误，我们将执行`App`类的`logout`函数。我们的`logout`函数的形式如下：

```rs
logout() {
  localStorage.removeItem("token");
  this.setState({"login_status": false});
}
```

我们可以看到我们从本地存储中删除了令牌，并将我们的`login_status`设置为`false`。当尝试编辑待办事项时，如果出现错误，也需要执行这个`logout`函数，因为我们必须记住我们的令牌可能会过期，所以它可能发生在任何地方，我们必须提示另一个登录。这意味着我们必须将`logout`函数通过以下代码传递给`ToDoItem`组件：

```rs
processItemValues(items) {
  let itemList = [];
  items.forEach((item, _)=>{
      itemList.push(
          <ToDoItem key={item.title + item.status}
                    title={item.title}
                    status={item.status}
                    passBackResponse={this.handleReturnedState}
                    logout={this.logout}/>
      )
  })
  return itemList
}
```

一旦我们将`logout`函数传递给`ToDoItem`组件，我们就可以更新`src/components/ToDoItem.js`文件中编辑待办事项的 API 调用，以下代码：

```rs
sendRequest = () => {
    axios.post("http://127.0.0.1:8000/v1/item/" +
                this.state.button,
        {
            "title": this.state.title,
            "status": this.inverseStatus(this.state.status)
        },
    {headers: {"token": localStorage.getItem(
         "user-token")}})
        .then(response => {
            this.props.passBackResponse(response);
        }).catch(error => {
            if (error.response.status === 401) {
                this.props.logout();
            }
    });
}
```

在这里，我们可以看到我们通过头部传递令牌从本地存储到 API 调用。如果我们收到未授权的状态，我们就会执行通过 props 传入的`logout`函数。

现在，我们回到`src/app.js`文件，总结我们应用程序的功能。记住，我们的应用程序在首次访问时需要加载数据。当我们的应用程序最初加载时，我们必须考虑以下代码中的本地存储中的令牌：

```rs
componentDidMount() {
  let token = localStorage.getItem("user-token");
  if (token !== null) {
      this.setState({login_status: true});
      this.getItems();
  }
}
```

现在我们的应用程序只有在有令牌的情况下才会从后端获取项目。我们必须在用`render`函数封装我们的应用程序之前只处理登录。你已经看到了我们如何使用本地存储处理令牌。在这个时候，你应该能够为`App`类构建`handleLogin`函数。如果你尝试编写自己的函数，它应该看起来像以下代码：

```rs
handleLogin = (token) => {
  localStorage.setItem("user-token", token);
  this.setState({"login_status": true});
  this.getItems();
}
```

我们现在处于为`App`类定义`render`函数的阶段。如果我们的登录状态是`true`，我们可以使用以下代码显示应用程序提供的一切：

```rs
if (this.state.login_status === true) {
    return (
        <div className="App">
            <div className="mainContainer">
                <div className="header">
                    <p>complete tasks:
                       {this.state.done_items_count}</p>
                    <p>pending tasks:
                       {this.state.pending_items_count}</p>
                </div>
                <h1>Pending Items</h1>
                {this.state.pending_items}
                <h1>Done Items</h1>
                {this.state.done_items}
                <CreateToDoItem
                 passBackResponse={this.handleReturnedState}/>
            </div>
        </div>
    )
}
```

这里没有太多新内容。然而，如果我们的登录状态不是`true`，我们就可以使用以下代码显示登录表单：

```rs
else {
    return (
        <div className="App">
            <div className="mainContainer">
                <LoginForm handleLogin={this.handleLogin}
                />
            </div>
        </div>
    )
}
```

正如我们所见，我们已经将`handleLogin`函数传递给了`LoginForm`组件。有了这个，我们就准备好运行应用程序了。我们的第一个视图看起来如下：

![图 7.8 – 应用程序的登录和加载视图](img/Figure_7.8_B18722.jpg)

图 7.8 – 应用程序的登录和加载视图

一旦我们输入正确的凭证，我们就能访问应用程序并交互待办事项。我们的应用程序本质上已经可以工作了！

# 摘要

在本章中，我们构建了用户数据模型结构，并将它们绑定到我们的迁移中的待办事项数据模型。然后我们通过在 SQL 文件中执行多个步骤来进一步深入我们的迁移，以确保迁移顺利运行。我们还探讨了如何向某些字段添加唯一约束。

在数据库中定义了我们的数据模型之后，我们在将它们与存储的用户一起存储在数据库之前，对一些密码进行了散列。然后我们创建了一个 JWT 结构，以便我们的用户可以在他们的浏览器中存储 JWT，这样他们就可以在发起 API 调用时提交它们。然后我们探讨了如何在 JavaScript 和 HTML 存储中重定向 URL，以便前端可以确定用户是否有凭证，在考虑发送 API 调用到项目之前。

我们在这里所做的是通过迁移更改数据库，以便我们的应用程序可以管理处理更多复杂性的数据模型。然后我们利用前端存储来允许用户传递凭证。这对于你将开始的任何其他 Rust 网络项目都直接适用。大多数网络应用程序都需要某种形式的身份验证。

在下一章中，我们将探讨**REST API**实践，我们将标准化接口、缓存和日志。

# 问题

1.  与服务器端代码相比，在 SQL 中定义唯一约束有什么优势？

1.  用户拥有 JWT 而不是存储密码的主要优势是什么？

1.  用户如何在前端存储 JWT？

1.  一旦我们验证 JWT 是可用的，JWT 在视图中有什么用？

1.  当用户点击端点时，更改前端数据并将其重定向到另一个视图的最小方法是什么？

1.  为什么在用户登录时使用一系列不同的响应代码比仅仅表示登录成功或失败更有用？

# 答案

1.  直接在数据库上添加唯一约束确保了无论数据操作是通过迁移还是服务器请求完成的，这一标准都得到执行。这也保护我们免受在另一个端点添加新功能时忘记执行这一标准或代码在端点的后续更改中更改所造成的数据损坏。

1.  如果攻击者设法获取 JWT，并不意味着他们可以直接访问用户的密码。此外，如果令牌被刷新，那么攻击者对项目的访问时间有限。

1.  JWT 可以存储在本地 HTML 存储或 cookies 中。

1.  在散列令牌时，我们可以在其中存储多个数据点。因此，我们可以加密用户 ID。有了这个，我们可以提取用户 ID，用于与待办事项创建、删除或编辑相关的操作。

1.  我们返回一个包含 HTML/text 主体的`HttpResponse`结构，该主体包含一个包含几个 HTML 标签的字符串。在这些标签之间是几个脚本标签。在脚本标签之间，我们可以通过分号分割我们的 JavaScript 命令。然后我们可以直接修改 HTML 存储和窗口位置。

1.  数据库中数据损坏可能有多种原因，包括迁移中的更改。然而，可能存在一个错误，这不是用户的责任——例如，两个不同用户的重复用户名。这是一个违反了我们的唯一约束的错误。我们需要知道这种情况已经发生，以便我们可以纠正它。

# 进一步阅读

JWT 标准：[`tools.ietf.org/html/rfc7519`](https://tools.ietf.org/html/rfc7519)

# 附录

登录表单所使用的 CSS：

```rs
body {
    background: #2d343d;
}
.login {
    margin: 20px auto;
    width: 300px;
    padding: 30px 25px;
    background: white;
    border: 1px solid #c4c4c4;
    border-radius: 25px;
}
h1.login-title {
    margin: -28px -25px 25px;
    padding: 15px 25px;
    line-height: 30px;
    font-size: 25px;
    font-weight: 300;
    color: #ADADAD;
    text-align:center;
    background: #f7f7f7;
    border-radius: 25px 25px 0px 0px;
}
.login-input {
    width: 285px;
    height: 50px;
    margin-bottom: 25px;
    padding-left:10px;
    font-size: 15px;
    background: #fff;
    border: 1px solid #ccc;
    border-radius: 4px;
}
.login-input:focus {
    border-color:#6e8095;
    outline: none;
}
.login-button {
    width: 100%;
    height: 50px;
    padding: 0;
    font-size: 20px;
    color: #fff;
    text-align: center;
    background: #f0776c;
    border: 0;
    border-radius: 5px;
    cursor: pointer;
    outline:0;
}
.login-lost
{
    text-align:center;
    margin-bottom:0px;
}
.login-lost a
{
    color:#666;
    text-decoration:none;
    font-size:13px;
}
.loggedInTitle {
    font-family: "Helvetica Neue";
    color: white;
}
```
