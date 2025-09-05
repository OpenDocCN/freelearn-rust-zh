# 第六章：可变性、所有权和纯函数

Rust 在对象所有权方面引入了一些自己的新概念。这些安全措施可以保护开发者免受某些类别的错误，例如双重释放内存或悬挂指针，但有时也会创建一些感觉不合理的约束。函数式编程可能通过鼓励使用不可变数据和纯函数来帮助缓解一些这种冲突。

在本章中，我们将探讨一个所有权出错的情况。你将继承已被放弃且难以工作的代码。在本章中，你的任务是解决前一个团队未能克服的问题。为了实现这一点，你需要使用迄今为止所学的大部分知识，以及你对 Rust 中所有权的特定行为和约束的理解。

学习成果：

+   识别复杂所有权的反模式

+   学习复杂所有权的具体规则

+   使用不可变数据来防止所有权的反模式

+   使用纯函数来防止所有权的反模式

# 技术要求

运行提供的示例需要 Rust 的最近版本：

[`www.rust-lang.org/en-US/install.html`](https://www.rust-lang.org/en-US/install.html)

本章的代码也可在 GitHub 上找到：

[`github.com/PacktPublishing/Hands-On-Functional-Programming-in-RUST`](https://github.com/PacktPublishing/Hands-On-Functional-Programming-in-RUST)

每章的`README.md`文件中也包含了具体的安装和构建说明。

# 识别所有权的反模式

考虑以下情况。

恭喜你，你继承了遗留代码。负责为电梯开发特权访问模块的前一个团队已被转移到不同的项目。他们成功开发了与一系列微控制器接口的代码库。然而，在用 Rust 开发访问逻辑时，他们发现对象所有权非常复杂，并且无法开发与 Rust 兼容的软件。

在本章中，你的任务将是分析他们的代码，寻找可能的解决方案，然后创建一个库以支持电梯的特权访问。为了明确，特权访问指的是提供给紧急服务（如警察、消防员等）的覆盖代码和密钥。

# 检查微控制器驱动程序

微控制器驱动程序是用其他语言编写的，并通过**外部函数接口**（**FFI**）功能暴露给 Rust。FFI 是连接 Rust 代码到用其他语言编写的库的一种方式。以下是在`src/magic.rs`中定义的外部库符号和绑定。

此函数向库和子系统发出一个覆盖代码，如下所示：

```rs
fn issue_override_code(code: c_int)
```

当输入覆盖代码时，它将通过此函数暴露。上层应解释覆盖代码的含义，以便可能进入紧急操作模式或其他维护功能，如下所示：

```rs
fn poll_override_code() -> c_int
```

当建立覆盖模式并且紧急服务人员进入楼层时，将调用此方法。紧急模式下的楼层请求应优先于正常的`elevator`操作：

```rs
fn poll_override_input_floor()
```

来自`override`操作的错误代码将通过此函数暴露。例如无效的覆盖代码等问题将呈现给上层以决定如何响应：

```rs
fn poll_override_error() -> c_int
```

如果输入了覆盖代码，将创建一个授权的覆盖会话：

```rs
fn poll_override_session() -> *const c_void
```

在覆盖会话完成后，应释放资源并重置状态：

```rs
fn free_override_session(session: *const c_void)
```

如果启动了电梯的物理密钥访问，则此方法将暴露结果：

```rs
fn poll_physical_override_privileged_session() -> *const c_void
```

如果管理员启动了物理密钥访问，则此方法将暴露结果，如下所示：

```rs
fn poll_physical_override_admin_session() -> *const c_void
```

此函数将强制电梯进入手动操作模式：

```rs
fn override_manual_mode()
```

此函数将强制电梯进入正常操作模式：

```rs
fn override_normal_mode()
```

此函数将重置电梯状态：

```rs
fn override_reset_state()
```

此函数将在电梯控制面板上执行灯光的定时闪烁模式：

```rs
 fn elevator_display_flash(pattern: c_int)
```

此函数将切换电梯控制面板上按钮或其他符号的灯光：

```rs
fn elevator_display_toggle_light(light_id: c_int)
```

此函数将改变电梯控制面板上灯光的显示颜色：

```rs
fn elevator_display_set_light_color(light_id: c_int, color: int)
```

# 检查类型和特质定义

Rust 类型和特质的定义主要目的是封装库接口。让我们快速浏览一下`src/admin.rs`中定义的符号，以便熟悉库的预期工作方式。

# 定义`OverrideCode`枚举

`OverrideCode`枚举为从链接库的不同覆盖代码提供了类型安全的定义和命名。此代码将命名枚举值与返回或发送给 FFI 的数值枚举值关联。注意分配整数值给每个枚举元素的语法模式：

```rs
pub enum OverrideCode {
   IssueOverride = 1,
   IssuePrivileged = 2,
   IssueAdmin = 3,
   IssueInputFloor = 4,
   IssueManualMode = 5,
   IssueNormalMode = 6,
   IssueFlash = 7,
   IssueToggleLight = 8,
   IssueSetLightColor = 9,
}

pub fn toOverrideCode(i: i32) -> OverrideCode {
   match i {
      1 => OverrideCode::IssueOverride,
      2 => OverrideCode::IssuePrivileged,
      3 => OverrideCode::IssueAdmin,
      4 => OverrideCode::IssueInputFloor,
      5 => OverrideCode::IssueManualMode,
      6 => OverrideCode::IssueNormalMode,
      7 => OverrideCode::IssueFlash,
      8 => OverrideCode::IssueToggleLight,
      9 => OverrideCode::IssueSetLightColor,
      _ => panic!("Unexpected override code: {}", i)
   }
}
```

# 定义`ErrorCode`枚举

与`OverrideCode`类似，`ErrorCode`枚举为库中的每个错误代码定义了类型安全的标签。还有一个辅助函数可以将整数转换为枚举类型：

```rs
pub enum ErrorCode {
   DoubleAuthorize = 1,
   DoubleFree = 2,
   AccessDenied = 3,
}

pub fn toErrorCode(i: i32) -> ErrorCode {
   match i {
      1 => ErrorCode::DoubleAuthorize,
      2 => ErrorCode::DoubleFree,
      3 => ErrorCode::AccessDenied,
      _ => panic!("Unexpected error code: {}", i)
   }
}
```

# 定义`AuthorizedSession`结构体和析构函数

`AuthorizedSession`结构体封装了从库中获取的会话指针。此结构体还实现了`Drop`特质，当对象超出作用域时会被调用。这里`free_override_session`调用非常重要，应作为潜在问题源进行标注：

```rs
#[derive(Clone)]
pub struct AuthorizedSession
{
   session: *const c_void
}

impl Drop for AuthorizedSession {
   fn drop(&mut self) {
      unsafe {
         magic::free_override_session(self.session);
      }
   }
}
```

# 授权会话

授权会话有三个步骤：

1.  授权会话

1.  轮询并检索会话对象

1.  检查错误

这些函数的结果是`Result`对象，这将是本库中常见的模式：

```rs
pub fn authorize_override() -> Result<AuthorizedSession,ErrorCode>
{
   let session = unsafe {
      magic::issue_override_code(OverrideCode::IssueOverride as i32);
      magic::poll_override_session()
   };
   let session = AuthorizedSession {
      session: session
   };
   check_error(session)
}

pub fn authorize_privileged() -> Result<AuthorizedSession,ErrorCode>
{ ... }

pub fn authorize_admin() -> Result<AuthorizedSession,ErrorCode>
{ ... }
```

# 检查错误和重置状态

有两个简单的实用函数可用于重置状态和检查错误。代码在`unsafe`块中包装 FFI 函数，并将错误转换为`Result`值。代码如下：

```rs
pub fn reset_state()
{
   unsafe {
      magic::override_reset_state();
   }
}

pub fn check_error<T>(t: T) -> Result<T,ErrorCode>
{
   let err = unsafe {
      magic::poll_override_error()
   };
   if err==0 {
      Result::Ok(t)
   } else {
      Result::Err(toErrorCode(err))
   }
}
```

# 特权命令

在调用之前必须授权特权命令，否则命令将被拒绝。每次操作后都会检查错误，并返回一个`Result`值：

```rs
pub fn input_floor(floor: i32) -> Result<(),ErrorCode>
{
   unsafe {
      magic::override_input_floor(floor);
   }
   check_error(())
}

pub fn manual_mode() -> Result<(),ErrorCode>
{
   unsafe {
      magic::override_manual_mode();
   }
   check_error(())
}

pub fn normal_mode() -> Result<(),ErrorCode>
{
   unsafe {
      magic::override_normal_mode();
   }
   check_error(())
}
```

# 普通命令

普通命令不需要授权会话即可调用。每次调用后都会检查错误，并返回一个`Result`值：

```rs
pub fn flash(pattern: i32) -> Result<(),ErrorCode>
{
   unsafe {
      magic::elevator_display_flash(pattern);
   }
   check_error(())
}

pub fn toggle_light(light_id: i32) -> Result<(),ErrorCode>
{
   unsafe {
      magic::elevator_display_toggle_light(light_id);
   }
   check_error(())
}

pub fn set_light_color(light_id: i32, color: i32) -> Result<(),ErrorCode>
{
   unsafe {
      magic::elevator_display_set_light_color(light_id, color);
   }
   check_error(())
}
```

# 查询库和会话状态

有几个查询库和会话状态的函数可用，主要用于调试目的：

```rs
pub fn is_override() -> bool
{
   unsafe {
      magic::is_override() != 0
   }
}

pub fn is_privileged() -> bool
{
   unsafe {
      magic::is_privileged() != 0
   }
}

pub fn is_admin() -> bool
{
   unsafe {
      magic::is_admin() != 0
   }
}
```

# 检查外部库测试

之前的团队似乎非常自信于他们开发的库子系统；然而，他们发现 Rust 代码难以处理。测试使这个问题明显。两组测试似乎支持库按预期工作的观点，但 Rust 组件在边缘情况下失败。将责任交给你来收拾残局并挽救项目。

查看`src/tests/magic.rs`中的库测试，预期行为如下：

+   覆盖代码通过电梯控制面板或直接从软件发布到子系统

+   通过`poll`函数访问状态信息和授权会话

+   在其他人可以授权之前，必须释放授权会话

+   在覆盖模式下，可以发布特权命令，例如：

    +   将电梯切换到手动操作

    +   使用电梯显示屏进行通信

+   没有活跃会话，不得发布特权命令

所有库测试均通过，确认库在有限测试条件下的正确行为。还应注意的是，库在处理状态、事件和会话方面有点晦涩。这些模式在链接库中很常见，但要看到这些模式，让我们看看 Rust 中产生的代码：

# 发布覆盖代码

这组 FFI 函数测试确认发布的命令代码被库接收：

```rs
#[test]
fn issue_override_code() {
   unsafe {
      magic::override_reset_state();
      magic::issue_override_code(1);
      assert!(magic::poll_override_code() == 1);
      assert!(magic::poll_override_error() == 0);
   }
}

#[test]
fn issue_privileged_code() {
   unsafe {
      magic::override_reset_state();
      magic::issue_override_code(2);
      assert!(magic::poll_override_code() == 2);
      assert!(magic::poll_override_error() == 0);
   }
}

#[test]
fn issue_admin_code() {
   unsafe {
      magic::override_reset_state();
      magic::issue_override_code(3);
      assert!(magic::poll_override_code() == 3);
      assert!(magic::poll_override_error() == 0);
   }
}
```

# 访问状态信息和会话

这些测试确认授权会话和释放会话工作正常：

```rs
#[test]
fn authorize_override_success() {
   unsafe {
      magic::override_reset_state();
      magic::issue_override_code(1);
      let session = magic::poll_override_session();
      assert!(session != (0 as *const c_void));
      magic::free_override_session(session);
      assert!(magic::poll_override_error() == 0);
   }
}

#[test]
fn authorize_privileged_success() {
   unsafe {
      magic::override_reset_state();
      magic::issue_override_code(2);
      let session = magic::poll_physical_override_privileged_session();
      assert!(session != (0 as *const c_void));
      magic::free_override_session(session);
      assert!(magic::poll_override_error() == 0);
   }
}

#[test]
fn authorize_admin_success() {
   unsafe {
      magic::override_reset_state();
      magic::issue_override_code(3);
      let session = magic::poll_physical_override_admin_session();
      assert!(session != (0 as *const c_void));
      magic::free_override_session(session);
      assert!(magic::poll_override_error() == 0);
   }
}
```

# 使活跃会话失效

使活跃会话失效是一个尝试同时授权两个会话的错误，如下所示：

```rs
#[test]
fn double_override_failure() {
   unsafe {
      magic::override_reset_state();
      magic::issue_override_code(1);
      magic::issue_override_code(1);
      assert!(magic::poll_override_session() == (0 as *const c_void));
      assert!(magic::poll_override_error() == 1);
   }
}

#[test]
fn double_privileged_failure() {
   unsafe {
      magic::override_reset_state();
      magic::issue_override_code(2);
      magic::issue_override_code(2);
      assert!(magic::poll_physical_override_privileged_session() == (0 as *const c_void));
      assert!(magic::poll_override_error() == 1);
   }
}

#[test]
fn double_admin_failure() {
   unsafe {
      magic::override_reset_state();
      magic::issue_override_code(3);
      magic::issue_override_code(3);
      assert!(magic::poll_physical_override_admin_session() == (0 as *const c_void));
      assert!(magic::poll_override_error() == 1);
   }
}
```

同一对象上两次调用空闲会话也是不允许的。由于可能发生内存损坏，因此强烈反对在外国库中多次调用析构函数：

```rs
#[test]
fn double_free_override_failure() {
   unsafe {
      magic::override_reset_state();
      magic::issue_override_code(1);
      let session = magic::poll_override_session();
      assert!(session != (0 as *const c_void));
      magic::free_override_session(session);
      magic::free_override_session(session);
      assert!(magic::poll_override_error() == 2);
   }
}

#[test]
fn double_free_privileged_failure() {
   unsafe {
      magic::override_reset_state();
      magic::issue_override_code(2);
      let session = magic::poll_physical_override_privileged_session();
      assert!(session != (0 as *const c_void));
      magic::free_override_session(session);
      magic::free_override_session(session);
      assert!(magic::poll_override_error() == 2);
   }
}

#[test]
fn double_free_admin_failure() {
   unsafe {
      magic::override_reset_state();
      magic::issue_override_code(3);
      let session = magic::poll_physical_override_admin_session();
      assert!(session != (0 as *const c_void));
      magic::free_override_session(session);
      magic::free_override_session(session);
      assert!(magic::poll_override_error() == 2);
   }
}
```

# 发布普通命令

普通命令不需要授权，因此这些测试只是检查命令是否发布和接收：

```rs
#[test]
fn flash() {
   unsafe {
      magic::override_reset_state();
      magic::elevator_display_flash(222);
      assert!(magic::poll_override_code() == 7);
      assert!(magic::poll_override_code() == 222);
   }
}

#[test]
fn toggle_light() {
   unsafe {
      magic::override_reset_state();
      magic::elevator_display_toggle_light(33);
      assert!(magic::poll_override_code() == 8);
      assert!(magic::poll_override_code() == 33);
      assert!(magic::poll_override_code() == 1);
      magic::elevator_display_toggle_light(33);
      assert!(magic::poll_override_code() == 8);
      assert!(magic::poll_override_code() == 33);
      assert!(magic::poll_override_code() == 0);
   }
}

#[test]
fn set_light_color() {
   unsafe {
      magic::override_reset_state();
      magic::elevator_display_set_light_color(33, 222);
      assert!(magic::poll_override_code() == 9);
      assert!(magic::poll_override_code() == 33);
      assert!(magic::poll_override_code() == 222);
   }
}
```

# 发布特权命令

如果有活跃的授权会话，将允许发布特权命令：

```rs
#[test]
fn input_floor() {
   unsafe {
      magic::override_reset_state();
      magic::issue_override_code(3);
      magic::override_input_floor(2);
      assert!(magic::poll_override_code() == 4);
      assert!(magic::poll_override_code() == 2);
      assert!(magic::poll_override_error() == 0);
   }
}

#[test]
fn manual_mode() {
   unsafe {
      magic::override_reset_state();
      magic::issue_override_code(3);
      magic::override_manual_mode();
      assert!(magic::poll_override_code() == 5);
      assert!(magic::poll_override_error() == 0);
   }
}

#[test]
fn normal_mode() {
   unsafe {
      magic::override_reset_state();
      magic::issue_override_code(3);
      magic::override_normal_mode();
      assert!(magic::poll_override_code() == 6);
      assert!(magic::poll_override_error() == 0);
   }
}
```

# 拒绝未经授权的命令

如果没有活跃的授权会话，将拒绝特权命令：

```rs
#[test]
fn deny_input_floor() {
   unsafe {
      magic::override_reset_state();
      magic::issue_override_code(4);
      magic::issue_override_code(2);
      assert!(magic::poll_override_error() == 3);
   }
}

#[test]
fn deny_manual_mode() {
   unsafe {
      magic::override_reset_state();
      magic::issue_override_code(5);
      assert!(magic::poll_override_error() == 3);
   }
}

#[test]
fn deny_normal_mode() {
   unsafe {
      magic::override_reset_state();
      magic::issue_override_code(6);
      assert!(magic::poll_override_error() == 3);
   }
}
```

# 检查 Rust 测试

这些在`src/tests/admin.rs`中的测试涵盖了在`src/admin.rs`中定义的高级语义。它们覆盖了与低级测试大致相同的测试用例；然而，其中一些测试失败了。为了挽救库，应该调整库，以便这些测试能够通过。

# 使用会话进行 Rust 授权

这里有一些高级测试，涵盖了会话的认证和停用：

```rs
#[test]
fn authorize_override() {
   admin::reset_state();
   {
      let session = admin::authorize_override().ok();
      assert!(admin::is_override());
   }
   assert!(!admin::is_override());
   assert!(admin::check_error(()).is_ok());
}

#[test]
fn authorize_privileged() {
   admin::reset_state();
   {
      let session = admin::authorize_privileged().ok();
      assert(admin::is_privileged());
   }
   assert!(!admin::is_privileged());
   assert!(admin::check_error(()).is_ok());
}

#[test]
fn issue_admin_code() {
   admin::reset_state();
   {
      let session = admin::authorize_admin().ok();
      assert(admin::is_admin());
   }
   assert(!admin::is_admin());
   assert!(admin::check_error(()).is_ok());
}
```

# Rust 会话引用共享

高级库支持克隆会话。哎呀！这可能会变得复杂，但测试清楚地说明了它应该如何工作：

```rs
#[test]
fn clone_override() {
   admin::reset_state();
   {
      let session = admin::authorize_override().ok().unwrap();
      let session2 = session.clone();
      assert!(admin::is_override());
   }
   assert!(!admin::is_override());
   assert!(admin::check_error(()).is_ok());
}

#[test]
fn clone_privileged() {
   admin::reset_state();
   {
      let session = admin::authorize_privileged().ok().unwrap();
      let session2 = session.clone();
      assert!(admin::is_privileged());
   }
   assert!(!admin::is_privileged());
   assert!(admin::check_error(()).is_ok());
}

#[test]
fn clone_admin() {
   admin::reset_state();
   {
      let session = admin::authorize_admin().ok().unwrap();
      let session2 = session.clone();
      assert!(admin::is_admin());
   }
   assert!(!admin::is_admin());
   assert!(admin::check_error(()).is_ok());
}
```

# 特权命令

如果存在活动的授权会话，则应允许特权命令：

```rs
#[test]
fn input_floor() {
   admin::reset_state();
   {
      let session = admin::authorize_admin().ok();
      admin::input_floor(2).ok();
   }
   assert!(!admin::is_admin());
   assert!(admin::check_error(()).is_ok());
}

#[test]
fn manual_mode() {
   admin::reset_state();
   {
      let session = admin::authorize_admin().ok();
      admin::manual_mode().ok();
   }
   assert!(!admin::is_admin());
   assert!(admin::check_error(()).is_ok());
}

#[test]
fn normal_mode() {
   admin::reset_state();
   {
      let session = admin::authorize_admin().ok();
      admin::normal_mode().ok();
   }
   assert!(!admin::is_admin());
   assert!(admin::check_error(()).is_ok());
} 
```

# 无权限命令

无权限命令应不受认证影响而允许：

```rs
#[test]
fn flash() {
   admin::reset_state();
   assert!(!admin::is_override());
   assert!(!admin::is_privileged());
   assert!(!admin::is_admin());
   admin::flash(222).ok();
   assert!(admin::check_error(()).is_ok());
}

#[test]
fn toggle_light() {
   admin::reset_state();
   assert!(!admin::is_override());
   assert!(!admin::is_privileged());
   assert!(!admin::is_admin());
   admin::toggle_light(7).ok();
   assert!(admin::check_error(()).is_ok());
}

#[test]
fn set_light_color() {
   admin::reset_state();
   assert!(!admin::is_override());
   assert!(!admin::is_privileged());
   assert!(!admin::is_admin());
   admin::set_light_color(33, 123).ok();
   assert!(admin::check_error(()).is_ok());
}
```

# 拒绝访问特权命令

如果没有授权的活跃会话，则应拒绝特权命令：

```rs
#[test]
fn deny_input_floor() {
   admin::reset_state();
   admin::input_floor(2).err();
   assert!(!admin::check_error(()).is_ok());
}

#[test]
fn deny_manual_mode() {
   admin::reset_state();
   admin::manual_mode().err();
   assert!(!admin::check_error(()).is_ok());
}

#[test]
fn deny_normal_mode() {
   admin::reset_state();
   admin::normal_mode().err();
   assert!(!admin::check_error(()).is_ok());
}
```

# 学习所有权规则

Rust 有三个所有权规则：

+   Rust 中的每个值都有一个称为其**所有者**的变量

+   一次只能有一个所有者

+   当所有者超出作用域时，其值将被释放

在最简单的情况下，我们可以在块的末尾定义一个超出作用域的变量：

```rs
fn main()
{
   //variable x has not yet been defined
   {
      let x = 5;
      //variable x is now defined and owned by this context

      //variable x is going out of scope and will be dropped here
   }
   //variable x has gone out of scope and is no longer defined
}
```

我们在前几章中已经接触到了所有权和生命周期的前两条规则。然而，这是第一次我们需要与第三条规则——释放（drop）——打交道。

# 当所有者超出作用域时，其值将被释放

在前面的代码中，我们可以看到函数块作为所有者的简单情况。当函数块退出时，变量将被释放。所有权也可以转移，因此当值被发送或返回到另一个块时，该块将成为新的所有者。剩下的情况是所有权转移到对象。当值被释放时，所有子对象也会自动释放。

在当前项目中，有三个测试失败，所有这些都与会话上的`.clone`方法有关。失败的会话看起来如下：

```rs
#[test]
fn clone_override() {
   admin::reset_state();
   {
      let session = admin::authorize_override().ok().unwrap();
      let session2 = session.clone();
      assert!(admin::is_override());
   }
   assert!(!admin::is_override());
   assert!(admin::check_error(()).is_ok());
}
```

移除样板代码后，我们可以看到三个测试都遵循相同的模式：

1.  打开一个新的块

    1.  授权新的会话

    1.  克隆新的会话

    1.  确认会话已授权

1.  关闭块

1.  确认会话未授权

1.  确认没有发生错误

所有测试都正常工作，除了在测试结束时检查到的错误。错误代码指示会话发生了双重释放。根据正常的 Rust 所有权规则，我们知道克隆的会话将分别被释放。这很有道理，因为`Drop`为作用域内的两个`AuthorizedSession`结构体都实现了。如果我们查看`Drop`的实现，我们可以看到它只是天真地调用了外部库，这会导致双重释放错误：

```rs
#[derive(Clone)]
pub struct AuthorizedSession
{
   session: *const c_void
}
impl Drop for AuthorizedSession {
   fn drop(&mut self) {
      unsafe {
         magic::free_override_session(self.session);
      }
   }
}
```

通常，Rust 可能会抱怨这种粗心的资源管理。然而，库使用一个不安全块来包装对外部函数的调用。将代码标记为不安全会关闭许多安全检查，并鼓励编译器信任程序员。调用外部库本质上是不可安全的，所以这个不安全块仍然是必要的。

这里的正确行为似乎是在所有克隆的会话都被释放后只释放会话一次。这是一个很好的 `std::rc::Rc` 用例，它代表引用计数。

`Rc` 通过在内部存储一个拥有的值来工作。`Rc` 的所有拥有者不再直接拥有引用计数容器内部的对象。要使用内部对象，借用人必须请求借用内部对象的指针。`Rc` 对象的所有权将被计数，当所有包含给定值的引用都消失时，该值将被释放。

这个内置功能正好提供了我们想要的功能。多次克隆，一次释放，如下所示：

```rs
struct AuthorizedSessionInner(*const c_void);

#[derive(Clone)]
pub struct AuthorizedSession
{
   session: Rc<AuthorizedSessionInner>
}

impl Drop for AuthorizedSessionInner {
   fn drop(&mut self) {
      unsafe {
         magic::free_override_session(self.0);
      }
   }
}
```

为了从原始指针初始化会话，我们需要将它们包装起来。否则，不需要更改任何代码：

```rs
let session = AuthorizedSession {
   session: Rc::new(AuthorizedSessionInner(session))
};
```

经过这些小的改动后，剩下的三个测试用例都通过了。看起来库似乎正在正常工作。这里要学到的重大教训是，`Drop` 实现有时可能非常敏感。不要假设多次释放会安全。为了处理复杂的情况，标准库中提供了 `std::rc::Rc` 和 `std::sync::Arc` 类型。`Arc` 是 `Rc` 的线程安全版本。

# 使用不可变数据

在使用真实电梯实现和测试库之后，你发现另一个 bug——当有人物理地进入一个会话时，有时他们在使用电梯的同时被取消授权。在 bug 报告中，“有时”这个词听起来很糟糕。

# 修复难以重现的 bug

经过大量的搜索和研究后，你找到了一个可以可靠地重现问题的测试用例：

```rs
#[test]
fn invalid_deauthorization() {
   admin::reset_state();
   let session = admin::authorize_admin().ok();
   assert!(admin::authorize_admin().is_err());
   assert!(admin::is_admin());
}
```

看到这个测试用例，我们可能会问的第一个问题是，为什么应该允许这样做？

在物理测试中遇到的问题是由有效会话的随机取消授权所表征的。在调查中发现，在物理授权会话期间，有时会启动软件授权会话。物理授权是当有人使用电梯上的钥匙来使用特殊命令时。软件授权是从运行软件而不是从电梯硬件发起的任何其他授权会话。这个双重授权动作违反了双重授权约束，因此两个会话都被无效化。解决方案显然是允许第一个授权会话继续，同时拒绝第二次授权。

这个解决方案看起来相当直接和简单。从 `src/admin.rs`，我们有能力检查是否有任何会话被授权，然后不调用库就拒绝第二次授权。

因此，在重写授权命令时，我们添加了一个检查来查看是否已经存在一个授权会话。如果存在这样的会话，则此授权失败：

```rs
pub fn authorize_override() -> Result<AuthorizedSession,ErrorCode>
{
   if is_override() || is_privileged() || is_admin() {
      return Result::Err(ErrorCode::DoubleAuthorize)
   }
   let session = unsafe {
      magic::issue_override_code(OverrideCode::IssueOverride as i32);
      magic::poll_override_session()
   };
   let session = AuthorizedSession {
      session: Rc::new(AuthorizedSessionInner(session))
   };
   check_error(session)
}

pub fn authorize_privileged() -> Result<AuthorizedSession,ErrorCode>
{ ... }

pub fn authorize_admin() -> Result<AuthorizedSession,ErrorCode>
{ ... }
```

这个更改解决了立即的问题，但导致双重释放测试失败，因为现在在双重释放后库中没有生成错误代码。我们本质上是在保护底层库免受双重释放的责任，因此这是一个可预见的后果。新的测试只是移除了之前检查错误代码的最后一行：

```rs
#[test]
fn double_override_failure() {
   admin::reset_state();
   let session = admin::authorize_override().ok();
   assert!(admin::authorize_override().err().is_some());
}

#[test]
fn double_privileged_failure() {
   admin::reset_state();
   let session = admin::authorize_privileged().ok();
   assert!(admin::authorize_privileged().err().is_some());
}

#[test]
fn double_admin_failure() {
   admin::reset_state();
   let session = admin::authorize_admin().ok();
   assert!(admin::authorize_admin().err().is_some());
}
```

# 防止难以重现的 bug

Rust 被特别设计来避免这种难以重现的 bug。在 Rust 中，原始指针的处理是被阻止或强烈劝阻的。原始指针就像 Rust 一无所知的引用，因此无法就其使用提供任何安全保证。不幸的是，这个 bug 是外部库内部的，因此我们的 Rust 项目没有管辖权来抱怨这里的根本问题。尽管如此，我们仍然可以遵循一些良好的实践来防止或限制与变异和奇怪的副作用相关的 bug 的发生。

我们将推荐的第一个技术是不可变性。默认情况下，所有变量都被声明为不可变。这是 Rust 以一种不太微妙的方式告诉你，如果可能的话，要避免修改值，如下所示：

```rs
fn main() {
   let a = 5;
   let mut b = 5;

   //a = 4; not valid
   b = 4;

   //*(&mut a) = 3; not valid
   *(&mut b) = 3;
}
```

不可变值不能作为可变借用（按设计），因此要求函数参数的可变性将需要从发送给它的每个值中获取可变性：

```rs
fn f(x: &mut i32) {
   *x = 2;
}

fn main() {
   let a = 5;
   let mut b = 5;

   //f(&mut a); not valid
   f(&mut b);
}
```

将不可变值转换为可变值可能就像克隆它以创建一个新相同值那样简单；然而，正如我们在本章中看到的，克隆并不总是简单操作，以下是一个示例：

```rs
use std::sync::{Mutex, Arc};

#[derive(Clone)]
struct TimeBomb {
   countdown: Arc<Mutex<i32>>
}
impl Drop for TimeBomb
{
   fn drop(&mut self) {
      let mut c = self.countdown.lock().unwrap();
      *c -= 1;
      if *c <= 0 {
         panic!("BOOM!!")
      }
   }
}

fn main()
{
   let t3 = TimeBomb {
      countdown: Arc::new(Mutex::new(3))
   };
   let t2 = t3.clone();
   let t1 = t2.clone();
   let t0 = t1.clone();
}
```

将变量声明为不可变并不能绝对防止所有变异，无论是内部还是外部。在 Rust 中，不可变变量允许持有可变数据类型的内部字段。例如，可以使用`std::cell::RefCell`在它持有的任何数据上实现内部可变性。

尽管有例外，但默认使用不可变变量可以帮助防止简单 bug 变成复杂 bug。不要让你的编程风格成为负担；练习防御性软件开发。

# 使用纯函数

纯函数是我们推荐的第二个技术，用于防止难以重现的 bug。纯函数可以被视为避免副作用原则的扩展。纯函数的定义是一个函数，其中以下条件是真实的：

+   函数外部没有引起任何变化（没有副作用）

+   返回值只依赖于函数参数

这里有一些纯函数的例子：

```rs
fn p0() {}

fn p1() -> u64 {
   444
}

fn p2(x: u64) -> u64 {
   x * 444
}

fn p3(x: u64, y: u64) -> u64 {
   x * 444 + y
}

fn main()
{
   p0();
   p1();
   p2(3);
   p3(3,4);
}
```

这里有一些不纯函数的例子：

```rs
use std::cell::Cell;

static mut blah: u64 = 3;
fn ip0() {
   unsafe {
      blah = 444;
   }
}

fn ip1(c: &Cell<u64>) {
   c.set(333);
}

fn main()
{
   ip0();
   let r = Cell::new(3);
   ip1(&r);
   ip1(&r);
}
```

Rust 没有任何语言特性专门指定一个函数是更纯还是更不纯。然而，正如前面的例子所展示的，Rust 在一定程度上不鼓励不纯函数。函数的纯度应该被视为一种设计模式，并且与良好的函数式风格紧密相关。

与顶级函数一样，闭包也可以是纯的或不纯的。因此，当与高级函数一起工作时，函数的纯度成为一个关注点。某些函数式编程模式期望函数是纯的。一个很好的例子是我们简要提到的第一章中的记忆化模式——“函数式编程——比较”。让我们比较一下，如果记忆化的函数是不纯的，会发生什么。

首先，这是一个关于记忆化应该如何工作的提醒：

```rs
#[macro_use] extern crate cached;

cached!{
   FIB;
   fn fib(n: u64) -> u64 = {
      if n == 0 || n == 1 { return n }
      fib(n-1) + fib(n-2)
   }
}

fn main() {
   fib(30); //call 1, generates correct value and returns it
   fib(30); //call 2, finds correct value and returns it
}
```

接下来，让我们看看一个记忆化的不纯函数：

```rs
#[macro_use] extern crate lazy_static;
#[macro_use] extern crate cached;
use std::collections::HashMap;
use std::sync::Mutex;

lazy_static! {
   static ref BUCKET_COUNTER: Mutex<HashMap<u64, u64>> = {
      Mutex::new(HashMap::new())
   };
}

cached!{
   BUCK;
   fn bucket_count(n: u64) -> u64 = {
      let mut counter = BUCKET_COUNTER.lock().unwrap();
      let r = match counter.get(&n) {
         Some(c) => { c+1 }
         None => { 1 }
      };
      counter.insert(n, r);
      r
   }
}

fn main() {
   bucket_count(30); //call 1, generates correct value and returns it
   bucket_count(30); //call 2, finds stale value and returns it
} 
```

这个第一个缓存示例应该每次都返回相同的值。第二个示例不应该每次都返回相同的值。从语义上讲，我们不希望第二个示例返回过时的值；然而，这也意味着我们无法安全地缓存结果。这里有一个必要的性能权衡。如果必要的话，这里两个示例的纯度或杂质都没有问题。这仅仅意味着第二个示例不应该被缓存。

然而，也存在不纯的反模式。让我们看看另一个表现不佳的不纯函数：

```rs
#[macro_use] extern crate cached;
use std::sync::{Arc,Mutex};

#[derive(Clone)]
pub struct TimeBomb {
   countdown: Arc<Mutex<i32>>
}

impl Drop for TimeBomb
{
   fn drop(&mut self) {
      let mut c = self.countdown.lock().unwrap();
      *c -= 1;
      if *c <= 0 {
         panic!("BOOM!!")
      }
   }
}

cached!{
   TICKING_BOX;
   fn tick_tock(v: i32) -> TimeBomb = {
      TimeBomb {
         countdown: Arc::new(Mutex::new(v))
      }
   }
}

fn main() {
   tick_tock(3);
   tick_tock(3);
   tick_tock(3);
}
```

在这个例子中，数据本身是不纯的。每次`tick_tock`都会移动和丢弃一个`TimeBomb`。最终，它会爆炸，我们的缓存无法帮助我们保护。希望你在你的程序中不需要处理炸弹。

# 摘要

在本章中，我们使用了 Rust 的遗留代码和外部库。Rust 的安全保障可能难以学习，有时使用起来也很繁琐，但快速而松散的编码方式同样令人压力山大且问题重重。

Rust 内存安全规则的一个动机是双重释放内存的概念，我们在本章中提到了这一点。然而，展示的代码并没有涉及真正的双重释放内存。真正的双重释放会导致称为未定义行为的现象。未定义行为是语言标准中用来指代会导致程序行为异常的操作的术语。双重释放的内存通常是未定义行为中最糟糕的类型之一，会导致内存损坏和随后的崩溃或难以追踪到原始原因的无效状态。

在本章的后半部分，我们考察了特定的 Rust 设计决策、特性和模式，例如所有权、不可变性和纯函数。这些都是 Rust 对抗未定义行为和其他问题的防御机制。

正确使用 Rust 的安全措施而不是规避它们有许多好处。Rust 鼓励一种有利于大型项目设计的编程风格。通常，项目架构遵循一个超过线性的错误/复杂度曲线。随着项目规模的扩大，错误和困难情况将以更快的速度增长。通过锁定常见的错误来源或代码依赖，可以开发出问题更少的大型项目。

在下一章中，我们将正式解释许多功能设计模式。这将是一个学习函数式编程原则在 Rust 中应用程度和相关性很好的机会。如果下一章中没有什么看起来酷或有用，那么作者就失败了。

# 问题

1.  `Rc`代表什么？

1.  `Arc`代表什么？

1.  什么是弱引用？

1.  在不安全块中启用了哪些超级能力？

1.  对象何时会被丢弃？

1.  生命周期和所有权的区别是什么？

1.  你如何确保一个函数是安全的？

1.  内存损坏是什么，它会如何影响程序？
