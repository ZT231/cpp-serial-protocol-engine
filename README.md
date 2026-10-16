# C++ 跨平台串口协议引擎（serial_engine）

> 一句话定位：把机械臂固件里那套用 C 手搓的 `uart_protocol`（双缓冲 / 状态机收帧 / 持锁解析）搬到 PC 端，
> 用现代 C++17 重新实现一遍，做成面向六自由度机械臂的 PC 端上位机（真机联调待硬件就位，见「阶段进度」）。

## 项目定位

- 三个项目的**完整故事链**：STM32 固件（下位机）→ 通信协议（本项目的骨架）→ PC 上位机（本项目的形态）。
- 可深入讨论的三个技术点：**粘包/半包怎么处理**、**应答帧被覆盖怎么办**、**Mock 串口为什么有价值** —— 每层都有真实排障素材（详见《排障记录.md》）。
- 技术要点一句话版：
  《基于 C++17 实现跨平台串口协议引擎：自研 RingBuffer 与状态机帧解析器解决粘包、半包与噪声干扰；
   抽象 ISerialPort 接口支持 Windows/Linux 串口与 Mock 测试；命令分派采用回调注册机制；
   协议与固件 `uart_protocol.c` 逐条对齐（异或校验、命令表、接收/发送缓冲上限）；真机联调待硬件就位。》

## 项目结构

```
3.C++ 跨平台串口协议引擎/
├── README.md              ← 本文件；模块设计说明见文末「模块设计说明（三行法）」
├── 排障记录.md            ← 踩坑记录（现象→排查→根因→解决）
├── 项目使用说明.md        ← 编译方式、armctl 命令详解、作为库使用的代码示例
├── CMakeLists.txt         ← 用 Qt Creator / CLion 直接打开这个文件
├── build.bat              ← 命令行一键编译 + 跑全部测试（双击即可）
├── build_win32.sh         ← 用 MinGW-w64 交叉编译真机串口测试 + armctl（Git Bash / Cygwin 下运行）
├── .gitignore              ← 忽略 build/、*.exe、*.o、CMake/Qt/编辑器产物
├── .gitattributes          ← 行尾规则：仓库内存 LF；.bat 检出 CRLF（cmd 解析批处理必须 CRLF），.sh 保持 LF
├── include/serial/
│   ├── ring_buffer.hpp           ← 模块 1：环形缓冲区              ✅
│   ├── checksum.hpp              ← 模块 2：异或校验 + 错误码         ✅
│   ├── error_code.hpp
│   ├── frame.hpp                 ← 模块 3：一帧的解析/生成           ✅
│   ├── frame_parser.hpp          ← 模块 4：字节流 → 帧 状态机         ✅
│   ├── command_dispatcher.hpp    ← 模块 5：命令分派 + 命令表          ✅
│   ├── commands.hpp
│   ├── iserial_port.hpp          ← 模块 6：串口抽象接口（虚析构）     ✅
│   ├── mock_serial_port.hpp      ← 模块 7：Mock 串口（不插硬件测）    ✅
│   ├── win32_serial_port.hpp     ← 模块 8：Windows 串口（CreateFile） ✅
│   ├── linux_serial_port.hpp     ←         Linux 串口（termios，未在真实 Linux 验证）⚠️
│   ├── serial_port_factory.hpp   ←         平台工厂（按宏选后端）     ✅
│   ├── logger.hpp                ← 模块 9：日志（Meyers 单例 + 线程安全） ✅
│   ├── session.hpp               ← 模块 10：发请求→收应答会话          ✅
│   ├── blocking_queue.hpp        ← 模块 11：线程安全队列（生产者-消费者）✅
│   └── async_receiver.hpp        ← 模块 12：异步接收线程（std::thread）  ✅
├── src/
│   └── main.cpp                  ← 上位机 CLI：armctl（selftest / --mock / 真机）
├── tests/
│   ├── test_ring_buffer.cpp
│   ├── test_frame.cpp
│   ├── test_frame_parser.cpp
│   ├── test_dispatcher.cpp
│   ├── test_mock_serial_port.cpp
│   ├── test_serial_integration.cpp
│   ├── test_serial_port_factory.cpp
│   ├── test_logger.cpp
│   ├── test_session.cpp
│   ├── test_blocking_queue.cpp
│   ├── test_async_receiver.cpp
│   └── test_win32_serial_live.cpp   ← 真机串口回环测试（需真实 COM 口，用 build_win32.sh 单独编译）
└── build/                 ← 编译产物（已在 .gitignore 中忽略）
```

当前测试：**243 处 `CHECK` 断言全部通过，`-Wall -Wextra` 零警告**；`armctl selftest` 自检通过、`--mock` 演示链路通。

> **断言计数口径**：`tests/` 内匹配 `CHECK(` 的共 254 行，其中 11 行是各测试文件自己的 `#define CHECK` 宏定义，254 − 11 = 243 处调用。
> `test_serial_port_factory` 有 3 处位于 `#if defined(_WIN32) || defined(__linux__)` 分支内，与 `#else` 分支互斥，
> 所以单次构建实际执行的断言数：Cygwin/POSIX 工具链 **240** 项，Windows/Linux 工具链 **242** 项。
> `tests/test_win32_serial_live.cpp` 需要真实 COM 口，不参与 CMake/CTest 与 `build.bat` 的自动跑测。
> 历史备注：曾写「247 项」属统计口径错误；`armctl` 也一度因在 `unique_ptr<ISerialPort>` 上调 Mock 独有的 `inject()` 而编译不过，已修复，详见排障记录第 6 条。

## 构建与运行

**方式一：命令行（推荐日常用）**
双击 `build.bat`（它会自动编译并运行 `tests\*.cpp` 里的每一个单元测试；真机测试 `test_win32_serial_live.cpp` 需要真实 COM 口 + MinGW-w64，脚本里已显式跳过），等价于：

