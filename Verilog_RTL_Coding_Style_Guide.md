# Verilog RTL编码与格式规范

## 1. 适用范围与规则等级

本规范适用于使用Verilog-2001开发的可综合FPGA和ASIC RTL代码，包括数据通路、控制逻辑、状态机、通信接口以及模块级和系统级顶层。Testbench可以使用仿真专用语句，但可综合RTL不得依赖仿真专用代码实现功能。

本文使用以下规则等级：

- **必须、不得、禁止**：强制要求，代码不符合时必须修改。
- **推荐、应、尽量**：默认采用；确有工程原因时可以偏离，但应在评审中说明。
- **可以、可选**：根据具体设计选择。

设计时应优先保证：

1. RTL结构简单、明确并可综合。
2. 时序、位宽、复位、握手和时钟域关系清晰。
3. 同一工程的命名、排版和实现风格一致。
4. 代码按照信号产生、处理和输出的因果关系组织，便于阅读和审查。

## 2. 文件结构与代码排版

### 2.1 文件结构

RTL文件推荐使用以下结构：

```verilog
`timescale 1ns/1ps

// Module description
module module_name #(
    // Parameters
) (
    // Ports
);

    // Local parameters

    // Internal signal declarations

    // Functional block A
    // Related assign, combinational, sequential and instance logic

    // Functional block B
    // Related logic

endmodule
```

参数、常量和信号声明必须统一放在模块前部，并按照功能分组。声明之后的逻辑代码应按功能组织：

- 存在直接数据或控制关系的代码应尽量相邻。
- 每个`always`块只承担一个明确功能。
- 每个功能组按照“条件或数据产生、中间处理、状态更新、结果输出”的顺序排列。
- 产生中间信号的代码原则上放在使用该信号的代码之前。
- 同一功能的连续赋值、组合逻辑、时序逻辑和子模块例化应放在同一区域。
- 不相关功能之间使用空行和简短的功能注释分隔。
- 不得为了靠近使用位置而将参数或信号声明分散到模块中部。

### 2.2 缩进、换行与对齐

- 使用4个空格缩进，不得使用Tab。
- 每行只写一条主要语句。
- 文件末尾必须保留换行，行尾不得保留无意义空格。
- 不得使用不可见字符或非标准标点。
- 相关代码之间只保留必要空行。
- 同一功能组内的参数、声明和赋值应按列对齐；不得跨越空行或功能组强行对齐。
- 对齐不得导致代码行过长或加入大量无意义空格。

`for`循环头以及`if`、`else if`等逻辑判断应尽量保持在同一行。只有单行过长并明显影响阅读，或确实需要表达逻辑分组时才换行；换行后的缩进必须一致。

```verilog
if (request_valid && request_ready && !transfer_busy && (payload_count < MAX_PAYLOAD_COUNT)) begin
    request_accept = 1'b1;
end
```

### 2.3 循环格式

`for`循环的临时变量优先使用`i`、`j`、`k`，并按照嵌套层级依次使用，不得使用`index`作为无功能含义的临时循环变量。

具有明确功能含义的索引信号仍应使用`byte_index`、`channel_index`等名称。

```verilog
integer i;
integer j;

always @(*) begin
    for (i = 0; i < CHANNEL_COUNT; i = i + 1) begin
        for (j = 0; j < BYTE_COUNT; j = j + 1) begin
            channel_data[i][(j*8) +: 8] = payload_data[(i*BYTE_COUNT*8) + (j*8) +: 8];
        end
    end
end
```

## 3. 命名规范

### 3.1 模块、文件、参数和常量

- 文件名必须与文件中的顶层模块名一致。
- 文件名和模块名使用小写下划线格式，例如`uart_tx_8n1.v`和`uart_tx_8n1`。
- 参数和本地常量使用大写下划线格式。
- 参数名称应包含必要的物理含义或单位。

```verilog
parameter integer CLK_FREQ_HZ = 96000000;
parameter integer DATA_WIDTH  = 8       ;

