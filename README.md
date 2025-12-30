# ysyx-d-stage-ci

工程目录结构说明
```python
ysyx-workbench
├── abstract-machine
├── nemu
├── npc
│   ├── ysyx_xxxxxxxx.v
│   └── Makefile
├── Makefile
└── patch
    └── ysyxSoC
```
注意：ysyx-workbench 可以更换名称，但 CI 流程会统一重命名为该名称

    大概运行45min左右

**重要说明**

目录名称规则

    请勿用工作分支进行测试,强烈建议新开一个测试分支进行测试
    abstract-machine、nemu 和 npc 不得更换名称
    Makefile 不得更换名称

文件要求

    npc目录下必须有 ysyx_xxxxxxxx.v 文件（xxxxxxxx为你的学号）,这个文件包含所有核的模块
    顶层模块名必须为 ysyx_xxxxxxxx
    主Makefile中必须正确设置 STUID 和 STUNAME

Makefile规范

    npc/Makefile文件路径需相对于环境变量 $YSYX_HOME（指向 ysyx-workbench 目录）
    将ysyx-workbench/abstract-machine/scripts/platform/npc.mk的image目标改成以下内容
```Makefile
# image: image-dep
# 	@$(OBJDUMP) -d $(IMAGE).elf > $(IMAGE).txt
# 	@echo + OBJCOPY "->" $(IMAGE_REL).bin
# 	@$(OBJCOPY) -S --set-section-flags .bss=alloc,contents -O binary $(IMAGE).elf $(IMAGE).bin
# 偏移量换算成 10 进制，dd 只认 10 进制

ELF_OFFSET := 370432

# 源模板文件（对应你 gen.sh 里的 HELLO_BIN）
HELLO_TEMPLATE := $(YSYX_HOME)/ysyxSoC/ready-to-run/D-stage/hello-minirv-ysyxsoc.bin

# 最终要生成的 bin
IMAGE_BIN := $(IMAGE).bin

# 真正的依赖：elf 文件变了就要重新 patch
$(IMAGE_BIN): $(IMAGE).elf $(HELLO_TEMPLATE)
	@echo "  PATCH  $@"
	@cp $(HELLO_TEMPLATE) $@.tmp
	@dd if=$< of=$@.tmp bs=1 seek=$(ELF_OFFSET) conv=notrunc 2>/dev/null
	@mv $@.tmp $@

# 如果你还想保留反汇编，可以保留原来的 image 目标
image: $(IMAGE_BIN)
	@$(OBJDUMP) -d $(IMAGE).elf > $(IMAGE).txt
``` 

CI流程要求
特别注意：

    关闭 DiffTest
    关闭波形和调试功能
    关于反汇编的部分必须关闭
    Makefile 中关于反汇编的编译和链接选项需要调整

本地测试内容:

    要在本地通过以下测试:
    am-kernels/kernels/hello运行make ARCH=minirv-npc run
    am-kernels/tests/cpu-tests运行make ARCH=minirv-npc run
    am-kernels/benchmarks/microbench下运行make ARCH=minirv-npc run 
    还有需要克隆的:
    git clone --depth 1 https://github.com/NJU-ProjectN/riscv-tests-am
    运行make ARCH=minirv-npc -C riscv-tests-am run TEST_ISA=i
    git clone --depth 1 https://github.com/NJU-ProjectN/riscv-arch-test-am
    运行make ARCH=minirv-npc -C riscv-arch-test-am run TEST_ISA="E"

提交步骤

    在 GitHub 上创建新的远程仓库
    将代码上传到仓库
    在 https://github.com/OSCPU/ysyx-d-stage-ci 仓库中点击 issue 页面
    点击右上角的 New issue
    选择 "D阶段考核提交表单"
    在标题或注释中填写提交人信息
    创建 issue 后，点击 Actions 查看 CI 进度



已完成

    hello
    cpu-tests
    riscv-tests
    microbench

待完成
    riscv-arch-test
    iverilog-microbench
    iverilog-netlist-microbench
    yosys-sta