```bat
g++ -std=c++17 -Wall -Wextra -Iinclude tests\test_frame.cpp -o build\test_frame.exe
build\test_frame.exe
```

**方式二：Qt Creator**
用 Qt Creator 打开 `CMakeLists.txt` → 选择 MinGW 或 MSVC 套件 → 直接构建运行（测试用例会注册到 CTest）。

> ⚠️ 中文路径注意事项：Git Bash 下凡是把中文路径**当参数写进命令**都会失败
> （报 `No such file or directory`，连 `ls "中文路径目录"` 都不行）。
> 解决办法有两个：① 用 ASCII 通配符进目录，例如 `cd 3.C++*`；
> ② 直接走 `build.bat`（cmd 对中文路径没有问题），或者用支持 Unicode 路径的编辑器打开。

## 阶段进度

| 阶段 | 内容 | 关键技术点 | 状态 |
|---|---|---|---|
| 1 | 协议核心：`RingBuffer<T>`、异或校验、`Frame`、状态机 `FrameParser`、`CommandDispatcher` + 命令表 | 类模板 / 拷贝控制（深拷贝·三法则）/ explicit / 运算符重载 / std::function 回调 | ✅ 已完成 |
| 2 | 跨平台串口层：`ISerialPort` 抽象 + Windows/Linux 实现 + `MockSerialPort` + 平台工厂 | 纯虚接口 / 虚析构 / unique_ptr 独占所有权 / RAII / 工厂 / 平台宏 | ✅ 已完成（Linux 版本机已用 `-D__linux__` 编译检查零警告，未在真实 Linux 环境验证）|
| 3 | 上位机 CLI：`armctl`（selftest / --mock / 真机命令）、`Logger`、`SerialSession`、连真机联调 | 单例 / 文件操作 / string 解析 / 命令行参数 / 超时循环 | ✅ 代码完成（真机联调待硬件就位）|
| 4 | 加分项：接收线程 + 线程安全队列、Logger/Mock 线程安全化 | std::thread / condition_variable / atomic / mutex | ✅ 已完成 |

---

# 协议速查表（v1，与固件逐条对齐）

> 来源：机械臂固件 `Core/Src/uart_protocol.c`（见 `robotic-arm-vision-stm32` 仓库）。**上位机必须与对端一致，
> 所以这里放弃了原计划的 CRC16，改用固件真正在用的异或校验。**

```
帧格式：  $<CMD>[,<ARG>...]*<CK>\r\n
  CK = CMD 字段（'$' 与 '*' 之间）所有字节逐字节 XOR，2 位大写十六进制
  例：$PING*10\r\n        （'P'^'I'^'N'^'G' = 0x10）

响应：   $OK[,<数据>]*XX\r\n
        $ERR,<code>*XX\r\n
错误码： 1=未知命令  2=校验失败  3=格式/参数错误  4=当前状态禁止

命令表：
  PING                      → OK,PONG
  STATE                     → OK,S,<M|C|R|P|E>,<6 路脉宽>   状态：手动/命令/录制/回放/急停
  MOVE,<ch>,<pulse>         → OK | ERR,3(ch/pulse 非法) | ERR,4(急停)
  JOINT,<6 路脉宽>          → OK | ERR,3 | ERR,4
  HOME / STOP / RESUME / MANUAL → OK | ERR,4
  REC / RECEND / PLAY       → OK（RECEND 返回 OK,<点数>）| ERR,3 | ERR,4
  TNUM                      → OK,<点数>
  CLEAR                     → OK | ERR,4
  LOG,<0|1>                 → OK | ERR,3
固件缓冲上限：接收 64 字节 / 发送 96 字节 → 本引擎 `kMaxContent = 63`
```

**两个已确认的设计决策（对应上面的选择）**
1. **解析从严**：CK 之后还有多余字符、CK 不是十六进制、缺 `*CK`，一律记 `format_errors` 并丢帧。
   固件对这个场景更宽容（只认第一个 `*`，后面的字符不管）；上位机是调试工具，宁可直接暴露问题。
2. **解析器只切字段，不判业务数值**：`FrameParser` 只负责把 `JOINT,1500,...` 切成命令+参数，
   关节个数够不够、脉宽是否 ≤2500 交给命令层判断（对应固件 `ERR,3`）。分层清楚了，加命令不用动解析器。

---

# 模块设计说明（三行法）

> **规矩：每写完一个模块，立刻回答这三行。** 用大白话写，写不出来说明没真懂。
> 每行分别是：① 用了什么语法 ② 为什么这么设计 ③ 不这么写会出什么问题。

## 1. `RingBuffer<T>` — `include/serial/ring_buffer.hpp`

**① 用了什么语法**
类模板 `<typename T>` + 三法则（拷贝构造 / 拷贝赋值 / 析构，全部做**深拷贝**）、`explicit` 构造函数、
`new[] delete[]` 手动内存管理、`std::size_t` 无符号下标、`% capacity_` 取模实现回绕、
`operator[]` 与 `operator<<` 运算符重载、`std::out_of_range` 异常（下标越界时抛）。

**② 为什么这么设计**
- **环形**：串口数据是流式的，用 `head_`/`tail_` 循环推进，读写都是 O(1)，不需要像 `vector` 那样搬移元素。
- **额外维护 `size_`**：只有 `head_==tail_` 时，"空"和"满"无法区分，用计数最省心（另一种方案是故意空一格）。
- **`pop` 返回 `bool`、`front` 抛异常**：暂时没数据是**正常流程**，用返回值；下标越界是**逻辑错误**，用异常，让调用方没法忽略。
- **`operator=` 先判自赋值再深拷贝**：自己先 `if (this == &another)` 直接返回（否则 `a=a` 会把自己 delete 了再读，崩）；
  然后 `delete[]` 旧数组、`new[]` 新数组、把元素一个个复制过去。不玩 copy-and-swap 花活，新手一眼能看明白。

