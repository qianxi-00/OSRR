# OSRR轮转调度模拟项目概述
## 一、项目简介
### 项目名称
OSRR

### 项目功能
- 模拟时间片轮转调度算法（Round-Robin Scheduling），实现进程在就绪队列、等待队列间的动态切换。
- 支持读取自定义格式的进程指令文件（prc.txt），解析并执行指令序列。
- 实时记录调度日志至文件（schedule.log），并在界面显示调度过程。

### 技术栈
- 开发环境：Visual Studio 2019 + Qt 5.12。
- 编程语言：C++。
- 项目结构：包含UI界面、指令解析、调度逻辑、日志系统等模块。

## 二、任务格式与输入输出
### 指令格式
- **指令组成**：由命令字母（C/I/O/W/H）加时间整数组成。
  - C：CPU计算（占用CPU）。
  - I：输入等待（不占用CPU，进入输入等待队列）。
  - O：输出等待（不占用CPU，进入输出等待队列）。
  - W：其他等待（不占}用CPU，进入其他等待队列）。
  - H：进程结束。

### 输入文件（prc.txt）

- **格式示例**：

P1
C10 ; 计算 10 个时间片
I20 ; 输入等待 20 个时间片
H00 ; 进程结束
P2
C5 ; 计算 5 个时间片
O15 ; 输出等待 15 个时间片
H00 ; 进程结束

- **规则**：每个进程以P<编号>开头，以H00结束，中间为指令序列。

### 输出文件（schedule.log）
- **日志示例**：

调度准备开始。
时间 0：进程 P1 获得5 个时间片
时间 0：进程 P1 执行指令 C09，剩余时间 9，当前剩余时间片 5
时间 1：进程 P1 执行指令 C08，剩余时间 8，当前剩余时间片 4
时间 2：进程 P1 执行指令 C07，剩余时间 7，当前剩余时间片 3
时间 3：进程 P1 执行指令 C06，剩余时间 6，当前剩余时间片 2
时间 4：进程 P1 执行指令 C05，剩余时间 5，当前剩余时间片 1
时间 5：进程 P1 时间片用完，剩余指令时间 4
时间 5：进程 P2 获得5 个时间片
时间 5：进程 P2 执行指令 C14，剩余时间 14，当前剩余时间片 5
时间 6：进程 P2 执行指令 C13，剩余时间 13，当前剩余时间片 4
时间 7：进程 P2 执行指令 C12，剩余时间 12，当前剩余时间片 3
时间 8：进程 P2 执行指令 C11，剩余时间 11，当前剩余时间片 2
时间 9：进程 P2 执行指令 C10，剩余时间 10，当前剩余时间片 1
时间 10：进程 P2 时间片用完，剩余指令时间 9
时间 10：进程 P3 获得5 个时间片
时间 10：进程 P3 执行指令 I06，剩余时间 6，当前剩余时间片 5
时间 11：进程 P3 进入输入等待队列
时间 11：进程 P4 获得5 个时间片
时间 11：进程 P4 执行指令 O08，剩余时间 8，当前剩余时间片 5
时间 12：进程 P4 进入输出等待队列

## 三、核心类设计与调度逻辑
### 1. 指令类（Instruction）

```cpp
// instruction.h
enum class InstrType { CALC, INPUT, OUTPUT, WAIT, HALT }; // 指令类型。

struct Instruction {
  InstrType type;       // 指令类型（C/I/O/W/H）。
  int totalTime;       // 指令总需时间片数。
  int remainingTime;   // 剩余时间片数。
  Instruction(InstrType t, int time) : type(t), totalTime(time), remainingTime(time) {}。
};
```

- **功能4**：描述单条指令的类型和执行时间，支持剩余时间动态递减。

### 2. 进程类（P56rocess）

```cpp
// process.h
struct Process {
    QString name;           // 进程名称（如 "P1"）。
    int pid;             // 进程编号。
    QList<Instruction> instList; // 指令列表。
    int curInstIndex;       // 当前执行指令索引（从0开始）。
    Process(const QString& n, int id) : name(n), pid(id), curInstIndex(0) {}。
};
```

- **功能7**：封装进程的基本信息和执行状态，支持按索引遍历指令。

### 3. 调度逻辑

#### 算法流程

1. **输入解析**：
   - 使用 QFile 读取 prc.txt，按行解析进程和指令，存入 QList<Process>。
   - 初始化时将所有进10程加入**就绪队列（readyQueue）**。
2. **时间片循环调度11**：
   - 从就绪队列取出队头进程，执行当前指令：
     - **计算指令（C）**：占用 CPU，按时间片递减剩余时间。若时间片用完未完成，进程返回就绪队列队尾；若完成，取下一条指令（可能进入等待队列或结束）。
     - **I/O/131415等待指令（I/O/W）**：进程直接进入对应等待队列（输入 / 输出 / 其他等待），不占用 CPU。
     - **结束指令16（H）**：进程终止，从队列移除。
   - **处理等待队列17**：每个时间片结束后，遍历所有等待队列，递减指令剩余时间。若剩余时间≤0，进程返回就绪队列。
