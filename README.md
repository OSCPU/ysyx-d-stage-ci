# ysyx-d-stage-ci

工程目录结构说明
```python
ysyx-workbench
├── abstract-machine
├── nemu
├── npc
│   ├── ysyx_xxxxxxxx.v //xxxxxxxx为你的学号
│   ├── csrc
│   ├── testbench
│   │    └──tb.v
│   └── Makefile
├── Makefile
└── fceux-minirv-npc.bin //完成Makefile规范后在fceux下面生成
```
注意：ysyx-workbench 可以更换名称，但 CI 流程会统一重命名为该名称
  
**重要说明**

目录名称规则

    请勿用工作分支进行测试,强烈建议新开一个测试分支进行测试
    abstract-machine、nemu 和 npc 不得更换名称
    Makefile 不得更换名称

文件要求

    npc目录下必须有 ysyx_xxxxxxxx.v 文件（xxxxxxxx为你的学号）,这个文件包含所有核的模块
    顶层模块名必须为 ysyx_xxxxxxxx
    主Makefile中必须正确设置 STUID 和 STUNAME
    testbench/tb.v为
```Verilog
`timescale 1ns/1ps
module tb;
  // -------------------------- 1. 时钟/复位（严格匹配CPU顶层） --------------------------
  reg         clock;  // 对应CPU的clock
  reg         reset;  // 对应CPU的reset

  initial begin
    clock  = 1'b0;
    forever #5 clock = ~clock;  // 100MHz时钟（周期10ns）
  end

  initial begin
    reset  = 1'b1;  // 高复位（若CPU是低复位，改为0，#20置1）
    #20    reset  = 1'b0;  // 20ns释放复位
  end

  // -------------------------- 2. 合并内存定义（8位宽，IFU/LSU共享） --------------------------
  localparam MEM_BASE     = 32'h30000000;  // 统一内存基址
  localparam MEM_SIZE_B   = 1024*1024*16;         
  reg [7:0]  mem [0:MEM_SIZE_B-1];         // 8位宽字节数组（物理内存本质）

  // 加载程序+初始化内存（合并为单个数组）
  initial begin
    static integer i;
    // 1. 清零整个内存（8位字节数组）
    for (i = 0; i < MEM_SIZE_B; i = i + 1) begin
      mem[i] = 8'h00;
    end

    // 2. 加载hex文件到内存（按字节加载，适配8位数组）
    // 注意：hex文件需为纯字节流格式（无地址标记，每行2位16进制），若仍为32位字，需拆分字节
    $readmemh("test.hex", mem);

    // // 3. 调试打印：验证内存加载（打印前4字节，对应1个32位指令）
    #1;
    // $display("[MEM] 0x30000000 = 0x%02x", mem[0]);
    // $display("[MEM] 0x30000001 = 0x%02x", mem[1]);
    // $display("[MEM] 0x30000002 = 0x%02x", mem[2]);
    // $display("[MEM] 0x30000003 = 0x%02x", mem[3]);
  end

  // -------------------------- 3. CPU顶层端口连接（严格匹配） --------------------------
  // CPU输出 → TB输入
  wire [31:0] io_ifu_addr;
  wire        io_ifu_reqValid;
  wire        io_lsu_reqValid;
  wire [31:0] io_lsu_addr;
  wire        io_lsu_wen;
  wire [31:0] io_lsu_wdata;
  wire [3:0]  io_lsu_wmask;
  wire [1:0]  io_lsu_size;

  // TB输出 → CPU输入
  reg         io_ifu_respValid;
  reg [31:0]  io_ifu_rdata;
  reg         io_lsu_respValid;
  reg [31:0]  io_lsu_rdata;
  
  // -------------------------- 4. IFU响应逻辑（32位取指，从8位内存拼接） --------------------------
  wire [31:0] addr_ifu;
  assign addr_ifu = io_ifu_addr - MEM_BASE;
  always @(posedge clock) begin
    if (reset) begin
      io_ifu_respValid <= 1'b0;
      io_ifu_rdata     <= 32'b0;
    end else begin
      io_ifu_respValid <= 1'b0;
      // 仅在CPU发起IFU请求时响应（io_ifu_reqValid有效）
      if (io_ifu_reqValid) begin
        // 检查地址是否在合法内存范围（字节编址）
        if ((io_ifu_addr >= MEM_BASE) && (io_ifu_addr < MEM_BASE + MEM_SIZE_B)) begin
          io_ifu_rdata[7:0]   <= mem[addr_ifu];
          io_ifu_rdata[15:8]  <= mem[addr_ifu + 1];
          io_ifu_rdata[23:16] <= mem[addr_ifu + 2];
          io_ifu_rdata[31:24] <= mem[addr_ifu + 3];
          io_ifu_respValid    <= 1'b1;
          // $display("[IFU][%0t] Req: addr=0x%08h → Rsp: data=0x%08h", 
          //          $time, io_ifu_addr, {mem[addr_ifu+3], mem[addr_ifu+2], mem[addr_ifu+1], mem[addr_ifu]});
        end else begin
          io_ifu_rdata     <= 32'hdeadbeef;      // 地址越界返回错误值
          io_ifu_respValid <= 1'b1;
          $display("[IFU][%0t] Req: addr=0x%08h → Out of range", $time, io_ifu_addr);
        end
      end
    end
  end

    // -------------------------- 5. LSU响应逻辑（适配8位内存，支持字节/半字/字操作） --------------------------
