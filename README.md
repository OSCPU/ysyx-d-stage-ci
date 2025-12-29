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
重要说明

目录名称规则

    abstract-machine、nemu 和 npc 不得更换名称
    Makefile 不得更换名称

文件要求

    npc目录下必须有 ysyx_xxxxxxxx.v 文件（xxxxxxxx为你的学号）,这个文件包含所有核的模块
    顶层模块名必须为 ysyx_xxxxxxxx
    主Makefile中必须正确设置 STUID 和 STUNAME

Makefile规范

    文件路径需相对于环境变量 $YSYX_HOME（指向 ysyx-workbench 目录）
    必须实现以下功能：
        make run ALL=hello # 能运行hello并输出 
        make run ALL=dummy # 能构建并运行dummy

CI流程要求
特别注意：

    关闭 DiffTest
    关闭波形和调试功能
    disasm.cpp 中关于反汇编的部分必须关闭
    Makefile 中关于反汇编的编译和链接选项需要调整

提交步骤

    在 GitHub 上创建新的远程仓库
    将代码上传到仓库
    在 https://github.com/OSCPU/ysyx-d-stage-ci 仓库中点击 issue 页面
    点击右上角的 New issue
    选择 "D阶段考核提交表单"
    在标题或注释中填写提交人信息
    创建 issue 后，点击 Actions 查看 CI 进度