**③ 不这么写会出什么问题**
- 不写拷贝构造 → 编译器生成**浅拷贝**，两个对象指向同一块内存，析构时 **double free 直接崩**。
- 拷贝赋值不先 `delete[]` 旧数组 → 旧内存泄漏；不判自赋值 → `a = a` 时把自己 delete 了再读就崩。
- 析构不写 `delete[]`（写成 `delete`）→ 数组元素只析构第一个，**内存泄漏 + UB**。
- `head_`/`tail_` 回绕时不取模 → 下标越界，读到别的内存（嵌入式里表现为"偶尔收到乱码"，极难查）。

## 2. 异或校验 + 错误码 — `checksum.hpp` / `error_code.hpp`

**① 用了什么语法**
命名空间 `serial`、`using Checksum = std::uint8_t` 类型别名、`inline` 头文件函数、
`static_cast<unsigned char>` 消除 `char` 符号性差异、`enum class ErrorCode : std::uint8_t`（强类型枚举 + 指定底层类型）、
`switch` + 返回 `const char*`、函数内 `static const char[]` 常量表。

**② 为什么这么设计**
- **校验方式跟着对端走**：固件实现的是异或（`proto_send_frame` 里 `ck ^= payload[i]`），PC 端必须一致，
  所以哪怕 CRC16 更强也不能用——**协议两端不一致，再强的校验都是零**。
- **`enum class` 而不是裸 `#define`**：不会隐式转成 `int`，不会被别的枚举/宏撞名，也能指定底层类型 `uint8_t` 好做线上下发。
- **`static_cast<unsigned char>`**：`char` 是否有符号是实现定义的，直接 `ck ^= ch` 在负值 `char` 上会得到实现相关的行为。

**③ 不这么写会出什么问题**
- 用 CRC16 而固件用异或 → 每条帧都被对端判成 `ERR,2`，链路根本不通。
- `ck ^= ch` 不做无符号转换 → 遇到 `0x80` 以上的字节（中文、UTF-8、二进制数据）结果错，现象是"大部分帧能过，偶尔校验失败"。
- 校验和输出用小写十六进制 → 固件 `hex_to_nib` 大小写都能收，但某些手机端工具只认大写，容易出现"你能收我不能发"。
- 用 `enum` 而非 `enum class` → 隐式转换成整数，`switch` 漏 case 不报警，错误码串味（这是真实事故源）。

## 3. `Frame` — `include/serial/frame.hpp`

**① 用了什么语法**
类封装（`std::string payload_` 私有成员）、**`bool` + 出参**表达"解析/取值是否成功"（`argUint`/`parse` 用 bool+out，队列取值见模块 11 用 `std::optional`）、
`std::string::find/substr` 切字段、**手写十进制解析 + 自写溢出检查**（不用 `stoul` 也不用 `std::numeric_limits`）、
`friend bool operator==/!=`、静态成员函数 `bool parse(text, out)` 做解析、默认构造 + `explicit` 单参构造。

**② 为什么这么设计**
- **`bool` + 出参而不是抛异常**：对端发来的数据脏属于**正常分支**，返回 `false` 让调用方显式处理；
  只有"我方代码逻辑错"（如容器越界）才用异常（和 `RingBuffer::front` 的分工一致）。
- **`argUint`/`parse` 用 bool+out 而不是 `std::optional`**：它们表达的是"解析/取值成不成功"（一个动作的结果），
  bool 最直白；而"有没有值"的语义（队列取帧，见模块 11 的 `tryPop`）才用 `std::optional`。两种语义、两种工具，各司其职。
- **`Frame` 只管切字段**：不判断"JOINT 是不是正好 6 个"——那是命令层的活儿，对应固件 `ERR,3`。
  这样加新命令不用改 `Frame`，也不会让解析层依赖业务规则。
- **`argUint` 带溢出检查**：`99999999999` 这种脏数据不能被悄悄截断成一个看起来正常的脉宽值，
  否则会把机械臂打到错误位置——**在机器人场景，脏数据比报错危险得多**。
- **`parse` 从严**：`$` 开头、`*`+2 位十六进制结尾、CK 后无多余字符、校验必须过，四条全满足才返回帧。

**③ 不这么写会出什么问题**
- 用 `stoul`/`atoi` 直接转数字 → `atoi("abc")` 返回 0 不报错，`"-1"` 变成巨大的无符号数，
  "非法参数"会被当成"合法参数 0"发下去（真实事故：舵机突然跳到极限位置）。
- 不做溢出检查 → `uint32_t` 溢出回绕，`99999999999` 被截成一个 8 位数脉宽。
- `parse` 不校验 CK → 噪声里偶然出现的 `$...*xx` 会被当成真命令执行。
- 不做 `explicit` → `Frame f = "PING";` 这种隐式转换能编译通过，代码意图变得模糊。

## 4. `FrameParser` — `include/serial/frame_parser.hpp`

**① 用了什么语法**
状态机（`active_` 标志 + `buf_` 缓冲）、类内 `static const std::size_t` 常量、
嵌套结构体 `Stats` 做统计、两个 `feed` 重载：
  - 单字节版 `feed(byte, Frame& out)` 返回 `bool`（凑成合法帧则 `out` 里有帧、返回 true）；
  - 整串版 `feed(string)` 返回 `vector<Frame>`（粘包一次出多帧）；
`isHexDigit` 复用。
（对应固件的 `UartProto_HandleByte`，但把全局变量 + 裸缓冲换成了对象状态。）
解析层帧直接拷贝进出（清晰优先、对控制链路够用）；需要"搬对象省拷贝"的地方（队列 push/pop，见模块 11）才用 `std::move`。

