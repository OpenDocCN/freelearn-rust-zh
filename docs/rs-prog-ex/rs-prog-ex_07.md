# 使用 Relm 以更 Rust 风格的方式创建音乐播放器

在上一章中，我们完成了我们的音乐播放器。这完全没问题，但直接在 Rust 中使用 `gtk-rs` 可能会导致错误。这就是为什么我们将使用 `relm`，这是一个 Rust 的惯用 GUI 库，来重写我们的音乐播放器。`Relm` 基于 `gtk-rs`，所以最终的应用程序看起来是一样的。然而，代码将更简洁、更具声明性。

在本章中，我们将涵盖以下主题：

+   Relm

+   Relm 小部件

+   模型-视图-控制器

+   声明性视图

+   消息传递

# 使用 relm 而不是直接使用 gtk-rs 的原因

正如你在前面的章节中看到的，我们使用了并不真正明显的一些概念，并且在使用 GTK+ 和 Rust 时，做一些通常很容易做的事情并不那么容易。这些都是使用 `relm` 的许多原因之一。

# 状态修改

从上一章可能不清楚，但我们间接使用了 `Rc<RefCell<T>>` 来进行状态修改。确实，我们的 `Playlist` 类型包含一个 `RefCell<Option<String>>`，并且我们将 `Playlist` 包裹在一个引用计数指针中。这是为了能够根据事件对状态进行修改，例如在点击播放按钮时播放歌曲：

```rs
let playlist = self.playlist.clone();
let play_image = self.toolbar.play_image.clone();
let cover = self.cover.clone();
let state = self.state.clone();
self.toolbar.play_button.connect_clicked(move |_| {
    if state.lock().unwrap().stopped {
        if playlist.play() {
            set_image_icon(&play_image, PAUSE_ICON);
            set_cover(&cover, &playlist);
        }
    } else {
        playlist.pause();
        set_image_icon(&play_image, PLAY_ICON);
    }
});
```

必须使用所有这些 `clone()` 调用是非常繁琐的，使用 `RefCell<T>` 类型可能会导致在复杂应用程序中难以调试的问题。这个类型的问题在于借用检查是在运行时发生的。例如，以下应用程序：

```rs
use std::cell::RefCell;
use std::collections::HashMap;

fn main() {
    let cell = RefCell::new(HashMap::new());
    cell.borrow_mut().insert("one", 1);
    let borrowed_cell = cell.borrow();
    if let Some(key) = borrowed_cell.get("one") {
        cell.borrow_mut().insert("two", 2);
    }
}
```

将会引发恐慌：

```rs
thread 'main' panicked at 'already borrowed: BorrowMutError', /checkout/src/libcore/result.rs:906:4
```

尽管在这个例子中为什么它会引发恐慌（我们在 `borrowed_cell` 中的借用仍然有效时调用了 `borrow_mut()`）是显而易见的，但在更复杂的应用程序中，理解为什么会发生恐慌会更困难，尤其是如果我们将 `RefCell<T>` 包裹在 `Rc` 中并在每个地方克隆它。这使我们来到了这个类型的第二个问题：使用 `Rc<T>` 鼓励我们过度克隆和共享我们的数据，这增加了我们模块之间的耦合度。

`relm` 包采用了一种不同的方法：小部件拥有自己的数据，不同的小部件之间通过消息传递进行通信。

# 异步用户界面

在创建用户界面时，另一个常见问题是，我们可能想要执行可能需要花费时间（例如网络请求）的操作，而不冻结用户界面。由于基于 `tokio`，这是一个为 Rust 设计的异步 I/O 框架，`relm` 允许你轻松地编写可以执行网络请求而不冻结界面的图形用户界面。

# 创建自定义小部件

在面向对象的语言中，创建新小部件并像内置小部件一样使用它们非常容易。在这个范例中，你只需要创建一个新的类，它继承自小部件，然后就可以了。

在 第五章 *创建音乐播放器* 中，我们创建了自定义小部件，例如 `Playlist` 和 `MusicToolbar`，但我们需要创建一个函数来获取真实的 GTK+ 小部件：

```rs
pub fn view(&self) -> &TreeView {
    &self.treeview
}
```

另一个选择是实现 `Deref` 特性：

```rs
use std::ops::Deref;

impl Deref for Playlist {
    type Target = TreeView;

    fn deref(&self) -> &TreeView {
        &self.treeview
    }
}
```

这种实现将允许我们像这样将小部件添加到其 `parent` 中：

```rs
parent.add(&*playlist);
```

（注意 `playlist` 前面的前导 `*`，这是对 `deref()` 的调用。）

而不是以下这种方式添加：

```rs
parent.add(playlist.view());
```

但这仍然与使用正常的 `gtk` 小部件不同。

`Relm` 解决了所有这些问题。让我们开始使用这个包。

# 使用 relm 创建窗口

首先，我们将使用 Rust 编译器的 Nightly 版本。

虽然使用这个 Nightly 版本不是使用 `relm` 的严格必要条件，但它提供了一种语法，使用这个版本上仅有的功能，语法会更好一些。

这将是一个学习如何安装不同版本的编译器的良好机会。Nightly 是 Rust 的不稳定版本；这是一个几乎每天都会编译的版本。Rust 的一些不稳定特性仅在 Nightly 版本中可用。但是，不用担心，我们也会看看如何在 Rust 的稳定版本上使用 `relm`。

# 安装 Rust Nightly

使用 `rustup`，我们在 第一章 “Rust 基础”中安装的工具，安装 Nightly 非常容易：

```rs
rustup default nightly
```

运行此命令将安装工具的 Nightly 版本（`cargo`、`rustc` 等）。同时，它还将切换相应的命令以使用 Nightly 版本。

如果你想要回到稳定版本，请发出以下命令：

```rs
rustup default stable
```

Nightly 版本更新非常频繁，所以你可能希望每周或更频繁地更新它。为此，你需要运行以下命令：

```rs
rustup update
```

如果发布了新版本，这将也会更新稳定版本（每 6 周发布一个稳定版本）。

现在我们正在使用 Rust Nightly，我们准备好创建一个 `new` 项目：

```rs
cargo new rusic-relm --bin
```

在 `Cargo.toml` 文件中添加以下依赖项：

```rs
[dependencies]
gtk = "⁰.3.0"
gtk-sys = "⁰.5.0"
relm = "⁰.11.0"
relm-attributes = "⁰.11.0"
relm-derive = "⁰.11.0"
```

我们仍然需要 `gtk`，因为 `relm` 是基于它的。让我们添加相应的 `extern crate` 语句：

```rs
#![feature(proc_macro)]

extern crate gtk;
extern crate gtk_sys;
#[macro_use]
extern crate relm;
extern crate relm_attributes;
#[macro_use]
extern crate relm_derive;
```

`relm` 提供了一些宏，这就是为什么我们需要添加 `#[macro_use]`。我们将通过使用 `relm` 创建一个简单的窗口来慢慢开始。

# 小部件

这个包围绕小部件的概念构建，这与 `gtk` 小部件不同。在 `relm` 中，小部件由视图、模型和用于响应事件更新模型的方法组成。小部件的概念通过 `relm` 中的一个 trait 实现：`Widget` trait。

# 模型

我们将从空模型开始，并在本章的后面部分填充它：

```rs
pub struct Model {
}
```

正如你所见，一个模型可以是一个简单的结构。如果你的小部件不需要模型，它也可以是 `()`。实际上，它可以是你想要的任何类型。

除了模型，小部件还需要知道其模型的初始值。为了指定它是什么，我们需要实现 `Widget` trait 的 `model()` 方法：

```rs
#[widget]
impl Widget for App {
    fn model() -> Model {
        Model {
        }
    }

    // …
}
```

这里，我们使用了由 `relm_attributes` 包提供的 `#[widget]` 属性。属性目前是语言的一个不稳定特性，这就是为什么我们使用 nightly。我们将在关于声明视图的部分看到为什么这个属性是必需的。所以，让我们回到我们的 `model()` 模型，我们现在只返回 `Model {}` 作为我们的模型不包含任何数据。这个特性还需要其他方法，所以这个实现现在是不完整的。

# 消息

`Relm` 小部件通过向其他小部件以及自身发送消息来进行通信。例如，当 `delete_event` 信号被触发时，我们可以向我们的小部件发送 `Quit` 消息，并在接收到此消息时采取适当的行动。消息被建模为一个使用特定于 `relm` 的自定义派生 `Msg` 的 `enum`：

```rs
#[derive(Msg)]
pub enum Msg {
    Quit,
}
```

这个自定义派生是由 `relm_derive` 包提供的。

# 视图

在 `relm` 中，视图以声明方式创建，作为 `Widget` 特性的一个部分：

```rs
use gtk::{
    GtkWindowExt,
    Inhibit,
    WidgetExt,
};
use relm::Widget;
use relm_attributes::widget;

use self::Msg::*;

#[widget]
impl Widget for App {
    // …

    view! {
        gtk::Window {
            title: "Rusic",
            delete_event(_, _) => (Quit, Inhibit(false)),
        }
    }
}
```