wire [31:0] addr_lsu;
assign addr_lsu = {io_lsu_addr[31:2],2'b0} - MEM_BASE;  // 内存访问仍用>>2，串口访问会跳过此逻辑
// 定义串口地址（匹配C代码中的SERIAL_PORT）
localparam SERIAL_PORT = 32'h10000000;

always @(posedge clock) begin
  if (reset) begin
    io_lsu_respValid <= 1'b0;
    io_lsu_rdata     <= 32'b0;
  end else begin
    // 仅在CPU发起LSU请求时响应（io_lsu_reqValid有效）
    if (io_lsu_reqValid) begin
      // 处理串口相关地址
      if (io_lsu_addr == SERIAL_PORT && io_lsu_wen) begin
        // 模拟串口输出字符
        $write("%c", io_lsu_wdata[7:0]);
        // $display("uart out");
        io_lsu_respValid <= 1'b1;  // 串口写操作响应
      end
      // 新增：串口状态寄存器（SERIAL_PORT+5）
      else if (io_lsu_addr == SERIAL_PORT + 32'd5 && !io_lsu_wen) begin
        io_lsu_rdata <= 32'h20202020;   // 返回状态：发送空闲
        io_lsu_respValid <= 1'b1;
        // $display("[UART][%0t] 读取状态寄存器: 0x%08h", $time, 32'h00000020);
      end
      // 其他串口寄存器（简单响应）
      else if (((io_lsu_addr == SERIAL_PORT+32'd3 && io_lsu_wen) || 
               (io_lsu_addr == SERIAL_PORT+32'd1 && io_lsu_wen))) begin
        io_lsu_respValid <= 1'b1;
      end
      // 原有逻辑：普通内存访问（保留>>2的地址计算）
      else if ((io_lsu_addr >= MEM_BASE) && (io_lsu_addr < MEM_BASE + MEM_SIZE_B)) begin
        // 写操作（sb/sh/sw）：按wmask逐字节写入8位内存
        if (io_lsu_wen) begin
          if (io_lsu_wmask[0]) mem[addr_lsu]     <= io_lsu_wdata[7:0];
          if (io_lsu_wmask[1]) mem[addr_lsu + 1] <= io_lsu_wdata[15:8];
          if (io_lsu_wmask[2]) mem[addr_lsu + 2] <= io_lsu_wdata[23:16];
          if (io_lsu_wmask[3]) mem[addr_lsu + 3] <= io_lsu_wdata[31:24];
          // $display("[LSU][%0t] Write: addr=0x%08h wdata=0x%08h wmask=0x%1h size=%02b pc:0x%08h " ,
                  //  $time, io_lsu_addr, io_lsu_wdata, io_lsu_wmask, io_lsu_size , addr_lsu, io_ifu_addr);
          io_lsu_respValid <= 1'b1;  // 写操作响应
        end
        // 读操作（lb/lh/lw/lbu/lhu）：从8位内存拼接数据
        else begin
          // 默认拼接32位（CPU内部LSU根据size做截断/符号扩展）
          io_lsu_rdata[7:0]   <= mem[addr_lsu];
          io_lsu_rdata[15:8]  <= mem[addr_lsu + 1];
          io_lsu_rdata[23:16] <= mem[addr_lsu + 2];
          io_lsu_rdata[31:24] <= mem[addr_lsu + 3];
          io_lsu_respValid    <= 1'b1;
          // $display("[LSU][%0t] Read: addr=0x%08h rdata=0x%08h size=%02b pc:0x%08h ", 
                  //  $time, io_lsu_addr, {mem[addr_lsu+3], mem[addr_lsu+2], mem[addr_lsu+1], mem[addr_lsu]}, io_lsu_size, io_ifu_addr);
        end
      end else begin
          io_lsu_rdata     <= 32'hdeadbeef;  // 地址越界
          io_lsu_respValid <= 1'b1;
          $display("[LSU][%0t] Req: addr=0x%08h → Out of range pc:0x%08h" , $time, io_lsu_addr ,io_ifu_addr);
      end
    end else begin
          io_lsu_respValid <= 1'b0;
    end
  end
end

  // -------------------------- 6. 例化CPU顶层（100%匹配端口） --------------------------
  ysyx_25080202 u_cpu_top (
    .clock          (clock),
    .reset          (reset),
    // IFU接口
    .io_ifu_respValid(io_ifu_respValid),
    .io_ifu_rdata   (io_ifu_rdata),
    .io_ifu_addr    (io_ifu_addr),
    .io_ifu_reqValid(io_ifu_reqValid),
    // LSU接口
    .io_lsu_rdata   (io_lsu_rdata),
    .io_lsu_respValid(io_lsu_respValid),
    .io_lsu_reqValid(io_lsu_reqValid),
    .io_lsu_addr    (io_lsu_addr),
    .io_lsu_wen     (io_lsu_wen),
    .io_lsu_wdata   (io_lsu_wdata),
    .io_lsu_wmask   (io_lsu_wmask),
    .io_lsu_size    (io_lsu_size)
  );
  always @(posedge clock) begin
    if (!reset) begin  
      if (io_ifu_respValid && (io_ifu_rdata == 32'h00100073)) begin
        $display("[TB][%0t] 检测到ebreak指令(0x00100073)，结束仿真", $time);
        #20 $finish;
      end
    end
  end


  // -------------------------- 7. 波形导出（调试用） --------------------------
  initial begin
    // $dumpfile("wave.vcd");
    // $dumpvars(0, tb);                // 导出TB所有信号
    // $dumpvars(1, u_cpu_top);         // 导出CPU顶层所有信号
  end

endmodule
```

Makefile规范

    npc/Makefile文件路径需相对于环境变量 $YSYX_HOME（指向 ysyx-workbench 目录）
    将ysyx-workbench/abstract-machine/scripts/platform/npc.mk的image目标和LDFLAGS的--defsym改成以下内容
```Makefile

IVERILOG ?= n
ifeq ($(IVERILOG), y)

LDFLAGS += --defsym=_pmem_start=0x30000000 --defsym=_entry_offset=0x0
image: image-dep
	@$(OBJDUMP) -d $(IMAGE).elf > $(IMAGE).txt
	@echo + OBJCOPY "->" $(IMAGE_REL).bin
	@$(OBJCOPY) -S --set-section-flags .bss=alloc,contents -O binary $(IMAGE).elf $(IMAGE).bin

else

LDFLAGS += --defsym=_pmem_start=0x80000000 --defsym=_entry_offset=0x0
ELF_OFFSET := 370432
HELLO_TEMPLATE := $(YSYX_HOME)/ysyxSoC/ready-to-run/D-stagehello-minirv-ysyxsoc.bin
IMAGE_BIN := $(IMAGE).bin
$(IMAGE_BIN): $(IMAGE).elf $(HELLO_TEMPLATE)
	@echo "  PATCH  $@"
	@cp $(HELLO_TEMPLATE) $@.tmp
	@dd if=$< of=$@.tmp bs=1 seek=$(ELF_OFFSET) conv=notrunc 2>/dev/null
	@mv $@.tmp $@
image: $(IMAGE_BIN)
	@$(OBJDUMP) -d $(IMAGE).elf > $(IMAGE).txt

endif

```
	在npc/Makefile下添加以下内容
```Makefile
IVL_TOP_MODULE ?= tb
IVL_SRC_VER = $(YSYX_HOME)/npc/ysyx_xxxxxxxx.v 改成你的学号
IVL_SRC_TB = $(shell find ./testbench -name "*.v")
IVL_SRC  = $(IVL_SRC_VER) $(IVL_SRC_TB)
IVL_SRC_NET = $(NETLIST) $(CELLS) $(IVL_SRC_TB)
VVP_FILE   = sim.vvp
VCD_FILE   = wave.vcd
IVL_FLAGS  = -g2012 -Wall -Winfloop -DCOSIM_XCHECK  # -g2012启用Verilog 2012标准，-Wall开启警告
VVP_FLAGS  = -v -n  # -v显示详细仿真日志，-n启用四值仿真严格检查
IMG_HEX    = test.hex 
LOAD_ADDR ?= 0
OBJCOPY    = riscv64-linux-gnu-objcopy
sim-iverilog:
	$(OBJCOPY) -I binary -O verilog --adjust-vma=$(LOAD_ADDR) $(IMG) $(IMG_HEX)
	@echo "========== 编译Verilog文件（启用四值仿真）=========="
	@echo $(IVL_SRC)
	iverilog $(IVL_FLAGS) -s $(IVL_TOP_MODULE) -o $(VVP_FILE) $(IVL_SRC)
	@echo "编译完成，生成文件：$(VVP_FILE)"
	@echo "========== 执行四值仿真 =========="
	vvp $(VVP_FLAGS) $(VVP_FILE)
# 	@echo "仿真完成，生成波形文件：$(VCD_FILE)"
# 	@echo "========== 打开波形文件 =========="
# 	gtkwave $(VCD_FILE) 
sim-iverilog-netlist:
	@$(OBJCOPY) -I binary -O verilog --adjust-vma=$(LOAD_ADDR) $(IMG) $(IMG_HEX)
	@echo "========== 编译Verilog文件（启用四值仿真）=========="
	@echo $(IVL_SRC_NET)
	iverilog $(IVL_FLAGS) -s $(IVL_TOP_MODULE) -o $(VVP_FILE) $(IVL_SRC_NET)
	@echo "编译完成，生成文件：$(VVP_FILE)"
	@echo "========== 执行四值仿真 =========="
	vvp $(VVP_FLAGS) $(VVP_FILE)
#	@echo "仿真完成，生成波形文件：$(VCD_FILE)"
#	@echo "========== 打开波形文件 =========="
#	gtkwave $(VCD_FILE) 
```
	

CI流程要求
特别注意：

    关闭 DiffTest
    关闭波形和调试功能
    关于反汇编的部分必须关闭
    Makefile 中关于反汇编的编译和链接选项需要调整

本地测试内容:

    要在本地通过以下测试:
    1.am-kernels/kernels/hello 下运行 make ARCH=minirv-npc run 成功输出
    2.am-kernels/tests/am-tests 下运行 make ARCH=minirv-npc run mainargs=t 使得测试程序输出信息的间隔接近1秒
    3.am-kernels/tests/cpu-tests 下运行 make ARCH=minirv-npc run 通过所有测试
    4.am-kernels/benchmarks/microbench 下运行 make ARCH=minirv-npc run 通过
    5.npc/ 下运行 make IMG=$YSYX_HOME/fceux-minirv-npc.bin run mainargs=mario 成功启动马里奥
    还有需要克隆的:
    git clone --depth 1 https://github.com/NJU-ProjectN/riscv-tests-am
    6.运行 make ARCH=minirv-npc -C riscv-tests-am run TEST_ISA=i 成功通过测试
    以下是iverilog仿真:(iverilog仿真禁止用DPI-C,可以用VERILATOR宏包裹DPI-C相关代码,方便verilator和iverilog都能仿真)(参考一生一芯讲义B5/流片前的准备工作/四值仿真和网表仿真)
    7.在$YSYX_HOME/am-kernels/benchmarks/microbench下make ARCH=minirv-npc IVERILOG=y 后生成对应的.bin然后在 npc/ 下运行 make -C sim-iverilog IMG=$YSYX_HOME/am-kernels/benchmarks/microbench/build/microbench-minirv-npc.bin 无x态传播并成功通过(四值仿真)
    克隆git clone -b ci https://github.com/OSCPU/yosys-sta并进行
```Makefile
cd yosys-sta
make init
echo exit | ./bin/iEDA -v
```
    8.在yosys-sta下面 make sta DESIGN=ysyx_xxxxxxxx CLK_FREQ_MHZ=500 CLK_PORT_NAME=clock RTL_FILES=$YSYX_HOME/npc/ysyx_xxxxxxxx.v 其中xxxxxxxx为你的学号,运行之后检查synth_stat.txt中是否包含以下标准单元: DLL_X1, DLL_X2, DLH_X1, DLH_X2. 若有, 说明你的设计中存在锁存器, 你需要修改相应模块中的代码去除它们.
    9.在$YSYX_HOME/am-kernels/benchmarks/microbench下make ARCH=minirv-npc IVERILOG=y 后生成对应的.bin然后在 npc/ 下运行 make sim-iverilog-netlist IMG=$YSYX_HOME/am-kernels/benchmarks/microbench/build/microbench-minirv-npc.bin NETLIST=$YSYX_HOME/yosys-sta/result/ysyx_xxxxxxxx-500MHz/ysyx_xxxxxxxx.netlist.fixed.v CELLS=$YSYX_HOME/yosys-sta/pdk/nangate45/sim/cells.v 其中xxxxxxxx为你的学号,运行之后无x态传播并成功通过(网表仿真)

提交步骤

    在 GitHub 上创建新的远程仓库
    将代码按照工程目录结构组织并上传到仓库
    在 https://github.com/OSCPU/ysyx-d-stage-ci 仓库中点击 issue 页面
    点击右上角的 New issue
    选择 "D阶段考核提交表单"
    在标题或注释中填写提交人信息
    创建 issue 后，点击 Actions 查看 CI 进度


已完成

    hello
    time
    cpu-tests
    riscv-tests
    riscv-arch-tests的部分测试
    microbench
    mario
    四值仿真
    锁存器检测
    网表仿真

问题

    riscv-arch-tests部分通过不了的测试原因为经minirv特殊的编译器编译之后运行一些标签的函数会跳转到0x0000000从而循环,还有访存0x01fddb79超限