localparam integer FRAME_BYTES = PAYLOAD_BYTES + 6;
localparam [7:0]   SOF_BYTE    = 8'hA5            ;
```

### 3.2 信号和寄存器

信号和寄存器使用小写下划线格式，名称必须表达功能，不得只描述数据类型。

推荐使用以下后缀：

| 后缀 | 含义 |
|---|---|
| `_en` | 使能 |
| `_start` | 启动脉冲 |
| `_done` | 完成脉冲 |
| `_valid` | 数据有效 |
| `_ready` | 接收就绪 |
| `_busy` | 忙状态 |
| `_error` | 错误状态或脉冲 |
| `_cnt`、`_counter` | 计数器 |
| `_index` | 具有功能含义的索引 |
| `_latched` | 锁存值 |
| `_next` | 下一计算值 |
| `_1d`、`_2d`、`_3d` | 延迟或同步链的第一级、第二级、第三级 |
| `_pos` | 上升沿检测产生的单周期脉冲 |
| `_neg` | 下降沿检测产生的单周期脉冲 |
| `_n` | 低有效信号 |

打拍寄存器和同步链的级数后缀必须：

- 与实际寄存器级数一致。
- 从`_1d`开始连续命名，不得跳级。
- 不得使用同一后缀表示不同延迟级数。

`_pos`和`_neg`只表示边沿检测脉冲，不表示寄存器使用`posedge`或`negedge`采样。

同一个概念必须使用同一种缩写，不得在同一工程中混用`frame/frm`、`count/cnt`、`enable/en`等同义名称。

## 4. 参数、端口和信号声明

### 4.1 参数和端口

参数和端口必须每行声明一个。端口按照以下顺序分组：

1. 时钟和复位。
2. 配置输入。
3. 数据输入。
4. 控制和握手输入。
5. 数据输出。
6. 状态和诊断输出。

```verilog
module example #(
    parameter integer DATA_WIDTH = 8 ,
    parameter integer DEPTH      = 16
) (
    input  wire                  clk       ,
    input  wire                  reset     ,

    input  wire [15:0]           cfg_period,

    input  wire [DATA_WIDTH-1:0] data_in   ,
    input  wire                  data_valid,
    input  wire                  data_ready,

    output reg  [DATA_WIDTH-1:0] data_out  ,
    output reg                   data_done
);
```

声明要求：

- 端口方向必须显式写出`input`或`output`。
- Verilog-2001端口必须显式使用`wire`或`reg`。
- 参数化位宽必须使用括号明确运算优先级。
- 端口名称应表达信号方向和功能。
- 同一端口组内应对齐方向、类型、位宽、名称和行尾逗号。
- 参数列表中的名称、`=`和行尾逗号应在相关参数组内对齐。
- 最后一个参数或端口不得添加多余逗号。

### 4.2 内部信号

相关信号必须分组声明：

```verilog
reg  [7:0]  byte_index     ;
reg  [15:0] crc_value      ;
reg  [15:0] period_counter ;

wire        byte_complete  ;
wire        frame_complete ;
wire [15:0] crc_next       ;
```

- 寄存器使用`reg`，组合连接和模块输出连接使用`wire`。
- 不得保留未使用的信号。
- 信号位宽必须满足功能需要，但不得无理由统一使用32 bit或64 bit。
- 协议字段宽度必须与协议定义一致。
- 同一功能组内应对齐类型、位宽、名称和行尾分号。
- 不同功能组使用空行分隔，不得为了跨组对齐加入大量空格。

## 5. 组合逻辑与连续赋值

### 5.1 组合过程块

组合逻辑使用`always @(*)`和阻塞赋值。所有输出必须在块开始处设置默认值，或通过完整分支覆盖所有路径。

```verilog
always @(*) begin
    result = default_value;

    if (condition_a) begin
        result = value_a;
    end
    else if (condition_b) begin
        result = value_b;
    end
end
```

- 不得在组合块中保留输出的上一次值。
- 不得遗漏可能执行路径上的赋值，否则会推断锁存器。
- 不得在组合逻辑中使用时钟或延时语句。
- 一个组合输出只能由一个组合过程块或一条连续赋值驱动。
- 不得形成组合环路。

### 5.2 连续赋值

简单逻辑表达式、比较结果、状态条件、模块间直接连接以及不需要过程控制的算术逻辑，优先使用`assign`。

```verilog
assign byte_complete  = byte_done && (byte_index == LAST_BYTE_INDEX);
assign frame_complete = byte_complete && crc_match                  ;
```

表达式格式要求：

- 较短的条件、比较和连续赋值必须保持单行。
- 只有表达式过长、条件较多或需要体现逻辑分组时才换行。
- 同一文件中相同复杂度的表达式应采用一致的换行方式。
- 同一功能组内的多条`assign`应对齐左值、`=`和行尾分号。
- 长表达式换行时优先保证逻辑分组和缩进，不得为了对齐行尾分号产生过长代码行。

```verilog
assign frame_accept = frame_valid && length_match &&
                      crc_match && sequence_match;
```

## 6. 时序逻辑与复位

### 6.1 时序过程块

时序逻辑必须使用非阻塞赋值。一个信号只能有一个驱动源，一个寄存器只能由一个时序`always`块赋值。

```verilog
always @(posedge clk or posedge reset) begin
    if (reset) begin
        counter <= 16'd0;
    end
    else if (counter_clear) begin
        counter <= 16'd0;
    end
    else if (counter_load) begin
        counter <= counter_load_value;
    end
    else if (counter_en) begin
        counter <= counter + 1'b1;
    end
