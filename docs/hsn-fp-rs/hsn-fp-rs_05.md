# 代码组织和应用架构

之前，我们概述了一些项目规划和代码架构的基本概念。我们推荐的策略特别强调在将需求适应为伪代码、存根代码和最终项目之前，先收集和列出需求。这个过程对大型项目仍然非常适用，但我们还没有涵盖文件和模块组织方面。代码应该如何分组到文件和模块中？

为了回答这个问题，我们推荐一种称为**工作坊模式**的方法。想象一个物理工作坊，里面有挂板、架子、罐子、工具箱和地板上的大型设备。当谈到代码架构时，专家们经常讨论不同的组织策略。代码可以按类型、目的、项目层或便利性进行分组。有无限的可能策略，这里只列举了四种常见的策略。虽然我们不建议选择任何一种特定的策略，但它们都没有错。螺母和螺栓可以按类型放入罐子中。手工具可以放在工具箱中（按目的）。大型工具可以放在地板上（按项目层）。常用工具可以挂在挂板上（按便利性）。这些策略都不是无效的，并且都可以在同一个工作坊（项目）中使用。

在本章中，我们将随着项目的增长而重新组织项目。我们将结合之前介绍的计划和架构原则，以及新的代码组织概念，来开发一个可导航且可维护的大型软件项目。

本章的学习成果如下：

+   通过类型组织识别和应用

+   通过目的组织识别和应用

+   通过分层组织识别和应用

+   通过便利性组织识别和应用

+   在项目重组过程中最小化代码浪费

# 技术要求

运行提供的示例需要 Rust 的最近版本：

