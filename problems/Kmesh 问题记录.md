# Kmesh 问题记录

## 关于 Kmesh

### 环境问题

1. **使用的机器**，开发使用本地台式机，远程连接远程的windows，然后在远程的windows上连接Linux服务器。
2. **开发工具 & 是否能够使用 AI？** 可以使用 curser 这样的 AI 开发工具辅助开发。
3. **如何配置 kmesh 环境？** 配置环境请先阅读 kmesh 的文档（链接：https://kmesh.net/docs/welcome）中 setup/quick-start 和 setup/develop-with-kind 中的内容。需要安装的内容如下：
   1. kind（需要先安装docker，docker的安装方法可以参考：https://docs.docker.com/engine/install/ubuntu）
   2. kubectl
   3.  istioctl
   4. helm，安装方法可以参考 https://helm.sh/zh/docs/intro/install
4. 使用`kind create cluster`的时候可以考虑减少 nodes 的数量来避免卡顿，如只设置一个 node。

### 编译问题

1. 不要单独执行 make kmesh-bpf，否则会在 ./bpf/kmesh/bpf2go/kernelnative 这个位置错误生成一些文件。
2. 一般可以直接执行make build即可

### 测试问题

1. 在进行测试之前请先阅读 https://kmesh.net/docs/developer-guide/Tests/unit-test 中的有关内容，然后再进行测试。

2. 在一次 PR 提交后，在 go test 的过程中出现了测试用例不通过的情况，但是明显有关代码并没有被修改。
``` plain
--- FAIL: TestSecurity (2.42s)
    --- PASS: TestSecurity/TestBaseCert (0.31s)
    --- FAIL: TestSecurity/TestCertRotate (2.11s)
panic: runtime error: invalid memory address or nil pointer dereference [recovered]
	panic: runtime error: invalid memory address or nil pointer dereference
[signal SIGSEGV: segmentation violation code=0x1 addr=0x0 pc=0x128a169]
```
经过测试函数的修改之后这个问题已经解决

#### 如何编写测试函数
1. 单元测试
2. e2e 测试


## 关于 git

1. 在进行本地开发并提交 PR 之前，需要在 github 上 fork kmesh 仓库，然后 git clone 到本地，然后新建一个 branch 进行开发。
2. 在提交之前**一定要**执行 make clean，否则会导致 ./config/kmesh_marcos_def.h 文件中的宏出错。执行 make clean 后会执行 git checkout 恢复该文件的内容至上一次提交的结果。
3. git commit 前设置好当前仓库的 user.name 和 user.email，提交时使用 git commit -s -m "your message"，**-s** 是必须要添加的，这样确保对 commit 签名，这样才能够通过 PR 审查。
4. **创建 PR**，代码 commit 并 push 之后，在自己 fork 的 kmesh github 仓库页面可以看到关于 PR 的提示，然后进行 PR。