**② 为什么这么设计**
- **按字节喂**：串口的本质是字节流，`$` 之前的一切都是噪声。所以用"看到 `$` 就抛弃之前全部"的模型，
  天然免疫 AT 回显、上电毛刺、乱码。
- **收到 `*CK` 立即成帧**：不强制等 `\r\n`，兼容不发回车换行的上位机工具（固件也是这个策略）。
- **`feed(string)` 返回多帧**：一次读回 3 条帧是常态（粘包），返回 `vector` 比让调用方循环更省事。
- **统计而不是静默丢帧**：`frames_ok / checksum_errors / format_errors / overflow_drops / noise_bytes`
  五个计数，接上真机第一件事就是看这几个数——**丢帧静默是最难查的一类问题**（我在固件里就被"应答帧被覆盖"坑过）。
- **坏帧后必须能自动重新同步**：清 `buf_`、回到等 `$` 状态，绝不能让一帧坏数据把整条链路拖死。

**③ 不这么写会出什么问题**
- 不做状态机，直接等整包到齐再解析 → 波特率一高就丢数据；也没有处理半包（TCP/串口都是流）。
- 超长帧不丢弃 → 缓冲越界写，嵌入式端直接 HardwareFault；PC 端至少也是内存被吃光。
- `'*'` + CK 的判定不用"最后 3 个字符"而用"任意位置" → payload 里恰好出现 `*1A` 就会被错误截断。
- 不在帧外的字节上区分 CR/LF → 每帧后面那两个 `\r\n` 都会被算成噪声，统计数值失真，看数据时误判链路质量。
- 坏帧后不清缓冲 → 后面所有帧全部错位，现象是"坏一次之后再也收不到正确数据"。

## 5. `CommandDispatcher` + 命令表 — `command_dispatcher.hpp` / `commands.hpp`

**① 用了什么语法**
`using Handler = std::function<Frame(const Frame&)>` **通用可调用对象**（函数指针 / lambda 都能装）、`std::unordered_map<string, Handler>` 查表、
**普通自由函数 + lambda** 做 handler（PING 用 lambda、复杂命令用命名函数）、`bool dispatch(req, out)` 用返回值表达"是否找到命令"、
`emplace(...).second` 判断是否插入成功、`std::to_string`、`inline` 头文件函数。

**② 为什么这么设计**
- **为什么用 `std::function` 而不是裸函数指针**：`std::function` 是「通用可调用对象包装器」，函数指针、lambda、仿函数都能装。
  这样注册命令时可以自由选：无状态的简单命令（PING）用 lambda 就地写最简洁，带参数校验的复杂命令（MOVE/JOINT/LOG）用命名函数更清晰。
  裸函数指针只能指向普通函数，接不了「捕获了外部变量的 lambda」，扩展性差。
- **用 `unordered_map` 而不是 if-else 链**：固件 `proto_dispatch()` 那 14 个 `strcmp` 是 O(n) 逐条比，
  换哈希表是 O(1)；而且"加新命令 = 多注册一行"，符合开闭原则（对扩展开放、对修改关闭）。
- **`dispatch` 返回 `bool` + 出参、未知命令返回 `false`**：把"没找到命令"这个结果显式化，
  回 `ERR,1` 是调用方的决策，分派器不该自作主张（它不知道上层想怎么报告）。
- **命令层只做参数校验，`Frame` 只切字段**：`MOVE,6,1500` 的"ch 越界"在 `commands.hpp` 的普通函数 `handleMove` 里判，
  对应固件 `ERR,3`——解析层和业务层彻底解耦。

**③ 不这么写会出什么问题**
- 用 if-else 链 → 加命令要改分派函数，分支越多越容易漏 `else`，且时间复杂度随命令数线性增长。
- 用裸函数指针 → 接不了「捕获了外部变量的 lambda」，以后想做"记住状态"的命令（比如记住上次位置）就没法就地写；
  `std::function` 统一了函数指针和 lambda，扩展性更好（代价是类型擦除、一点点额外开销，控制链路可忽略）。
- `emplace` 不检查 `.second` → 同名命令被悄悄覆盖，旧逻辑被替换还不知道（`registerCommand` 返回 false 就是要拦住这个）。
- `registerCommand` 不判 handler 是否为 `nullptr` → 调用时空指针函数直接崩。所以注册前必须检查。
- 命令层和解析层混在一起 → 加一条命令要同时改解析器，改一处协议会牵连所有命令。

## 6. `ISerialPort` — `iserial_port.hpp`

**① 用了什么语法**
纯虚函数接口（`= 0`）、`virtual ~ISerialPort() = default` 虚析构、`struct SerialConfig` 默认成员初始化、
`std::uint8_t*` 裸指针作为读写缓冲、`const` 成员函数（`isOpen`）、注释里写清返回约定（>0 有数据 / 0 无数据 / 负数出错）。

**② 为什么这么设计**
- **抽象成纯虚接口**：上层（阶段 3 的 CLI、阶段 4 的接收线程）只依赖 `ISerialPort`，
  不关心底层是 `CreateFile`、`termios` 还是内存里的 Mock——这是"跨平台"三个字真正的落点，
  也是"依赖倒置"（面向抽象编程，不面向具体实现）的直接体现。
- **虚析构必须有**：用基类指针/引用释放派生对象时，没有虚析构，派生类析构不执行
  → 串口句柄/文件描述符泄漏 + 未定义行为。这是"为什么 `~ISerialPort` 要 `virtual`"的典型答案。
- **返回约定明确**：`read` 用正/零/负区分"有数据/没数据/出错"，把"读不到数据"和"读失败"分开，
  避免调用方用错误处理逻辑去兜"暂时没数据"这种正常情况。

