# 内网dify代码节点解决operation not permitted和安装依赖问题
## 写在最前
由于部门最近拥抱大模型，想基于agent做一些结合部门开发的工作。现有环境是公司内网已经部署了dify，但存在以下难点
- dify因为自身安全设定，对代码节点存在受限设置，只允许部分简单内置的python库，而如果引入别的依赖会涉及到系统进程syscall调用的权限，而**dify关闭了这些权限白名单**

具体可以通过查看某个配置确定
```
# 进入dify docker文件夹下
cd volumes/sandbox/conf

cat config.yaml

#这里可以查看到配置如下
allowed_syscalls: 

其中syscalls为空，也就是没有额外引入syscall
```
- 由于dify是内网部署，想下载所安装的python依赖需要手动引入（~~其实我一直觉得可以靠http节点解决，因为dify环境其实不太适合debug，dify代码节点返回的结果往往只是调用了一个python进程返回的结果，即使进入sandbox查看日志也无法看到具体报错。。。但是有这个要求我们就严格执行。。~~）

综合以上难点，具体操作流程如下

## 参考链接
https://blog.csdn.net/engchina/article/details/147641334
## 系统环境
宿主机架构 `arm64`（这也决定了下文的操作和x86_64的syscall序号不一样）

## 操作流程
### 解决依赖问题
依旧写在最前的前提
1. 首先明确，dify是基于多容器执行的，也就是基于docker-compose，而dify的代码节点主要是基于**sandbox**节点，因此具体安装是要进入sandbox节点里安装依赖。
2. 如何实现sandbox节点和宿主机文件交互：根据**docker-compose.yaml**配置显示，课明确知道`docker/volumes/sandbox/dependencies`是直接挂载在sandbox节点内的`/dependencies`，因此我们只要在宿主机该文件夹下安装对应的依赖库即可。

*这里给出网上的教程，是基于在线实现的*。
```
# 进入宿主机文件夹
cd docker/volumes/sandbox/dependencies
# 创建python-requirements.txt
touch python-requirements.txt
# 写入python-requirements.txt 
cat >> python-requirements.txt << EOF

#这里可以直接敲进去自己需要的库，下面仅为案例
> pandas
> openpyxl
> EOF

# 修改完后重启sandbox容器（进入docker文件夹内）
docker-compose restart sandbox

## 这里执行完，基于 docker logs -f docker-sandbox-1 --tail 100，
可以看到sandbox重启后会读取python-requirements下载依赖。
```

但显然由于公司内部dify是内网，并且公司内部仓库无法直接访问到，因此这里通过基于`pip download xxx --index http://xx.xx.xx.xx:yyyy 下载好对应的whl文件传给容器，容器执行pip install安装

### 解决syscall调用权限
首先明确一下什么是`syscall`
**syscall** ：系统调用，也就是用户程序需要调用系统内核的入口，主要是一些底层操作，比如**读写、复制**，但由于dify设置，默认是只保留了最基本的一些syscall，设计复杂python依赖都会报错`operation not permitted` .

具体操作参考https://blog.csdn.net/engchina/article/details/147641334。这里应注意给权限需要确保环境安全，题主因为是测试环境随便造所以全开了。开启后，也需要重新启动sandbox服务。

## 最后
以上即为dify代码节点修改权限问题，最后来看，dify的设计思路显然是不想让客户基于python节点做过多操作。但这里因为是不得不有这个需求才开始这个策略
