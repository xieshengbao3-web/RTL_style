# Verilog RTL编码与格式规范

## 1. 适用范围

本规范适用于使用Verilog-2001开发的可综合RTL代码，包括：

- FPGA和ASIC逻辑模块；
- 时序控制模块；
- 数据通路；
- 状态机；
- 通信接口；
- 寄存器和计数器；
- 模块级及系统级顶层。

测试平台可以使用仿真专用语句，但可综合RTL中不得依赖仿真专用代码实现功能。

## 2. 基本原则

1. RTL结构应简单、明确和可综合。
2. 一个信号只能有一个驱动源。
3. 一个寄存器只能由一个时序 `always` 块赋值。
4. 每个 `always` 块只承担一个明确功能。
5. 功能无关或更新条件不同的寄存器应拆分到不同的 `always` 块。
6. 时序逻辑使用非阻塞赋值 `<=`。
7. 组合逻辑使用阻塞赋值 `=`。
8. 所有组合逻辑必须覆盖完整赋值路径，避免产生锁存器。
9. 所有计数器、索引和算术运算必须明确考虑位宽和溢出。
10. 代码格式在同一工程中保持一致。
11. 功能相关的代码应相邻，并按照信号产生到结果输出的因果顺序排列。
12. 相关代码组内的声明和赋值符号应按列对齐。

## 3. 文件格式

RTL文件建议使用以下结构：

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
    // Related continuous assignments
    // Related combinational logic
    // Related sequential logic
    // Related submodule instances

    // Functional block B
    // Related logic and submodule instances

endmodule
```

格式要求：

- 使用4个空格缩进，不使用Tab。
- 每行只写一条主要语句。
- 文件末尾保留换行。
- 不在行尾保留无意义空格。
- 相关代码之间不插入过多空行。
- 不使用不可见字符或非标准标点。

### 3.1 功能组织和阅读顺序

模块内的参数、常量和信号声明统一放在模块前部，并按照功能分组。声明完成后，逻辑代码按照功能组织，而不是将整个模块机械地拆分成互相远离的 `assign`、组合逻辑、时序逻辑和实例化区域。

组织要求：

- 功能相近、存在直接数据关系或控制关系的代码应尽量相邻。
- 每个功能组按照“条件或数据产生、中间处理、状态更新、结果输出”的因果顺序排列。
- 产生某个中间信号的代码原则上放在使用该信号的代码之前。
- 同一功能的 `assign`、组合逻辑、时序逻辑和子模块实例应保持在同一区域。
- 不相关功能之间使用空行和功能注释分隔。
- 不为了靠近代码而将参数或信号声明分散到模块中部。

示例：

```verilog
wire request_accept;
wire transfer_done ;

reg  busy          ;
reg  done          ;

assign request_accept = request_valid && request_ready ;
assign transfer_done  = busy && engine_done            ;

always @(posedge clk or posedge reset) begin
    if (reset) begin
        busy <= 1'b0;
    end
    else if (request_accept) begin
        busy <= 1'b1;
    end
    else if (transfer_done) begin
        busy <= 1'b0;
    end
end

always @(posedge clk or posedge reset) begin
    if (reset) begin
        done <= 1'b0;
    end
    else begin
        done <= transfer_done;
    end
end
```

## 4. 命名规范

### 4.1 模块和文件

- 文件名与顶层模块名保持一致。
- 模块名使用小写下划线格式。

```text
uart_tx_8n1.v
uart_tx_8n1
```

### 4.2 参数和常量

参数和本地常量使用大写下划线格式：

```verilog
parameter integer CLK_FREQ_HZ = 96000000;
parameter integer DATA_WIDTH  = 8       ;