**③ 不这么写会出什么问题**
- 析构不加 `virtual` → `delete 基类指针` 只析构基类，派生类的 `CloseHandle`/`::close` 不执行，句柄泄漏。
- 不抽接口、直接在上层写 `#ifdef` 调 Win/Linux API → 业务逻辑和平台代码缠死，换平台要改一堆，
  Mock 也插不进去（没法脱离硬件测试）。
- 接口里塞进平台相关成员（比如 `HANDLE`）→ 接口不再是"接口"，Linux 那边根本编译不过。

## 7. `MockSerialPort` — `mock_serial_port.hpp`

**① 用了什么语法**
继承 `ISerialPort` 实现全部纯虚函数、复用 `RingBuffer<std::uint8_t>` 作为接收缓冲（模块间协作）、
`std::vector<std::uint8_t> tx_` 记录发送、`std::string` 构造（不用 `std::move`）、`override` 关键字、
`inject()`/`sentBytes()` 等测试辅助方法（Mock 独有）。

**② 为什么这么设计**
- **不碰硬件**：所有逻辑（收字节→解析→分派→应答→写回）都能脱离机械臂验证，
  这本身就是接口抽象的价值证明——"换一个实现，业务逻辑一行不用改"。
- **两个方向分开**：`inject()` 模拟"设备发来"，`write()` 进 `sentBytes()` 供断言"上位机发出去了什么"，
  测试就能精确断言 `$PING*10` 进来、`$OK,PONG*3E` 出去。
- **接收端复用 `RingBuffer`**：一石二鸟——顺带在真实数据流里锻炼了模块 1 的代码。

**③ 不这么写会出什么问题**
- 用真串口写测试 → 测试依赖硬件在位、波特率正确、设备响应及时，CI 上根本跑不了（一次改坏还要插线才能发现）。
- Mock 里不用 `override` → 拼错虚函数名（如 `Read`）编译器不报错，你调用的还是基类的纯虚函数，链接期才发现。

## 8. Win32 / Linux 串口 + 平台工厂 — `win32_serial_port.hpp` / `linux_serial_port.hpp` / `serial_port_factory.hpp`

**① 用了什么语法**
`#ifdef _WIN32` / `#ifdef __linux__` 条件编译、Win32 API（`CreateFileA`/`DCB`/`COMMTIMEOUTS`/`ReadFile`/`WriteFile`）、
POSIX API（`open`/`tcgetattr`/`tcsetattr`/`select`）、RAII（析构里 close）。
（本文件是平台胶水层，核心是条件编译 + 平台 API；工厂返回 `std::unique_ptr` 见下文。）

**② 为什么这么设计**
- **用平台宏隔离**：Win 代码只在 `_WIN32` 下参与编译，Linux 代码只在 `__linux__` 下，
  换台机器换个宏，同一个接口三种实现互不干扰。
- **工厂统一入口**：上层只写 `createSerialPort("COM3")`，不碰任何 `#ifdef`——平台判断收进一个函数，
  调用点干净，也方便以后加新平台（比如 macOS）。
- **返回 `std::unique_ptr<ISerialPort>`（RAII 独占所有权）**：工厂内部 `make_unique` 建对象，串口的生命周期跟着这个 `unique_ptr` 走，
  谁持有谁负责，出了作用域自动释放——「谁申请、谁释放」交给编译器，杜绝忘记 `delete` 的泄漏。
- **`COM10` 以上要写 `\\.\COM10`**：Win32 的知名坑，代码里已经处理了（`"\\\\.\\" + name`）。

**③ 不这么写会出什么问题**
- Win/Linux 代码不隔离 → 一个文件里 `CreateFile` 和 `termios` 并存，任何一边编译都报错。
- 平台宏写错（比如用 `__linux` 而不是 `__linux__`）→ 判断静默失效，选了错误的实现，还不好查。
- 句柄不 RAII → 忘记 close 就泄漏；异常路径（open 中途失败）更容易漏掉清理（本实现每个失败分支都 close）。
- 用裸指针就得手动 `delete`：一旦某个提前 return 忘了释放就泄漏；`std::unique_ptr` 把这个风险交给编译器，
  出了作用域自动释放（这是它相对裸指针的核心价值——RAII 覆盖所有退出路径）。

## 9. `Logger` — `logger.hpp`

**① 用了什么语法**
Meyers 单例（函数内 `static Logger inst`）、`enum class LogLevel`、`= delete` 禁用拷贝、
`std::ofstream` 写文件（`std::ios::app` 追加 + `std::endl` 刷盘）、`std::chrono::system_clock` + `std::localtime` + `std::strftime` 时间戳、`switch` 返回 `const char*`。

**② 为什么这么设计**
- **Meyers 单例**：函数内静态变量是 C++11 起**线程安全初始化**的，且**懒加载**（第一次调用才构造）——
  比"类内 static 成员"或"全局对象"更省事也更安全，是现在推荐的单例写法。
- **分级过滤**：`Debug/Info/Warn/Error` 加阈值，正式跑起来只打 `Info` 以上，调试时降到 `Debug`；
  避免日志淹没关键信息，这也是"日志怎么分级"的常见做法。
- **文件用追加 + endl**：`ios::app` 不覆盖历史日志；`endl` 刷盘保证日志立刻落文件（而不是程序崩了最后几行没写进去）。

**③ 不这么写会出什么问题**
- 单例不删拷贝构造 → 有人 `Logger copy = Logger::instance();` 出第二个实例，全局状态被复制，出现两套日志。
- 用"全局对象"而非函数内 static → 静态初始化顺序不确定（跨编译单元谁先初始化说不清），可能用到还没构造的日志器。
- 文件流不 `endl` 刷盘 → 缓冲区没写满时程序崩溃，最后几行关键日志丢失（排障时最要命）。

## 10. `SerialSession` — `session.hpp`

