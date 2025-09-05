# 函数式数据结构

数据结构是编程的第二个最基本构建块，仅次于控制流。在早期语言开发控制流结构之后，很快就很明显，简单的变量标签不足以开发复杂程序。数据结构已经从存储在地址上的固定大小数据的基本概念演变为字符串和数组，然后是混合结构，最后是集合。

在本章中，我们将回顾在第二章中介绍的项目，即*函数式控制流*。由于潜在客户反馈，项目需求已经扩展以适应。由于竞争对手的开发，还必须满足特定的性能目标。为了帮助我们的业务成功，我们现在必须改进之前的模拟，并确保它满足客户需求和性能目标。

在本章中，我们将涵盖以下内容：

+   适应项目范围的变更

+   重新格式化代码以支持多个用例

+   使用适当的数据结构来收集、存储和处理数据

+   将代码组织到特性和数据类中

# 技术需求

运行提供的示例需要使用 Rust 的最新版本：

[`www.rust-lang.org/en-US/install.html`](https://www.rust-lang.org/en-US/install.html)

本章的代码也可在 GitHub 上找到：

[`github.com/PacktPublishing/Hands-On-Functional-Programming-in-RUST`](https://github.com/PacktPublishing/Hands-On-Functional-Programming-in-RUST)

每章的 `README.md` 文件中也包含了具体的安装和构建说明。

# 适应项目范围的变更

你不可能为所有事情都做计划。你也可能不想尝试为所有事情都做计划。灵活的软件开发和强调健壮、逻辑上独立的组件，当需求或依赖不可避免地发生变化时，可以减少工作量。

# 收集新的项目需求

在初步演示之后，您的团队收到了潜在客户的评论和反馈。观看模拟时，电梯似乎经常在停止前经过并返回楼层。客户表示担忧，这不仅效率低下，而且对乘客来说可能是不舒适或烦躁的。为了赢得合同，客户希望看到改进和证据，以证明：

+   乘坐体验舒适且可靠直接

+   电梯从每个源头高效地移动到每个目标楼层

此外，您还了解到竞争对手提交了一份单独的提案。竞争对手具体声称其电梯控制系统保持加速度在舒适水平，速度在安全范围内，并在物理理论极限的 20%内准确到达目的地。没有提供具体数字，也没有演示模拟，但客户似乎非常确信，并保证项目成本将降低 10%。

# 从需求构建变更图

在收到反馈和新期望后，我们必须将这些需求转化为行动计划。需要更新模拟，并需要构建额外的工具。让我们回顾新信息，并设计一个解决方案以满足新的需求。

# 将期望转化为需求

审查反馈，很明显需要解决两个观点：

+   竞争对手提出了具体的主张，我们的公司需要超越

+   客户对第一次演示中的担忧有明确的期望

竞争对手的具体主张可以列出如下：

+   加速度在舒适范围内

+   速度在安全范围内

+   从任何楼层到任何其他楼层的行程时间在物理理论极限的 20%以内

+   该软件便宜 10%

我们将把价格谈判委托给我们的销售团队，但除此之外，我们需要调整我们的软件以超越其他三个主张。如果我们能够满足这些要求并提供充分的证据，那么这也应该解决客户的大部分明确担忧。

此外，客户特别关注电梯通过目标楼层并需要倒退的行为。我们应该解决这个问题，并确认在模拟中不会发生这种行为。

到目前为止，很明显之前的电机控制逻辑是不够的。在头脑风暴后，您的团队开发了两种可能的改进：

+   使用可变加速度/减速度计算，而不是开/关调整

+   减少更新间隔以允许更快且更精确的决策

# 将需求转化为变更图

考虑到各种新的要求，将之前的模拟代码拆分为不同的库和可执行文件似乎是合适的。我们将为以下每个创建一个单独的模块：

+   一个物理模拟器

+   电机控制

+   一个用于演示模拟的可执行文件

+   一个用于进一步分析模拟的可执行文件

物理模拟器应接受一个泛型电机控制器和一个测量累加器。提供的测量累加器将接受速度、加速度和模拟器可用的所有其他信息的读数。提供的电机控制器将接受类似的速度等读数，并产生对电机所需的电压输出。该函数将负责准确模拟任何指定电梯和建筑的物理操作。

电机控制将与模拟器或最终的实际电梯连接，以使用可用信息来决定如何操作电梯。

模拟可执行文件将包装物理模拟器和电机控制，以创建与第二章 功能控制流中的模拟等效的程序。此外，所有记录的模拟信息都应保存到文件中，以进行进一步详细分析。

分析可执行文件应接受模拟器跟踪文件并检查是否满足所有性能要求。此外，任何对开发目的有用的分析都将在此处添加。

# 将需求直接映射到代码

并非总是希望为每个项目或变更经历创建依赖图和伪代码的完整过程。在这里，我们将直接从先前的计划过渡到以下代码片段。

# 编写物理模拟器

`src/physics.rs` 中的物理模拟器负责模拟建筑和电梯的物理和布局。模拟器将提供一个对象来处理电机控制，另一个对象来处理数据收集。物理模拟器模块将为每个接口定义特质，电机控制和数据收集对象应分别实现每个 `trait`。

让我们从定义 `physics` 模块的一些类型声明开始。首先，让我们看看一个关键接口——直接电机输入。到目前为止，我们一直假设电机输入将具有简单的电压控制，我们可以将其表示为正或负浮点整数。这种定义是有问题的，主要是在于所有对此类型的引用都将引用 `f64`。此类型指定了一种非常具体的数据表示，没有调整的余地。如果我们用此类型的引用充斥我们的代码，那么任何更改都需要我们回过头来编辑每一个引用。

相反，对于电机输入类型，我们为该类型提供一个名称。这可以是对`f64`类型的别名，这将解决当前的问题。尽管这是可以接受的，但我们将选择更明确地定义类型并提供向上和向下的`enum`情况。`enum`类型，也称为**标签联合体**，用于定义可能具有多种结构或用例的数据。在这里，构造函数是相同的，但每个电压字段的意义是相反的。

此外，当与`MotorInput`类型交互时，我们应该避免假设任何内部结构。这最小化了我们未来接口变化的风险，这些变化可能是因为`MotorInput`定义了一个具有当前未知物理组件的接口。我们将负责与该接口的软件兼容性。因此，为了抽象与`MotorInput`的任何交互，我们将使用特性。那些不定义类型内在行为，而是定义相关行为的特性有时被称为**数据类**。

下面是`enum`和一个定义从输入推导出力的数据类的示例：

```rs
#[derive(Clone,Serialize,Deserialize,Debug)]
pub enum MotorInput
{
   Up { voltage: f64 },
   Down { voltage: f64 }
}

pub trait MotorForce {
   fn calculate_force(&self) -> f64;
}

impl MotorForce for MotorInput {
   fn calculate_force(&self) -> f64
   {
      match *self {
         MotorInput::Up { voltage: v } => { v * 8.0 }
         MotorInput::Down { voltage: v } => { v * -8.0 }
      }
   }
}

pub trait MotorVoltage {
   fn voltage(&self) -> f64;
}

impl MotorVoltage for MotorInput {
   fn voltage(&self) -> f64
   {
      match *self {
         MotorInput::Up { voltage: v } => { v }
         MotorInput::Down { voltage: v } => { -v }
      }
   }
} 
```

接下来，让我们定义电梯信息。我们将创建一个`ElevatorSpecification`，它描述了建筑和电梯的结构。我们还需要一个`ElevatorState`来保存有关当前电梯状态的信息。为了明确楼层请求的使用，我们还将为`FloorRequests`向量创建一个别名，使其意义明确。在这里，我们选择使用`struct`而不是元组来创建明确的字段名称。否则，`struct`和元组可以互换用于存储杂项数据。定义如下：

```rs
#[derive(Clone,Serialize,Deserialize,Debug)]
pub struct ElevatorSpecification
{
   pub floor_count: u64,
   pub floor_height: f64,
   pub carriage_weight: f64
}

#[derive(Clone,Serialize,Deserialize,Debug)]
pub struct ElevatorState
{
   pub timestamp: f64,
   pub location: f64,
   pub velocity: f64,
   pub acceleration: f64,
   pub motor_input: MotorInput
}

pub type FloorRequests = Vec<u64>;
```

`MotorController`和`DataRecorder`的特性几乎相同。唯一的区别是轮询`MotorController`期望返回一个`MotorInput`。在这里，我们选择使用`init`方法而不是构造函数来允许对每个资源的额外外部初始化。例如，`DataRecorder`可能需要在模拟期间打开文件或其他资源。以下是`trait`定义：

```rs
pub trait MotorController
{
   fn init(&mut self, esp: ElevatorSpecification, est: ElevatorState);
   fn poll(&mut self, est: ElevatorState, dst: u64) -> MotorInput;
}

pub trait DataRecorder
{
   fn init(&mut self, esp: ElevatorSpecification, est: ElevatorState);
   fn poll(&mut self, est: ElevatorState, dst: u64);
   fn summary(&mut self);
}
```

为了模拟电梯的物理特性，我们将从第二章的模拟中心循环中复制，*功能控制流*。一些状态已经被组织成结构而不是松散的变量。电机控制决策已经委托给`MotorController`对象。输出和数据记录已经委托给`DataRecorder`。还有一个新的参数字段来指定电梯的载重。有了所有这些概括，代码如下：

```rs
pub fn simulate_elevator<MC: MotorController, DR: DataRecorder>(esp: ElevatorSpecification, est: ElevatorState, req: FloorRequests,
                         mc: &mut MC, dr: &mut DR) {

   //immutable input becomes mutable local state
   let mut esp = esp.clone();
   let mut est = est.clone();
   let mut req = req.clone();

   //initialize MotorController and DataController
   mc.init(esp.clone(), est.clone());
   dr.init(esp.clone(), est.clone());

   //5\. Loop while there are remaining floor requests
   let original_ts = Instant::now();
   thread::sleep(time::Duration::from_millis(1));
   while req.len() > 0
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
         let F = est.motor_input.calculate_force();
         let m = esp.carriage_weight;
         -9.8 + F/m
      };
```

在声明状态和计算时间相关变量之后，我们添加电梯控制逻辑：

```rs
      //5.2\. If next floor request in queue is satisfied, 
          then remove from queue
      let next_floor = req[0];
      if (est.location - (next_floor as f64)*esp.floor_height).abs() 
         < 0.01 &&
         est.velocity.abs() < 0.01
      {
         est.velocity = 0.0;
         req.remove(0);
        //remove is an O(n) operation
        //Vec should not be used like this for large data
      }

      //5.4\. Print realtime statistics
      dr.poll(est.clone(), next_floor);

      //5.3\. Adjust motor control to process next floor request
      est.motor_input = mc.poll(est.clone(), next_floor);

      thread::sleep(time::Duration::from_millis(1));
   }
}
```

# 编写电机控制器

`src/motor.rs`中的电机控制器将负责决定从电机产生多少力。物理驱动器将提供有关所有已知测量值（如位置、速度等）的当前状态信息。目前，电机控制器仅使用最新信息做出控制决策。然而，这可能在将来发生变化，在这种情况下，控制器可能会存储过去的测量值。

从上一章提取相同的控制算法，新的`MotorController`定义如下：

```rs
pub struct SimpleMotorController
{
   pub esp: ElevatorSpecification
}

impl MotorController for SimpleMotorController
{
   fn init(&mut self, esp: ElevatorSpecification, est: ElevatorState)
   {
      self.esp = esp;
   }

   fn poll(&mut self, est: ElevatorState, dst: u64) -> MotorInput
   {
      //5.3\. Adjust motor control to process next floor request

      //it will take t seconds to decelerate from velocity v 
        at -1 m/s²
      let t = est.velocity.abs() / 1.0;

      //during which time, the carriage will travel d=t * v/2 meters
      //at an average velocity of v/2 before stopping
      let d = t * (est.velocity/2.0);

      //l = distance to next floor
      let l = (est.location - (dst as 
          f64)*self.esp.floor_height).abs();     
```

在确定基本常数和值之后，我们需要确定目标加速度：

```rs
      let target_acceleration = {
         //are we going up?
         let going_up = est.location < (dst as 
            f64)*self.esp.floor_height;

         //Do not exceed maximum velocity
         if est.velocity.abs() >= 5.0 {
            if going_up==(est.velocity>0.0) {
               0.0
            //decelerate if going in wrong direction
            } else if going_up {
               1.0
            } else {
               -1.0
            }

         //if within comfortable deceleration range and moving 
             in right direction, decelerate
         } else if l < d && going_up==(est.velocity>0.0) {
            if going_up {
               -1.0
            } else {
               1.0
            }

         //else if not at peak velocity, accelerate
         } else {
            if going_up {
               1.0
            } else {
               -1.0
            }
         }
      };      
```

确定目标加速度后，应将其转换为`MotorInput`值：

```rs
      let gravity_adjusted_acceleration = target_acceleration + 9.8;
      let target_force = gravity_adjusted_acceleration * 
             self.esp.carriage_weight;
      let target_voltage = target_force / 8.0;
      if target_voltage > 0.0 {
         MotorInput::Up { voltage: target_voltage }
      } else {
         MotorInput::Down { voltage: target_voltage.abs() }
      }
   }
}
```

现在，让我们编写第二个控制器，实现所提出的改进。我们将在模拟的后期比较这两个控制器。第一个建议是减少轮询间隔。此更改必须在物理模拟器中完成，因此我们将测量其效果，但不会将其与电机控制器绑定。第二个建议是平滑加速度曲线。

经过考虑，我们意识到加速度（也称为**jerk**）的变化是让人感到不适的原因，比小的加速度力更甚。理解这一点后，只要加速度变化率保持小，我们将允许更快的加速度。我们将用以下约束和目标替换当前的目标加速度计算：

+   最大加速度变化率 = `0.2` m/s³

+   最大加速度 = `2.0` m/s²

+   最大速度 = `5.0` m/s

+   目标加速度变化：

    +   如果加速，则输出 0.2

    +   如果减速，则输出-0.2

    +   如果处于稳定速度，则输出 0.0

结果控制器如下所示：

```rs
const MAX_JERK: f64 = 0.2;
const MAX_ACCELERATION: f64 = 2.0;
const MAX_VELOCITY: f64 = 5.0;

pub struct SmoothMotorController
{
   pub esp: ElevatorSpecification,
   pub timestamp: f64
}

impl MotorController for SmoothMotorController
{
   fn init(&mut self, esp: ElevatorSpecification, est: ElevatorState)
   {
      self.esp = esp;
      self.timestamp = est.timestamp;
   }

   fn poll(&mut self, est: ElevatorState, dst: u64) -> MotorInput
   {
      //5.3\. Adjust motor control to process next floor request

      //it will take t seconds to reach max from max
      let t_accel = MAX_ACCELERATION / MAX_JERK;
      let t_veloc = MAX_VELOCITY / MAX_ACCELERATION;

      //it may take up to d meters to decelerate from current
      let decel_t = if (est.velocity>0.0) == (est.acceleration>0.0) {
         //this case deliberately overestimates d to prevent "back up"
         (est.acceleration.abs() / MAX_JERK) +
         (est.velocity.abs() / (MAX_ACCELERATION / 2.0)) +
         2.0 * (MAX_ACCELERATION / MAX_JERK)
      } else {
         //without the MAX_JERK, this approaches infinity and 
            decelerates way too soon
         //MAX_JERK * 1s = acceleration in m/s²
         est.velocity.abs() / (MAX_JERK + est.acceleration.abs())
      };
      let d = est.velocity.abs() * decel_t;

      //l = distance to next floor
      let l = (est.location - (dst as 
              f64)*self.esp.floor_height).abs();
```

确定基本常数和值后，我们可以计算目标加速度：

```rs
      let target_acceleration = {
         //are we going up?
         let going_up = est.location < (dst as 
             f64)*self.esp.floor_height;

         //time elapsed since last poll
         let dt = est.timestamp - self.timestamp;
         self.timestamp = est.timestamp;

         //Do not exceed maximum acceleration
         if est.acceleration.abs() >= MAX_ACCELERATION {
            if est.acceleration > 0.0 {
               est.acceleration - (dt * MAX_JERK)
            } else {
               est.acceleration + (dt * MAX_JERK)
            }

         //Do not exceed maximum velocity
         } else if est.velocity.abs() >= MAX_VELOCITY
            || (est.velocity + est.acceleration * 
               (est.acceleration.abs() / MAX_JERK)).abs() >= 
                          MAX_VELOCITY {
            if est.velocity > 0.0 {
               est.acceleration - (dt * MAX_JERK)
            } else {
               est.acceleration + (dt * MAX_JERK)
            }

         //if within comfortable deceleration range and 
             moving in right direction, decelerate
         } else if l < d && (est.velocity>0.0) == going_up {
            if going_up {
               est.acceleration - (dt * MAX_JERK)
            } else {
               est.acceleration + (dt * MAX_JERK)
            }

         //else if not at peak velocity, accelerate smoothly
         } else {
            if going_up {
               est.acceleration + (dt * MAX_JERK)
            } else {
               est.acceleration - (dt * MAX_JERK)
            }
         }
      };
```

确定目标加速度后，我们应该计算目标力：

```rs
      let gravity_adjusted_acceleration = target_acceleration + 9.8;
      let target_force = gravity_adjusted_acceleration
            * self.esp.carriage_weight;
      let target_voltage = target_force / 8.0;
      if !target_voltage.is_finite() {
         //divide by zero etc.
         //may happen if time delta underflows
         MotorInput::Up { voltage: 0.0 }
      } else if target_voltage > 0.0 {
         MotorInput::Up { voltage: target_voltage }
      } else {
         MotorInput::Down { voltage: target_voltage.abs() }
      }
   }
}
```

# 编写运行模拟的可执行文件

运行模拟的可执行文件，包含在`src/lib.rs`中，由上一章模拟的所有输入和配置组成。以下是配置和运行模拟所使用的工具：

```rs
pub fn run_simulation()
{

   //1\. Store location, velocity, and acceleration state
   //2\. Store motor input voltage
   let mut est = ElevatorState {
      timestamp: 0.0,
      location: 0.0,
      velocity: 0.0,
      acceleration: 0.0,
      motor_input: MotorInput::Up {
         //a positive force is required to counter gravity and
         voltage: 9.8 * (120000.0 / 8.0)
      }
   };

   //3\. Store input building description and floor requests
   let mut esp = ElevatorSpecification {
      floor_count: 0,
      floor_height: 0.0,
      carriage_weight: 120000.0
   };
   let mut floor_requests = Vec::new();

   //4\. Parse input and store as building description
           and floor requests
   let buffer = match env::args().nth(1) {
      Some(ref fp) if *fp == "-".to_string()  => {
         let mut buffer = String::new();
         io::stdin().read_to_string(&mut buffer)
                    .expect("read_to_string failed");
         buffer
      },
      None => {
         let fp = "test1.txt";
         let mut buffer = String::new();
         File::open(fp)
              .expect("File::open failed")
              .read_to_string(&mut buffer)
              .expect("read_to_string failed");
         buffer
      },
      Some(fp) => {
         let mut buffer = String::new();
         File::open(fp)
              .expect("File::open failed")
              .read_to_string(&mut buffer)
              .expect("read_to_string failed");
         buffer
      }
   };
   for (li,l) in buffer.lines().enumerate() {
      if li==0 {
         esp.floor_count = l.parse::<u64>().unwrap();
      } else if li==1 {
         esp.floor_height = l.parse::<f64>().unwrap();
      } else {
         floor_requests.push(l.parse::<u64>().unwrap());
      }
   }  
```

在建立模拟状态并读取输入配置后，我们运行模拟：

```rs
   let termsize = termion::terminal_size().ok();
   let mut dr = SimpleDataRecorder {
      esp: esp.clone(),
      termwidth: termsize.map(|(w,_)| w-2).expect("termwidth") 
         as u64,
      termheight: termsize.map(|(_,h)| h-2).expect("termheight")
         as u64,
      stdout: &mut io::stdout().into_raw_mode().unwrap(),
      log: File::create("simulation.log").expect("log file"),
      record_location: Vec::new(),
      record_velocity: Vec::new(),
      record_acceleration: Vec::new(),
      record_voltage: Vec::new()
   };
   /*
   let mut mc = SimpleMotorController {
      esp: esp.clone()
   };
   */
   let mut mc = SmoothMotorController {
      timestamp: 0.0,
      esp: esp.clone()
   };

   simulate_elevator(esp, est, floor_requests, &mut mc, &mut dr);
   dr.summary();

}
```

`DataRecorder`实现，同样在`src/lib.rs`中，负责输出实时信息以及汇总信息。此外，我们还将序列化和存储模拟数据到日志文件中。注意使用`lifetime`参数以及参数化的`trait`：

```rs
struct SimpleDataRecorder<'a, W: 'a + Write>
{
   esp: ElevatorSpecification,
   termwidth: u64,
   termheight: u64,
   stdout: &'a mut raw::RawTerminal<W>,
   log: File,
   record_location: Vec<f64>,
   record_velocity: Vec<f64>,
   record_acceleration: Vec<f64>,
   record_voltage: Vec<f64>,
}
impl<'a, W: Write> DataRecorder for SimpleDataRecorder<'a, W>
{
   fn init(&mut self, esp: ElevatorSpecification, est: ElevatorState)
   {
      self.esp = esp.clone();
      self.log.write_all(serde_json::to_string(&esp).unwrap().as_bytes()).expect("write spec to log");
      self.log.write_all(b"\r\n").expect("write spec to log");
   }
   fn poll(&mut self, est: ElevatorState, dst: u64)
   {
      let datum = (est.clone(), dst);
      self.log.write_all(serde_json::to_string(&datum).unwrap().as_bytes()).expect("write state to log");
      self.log.write_all(b"\r\n").expect("write state to log");

      self.record_location.push(est.location);
      self.record_velocity.push(est.velocity);
      self.record_acceleration.push(est.acceleration);
      self.record_voltage.push(est.motor_input.voltage());
```

`DataRecorder`不仅负责将模拟数据记录到日志中，还负责将统计数据打印到终端：

```rs
      //5.4\. Print realtime statistics
      print!("{}{}{}", clear::All, cursor::Goto(1, 1), cursor::Hide);
      let carriage_floor = (est.location / self.esp.floor_height).floor();
      let carriage_floor = if carriage_floor < 1.0 { 0 } else { carriage_floor as u64 };
      let carriage_floor = cmp::min(carriage_floor, self.esp.floor_count-1);
      let mut terminal_buffer = vec![' ' as u8; (self.termwidth*self.termheight) as usize];
      for ty in 0..self.esp.floor_count
      {
         terminal_buffer[ (ty*self.termwidth + 0) as usize ] = '[' as u8;
         terminal_buffer[ (ty*self.termwidth + 1) as usize ] =
            if (ty as u64)==((self.esp.floor_count-1)-carriage_floor) { 'X' as u8 }
            else { ' ' as u8 };
         terminal_buffer[ (ty*self.termwidth + 2) as usize ] = ']' as u8;
         terminal_buffer[ (ty*self.termwidth + self.termwidth-2) as usize ] = '\r' as u8;
         terminal_buffer[ (ty*self.termwidth + self.termwidth-1) as usize ] = '\n' as u8;
      }
      let stats = vec![
         format!("Carriage at floor {}", carriage_floor+1),
         format!("Location {:.06}", est.location),
         format!("Velocity {:.06}", est.velocity),
         format!("Acceleration {:.06}", est.acceleration),
         format!("Voltage [up-down] {:.06}", est.motor_input.voltage()),
      ];
      for sy in 0..stats.len()
      {
         for (sx,sc) in stats[sy].chars().enumerate()
         {
            terminal_buffer[ sy*(self.termwidth as usize) + 6 + sx ] = sc as u8;
         }
      }
      write!(self.stdout, "{}",
```

```rs
String::from_utf8(terminal_buffer).ok().unwrap());
      self.stdout.flush().unwrap();
   }
```

`DataRecorder`还负责在模拟结束时打印汇总信息：

```rs
   fn summary(&mut self)
   {
      //6 Calculate and print summary statistics
      write!(self.stdout, "{}{}{}", clear::All, cursor::Goto(1, 1), cursor::Show).unwrap();
      variable_summary(&mut self.stdout, "location".to_string(), &self.record_location);
      variable_summary(&mut self.stdout, "velocity".to_string(), &self.record_velocity);
      variable_summary(&mut self.stdout, "acceleration".to_string(), &self.record_acceleration);
      variable_summary(&mut self.stdout, "voltage".to_string(), &self.record_voltage);
      self.stdout.flush().unwrap();
   }
}
```

# 编写可执行文件以分析模拟

`src/analyze.rs`中的分析可执行文件应查看日志文件并确认所有要求都已满足——即以下内容：

+   加速度变化率（也称为**jerk**）小于`0.2` m/s³

+   加速度低于 `2.0` m/s²

+   速度低于 `5.0` m/s

+   电梯在行程中不会倒退

+   所有行程都在物理理论极限的 20% 以内完成

程序设计将是遍历日志文件并检查所有值是否在指定的限制内。还需要一个方向标志来提醒我们备份事件。当行程完成后，我们将比较经过的时间与理论极限。如果任何要求未满足，我们将立即失败并打印一些基本信息。代码如下：

```rs
#[derive(Clone)]
struct Trip {
   dst: u64,
   up: f64,
   down: f64
}

const MAX_JERK: f64 = 0.2;
const MAX_ACCELERATION: f64 = 2.0;
const MAX_VELOCITY: f64 = 5.0;

fn main()
{
   let simlog = File::open("simulation.log").expect("read simulation log");
   let mut simlog = BufReader::new(&simlog);
   let mut jerk = 0.0;
   let mut prev_est: Option<ElevatorState> = None;
   let mut dst_timing: Vec<Trip> = Vec::new();
   let mut start_location = 0.0;
```

初始化分析状态后，我们将遍历日志中的行以计算统计数据：

```rs
   let mut first_line = String::new();
   let len = simlog.read_line(&mut first_line).unwrap();
   let esp: ElevatorSpecification = serde_json::from_str(&first_line).unwrap();

   for line in simlog.lines() {
      let l = line.unwrap();
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
   }
```

分析在处理文件时验证一些要求；其他要求必须在处理完整个日志后才能验证：

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

   //trips should finish within 20% of theoretical limit
   let mut trip_start_location = start_location;
   let mut theoretical_time = 0.0;
   let floor_height = esp.floor_height;
   for trip in dst_timing.clone()
   {
      let next_floor = (trip.dst as f64) * floor_height;
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

   println!("All simulation checks passing.");
}
```

# 运行模拟和分析数据

运行 `SimpleMotorController` 模拟后，我们收集初始模拟日志。由于方便的 SerDe 库，模拟日志将以 JSON 格式保存。对于模拟器的每次迭代，都应该有一个初始电梯规范，然后是一个电梯状态。`simulation.log` 最终看起来可能如下所示：

```rs
{"floor_count":5,"floor_height":5.67,"carriage_weight":120000.0}[{"timestamp":0.001288587,"location":0.0,"velocity":0.0,"acceleration":0.0,"motor_input":{"Up":{"voltage":147000.0}}},2][{"timestamp":0.002877568,"location":0.0,"velocity":0.0,"acceleration":0.0002577174000002458,"motor_input":{"Up":{"voltage":147003.86576100003}}},2][{"timestamp":0.004389254,"location":0.0,"velocity":3.8958778553677168e-7,"acceleration":0.000575513599999411,"motor_input":{"Up":{"voltage":147008.632704}}},2][{"timestamp":0.005886777,"location":5.834166693603828e-10,"velocity":0.0000012514326383486894,"acceleration":0.0008778508000002461,"motor_input":{"Up":{"voltage":147013.16776200004}}},2][{"timestamp":0.007377939,"location":2.449505465225691e-9,"velocity":0.0000025604503929786564,"acceleration":0.0011773553999994136,"motor_input":{"Up":{"voltage":147017.660331}}},2][{"timestamp":0.008929299,"location":6.421685786877059e-9,"velocity":0.000004386952466321746,"acceleration":0.0014755878000016765,"motor_input":{"Up":{"voltage":147022.13381700003}}},2]
```

此序列化输出是由我们的 SerDe 序列化库创建的。使用 SerDe 实现序列化的步骤有几个，这对了解复杂库的工作方式非常有信息量。为了使用 SerDe 进行 JSON 序列化和反序列化，我们必须执行以下操作：

1.  按照以下方式将 SerDe 添加到 `Cargo.toml` 依赖项中：

```rs
[dependencies]
serde = "1.0"
serde_json = "1.0"
serde_derive = "1.0"
```

1.  将 `macro_use` 指令和 `extern crate` 导入添加到项目根目录：

```rs
#[macro_use] extern crate serde_derive;
extern crate serde;
extern crate serde_json;
```

1.  为将要序列化的数据推导 `Serialize` 和 `Deserialize` 特性。为了在声明上进行宏操作以推导特性，使用 `derive` 指令。对于指令中的每个宏，都期望有一个相应的过程宏。考虑以下代码：

```rs
#[derive(Clone,Serialize,Deserialize,Debug)]
pub enum MotorInput
{
   Up { voltage: f64 },
   Down { voltage: f64 }
}

#[derive(Clone,Serialize,Deserialize,Debug)]
pub struct ElevatorSpecification
{
   pub floor_count: u64,
   pub floor_height: f64,
   pub carriage_weight: f64
}

#[derive(Clone,Serialize,Deserialize,Debug)]
pub struct ElevatorState
{
   pub timestamp: f64,
   pub location: f64,
   pub velocity: f64,

```

```rs
   pub acceleration: f64,
   pub motor_input: MotorInput
}
```

1.  根据需要序列化数据。在 `lib.rs` 中，我们序列化 `ElevatorSpecification` 和 `ElevatorState` 结构体。类型提示通常是必要的，因为类型系统不喜欢猜测：

```rs
serde_json::to_string(&datum).unwrap().as_bytes()
```

1.  根据需要反序列化数据。在 `analyze.rs` 中，我们将行反序列化到 `ElevatorSpecification` 和 `ElevatorState` 结构体中。类型提示通常是必要的，因为类型系统不喜欢猜测：

```rs
serde_json::from_str(&l).unwrap()
```

SerDe 支持许多内置类型以进行序列化和反序列化。这些大致对应于 JSON 允许的所有类型，通过类型提示允许额外的结构体。

查看 `simulation.log`，我们可以找到大多数内置类型：

+   **整数类型**：整数类型变为直接的 JSON 整数：

```rs
5
```

+   **浮点类型**：浮点整数变为直接的 JSON 浮点数：

```rs
6.54321
```

+   **字符串**：Rust 字符串也被直接翻译成 JSON 等效项：

```rs
"timestamp"
```

+   **向量和数组**：Rust 集合有时以意想不到的方式序列化。大多数情况下，向量类型被直接翻译成 JSON 数组；包含向量包含的序列化版本：

```rs
[1,2,3,4,5,6,0]
```

+   **元组**：元组被序列化为 JSON 数组，然而，编译器通常需要一个类型提示来理解如何序列化和反序列化这些类型：

```rs
[{"timestamp":0.007377939,"location":2.449505465225691e-9,"velocity":0.0000025604503929786564,"acceleration":0.0011773553999994136,"motor_input":{"Up":{"voltage":147017.660331}}},2][{"timestamp":0.008929299,"location":6.421685786877059e-9,"velocity":0.000004386952466321746,"acceleration":0.0014755878000016765,"motor_input":{"Up":{"voltage":147022.13381700003}}},2]
```

+   **结构体**：Rust 结构体直接转换为 JSON 对象。这总是成功的，因为 Rust 字段名是有效的对象键，如下所示：

```rs
{"floor_count":5,"floor_height":5.67,"carriage_weight":120000.0}
```

+   **标签联合**：标签联合是一个稍微有些奇怪的情况。`union` 构造函数被转换成与任何其他结构体一样的 JSON 对象。然而，`union` 标签也被赋予了自己的结构体，将 `union` 构造函数包裹在一个单独的对象中。类型提示在这里对于编译器正确地序列化和反序列化是非常必要的：

```rs
{"Up":{"voltage":147003.86576100003}}
```

+   **HashMap**：Rust 的 HashMap 在序列化方面是一个特殊案例。库尝试将它们转换为 JSON 对象。然而，并非所有 HashMap 键都可以序列化。因此，某些序列化可能会失败，需要自定义序列化器：

```rs
{"a":5,"b":6,"c":7}
```

一些类型难以序列化，包括时间结构，如 `Instant`。尽管处理某些数据类型存在困难，但 SerDe 库在存储和加载数据时非常稳定、快速且不可或缺。

运行分析程序，我们可以确认这个电机控制器不足以满足当前项目要求：

```rs
jerk is outside of acceptable limits: ElevatorState {
   timestamp: 0.023739637,
   location: 0,
   velocity: 0,
   acceleration: 1,
   motor_input: Up { voltage: 162000 }
}
```

切换到 `SmoothMotorController`，我们可以看到所有规格都符合要求：

```rs
All simulation checks passing.
```

# 摘要

在本章中，我们概述了处理项目范围变更和新规格的步骤。我们专注于如何编写健壮的代码，这将鼓励在未来的额外项目或改进中重用。

使用各种数据结构有助于组织我们的项目和数据。代码应尽可能实现自文档化。此外，类型安全的代码可以强制一些关于代码的假设，以阻止错误的输入和不适当的用法。通过使用数据类，我们还学会了如何扩展现有的数据结构以支持新的用途。我们还使用数据类作为接口来推迟对项目元素不确定性的假设。

在下一章中，我们将学习参数化和泛型。我们将进行深入的代码审查和案例分析。

# 问题

1.  哪个库适合序列化和反序列化数据？

1.  在 `physics.rs` 中结构声明前的哈希标签派生行有什么作用？

1.  在参数化声明中，先声明生命周期还是特质？

1.  在 `trait` 实现中，`impl`、`trait` 或类型上的参数有什么区别？

1.  `trait` 和数据类之间有什么区别？

1.  你应该如何声明一个包有多个可执行文件？

1.  你如何声明一个结构体字段为私有？