localparam integer FRAME_BYTES = PAYLOAD_BYTES + 6;
localparam [7:0]   SOF_BYTE    = 8'hA5            ;
```

同一功能组内的参数和本地常量应对齐名称、赋值符号和行尾分号。不同功能组不要求跨组对齐。

### 4.3 信号和寄存器

信号使用小写下划线格式：

```verilog
frame_start
byte_index
payload_latched
crc_value
```

推荐后缀：

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
| `_index` | 索引 |
| `_latched` | 锁存值 |
| `_next` | 下一计算值 |
| `_1d` | 延迟一个目标时钟周期或同步链第一级 |
| `_2d` | 延迟两个目标时钟周期或同步链第二级 |
| `_3d` | 延迟三个目标时钟周期；更多级数按相同规则扩展 |
| `_pos` | 检测到上升沿时产生的单周期脉冲 |
| `_neg` | 检测到下降沿时产生的单周期脉冲 |
| `_n` | 低有效信号 |

打拍寄存器的级数后缀必须与实际寄存器级数一致，并从 `_1d` 开始连续命名，不得跳级或使用同一后缀表示不同延迟级数：

```text
data_valid_1d
data_valid_2d
data_valid_3d
```

同步链与普通打拍寄存器统一使用 `_1d`、`_2d`、`_3d` 等级数后缀。

`_pos` 和 `_neg` 只表示边沿检测产生的单周期脉冲，不表示寄存器使用 `posedge` 或 `negedge` 采样。

同一个概念应使用同一种缩写，禁止在同一工程中混用类似名称：

```text
frame / frm
count / cnt
enable / en
```

## 5. 参数和端口声明

参数每行一个：

```verilog
module example #(
    parameter integer DATA_WIDTH = 8 ,
    parameter integer DEPTH      = 16
) (
```

端口按照以下顺序分组：

1. 时钟和复位；
2. 配置输入；
3. 数据输入；
4. 控制和握手输入；
5. 数据输出；
6. 状态和诊断输出。

```verilog
    input  wire                  clk       ,
    input  wire                  reset     ,

    input  wire [15:0]           cfg_period,

    input  wire [DATA_WIDTH-1:0] data_in   ,
    input  wire                  data_valid,

    output reg  [DATA_WIDTH-1:0] data_out  ,
    output reg                   data_done
);
```

要求：

- 一个端口占一行。
- 参数化位宽使用括号明确运算优先级。
- 端口方向必须显式写出 `input` 或 `output`。
- Verilog-2001中显式使用 `wire` 或 `reg`。
- 端口名称应表达信号方向和功能。
- 同一相关端口组内应对齐方向、类型、位宽、名称和行尾逗号；最后一个端口不添加多余逗号。
- 参数列表中的名称、`=`和行尾逗号应在相关参数组内对齐；最后一个参数不添加多余逗号。

## 6. 信号声明

相关信号分组声明：

```verilog
reg  [7:0]  byte_index     ;
reg  [15:0] crc_value      ;
reg  [15:0] period_counter ;

wire        byte_complete  ;
wire        frame_complete ;
wire [15:0] crc_next       ;
```

要求：

- 寄存器使用 `reg`。
- 组合连接和模块输出连接使用 `wire`。
- 不保留未使用的信号。
- 信号位宽应满足功能需要，但避免无理由使用32 bit或64 bit。
- 协议字段宽度应与协议定义一致。
- 同一功能组内的声明应对齐类型、位宽、名称和行尾分号。
- 不同功能组应通过空行分隔，不为跨组对齐而加入大量空格。

## 7. 时序逻辑

### 7.1 基本格式

时序逻辑使用非阻塞赋值：

```verilog
always @(posedge clk or posedge reset) begin
    if (reset) begin
        counter <= 16'd0;
    end
    else if (counter_en) begin
        counter <= counter + 1'b1;
    end
end
```

条件分支格式要求：

- `if`、`else if`和`else`分支必须使用 `begin/end`，即使分支中只有一条语句也不能省略。
- `begin`与条件写在同一行。
- `else if`和`else`单独换行，并与对应的 `if`保持同一级缩进。
- `end`与所属的 `if`、`else if`或 `else`保持同一级缩进。
- 同一个条件链中的语句和赋值必须保持统一缩进。

推荐格式：

```verilog
always @(posedge S_AXI_ACLK) begin
    if (!S_AXI_ARESETN) begin
        S_AXI_BVALID <= 1'b0;
    end
    else if (write_commit) begin
        S_AXI_BVALID <= 1'b1;
    end
    else if (S_AXI_BVALID && S_AXI_BREADY) begin
        S_AXI_BVALID <= 1'b0;
    end
end
```

禁止省略单语句分支的 `begin/end`：

```verilog
if (write_commit)
    S_AXI_BVALID <= 1'b1;
else if (S_AXI_BREADY)
    S_AXI_BVALID <= 1'b0;
```

禁止：

```verilog
always @(posedge clk)
    counter = counter + 1'b1;
```

### 7.2 always块职责

默认一个寄存器使用一个时序 `always` 块。

允许多个寄存器放入同一个块的情况：

- 同一同步链中的多级寄存器；
- 同一流水线级的数据和有效标志；
- 必须原子更新的一组寄存器；
- 复位、使能和更新条件完全一致的相关寄存器。

示例：

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

不同功能应拆分：

```verilog
always @(posedge clk or posedge reset) begin
    if (reset) begin
        counter <= 16'd0;
    end
    else if (counter_en) begin
        counter <= counter + 1'b1;
    end
end

always @(posedge clk or posedge reset) begin
    if (reset) begin
        done <= 1'b0;
    end
    else begin
        done <= complete;
    end