**① 用了什么语法**
组合（持 `std::unique_ptr<ISerialPort>` 独占所有权 + `FrameParser`）、`std::chrono::steady_clock` + `milliseconds` 算超时、
`std::this_thread::sleep_for` 让出 CPU、`std::vector<uint8_t>` 逐字符复制做字符串↔字节转换（不用 `reinterpret_cast`）、
`bool request(req, out, timeout)` 用返回值表达超时 / 无应答。

**② 为什么这么设计**
- **把"发请求→收应答"封装成一次调用**：上层只写 `session.request(Frame("PING"), 1000)`，不用自己拼写循环、
  算超时、喂解析器——这是把"易错的控制流"收进一个函数，调用点保持干净。
- **用 `steady_clock` 而不是 `system_clock`**：`steady_clock` 单调递增、不受系统改时间影响，算超时必须用它
  （`system_clock` 会被 NTP/手动改时间打断，超时计算就乱了）。
- **写失败/读错误返回 `false`（应答写在 `out` 出参），超时也返回 `false`**：把"没有应答"统一成一个 `bool`，调用方一个 `if (!ok)` 就兜住。
- **只取第一帧应答**：设备可能连发（粘包），`frames.front()` 取最早一帧，后续帧留给下轮（或阶段 4 的异步队列）。

**③ 不这么写会出什么问题**
- 用 `system_clock` 算超时 → 系统一校时，超时判断要么提前要么拖后，行为不可复现。
- 读返回 0 时不 `sleep_for` → 忙等循环吃满一个 CPU 核（Mock 那种"立即返回 0"的实现尤其容易踩）。
- `read` 返回 -1（真错误）还继续循环 → 死等超时，掩盖了"串口断了/被拔了"这个真实故障。
- 不判 `written < 0` → 串口没开就发，把空字节当成功，然后傻等应答直到超时。

## 11. `BlockingQueue<T>` — `blocking_queue.hpp`

**① 用了什么语法**
`std::mutex` + `std::lock_guard` / `std::unique_lock`、`std::condition_variable` 的 `wait_for`（带谓词）、
`std::deque` 做底层队列、`push(T value)` **按值传参 + `std::move` 搬进队列**（没人再用的副本直接搬，不拷贝）、
`tryPop` 用 `std::move` 把队头搬出来构造 `std::optional<T>` 返回、类模板、`mutable std::mutex`（供 const 成员函数加锁）。

**② 为什么这么设计**
- **用条件变量而不是忙等**：没数据时消费者 `wait_for` 睡着，生产者 `push` 时 `notify_one` 唤醒——空转 0 CPU，
  这是"线程协作"最基础的积木，也是生产者-消费者模型的教科书实现。
- **`wait_for` 带谓词**：醒来后再查一次 `!queue_.empty() || closed_`，防"伪唤醒"（spurious wakeup）——
  系统可能无缘无故唤醒线程，不带谓词就可能在空队列上继续取。
- **`close()` 唤醒所有等待者**：接收线程退出时用它通知主线程"不会再有数据了"，让卡在 `tryPop` 的消费者立即返回，
  而不是傻等满整个超时——这是优雅停机的关键一步。
- **`push` 用 `notify_one`、`close` 用 `notify_all`**：入队只需要唤醒一个消费者抢；关闭要唤醒全部（大家都要退出）。

**③ 不这么写会出什么问题**
- 用忙等轮询标志位 → 空转吃满一个 CPU 核，还可能在多核上出现缓存一致性的可见性问题。
- 不带谓词的 `wait` → 伪唤醒后拿空数据、或漏掉"已关闭"这个状态，逻辑错乱（这是并发里最经典的坑）。
- 只在 `push` 里 `notify_one`、忘了在 `close` 里 `notify_all` → 消费者可能永远等在那儿，程序退不出去。
- 返回容器内部引用（`const T& front()`）而不是拷贝 → 锁一释放，别的线程改数据，引用就悬了。

## 12. `AsyncReceiver` — `async_receiver.hpp`

**① 用了什么语法**
`std::thread`（RAII 线程对象）、`std::atomic<bool>` 做停止标志、`std::this_thread::sleep_for` 让出 CPU、
组合（持非拥有 `ISerialPort*` 观察者 + `FrameParser` + `BlockingQueue`）、`= delete` 禁拷贝、析构里 `stop()`。

**② 为什么这么设计**
- **把阻塞读从主线程剥离**：串口读是阻塞的，放主线程会卡住 UI/按键/遥测；丢进后台线程，主线程 `tryPop` 拿帧即可。
- **`std::atomic<bool> stop_` 做停止标志**：主线程写、工作线程读，跨线程共享一个 bool 必须用原子或加锁，
  否则是数据竞争（UB）。原子 bool 是成本最低、语义最清楚的做法。
- **read 带 50ms 超时**：让工作线程每个循环都有机会检查 `stop_`，否则 `stop()` 会永久卡在 `join()` 上。
- **读错误不退出、只记日志 + 短睡**：串口可能被临时拔掉又插回，直接退出线程就失去了自动恢复能力。
- **析构里 `stop()`**：对象销毁时保证线程已 join，避免"析构时线程还在跑"的经典崩溃。

**③ 不这么写会出什么问题**
- 停止标志用普通 `bool` → 主线程写、工作线程读是数据竞争（UB），`stop()` 可能永远看不到置位。
- `stop()` 不 `join()` → 线程还在跑就销毁对象/退出 main，资源泄漏甚至崩溃（`std::thread` 析构时仍在运行会直接 terminate）。
- 析构不 `stop()` → 同样的"可 join 线程析构即终止"问题。
- `stats()` 不加说明就声称精确 → 运行中读它其实和工作线程有竞态，把诊断值当成精确值用会误导判断（本实现已在文档里写清：stop 后读才准）。

---

# 设计取舍 FAQ