我们首先从 `gtk` 包中导入了一些内容。然后，我们导入了 `relm` 中的 `Widget` 特性和 `widget` 属性。稍后，我们导入了我们的 `enum Msg` 的变体，因为我们在这段代码中使用它。为了声明视图，我们使用了 `view!` 宏。这个宏非常特别，它不是一个像我们在 第一章 中看到的 `macro_rules!` 声明的宏。相反，它是通过实现 `#[widget]` 属性的进程宏来解析的，以便提供在 Rust 中不允许的语法。

为了声明我们的视图，我们首先指定 `gtk::Window` 小部件的名称。

我们不能导入 `gtk::Window` 以便在视图声明中只使用 `Window`。

之后，我们使用花括号，并在其中指定小部件处理的属性和事件。

# 属性

在这里，我们声明 `title` 属性为 `"Rusic"`。因此，我们将 `set_title()` 调用从 `gtk` 转换为 `title` 属性，只需要 `set_` 之后的部分。实际上，`relm` 会将属性 (`title: "Rusic"`) 转换为 `set_title("Rusic")` 调用，正如我们稍后将会看到的。

# 事件

事件处理器的语法有点特殊：

```rs
delete_event(_, _) => (Quit, Inhibit(false)),
```

首先，我们只需要写 `delete_event(_, _) =>` 而不是 `connect_delete_event(move |_, _| { })`。如果我们需要信号的参数，我们可以写一个标识符的名字而不是使用下划线（`_`）。在粗箭头（`=>`）的右侧，我们指定两个用括号括起来并用逗号分隔的东西。首先，是 `Quit`，这是当事件被触发时将发送到当前小部件的消息。其次是要返回给 `gtk` 回调的值。在这里，我们返回 `Inhibit(false)` 以指定我们不想阻止默认事件处理器运行。

# 代码生成

由属性生成的代码是一个看起来像正常的 Rust 方法的代码：

```rs
fn view(relm: &Relm<Self>, model: Self::Model) -> Self {
   // This method does not actually exist, but relm directly create a window using the functions from the sys crates.
    let window = gtk::Window::new();
    window.set_title("Rusic");

    window.show();

    connect!(relm, window, connect_delete_event(_, _), return 
     (Some(Quit), Inhibit(false)));

    Win {
        model,
        window: window,
    }
}
```

# 更新函数

`Widget`特质唯一剩余的必需方法是`update()`。在这个方法中，我们将管理`Quit`消息：

```rs
#[widget]
impl Widget for App {
    fn update(&mut self, event: Msg) {
        match event {
            Quit => gtk::main_quit(),
        }
    }

    // …
}
```

在这里，我们指定当接收到`Quit`消息时，我们调用`gtk::main_quit()`，这是一个类似于我们在第五章中使用的`Application::quit()`的函数。

应该注意的是，`#[widget]`属性也会生成包含小部件和模型的`App`结构。

我们可以通过在`main`函数中调用其`run()`方法来最终显示这个窗口：

```rs
fn main() {
    App::run(()).unwrap();
}
```

之后，我们将看到为什么需要指定`()`作为`run()`的参数。

# 添加子小部件

我们看到了如何使用 relm 创建小部件的基本方法。现在，让我们继续创建我们的用户界面。我们将从添加工具栏开始。除了在`view!`宏中指定属性和信号外，我们还可以嵌套小部件，以便将子小部件添加到容器中。因此，要将`gtk::Box`作为窗口的子小部件添加，我们只需将前者嵌套在后者内部：

```rs
view! {
    gtk::Window {
        title: "Rusic",
        delete_event(_, _) => (Quit, Inhibit(false)),
        gtk::Box {
        },
    }
}
```

要将工具栏添加到`gtk::Box`中，我们创建一个新的嵌套层级：

```rs
view! {
    gtk::Window {
        title: "Rusic",
        delete_event(_, _) => (Quit, Inhibit(false)),
        gtk::Box {
            orientation: Vertical,
            #[name="toolbar"]
            gtk::Toolbar {
            },
        },
    }
}
```

在这里，我们可以看到一个属性：`#[name]`属性给一个小部件命名，这将允许我们通过指定的标识符访问这个小部件，就像我们稍后将要看到的那样。在本章的其余部分，我们还会遇到其他属性。

我们将在我们的模型中添加一个属性来保持播放/暂停按钮上要显示的图像：

```rs
use gtk::Image;

pub const PAUSE_ICON: &str = "gtk-media-pause";
pub const PLAY_ICON: &str = "gtk-media-play";

pub struct Model {
    play_image: Image,
}
```

我们还添加了表示按钮状态的图像名称的常量。我们需要更新`model()`方法来指定这个新字段：

```rs
fn model() -> Model {
    Model {
        play_image: new_icon(PLAY_ICON),
    }
}
```

这使用以下函数来创建一个图像：

```rs
fn new_icon(icon: &str) -> Image {
    Image::new_from_file(format!("assets/{}.png", icon))
}
```

让我们向工具栏添加项目：

```rs
use gtk::{
    OrientableExt,
    ToolButtonExt,
};
use gtk::Orientation::Vertical;

view! {
    gtk::Window {
        title: "Rusic",
        delete_event(_, _) => (Quit, Inhibit(false)),
        gtk::Box {
            orientation: Vertical,
            #[name="toolbar"]
            gtk::Toolbar {
                gtk::ToolButton {
                    icon_widget: &new_icon("document-open"),
                    clicked => Open,
                },
                gtk::ToolButton {
                    icon_widget: &new_icon("document-save"),
                    clicked => Save,
                },
                gtk::SeparatorToolItem {
                },
                gtk::ToolButton {
                    icon_widget: &new_icon("gtk-media-previous"),
                },
                gtk::ToolButton {
                    icon_widget: &self.model.play_image,
                    clicked => PlayPause,
                },
                gtk::ToolButton {
                    icon_widget: &new_icon("gtk-media-stop"),
                    clicked => Stop,
                },
                gtk::ToolButton {
                    icon_widget: &new_icon("gtk-media-next"),
                },
                gtk::SeparatorToolItem {
                },
                gtk::ToolButton {
                    icon_widget: &new_icon("remove"),
                },
                gtk::SeparatorToolItem {
                },
                gtk::ToolButton {
                    icon_widget: &new_icon("gtk-quit"),
                    clicked => Quit,
                },
            },
        },
    }
}
```

这里没有展示新的语法。请注意，我们可以在属性的值中指定函数调用以及模型属性。我们需要在`new_icon()`之前放置一个`&`，因为代码被这样翻译：

```rs
tool_button.set_icon_widget(&new_icon("gtk-quit"));
```

这个`set_icon_widget()`方法需要可以转换成`Option<&P>`的东西，其中`P`是一个小部件。它需要一个引用，所以我们给它一个引用。

# 单向数据绑定

在 relm 中，从模型属性设置属性是非常频繁的，实际上它创建了一个模型属性和属性之间的单向绑定。这意味着当属性被更新时，小部件属性也会被更新。尽管这个特性有一些限制：

+   只有对模型属性的赋值才会更新属性。

+   这个赋值*必须*在一个带有`#[widget]`属性的实现中。

这些限制来自于`relm`只分析由这个属性装饰的源代码。它只考虑赋值是对模型数据的更新。

这可能需要更改一些代码。例如，以下代码不会触发属性更新：

```rs
self.model.string.push_str("string");
```

你可以这样重写它，以便`relm`将其视为更新：

```rs
self.model.string += "string";
```

如你所见，`relm` 不仅识别 `=` 赋值，还识别使用运算符（如 `+=`）的赋值。

在之前的代码中，我们使用了许多新的消息，所以让我们相应地更新我们的枚举：

```rs
#[derive(Msg)]
pub enum Msg {
    Open,
    PlayPause,
    Quit,
    Save,
    Stop,
}
```

我们还需要更改 `update()` 方法以考虑这些新消息：

```rs
    fn update(&mut self, event: Msg) {
        match event {
            Open => (),
            PlayPause => (),
            Quit => gtk::main_quit(),
            Save => (),
            Stop => (),
        }
    }
```

目前，因为我们只编写了界面，所以当我们收到这些消息时，我们不做任何事情。

# 视图的后续初始化

如果你运行应用程序，你会看到图像没有显示在工具栏按钮上。这是因为 `relm` 的工作方式。当它生成代码时，它会在每个小部件上调用 `show()` 方法，而不是 `show_all()`。所以，工具栏和工具按钮会显示，但图像不会显示，因为它们只是按钮的属性，它们不是使用小部件语法创建的。为了解决这个问题，我们将在 `init_view()` 方法中调用 `show_all()`：

```rs
#[widget]
impl Widget for App {
    fn init_view(&mut self) {
        self.toolbar.show_all();
    }

    // …
}
```

这就是为什么我们之前给工具栏小部件命名的原因：我们需要在这个小部件上调用一个方法。`init_view()` 方法在创建 `view` 之后被调用。这在无法使用 `view!` 语法进行操作时，执行一些代码来自定义视图是有用的。如果你再次运行应用程序，你会看到按钮现在有一个图像。