end
```

### 7.3 优先级

条件优先级一般为：

1. 复位；
2. 清除或禁止；
3. 装载；
4. 功能更新；
5. 保持原值。

优先级必须通过 `if / else if / else`明确表达。

## 8. 复位规范

同一模块内必须使用统一的复位极性和复位方式。

高有效异步复位：

```verilog
always @(posedge clk or posedge reset) begin
    if (reset) begin
        state <= STATE_IDLE;
    end else begin
        state <= state_next;
    end
end
```

低有效异步复位：

```verilog
always @(posedge clk or negedge reset_n) begin
    if (!reset_n) begin
        state <= STATE_IDLE;
    end else begin
        state <= state_next;
    end
end
```

要求：

- 不在同一模块中混用不同复位形式。
- 所有控制寄存器必须有明确复位值。
- 输出接口复位值应符合协议空闲状态。
- 异步复位释放应由系统级设计保证同步。

## 9. 组合逻辑

组合逻辑使用：

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

要求：

- 使用阻塞赋值。
- 在块开始处提供默认值，或通过完整分支覆盖所有路径。
- 不在组合块中保留上一次值。
- 不在组合逻辑中使用时钟或延时语句。
- 一个组合输出只能由一个组合块驱动。

禁止可能产生锁存器的写法：

```verilog
always @(*) begin
    if (enable) begin
        data_out = data_in;
    end
end
```

应改为：

```verilog
always @(*) begin
    if (enable) begin
        data_out = data_in;
    end
    else begin
        data_out = 8'd0;
    end
end
```

## 10. 连续赋值

简单组合关系使用 `assign`：

```verilog
assign byte_complete  = byte_done && (byte_index == LAST_BYTE_INDEX);
assign frame_complete = byte_complete && crc_match                  ;
```

以下情况优先使用连续赋值：

- 简单逻辑表达式；
- 比较结果；
- 状态组合条件；
- 模块之间的直接连接；
- 不需要过程控制的算术逻辑。

表达式换行要求：

- 较短的条件、比较和连续赋值表达式应写在同一行，不为了逐项显示条件而强制换行。
- 只有表达式在一行中过长、条件数量较多或确实需要体现逻辑分组时才允许换行。
- 同一文件应采用一致的换行风格，禁止相同复杂度的表达式有时单行、有时拆成多行。
- 同一功能组内的多条 `assign` 应对齐左值、`=`和行尾分号。
- 对齐只在相关代码组内进行，不跨越空行、功能注释或无关逻辑。
- 长表达式换行时优先保证逻辑分组和缩进清晰，不为对齐行尾分号而产生过长代码行。

推荐：

```verilog
assign write_commit = axi_awaddr_valid && axi_wdata_valid && !axi_bvalid;
```

不推荐将较短表达式拆成多行：

```verilog
assign write_commit =
    axi_awaddr_valid &&
    axi_wdata_valid &&
    !axi_bvalid;
```

较长表达式需要换行时，应按逻辑关系分组并保持缩进一致：

```verilog
assign frame_accept = frame_valid && length_match &&
                      crc_match && sequence_match;
```

## 11. 状态机

状态使用有意义的常量名称：

```verilog
localparam [1:0] STATE_IDLE = 2'd0;
localparam [1:0] STATE_SEND = 2'd1;
localparam [1:0] STATE_WAIT = 2'd2;
```

推荐将状态寄存器和下一状态组合逻辑分开：

```verilog
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
            if (done) begin
                state_next = STATE_IDLE;
            end
        end

        default: begin
            state_next = STATE_IDLE;
        end
    endcase
end
```

要求：

- `case`必须包含 `default`。
- 非法状态应恢复到安全状态。
- 状态输出不得形成组合环路。

## 12. 计数器和位宽

计数器位宽必须覆盖最大计数值：

```verilog
reg [15:0] period_counter;
```

所有常量建议明确位宽：

```verilog
1'b0
1'b1
8'd0
16'd0
16'hFFFF
```

算术要求：

- 检查加法、乘法和移位是否溢出。
- 检查减1时是否可能发生无符号下溢。
- 比较双方应使用兼容位宽。
- 不依赖工具对有符号和无符号数的隐式转换。
- 参数参与切片或索引时必须确认合法范围。

参数化部分选择示例：

```verilog
payload[(byte_index*8) +: 8]
```

## 13. 脉冲和握手

单周期完成脉冲推荐写法：

```verilog
always @(posedge clk or posedge reset) begin
    if (reset) begin
        done <= 1'b0;
    end
    else begin
        done <= complete;
    end