end
```

条件分支必须遵守以下格式：

- `if`、`else if`和`else`必须使用`begin/end`，即使分支中只有一条语句。
- `begin`与条件写在同一行。
- `else if`和`else`单独换行，并与对应的`if`保持同一级缩进。
- `end`与所属分支保持同一级缩进。

条件优先级通常为复位、清除或禁止、装载、功能更新、保持原值，必须通过`if / else if / else`明确表达。

### 6.2 时序块职责

默认一个寄存器使用一个时序`always`块。只有以下相关寄存器可以放入同一个块：

- 同一同步链中的多级寄存器。
- 同一流水线级的数据和有效标志。
- 必须原子更新的一组寄存器。
- 复位、使能和更新条件完全一致的一组寄存器。

功能无关或更新条件不同的寄存器必须拆分到不同的时序块。

### 6.3 复位

- 同一模块必须使用统一的复位极性和复位方式。
- 所有控制寄存器必须有明确复位值。
- 输出接口的复位值必须符合协议空闲状态。
- 异步复位的释放必须由系统级设计保证同步。

高有效异步复位使用`posedge reset`并在分支中判断`if (reset)`；低有效异步复位使用`negedge reset_n`并判断`if (!reset_n)`。两种形式不得在同一模块中混用。

## 7. 状态机、计数器、脉冲与握手

### 7.1 状态机

状态必须使用有意义的常量名称。状态机推荐采用三段式：状态寄存器、下一状态组合逻辑和输出组合逻辑分别放在独立的`always`块中。

```verilog
localparam [1:0] STATE_IDLE = 2'd0;
localparam [1:0] STATE_SEND = 2'd1;
localparam [1:0] STATE_WAIT = 2'd2;

reg [1:0] state     ;
reg [1:0] state_next;

always @(posedge clk or posedge reset) begin
    if (reset) begin
        state <= STATE_IDLE;
    end
    else begin
        state <= state_next;
    end
end

always @(*) begin
    state_next = state;

    case (state)
        STATE_IDLE: begin
            if (start) begin
                state_next = STATE_SEND;
            end
        end

        STATE_SEND: begin
            if (send_ready) begin
                state_next = STATE_WAIT;
            end
        end

        STATE_WAIT: begin
            if (response_valid) begin
                state_next = STATE_IDLE;
            end
        end

        default: begin
            state_next = STATE_IDLE;
        end
    endcase
end

always @(*) begin
    send_valid = 1'b0;
    busy       = 1'b0;

    case (state)
        STATE_SEND: begin
            send_valid = 1'b1;
            busy       = 1'b1;
        end

        STATE_WAIT: begin
            busy = 1'b1;
        end

        default: begin
            send_valid = 1'b0;
            busy       = 1'b0;
        end
    endcase
end
```

状态机必须满足：

- 每个`case`包含`default`。
- 非法状态恢复到安全状态。
- 下一状态和输出组合逻辑具有完整默认赋值。
- 状态输出不得形成组合环路。

### 7.2 计数器和位宽

- 计数器和索引的位宽必须覆盖合法最大值，但不得无理由扩大。
- 常量推荐明确位宽，例如`1'b0`、`8'd0`和`16'hFFFF`。
- 协议常量和设计常量应使用`parameter`或`localparam`命名，不得使用魔法数字。
- 必须检查加法、乘法和移位是否溢出。
- 必须检查减1操作是否可能发生无符号下溢。
- 比较双方必须使用兼容位宽。
- 不得依赖工具对有符号数和无符号数的隐式转换。
- 参数参与切片或索引时必须确认范围合法。

参数化部分选择可以写为：

```verilog
payload[(byte_index*8) +: 8]
```

### 7.3 脉冲和握手

将事件条件直接寄存，例如`done <= complete;`，只有在`complete`本身保证为单周期事件时才能得到单周期脉冲。若输入是可能持续多个周期的同步电平，必须先进行边沿检测：

```verilog
reg  signal_1d  ;

wire signal_pos ;
wire signal_neg ;

always @(posedge clk or posedge reset) begin
    if (reset) begin
        signal_1d <= 1'b0;
    end
    else begin
        signal_1d <= signal;
    end
end

assign signal_pos = signal && !signal_1d;
assign signal_neg = !signal && signal_1d;
```

`signal`必须已经同步到当前时钟域。`signal_pos`和`signal_neg`分别只在检测到对应边沿时保持一个时钟周期。

Valid/Ready接口以`valid && ready`作为传输成功条件，并满足：

- 发送端有数据时即可拉高`valid`，不得以等待`ready`作为拉高`valid`的前提。
- `valid`拉高后，必须保持`valid`和数据稳定，直到握手完成。
- 接收端只在能够接受数据时拉高`ready`。
- 忙状态期间的新请求必须明确规定为忽略、缓存或报错。
- 所有脉冲接口必须明确脉冲宽度。

