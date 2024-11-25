## WasmFunction 👋

WasmFunction 是一个基于 WebAssembly 构建的 Kubernetes 原生无服务器 FaaS（函数即服务）解决方案。

![WasmFunction Architecture](https://github.com/WasmFunction/.github/blob/main/profile/wasmfunction.png?raw=true)

我们专注于：

- 在 Kubernetes 中无缝使用 WebAssembly，如同使用 OCI 容器
- 减少 Kubernetes 容器冷启动时间
- 在优化的 Kubernetes 基础上构建基于 WebAssembly 的 FaaS 平台

我们的解决方案旨在实现无需容器预热的情况下，与传统容器相媲美的冷启动性能。



### 贡献

我们的解决方案涉及在不同层次上开发和优化项目及应用。

- [1999huye1104/wasm-faas](https://github.com/1999huye1104/wasm-faas)：对 [fission/fission](https://github.com/fission/fission) 进行了一些扩展和定制，用于管理 WebAssembly 函数。

- [WasmFunction/k3s-k8s](https://github.com/WasmFunction/k3s-k8s)：对计时器进行了一些优化，减少了冷启动时间。

- [WasmFunction/containerd](https://github.com/WasmFunction/containerd)：对 pod 沙盒和状态报告进行了一些优化。

- [WasmFunction/kuasar](https://github.com/WasmFunction/kuasar)：从 [kuasar-io/kuasar](https://github.com/kuasar-io/kuasar) 定制了 wasm-sandboxer，以使用我们的外部 WebAssembly 运行时。

- [WasmFunction/rust-extensions](https://github.com/WasmFunction/rust-extensions)：为 [WasmFunction/kuasar](https://github.com/WasmFunction/kuasar) 提供稳定依赖，修复并完成了指标收集。

- [WasmFunction/wasmkeeper](https://github.com/WasmFunction/wasmkeeper)：定制的 WebAssembly 运行时，同时也是一个服务 WebAssembly 函数的 HTTP 服务器。由 [WasmFunction/kuasar](https://github.com/WasmFunction/kuasar) 调用，并作为容器进程运行。

  
### 快速开始

​**测试环境: Ubuntu 24.04 LTS**

1. 首先下载安装包

   ```bash
   wget https://github.com/WasmFunction/WasmFunction/releases/download/v0.0.2/wasm-function.tar.gz
   ```

2. 运行安装脚本 

   ``` bash
   source install.sh
   注意: 在安装 WasmEdge 时，脚本会临时设置以下环境变量以确保库文件能够正确加载：
   - `LD_LIBRARY_PATH`: `/usr/local/lib/`
   如果你希望这些环境变量在后续的终端会话中也有效，可以将它们添加到你的 `~/.bashrc` 文件中。方法如下：
   echo 'export LD_LIBRARY_PATH=/usr/local/lib:$LD_LIBRARY_PATH' >> ~/.bashrc
   source ~/.bashrc
   ```

3. 运行启动脚本 

   ```bash
   source startup.sh
   注意: 在本脚本中，以下环境变量已被设置，但仅在当前的 shell 会话中有效：
   - `FISSION_NAMESPACE`: 当前被设置为 `fission`
   - `FISSION_ROUTER`: 当前被设置为 `localhost:32046`
   
   如果你希望这些环境变量在后续的终端会话中也有效，可以将它们添加到你的 `~/.bashrc` 文件中。方法如下：
   echo 'export FISSION_NAMESPACE=fission' >> ~/.bashrc
   echo 'export FISSION_ROUTER=localhost:32046' >> ~/.bashrc
   source ~/.bashrc
   ```

4. 可以使用以下命令查看 Fission 相关组件是否已启动，以及使用 tmux 命令查看 Containerd 与 Wasm-sandboxer 是否在后台运行，若发现其中有未启动的，可以重新运行启动脚本

   ```bash
   kubectl get pods
   tmux ls
   ```



### 简单使用

​**测试环境: Ubuntu 24.04 LTS**
<br></br>

1. 创建 "sort-wasm" 和 "hello-wasm" 环境

   **为 webassembly 环境命名时，目前规定为 xx-wasm 的名称格式。**

   原因：由于 kuasar wasm-sandboxer 的特性，函数与对应的环境概念是绑定的、一一对应的。例如，在使用默认运行时时，fission 官方有通用 pod 和 特化 pod 的概念，也就是说，如果创建一个 python env，fission 会为其创建一个通用池，当你需要触发一个 python 函数时，fission 会选择支持 python env 的 pod，对该函数特化，供其运行。
而我们为了支持 wasm 函数高效地部署和触发，使用了 kuasar 作为运行时。由于 kuasar 的特性，当我们使用 wasm-sandboxer 时，函数和环境其实是一一对应、融为一体的，所以我们的环境命名为 xx-wasm 的形式，体现了唯一性。
   我们支持两种 wasm 环境的创建方式：
   - 当 --image 参数的内容为镜像地址时，环境容器的镜像来源由该地址决定。
   - --image 参数的内容也可以为 xx.wasm 本地文件，此时我们会利用 kaniko 为用户打包本地镜像，作为容器的镜像来源。
   - **重要补充说明：规定了 kaniko 的工作区为 /var/lib/kaniko/workplace/ ，在安装脚本中会自动创建，如果 --image 参数的内容为 xx.wasm 本地文件，xx.wasm 文件必须在 /var/lib/kaniko/workplace/ 路径下存在， 否则 kaniko pod 无法在本项目指定的位置进行镜像构建。**
   ```bash
   --image 参数的内容为镜像地址时
   fission env create --name sort-wasm --image docker.io/amnesia1997/sortwasm
   --image 参数的内容为本地镜像文件时
   fission env create --name hello-wasm --image hello.wasm
   ```
2. 创建 "sort" 和 "wasm" 函数

   说明：
   - 目前没有对该命令做针对 wasm 的过多改写，此处 --code 参数在 fission 原架构中当然是必须的。
   - 但由于我们只是提供一种基于 kuasar 的 wasm 函数的部署和实现，实际上如上面所说，在 wasm + kuasar 的情况下，函数和环境是融为一体的，函数作为构建容器镜像的一部分，所以此处 --code 参数可以提供一个空文件，或者任何文件也是没有影响的，后续会进行优化。
   ```bash
   fission fn create --name sort --env sort-wasm --code sort.wasm
   fission fn create --name hello --env hello-wasm --code hello.wasm
   ```

3. 创建相应的 http 触发器。

   ```bash
   fission route create --url /sort --method POST --function sort
   fission route create --url /hello --method POST --function hello
   ```

4. 对两个函数分别进行触发。

   ```bash
   curl -X POST -H "Content-Type: application/json" -d '{"args":["1111", "2345", "8", "99", "11234", "12312", "1231231", "-10000"]}' http://$FISSION_ROUTER/sort
   
   curl -X POST -H "Content-Type: application/json" -d '{}' http://$FISSION_ROUTER/hello
   ```
可以得到如下结果:
![test result](https://github.com/WasmFunction/.github/blob/main/profile/test%20function.png?raw=true)

4. 可以在 "fission-function"命名空间下查看两个函数在 kubernetes 中创建的对应资源，以及使用 fission 的命令查看已经创建的函数和触发器。

   

### 如何编写自己的 wasm 函数，使其在本项目中能够顺利运行？

**以 sort.cpp 为例，其内容如下（使用GPT4生成）:**

```cpp
#include <stdio.h>
#include <stdlib.h>

int compare(const void* a, const void* b) {
    return (*(int*)a - *(int*)b);
}

int main(int argc, char** argv) {
    if (argc < 2) {
        printf("No numbers to sort\n");
        return 0;
    }

    int count = argc - 1;
	int* numbers = (int*)malloc(count * sizeof(int));
    for (int i = 1; i < argc; i++) {
        numbers[i - 1] = atoi(argv[i]);
    }

    qsort(numbers, count, sizeof(int), compare);

    for (int i = 0; i < count; i++) {
        printf("%d ", numbers[i]);
    }
    printf("\n");
    free(numbers);
    return 0;
}
```

**然后可以使用各种方式将其编译为.wasm 可执行文件，此处不另作介绍。**

在进行 http 触发函数时:  ``` curl -X POST -H "Content-Type: application/json" -d '{"args":["1111", "2345", "8", "99", "11234", "12312", "1231231", "-10000"]}' http://$FISSION_ROUTER/sort```

1. 当通过 HTTP 请求触发该排序函数时，请求体中的 args 会包含一组字符串形式的数字: ```{"args":["1111", "2345", "8", "99", "11234", "12312", "1231231", "-10000"]}```。这些参数最终会被解析为命令行参数形式，传递给 WebAssembly 运行时。具体来说，运行时会将 args 列表传递给 Wasm 函数的 argv 参数，类似于执行以下命令：```wasmedge sort.wasm 1111 2345 8 99 11234 12312 1231231 -10000```。
   注意：在实际执行的 C 程序中，argv[0] 是 "sort.wasm"（程序名称），而参数 "1111", "2345", ... 等则从 argv[1] 开始。这意味着在处理输入时，我们需要从 argv[1] 开始将这些参数转化为整数。
2. 因此，两个核心注意点是：
   命令行参数的处理：argv[0] 是程序名称，数字从 argv[1] 开始处理。
   HTTP 请求体与程序参数的对应：args 列表中的字符串会被解析为整数进行排序。
3. 根据以上规则，开发者可以自定义各种函数。




目前将函数打包成镜像的 Dockerfile 内容统一为：

```dockerfile
FROM scratch
COPY YOUR_FUNCTION.WASM /
ENTRYPOINT = ["YOUR_FUNCTION.WASM"]

例如:
FROM scratch
COPY sort.wasm /
ENTRYPOINT = ["sort.wasm"]
```





### 如何编译本项目

本项目结构主要包括 Fission 和 K8s 部分。K8s 部分包括 K3s，Containerd，Kuasar，以及 Wasmkeeper。

本文将按照Wasmkeeper，Kuasar，Containerd，K3s， Fission 的顺序说明如何编译，通过这些步骤，您可以自己编译并顺利运行该项目。

#### 1. Wasmkeeper

wasmkeeper用于创建容器进程，运行WebAssembly实例。

- [WasmFunction/wasmkeeper: A simple http server that runs wasm functions. (github.com)](https://github.com/WasmFunction/wasmkeeper)

**构建条件**

- cmake

- vcpkg

- wasmedge v0.11.2

  ```bash
  wget https://github.com/WasmEdge/WasmEdge/releases/download/0.11.2/WasmEdge-0.11.2-ubuntu20.04_x86_64.tar.gz
  tar -zxvf WasmEdge-0.11.2-ubuntu20.04_x86_64.tar.gz
  cp -r WasmEdge-0.11.2-Linux/include/* /usr/local/include/
  cp -r WasmEdge-0.11.2-Linux/lib/* /usr/local/lib/
  cp WasmEdge-0.11.2-Linux/bin/wasmedge /usr/local/lib/
  export LD_LIBRARY_PATH=/usr/local/lib/
  ```

**编译与安装**

```bash
# install dependency using vcpkg
# cd path/to/vcpkg
./vcpkg install jsoncpp

# clone repository
git clone https://github.com/WasmFunction/wasmkeeper.git
cd wasmkeeper

# cd path/to/wasmkeeper
mkdir build && cd build

# change [vcpkg root] to path/to/vcpkg
cmake .. -DCMAKE_TOOLCHAIN_FILE=[vcpkg root]/scripts/buildsystems/vcpkg.cmake

# compile
cmake --build .

# install
cp wasmkeeper /usr/local/bin/
```



#### 2. Wasm-sandboxer（基于 Kuasar v0.3.0）

用于被Containerd调用，提供沙盒管理服务和Task服务，负责调用wasmkeeper创建容器进程、管理容器。

- [WasmFunction/kuasar: A multi-sandbox container runtime that provides cloud-native, all-scenario multiple sandbox container solutions. (github.com)](https://github.com/WasmFunction/kuasar)

**构建条件**

- Rust

**编译与安装**

```bash
# clone repository
git clone https://github.com/WasmFunction/kuasar.git
cd kuasar
# *** important ***
git checkout wasmkeeper

# compile
make bin/wasm-sandboxer

# install
cp bin/wasm-sandboxer /usr/local/bin/
```

**运行**

这里推荐使用tmux在后台运行wasm-sandboxer。

```bash
tmux new-session -d -s wasm-sandboxer 'wasm-sandboxer --listen /run/wasm-sandboxer.sock --dir /run/kuasar-wasm'
```



#### 3. Containerd（基于 Containerd v1.5.11）

适配wasm-sandboxer的Containerd，同时配合Fission更新Pod IP。

- [WasmFunction/containerd at fission_event (github.com)](https://github.com/WasmFunction/containerd/tree/fission_event)

**构建条件**

- Go

**编译与安装**

其中**runc**和**CNI**安装与原版 Containerd 相同，如果以下安装方法失效可以参考Containerd官方文档安装。

- [containerd/docs/getting-started.md at main · containerd/containerd (github.com)](https://github.com/containerd/containerd/blob/main/docs/getting-started.md)

1. 安装**runc**，以能够运行OCI容器。

   Download the `runc.<ARCH>` binary from https://github.com/opencontainers/runc/releases , verify its sha256sum, and install it as `/usr/local/sbin/runc`.

   ```bash
   install -m 755 runc.amd64 /usr/local/sbin/runc
   ```

2. 安装**CNI**插件。

   Download the `cni-plugins-<OS>-<ARCH>-<VERSION>.tgz` archive from https://github.com/containernetworking/plugins/releases , verify its sha256sum, and extract it under `/opt/cni/bin`：

   ```bash
   mkdir -p /opt/cni/bin
   
   tar Cxzvf /opt/cni/bin cni-plugins-<OS>-<ARCH>-<VERSION>.tgz
   ```

3. 编译与安装**Containerd**。

   ```bash
   # clone repository
   git clone https://github.com/WasmFunction/containerd.git
   cd containerd
   # *** important ***
   git checkout fission_event
   
   make bin/containerd
   
   cp bin/containerd  /usr/local/bin/
   ```

   将 Containerd 配置文件 `config.toml.kuasar`，放至 Containerd 的配置目录。

   ```bash
   wget https://raw.githubusercontent.com/WasmFunction/WasmFunctionFiles/refs/heads/main/config.toml.kuasar
   mkdir /etc/containerd/
   cp config.toml.kuasar /etc/containerd/config.toml
   ```

​		*该配置文件在 Kuasar 的 Containerd 的配置文件的基础上，将CNI插件配置更换为 K3s 的配置。*

**运行**

这里推荐使用tmux在后台运行Containerd。

```bash
tmux new-session -d -s containerd 'ENABLE_CRI_SANDBOXES=1 containerd'
```



#### 4. K3s（基于 K3s v1.28.2）

在原K3s的基础上优化了定时器问题。

- [WasmFunction/k3s-k8s: customized kubernetes from k3s (github.com)](https://github.com/WasmFunction/k3s-k8s)

**构建条件**

- Go
- Docker

**编译与安装**

```bash
# clone repository
git clone https://github.com/WasmFunction/k3s.git
cd k3s

# config
mkdir -p build/data && make download && make generate

# compile
SKIP_VALIDATE=true make

# install
cp ./dist/artifacts/k3s /usr/local/bin/
```

**运行**

使用`k3s_install.sh`来初始化k3s service。

**主节点**

```bash
wget https://raw.githubusercontent.com/WasmFunction/WasmFunctionFiles/refs/heads/main/k3s_install.sh
# init service
INSTALL_K3S_SKIP_DOWNLOAD=true sh ./k3s_install.sh --container-runtime-endpoint /run/containerd/containerd.sock

# kubeconfig set up
mkdir ~/.kube
cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
```

*如果主节点不运行wasm工作负载，则无需安装上述程序。*

```bash
# init service
INSTALL_K3S_SKIP_DOWNLOAD=true sh ./k3s_install.sh

# kubeconfig set up
mkdir ~/.kube
cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
```

**从节点**

主节点初始化k3s之后在**主节点**查看`token`

```bash
cat /var/lib/rancher/k3s/server/token
```

将`[master ip]`和`token`替换为主节点实际值，运行命令使从节点加入集群。

```bash
INSTALL_K3S_SKIP_DOWNLOAD=true K3S_URL=https://[master ip]:6443 K3S_TOKEN=[token] sh ./k3s_install.sh --container-runtime-endpoint /run/containerd/containerd.sock
```



#### 5. Fission（基于 Fission v1.17.0）

**编译过程请参考 [Fission 的编译和打包过程](https://github.com/1999huye1104/wasm-faas/blob/main/README.md) 中的编译部分**

**部署过程可参考fission官方教程创建 crd 资源和命名空间等，但使用本项目提供的yaml文件进行部署，以及使用本项目拓展后的fission cli （在 Fission 编译中获得）**

```bash
#创建CRD资源
kubectl create -k "github.com/fission/fission/crds/v1?ref=v1.17.0"

#设置默认当前namespace为fission
export FISSION_NAMESPACE="fission"
kubectl create namespace $FISSION_NAMESPACE
kubectl config set-context --current --namespace=$FISSION_NAMESPACE

#获取yaml文件
wget https://raw.githubusercontent.com/WasmFunction/WasmFunctionFiles/refs/heads/main/wasm-function.yaml

#安装fissionl
kubectl apply -f wasm-function.yaml

#CLI安装,如果是自己编译后的版本可以将url改为自己编译发布的CLI，例如:
curl -Lo faas-cli https://github.com/1999huye1104/wasm-faas/releases/download/v1.18.14/fission-v1.18.14-linux-amd64 && chmod +x faas-cli && sudo mv fission /usr/local/bin/
```



### 参考资料

* https://www.infracloud.io/serverless-functions-kubernetes/
* https://knative.dev/docs/
* https://openwhisk.apache.org/
* https://www.openfaas.com/pricing/
* https://cn.serverless.com/
* https://kuasar.io/

