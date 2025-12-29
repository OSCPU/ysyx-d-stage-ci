# ysyx-d-stage-ci
目录结构:
ysyx-workbench
├── abstract-machine
├── nemu
├── npc
│   ├── ysyx_xxxxxxxx.v
│   └── Makefile
├── Makefile
└── patch
    └── ysyxSoC

ysyx-workbench可以更换名称, CI流程从远程仓库拉取你的代码时, 会自动将工程目录统一更名为ysyx-workbench. 因此, 我们对远程仓库的名称不作要求.
abstract-machine, nemu和npc不得更换名称.
Makefile不得更换名称, 其中STUID和STUNAME需要正确设置.

CI流程对make目标有如下约定:
    npc下面要有ysyx_xxxxxxxx.v文件, 其中的xxxxxxxx为你的学号.顶层模块名为ysyx_xxxxxxxx
    在npc/Makefile中, 文件路径的指定需要相对于环境变量$YSYX_HOME, 该环境变量指向ysyx-workbench目录.
    Makefile要实现在$YSYX_HOME/npc/下make run ALL=hello能运行hello并输出.make run ALL=dummy能构建并运行dummy.

CI流程需要关闭DiffTest.关闭波形和调试功能.(特别是disasm.cpp关于反汇编的部分以及Makefile里相应的编译和链接选项)

整理好之后在github上创建一个新的远程仓库将代码上传,然后点开issue,右上角有一个绿色的New issue,点开选择D阶段考核提交表单,要在标题或者注释填写提交人,Create之后点Actions查看对应的进度