**Q：为什么用异或校验，不用 CRC16？**
A：因为对端（STM32 固件）实现的就是异或。协议是两端契约，单方面升级校验方式等于把链路打断。
如果真要升级，正确做法是**协议版本号 + 双端同步改**，这也说明为什么真实协议里都要留版本字段。

**Q：一个比特错的检错能力上异或不如 CRC，你怎么看？**
A：异或对**奇数个比特翻转**能检出，偶数个翻转会漏。对短帧（≤60 字节）的嵌入式控制链路够用，
代价是几乎零计算量（单片机上一个循环就够）。CRC16 的代价是每帧多几十个周期的查表/移位，
换来更强的检错——**选哪个取决于链路噪声和算力预算，不是越强越好**。

**Q：粘包、半包、噪声分别是怎么处理的？**
A：同一套状态机解决：`$` 之前全部丢弃（噪声）；`$` 之后逐字节累积（半包天然支持）；
遇到 `*`+2 位十六进制即成帧（不依赖 CRLF）；成帧后如果缓冲里还有字节，下一次 `feed` 继续处理（粘包）。
关键点是**解析器是无状态依赖的**——它不假设"一次 read 就是一帧"。

**Q：为什么要统计 `checksum_errors`、`noise_bytes` 这些数？**
A：因为它们能直接区分故障类型：`noise_bytes` 大 → 波特率/接线不对，或数据经无线透传模块被重新分包；
`checksum_errors` 多 → 链路干扰或对端编码错误；`overflow_drops` 多 → 对端长度字段不可信或波特率不匹配。
没有这些计数，你就只能盯着屏幕猜。

**Q：为什么 `RingBuffer` 只写深拷贝，`BlockingQueue` 却用了移动语义？**
A：两个取舍，关键看"对象大不大、拷贝贵不贵"。`RingBuffer` 存小对象（字节/小结构），且要练"三法则深拷贝"这个必考点，
所以老老实实拷贝，两个缓冲区互不影响；`BlockingQueue` 存 `Frame`（里面带 `std::string`，拷贝要复制整段字符串内存），
所以 push/pop 用 `std::move` 把对象"搬"进/出队列，省一次字符串复制。
一句话：**小的、教学优先的用拷贝；大的、有真实拷贝成本的用移动语义**——移动不是"更高级就一定用"，是"有成本才用"。

**Q：`RingBuffer` 用 `new T[capacity]` 要求 `T` 可默认构造，这是缺陷吗？**
A：是已知限制。生产级实现用 `std::aligned_storage` + placement new，只在需要时构造元素（`std::vector` 就是这么干的），列为进阶练习。

**Q：函数指针、虚函数接口、`std::function`，什么时候用哪个？**
A：**函数指针**——最简单、零开销，但只能指向普通函数，接不了「捕获了外部变量的 lambda」。
**虚函数接口**——类型关系固定、扩展点明确的场景（比如阶段 2 的 `ISerialPort`），运行时多态是语义的一部分。
**`std::function`**——能存任意可调用对象（函数指针、lambda、仿函数、`std::bind` 结果），适合"要存带状态的行为"；
代价是类型擦除 + 一点额外开销。
本项目命令分派用 `std::function`（`using Handler = std::function<Frame(const Frame&)>`）：无状态的 PING 用 lambda 就地注册，
带参数校验的 MOVE/JOINT/LOG 用命名函数——两种都能装进同一个 `std::function`，扩展性最好。

**Q：为什么用 `unordered_map` 而不是 `switch`/if-else 链分发命令？**
A：① 时间复杂度：if-else 逐条 `strcmp` 是 O(n)，哈希表是 O(1)；② 开闭原则：加命令只改"注册"那一步，
分派逻辑不用动（固件 `proto_dispatch()` 每加一条命令都要改那个长函数）；③ 能运行时注册/注销（可测试、可热插拔）。
代价是内存多一点、哈希碰撞要处理，但对几十条命令的场景完全可忽略。

**Q：lambda 捕获 `[&]` 和 `[=]` 是什么，本项目哪些地方用了 lambda？**
A：`[&]` 按引用捕获、`[=]` 按值捕获（按值下想改外部变量还要加 `mutable`）。回调想"看到外面的实时状态"就按引用捕获；
但要小心**悬垂引用**——如果被捕获的局部变量比 lambda 活得短（比如存进 `std::function` 后又返回），`[&]` 就是定时炸弹，
这时要么用 `[=]`，要么捕获 `shared_ptr`。
本项目 lambda 的真实落点：① `commands.hpp` 里 PING 命令用空捕获 `[]` 的 lambda 就地注册（无状态，最简洁）；
② `tests/test_blocking_queue.cpp` 里 `std::thread` 启动线程传的一次性 lambda（标准库 API 要求可调用对象）。
带捕获 `[&]`/`[=]` 的 lambda 本项目暂时用不到——命令都是无状态纯函数，`AsyncReceiver` 的线程靠成员函数 + `this` 而非捕获。
什么时候才会用带捕获的 lambda？当 handler 需要"记住外部状态"（比如计数、绑定某个串口对象）又不想搞全局变量时。

**Q：`ISerialPort` 的析构为什么必须 `virtual`？**
A：因为 `std::unique_ptr<ISerialPort>` 在析构时会 `delete` 它持有的指针，而那个指针的类型是**基类 `ISerialPort*`**。
如果没有虚析构，编译器只调用基类析构，派生类 `Win32SerialPort` 里的 `CloseHandle` 不会执行，句柄就泄漏了，
而且这属于未定义行为。反过来，只要基类析构是虚的，`delete` 基类指针就会正确一路析构到最派生类。