现在让我们添加封面图像小部件和光标小部件。对于图像，我们需要在 `Cargo.toml` 中添加一个新的 crate：

```rs
[dependencies]
gdk-pixbuf = "⁰.3.0"
```

让我们再添加相应的 `extern crate` 语句：

```rs
extern crate gdk_pixbuf;
```

我们还需要新的导入语句：

```rs
use gdk_pixbuf::Pixbuf;
use gtk::{
    Adjustment,
    BoxExt,
    ImageExt,
    LabelExt,
    ScaleExt,
};
use gtk::Orientation::Horizontal;
```

让我们在我们的 `Model` 中添加几个新字段：

```rs
pub struct Model {
    adjustment: Adjustment,
    cover_pixbuf: Option<Pixbuf>,
    cover_visible: bool,
    current_duration: u64,
    current_time: u64,
    play_image: Image,
}
```

大多数新字段都存在于我们在前两章中开发的应用程序中。不过，`cover_visible` 属性是新的。我们将使用它来知道是否应该显示封面图像。别忘了更新模型的初始化：

```rs
fn model() -> Model {
    Model {
        adjustment: Adjustment::new(0.0, 0.0, 0.0, 0.0, 0.0, 0.0),
        cover_pixbuf: None,
        cover_visible: false,
        current_duration: 0,
        current_time: 0,
        play_image: new_icon(PLAY_ICON),
    }
}
```

我们现在可以在 `Toolbar` 小部件之后添加 `Image`：

```rs
gtk::Image {
    from_pixbuf: self.model.cover_pixbuf.as_ref(),
    visible: self.model.cover_visible,
},
```

在这里，我们在 `cover_pixbuf` 属性上调用 `as_ref()`，因为，又一次，该方法（`set_from_pixbuf()`）需要可以被转换成 `Option<&Pixbuf>` 的东西。我们还指定了图像的 `visible` 属性绑定到 `cover_visible` 模型属性。这意味着我们可以通过将此属性设置为 `false` 来隐藏图像。

然后，我们将添加光标，这将给我们以下视图：

```rs
view! {
    gtk::Window {
        title: "Rusic",
        delete_event(_, _) => (Quit, Inhibit(false)),
        gtk::Box {
            orientation: Vertical,
            #[name="toolbar"]
            gtk::Toolbar {
                // …
            },
            gtk::Image {
                from_pixbuf: self.model.cover_pixbuf.as_ref(),
                visible: self.model.cover_visible,
            },
            gtk::Box {
                orientation: Horizontal,
                spacing: 10,
                gtk::Scale(Horizontal, &self.model.adjustment) {
                    draw_value: false,
                    hexpand: true,
                },
                gtk::Label {
                    text: &millis_to_minutes(self.model.current_time),
                },
                gtk::Label {
                    text: "/",
                },
                gtk::Label {
                    margin_right: 10,
                    text: &millis_to_minutes(self.model.current_duration),
                },
            },
        },
    }
}
```

这需要我们在上一章中看到的方法：

```rs
fn millis_to_minutes(millis: u64) -> String {
    let mut seconds = millis / 1_000;
    let minutes = seconds / 60;
    seconds %= 60;
    format!("{}:{:02}", minutes, seconds)
}
```

我们使用了另一种创建小部件的方法：

```rs
gtk::Scale(Horizontal, &self.model.adjustment) {
    draw_value: false,
    hexpand: true,
}
```

这个语法将调用小部件的构造函数，如下所示：

```rs
gtk::Scale::new(Horizontal, &self.model.adjustment);
```

我们也可以使用传统的语法来创建小部件：

```rs
use gtk::RangeExt;

gtk::Scale {
    adjustment: &self.model.adjustment,
    orientation: Horizontal,
    draw_value: false,
    hexpand: true,
}
```

这只是做同样事情的两个方法。

# 对话框

对于打开和保存对话框，我们将使用与上一章相同的功能：

```rs
use std::path::PathBuf;

use gtk::{FileChooserAction, FileChooserDialog, FileFilter};
use gtk_sys::{GTK_RESPONSE_ACCEPT, GTK_RESPONSE_CANCEL};

const RESPONSE_ACCEPT: i32 = GTK_RESPONSE_ACCEPT as i32;
const RESPONSE_CANCEL: i32 = GTK_RESPONSE_CANCEL as i32;

fn show_open_dialog(parent: &Window) -> Option<PathBuf> {
    let mut file = None;
    let dialog = FileChooserDialog::new(Some("Select an MP3 audio file"), 
    Some(parent), FileChooserAction::Open);

    let mp3_filter = FileFilter::new();
    mp3_filter.add_mime_type("audio/mp3");
    mp3_filter.set_name("MP3 audio file");
    dialog.add_filter(&mp3_filter);

    let m3u_filter = FileFilter::new();
    m3u_filter.add_mime_type("audio/x-mpegurl");
    m3u_filter.set_name("M3U playlist file");
    dialog.add_filter(&m3u_filter);

    dialog.add_button("Cancel", RESPONSE_CANCEL);
    dialog.add_button("Accept", RESPONSE_ACCEPT);
    let result = dialog.run();
    if result == RESPONSE_ACCEPT {
        file = dialog.get_filename();
    }
    dialog.destroy();
    file
}

fn show_save_dialog(parent: &Window) -> Option<PathBuf> {
    let mut file = None;
    let dialog = FileChooserDialog::new(Some("Choose a destination M3U playlist 
    file"), Some(parent), FileChooserAction::Save);
    let filter = FileFilter::new();
    filter.add_mime_type("audio/x-mpegurl");
    filter.set_name("M3U playlist file");
    dialog.set_do_overwrite_confirmation(true);
    dialog.add_filter(&filter);
    dialog.add_button("Cancel", RESPONSE_CANCEL);
    dialog.add_button("Save", RESPONSE_ACCEPT);
    let result = dialog.run();
    if result == RESPONSE_ACCEPT {
        file = dialog.get_filename();
    }
    dialog.destroy();
    file
}
```

但这次，我们将打开操作的代码放在 `App` 小部件的方法中：

```rs
use gtk::{ButtonsType, DialogFlags, MessageDialog, MessageType};

impl App {
    fn open(&self) {
        let file = show_open_dialog(&self.window);
        if let Some(file) = file {
            let ext = file.extension().map(|ext| 
             ext.to_str().unwrap().to_string());
            if let Some(ext) = ext {
                match ext.as_str() {
                    "mp3" => (),
                    "m3u" => (),
                    extension => {
                        let dialog = 
                        MessageDialog::new(Some(&self.window),  
                        DialogFlags::empty(), MessageType::Error,
                        ButtonsType::Ok, &format!("Cannot open file 
                         with extension . {}", extension));
                        dialog.run();
                        dialog.destroy();
                    },
                }
            }
        }
    }
}
```

我们可以在 `update()` 方法中调用这些函数：

```rs
fn update(&mut self, event: Msg) {
    match event {
        Open => self.open(),
        PlayPause =>  (),
        Quit => gtk::main_quit(),
        Save => show_save_dialog(&self.window),
        Stop => (),
    }
}
```

让我们管理一些其他操作。

# 其他方法

这将需要在 `impl Widget` 中添加两个新方法：

```rs
#[widget]
impl Widget for App {
    // …

    fn set_current_time(&mut self, time: u64) {
        self.model.current_time = time;
        self.model.adjustment.set_value(time as f64);
    }

    fn set_play_icon(&self, icon: &str) {
        self.model.play_image.set_from_file(format!("assets/{}.png", icon));
    }
}
```

但这些方法与`Widget`没有任何关系，那么为什么我们可以在特质实现中添加`custom`方法呢？嗯，`#[widget]`属性会将这些方法移动到属于它们的单独的`impl App`中。但为什么我们想要这样做而不是自己放置它们呢？那是因为`relm`会分析由`#[widget]`属性装饰的实现中的方法对模型属性的赋值。正如我们之前看到的，对模型字段的赋值将自动更新视图。如果我们将这些方法放在单独的`impl App`中，`relm`将无法分析这些方法并生成自动更新视图的代码。

这是一个常见的错误，如果你在分配给模型属性时视图没有更新，那么可能是因为你的分配不在由`#[widget]`属性装饰的实现中。

我们还需要为我们的模型添加一个新属性：

```rs
pub struct Model {
    adjustment: Adjustment,
    cover_pixbuf: Option<Pixbuf>,
    cover_visible: bool,
    current_duration: u64,
    current_time: u64,
    play_image: Image,
    stopped: bool,
}
```

我们添加了一个`stopped`属性，我们还需要在模型初始化中添加它：

```rs
fn model() -> Model {
    Model {
        adjustment: Adjustment::new(0.0, 0.0, 0.0, 0.0, 0.0, 0.0),
        cover_pixbuf: None,
        cover_visible: false,
        current_duration: 0,
        current_time: 0,
        play_image: new_icon(PLAY_ICON),
        stopped: true,
    }
}
```

我们现在可以将`update()`方法改为使用这些新方法：

