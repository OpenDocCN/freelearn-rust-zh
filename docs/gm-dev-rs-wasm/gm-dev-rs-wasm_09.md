# 第七章：*第七章*：音效和音乐

抽空想想游戏俄罗斯方块。如果你像我一样，你可能已经哼起了它的主题曲《Korobeiniki》，因为这首歌与游戏本身如此同义。除了音乐的吸引力之外，音效对于创造沉浸式体验至关重要。我们玩游戏不仅仅是通过键盘或摇杆的触摸和我们的眼睛；我们听到马里奥跳跃或索尼克抓住一个圈。虽然我们的游戏可能是可玩的，但没有一些声音它就不是一个游戏。要在我们的游戏中播放声音，我们需要学习如何使用浏览器的 Web Audio API 来播放短的和长的声音。

在本章中，我们将涵盖以下主题：

+   将 Web Audio API 添加到引擎中

+   播放音效

+   播放长音乐

到本章结束时，你不仅会看到 RHB 跑、跳和躲避障碍，在我们为游戏添加音效和音乐后，你还将能够听到他的声音。让我们开始吧！

# 技术要求

技术要求与前面的章节基本相同。你需要从 [`github.com/PacktPublishing/Game-Development-with-Rust-and-WebAssembly/wiki/Assets`](https://github.com/PacktPublishing/Game-Development-with-Rust-and-WebAssembly/wiki/Assets) 的资产下载中的 `sound` 目录获取 `sound` 资产。

所有声音均来自开源声音集合，并已获得许可。更多信息请参阅 `sounds/credits.txt` 文件。本章的代码可在 [`github.com/PacktPublishing/Game-Development-with-Rust-and-WebAssembly/tree/chapter_7`](https://github.com/PacktPublishing/Game-Development-with-Rust-and-WebAssembly/tree/chapter_7) 找到。

查看以下视频以查看代码的实际应用：[`bit.ly/3JUdA2R`](https://bit.ly/3JUdA2R)

# 将 Web Audio API 添加到引擎中

在本节中，我们将使用浏览器的 Web Audio API 为我们的游戏添加声音。该 API 功能非常全面，允许混合音频源和特殊效果，但我们将仅使用它来播放背景音乐和声音。实际上，Web Audio API 是一本自己的书，如果你感兴趣，可以在 [`webaudioapi.com/book/`](https://webaudioapi.com/book/) 找到。虽然添加诸如空间化音频到我们的游戏会很有趣，但我们将专注于添加一些音乐和音效。我鼓励你在制作自己的更复杂游戏时进行实验。

一旦我们了解了 Web Audio API 的概览，我们将创建一个模块来在 Rust 中播放声音，以与加载我们的图片相同的方式加载声音，最后将那个声音添加到引擎中。

Web Audio API 是一种相对较新的技术，旨在取代旧的音频技术，如 QuickTime 和 Flash，同时比使用音频元素提供更灵活的解决方案。它被所有主要浏览器支持，只有旧版本的 Internet Explorer 可能存在问题。鉴于 Internet Explorer 的最后一次发布是在 2013 年，Windows 使用 Edge 浏览器代替，你的游戏可能不需要牺牲那个市场。

与 Canvas 相比，Web Audio API 可能一开始看起来很熟悉。与 Canvas 一样，你创建一个上下文，然后它提供了一个用于播放声音的 API。在那个点上，相似之处就结束了。因为 Web Audio API 具有我之前提到的所有功能，所以很难弄清楚如何进行基本的播放声音操作。与 Canvas 不同，没有`drawImage`等价物称为`playSound`或类似的东西。相反，你必须获取声音数据，创建`AudioBufferSourceNode`，将其连接到目的地，然后最后启动它。这可以实现一些非常令人印象深刻的效果（例如在[`webaudiodemos.appspot.com/`](https://webaudiodemos.appspot.com/)上找到的），但对于我们的游戏来说，我们将编写一次性的代码，然后把它忘掉。在 JavaScript 中，加载和准备声音以供播放的代码如下：

```rs
const audioContext = new AudioContext();
let sound = await fetch("SFX_Jump_23.mp3");
let soundBuffer = await sound.arrayBuffer();
let decodedArray = await audioContext.decodeAudioData(soundBuffer);
```

它首先创建一个新的`AudioContext`，这是内置在浏览器引擎中的，然后从服务器获取声音文件。`fetch`调用最终返回一个响应，我们需要对其进行解码。我们首先获取它的`arrayBuffer`，这会消耗它，然后我们使用我们最初创建的`audioContext`将缓冲区解码成可以播放的声音。注意，一切都是异步的，这将在我们将 JavaScript 承诺映射到 Rust 未来时给我们带来一点麻烦。之前的代码对于任何声音资源只应该执行一次，因为加载和解码文件可能需要很长时间。

以下代码将播放声音：

```rs
let trackSource = audioContext.createBufferSource();
trackSource.buffer = decodedArray;
trackSource.connect(audioContext.destination);
trackSource.start();
```

哎呀，这不太直观，但我们只能这样。幸运的是，我们可以用几个简单的函数把它包装起来，这样我们就能记住它，然后把它忘掉。它通过`createBufferSource`创建我们需要的`AudioBufferSourceNode`，将其分配给我们在上一节解码成音频数据的数组，连接到`audioContext`，最后用`start`播放声音。重要的是要知道，你不能对`trackSource`两次调用`start`，但幸运的是，缓冲源创建非常快，不需要我们缓存它。

太好了！我们知道在 JavaScript 中播放声音的八行代码，但我们如何将其放入我们的引擎中？

## 在 Rust 中播放声音

我们将创建一个`sound`模块，它与我们的`browser`模块非常相似，一系列只是将权利委托给底层 JavaScript 的函数。这将是一个自下而上的方法，我们将创建我们的实用函数，然后创建使用它们的最终函数。我们将首先关注`play_sound`函数所需的各个部分。

注意

记住，你希望这些函数非常小——这是 Rust 和 JavaScript 之间的*薄层*——但也要改变接口以更好地匹配你想要做的事情。所以，最终，我们不会谈论缓冲源和上下文，而是会调用我们最初希望存在的`play_sound`函数。

我们将首先在名为`sound.rs`的文件中创建模块，这个文件位于`src`目录中我们其他模块旁边。别忘了在`src/lib.rs`中添加对其的引用，如下所示：

```rs
#[macro_use]
mod browser;
mod engine;
mod game;
mod segments;
mod sound;
```

这是我总是忘记的部分。我们的第一个函数将以一种*Rusty*的方式创建`AudioContext`，而不是我们已经看到的 JavaScript 方式，如下所示：

```rs
use anyhow::{anyhow, Result};
use web_sys::AudioContext;
pub fn create_audio_context() -> Result<AudioContext> {
    AudioContext::new().map_err(|err| anyhow!
        ("Could not create audio context: {:#?}", err))
}
```

如往常一样，Rust 版本的代码比 JavaScript 版本更冗长。这是我们为 Rust 的优点所付出的代价。这些代码并没有什么特别新颖的地方；我们正在将`new AudioContext`映射到`AudioContext::new`，并将`JsResult`错误映射到可能返回的`anyhow`结果，使其更符合 Rust 的风格。然而，这段代码无法编译；花点时间思考一下原因。这是臭名昭著的`Cargo.toml`中`web-sys`的功能标志，我们还没有添加`AudioContext`，所以现在就添加它：

```rs
[dependencies.web-sys]
version = "0.3.55"
features = ["console",
           "Window",
           "Document",
           "HtmlCanvasElement",
           "CanvasRenderingContext2d",
           "Element",
           "HtmlImageElement",
           "Response",
           "Performance",
           "KeyboardEvent",
           "AudioContext"
           ]
```

注意

`AudioContext`绑定的文档可以在[`bit.ly/3tv5PsD`](https://bit.ly/3tv5PsD)找到。记住，你可以搜索`web-sys`文档中的任何 JavaScript 对象，以找到其对应的 Rust 库。

小贴士

根据你选择的编辑器，你可能需要在向项目添加新文件（如`sound.rs`）和/或将功能标志添加到`Cargo.toml`文件时重新启动`rust-analyzer`，以获得正确的编译错误和代码操作。

现在我们已经设置了`sound`模块，创建了创建`AudioContext`的函数，并且回顾了向`web-sys`依赖项添加新功能的过程，我们可以继续添加一些代码来播放声音。让我们介绍所有需要添加到`Cargo.toml`中的剩余功能标志：

```rs
[dependencies.web-sys]
version = "0.3.55"
features = ["console",
           "Window",
           "Document",
           "HtmlCanvasElement",
           "CanvasRenderingContext2d",
           "Element",
           "HtmlImageElement",
           "Response",
           "Performance",
           "KeyboardEvent",
           "AudioContext",
           "AudioBuffer",
           "AudioBufferSourceNode",
           "AudioDestinationNode",
           ]
```

这三个功能，`AudioBuffer`、`AudioBufferSourceNode`和`AudioDestinationNode`，对应于原始 JavaScript 代码中的相同对象。例如，`let trackSource = audioContext.createBufferSource();`函数返回`AudioBufferSourceNode`。`web-sys`的作者选择将大量音频功能隐藏在单个标志下，因此我们需要逐个命名它们。

小贴士

记得在无法使用`web-sys`功能时检查功能标志。它总是在文档中列出，并带有如下注释：“此 API 需要以下 crate 功能被激活：`AudioContext`。”

现在我们已经准备好了功能，我们可以添加剩余的代码，而不必担心那些错误。回到`sound`模块，代码将看起来像这样：

```rs
use anyhow::{anyhow, Result};
use web_sys::{AudioBuffer, AudioBufferSourceNode, AudioContext, AudioDestinationNode, AudioNode};
...
fn create_buffer_source(ctx: &AudioContext) -> Result<AudioBufferSourceNode> {
    ctx.create_buffer_source()
        .map_err(|err| anyhow!("Error creating buffer 
            source {:#?}", err))
}
fn connect_with_audio_node(
    buffer_source: &AudioBufferSourceNode,
    destination: &AudioDestinationNode,
) -> Result<AudioNode> {
    buffer_source
        .connect_with_audio_node(&destination)
        .map_err(|err| anyhow!("Error connecting audio 
            source to destination {:#?}", err))
}
```

在这本书中，我们通常一次通过一个函数来查看代码，但对于这两个函数来说，这不是必要的。这些函数分别对应于`audioContext.createBufferSource`和`trackSource.connect(audioContext.destination)`的调用。我们将代码从 JavaScript 的面向对象风格转换为稍微更过程化的格式，函数接受参数，部分原因是为了能够通过`anyhow!`宏将`JsValue`类型中的错误映射到适当的 Rust `Error`类型。

现在我们已经有了三个函数，我们需要播放一个声音。我们可以继续编写紧随其后的播放函数，如下所示：

```rs
pub fn play_sound(ctx: &AudioContext, buffer: &AudioBuffer) -> Result<()> {
    let track_source = create_buffer_source(ctx)?;
    track_source.set_buffer(Some(&buffer));
    connect_with_audio_node(&track_source, 
    &ctx.destination())?;
        track_source
        .start()
        .map_err(|err| anyhow! 
            ("Could not start sound!{:#?}", err))
}
```

`play_sound`函数接受`AudioContext`和`AudioBuffer`作为参数，然后返回`start`调用的结果，将`JsValue`映射到`Error`。我们还没有在任何地方创建`AudioBuffer`，所以不必担心你不知道如何创建，因为我们将在需要的时候解决这个难题。这里有一个与原始 JavaScript 播放声音非常相似的功能，但增加了 Rust 带来的错误处理，包括使用`?`操作符使其更容易阅读，以及在`track_source.set_buffer(Some(&buffer));`行中围绕`None`进行的一点点额外工作，因为我们需要在`Some`中包装`AudioBuffer`的引用，因为`track_source`有一个可选的缓冲区。在 JavaScript 中，这是`null`或`undefined`，但在 Rust 中，我们需要使用`Option`类型。否则，JavaScript 和 Rust 版本在播放声音时做的是同样的事情：

1.  从`AudioContext`创建`AudioBufferSource`。

1.  在源上设置`AudioBuffer`。

1.  将`AudioBufferSource`连接到`AudioContext`的目标。

1.  调用`start`来播放声音。

这看起来很多，但实际上非常快，所以缓存`AudioBufferSource`没有太大的用处，特别是因为你只能调用一次`start`。现在我们可以播放声音了，是时候加载声音资源并解码它，以便我们有`AudioBuffer`来播放。让我们现在就做吧。

## 加载声音

要从服务器加载声音，我们需要将以下代码翻译成 Rust，这些代码你已经见过：

```rs
let sound = await fetch("SFX_Jump_23.mp3");
let soundBuffer = await sound.arrayBuffer();
let decodedArray = await audioContext.decodeAudioData(soundBuffer);
```

获取资源是我们已经在我们的`browser`模块中可以做到的事情，但我们没有一种方便的方法来获取它的`arrayBuffer`，所以我们需要添加这个功能。我们还需要创建一个 Rust 版本的`decodeAudioData`。让我们从需要添加到`browser`中的更改开始，这些是现有方法的修改。我们希望将旧的`fetch_json`函数拆分，如下所示：

```rs
pub async fn fetch_json(json_path: &str) -> Result<JsValue> {
    let resp_value = fetch_with_str(json_path).await?;
    let resp: Response = resp_value
        .dyn_into()
        .map_err(|element| anyhow!("Error converting {:#?} 
            to Response", element))?;
    JsFuture::from(
        resp.json()
            .map_err(|err| anyhow!("Could not get JSON from 
                response {:#?}", err))?,
    )
    .await
    .map_err(|err| anyhow!("error fetching json {:#?}", err
        ))
}
```

我们需要将其拆分为两个函数，第一个函数首先获取 `Result<Response>`，第二个函数将其转换为 JSON：

```rs
pub async fn fetch_response(resource: &str) -> Result<Response> {
    fetch_with_str(resource)
        .await?
        .dyn_into()
        .map_err(|err| anyhow!("error converting fetch to 
            Response {:#?}", err))
}
pub async fn fetch_json(json_path: &str) -> Result<JsValue> {
    let resp = fetch_response(json_path).await?;
    JsFuture::from(
        resp.json()
            .map_err(|err| anyhow!("Could not get JSON from 
                response {:#?}", err))?,
    )
    .await
    .map_err(|err| anyhow!("error fetching JSON {:#?}", err
        ))
}
```

这是一个典型的“第二个人为抽象付费”的案例，我们在 *第二章* “绘制精灵”中编写的代码，用于加载 JSON，但现在我们需要一个可以处理多种响应的 `fetch` 版本，特别是那些可以作为 `ArrayBuffer` 访问的声音文件。这段代码将需要 `fetch_response`，但会将其转换为不同的对象。现在，让我们编写这段代码，就在 `fetch_json` 下方：

```rs
pub async fn fetch_array_buffer(resource: &str) -> Result<ArrayBuffer> {
    let array_buffer = fetch_response(resource)
        .await?
        .array_buffer()
        .map_err(|err| anyhow!("Error loading array buffer 
            {:#?}", err))?;
    JsFuture::from(array_buffer)
        .await
        .map_err(|err| anyhow!("Error converting array 
            buffer into a future {:#?}", err))?
        .dyn_into()
        .map_err(|err| anyhow!("Error converting raw 
            JSValue to ArrayBuffer {:#?}", err))
}
```

就像 `fetch_json` 一样，它首先通过传入的资源调用 `fetch_response`。然后，它调用响应上的 `array_buffer()` 函数，这将返回一个解析为 `ArrayBuffer` 的承诺。然后，我们像往常一样将承诺转换为 `JsFuture`，以便使用 `await` 语法。最后，我们调用 `dyn_into` 将所有 `Promise` 类型返回的 `JsValue` 转换为 `ArrayBuffer`。我跳过了它，但在每个步骤中，我们都使用 `map_err` 将 `JsValue` 错误转换为 `Error` 类型。

`ArrayBuffer` 类型是 JavaScript 的一种类型，目前我们的代码还无法使用。它是一个核心 JavaScript 类型，定义在 ECMAScript 标准 中，为了直接使用它，我们需要添加 `js-sys` 包。这有些令人惊讶，因为我们已经引入了 `wasm-bindgen` 和 `web-sys`，这两个包都依赖于 JavaScript，那么为什么我们还需要为 `ArrayBuffer` 引入另一个包呢？这与各种包的排列方式有关。`web-sys` 包包含了所有的 Web API，而 `js-sys` 只限于 ECMAScript 标准中的代码。到目前为止，我们除了 `web-sys` 暴露的内容外，还没有使用过核心 JavaScript 中的任何东西，但现在随着 `ArrayBuffer` 的出现，这种情况发生了变化。

为了使此代码能够编译，您需要在 `Cargo.toml` 中的依赖项列表中添加 `js-sys = "0.3.55"`。它已经在 `dev-dependencies` 中，所以您只需将其从那里移动即可。您还需要添加一个 `use` `js_sys::ArrayBuffer` 声明来导入 `ArrayBuffer` 结构体。

小贴士

在本书出版后，各种库可能会以小的方式发生变化。如果您在使用这些依赖项时遇到任何困难，请检查 [`github.com/rustwasm/wasm-bindgen`](https://github.com/rustwasm/wasm-bindgen) 的文档。

现在我们可以获取声音文件并将其作为 `ArrayBuffer` 获取，我们就可以编写我们自己的 `await audioContext.decodeAudioData(soundBuffer)` 版本了。到现在为止，你可能已经注意到我们正在遵循相同的模式来包装每个 JavaScript 函数：

1.  将任何返回承诺的函数，如 `decode_audio_data`，转换为 `JsFuture`，以便您可以在异步 Rust 代码中使用它。

1.  将 `JsValue` 中的任何错误映射到您自己的错误类型中；在这种情况下，我们使用 `anyhow::Result`，但你可能需要更具体的错误。

1.  使用 `?` 操作符来传播错误。

1.  检查特性标记，尤其是在使用 `web_sys` 并且你确信一个库存在时。

在此基础上，我们再添加一个步骤。

1.  使用 `dyn_into` 函数将 `JsValue` 类型转换为更具体的类型。

按照相同的模式，Rust 版本的 `decodeAudioData` 放在 `sound` 模块中，如下所示：

```rs
pub async fn decode_audio_data(
    ctx: &AudioContext,
    array_buffer: &ArrayBuffer,
) -> Result<AudioBuffer> {
    JsFuture::from(
        ctx.decode_audio_data(&array_buffer)
            .map_err(|err| anyhow!("Could not decode audio from array buffer {:#?}", err))?,
    )
    .await
    .map_err(|err| anyhow!("Could not convert promise to 
        future {:#?}", err))?
    .dyn_into()
    .map_err(|err| anyhow!("Could not cast into AudioBuffer 
        {:#?}", err))
}
```

你需要确保添加对 `js_sys::ArrayBuffer` 和 `wasm_bindgen_futures::JsFuture` 以及 `wasm_bindgen::JsCast` 的 `use` 声明，以便将 `dyn_into` 函数引入作用域。再次提醒，我们不是直接在 `AudioContext` 上调用方法，在这个例子中是 `decodeAudioData`，而是创建了一个封装调用的函数。该函数将 `AudioContext` 的引用作为第一个参数，并将 `ArrayBuffer` 类型作为第二个参数。这使得我们可以将错误映射和结果转换封装到函数中。

这个函数随后将 `ctx.decode_audio_data` 作为参数传递，并传递 `ArrayBuffer`，但如果它只是这样做，我们实际上并不需要它。然后它将 `ctx.decode_audio_data` 中的任何错误映射到 `Error` 上，使用 `anyhow!`；实际上，正如你所看到的，它将在处理过程中的每个步骤都这样做，并与 `?` 操作符配对以传播错误。它从 `decode_audio_data` 中获取一个承诺，并从中创建 `JsFuture`，然后立即调用 `await` 以等待完成，对应于 JavaScript 中的 `await` 调用。在处理将承诺转换为 `JsFuture` 的任何错误后，我们使用 `dyn_into` 函数将其转换为 `AudioBuffer`，最终处理任何相关的错误。

这个函数是所有封装函数中最复杂的，所以让我们回顾一下从一行 JavaScript 转换为九行 Rust 时所采取的步骤：

1.  将任何返回承诺的函数转换为 `JsFuture`，以便你可以在异步 Rust 代码中使用它。

在这种情况下，`decode_audio_data` 返回了一个承诺，我们使用 `JsFuture::from` 将其转换为 `JsFuture`，然后立即调用 `await`。

1.  将 `JsValue` 中的任何错误映射到你的自定义错误类型；在这种情况下，我们使用 `anyhow::Result`，但你可能需要更具体的错误。

我们这样做三次，因为每个调用似乎都返回一个 `JsValue` 版本的结果，我们在错误信息中添加了说明性语言。

1.  使用 `dyn_into` 函数将 `JsValue` 类型转换为更具体的类型。

我们这样做是为了将 `decode_audio_data` 的最终结果从 `JsValue` 转换为 `AudioBuffer`，并且 Rust 的编译器可以从函数的返回值推断出适当的数据类型。

1.  不要忘记使用 `?` 操作符来传播错误；注意这个函数是如何两次做到这一点的。

我们使用了 `?` 操作符两次，使函数更容易阅读。

1.  检查特性标记，尤其是在使用 `web_sys` 并且你确信一个库存在时。

`AudioBuffer` 是特性标记的，但我们又在开始时将其添加了回来。

这个过程比实际操作要复杂一些。大部分情况下，你可以遵循编译器并使用像`rust-analyzer`这样的工具来自动添加`use`声明。

现在我们已经拥有了所有这些实用工具，我们需要播放一个声音。是时候将这个功能添加到`engine`模块中，以便我们的游戏可以使用它。

## 将音频添加到引擎中

我们在`sound`模块中刚刚创建的函数可以直接通过委托函数由引擎使用，但我们不想让游戏担心`AudioContext`、`AudioBuffer`和类似的东西。就像`Renderer`一样，我们将创建一个`Audio`结构体来封装该实现的细节。我们还将创建一个`Sound`结构体，将`AudioBuffer`转换为对整个系统更友好的类型。这些结构体将会非常小，如下所示：

```rs
#[derive(Clone)]
pub struct Audio {
    context: AudioContext,
}
#[derive(Clone)]
pub struct Sound {
    buffer: AudioBuffer,
}
```

这些结构体被添加到`engine`模块的底部，但它们实际上可以放在文件的任何地方。别忘了导入`AudioContext`和`AudioBuffer`！如果你发现自己随着`engine`和`game`的增大而感到困惑，你可以通过创建一个`mod.rs`文件和目录将其拆分成多个文件，但为了跟上进度，所有内容都需要最终放在`engine`模块中。我不会这样做，因为虽然这样做可以使代码更容易导航，但它会使解释和跟进变得更加困难。稍后将其拆分成更小的块是一个很好的练习，以确保你理解我们正在编写的代码。

现在我们有一个表示`Audio`并持有`AudioContext`的结构体，以及一个相应的`Sound`结构体，它持有`AudioBuffer`，我们可以向`Audio`添加`impl`，使用我们之前编写的函数来播放声音。现在，我们将向`Audio`结构体添加`impl`来播放和加载声音。让我们从加载实现开始，这可能是最难的，如下所示：

```rs
impl Audio {
    pub fn new() -> Result<Self> {
        Ok(Audio {
            context: sound::create_audio_context()?,
        })
    }
    pub async fn load_sound(&self, filename: &str) ->         Result<Sound> {
        let array_buffer = 
            browser::fetch_array_buffer(filename).await?;
        let audio_buffer = 
            sound::decode_audio_data(&self.context, 
                &array_buffer).await?;
        Ok(Sound {
            buffer: audio_buffer,
        })
    }
}
```

这个`impl`将从两个方法开始，一个是熟悉的`new`方法，它使用`AudioContext`创建一个`Audio`结构体。请注意，在这种情况下`new`返回一个结果，因为`create_audio_context`可能会失败。然后，我们有`load_sound`方法，它也返回一个结果，这次是`Sound`类型，只有三行。这是我们正确组织`sound`和`browser`模块中的函数的一个迹象，因为我们只需简单地调用我们的`fetch_array_buffer`和`decode_audio_data`函数来获取`AudioBuffer`，然后将其包装在一个`Sound`结构体中。我们返回一个结果并通过`?`传播错误。如果加载声音很简单，那么在这个`Audio`实现的方法中播放它就很容易：

```rs
impl Audio {
    ...
    pub fn play_sound(&self, sound: &Sound) -> Result<()> {
        sound::play_sound(&self.context, &sound.buffer)
    }
}
```

对于`play_sound`，我们实际上只是委托，传递`Audio`持有的`AudioContext`和从传入的声音中获取的`AudioBuffer`。

我们已经编写了一个模块来在 API 中播放声音，添加了加载声音到浏览器，并最终创建了游戏引擎的音频部分。这足以在引擎中播放音效；现在我们需要将其添加到我们的游戏中，这将会变得复杂。

# 播放音效

将音效添加到我们的游戏是一个挑战，原因有几个：

+   效果必须只发生一次：

我们将为跳跃（*boing!*）添加音效，并确保它只发生一次。幸运的是，我们已经有了一个解决方案，那就是我们的状态机！我们可以使用`RedHatBoyContext`在发生某些事情时播放声音，如下所示（现在不要添加它）：

```rs
impl RedHatBoyContext {
    ...
    fn play_jump_sound(audio: &Audio) {
        audio.play_sound(self.sound)
    }
}
```

这直接引出了我们的第二个挑战。

+   在过渡时播放音频：

我们希望在过渡时刻播放声音，但大多数过渡不会播放声音。记住我们的状态机使用`transition`从一个事件过渡到另一个事件，虽然我们可以在那里传递音频，但它只会被该方法中一小部分代码使用。这是一个代码问题，所以我们不会这样做。`RedHatBoyContext`将必须拥有音频和声音。这不是理想的，我们更希望系统中只有一个音频，但这对我们的状态机来说不可行。这导致了我们的第三个问题。

+   `AudioContext`和`AudioBuffer`不是`Copy`：

为了在`RedHatBoy`实现中使用`self.state = self.state.jump();`这样的语法，并让每个状态过渡消耗`RedHatBoyContext`，我们需要`RedHatBoyContext`是`Copy`。不幸的是，`AudioContext`和`AudioBuffer`不是`Copy`，这意味着`Audio`和`Sound`不能是`Copy`，因此，如果`RedHatBoyContext`要持有音频和声音，它也不能是一个副本。这很糟糕，但我们可以通过重构`RedHatBoyContext`和`RedHatBoy`以使用所需的`clone`函数来修复它。

让`RedHatBoyContext`拥有音频意味着系统中可能存在多个`Audio`对象，其中另一个将播放音乐。这是多余的，但大多数情况下是无害的，所以我们选择这个解决方案。它使我们能够继续开发，最终，这个解决方案效果很好。如果有疑问，选择现成的解决方案。

注意

你可能会想知道为什么我们不在`RedHatBoyContext`中存储`Audio`的引用。最终，在我们的引擎中`Game`是静态的，因此，如果将`Audio`引用存储在`RedHatBoyContext`上，它必须保证与`Game`一样长命。 

有其他选项，包括使用服务定位器模式([`bit.ly/3A4th2f`](https://bit.ly/3A4th2f))或将音频作为参数传递到`update`函数中，但它们都需要更长的时间才能达到我们的最终目标——播放声音，这是本章的真实目标。

在我们能够将音效添加到游戏中之前，我们需要重构代码以包含一个`Audio`元素。然后我们将播放音效。

## 重构 RedHatBoyContext 和 RedHatBoy

在我们真正这样做之前，我们将准备`RedHatBoyContext`和`RedHatBoy`来存储音频和一首歌，因为这会使添加声音更容易。让我们首先将`RedHatBoyContext`设置为`clone`，如下所示：

```rs
#[derive(Clone)]
struct RedHatBoyContext {
    frame: u8,
    position: Point,
    velocity: Point,
}
```

我们所做的一切就是从`derive`声明中移除了`Copy`特质。这将在`RedHatBoyStateMachine`和`RedHatBoyState<S>`上引起编译错误，这两个结构都派生了`Copy`，因此你还需要从这些结构中移除那个声明。一旦你这样做，你将看到一大堆类似这样的错误：

```rs
nerror[E0507]: cannot move out of `self.state` which is behind a mutable reference
   --> src/game.rs:134:22
    |
134 |         self.state_machine = self.state_machine.run();
    |                      ^^^^^^^^^^ move occurs because `self.state` has type `RedHatBoyStateMachine`, which does not implement the `Copy` trait
```

如预期的那样，调用`self.state.<method>`，其中方法接受`self`，所有调用都未能编译，因为`RedHatBoyStateMachine`不再实现`Copy`。解决方案，我们将在每一行遇到这个编译错误时这样做，就是在我们需要进行更改时显式地克隆状态。以下是带有错误的`run_right`函数：

```rs
impl RedHatBoy {
    ...
    fn run_right(&mut self) {
        self.state_machine = self.state_machine.            transition(Event::Run);
    }
```

然后，这是修复后的结果：

```rs
impl RedHatBoy {
    ...
    fn run_right(&mut self) {
        self.state_machine = self.state_machine
            clone().transition(Event::Run);
    }
```

可能最令人牙痒痒的例子是在`transition`方法中，我们将因为`match`语句而得到一个移动，如下所示：

```rs
impl RedHatBoyStateMachine {
    fn transition(self, event: Event) -> Self {
        match (self, event) {
            ...
            _ => self,
        }
    }
```

这个部分的问题在于`self`被移动到了`match`语句中，并且在默认情况下无法返回。试图使用`match`和`self`来解决这个问题会导致所有的类型状态方法，例如`land_on`和`knock_out`，失败，因为它们需要消耗`self`。*最干净*的修复方法如下所示：

```rs
impl RedHatBoyStateMachine {
    fn transition(self, event: Event) -> Self {
        match (self.clone(), event) {
             ...
             _ => self,
        }
    }
```

我承认这很糟糕，但我们能够继续进步。

小贴士

我知道你在想什么——性能！我们在每个转换时都在克隆！你完全正确，但你知道性能会受到负面影响吗？性能的第一条规则是*先测量*，在我们测量这个之前，我们实际上不知道这个代码的最终版本是否是问题。我花了很多时间试图避免这个`clone`调用，因为担心性能问题，结果发现这并没有多大影响。先让它工作，然后再让它变得更快。

一旦你修复那个错误几次，你就可以为`RedHatBoyContext`添加音频和声音了，但我们会播放什么声音呢？

## 添加音效

使用 Web Audio API，我们可以播放任何由`audio`HTML 元素支持的音频格式，包括所有常见的 WAV、MP3、MP4 和 Ogg 格式。此外，2017 年，MP3 许可证到期，所以如果你对此担心，不要担心；你可以无忧无虑地使用 MP3 文件作为声音。

由于 Web Audio API 与许多音频格式兼容，因此只要它是在适当的许可下发布的，你可以使用来自互联网任何地方的声音。我们将用于跳跃的声音效果可在 [`opengameart.org/content/8-bit-jump-1`](https://opengameart.org/content/8-bit-jump-1) 找到，并且它是在 *Creative Commons 公共领域* 许可下发布的，因此我们可以放心使用。你不需要下载那个捆绑包并浏览它，尽管你可以这样做，但跳跃声音已经捆绑在这本书的资产中，位于 [`github.com/PacktPublishing/Game-Development-with-Rust-and-WebAssembly/wiki/Assets`](https://github.com/PacktPublishing/Game-Development-with-Rust-and-WebAssembly/wiki/Assets) 的 `sounds` 目录下。我们想要的特定文件是 `SFX_Jump_23.mp3`。你需要将这个文件复制到你的 Rust 项目的 `static` 目录中，以便它可以在你的游戏中使用。

现在，`RedHatBoyContext` 已经准备好容纳 `Audio` 结构体，并且 `SFX_Jump_23.mp3` 文件可供加载，我们可以开始添加代码。从添加 `Audio` 和 `Sound` 到 `RedHatBoyContext`，如下所示：

```rs
#[derive(Clone)]
pub struct RedHatBoyContext {
    pub frame: u8,
    pub position: Point,
    pub velocity: Point,
    audio: Audio,
    jump_sound: Sound,
}
```

记得将 `Audio` 和 `Sound` 的 `use` 声明添加到 `red_hat_boy_states` 模块中。代码将无法编译，因为 `RedHatBoyContext` 在没有 `audio` 或 `jump_sound` 的情况下被初始化，所以我们需要添加它。`RedHatBoyContext` 在 `RedHatBoyState<Idle>` 实现的 `new` 方法中被初始化，所以我们将该方法更改为接受 `Audio` 和 `Sound` 对象，并将它们传递给 `RedHatBoyContext`，如下所示：

```rs
impl RedHatBoyState<Idle> {
    fn new(audio: Audio, jump_sound: Sound) -> Self {
        RedHatBoyState {
            game_object: RedHatBoyContext {
                frame: 0,
                position: Point {
                    x: STARTING_POINT,
                    y: FLOOR,
                },
                velocity: Point { x: 0, y: 0 },
                audio,
                jump_sound,
            },
            _state: Idle {},
        }
    }
}
```

我们可以在这里创建一个 `Audio` 对象，但这样 `new` 方法就需要返回 `Result<Self>`，我认为这不合适。这将移动编译器错误，因为我们调用 `RedHatBoyState<Idle>::new` 的地方现在是不正确的。那就是在 `RedHatBoy::new` 中，它现在也可以接受 `Audio` 和 `Sound` 对象并将它们传递下去。

这将带我们来到 `Game` 实现中臭名昭著的 `initialize` 函数，因为它在没有任何 `Audio` 或 `Sound` 的情况下调用 `RedHatBoy::new`，因此无法编译。这是一个加载文件的合适位置，因为它既是 `async` 的，又返回一个结果。我们将在 `initialize` 中创建一个 `Audio` 对象，加载我们想要的音效，并将其传递给 `RedHatBoy::new` 函数，如下所示：

```rs
#[async_trait(?Send)]
impl Game for WalkTheDog {
    async fn initialize(&mut self) -> Result<Box<dyn Game>> {
        match self {
            WalkTheDog::Loading => {
                ...
                let audio = Audio::new()?;
                let sound = audio.load_sound
                    ("SFX_Jump_23.mp3").await?;
                let rhb = RedHatBoy::new(
                    sheet,
                    engine::load_image("rhb.png").await?,
                    audio,
                    sound,
                );
                ...
            }
```

这将使应用程序再次编译，但我们没有对 `audio` 或 `sound` 做任何事情。记住，所有这些工作都是因为我们想确保在跳跃时声音只播放 *一次*，而确保这一点的方法是将声音播放放在从 `Running` 到 `Jumping` 的转换中。转换是通过 `RedHatBoyContext` 上的各种 `From` 实现通过方法完成的。让我们在 `RedHatBoyContext` 上编写一个名为 `play_jump_sound` 的小函数，如下所示：

```rs
impl RedHatBoyContext {
    ...
    fn play_jump_sound(self) -> Self {
        if let Err(err) = self.audio.play_sound
           (&self.jump_sound) {
            log!("Error playing jump sound {:#?}", err);
        }
        self
    }
}
```

这个函数的编写方式与这个实现中其他过渡副作用函数略有不同，因为`play_sound`返回一个结果，但为了与其他过渡方法保持一致，`play_jump_sound`实际上不应该这样做。幸运的是，未能播放声音虽然令人烦恼，但不会致命，所以如果声音无法播放，我们将记录错误并继续。现在代码可以编译，但我们需要将`play_jump_sound`的调用添加到过渡中。在`RedHatBoyState<Running>`上查找`jump`，并将该过渡修改为调用`play_jump_sound`，如下所示：

```rs
    impl RedHatBoyState<Running> {
        ...
        pub fn jump(self) -> RedHatBoyState<Jumping> {
            RedHatBoyState {
                context: self
                    .context
                    .reset_frame()
                    .set_vertical_velocity(JUMP_SPEED)
                    .play_jump_sound(),
                _state: Jumping {},
            }
        }
```

当这个程序编译完成后，运行游戏，你就会看到，也会听到 RHB 跳上平台。

![图 7.1 – 你能听到吗？](img/Figure_7.01_B17151.jpg)

图 7.1 – 你能听到吗？

小贴士

如果你像我认识的许多开发者一样，现在有 20 多个浏览器标签页打开，你可能想关闭它们。这可能会减慢浏览器的声音播放，并使声音时间不准确。

现在你已经播放了一个音效，考虑添加更多，例如，当 RHB 撞到障碍物，或者平稳着陆，或者滑动时。选择权在你！在你对音效玩得有点乐趣之后，让我们添加一些背景音乐。

# 播放长音乐

你可能会认为播放音乐意味着检测声音是否播放完成并重新开始。这可能对于浏览器的实现来说是正确的，但幸运的是，你不必这样做。Web Audio API 已经在`AudioBufferSourceNode`的循环上设置了一个标志，可以在声音被明确停止之前循环播放声音。这将使播放背景音频变得相当简单。我们可以在`sound`模块的`play_sound`函数中添加一个标志到`loop`参数，如下所示：

```rs
fn create_track_source(ctx: &AudioContext, buffer: &AudioBuffer) -> Result<AudioBufferSourceNode> {
    let track_source = create_buffer_source(ctx)?;
    track_source.set_buffer(Some(&buffer));
    connect_with_audio_node(&track_source, 
        &ctx.destination())?;
    Ok(track_source)
}
pub enum LOOPING {
    NO,
    YES,
}
pub fn play_sound(ctx: &AudioContext, buffer: &AudioBuffer, looping: LOOPING) -> Result<()> {
    let track_source = create_track_source(ctx, buffer)?;
    if matches!(looping, LOOPING::YES) {
        track_source.set_loop(true);
    }
    track_source
        .start()
        .map_err(|err| anyhow!("Could not start sound! 
            {:#?}", err))
}
```

这是从`create_track_source`函数开始的，实际上是对`play_sound`函数的重构。它提取了它的前三行，并将它们提取到一个单独的函数中以提高可读性。之后，我们创建了一个`LOOPING`枚举，并使用它来检查我们是否应该在`track_source`上调用`set_loop`。你可能想知道为什么我们不直接传递`bool`作为第三个参数，答案是，阅读这里显示的第一行代码将比第二行代码容易得多：

```rs
play_sound(ctx, buffer, LOOPING::YES)
play_sound(ctx, buffer, true)
```

六个月后，当我不知道那个布尔值是做什么的时候，我不得不去查找，而使用枚举的版本则很明显。通过添加这个标志，我们的程序停止编译，因为引擎中的`Audio`仍然使用两个参数调用`play_sound`。我们可以快速修复这个问题，如下所示：

```rs
impl Audio {
    ...
    pub fn play_sound(&self, sound: &Sound) -> Result<()> {
        sound::play_sound(&self.context, &sound.buffer, 
            sound::LOOPING::NO)
    }
```

我们还将添加一种播放背景音乐的新方法，即通过开启循环播放来播放声音：

```rs
impl Audio {
    ...
    pub fn play_looping_sound(&self, sound: &Sound) -> 
        Result<()> {
        sound::play_sound(&self.context, &sound.buffer, 
            sound::LOOPING::YES)
    }
}
```

我喜欢引擎比`sound`模块的灵活性逐渐减少。`sound`和`browser`模块是浏览器功能的包装器；引擎提供了帮助您制作游戏的工具。现在，引擎提供了一种播放背景音乐的方式，我们实际上可以将它添加到游戏中。在资源中，`sounds`目录中有一个第二个文件，`background_song.mp3`，您可以将其复制到本项目的`static`目录中。一旦完成，我们就可以在`Game::initialize`函数中加载并播放背景音乐：

```rs
#[async_trait(?Send)]
impl Game for WalkTheDog {
    async fn initialize(&mut self) -> Result<Box<dyn Game>> {
        match self {
            WalkTheDog::Loading => {
                ...
                let audio = Audio::new()?;
                let sound = audio.load_sound
                    ("SFX_Jump_23.mp3").await?;
                let background_music = audio.load_sound
                    ("background_song.mp3").await?;
                    audio.play_looping_sound
                        (&background_music)?;
                let rhb = RedHatBoy::new(
                    sheet,
                    engine::load_image("rhb.png").await?,
                    audio,
                    sound,
                );
                ...
```

小贴士

查看 https://gamesounds.xyz/获取您游戏的无版权声音。

在这里，我们加载第二首歌曲`background_song.mp3`，并使用`play_looping_sound`立即播放。在大多数浏览器上，您可能需要点击画布以将其聚焦，才会听到音乐，所以如果听不到任何声音，请检查一下。需要注意的是，尽管那个声音即将超出作用域，浏览器仍然会愉快地继续播放它。我们已经将歌曲传递给浏览器，现在由它负责。当音频移动到`RedHatBoy`中时，`RedHatBoy`的创建没有发生变化，它最终将负责播放游戏的声音效果。

小贴士

在开发过程中，你可能想静音浏览器，因为每次浏览器刷新时，歌曲都会重新开始播放。

就这样！一个带有音乐和声音效果的完整游戏！现在要添加 UI，这样我们就可以点击它上的**新游戏**了。

# 摘要

在本章中，您使用 Web Audio API 为您的游戏添加了声音，并对 API 本身进行了概述。Web Audio API 非常广泛，具有众多功能，我鼓励您去探索它。您的第一个挑战是使用`gain`属性来改变音乐的音量，目前音量相当大。Web Audio API 还支持立体声环绕声和程序生成音乐等功能。尽情享受并尝试一下吧！

您还为游戏添加了一个新模块，并进一步扩展了游戏引擎以支持它。我们甚至涵盖了重构，并做出了一些权衡，以确保游戏可以在不要求耗时*理想*设计的情况下完成。我鼓励您花些时间给游戏添加更多声音效果；您现在有技能让 RHB 在着陆或撞到岩石时发出*砰*的声音。说到撞到岩石，你可能已经厌倦了每次都不得不点击*刷新*，所以在下一章中，我们将添加一个小型 UI，并带有一个精彩的**新游戏**按钮。