## 8. 时钟域跨越

异步单bit输入必须至少使用两级同步器：

```verilog
always @(posedge clk or posedge reset) begin
    if (reset) begin
        async_in_1d <= 1'b0;
        async_in_2d <= 1'b0;
    end
    else begin
        async_in_1d <= async_in;
        async_in_2d <= async_in_1d;
    end
end
```

多bit数据不得直接对每一位分别同步。必须根据场景选择：

- 握手。
- 异步FIFO。
- Gray码计数器。
- 数据保持配合Valid同步。

同步级数后缀必须与实际同步级数一致。

## 9. 模块例化

模块例化（即模块实例化）的参数和端口必须使用具名连接，禁止使用位置连接。

```verilog
submodule #(
    .DATA_WIDTH(DATA_WIDTH),
    .DEPTH     (DEPTH     )
) submodule_inst (
    .clk     (clk     ),
    .reset   (reset   ),
    .data_in (data_in ),
    .data_out(data_out)
);
```

例化要求：

- 每个参数和每个端口连接各占一行，不得在同一行连接多个端口。
- 连接名称必须与模块端口含义一致。
- 未使用输出必须明确悬空，或连接到命名清晰的未使用信号。
- 不得保留无意义的中间连线。

## 10. 注释与可综合性

### 10.1 注释

文件头注释应说明模块用途、输入输出行为、数据格式、时序或握手规则、重要参数和关键设计假设。

功能块注释应解释设计意图和原因，不得只是重复代码本身。涉及时间、频率、长度和地址时必须明确单位，例如：

```verilog
// Period, unit: us.
// Clock frequency, unit: Hz.
// Payload length, unit: bytes.
```

### 10.2 可综合性

可综合RTL中禁止：

- 使用`#delay`或`wait`实现硬件功能。
- 使用无界循环。
- 使用仿真文件读写实现逻辑。
- 使用`$display`或`$finish`实现RTL功能。
- 同一个信号由多个过程块驱动。
- 组合逻辑形成环路。
- 依赖未初始化寄存器。
- 依赖仿真器特有行为。
- 使用工程工具链未明确支持的SystemVerilog语法。

## 11. Testbench规范

Testbench文件使用`tb_`前缀，例如`tb_uart_tx.v`。

Testbench必须包含：

- 时钟和复位生成。
- 合法、边界和错误场景。
- 自动结果检查。
- 明确的PASS/FAIL输出。
- 调试所需的波形文件。

同步输入激励应在`clk`上升沿使用非阻塞赋值更新，DUT从下一个上升沿开始采样，以避免Testbench与DUT之间的仿真竞争。异步复位可以在任意时刻拉高，释放应与目标时钟边沿对齐。

```verilog
initial begin
    $dumpfile("simulation.vcd");
    $dumpvars(0, tb_top);

    reset = 1'b1;
    repeat (3) begin
        @(posedge clk);
    end

    reset <= 1'b0;

    // Test stimulus and checks

    if (error_count == 0) begin
        $display("PASS");
    end
    else begin
        $display("FAIL: %0d errors", error_count);
    end

    $finish;
end
```

业务仿真应优先采用真实或接近真实的系统时钟、通信速率、数据长度、配置周期以及使能和禁用时序。

## 12. 代码审查检查表

提交代码前检查：

- [ ] 文件、模块、参数、端口和信号命名符合规范，打拍及边沿后缀与实际行为一致。
- [ ] 参数、端口、声明和赋值已按相关功能组对齐，循环及条件表达式换行一致。
- [ ] 功能相关代码保持相邻，并按信号产生、处理、状态更新和输出顺序组织。
- [ ] 每个信号只有一个驱动源，每个寄存器只由一个时序块赋值。
- [ ] 时序逻辑使用非阻塞赋值，组合逻辑使用阻塞赋值。
- [ ] 所有条件分支使用`begin/end`，组合逻辑不存在不完整赋值或组合环路。
- [ ] 复位极性和方式统一，控制寄存器及接口具有正确复位值。
- [ ] 状态机结构、默认分支、非法状态恢复和输出逻辑完整。
- [ ] 计数器、常量、算术、比较和切片的位宽正确，不存在溢出或下溢。
- [ ] 单周期脉冲和Valid/Ready握手行为明确，数据在接受前保持稳定。
- [ ] CDC同步方式正确，多bit数据未直接逐位同步。
- [ ] 模块例化使用具名连接，每个参数和端口连接各占一行。
- [ ] 不存在未使用信号、魔法数字或不可综合代码。
- [ ] 注释说明设计意图，涉及物理量时单位正确。
- [ ] Verilog-2001编译无错误和警告。
- [ ] 自检Testbench覆盖合法、边界和错误场景，并输出PASS。
- [ ] 关键场景已保留可用于调试的波形。