end
```

同步信号的边沿检测推荐写法：

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

要求：

- `signal`必须已经同步到当前时钟域。
- `signal_pos`只在检测到上升沿时保持一个时钟周期。
- `signal_neg`只在检测到下降沿时保持一个时钟周期。
- 边沿检测历史值使用 `_1d`，上升沿和下降沿脉冲分别使用 `_pos`和 `_neg`。

Valid/Ready握手规则：

```text
transfer = valid && ready
```

要求：

- 启动请求必须在接收模块可接受时发出。
- 忙状态期间的新请求必须明确规定为忽略、缓存或报错。
- 脉冲宽度必须明确为一个时钟周期。
- 数据必须在请求被接受前保持稳定。

## 14. 时钟域跨越

异步单bit输入至少使用两级同步器：

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

多bit数据不得直接对每一位分别同步。应根据场景使用：

- 握手；
- 异步FIFO；
- Gray码计数器；
- 数据保持配合Valid同步。

## 15. 模块实例化

所有参数和端口使用具名连接：

```verilog
submodule #(
    .DATA_WIDTH(DATA_WIDTH),
    .DEPTH(DEPTH)
) submodule_inst (
    .clk(clk),
    .reset(reset),
    .data_in(data_in),
    .data_out(data_out)
);
```

禁止位置连接：

```verilog
submodule submodule_inst (
    clk,
    reset,
    data_in,
    data_out
);
```

实例化要求：

- 一个参数或端口占一行。
- 连接名称和模块端口含义一致。
- 未使用输出应明确悬空或连接到命名清晰的未使用信号。
- 不保留无意义的中间连线。

## 16. 注释规范

文件头注释说明：

- 模块用途；
- 输入输出行为；
- 数据格式；
- 时序或握手规则；
- 重要参数；
- 关键设计假设。

功能块注释说明设计意图：

```verilog
// Latch the input data when a new transaction starts.
```

避免重复代码本身：

```verilog
// Add one to counter.
counter <= counter + 1'b1;
```

涉及时间、频率、长度和地址时必须明确单位：

```verilog
// Period, unit: us.
// Clock frequency, unit: Hz.
// Payload length, unit: bytes.
```

## 17. 可综合RTL禁止事项

可综合RTL中禁止：

- 使用 `#delay`实现功能；
- 使用 `wait`实现硬件控制；
- 使用无界循环；
- 使用仿真文件读写实现逻辑；
- 使用 `$display`、`$finish`实现RTL功能；
- 同一个信号由多个过程块驱动；
- 组合逻辑形成环路；
- 依赖未初始化寄存器；
- 依赖仿真器特有行为；
- 使用未明确支持的SystemVerilog语法。

## 18. Testbench规范

Testbench文件使用 `tb_`前缀：

```text
tb_uart_tx.v
tb_frame_parser.v
```

Testbench应包含：

- 时钟和复位生成；
- 合法场景；
- 边界场景；
- 错误场景；
- 自动结果检查；
- 明确的PASS/FAIL输出；
- 必要的波形文件。

示例：

```verilog
initial begin
    $dumpfile("simulation.vcd");
    $dumpvars(0, tb_top);

    reset = 1'b1;
    repeat (3)
        @(posedge clk);

    reset = 1'b0;

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

业务仿真优先使用真实或接近真实的：

- 系统时钟；
- 通信速率；
- 数据长度；
- 配置周期；
- 使能和禁用时序。

## 19. 代码审查检查表

提交代码前检查：

- [ ] 文件名与模块名一致。
- [ ] 所有端口方向和位宽正确。
- [ ] 所有寄存器只有一个驱动源。
- [ ] 时序逻辑全部使用非阻塞赋值。
- [ ] 组合逻辑全部使用阻塞赋值。
- [ ] 所有 `if / else if / else`分支均使用 `begin/end`并正确对齐。
- [ ] 组合逻辑不存在不完整赋值。
- [ ] 复位极性和复位方式统一。
- [ ] 计数器位宽足够且不过度。
- [ ] 不存在加减法下溢或溢出。
- [ ] 单周期脉冲宽度正确。
- [ ] 打拍寄存器使用连续且与实际级数一致的 `_1d`、`_2d`、`_3d` 后缀。
- [ ] CDC同步链使用与实际同步级数一致的级数后缀。
- [ ] 上升沿和下降沿检测脉冲分别使用 `_pos`和 `_neg`，且脉冲宽度为一个时钟周期。
- [ ] 模块握手关系明确。
- [ ] 参数和协议常量没有魔法数字。
- [ ] 所有模块使用具名端口连接。
- [ ] 不存在未使用信号。
- [ ] 注释中的单位和实际接口一致。
- [ ] 功能相关的代码保持相邻，不相关功能之间有明确分隔。
- [ ] 代码按照条件或数据产生、中间处理、状态更新和结果输出的因果顺序排列。
- [ ] 相关代码组内的参数、声明和 `assign` 已对齐名称、赋值符号及行尾分隔符。
- [ ] 较短表达式保持单行，长表达式换行和缩进一致。
- [ ] Verilog-2001编译无错误和警告。
- [ ] 自检Testbench输出PASS。
- [ ] 关键场景已保留波形。