[Rust 安装指南](https://www.rust-lang.org/en-US/install.html)

本章的代码也可在 GitHub 上找到：

[《RUST 函数式编程实战》](https://github.com/PacktPublishing/Hands-On-Functional-Programming-in-RUST)

每章的`README.md`文件中也包含了具体的安装和构建说明。

# 在不牺牲质量的情况下交付产品

客户已经与你的销售团队完成了谈判——你赢得了合同。现在合同已经签署，你的团队的任务是将模拟提升到规格，以便运行所有电梯系统。客户为每个三个建筑、电梯、电机控制和制动系统提供了规格。你还了解到电梯电机具有智能电机控制软件，该软件可以动态调节内部电压和电流。为了控制电机，你只需提供所需的力输出。完整的规格如下：

+   对于建筑 1，有以下几点：

    +   **楼层高度**: 8m, 4m, 4m, 4m, 4m

    +   **电梯重量**: 1,200 kg

    +   **电梯电机**: 最大 50,000 N

    +   **电梯驱动器**: 提供软件接口

+   对于建筑 2，有以下几点：

    +   **楼层高度**: 5m, 5m, 5m, 5m, 5m, 5m, 5m, 5m

    +   **电梯重量**: 1,350 kg

    +   **电梯电机**: 最大 1,00,000 N

    +   **电梯驱动器**: 提供软件接口

+   对于建筑 3，有以下几点：

    +   **楼层高度**: 6m, 4m, 4m, 4m

    +   **电梯重量**: 1,400 kg

    +   **电梯电机**: 最大 90,000 N

    +   **电梯驱动器**: 提供软件接口

程序现在需要以操作模式运行，接受并添加新的楼层请求到队列中。模拟也应该继续运行，现在包含所有三个建筑规范。模拟应该验证承诺的性能和质量指标是否都满足。除此之外，你的团队可以自由地按照你的想法开发项目。

你决定现在是重新思考项目组织的好时机，需要重大的新变化。使用良好的架构和项目组织实践，你将相应地移动代码，以便有序且方便地分组组件。

# 重新组织项目

既然我们已经对良好的项目架构有了些想法，让我们规划项目的重组。让我们列出可能的研讨会组织方法：

+   按类型

+   按用途

+   按层

+   按便利性

应该使用按类型组织来处理车间螺母和螺栓类型的组件。螺母和螺栓是高度统一的组件，具有不同的直径、长度、等级等。我们这里也有一些很好的匹配，让我们列出可以按这种方式分组的对象和接口：

+   电机

+   建筑物

+   电梯控制器/驱动器

应该使用按用途组织来处理具有共同目的的杂项工具。我们也有一些适合这种组织风格的优秀候选者：

+   运输计划（静态/动态）

+   电梯的物理接口

应该使用按层组织来处理适合正常程序逻辑的独立建筑组件。一个例子是我们的物理层，它在逻辑上独立于其他模块。物理层仅用于存储常数、公式和建模过程。在这里，我们按层分组：

+   物理建模

应使用方便组织来组织常见或难以组织的组件。可执行文件非常适合这种类型的组织，因为它们始终是终点，而不是库，并且通常不适合其他任何组织：

+   模拟可执行文件

+   分析可执行文件

+   物理电梯驱动程序可执行文件

# 根据类型规划文件内容

这些文件将使用按类型方法进行组织。

# 组织 motor_controllers.rs 模块

所有电机将在 `motor_controller.rs` 模块中按类型分组。将有三种具有不同特性的电机。该模块应提供对所有电机以及每个实现的特质接口。特质应定义一个方法，从所需的力输出生成电机输入，以及一个方法来接受电机输入以生成力。该模块还必须链接到每个电机控制器的二进制驱动程序。旧的电机控制器逻辑将移动到一个名为 `motion_controllers.rs` 的新文件中，以动态控制电梯电机。以下内容应在该模块中定义：

+   电机输入特质

+   电机控制器特质

+   电机输入 1 实现

+   电机控制器 1 实现

+   电机输入 2 实现

+   电机控制器 2 实现

+   电机输入 3 实现

+   电机控制器 3 实现

# 组织 buildings.rs 模块

所有建筑规范将在 `building.rs` 模块中按类型分组。将有三种建筑规范。建筑物应封装电梯行为和控制的所有方面，以及建筑本身的规范。该模块应包含以下内容：

+   建筑特质

+   建筑物 1 实现

+   建筑物 2 实现

+   建筑物 3 实现

# 根据目的规划文件内容

这些文件将使用按目的方法进行组织。

# 组织 motion_controllers.rs 模块

运动控制器将根据目的进行组织。运动控制器将负责跟踪电梯状态以控制电机的动态。运动控制器模块应包含以下内容：

+   运动控制器特质

+   平滑运动控制器实现

# 组织 trip_planning.rs 模块

行程规划将根据目的进行组织。规划器应工作在两种模式下：静态和动态。对于静态模式，规划器应接受要处理的楼层请求列表。对于动态模式，规划器应动态接受楼层请求并将它们添加到队列中。规划器模块应包含以下内容：

+   规划器特质

+   静态规划器实现

+   动态规划器实现

# 组织 elevator_drivers.rs 模块

所有电梯驱动器将在`elevator_driver.rs`模块中按目的组织。有三个电梯驱动器提供二进制接口以进行链接。`elevator driver`模块应包含一个 trait，用于定义电梯驱动器的接口以及三个实现。`planner`模块应包含以下内容：

+   电梯驱动器 trait

+   电梯司机 1 实现

+   电梯司机 2 实现

+   电梯司机 3 实现

# 按层规划文件内容

这些文件将使用“分层”方法进行组织。

# 组织 physics.rs 模块

`physics`模块将按层组织所有与物理相关的代码。虽然这里会有一些杂项代码，但它们都应该以某种模拟或预测的形式存在。该模块应包含以下内容：

+   单位转换

+   公式实现

+   任何其他用于电梯模拟或操作所需的逻辑

+   物理模拟循环

# 组织 data_recorder.rs 模块

数据记录器模块将`DataRecorder` trait 和实现移动到其自己的模块。它应包含以下内容：

+   `DataRecorder` trait

+   简单数据记录器实现

# 按便利性规划文件内容

这些文件将使用“便利性”方法进行组织。

# 组织 simulate_trip.rs 可执行文件

`simulate_trip.rs`可执行文件将根据便利性进行组织。行程模拟可执行文件的范围没有发生显著变化。此文件应包含以下内容：

+   参数和输入解析

+   数据记录器定义

+   模拟设置

+   运行模拟

# 组织 analyze_trip.rs 可执行文件

`analyze_trip.rs`可执行文件将根据便利性进行组织。分析行程可执行文件的范围没有发生显著变化。此文件应包含以下内容：

+   参数和输入解析

+   检查规格以确定接受或拒绝

# 组织 operate_elevator.rs 可执行文件

`operate_elevator.rs`可执行文件将根据便利性进行组织。操作电梯可执行文件应与模拟电梯可执行文件的逻辑非常相似。此文件应包含以下内容：

+   参数和输入解析

+   设置电梯驱动器以匹配指定的建筑规范

+   使用动态规划运行电梯

# 映射代码更改和添加

现在我们已经将概念、数据结构和逻辑组织到文件中，我们可以继续进行将需求转换为代码的正常流程。对于每个模块，我们将查看所需元素并生成代码以满足这些需求。

在这里，我们通过模块分解所有代码开发步骤。不同的模块有不同的组织结构，因此请注意有关组织和代码开发的模式。

# 按类型开发代码

这些文件将使用“按类型”方法进行组织。

# 编写 motor_controllers.rs 模块

新的 `motor_controller` 模块作为所有链接的电机驱动器和它们的接口的适配器，并提供一个单一的统一接口。让我们看看它是如何实现的：

1.  首先，让我们将软件提供的所有驱动器链接到我们的程序中：

```rs
use libc::c_int;

#[link(name = "motor1")]
extern {
   pub fn motor1_adjust_motor(target_force: c_int) -> c_int;
}

#[link(name = "motor2")]
extern {
   pub fn motor2_adjust_motor(target_force: c_int) -> c_int;
}

#[link(name = "motor3")]
extern {
   pub fn motor3_adjust_motor(target_force: c_int) -> c_int;
}
```

这一部分告诉我们的程序链接到名为 `libmotor1.a`、`libmotor2.a` 和 `libmotor3.a` 等的静态编译库。我们的示例章节还包含了这些库的源代码和构建脚本，因此您可以检查每个库。在一个完整的项目中，有许多方法可以链接到外部二进制库，这仅仅是许多选项之一。

1.  接下来，我们应该为 `MotorInput` 创建一个特质，并为每个电机创建一个泛型 `MotorDriver` 接口，包括每个电机的实现。代码如下：

```rs
#[derive(Clone,Serialize,Deserialize,Debug)]
pub enum MotorInput
{
   Motor1 { target_force: f64 },
   Motor2 { target_force: f64 },
   Motor3 { target_force: f64 },
}

pub trait MotorDriver
{
   fn adjust_motor(&self, input: MotorInput);
}

struct Motor1;
impl MotorDriver for Motor1 { ... }

//Motor 2

//Motor 3
```

1.  接下来，我们应该实现电机控制器特质及其实现。电机控制器应该将电机信息和驱动器封装成一个统一的接口。这里的 `MotorDriver` 和 `MotorController` 特质被强制转换为简单的上下力模型。因此，驱动器和控制器之间的关系是一对一，不能完全抽象成一个通用特质。相应的代码如下：

```rs
pub trait MotorController
{
   fn adjust_motor(&self, f: f64);
   fn max_force(&self) -> f64;
}

pub struct MotorController1
{
   motor: Motor1
}

impl MotorController for MotorController1 { ... }

//Motor Controller 2 ...

//Motor Controller 3 ...
```

这些模块的完整代码可以在 GitHub 仓库中找到：[`github.com/PacktPublishing/Hands-On-Functional-Programming-in-RUST`](https://github.com/PacktPublishing/Hands-On-Functional-Programming-in-RUST).

# 编写 buildings.rs 模块

构建模块再次按类型分组。应该有一个通用的特质接口，由三个建筑实现。建筑特质和结构应该额外封装并暴露给适当的电梯驱动器和电机控制器。代码如下：

1.  首先，我们定义 `Building` 特质：

```rs
pub trait Building
{
   fn get_elevator_driver(&self) -> Box<ElevatorDriver>;
   fn get_motor_controller(&self) -> Box<MotorController>;
   fn get_floor_heights(&self) -> Vec<f64>;
   fn get_carriage_weight(&self) -> f64;
   fn clone(&self) -> Box<Building>;
   fn serialize(&self) -> u64;
}
```

1.  然后，我们定义一个 `deserialize` 辅助函数：

```rs
pub fn deserialize(n: u64) -> Box<Building>
{
   if n==1 {
      Box::new(Building1)
   } else if n==2 {
      Box::new(Building2)
   } else {
      Box::new(Building3)
   }
}
```

1.  然后，我们定义一些杂项辅助函数：

```rs
pub fn getCarriageFloor(floorHeights: Vec<f64>, height: f64) -> u64
{
   let mut c = 0.0;
   for (fi, fht) in floorHeights.iter().enumerate() {
      c += fht;
      if height <= c {
         return (fi as u64)
      }
   }
   (floorHeights.len()-1) as u64
}

pub fn getCumulativeFloorHeight(heights: Vec<f64>, floor: u64) -> f64
{
   heights.iter().take(floor as usize).sum()
}
```

1.  最后，我们定义建筑及其特质实现：

```rs
pub struct Building1;
impl Building for Building1 { ... }

//Building 2

//Building 3
```

# 按目的开发代码

这些文件将使用按目的方法进行组织。

# 编写 motion_controllers.rs 模块

从 `motor_controllers.rs` 中来的旧逻辑，用于动态调整电机力，将被移动到这个模块。`SmoothMotionController` 没有太大变化，代码如下：

```rs
pub trait MotionController
{
   fn init(&mut self, esp: Box<Building>, est: ElevatorState);
   fn adjust(&mut self, est: &ElevatorState, dst: u64) -> f64;
}

pub struct SmoothMotionController
{
   pub esp: Box<Building>,
   pub timestamp: f64
}

impl MotionController for SmoothMotionController
{
   ...
}
```

# 编写 trip_planning.rs 模块

行程规划器应在静态和动态模式下工作。基本结构是一个 FIFO 队列，将请求推入队列，并弹出最旧的元素。我们可能能够将静态和动态模式统一到一个实现中，其外观如下。

行程规划将按目的进行组织。规划器应在两种模式下工作——静态和动态。对于静态模式，规划器应接受一个楼层请求列表进行处理。对于动态模式，规划器应动态接受楼层请求并将它们添加到队列中。规划器模块应包含以下内容：

```rs
use std::collections::VecDeque;

pub struct FloorRequests
{
   pub requests: VecDeque<u64>
}

pub trait RequestQueue
{
   fn add_request(&mut self, req: u64);
   fn add_requests(&mut self, reqs: &Vec<u64>);
   fn pop_request(&mut self) -> Option<u64>;
}

impl RequestQueue for FloorRequests
{
   fn add_request(&mut self, req: u64)
   {
      self.requests.push_back(req);
   }
   fn add_requests(&mut self, reqs: &Vec<u64>)
   {
      for req in reqs
      {
         self.requests.push_back(*req);
      }
   }
   fn pop_request(&mut self) -> Option<u64>
   {
      self.requests.pop_front()
   }
}
```

# 编写 elevator_drivers.rs 模块

电梯驱动器模块应与提供的静态库接口，并额外提供一个通用接口供所有电梯驱动器使用。代码如下：

```rs
use libc::c_int;

#[link(name = "elevator1")]
extern {
   pub fn elevator1_poll_floor_request() -> c_int;
}

#[link(name = "elevator2")]
extern {
   pub fn elevator2_poll_floor_request() -> c_int;
}

#[link(name = "elevator3")]
extern {
   pub fn elevator3_poll_floor_request() -> c_int;
}

pub trait ElevatorDriver
{
   fn poll_floor_request(&self) -> Option<u64>;
}

pub struct ElevatorDriver1;
impl ElevatorDriver for ElevatorDriver1
{
   fn poll_floor_request(&self) -> Option<u64>
   {
      unsafe {
         let req = elevator1_poll_floor_request();
         if req > 0 {
            Some(req as u64)
         } else {
            None
         }
      }
   }
}

//Elevator Driver 2

//Elevator Driver 3
```

# 分层开发代码

这些文件将使用“分层”方法进行组织。

# 编写 `physics.rs` 模块

物理模块已经变得很小。它现在包含一些结构定义和常量以及中心的 `simulate_elevator` 方法。结果如下：

```rs
#[derive(Clone,Debug,Serialize,Deserialize)]
pub struct ElevatorState {
   pub timestamp: f64,
   pub location: f64,
   pub velocity: f64,
   pub acceleration: f64,
   pub motor_input: f64
}

pub const MAX_JERK: f64 = 0.2;
pub const MAX_ACCELERATION: f64 = 2.0;
pub const MAX_VELOCITY: f64 = 5.0;

pub fn simulate_elevator(esp: Box<Building>, est: ElevatorState, floor_requests: &mut Box<RequestQueue>,
                         mc: &mut Box<MotionController>, dr: &mut Box<DataRecorder>)
{
   //immutable input becomes mutable local state
   let mut esp = esp.clone();
   let mut est = est.clone();

   //initialize MotorController and DataController
   mc.init(esp.clone(), est.clone());
   dr.init(esp.clone(), est.clone());

   //5\. Loop while there are remaining floor requests
   let original_ts = Instant::now();
   thread::sleep(time::Duration::from_millis(1));
   let mut next_floor = floor_requests.pop_request();
   while let Some(dst) = next_floor
   {
      //5.1\. Update location, velocity, and acceleration
      let now = Instant::now();
      let ts = now.duration_since(original_ts)
                  .as_fractional_secs();
      let dt = ts - est.timestamp;
      est.timestamp = ts;

      est.location = est.location + est.velocity * dt;
      est.velocity = est.velocity + est.acceleration * dt;
      est.acceleration = {
         let F = est.motor_input;
         let m = esp.get_carriage_weight();
         -9.8 + F/m
      };

      //5.2\. If next floor request in queue is satisfied, then remove from queue
      if (est.location - getCumulativeFloorHeight(esp.get_floor_heights(), dst)).abs() < 0.01 &&
         est.velocity.abs() < 0.01
      {
         est.velocity = 0.0;
         next_floor = floor_requests.pop_request();
      }

      //5.4\. Print realtime statistics
      dr.poll(est.clone(), dst);

      //5.3\. Adjust motor control to process next floor request
      est.motor_input = mc.poll(est.clone(), dst);

      thread::sleep(time::Duration::from_millis(1));
   }
}
```

# 编写 `data_recorders.rs` 模块

为了分离责任并防止单个模块过大，我们应该将数据记录器实现从模拟中移出，并放入其自己的模块。结果如下：

1.  定义 `DataRecorder` 特性：

```rs
pub trait DataRecorder
{
   fn init(&mut self, esp: Box<Building>, est: ElevatorState);
   fn record(&mut self, est: ElevatorState, dst: u64);
   fn summary(&mut self);
}

```

1.  定义 `SimpleDataRecorder` 结构体：

```rs
struct SimpleDataRecorder<W: Write>
{
   esp: Box<Building>,
   termwidth: u64,
   termheight: u64,
   stdout: raw::RawTerminal<W>,
   log: File,
   record_location: Vec<f64>,
   record_velocity: Vec<f64>,
   record_acceleration: Vec<f64>,
   record_force: Vec<f64>,
}
```

1.  定义 `SimpleDataRecorder` 构造函数：

```rs
pub fn newSimpleDataRecorder(esp: Box<Building>) -> Box<DataRecorder>
{
   let termsize = termion::terminal_size().ok();
   Box::new(SimpleDataRecorder {
      esp: esp.clone(),
      termwidth: termsize.map(|(w,_)| w-2).expect("termwidth") as u64,
      termheight: termsize.map(|(_,h)| h-2).expect("termheight") as u64,
      stdout: io::stdout().into_raw_mode().unwrap(),
      log: File::create("simulation.log").expect("log file"),
      record_location: Vec::new(),
      record_velocity: Vec::new(),
      record_acceleration: Vec::new(),
      record_force: Vec::new()
   })
}
```

1.  定义 `DataRecorder` 特性的 `SimpleDataRecorder` 实现：

```rs
impl<W: Write> DataRecorder for SimpleDataRecorder<W>
{
   fn init(&mut self, esp: Box<Building>, est: ElevatorState)
   {
      ...
   }
   fn record(&mut self, est: ElevatorState, dst: u64)
      ...
   }
   fn summary(&mut self)
   {
      ...
   }
}
```

1.  定义各种辅助函数：

```rs
fn variable_summary<W: Write>(stdout: &mut raw::RawTerminal<W>, vname: String, data: &Vec<f64>) {
   let (avg, dev) = variable_summary_stats(data);
   variable_summary_print(stdout, vname, avg, dev);
}

fn variable_summary_stats(data: &Vec<f64>) -> (f64, f64)
{
   //calculate statistics
   let N = data.len();
   let sum = data.iter().sum::<f64>();
   let avg = sum / (N as f64);
   let dev = (
       data.clone().into_iter()
       .map(|v| (v - avg).powi(2))
       .sum::<f64>()
       / (N as f64)
   ).sqrt();
   (avg, dev)
}

fn variable_summary_print<W: Write>(stdout: &mut raw::RawTerminal<W>, vname: String, avg: f64, dev: f64)
{
   //print formatted output
   writeln!(stdout, "Average of {:25}{:.6}", vname, avg);
   writeln!(stdout, "Standard deviation of {:14}{:.6}", vname, dev);
   writeln!(stdout, "");
}
```

# 按方便开发代码

这些文件将使用“方便”方法进行组织。

# 编写 `simulate_trip.rs` 可执行文件

模拟行程变化很大，因为已经移除了 `DataRecorder` 逻辑。模拟的初始化也与之前有很大不同。最终结果如下：

1.  初始化 `ElevatorState`：

```rs
//1\. Store location, velocity, and acceleration state
//2\. Store motor input target force
let mut est = ElevatorState {
   timestamp: 0.0,
   location: 0.0,
   velocity: 0.0,
   acceleration: 0.0,
   motor_input: 0.0
};
```

1.  初始化建筑描述和楼层请求：

```rs
//3\. Store input building description and floor requests
let mut esp: Box<Building> = Box::new(Building1);
let mut floor_requests: Box<RequestQueue> = Box::new(FloorRequests {
   requests: Vec::new()
});
```

1.  解析输入并将其存储为建筑描述和楼层请求：

```rs
//4\. Parse input and store as building description and floor requests
match env::args().nth(1) {
   Some(ref fp) if *fp == "-".to_string()  => {
      ...
   },
   None => {
      ...
   },
   Some(fp) => {
      ...
   }
}
```

1.  初始化数据记录器和运动控制器：

```rs
let mut dr: Box<DataRecorder> = newSimpleDataRecorder(esp.clone());
let mut mc: Box<MotionController> = Box::new(SmoothMotionController {
  timestamp: 0.0,
  esp: esp.clone()
});
```

1.  运行电梯模拟：

```rs
simulate_elevator(esp, est, &mut floor_requests, &mut mc, &mut dr);
```

1.  打印模拟摘要：

```rs
dr.summary();
```

# 编写 `analyze_trip.rs` 可执行文件

分析行程的可执行文件将只做一点点改变，但只是为了适应已经移动的符号和现在可以使用 SerDe 序列化的类型。结果如下：

1.  定义 `Trip` 数据结构：

```rs
#[derive(Clone)]
struct Trip {
   dst: u64,
   up: f64,
   down: f64
}
```

1.  初始化变量：

```rs
let simlog = File::open("simulation.log").expect("read simulation log");
let mut simlog = BufReader::new(&simlog);
let mut jerk = 0.0;
let mut prev_est: Option<ElevatorState> = None;
let mut dst_timing: Vec<Trip> = Vec::new();
let mut start_location = 0.0;
```

1.  遍历日志行并初始化电梯规范：

```rs
let mut first_line = String::new();
let len = simlog.read_line(&mut first_line).unwrap();
let spec: u64 = serde_json::from_str(&first_line).unwrap();
let esp: Box<Building> = buildings::deserialize(spec);

for line in simlog.lines() {
   let l = line.unwrap();
   //Check elevator state records
}
```

1.  检查电梯状态记录：

```rs
let (est, dst): (ElevatorState,u64) = serde_json::from_str(&l).unwrap();
let dl = dst_timing.len();
if dst_timing.len()==0 || dst_timing[dl-1].dst != dst {
   dst_timing.push(Trip { dst:dst, up:0.0, down:0.0 });
}

if let Some(prev_est) = prev_est {
   let dt = est.timestamp - prev_est.timestamp;
   if est.velocity > 0.0 {
      dst_timing[dl-1].up += dt;
   } else {
      dst_timing[dl-1].down += dt;
   }
   let da = (est.acceleration - prev_est.acceleration).abs();
   jerk = (jerk * (1.0 - dt)) + (da * dt);
   if jerk.abs() > 0.22 {
      panic!("jerk is outside of acceptable limits: {} {:?}", jerk, est)
   }
} else {
   start_location = est.location;
}

if est.acceleration.abs() > 2.2 {
   panic!("acceleration is outside of acceptable limits: {:?}", est)
}

if est.velocity.abs() > 5.5 {
   panic!("velocity is outside of acceptable limits: {:?}", est)
}

prev_est = Some(est);
```

1.  检查电梯是否倒退：

```rs
//elevator should not backup
let mut total_time = 0.0;
let mut total_direct = 0.0;
for trip in dst_timing.clone()
{
   total_time += (trip.up + trip.down);
   if trip.up > trip.down {
      total_direct += trip.up;
   } else {
      total_direct += trip.down;
   }
}

if (total_direct / total_time) < 0.9 {
   panic!("elevator back up is too common: {}", total_direct / total_time)
}
```

1.  确保行程在理论极限的 20%内完成：

```rs
let mut trip_start_location = start_location;
let mut theoretical_time = 0.0;
let floor_heights = esp.get_floor_heights();
for trip in dst_timing.clone()
{
   let next_floor = getCumulativeFloorHeight(floor_heights.clone(), trip.dst);
   let d = (trip_start_location - next_floor).abs();
   theoretical_time += (
      2.0*(MAX_ACCELERATION / MAX_JERK) +
      2.0*(MAX_JERK / MAX_ACCELERATION) +
      d / MAX_VELOCITY
   );
   trip_start_location = next_floor;
}

if total_time > (theoretical_time * 1.2) {
   panic!("elevator moves to slow {} {}", total_time, theoretical_time * 1.2)
}
```

# 编写 `operate_elevator.rs` 可执行文件

操作电梯与 `simulate_trip.rs` 和物理 `run_simulation` 代码非常相似。最显著的区别是能够在动态接受新请求的同时继续运行，并使用链接库调整电机控制。在主可执行文件中，我们遵循与之前相同的逻辑过程，但调整了新名称和类型签名：

1.  初始化 `ElevatorState`：

```rs
//1\. Store location, velocity, and acceleration state
//2\. Store motor input target force
let mut est = ElevatorState {
   timestamp: 0.0,
   location: 0.0,
   velocity: 0.0,
   acceleration: 0.0,
   motor_input: 0.0
};
```

1.  初始化 `MotionController`：

```rs
let mut mc: Box<MotionController> = Box::new(SmoothMotionController {
   timestamp: 0.0,
   esp: esp.clone()
});
mc.init(esp.clone(), est.clone());
```

1.  启动操作循环以处理传入的楼层请求：

```rs
//5\. Loop continuously checking for new floor requests
let original_ts = Instant::now();
thread::sleep(time::Duration::from_millis(1));
let mut next_floor = floor_requests.pop_request();
while true
{
   if let Some(dst) = next_floor {
      //process floor request
   }
```

```rs

   //check for dynamic floor requests
   if let Some(dst) = esp.get_elevator_driver().poll_floor_request()
   {
      floor_requests.add_request(dst);
   }
}
```

1.  在处理循环中，更新物理近似值：

```rs
//5.1\. Update location, velocity, and acceleration
let now = Instant::now();
let ts = now.duration_since(original_ts)
            .as_fractional_secs();
let dt = ts - est.timestamp;
est.timestamp = ts;

est.location = est.location + est.velocity * dt;
est.velocity = est.velocity + est.acceleration * dt;
est.acceleration = {
   let F = est.motor_input;
   let m = esp.get_carriage_weight();
   -9.8 + F/m
};
```

1.  如果当前楼层请求得到满足，则从队列中移除它：

```rs
//5.2\. If next floor request in queue is satisfied, then remove from queue
if (est.location - getCumulativeFloorHeight(esp.get_floor_heights(), dst)).abs() < 0.01 && est.velocity.abs() < 0.01
{
   est.velocity = 0.0;
   next_floor = floor_requests.pop_request();
}
```

1.  调整电机控制：

```rs
//5.3\. Adjust motor control to process next floor request
est.motor_input = mc.poll(est.clone(), dst);

//Adjust motor
esp.get_motor_controller().adjust_motor(est.motor_input);
```

# 反思项目结构

现在我们已经开发了代码来组织和连接不同的电梯功能，以及三个可执行文件来模拟、分析和操作电梯，让我们问问自己这个问题——这一切是如何结合在一起的，我们到目前为止在架构这个项目方面做得好吗？

回顾本章，我们可以迅速看到我们已经使用了四种不同的代码组织技术。在更随意的层面上，代码似乎可以分为以下类别：

+   **行李**：就像需要连接的驾驶员一样，但可能难以合作

+   **螺母**、**螺栓**和**齿轮**：就像结构体和特质一样，我们对如何设计有相当的控制

+   **可交付成果**：就像可执行文件一样，这些必须满足特定要求

我们已经根据便利性组织了所有可交付成果；所有行李根据类型或目的进行分类；螺母、螺栓和齿轮根据类型、目的或层次进行组织。结果可能更糟，但按照不同的标准组织并不意味着代码会有显著变化。总的来说，可交付成果由相当可维护的代码支持，项目正在朝着良好的方向发展。

# 摘要

在本章中，我们探讨了四种代码组织原则，这些原则可以单独使用或组合使用来开发结构良好的项目。按类型、按目的、按层次和按便利性组织的四个组织原则是帮助在构建大型项目时做出良好架构选择的有帮助的视角。项目越大、越复杂，这些决策就越重要，尽管同时改变这些决策也更困难。

应用这些概念，我们根据每个原则的不同程度重构了整个项目。我们还进行了重大更改，以允许与外部库进行接口，并应用电梯操作，而不是封闭的模拟。现在，三座建筑的电梯应该能够完全运行我们在这里开发的软件。

在下一章中，我们将学习关于可变性和所有权的知识。我们已经对这些概念进行了一定程度的覆盖，但下一章将要求对具体细节和限制有更深入的理解。

# 问题

1.  有四种方法可以将代码分组到模块中吗？

1.  FFI 代表什么？

1.  为什么不安全块是必要的？

1.  是否在某个时候使用不安全块是安全的？

1.  `libc::c_int`和`int32`之间有什么区别？

1.  链接库能否定义具有相同名称的函数？

1.  可以将哪些类型的文件链接到 Rust 项目中？
