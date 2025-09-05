# 设计模式

函数式编程已经发展出了类似于面向对象或其他社区的设计模式。这些模式，不出所料，利用函数作为核心概念。它们还强调了一个称为**单一职责原则**的概念。单一职责原则指出，程序的逻辑组件应该只做一件事，并且要做好这件事。在本章中，我们将关注几个非常常见的模式。其中一些概念非常简单，以至于它们反直觉地变得难以解释。在这些情况下，我们将使用各种示例来展示一个简单的概念如何表现出复杂的行为。

在本章中，你将执行以下操作：

+   学习识别和使用函子

+   学习识别和使用单子

+   学习识别和使用组合子

+   学习识别和使用惰性求值

# 技术要求

运行提供的示例需要一个较新的 Rust 版本：

[`www.rust-lang.org/en-US/install.html`](https://www.rust-lang.org/en-US/install.html)

本章的代码也可在 GitHub 上找到：

[`github.com/PacktPublishing/Hands-On-Functional-Programming-in-RUST`](https://github.com/PacktPublishing/Hands-On-Functional-Programming-in-RUST)

每个章节的 `README.md` 文件中也包含了具体的安装和构建说明。

# 使用函子模式

函子大致是函数的逆：

+   函数定义了一个转换，接受数据，并返回转换的结果

+   函子定义数据，接受一个函数，并返回转换的结果

一个简单的函子例子是 Rust 向量和其伴随的 `map` 函数：

```rs
fn main() {
   let m: Vec<u64> = vec![1, 2, 3];
   let n: Vec<u64> = m.iter().map(|x| { x*x }).collect();
   println!("{:?}", m);
   println!("{:?}", n);
}
```

由于构成函子或非函子的规则，函子通常被认为只是 `map` 函数。前面提到的常见情况被称为**结构保持映射**。函子不需要是结构保持的。例如，考虑以下代码中为集合实现的类似映射情况：

```rs
use std::collections::{HashSet};

fn main() {
   let mut a: HashSet<u64> = HashSet::new();
   a.insert(1);
   a.insert(2);
   a.insert(3);
   a.insert(4);
   let b: HashSet<u64> = a.iter().cloned().map(|x| x/2).collect();
   println!("{:?}", a);
   println!("{:?}", b);
}
```

我们在这里看到，由于冲突，结果集比原始集小。这个映射仍然满足函子的性质。函子的定义性质如下：

+   一组对象，`C`

+   一个将 `C` 中的对象映射到 `D` 中的对象的映射函数

前面的 `Set` 映射满足第一和第二个性质，因此是一个合适的函子。它还展示了如何通过函子将数据转换成不同形状的结构。发挥一点想象力，我们还可以考虑每个映射值可能产生多个输出的情况：

```rs
fn main() {
   let sentences = vec!["this is a sentence","paragraphs have many sentences"];
   let words:Vec<&str> = sentences.iter().flat_map(|&x| x.split(" ")).collect();
   println!("{:?}", sentences);
   println!("{:?}", words);
}
```

从技术角度讲，这个最后的例子不是一个正常的函子，而是一个逆变函子。所有函子都是协变的。协变与逆变之间的区别对我们来说并不重要，所以我们把这个问题留给最好奇的读者。

作为通过例子给出的最终定义，我们应该注意函子的输入和输出不需要是同一类型。例如，我们可以从向量映射到`HashSet`：

```rs
use std::collections::{HashSet};

fn main() {
   let v: Vec<u64> = vec![1, 2, 3];
   let s: HashSet<u64> = v.iter().cloned().map(|x| x/2).collect();
   println!("{:?}", v);
   println!("{:?}", s);
}
```

为了给出一个非平凡的例子来说明函子模式如何使用，让我们看看网络摄像头和 AI。现代 AI 面部识别软件能够在图片中识别人类面孔，甚至可见的情绪状态。让我们想象一个连接到网络摄像头并使用过滤器处理输入的应用程序。以下是程序的一些类型定义：

```rs
struct WebCamera;

#[derive(Debug)]
enum VisibleEmotion {
   Anger,
   Contempt,
   Disgust,
   Fear,
   Happiness,
   Neutral,
   Sadness,
   Surprise
}

#[derive(Debug,Clone)]
struct BoundingBox {
   top: u64,
   left: u64,
   height: u64,
   width: u64
}

#[derive(Debug)]
enum CameraFilters {
   Sparkles,
   Rain,
   Fire,
   Disco
}

```

在`WebCamera`类型上，我们将实现两个函子。一个函子，`map_emotion`，将情绪映射到其他情绪。这可能被用来向文本聊天添加表情符号。第二个协变函子，`flatmap_emotion`，将情绪映射到零个或多个过滤器。这些是可以应用到网络摄像头视野中的动画或效果：

```rs
impl WebCamera {
   fn map_emotion<T,F>(&self, translate: F) -> Vec<(BoundingBox,T)>
   where F: Fn(VisibleEmotion) -> T {
      //Simulate emotion extracted from WebCamera
      vec![
         (BoundingBox { top: 1, left: 1, height: 1, width: 1 }, VisibleEmotion::Anger),
         (BoundingBox { top: 1, left: 1, height: 1, width: 1 }, VisibleEmotion::Sadness),
         (BoundingBox { top: 4, left: 4, height: 1, width: 1 }, VisibleEmotion::Surprise),
         (BoundingBox { top: 8, left: 1, height: 1, width: 1 }, VisibleEmotion::Neutral)
      ].into_iter().map(|(bb,emt)| {
         (bb, translate(emt))
      }).collect::<Vec<(BoundingBox,T)>>()
   }
   fn flatmap_emotion<T,F,U:IntoIterator<Item=T>>(&self, mut translate: F) -> Vec<(BoundingBox,T)>
   where F: FnMut(VisibleEmotion) -> U {
      //Simulate emotion extracted from WebCamera
      vec![
         (BoundingBox { top: 1, left: 1, height: 1, width: 1 }, VisibleEmotion::Anger),
         (BoundingBox { top: 1, left: 1, height: 1, width: 1 }, VisibleEmotion::Sadness),
         (BoundingBox { top: 4, left: 4, height: 1, width: 1 }, VisibleEmotion::Surprise),
         (BoundingBox { top: 8, left: 1, height: 1, width: 1 }, VisibleEmotion::Neutral)
      ].into_iter().flat_map(|(bb,emt)| {
         translate(emt).into_iter().map(move |t| (bb.clone(), t))
      }).collect::<Vec<(BoundingBox,T)>>()
   }
}

```

要使用函子，程序员需要提供哪些情绪映射到哪些过滤器。由于函子模式提供的封装，复杂的 AI 和效果可以很容易地修改：

```rs
fn main() {
   let camera = WebCamera;
   let emotes: Vec<(BoundingBox,VisibleEmotion)> = camera.map_emotion(|emt| {
      match emt {
         VisibleEmotion::Anger |
         VisibleEmotion::Contempt |
         VisibleEmotion::Disgust |
         VisibleEmotion::Fear |
         VisibleEmotion::Sadness => VisibleEmotion::Happiness,
         VisibleEmotion::Neutral |
         VisibleEmotion::Happiness |
         VisibleEmotion::Surprise => VisibleEmotion::Sadness
      }
   });

   let filters: Vec<(BoundingBox,CameraFilters)> = camera.flatmap_emotion(|emt| {
      match emt {
         VisibleEmotion::Anger |
         VisibleEmotion::Contempt |
         VisibleEmotion::Disgust |
         VisibleEmotion::Fear |
         VisibleEmotion::Sadness => vec![CameraFilters::Sparkles, CameraFilters::Rain],
         VisibleEmotion::Neutral |
         VisibleEmotion::Happiness |
         VisibleEmotion::Surprise => vec![CameraFilters::Disco]
      }
   });

   println!("{:?}",emotes);
   println!("{:?}",filters);
}
```

# 使用单子模式

单子为一种类型定义了`return`和`bind`操作。`return`操作就像一个构造函数来创建单子。`bind`操作结合新的信息并返回一个新的单子。单子还应遵守一些定律。我们不会引用这些定律，只是说单子应该像以下这样在链式操作中表现良好：

```rs
MyMonad::return(value)  //We start with a new MyMonad<A>
        .bind(|x| x+x)  //We take a step into MyMonad<B>
        .bind(|y| y*y); //Similarly we get to MyMonad<C>
```

在 Rust 中，标准库中有几个半单子：

```rs
fn main()
{
   let v1 = Some(2).and_then(|x| Some(x+x)).and_then(|y| Some(y*y));
   println!("{:?}", v1);

   let v2 = None.or_else(|| None).or_else(|| Some(222));
   println!("{:?}", v2);
}
```

在这个例子中，正常的`Option`构造函数，`Some`或`None`，取代了单子的命名约定，即`return`。这里实现了两个半单子，一个与`and_then`相关联，另一个与`or_else`相关联。这两个都对应于单子的`bind`命名约定，用于将新信息结合到新的单子返回值中。

单子的`bind`操作也是多态的，这意味着它们应该允许从当前单子返回不同类型的单子。根据这个规则，`or_else`在技术上不是一个单子；因此它是一个半单子：

```rs
fn main() {
   let v3 = Some(2).and_then(|x| Some("abc"));
   println!("{:?}", v3);

   // or_else is not quite a monad
   // does not permit polymorphic bind
   //let v4 = Some(2).or_else(|| Some("abc"));
   //println!("{:?}", v4);
}
```

单子最初是为了在纯函数式语言中表达副作用而开发的。这不是一个矛盾——纯函数式语言中的副作用？

如果效果作为输入和输出通过纯函数传递，答案是*否*。然而，为了使这可行，每个函数都需要声明每个状态变量并将其传递，这可能会变成一个非常长的参数列表。这就是单子的作用。单子可以隐藏自身内部的状态，这本质上比程序员交互的函数更大、更复杂。

一个具体的副作用隐藏的例子是通用日志器的概念。单子的`return`和`bind`可以用来封装状态和计算，在单子中记录所有中间结果。以下是日志单子的示例：

```rs
use std::fmt::{Debug};

struct LogMonad<T>(T);
impl<T> LogMonad<T> {
   fn _return(t: T) -> LogMonad<T>
   where T: Debug {
      println!("{:?}", t);
      LogMonad(t)
   }
   fn bind<R,F>(&self, f: F) -> LogMonad<R>
   where F: FnOnce(&T) -> R,
   R: Debug {
      let r = f(&self.0);
      println!("{:?}", r);
      LogMonad(r)
   }
}

fn main() {
   LogMonad::_return(4)
            .bind(|x| x+x)
            .bind(|y| y*y)
            .bind(|z| format!("{}{}{}", z, z, z));
}
```

只要每个结果实现了`Debug`特质，就可以使用这种模式自动记录。

单子模式对于将无法用正常代码块编写的代码连接起来也非常有用。例如，代码块总是被急切地评估。如果你想定义稍后或在片段中评估的代码，懒单子模式非常方便。懒评估是一个术语，用来描述只有在被引用时才进行评估的代码或数据。这与 Rust 代码的典型急切评估相反，Rust 代码将立即执行，无论上下文如何。以下是一个懒单子模式的例子：

```rs
struct LazyMonad<A,B>(Box<Fn(A) -> B>);

impl<A: 'static,B: 'static> LazyMonad<A,B> {
   fn _return(u: A) -> LazyMonad<B,B> {
      LazyMonad(Box::new(move |b: B| b))
   }
   fn bind<C,G: 'static>(self, g: G) -> LazyMonad<A,C>
   where G: Fn(B) -> C {
      LazyMonad(Box::new(move |a: A| g(self.0(a))))
   }
   fn apply(self, a: A) -> B {
      self.0(a)
   }
}

fn main() {
   let notyet = LazyMonad::_return(())   //we create LazyMonad<()>
                          .bind(|x| x+2) //and now a LazyMonad<A>
                          .bind(|y| y*3) //and now a LazyMonad<B>
                          .bind(|z| format!("{}{}", z, z));

   let nowdoit = notyet.apply(222); //The above code now run
   println!("nowdoit {}", nowdoit);
}
```

这个块定义了在提供值之后、但在之前不会逐个评估的语句。这可能看起来有点微不足道，因为我们可以用简单的闭包和代码块做到同样的事情；然而，为了使这个模式更加牢固，让我们考虑一个更复杂的案例——异步 Web 服务器。

Web 服务器通常在处理之前会接收到一个完整的 HTTP 请求。决定如何处理请求有时被称为**路由**。然后请求被发送到请求处理器。在以下代码中，我们定义了一个服务器，它帮助我们将路由和处理器包装成一个单一的 Web 服务器对象。以下是类型和方法定义：

```rs
use std::io::prelude::*;
use std::net::TcpListener;
use std::net::TcpStream;

struct ServerMonad<St> {
   state: St,
   handlers: Vec<Box<Fn(&mut St,&String) -> Option<String>>>
}

impl<St: Clone> ServerMonad<St> {
   fn _return(st: St) -> ServerMonad<St> {
      ServerMonad {
         state: st,
         handlers: Vec::new()
      }
   }
   fn listen(&mut self, address: &str) {
      let listener = TcpListener::bind(address).unwrap();
      for stream in listener.incoming() {
         let mut st = self.state.clone();
         let mut buffer = [0; 2048];
         let mut tcp = stream.unwrap();
         tcp.read(&mut buffer);
         let buffer = String::from_utf8_lossy(&buffer).into_owned();
         for h in self.handlers.iter() {
            if let Some(response) = h(&mut st,&buffer) {
               tcp.write(response.as_bytes());
               break
            }
         }
      }
   }
   fn bind_handler<F>(mut self, f: F) -> Self
      where F: 'static + Fn(&mut St,&String) -> Option<String> {
      self.handlers.push(Box::new(f));
      self
   }
}
```

这种类型定义了`return`和`bind`等操作。然而，`bind`函数不是多态的，操作也不是一个纯函数。如果没有这些妥协，我们就需要与 Rust 的类型和所有权系统作斗争；前面的例子不是以单子方式编写的，因为在尝试装箱和复制闭包时出现了复杂性。这是一个预期的权衡，当适当的时候，半单子模式不应该被劝阻。

为了定义我们的 Web 服务器响应，我们可以像以下代码那样附加处理器：

```rs
fn main() {
   ServerMonad::_return(())
               .bind_handler(|&mut st, ref msg| if msg.len()%2 == 0 { Some("divisible by 2".to_string()) } else { None })
               .bind_handler(|&mut st, ref msg| if msg.len()%3 == 0 { Some("divisible by 3".to_string()) } else { None })
               .bind_handler(|&mut st, ref msg| if msg.len()%5 == 0 { Some("divisible by 5".to_string()) } else { None })
               .bind_handler(|&mut st, ref msg| if msg.len()%7 == 0 { Some("divisible by 7".to_string()) } else { None })
               .listen("127.0.0.1:8888");
}
```

如果你运行这个程序并向本地主机`8888`发送消息，那么如果消息长度能被`2`、`3`、`5`或`7`整除，你可能会收到响应。

# 使用组合子模式

组合子是一个函数，它接受其他函数作为参数并返回一个新的函数。

组合子的一个简单例子是组合操作符，它将两个函数连接在一起：

```rs
fn compose<A,B,C,F,G>(f: F, g: G) -> impl Fn(A) -> C
   where F: 'static + Fn(A) -> B,
         G: 'static + Fn(B) -> C {
   move |x| g(f(x))
}

fn main() {
   let fa = |x| x+1;
   let fb = |y| y*2;
   let fc = |z| z/3;
   let g = compose(compose(fa,fb),fc);
   println!("g(1) = {}", g(1));
   println!("g(12) = {}", g(12));
   println!("g(123) = {}", g(123));
}
```

# 解析器组合子

组合子的另一个主要应用是解析器组合子。解析器组合子利用了单子和组合子模式。单子的`bind`函数用于从稍后返回的解析结果中绑定数据。组合子将解析器连接成序列、故障转移或其他模式。

`chomp`解析器组合子库是这个概念的很好实现。此外，该库提供了一个很好的`parse!`宏，使得组合子逻辑更容易阅读。以下是一个例子：

```rs
#[macro_use]
extern crate chomp;
use chomp::prelude::*;

#[derive(Debug, Eq, PartialEq)]
struct Name<B: Buffer> {
   first: B,
   last:  B,
}

fn name<I: U8Input>(i: I) -> SimpleResult<I, Name<I::Buffer>> {
   parse!{i;
      let first = take_while1(|c| c != b' ');
      token(b' ');  // skipping this char
      let last  = take_while1(|c| c != b'\n');

      ret Name{
         first: first,
         last:  last,
      }
   }
}

fn main() {
   let parse_result = parse_only(name, "Martin Wernstål\n".as_bytes()).unwrap();
   println!("first:{} last:{}",

```

```rs
      String::from_utf8_lossy(parse_result.first),
      String::from_utf8_lossy(parse_result.last));
}
```

在这里，示例定义了一个用于姓氏和名字的语法。在名字函数中，解析器使用宏定义。宏的内部看起来几乎像正常代码，如`let`语句、函数调用和闭包定义。然而，生成的代码实际上是单子和组合器的混合。

每个`let`绑定对应一个组合器。每个分号对应一个组合器。函数`take_while1`和`token`都是引入解析器单子的组合器。然后，当宏结束时，我们留下一个处理输入以解析结果的表达式。

这个`chomp`解析器组合库功能齐全，如果你只是随意查看源代码，可能会难以理解。为了了解这里发生了什么，让我们创建自己的解析器组合器。首先，让我们定义解析器状态：

```rs
use std::rc::Rc;

#[derive(Clone)]
struct ParseState<A: Clone> {
   buffer: Rc<Vec<char>>,
   index: usize,
   a: A
}

impl<A: Clone> ParseState<A> {
   fn new(a: A, buffer: String) -> ParseState<A> {
      let buffer: Vec<char> = buffer.chars().collect();
      ParseState {
         buffer: Rc::new(buffer),
         index: 0,
         a: a
      }
   }
   fn next(&self) -> (ParseState<A>,Option<char>) {
      if self.index < self.buffer.len() {
         let new_char = self.buffer[self.index];
         let new_index = self.index + 1;
         (ParseState {
            buffer: Arc::clone(&self.buffer),
            index: new_index,
            a: self.a.clone()
         }, Some(new_char))
      } else {
         (ParseState {
            buffer: Rc::clone(&self.buffer),
            index: self.index,
            a: self.a.clone()
         },None)
      }
   }
}

#[derive(Debug)]
struct ParseRCon<A,B>(A,Result<Option<B>,String>);

#[derive(Debug)]
enum ParseOutput<A> {
   Success(A),
   Failure(String)
}
```

在这里，我们定义了`ParseState`、`ParseRCon`和`ParseResult`。解析器状态跟踪解析器所在的字符索引。解析器状态通常还记录信息，如行号和列号。

`ParseRCon`结构封装了状态以及一个可选值，该值被封装在结果中。如果在解析过程中发生不可恢复的错误，结果将变为`Err`。如果在解析过程中发生可恢复的错误，选项将为`None`。否则，解析器应该基本上像它们期望始终有可选值一样工作。

`ParseResult`类型在解析执行的最后返回，以提供成功的结果或错误信息。

解析器单子和组合器使用不同的函数定义。要创建一个解析器，最简单的选项可能是`parse_mzero`和`parse_return`：

```rs
fn parse<St: Clone,A,P>(p: &P, st: &ParseState<St>) -> ParseOutput<A>
   where P: Fn(ParseState<St>) -> ParseRCon<ParseState<St>,A> {
   match p(st.clone()) {
      ParseRCon(_,Ok(Some(a))) => ParseOutput::Success(a),
      ParseRCon(_,Ok(None)) => ParseOutput::Failure("expected input".to_string()),
      ParseRCon(_,Err(err)) => ParseOutput::Failure(err)
   }
}

fn parse_mzero<St: Clone,A>(st: ParseState<St>) -> ParseRCon<ParseState<St>,A> {
   ParseRCon(st,Err("mzero failed".to_string()))
}

fn parse_return<St: Clone,A: Clone>(a: A) -> impl (Fn(ParseState<St>) -> ParseRCon<ParseState<St>,A>) {
   move |st| { ParseRCon(st,Ok(Some(a.clone()))) }
}

fn main() {
   let input1 = ParseState::new((), "1 + 2 * 3".to_string());
   let input2 = ParseState::new((), "3 / 2 - 1".to_string());

   let p1 = parse_mzero::<(),()>;
   println!("p1 input1: {:?}", parse(&p1,&input1));
   println!("p1 input2: {:?}", parse(&p1,&input2));

   let p2 = parse_return(123);
   println!("p2 input1: {:?}", parse(&p2,&input1));
   println!("p2 input2: {:?}", parse(&p2,&input2));
}
```

`parse_mzero`单子总是失败并返回一个简单的消息。`parse_return`总是成功并返回一个给定的值。

为了使事情更有趣，让我们实际看看一个消耗输入的解析器。我们创建了以下两个函数——`parse_token`和`parse_satisfy`。`parse_token`将始终消耗一个标记并返回其值，除非没有更多输入。`parse_satisfy`将消耗一个标记，如果标记满足某些条件。以下是定义：

```rs
fn parse_token<St: Clone,A,T>(t: T) -> impl (Fn(ParseState<St>) -> ParseRCon<ParseState<St>,A>)
   where T: 'static + Fn(char) -> Option<A> {
   move |st: ParseState<St>| {
      let (next_state,next_char) = st.clone().next();
      match next_char {
         Some(c) => ParseRCon(next_state,Ok(t(c))),
         None => ParseRCon(st,Err("end of input".to_string()))
      }
   }
}

fn parse_satisfy<St: Clone,T>(t: T) -> impl (Fn(ParseState<St>) -> ParseRCon<ParseState<St>,char>)
   where T: 'static + Fn(char) -> bool {
   parse_token(move |c| if t(c) {Some(c)} else {None})
}

fn main() {
   let input1 = ParseState::new((), "1 + 2 * 3".to_string());
   let input2 = ParseState::new((), "3 / 2 - 1".to_string());

   let p3 = parse_satisfy(|c| c=='1');
   println!("p3 input1: {:?}", parse(&p3,&input1));
   println!("p3 input2: {:?}", parse(&p3,&input2));

   let digit = parse_satisfy(|c| c.is_digit(10));
   println!("digit input1: {:?}", parse(&digit,&input1));
   println!("digit input2: {:?}", parse(&digit,&input2));

   let space = parse_satisfy(|c| c==' ');
   println!("space input1: {:?}", parse(&space,&input1));
   println!("space input2: {:?}", parse(&space,&input2));

   let operator = parse_satisfy(|c| c=='+' || c=='-' || c=='*' || c=='/');
   println!("operator input1: {:?}", parse(&operator,&input1));
   println!("operator input2: {:?}", parse(&operator,&input2));
}
```

`parse_token`和`parse_satisfy`查看一个标记。如果标记满足提供的条件，它将返回输入标记。在这里，我们创建几个条件来对应单个字符匹配、数字、空格或算术运算符。

这些函数可以使用高级组合器组合起来创建复杂的语法：

```rs
fn parse_bind<St: Clone,A,B,P1,P2,B1>(p1: P1, b1: B1)
   -> impl Fn(ParseState<St>) -> ParseRCon<ParseState<St>,B>
   where P1: Fn(ParseState<St>) -> ParseRCon<ParseState<St>,A>,
         P2: Fn(ParseState<St>) -> ParseRCon<ParseState<St>,B>,
         B1: Fn(A) -> P2 {
   move |st| {
      match p1(st) {
         ParseRCon(nst,Ok(Some(a))) => b1(a)(nst),
         ParseRCon(nst,Ok(None)) => ParseRCon(nst,Err("bind failed".to_string())),
         ParseRCon(nst,Err(err)) => ParseRCon(nst,Err(err))
      }
   }
}

fn parse_sequence<St: Clone,A,B,P1,P2>(p1: P1, p2: P2)
   -> impl Fn(ParseState<St>) -> ParseRCon<ParseState<St>,B>
   where P1: Fn(ParseState<St>) -> ParseRCon<ParseState<St>,A>,
         P2: Fn(ParseState<St>) -> ParseRCon<ParseState<St>,B> {
   move |st| {
      match p1(st) {
         ParseRCon(nst,Ok(_)) => p2(nst),
         ParseRCon(nst,Err(err)) => ParseRCon(nst,Err(err))
      }
   }
}

fn parse_or<St: Clone,A,P1,P2>(p1: P1, p2: P2)
   -> impl Fn(ParseState<St>) -> ParseRCon<ParseState<St>,A>
   where P1: Fn(ParseState<St>) -> ParseRCon<ParseState<St>,A>,
         P2: Fn(ParseState<St>) -> ParseRCon<ParseState<St>,A> {
   move |st| {
      match p1(st.clone()) {
         ParseRCon(nst,Ok(Some(a))) => ParseRCon(nst,Ok(Some(a))),
         ParseRCon(_,Ok(None)) => p2(st),
         ParseRCon(nst,Err(err)) => ParseRCon(nst,Err(err))
      }
   }
}

fn main() {
   let input1 = ParseState::new((), "1 + 2 * 3".to_string());
   let input2 = ParseState::new((), "3 / 2 - 1".to_string());

   let digit = parse_satisfy(|c| c.is_digit(10));
   let space = parse_satisfy(|c| c==' ');
   let operator = parse_satisfy(|c| c=='+' || c=='-' || c=='*' || c=='/');
   let ps1 = parse_sequence(digit,space);
   let ps2 = parse_sequence(ps1,operator);
   println!("digit,space,operator input1: {:?}", parse(&ps2,&input1));
   println!("digit,space,operator input2: {:?}", parse(&ps2,&input2));
}
```

这里，我们看到如何使用单子的`parse_bind`或其衍生物`parse_sequence`来串联两个解析器。这里没有示例，但失败组合器也在`parse_or`中定义。

使用这些原始工具，我们可以创建一些很好的工具来帮助我们生成复杂的解析器，这些解析器期望、存储和操作来自标记流的 数据。解析组合器是单子和组合器更实用但更具挑战性的应用之一。这些概念在 Rust 中成为可能的事实展示了该语言在支持函数式概念方面的发展程度。

# 使用惰性评估模式

惰性评估是推迟，将工作推迟到以后而不是现在。为什么这很重要？好吧，结果证明，如果你推迟足够长的时间，有时最终发现这项工作根本不需要完成！

以一个简单的表达式评估为例：

```rs
fn main()
{
   2 + 3;

   || 2 + 3;
}
```

在严格的解释下，第一个表达式将执行一个算术计算。第二个表达式将定义一个算术计算，但会等待然后再进行评估。

这种情况如此简单，以至于编译器会发出警告，并可能选择丢弃未使用的常量表达式。在更复杂的情况下，未评估的惰性评估情况将始终表现得更好。这应该是预期的，因为未使用的惰性表达式什么也不做，这是故意的。

迭代器是惰性的。它们在你收集或以其他方式迭代它们之前不会做任何事情：

```rs
fn main() {
   let a = (0..10).map(|x| x * x);
   //nothing yet

   for x in a {
      println!("{}", x);

```

```rs
   }
   //now it ran
}
```

另一个故意使用惰性评估的数据结构是惰性列表。惰性列表与迭代器非常相似，除了惰性列表可以独立地共享和以不同的速度消费。

在解析组合器示例中，我们在解析器状态结构中隐藏了一个惰性列表。让我们将其隔离出来，看看一个纯定义看起来像什么：

```rs
use std::rc::Rc;

#[derive(Clone)]
struct LazyList<A: Clone> {
   buffer: Rc<Vec<A>>,
   index: usize
}

impl<A: Clone> LazyList<A> {
   fn new(buf: Vec<A>) -> LazyList<A> {
      LazyList {
         buffer: Rc::new(buf),
         index: 0
      }
   }
   fn next(&self) -> Option<(LazyList<A>,A)> {
      if self.index < self.buffer.len() {
         let new_item = self.buffer[self.index].clone();
         let new_index = self.index + 1;
         Some((LazyList {
            buffer: Rc::clone(&self.buffer),
            index: new_index
         },new_item))
      } else {
         None
      }
   }
}

fn main()
{
   let ll = LazyList::new(vec![1,2,3]);
   let (ll1,a1) = ll.next().expect("expect 1 item");
   println!("lazy item 1: {}", a1);

   let (ll2,a2) = ll1.next().expect("expect 2 item");
   println!("lazy item 2: {}", a2);

   let (ll3,a3) = ll2.next().expect("expect 3 item");
   println!("lazy item 3: {}", a3);

   let (ll2,a2) = ll1.next().expect("expect 2 item");
   println!("lazy item 2: {}", a2);
}
```

在这里，我们可以看到惰性列表与迭代器非常相似。事实上，惰性列表可以实现 `Iterator` 特性；那么它就真的是一个迭代器了。然而，迭代器不是惰性列表。惰性列表本质上具有无限的前瞻能力，可以查看任意数量的项目。另一方面，迭代器可选地可以实现 `Peekable` 特性，允许向前查看。

尽管惰性编程的核心存在一个基本问题。过多的推迟将永远不会完成任何任务。如果你编写一个发射导弹的程序，在程序的某个时刻，它需要实际发射导弹。这是程序运行的一个不可逆的副作用。我们不喜欢副作用，而惰性编程对副作用持极端的反对态度。同时，我们还需要完成给定的任务，这涉及到在某个时刻做出选择，按下发射按钮。

显然，我们永远无法完全包含具有副作用程序的行为。然而，我们可以使它们更容易处理。通过将副作用包装到惰性评估表达式中，然后将它们转换为单子，我们创建的是副作用单元。然后我们可以以更函数式的方式对这些单元进行操作和组合。

我们将要引入的最后一种懒惰模式是**函数式响应式编程**，简称**FRP**。有一些基于这个概念的整个编程语言，例如 Elm。流行的网络 UI 框架，如 React 或 Angular，也受到了 FRP 概念的影响。

FRP 概念是副作用/状态单例示例的扩展。事件处理、状态转换和副作用可以转换为响应式编程的单位。让我们定义一个单例来捕获这个响应式单元概念：

```rs
struct ReactiveUnit<St,A,B> {
   state: Arc<Mutex<St>>,
   event_handler: Arc<Fn(&mut St,A) -> B>
}

impl<St: 'static,A: 'static,B: 'static> ReactiveUnit<St,A,B> {
   fn new<F>(st: St, f: F) -> ReactiveUnit<St,A,B>
      where F: 'static + Fn(&mut St,A) -> B
   {
      ReactiveUnit {
         state: Arc::new(Mutex::new(st)),
         event_handler: Arc::new(f)
      }
   }
   fn bind<G,C>(&self, g: G) -> ReactiveUnit<St,A,C>
      where G: 'static + Fn(&mut St,B) -> C {
      let ev = Arc::clone(&self.event_handler);
      ReactiveUnit {
         state: Arc::clone(&self.state),
         event_handler: Arc::new(move |st: &mut St,a| {
            let r = ev(st,a);
            let r = g(st,r);
            r
         })
      }
   }
   fn plus<St2: 'static,C: 'static>(&self, other: ReactiveUnit<St2,B,C>) -> ReactiveUnit<(Arc<Mutex<St>>,Arc<Mutex<St2>>),A,C> {
      let ev1 = Arc::clone(&self.event_handler);
      let st1 = Arc::clone(&self.state);
      let ev2 = Arc::clone(&other.event_handler);
      let st2 = Arc::clone(&other.state);
      ReactiveUnit {
         state: Arc::new(Mutex::new((st1,st2))),
         event_handler: Arc::new(move |stst: &mut (Arc<Mutex<St>>,Arc<Mutex<St2>>),a| {
            let mut st1 = stst.0.lock().unwrap();
            let r = ev1(&mut st1, a);
            let mut st2 = stst.1.lock().unwrap();
            let r = ev2(&mut st2, r);
            r
         })
      }
   }
   fn apply(&self, a: A) -> B {
      let mut st = self.state.lock().unwrap();
      (self.event_handler)(&mut st, a)
   }
}
```

在这里，我们发现`ReactiveUnit`可以持有状态，可以响应输入，产生副作用，并返回一个值。可以通过`bind`扩展`ReactiveUnit`或通过`plus`连接它们。

现在，让我们创建一个响应式单元。我们将关注网络框架，因为它们似乎很受欢迎。首先，我们渲染一个简单的 HTML 页面，如下所示：

```rs
let render1 = ReactiveUnit::new((),|(),()| {
   let html = r###"$('body').innerHTML = '
      <header>
         <h3 data-section="1" class="active">Section 1</h3>
         <h3 data-section="2">Section 2</h3>
         <h3 data-section="3">Section 3</h3>
      </header>
      <div>page content</div>
    <footer>Copyright</footer>
  ';"###;
  html.to_string()
});
println!("{}", render1.apply(()));
```

在这里，单元渲染了一个简单的页面，对应于网站上的`第一部分`。这个单元将始终渲染整个页面，不考虑任何状态或输入。让我们通过告诉单元根据哪个部分是活动的来给它更多的责任：

```rs
let render2 = ReactiveUnit::new((),|(),section: usize| {

   let section_1 = r###"$('body').innerHTML = '
      <header>
         <h3 data-section="1" class="active">Section 1</h3>
         <h3 data-section="2">Section 2</h3>
         <h3 data-section="3">Section 3</h3>
      </header>
      <div>section 1 content</div>
      <footer>Copyright</footer>
    ';"###;

    let section_2 = r###"$('body').innerHTML = '
      <header>
        <h3 data-section="1">Section 1</h3>
        <h3 data-section="2" class="active">Section 2</h3>
        <h3 data-section="3">Section 3</h3>
      </header>
      <div>section 2 content</div>
      <footer>Copyright</footer>
    ';"###;

    let section_3 = r###"$('body').innerHTML = '
      <header>
        <h3 data-section="1">Section 1</h3>
        <h3 data-section="2">Section 2</h3>
        <h3 data-section="3" class="active">Section 3</h3>
      </header>
      <div>section 3 content</div>
      <footer>Copyright</footer>
   ';"###;

   if section==1 {
      section_1.to_string()
   } else if section==2 {
      section_2.to_string()
   } else if section==3 {
      section_3.to_string()
   } else {
      panic!("unknown section")
   }
});

println!("{}", render2.apply(1));
println!("{}", render2.apply(2));
println!("{}", render2.apply(3));
```

在这里，单元使用参数来决定应该渲染哪个部分。这开始感觉更像是一个 UI 框架，但我们还没有使用状态。让我们尝试使用它来解决一个常见的网络问题——页面撕裂。当网页上的大量 HTML 发生变化时，浏览器必须重新计算页面应该如何显示。大多数现代浏览器都是分阶段进行这一操作的，结果是组件在页面上被明显地扔来扔去，显得很丑陋。

为了减少或防止页面撕裂，我们应只更新已更改的页面部分。让我们使用状态变量和输入参数，仅在组件发生变化时发送更新：

```rs
let render3header = ReactiveUnit::new(None,|opsec: &mut Option<usize>,section: usize| {
   let section_1 = r###"$('header').innerHTML = '
      <h3 data-section="1" class="active">Section 1</h3>
      <h3 data-section="2">Section 2</h3>
      <h3 data-section="3">Section 3</h3>
   ';"###;
   let section_2 = r###"$('header').innerHTML = '
      <h3 data-section="1">Section 1</h3>
      <h3 data-section="2" class="active">Section 2</h3>
      <h3 data-section="3">Section 3</h3>
   ';"###;
   let section_3 = r###"$('header').innerHTML = '
      <h3 data-section="1">Section 1</h3>
      <h3 data-section="2">Section 2</h3>
      <h3 data-section="3" class="active">Section 3</h3>
   ';"###;
   let changed = if section==1 {
      section_1
   } else if section==2 {
      section_2
   } else if section==3 {
      section_3
   } else {
      panic!("invalid section")
   };
   if let Some(sec) = *opsec {
      if sec==section { "" }
      else {
         *opsec = Some(section);
         changed
      }
   } else {
      *opsec = Some(section);
      changed
   }
});

```

在这里，我们发出命令以有条件地渲染标题的变化。如果标题已经处于正确的状态，则不执行任何操作。此代码仅负责标题组件。我们还需要渲染页面内容的变化：

```rs
let render3content = ReactiveUnit::new(None,|opsec: &mut Option<usize>,section: usize| {
   let section_1 = r###"$('div#content').innerHTML = '
      section 1 content
   ';"###;
   let section_2 = r###"$('div#content').innerHTML = '
      section 2 content
   ';"###;
   let section_3 = r###"$('div#content').innerHTML = '
      section 3 content
   ';"###;
   let changed = if section==1 {
      section_1
   } else if section==2 {
      section_2
   } else if section==3 {
      section_3
   } else {
      panic!("invalid section")
   };
   if let Some(sec) = *opsec {
      if sec==section { "" }
      else {
         *opsec = Some(section);
         changed
      }
   } else {
      *opsec = Some(section);
      changed
   }
});
```

现在，我们有一个用于标题的组件和另一个用于内容的组件。我们应该将这两个组件合并成一个单元。FRP 库可能有一个很酷的整洁方法来做这件事，但我们没有；所以，我们只是编写了一个小单元来手动合并它们：

```rs
let render3 = ReactiveUnit::new((render3header,render3content), |(rheader,rcontent),section: usize| {
   let header = rheader.apply(section);
   let content = rcontent.apply(section);
   format!("{}{}", header, content)
});
```

现在，让我们测试一下：

```rs
println!("section 1: {}", render3.apply(1));
println!("section 2: {}", render3.apply(2));
println!("section 2: {}", render3.apply(2));
println!("section 3: {}", render3.apply(3));
```

每个`apply`都会发出适当的新的更新命令。再次渲染`第二部分`的冗余`apply`不会返回任何命令，正如预期的那样。这实际上是一种懒惰的代码；是好的懒惰。

没有事件处理，响应式编程会是什么样子？让我们处理一些信号和事件。在页面状态之上，让我们引入一些数据库交互：

```rs
let database = ("hello world", 5, 2);
let react1 = ReactiveUnit::new((database,render3), |(database,render),evt:(&str,&str)| {
   match evt {
      ("header button click",n) => render.apply(n.parse::<usize>().unwrap()),
      ("text submission",s) => { database.0 = s; format!("db.textfield1.set(\"{}\")",s) },
      ("number 1 submission",n) => { database.1 += n.parse::<i32>().unwrap(); format!("db.numfield1.set(\"{}\")",database.1) },
      ("number 2 submission",n) => { database.2 += n.parse::<i32>().unwrap(); format!("db.numfield2.set(\"{}\")",database.2) },
      _ => "".to_string()
   }
});

println!("react 1: {}", react1.apply(("header button click","2")));
println!("react 1: {}", react1.apply(("header button click","2")));
println!("react 1: {}", react1.apply(("text submission","abc def")));
println!("react 1: {}", react1.apply(("number 1 submission","123")));
println!("react 1: {}", react1.apply(("number 1 submission","234")));
println!("react 1: {}", react1.apply(("number 2 submission","333")));
println!("react 1: {}", react1.apply(("number 2 submission","222")));
```

我们定义了四种事件类型以进行响应。响应页面状态变化仍然像之前定义的那样工作。应该与数据库交互的事件会发出命令以在本地和远程更新数据库。输出 JavaScript 的视图如下所示：

```rs
event: ("header button click", "2")
$('header').innerHTML = '
   <h3 data-section="1">Section 1</h3>
   <h3 data-section="2" class="active">Section 2</h3>
   <h3 data-section="3">Section 3</h3>
';$('div#content').innerHTML = '
   section 2 content
';

event: ("header button click", "2")

event: ("text submission", "abc def")
db.textfield1.set("abc def")

event: ("number 1 submission", "123")
db.numfield1.set("128")

event: ("number 1 submission", "234")
db.numfield1.set("362")

event: ("number 2 submission", "333")
db.numfield2.set("335")

event: ("number 2 submission", "222")
db.numfield2.set("557")
```

这个对应关系展示了如何将简单的副作用单元组合起来以创建复杂的程序行为。这一切都是从一个少于 50 行代码的 FRP 库中构建的。想象一下增加几个辅助函数的潜在效用。

# 摘要

在本章中，我们介绍了许多常见的函数式设计模式。我们使用了大量令人畏惧的词汇，如函子、单子和组合子。你应该努力记住这些词汇及其含义。其他令人畏惧的词汇，如逆变，除非你想追求数学，否则你可能可以忘记。

在应用场景中，我们了解到函子可以隐藏信息以暴露对数据的简单转换。单子模式允许我们将顺序操作转换为计算单元。单子可以用来创建也表现得像列表的迭代器。惰性求值可以用来延迟计算。此外，这些模式通常可以以有用的方式组合，例如 FRP，它作为开发用户界面和其他复杂交互程序的工具而越来越受欢迎。

在下一章中，我们将探讨并发。我们将介绍 Rust 的线程/数据所有权、共享同步数据和消息传递的概念。线程级别的并发是 Rust 特别设计用于的功能。如果你在其他语言中处理过线程，那么下一章可能会给你带来鼓舞。

# 问题

1.  什么是函子？

1.  逆变函子是什么？

1.  什么是单子？

1.  单子法则是什么？

1.  什么是组合子？

1.  为什么闭包返回值需要使用`impl`关键字？

1.  惰性求值是什么？