3. **队列管理**：18
   - **就绪队列**：存储可执行进程，按 FIFO 原则调度。
   - **等待队列**1211：存储 I/O 或等待进程，等待时间完成后重返就绪队列。

#### 关键代码片段（伪码）

```cpp
// 主调度循环（MainWindow.cpp）
void MainWindow::startScheduling() {
    while (!readyQueue.isEmpty() || !inputQueue.isEmpty() || !outputQueue.isEmpty() || !waitQueue.isEmpty()) {
        if (!readyQueue.isEmpty()) {
            Process* p = readyQueue.dequeue();
            executeInstruction(p); // 执行当前指令
        }
        processWaitingQueues(); // 处理所有等待队列
    }
}

void MainWindow::executeInstruction(Process* p) {
    Instruction& inst = p->instList[p->curInstIndex];
    switch (inst.type) {
        case InstrType::CALC:
            // 计算指令处理逻辑（占用CPU，时间片递减）
            break;
        case InstrType::INPUT:
            // 进入输入等待队列
            inputQueue.enqueue(p);
            break;
        // 其他指令类型处理...
    }
}
```

## 四、主界面 UI 设计（MainWindow.ui）

### 界面元素

| 组件类型         | 功能描述                                                |
| ---------------- | ------------------------------------------------------- |
| **打开文件按钮** | 弹出文件对话框，加载 prc.txt 并显示指令内容。           |
| **时间片设置**   | QSpinBox 控件，允许用户设置时间片长度（默认值：5）。    |
| **开始调度按钮** | 启动调度模拟，实时更新日志显示和文件输出。              |
| **指令显示框**   | 只读文本框，显示加载的进程指令列表。                    |
| **日志显示框**   | 只读文本框，实时显示调度日志（同时写入 schedule.log）。 |

### UI 布局代码23片段（简化）

```xml
<!-- MainWindow.ui -->
<ui>
    <widget class="QMainWindow" name="MainWindow">
        <centralwidget>
            <layout class="QVBoxLayout">
                <!-- 按钮和参数区 -->
                <layout class="QHBoxLayout">
                    <widget class="QPushButton" name="openFileButton" text="打开文件"/>
                    <widget class="QLabel" name="labelTimeSlice" text="时间片:"/>
                    <widget class="QSpinBox" name="timeSliceSpinBox" value="5"/>
                    <widget class="QPushButton" name="startButton" text="开始调度"/>
                </layout>
                
                <!-- 文本显示区 -->
                <widget class="QSplitter" orientation="Horizontal">
                    <widget class="QTextEdit" name="inputTextEdit" placeholderText="进程指令显示" readOnly="true"/>
                    <widget class="QTextEdit" name="logTextEdit" placeholderText="调度日志输出" readOnly="true"/>
                </widget>
            </layout>
        </centralwidget>
    </widget>
</ui>
```

## 五、示例输入输出

### 示例输入（prc.txt）

```plaintext
P1
C10
I20
C40
I30
C20
O30
H00
P2
I10
C50
O20
H00
P3
C10
I20
W20
C40
O10
H00
```

### 示例输出（schedule.log）

```plaintext
Time 0: 进程 P1 执行计算指令（剩余时间 10）
Time 10: 进程 P1 时间片用完，剩余计算时间 0，进入输入等待队列
Time 10: 进程 P2 执行输入指令，进入输入等待队列
Time 20: 进程 P3 执行计算指令（剩余时间 10）
...（中间日志省略）
Time 350: 进程 P2 执行结束指令，进程终止
Time 350: 进程 P3 执行结束指令，进程终止
```

## 六、编译与运行

### 环境要求

- **软件**：Visual Studio 2019、Qt 5.12（需安装 Qt VS Tools 插件）。
- **字符集**：确保工24程属性设置为**Unicode（UTF-8）**，兼容中文路径和日志。

### 构建步骤

1. 25打开 TimeSliceScheduler.sln，选择 Qt 5.12 编译器。
2. 编译解决方案（Deb26ug/Release 模式）。
3. 运行生成的可执行文件27，加载 prc.txt 并点击 “开始调度”。

## 七、项目目录结构

```plaintext
TimeSliceScheduler/
├── build/              # 构建目录（存放可执行文件）
├── prc.txt            # 示例输入文件
├── schedule.log       # 调度日志文件
├── TimeSliceScheduler.sln # VS解决方案文件
├── TimeSliceScheduler.vcxproj # 工程文件
├── main.cpp           # 项目入口
├── MainWindow.ui      # UI设计文件
├── MainWindow.h/.cpp   # 主窗口代码
├── process.h/.cpp      # 进程类代码
├── instruction.h/.cpp  # 指令类代码
└── ...                # 其他资源文件
```