```rs
fn update(&mut self, event: Msg) {
    match event {
        Open => self.open(),
        PlayPause =>  {
            if !self.model.stopped {
                self.set_play_icon(PLAY_ICON);
            }
        },
        Quit => gtk::main_quit(),
        Save => show_save_dialog(&self.window),
        Stop => {
            self.set_current_time(0);
            self.model.current_duration = 0;
            self.model.cover_visible = false;
            self.set_play_icon(PLAY_ICON);
        },
    }
}
```

`update()`方法通过可变引用接收`self`，这允许我们更新模型属性。

# 播放列表

现在我们已经准备好创建一个新的小部件：播放列表。我们需要以下新的`dependencies`：

```rs
[dependencies]
id3 = "⁰.2.0"
m3u = "¹.0.0"
```

添加它们相应的`extern crate`语句：

```rs
extern crate id3;
extern crate m3u;
```

让我们为我们的`playlist`创建一个新的模块：

```rs
mod playlist;
```

在`src/playlist.rs`文件中，我们首先创建我们的模型：

```rs
use gtk::ListStore;

pub struct Model {
    current_song: Option<String>,
    model: ListStore,
    relm: Relm<Playlist>,
}
```

`Relm`类型来自`relm`包：

```rs
use relm::Relm;
```

向小部件发送消息是有用的。我们将在关于小部件通信的部分了解更多。让我们添加模型初始化函数：

```rs
use gdk_pixbuf::Pixbuf;
use gtk::{StaticType, Type};

#[widget]
impl Widget for Playlist {
    fn model(relm: &Relm<Self>, _: ()) -> Model {
        Model {
            current_song: None,
            model: ListStore::new(&[
                Pixbuf::static_type(),
                Type::String,
                Type::String,
                Type::String,
                Type::String,
                Type::String,
                Type::String,
                Type::String,
                Pixbuf::static_type(),
            ]),
            relm: relm.clone(),
        }
    }
}
```

这里，我们注意到我们使用了`model()`方法的不同的签名。这是怎么可能的？特质的方法定义不能改变，对吧？这是`#[widget]`包带来的另一个便利。在许多情况下，我们不需要这些参数，所以如果需要，它们会自动添加。第一个参数是`relm`，我们将其副本保存在模型中。第二个参数是模型初始化参数。`ListStore`与第五章中的相同，*创建音乐播放器*，我们只将其保存在我们的模型中，因为我们以后会用到它。

# 模型参数

让我们更详细地谈谈这个第二个参数。它可以在创建小部件时用来向小部件发送数据。记得我们调用`run()`时：

```rs
App::run(()).unwrap();
```

这里，我们指定`()`作为模型参数，因为我们不需要它。但我们可以使用不同的值，例如`42`，这个值将会在`model()`方法的第二个参数中被接收。

现在我们已经准备好创建视图：

```rs
use gtk;
use gtk::{TreeViewExt, WidgetExt};
use relm::Widget;
use relm_attributes::widget;

#[widget]
impl Widget for Playlist {
    // …

    view! {
        #[name="treeview"]
        gtk::TreeView {
            hexpand: true,
            model: &self.model.model,
            vexpand: true,
        }
    }
}
```

它真的很简单：我们给它一个名字，并将`hexpand`和`vexpand`属性都设置为`true`，并将`model`属性绑定到我们的`ListStore`。

让我们现在创建一个空的`update()`方法：

```rs
#[widget]
impl Widget for Playlist {
    // …

    fn update(&mut self, event: Msg) {
    }
}
```