**Q：为什么要把串口抽象成接口，直接 `#ifdef` 写 Win/Linux 不行吗？**
A：两个理由。① **可测试性**：抽象成接口后能塞一个 `MockSerialPort`，不插机械臂就能跑通"收字节→解析→分派→应答"
整条链路，测试快且稳定（不依赖硬件在位、波特率、设备响应）。② **解耦**：业务逻辑只依赖 `ISerialPort`，
换平台只换工厂那一行，不用在几百行逻辑里到处 `#ifdef`。Mock 的价值不是"模拟"，是"让你敢在没硬件时也写测试"。

**Q：串口句柄是怎么保证不泄漏的？**
A：RAII + **显式释放**。`Win32SerialPort`/`LinuxSerialPort` 的析构里 `close()`，`open()` 的每个失败分支也都先 close 再返回；
工厂返回 `std::unique_ptr<ISerialPort>`（内部 `make_unique`），`SerialSession` 用 `std::move` 把它接进来、独占所有权，
析构时自动释放——「谁申请、谁释放」交给编译器，正常和异常路径都不漏。

**Q：单例有几种写法，为什么项目里用 Meyers 单例？**
A：常见三种：① 全局对象/类内 static 成员（**饿汉**，启动就构造，静态初始化顺序有隐患）；
② 双检锁 + `new`（**懒汉**，写法繁琐、历史上易踩内存序的坑）；③ 函数内 static（**Meyers 单例**）。
Meyers 写法在 C++11 起由标准保证**线程安全的懒加载**，代码还最短——所以现代 C++ 推荐它。

**Q：为什么"客户端(发请求)"和"设备端(应答分派)"要分成两个东西？**
A：因为它们是**两个角色**。`SerialSession` 是客户端——发命令、等应答、解析返回；`CommandDispatcher` + `commands.hpp`
是设备端逻辑（模拟/实现固件怎么处理命令）。分开后：客户端能被真机、Mock 通用；设备端逻辑能单独拿来
做模拟器、做单元测试（`test_serial_integration` 就是拿它当固件的替身）。混在一起就会"既当运动员又当裁判"，没法测。

**Q：请求超时为什么用 `steady_clock`，不能用 `system_clock`？**
A：`steady_clock` 是单调递增的，只往前不会回退；`system_clock` 是墙上时钟，会被 NTP 校时、手动改时间打断。
算"过了多少毫秒"必须用单调时钟，否则校时一下，超时可能瞬间"归零"或"暴涨"，行为不可复现。

**Q：`wait_for` 为什么要带谓词（伪唤醒是什么）？**
A：条件变量可能被**无缘无故唤醒**（spurious wakeup），即使没人 `notify`。所以标准写法是
`cv.wait(lock, 谓词)`，醒来后**重新检查一次条件**，不满足就继续睡。不带谓词、只写 `wait` 然后直接取数据，
就会在空队列上取到脏值——这是并发编程最经典的坑，也是为什么标准库提供了带谓词的重载。

**Q：跨线程共享的停止标志，为什么用 `std::atomic<bool>` 而不是普通 `bool`？**
A：主线程写、工作线程读同一个 `bool`，不加同步就是**数据竞争**（C++ 标准定义的未定义行为），
编译器可能把读优化掉、或 CPU 缓存不一致导致永远看不到更新。`std::atomic<bool>` 用 `store`/`load` 保证可见性，
是成本最低、意图最清楚的做法（比整把锁轻量得多）。

**Q：这个项目里"线程安全"具体做了哪些？**
A：三处。① `BlockingQueue` 用 `mutex + condition_variable`；② `AsyncReceiver` 用 `atomic<bool>` 做停止标志；
③ `Logger` 和 `MockSerialPort` 加 `mutex`（因为接收线程和主线程会并发写日志、并发读写 Mock 缓冲）。
原则是：**单线程时省锁，一旦引入多线程，凡是跨线程共享的数据都得同步**——不能想当然"不会并发"。

# 已知限制 / 待办

- [ ] `RingBuffer` 暂不支持非默认构造的 `T`。
- [ ] `RingBuffer` 满时的策略目前是"丢新数据"，后续给 `FrameParser` 增加"丢最旧"的覆盖模式。
- [ ] `FrameParser` 没有行长度上限之外的 DoS 防护（恶意对端持续发 `$` 会不断清缓冲，当前无害但值得写进文档）。
- [ ] `FrameParser` 只在单线程使用：`AsyncReceiver` 由单一工作线程独占解析器，跨线程交接走 `BlockingQueue`（不是 RingBuffer）。
- [ ] `AsyncReceiver` 单次使用（`start()` 只能调一次）；运行中读 `stats()` 是近似值（与工作线程有良性竞态，`stop()` 之后读才准确）。
- [ ] 涉及运动/状态的命令（STATE/HOME/STOP/RESUME/MANUAL/REC/RECEND/PLAY/TNUM/CLEAR）尚未注册，
  需要串口后端（阶段 2）与机械臂状态模型（尚未单独成阶段；`STATE` 要回当前状态和 6 路脉宽，`REC`/`PLAY` 要状态机）。
- [ ] `Frame::parse` 的严格度已确认（从严），若将来要与固件宽容策略完全一致，需要一个 `parseLenient` 变体。
- [ ] Linux 串口后端（termios）已写完源码，本机已用 `-D__linux__` 做过编译检查（零警告），
  但未在真实 Linux 环境（真机 / 虚拟机 / WSL）编译运行过，需在 Linux 上 `g++ -Iinclude -c` 复核。
- [ ] 平台宏依赖 `_WIN32`/`__linux__`；Cygwin/POSIX 的 g++ 不定义 `_WIN32`，在该工具链下只有 Mock 可用（真实 MinGW-w64 / MSVC / GCC on Linux 都正常定义，不受影响）。
- [ ] `armctl` 的 `--baud` 参数目前只被解析、还没喂给 `SerialConfig`（真实串口默认 115200 8N1，够用，后续把 baud 传进 config）。
- [ ] 尚未与真机联调（阶段 3 的验收标准，需要机械臂 + 串口线就位）。