我们将在稍后看到 `Msg` 类型。现在，我们将添加列，就像我们在 [第五章](https://cdp.packtpub.com/rust_by_example/wp-admin/post.php?post=121&action=edit#post_86) 中 *创建音乐播放器* 所做的那样。让我们复制以下枚举和常量：

```rs
use self::Visibility::*;

#[derive(PartialEq)]
enum Visibility {
    Invisible,
    Visible,
}

const THUMBNAIL_COLUMN: u32 = 0;
const TITLE_COLUMN: u32 = 1;
const ARTIST_COLUMN: u32 = 2;
const ALBUM_COLUMN: u32 = 3;
const GENRE_COLUMN: u32 = 4;
const YEAR_COLUMN: u32 = 5;
const TRACK_COLUMN: u32 = 6;
const PATH_COLUMN: u32 = 7;
const PIXBUF_COLUMN: u32 = 8;
```

然后让我们向 `Paylist` 添加新方法：

```rs
impl Playlist {
    fn add_pixbuf_column(&self, column: i32, visibility: Visibility) {
        let view_column = TreeViewColumn::new();
        if visibility == Visible {
            let cell = CellRendererPixbuf::new();
            view_column.pack_start(&cell, true);
            view_column.add_attribute(&cell, "pixbuf", column);
        }
        self.treeview.append_column(&view_column);

    }

    fn add_text_column(&self, title: &str, column: i32) {
        let view_column = TreeViewColumn::new();
        view_column.set_title(title);
        let cell = CellRendererText::new();
        view_column.set_expand(true);
        view_column.pack_start(&cell, true);
        view_column.add_attribute(&cell, "text", column);
        self.treeview.append_column(&view_column);
    }

    fn create_columns(&self) {
        self.add_pixbuf_column(THUMBNAIL_COLUMN as i32, Visible);
        self.add_text_column("Title", TITLE_COLUMN as i32);
        self.add_text_column("Artist", ARTIST_COLUMN as i32);
        self.add_text_column("Album", ALBUM_COLUMN as i32);
        self.add_text_column("Genre", GENRE_COLUMN as i32);
        self.add_text_column("Year", YEAR_COLUMN as i32);
        self.add_text_column("Track", TRACK_COLUMN as i32);
        self.add_pixbuf_column(PIXBUF_COLUMN as i32, Invisible);
    }
}
```

与 [第五章](https://cdp.packtpub.com/rust_by_example/wp-admin/post.php?post=121&action=edit#post_86) 中 *创建音乐播放器* 的这些函数相比，这里的区别在于，我们直接通过属性访问 `treeview`。这需要新的导入语句：

```rs
use gtk::{
    CellLayoutExt,
    CellRendererPixbuf,
    CellRendererText,
    TreeViewColumn,
    TreeViewColumnExt,
    TreeViewExt,
};
```

现在，在 `init_view()` 方法中调用 `create_columns()` 方法：

```rs
#[widget]
impl Widget for Playlist {
    fn init_view(&mut self) {
        self.create_columns();
    }

    // …
}
```

让我们从与播放列表的交互开始。我们将创建一个将歌曲添加到播放列表的方法：

```rs
use std::path::Path;

use gtk::{ListStoreExt, ListStoreExtManual, ToValue};
use id3::Tag;

impl Playlist {
    fn add(&self, path: &Path) {
        let filename =  
         path.file_stem().unwrap_or_default().to_str().unwrap_or_default();

        let row = self.model.model.append();

        if let Ok(tag) = Tag::read_from_path(path) {
            let title = tag.title().unwrap_or(filename);
            let artist = tag.artist().unwrap_or("(no artist)");
            let album = tag.album().unwrap_or("(no album)");
            let genre = tag.genre().unwrap_or("(no genre)");
            let year = tag.year().map(|year|
            year.to_string()).unwrap_or("(no year)".to_string());
            let track = tag.track().map(|track|  
             track.to_string()).unwrap_or("??".to_string());
            let total_tracks = 
             tag.total_tracks().map(|total_tracks|  
             total_tracks.to_string()).unwrap_or("??".to_string());
            let track_value = format!("{} / {}", track, 
             total_tracks);

            self.set_pixbuf(&row, &tag);

            self.model.model.set_value(&row, TITLE_COLUMN, 
            &title.to_value());
            self.model.model.set_value(&row, ARTIST_COLUMN,
            &artist.to_value());
            self.model.model.set_value(&row, ALBUM_COLUMN, 
            &album.to_value());
            self.model.model.set_value(&row, GENRE_COLUMN, 
            &genre.to_value());
            self.model.model.set_value(&row, YEAR_COLUMN,
            &year.to_value());
            self.model.model.set_value(&row, TRACK_COLUMN,
            &track_value.to_value());
        }
        else {
            self.model.model.set_value(&row, TITLE_COLUMN, 
             &filename.to_value());
        }

        let path = path.to_str().unwrap_or_default();
        self.model.model.set_value(&row, PATH_COLUMN,
         &path.to_value());
    }
}
```

调用 `set_pixbuf()` 方法，因此让我们来定义它：

```rs
use gdk_pixbuf::{InterpType, PixbufLoader};
use gtk::TreeIter;

const INTERP_HYPER: InterpType = 3;

const IMAGE_SIZE: i32 = 256;
const THUMBNAIL_SIZE: i32 = 64;

fn set_pixbuf(&self, row: &TreeIter, tag: &Tag) {
    if let Some(picture) = tag.pictures().next() {
        let pixbuf_loader = PixbufLoader::new();
        pixbuf_loader.set_size(IMAGE_SIZE, IMAGE_SIZE);
        pixbuf_loader.loader_write(&picture.data).unwrap();
        if let Some(pixbuf) = pixbuf_loader.get_pixbuf() {
            let thumbnail = pixbuf.scale_simple(THUMBNAIL_SIZE, 
             THUMBNAIL_SIZE, INTERP_HYPER).unwrap();
            self.model.model.set_value(row, THUMBNAIL_COLUMN, 
             &thumbnail.to_value());
            self.model.model.set_value(row, PIXBUF_COLUMN, 
             &pixbuf.to_value());
        }
        pixbuf_loader.close().unwrap();
    }
}
```

它与 [第五章](https://cdp.packtpub.com/rust_by_example/wp-admin/post.php?post=121&action=edit#post_86) 中 *创建音乐播放器* 创建的方法非常相似。当接收到 `AddSong(path)` 消息时，将调用此方法，因此我们现在创建我们的消息类型：

```rs
use std::path::PathBuf;

use self::Msg::*;

#[derive(Msg)]
pub enum Msg {
    AddSong(PathBuf),
    LoadSong(PathBuf),
    NextSong,
    PauseSong,
    PlaySong,
    PreviousSong,
    RemoveSong,
    SaveSong(PathBuf),
    SongStarted(Option<Pixbuf>),
    StopSong,
}
```

然后，相应地修改 `update()` 方法：

```rs
   fn update(&mut self, event: Msg) {
      match event {
          AddSong(path) => self.add(&path),
          LoadSong(path) => (),
          NextSong => (),
          PauseSong => (),
          PlaySong => (),
          PreviousSong => (),
          RemoveSong => (),
          SaveSong(path) => (),
          SongStarted(_) => (),
          StopSong => (),
        }
    }
```

在这里，当接收到 `AddSong` 消息时，我们调用 `add()` 方法。但这条消息是从哪里发出的？嗯，它将由 `App` 类型发出，当用户请求打开文件时。现在是时候回到主模块并使用这个新的 `relm` 小部件了。

# 添加 relm 小部件

首先，我们需要这些新的导入语句：

```rs
use playlist::Playlist;
use playlist::Msg::{
    AddSong,
    LoadSong,
    NextSong,
    PlaySong,
    PauseSong,
    PreviousSong,
    RemoveSong,
    SaveSong,
    SongStarted,
    StopSong,
};
```

然后，在工具栏下方添加 `Playlist` 小部件：

```rs
view! {
    #[name="window"]
    gtk::Window {
        title: "Rusic",
        gtk::Box {
            orientation: Vertical,
            #[name="toolbar"]
            gtk::Toolbar {
                // …
            },
            #[name="playlist"]
            Playlist {
            },
            gtk::Image {
                from_pixbuf: self.model.cover_pixbuf.as_ref(),
                visible: self.model.cover_visible,
            },
            gtk::Box {
                // …
            },
        },
        delete_event(_, _) => (Quit, Inhibit(false)),
    }
}
```

使用 `relm` 小部件和 `gtk` 小部件时有一些不同。`Relm` 小部件不得包含模块前缀，而 `gtk` 小部件必须包含一个。这就是为什么我们导入了 `Playlist`，但现在导入了 `gtk::Toolbar`。但为什么需要它呢？嗯，`relm` 小部件与 `gtk` 小部件不同，因此它们不是以相同的方式创建或添加到其他小部件中的。因此，`relm` 可以通过这种方式区分它们：如果有前缀，这是一个内置的 `gtk` 小部件，否则它是一个自定义的 `relm` 小部件。当我说 `gtk` 小部件时，这甚至包括来自其他包的 `gtk` 小部件，例如 `webkit2gtk::WebView`。

# 小部件之间的通信

现在，我们将在小部件之间进行通信，以表示我们想要将歌曲添加到播放列表中。但在这样做之前，我们将更详细地了解小部件如何与自身通信。

# 与同一小部件的通信

我们之前看到了如何与同一小部件通信。要从视图中的事件处理器向同一小部件发送消息，我们只需在 `=>` 的右侧指定要发送的消息，如下例所示：

```rs
gtk::ToolButton {
    icon_widget: &new_icon("gtk-quit"),
    clicked => Quit,
}
```

在这里，当用户点击此工具按钮时，将向同一小部件（即 `App`）发送 `Quit` 消息。但这是对 `relm` 小部件事件流上的 `emit()` 方法的调用的一种语法糖。

# 发射

因此，让我们看看如何在不使用这种语法的情况下向同一个小部件发送消息：这在更复杂的情况下很有用，例如当我们想要有条件地发送消息时。让我们回到我们的`Playlist`并添加一个`play()`方法：

```rs
impl Playlist {
    fn play(&mut self) {
        if let Some(path) = self.selected_path() {
            self.model.current_song = Some(path.into());
            self.model.relm.stream().emit(SongStarted(self.pixbuf()));
        }
    }
}
```

这行代码向当前小部件发送一个消息：

```rs
self.model.relm.stream().emit(SongStarted(self.pixbuf()));
```

我们首先从`relm`小部件获取事件流，然后在上面调用`emit()`方法并传递一个消息。这个`play()`方法需要两个新方法：

```rs
use gtk::{
    TreeModelExt,
    TreeSelectionExt,
};

impl Playlist {
    fn pixbuf(&self) -> Option<Pixbuf> {
        let selection = self.treeview.get_selection();
        if let Some((_, iter)) = selection.get_selected() {
            let value = self.model.model.get_value(&iter, 
             PIXBUF_COLUMN as i32);
            return value.get::<Pixbuf>();
        }
        None
    }

    fn selected_path(&self) -> Option<String> {
        let selection = self.treeview.get_selection();
        if let Some((_, iter)) = selection.get_selected() {
            let value = self.model.model.get_value(&iter, PATH_COLUMN as i32);
            return value.get::<String>();
        }
        None
    }
}
```

这些与我们在前几章中编写的非常相似。我们现在可以在`update()`方法中调用`play()`方法：

```rs
    fn update(&mut self, event: Msg) {
        match event {
            AddSong(path) => self.add(&path),
            LoadSong(path) => (),
            NextSong => (),
            PauseSong => (),
            PlaySong => self.play(),
            PreviousSong => (),
            RemoveSong => (),
            SaveSong(path) => (),
            // To be listened by App.
            SongStarted(_) => (),
            StopSong => (),
        }
    }
```

我还在`SongStarted`之前添加了一个注释，表明这个消息不会被`Paylist`小部件处理，而是由`App`小部件处理。现在，让我们看看如何在不同的小部件之间进行通信。

# 使用不同的小部件

让我们更新`open()`方法以与播放列表进行通信：

```rs
impl App {
    fn open(&self) {
        let file = show_open_dialog(&self.window);
        if let Some(file) = file {
            let ext = file.extension().map(|ext| ext.to_str().unwrap().to_string());
            if let Some(ext) = ext {
                match ext.as_str() {
                    "mp3" => self.playlist.emit(AddSong(file)),
                    "m3u" => self.playlist.emit(LoadSong(file)),
                    extension => {
                        let dialog = MessageDialog::new(Some(&self.window),  
                        DialogFlags::empty(), MessageType::Error,
                        ButtonsType::Ok, &format!("Cannot open file with 
                         extension . {}", extension));
                        dialog.run();
                        dialog.destroy();
                    },
                }
            }
        }
    }
}
```

因此，我们调用相同的`emit()`方法向另一个小部件发送消息：

```rs
self.playlist.emit(AddSong(file))
```

这里，我们发送了一个尚未由`Playlist`（`LoadSong`）处理的消息，所以让我们修复这个问题：

```rs
use m3u;

impl Playlist {
    fn load(&self, path: &Path) {
        let mut reader = m3u::Reader::open(path).unwrap();
        for entry in reader.entries() {
            if let Ok(m3u::Entry::Path(path)) = entry {
                self.add(&path);
            }
        }
    }
}
```

这个方法在`update()`方法中被调用：

```rs
fn update(&mut self, event: Msg) {
    match event {
        AddSong(path) => self.add(&path),
        LoadSong(path) => self.load(&path),
        NextSong => (),
        PauseSong => (),
        PlaySong => self.play(),
        PreviousSong => (),
        RemoveSong => (),
        SaveSong(path) => (),
        // To be listened by App.
        SongStarted(_) => (),
        StopSong => (),
    }
}
```

# 处理来自 relm 小部件的消息

现在，让我们看看如何处理`SongStarted`消息。为了做到这一点，我们使用与处理`gtk`事件类似的语法。消息位于`=>`的左侧，而处理程序位于其右侧：

```rs
#[widget]
impl Widget for App {
    // …

    view! {
        // …
        #[name="playlist"]
        Playlist {
            SongStarted(ref pixbuf) => Started(pixbuf.clone()),
        }
    }
}
```

我们可以看到，当我们从播放列表接收到`SongStarted`消息时，我们在同一个小部件（`App`）上发出`Started`消息。我们需要在这里使用`ref`然后`clone()`消息中包含的值，因为我们不拥有这个消息。确实，多个小部件可以监听同一个消息，即发出消息的小部件及其父小部件。在我们处理这个新消息之前，我们将它添加到我们的`Msg`枚举中：

```rs
#[derive(Msg)]
pub enum Msg {
    Open,
    PlayPause,
    Quit,
    Save,
    Started(Option<Pixbuf>),
    Stop,
}
```

这个变体接受一个可选的`pixbuf`，因为一些 MP3 文件内部没有封面图像。以下是处理这个消息的方法：

```rs
fn update(&mut self, event: Msg) {
    match event {
        // …
        Started(pixbuf) => {
            self.set_play_icon(PAUSE_ICON);
            self.model.cover_visible = true;
            self.model.cover_pixbuf = pixbuf;
        },
    }
}
```

当歌曲开始播放时，我们显示暂停图标和封面。

# 向另一个 relm 小部件发送消息的语法糖

使用`emit()`向另一个小部件发送消息有点冗长，所以`relm`为此情况提供了语法糖。让我们在用户点击删除按钮时向播放列表发送消息：

```rs
gtk::ToolButton {
    icon_widget: &new_icon("remove"),
    clicked => playlist@RemoveSong,
}
```

在这里，我们使用了`@`语法来指定消息将被发送到另一个小部件。`@`之前的部分是接收器小部件，而`@`之后的部分是消息。因此，这段代码的意思是，每当用户点击删除按钮时，将`RemoveSong`消息发送到`playlist`小部件。

让我们在`Paylist::update()`方法中处理这个消息：

```rs
#[widget]
impl Widget for Playlist {
    fn update(&mut self, event: Msg) {
        match event {
            AddSong(path) => self.add(&path),
            LoadSong(path) => self.load(&path),
            NextSong => (),
            PauseSong => (),
            PlaySong => self.play(),
            PreviousSong => (),
            RemoveSong => self.remove_selection(),
            SaveSong(path) => (),
            // To be listened by App.
            SongStarted(_) => (),
            StopSong => (),
        }
    }

    // …
}
```

这将调用`remove_selection()`方法，如下所示：

```rs
fn remove_selection(&self) {
    let selection = self.treeview.get_selection();
    if let Some((_, iter)) = selection.get_selected() {
        self.model.model.remove(&iter);
    }
}
```

这与第五章中提到的相同方法，*创建音乐播放器*。现在，让我们发送剩余的消息。`PlaySong`、`PauseSong`、`SaveSong`和`StopSong`消息在`update()`方法中发出：

```rs
#[widget]
impl Widget for App {
    fn update(&mut self, event: Msg) {
        match event {
            PlayPause =>  {
                if self.model.stopped {
                    self.playlist.emit(PlaySong);
                } else {
                    self.playlist.emit(PauseSong);
                    self.set_play_icon(PLAY_ICON);
                }
            },
            Save => {
                let file = show_save_dialog(&self.window);
                if let Some(file) = file {
                    self.playlist.emit(SaveSong(file));
                }
            },
            Stop => {
                self.set_current_time(0);
                self.model.current_duration = 0;
                self.playlist.emit(StopSong);
                self.model.cover_visible = false;
                self.set_play_icon(PLAY_ICON);
            },
            // …
        }
    }
}
```

其他消息使用视图中的 `@` 语法发送：

```rs
view! {
    #[name="window"]
    gtk::Window {
        title: "Rusic",
        gtk::Box {
            orientation: Vertical,
            #[name="toolbar"]
            gtk::Toolbar {
                // …
                gtk::ToolButton {
                    icon_widget: &new_icon("gtk-media-previous"),
                    clicked => playlist@PreviousSong,
                },
                // …
                gtk::ToolButton {
                    icon_widget: &new_icon("gtk-media-next"),
                    clicked => playlist@NextSong,
                },
            },
            // …
        },
        delete_event(_, _) => (Quit, Inhibit(false)),
    }
}
```

我们将在 `Paylist::update()` 方法中处理这些消息：

```rs
fn update(&mut self, event: Msg) {
    match event {
        AddSong(path) => self.add(&path),
        LoadSong(path) => self.load(&path),
        NextSong => self.next(),
        PauseSong => (),
        PlaySong => self.play(),
        PreviousSong => self.previous(),
        RemoveSong => self.remove_selection(),
        SaveSong(path) => self.save(&path),
        // To be listened by App.
        SongStarted(_) => (),
        StopSong => self.stop(),
    }
}
```

这需要一些新方法：

```rs
fn next(&mut self) {
    let selection = self.treeview.get_selection();
    let next_iter =
        if let Some((_, iter)) = selection.get_selected() {
            if !self.model.model.iter_next(&iter) {
                return;
            }
            Some(iter)
        }
        else {
            self.model.model.get_iter_first()
        };
    if let Some(ref iter) = next_iter {
        selection.select_iter(iter);
        self.play();
    }
}
```

```rs
fn previous(&mut self) {
    let selection = self.treeview.get_selection();
    let previous_iter =
        if let Some((_, iter)) = selection.get_selected() {
            if !self.model.model.iter_previous(&iter) {
                return;
            }
            Some(iter)
        }
        else {
            self.model.model.iter_nth_child(None, max(0,  
            self.model.model.iter_n_children(None) - 1))
        };
    if let Some(ref iter) = previous_iter {
        selection.select_iter(iter);
        self.play();
    }
}
```

```rs
use std::fs::File;

fn save(&self, path: &Path) {
    let mut file = File::create(path).unwrap();
    let mut writer = m3u::Writer::new(&mut file);

    let mut write_iter = |iter: &TreeIter| {
        let value = self.model.model.get_value(&iter, PATH_COLUMN as i32);
        let path = value.get::<String>().unwrap();
        writer.write_entry(&m3u::path_entry(path)).unwrap();
    };

    if let Some(iter) = self.model.model.get_iter_first() {
        write_iter(&iter);
        while self.model.model.iter_next(&iter) {
            write_iter(&iter);
        }
    }
}
```

并且函数 `stop`:

```rs
fn stop(&mut self) {
    self.model.current_song = None;
}
```

这些方法都与我们在上一章中创建的方法相似。你可以运行应用程序以查看我们可以打开和删除歌曲，但我们还不能播放它们。所以让我们解决这个问题。

# 播放音乐

首先，添加 `mp3` 模块：

```rs
mod mp3;
```

将上一章的 `src/mp3.rs` 文件复制过来。

我们还需要以下依赖项：

```rs
[dependencies]
crossbeam = "⁰.3.0"
futures = "⁰.1.16"
pulse-simple = "¹.0.0"
simplemad = "⁰.8.1"
```

然后将这些语句添加到 `main` 模块中：

```rs
extern crate crossbeam;
extern crate futures;
extern crate pulse_simple;
extern crate simplemad;
```

现在，我们将添加一个 `player` 模块：

```rs
mod player;
```

这个新模块将开始于一系列导入语句：

```rs
use std::cell::Cell;
use std::fs::File;
use std::io::BufReader;
use std::path::{Path, PathBuf};
use std::sync::{Arc, Condvar, Mutex};
use std::thread;
use std::time::Duration;

use crossbeam::sync::SegQueue;
use futures::{AsyncSink, Sink};
use futures::sync::mpsc::UnboundedSender;
use pulse_simple::Playback;

use mp3::Mp3Decoder;
use playlist::PlayerMsg::{
    self,
    PlayerPlay,
    PlayerStop,
    PlayerTime,
};
use self::Action::*;
```

我们从 `playlist` 模块导入了新的 `PlayerMsg` 类型，所以让我们添加它：

```rs
#[derive(Clone)]
pub enum PlayerMsg {
    PlayerPlay,
    PlayerStop,
    PlayerTime(u64),
}
```

我们将定义一些常量：

```rs
const BUFFER_SIZE: usize = 1000;
const DEFAULT_RATE: u32 = 44100;
```

然后让我们创建我们需要的类型：

```rs
enum Action {
    Load(PathBuf),
    Stop,
}

#[derive(Clone)]
struct EventLoop {
    condition_variable: Arc<(Mutex<bool>, Condvar)>,
    queue: Arc<SegQueue<Action>>,
    playing: Arc<Mutex<bool>>,
}

pub struct Player {
    event_loop: EventLoop,
    paused: Cell<bool>,
    tx: UnboundedSender<PlayerMsg>,
}
```

`Action` 和 `EventLoop` 与上一章相同，但 `Player` 类型略有不同。它不包含表示应用程序状态的字段，而是包含一个用于向播放列表和最终向应用程序本身发送消息的发送者。因此，我们不会像上一章那样使用共享状态和超时，而是使用消息传递，这更有效率。

我们需要一个 `EventLoop` 的构造函数：

```rs
impl EventLoop {
    fn new() -> Self {
        EventLoop {
            condition_variable: Arc::new((Mutex::new(false), Condvar::new())),
            queue: Arc::new(SegQueue::new()),
            playing: Arc::new(Mutex::new(false)),
        }
    }
}
```

让我们为 `Player` 创建构造函数：

```rs
impl Player {
    pub(crate) fn new(tx: UnboundedSender<PlayerMsg>) -> Self {
        let event_loop = EventLoop::new();

        {
            let mut tx = tx.clone();
            let event_loop = event_loop.clone();
            let condition_variable = event_loop.condition_variable.clone();
            thread::spawn(move || {
                let block = || {
                    let (ref lock, ref condition_variable) = *condition_variable;
                    let mut started = lock.lock().unwrap();
                    *started = false;
                    while !*started {
                        started = condition_variable.wait(started).unwrap();
                    }
                };

                let mut buffer = [[0; 2]; BUFFER_SIZE];
                let mut playback = Playback::new("MP3", "MP3 Playback", None,  
                DEFAULT_RATE);
                let mut source = None;
                loop {
                    if let Some(action) = event_loop.queue.try_pop() {
                        match action {
                            Load(path) => {
                                let file = File::open(path).unwrap();
                                source = 
                Some(Mp3Decoder::new(BufReader::new(file)).unwrap());
                                let rate = source.as_ref().map(|source|  
                                source.samples_rate()).unwrap_or(DEFAULT_RATE);
                                playback = Playback::new("MP3", "MP3 Playback",  
                                 None, rate);
                                send(&mut tx, PlayerPlay);
                            },
                            Stop => {
                                source = None;
                            },
                        }
                    } else if *event_loop.playing.lock().unwrap() {
                        let mut written = false;
                        if let Some(ref mut source) = source {
                            let size = iter_to_buffer(source, &mut buffer);
                            if size > 0 {
                                send(&mut tx, PlayerTime(source.current_time()));
                                playback.write(&buffer[..size]);
                                written = true;
                            }
                        }

                        if !written {
                            send(&mut tx, PlayerStop);
                            *event_loop.playing.lock().unwrap() = false;
                            source = None;
                            block();
                        }
                    } else {
                        block();
                    }
                }
            });
        }

        Player {
            event_loop,
            paused: Cell::new(false),
            tx,
        }
    }
}
```

它与我们在上一章中编写的类似，但不是使用共享状态，而是将消息发送回播放列表。以下是如何发送这些消息的示例：

```rs
send(&mut tx, PlayerTime(source.current_time()));
```

这会将当前时间发送回 UI，以便它可以显示它。这需要定义 `send()` 函数：

```rs
fn send(tx: &mut UnboundedSender<PlayerMsg>, msg: PlayerMsg) {
    if let Ok(AsyncSink::Ready) = tx.start_send(msg) {
        tx.poll_complete().unwrap();
    } else {
        eprintln!("Unable to send message to sender");
    }
}
```

此代码使用 `future` crate 发送消息，并在失败时显示错误。`iter_to_buffer()` 函数与上一章中的相同：

```rs
fn iter_to_buffer<I: Iterator<Item=i16>>(iter: &mut I, buffer: &mut [[i16; 2]; BUFFER_SIZE]) -> usize {
    let mut iter = iter.take(BUFFER_SIZE);
    let mut index = 0;
    while let Some(sample1) = iter.next() {
        if let Some(sample2) = iter.next() {
            buffer[index][0] = sample1;
            buffer[index][1] = sample2;
        }
        index += 1;
    }
    index
}
```

现在，我们将添加播放和暂停歌曲的方法：

```rs
pub fn load<P: AsRef<Path>>(&self, path: P) {
    let pathbuf = path.as_ref().to_path_buf();
    self.emit(Load(pathbuf));
    self.set_playing(true);
}

pub fn pause(&mut self) {
    self.paused.set(true);
    self.send(PlayerStop);
    self.set_playing(false);
}

pub fn resume(&mut self) {
    self.paused.set(false);
    self.send(PlayerPlay);
    self.set_playing(true);
}
```

它们与上一章的非常相似，但我们发送一个消息而不是修改状态。它们需要以下方法：

```rs
fn emit(&self, action: Action) {
    self.event_loop.queue.push(action);
}

fn send(&mut self, msg: PlayerMsg) {
    send(&mut self.tx, msg);
}

fn set_playing(&self, playing: bool) {
    *self.event_loop.playing.lock().unwrap() = playing;
    let (ref lock, ref condition_variable) = *self.event_loop.condition_variable;
    let mut started = lock.lock().unwrap();
    *started = playing;
    if playing {
        condition_variable.notify_one();
    }
}
```

`emit()` 和 `set_playing()` 方法与上一章相同。`send()` 方法只是调用我们之前定义的 `send()` 函数。

我们还需要这两个方法：

```rs
pub fn is_paused(&self) -> bool {
    self.paused.get()
}

pub fn stop(&mut self) {
    self.paused.set(false);
    self.send(PlayerTime(0));
    self.send(PlayerStop);
    self.emit(Stop);
    self.set_playing(false);
}
```

`is_paused()` 方法没有变化。而 `stop()` 方法类似，但同样，它发送消息而不是直接更新应用程序状态。让我们回到我们的 `Paylist` 来使用这个新的播放器。现在模型将包含播放器本身：

```rs
use player::Player;

pub struct Model {
    current_song: Option<String>,
    player: Player,
    model: ListStore,
    relm: Relm<Playlist>,
}
```

`Msg` 类型将包含一个名为 `PlayerMsgRecv` 的新变体，每当播放器发送消息时都会发出：

```rs
#[derive(Msg)]
pub enum Msg {
    AddSong(PathBuf),
    LoadSong(PathBuf),
    NextSong,
    PauseSong,
    PlayerMsgRecv(PlayerMsg),
    PlaySong,
    PreviousSong,
    RemoveSong,
    SaveSong(PathBuf),
    SongStarted(Option<Pixbuf>),
    StopSong,
}
```

现在，我们已经准备好更新模型初始化：

```rs
use futures::sync::mpsc;

fn model(relm: &Relm<Self>, _: ()) -> Model {
    let (tx, rx) = mpsc::unbounded();
    relm.connect_exec_ignore_err(rx, PlayerMsgRecv);
    Model {
        current_song: None,
        player: Player::new(tx),
        model: ListStore::new(&[
            Pixbuf::static_type(),
            Type::String,
            Type::String,
            Type::String,
            Type::String,
            Type::String,
            Type::String,
            Type::String,
            Pixbuf::static_type(),
        ]),
        relm: relm.clone(),
    }
}
```

现在它从`future` crate 的`mpsc`类型创建一个发送者和接收者对。**MPSC**代表**Multiple-Producers-Single-Consumer**。我们现在调用`Relm::connect_exec_ignore_err()`方法，这个方法将一个`Future`或一个`Stream`连接到一个消息。这意味着每当`Stream`中产生一个值时，就会发出一个消息。这个消息需要接受一个与`Stream`产生的值相同类型的参数。一个`Future`代表一个可能尚未可用，但将来会可用，除非发生错误。一个`Stream`类似，但可以在未来不同时间产生多个值。与`connect_exec_ignore_err()`方法类似，还有一个`connect_exec()`方法，它接受另一个消息变体作为参数，当发生错误时，将发出第二个消息。在这里，我们简单地忽略错误。

在`update()`方法中：

```rs
fn update(&mut self, event: Msg) {
    match event {
        // To be listened by App.
        PlayerMsgRecv(_) => (),
        // …
    }
}
```

我们与这个消息无关，因为它将由`App`小部件处理。我们现在将添加一个暂停播放器的函数：

```rs
fn pause(&mut self) {
    self.model.player.pause();
}
```

接下来，我们需要更新`play()`和`stop()`方法：

```rs
fn play(&mut self) {
    if let Some(path) = self.selected_path() {
        if self.model.player.is_paused() && Some(&path) == self.path().as_ref() {
            self.model.player.resume();
        } else {
            self.model.player.load(&path);
            self.model.current_song = Some(path.into());
            self.model.relm.stream().emit(SongStarted(self.pixbuf()));
        }
    }
}

fn stop(&mut self) {
    self.model.current_song = None;
    self.model.player.stop();
}
```

`stop()`方法与之前相同，只是我们可以直接更新模型，因为我们不再需要使用`RefCell`类型。`play()`方法现在将根据播放器的状态加载或恢复歌曲。

`play()`方法需要一个`path()`方法：

```rs
fn path(&self) -> Option<String> {
    self.model.current_song.clone()
}
```

让我们回到`main`模块来管理播放器发送的消息。首先，我们需要为我们的`enum Msg`添加一个新的变体：

```rs
#[derive(Msg)]
pub enum Msg {
    MsgRecv(PlayerMsg),
    // …
}
```

我们将在`update()`方法中处理这个问题：

```rs
fn update(&mut self, event: Msg) {
    match event {
        MsgRecv(player_msg) => self.player_message(player_msg),
        // …
    }
}
```

这需要在`impl Widget for App`中添加一个新的方法：

```rs
#[widget]
impl Widget for App {
    fn player_message(&mut self, player_msg: PlayerMsg) {
        match player_msg {
            PlayerPlay => {
                self.model.stopped = false;
                self.set_play_icon(PAUSE_ICON);
            },
            PlayerStop => {
                self.set_play_icon(PLAY_ICON);
                self.model.stopped = true;
            },
            PlayerTime(time) => self.set_current_time(time),
        }
    }
}
```

这也是一个`custom`方法，即不是`Widget`特质的组成部分，但由`#[widget]`属性分析的方法。我们将其放在那里而不是单独的`impl App`中，因为我们更新了模型。在这个方法中，我们要么更新图标以显示播放按钮，要么显示当前时间。

# 计算歌曲时长

为了与上一章的音乐播放器相匹配，唯一需要实现的功能是计算并显示歌曲时长。首先，我们将从上一章复制`compute_duration()`方法并将其粘贴到我们的`Player`中：

```rs
pub fn compute_duration<P: AsRef<Path>>(path: P) -> Option<Duration> {
    let file = File::open(path).unwrap();
    Mp3Decoder::compute_duration(BufReader::new(file))
}
```

我们现在将在`Playlist`中调用这个方法：

```rs
use std::thread;
use futures::sync::oneshot;

fn compute_duration(&self, path: &Path) {
    let path = path.to_path_buf();
    let (tx, rx) = oneshot::channel();
    thread::spawn(move || {
        if let Some(duration) = Player::compute_duration(&path) {
            tx.send((path, duration))
                .expect("Cannot send computed duration");
        }
    });
    self.model.relm.connect_exec_ignore_err(rx, |(path, duration)| DurationComputed(path, duration));
}
```

这里，我们使用`oneshot`，它也是一个通道，类似于`mpsc`，但`oneshot`只能发送一次消息。发送的消息是一个元组，因此我们通过使用一个新添加的`DurationComputed`变体将其转换为我们的`Msg`类型：

```rs
use std::time::Duration;

#[derive(Msg)]
pub enum Msg {
    AddSong(PathBuf),
    DurationComputed(PathBuf, Duration),
    SongDuration(u64),
    // …
}
```

我们还添加了一个即将使用的`SongDuration`消息。

我们需要在`Playlist::add()`中调用这个方法：

```rs
impl Playlist {
    fn add(&self, path: &Path) {
        self.compute_duration(path);
        // …
    }
}
```

然后，我们需要在`Playlist::update()`中处理新的`DurationComputed`消息：

```rs
use to_millis;

fn update(&mut self, event: Msg) {
    match event {
        DurationComputed(path, duration) => {
            let path = path.to_string_lossy().to_string();
            if self.model.current_song.as_ref() == Some(&path) {
                self.model.relm.stream().emit(SongDuration(to_millis(duration)));
            }
            self.model.durations.insert(path, to_millis(duration));
        },
        // To be listened by App.
        SongDuration(_) => (),
        // …
    }
}
```

这里，我们将计算出的时长插入到模型中。如果歌曲是当前正在播放的，我们将发送`SongDuration`消息，以便`App`小部件可以更新自己。

这需要在模型中的时长添加一个新的字段：

```rs
use std::collections::HashMap;

pub struct Model {
    current_song: Option<String>,
    durations: HashMap<String, u64>,
    player: Player,
    model: ListStore,
    relm: Relm<Playlist>,
}
```

添加新的模型初始化：

```rs
fn model(relm: &Relm<Self>, _: ()) -> Model {
    // …
    Model {
        durations: HashMap::new(),
        // …
    }
}
```

这也要求在 `main` 模块中添加 `to_millis()` 函数，这与上一章相同：

```rs
use std::time::Duration;

fn to_millis(duration: Duration) -> u64 {
    duration.as_secs() * 1000 + duration.subsec_nanos() as u64 / 1_000_000
}
```

由于持续时间只计算一次，因此在我们开始播放歌曲时也需要发送它，所以让我们更新 `Playlist::play()` 方法：

```rs
fn play(&mut self) {
    if let Some(path) = self.selected_path() {
        if self.model.player.is_paused() && Some(&path) == self.path().as_ref() {
            self.model.player.resume();
        } else {
            self.model.player.load(&path);
            if let Some(&duration) = self.model.durations.get(&path) {
                self.model.relm.stream().emit(SongDuration(duration));
            }
            self.model.current_song = Some(path.into());
            self.model.relm.stream().emit(SongStarted(self.pixbuf()));
        }
    }
}
```

如果我们在 `HashMap` 中找到了 `SongDuration` 消息（歌曲可能在计算持续时间之前开始播放），我们将发送 `SongDuration` 消息。

最后，我们需要在 `App` 视图中处理以下消息：

```rs
view! {
    Playlist {
        PlayerMsgRecv(ref player_msg) => MsgRecv(player_msg.clone()),
        SongDuration(duration) => Duration(duration),
        SongStarted(ref pixbuf) => Started(pixbuf.clone()),
    }
    // …
}
```

当我们从播放列表接收到 `SongDuration` 消息时，我们会向 `App` 发送 `Duration` 消息，因此我们需要将这个变体添加到它的 `Msg` 类型中：

```rs
#[derive(Msg)]
pub enum Msg {
    Duration(u64),
    // …
}
```

我们将在 `update()` 方法中简单地处理它：

```rs
fn update(&mut self, event: Msg) {
    match event {
        Duration(duration) => {
            self.model.current_duration = duration;
            self.model.adjustment.set_upper(duration as f64);
        },
        // …
    }
}
```

现在，你可以运行应用程序并看到它的工作方式与上一章中的完全相同。

# 在稳定版 Rust 上使用 relm

在整个这一章中，我们使用了 Rust 夜间版，以便能够使用当前不稳定的 `custom` 属性。`relm` 提供的 `#[widget]` 属性提供了许多优势：

+   声明性视图

+   数据绑定

+   输入更少

因此，能够在稳定版上使用类似的语法并且提供相同优势将会很棒。通过使用 `relm_widget!` 宏，我们可以做到这一点。我们将重写 `App` 小部件以使用这个宏：

```rs
relm_widget! {
    impl Widget for App {
        fn init_view(&mut self) {
            self.toolbar.show_all();
        }

        fn model() -> Model {
            Model {
                adjustment: Adjustment::new(0.0, 0.0, 0.0, 0.0, 0.0, 0.0),
                cover_pixbuf: None,
                cover_visible: false,
                current_duration: 0,
                current_time: 0,
                play_image: new_icon(PLAY_ICON),
                stopped: true,
            }
        }

        fn open(&self) {
            // …
        }

        // …

        fn update(&mut self, event: Msg) {
            // …
        }

        view! {
            #[name="window"]
            gtk::Window {
                title: "Rusic",
                // …
            }
        }
    }
}
```

如你所见，我们将外部 `open()` 方法移动到了由 `relm_widget!` 宏装饰的实现内部。这是由于这个宏的限制，虽然它允许我们在稳定版 Rust 上使用 relm 提供的漂亮语法，但我们无法从宏外部访问模型字段。其余部分与之前的版本完全相同。

# Relm 小部件数据绑定

relm 中还有许多其他功能可用，我想向你展示其中最重要的：提供的用于模拟属性绑定的语法。正如你可能已经注意到的，`relm` 小部件中没有属性，但你可以使用消息传递来更新 `relm` 小部件的内部状态。为了使其更方便，`#[widget]` 属性还允许你将模型属性绑定到消息，这意味着每当属性更新时，都会发出带有这个新值的消息。

我们将添加一个切换按钮，以便能够在播放列表的简单视图和详细视图之间切换。简单视图将只显示封面和标题，而详细视图将显示所有列。首先，让我们向 `App` 模型添加一个属性：

```rs
pub struct Model {
    detailed_view: bool,
    // …
}

    fn model() -> Model {
        Model {
            detailed_view: false,
            // …
        }
    }
```

此字段指定我们是否处于详细视图模式。我们还需要一个在点击切换按钮时将发出的消息：

```rs
#[derive(Msg)]
pub enum Msg {
    ViewToggle,
    // …
}
```

然后，我们将切换按钮添加到工具栏中：

```rs
#[name="toggle_button"]
gtk::ToggleToolButton {
    label: "Detailed view",
    toggled => ViewToggle,
}
```

当我们收到这个消息时，我们将相应地设置 `model` 属性：

```rs
fn update(&mut self, event: Msg) {
    match event {
        ViewToggle => self.model.detailed_view = self.toggle_button.get_active(),
        // …
    }
}
```

现在，让我们向 `Playlist` 发送一个消息：

```rs
#[derive(Msg)]
pub enum Msg {
    DetailedView(bool),
    // …
}
```

这是我们将用于绑定的消息。让我们来处理它：

```rs
fn update(&mut self, event: Msg) {
    match event {
        DetailedView(detailed) => self.set_detailed_view(detailed),
        // …
    }
}

fn set_detailed_view(&self, detailed: bool) {
    for column in self.treeview.get_columns().iter().skip(2) {
        column.set_visible(detailed);
    }
}
```

后者方法切换除前两个之外的所有列的可见性。现在我们可以在 `App` 视图中创建绑定：

```rs
use playlist::Msg::DetailedView;

view! {
    // …
    #[name="playlist"]
    Playlist {
        // …
        DetailedView: self.model.detailed_view,
    }
}
```

此代码将在指定的属性发生变化时发送 `DetailedView` 消息。

# 摘要

在本章中，我们使用 `relm` 创建了一个音乐播放器。我们看到了如何使用 `rustup` 与 rust nightly 版本结合使用是多么简单。我们学习了如何声明式地创建视图，并使用消息传递在部件之间进行通信。我们还学习了如何通过分离模型、视图以及更新模型的函数来结构化 GUI 应用程序。在下一章中，我们将切换到另一个项目：一个 FTP 服务器。
