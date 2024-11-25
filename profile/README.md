## WasmFunction 👋

WasmFunction is a Kubernetes-native serverless FaaS solution built on top of WebAssembly.

![WasmFunction Architecture](https://github.com/WasmFunction/.github/blob/main/profile/wasmfunction.png?raw=true)

We focuses on:

- Seamlessly using WebAssembly in Kubernetes like OCI Container.
- Reduce Kubernetes container cold start time.
- Build a WebAssembly-based FaaS platform on top of the optimized Kubernetes.

Our solution aims to achieve comparable cold start performance without container pre-warming.

### Contributing

Out solution involves development and optimization of projects and applications at different levels.

- [1999huye1104/wasm-faas](https://github.com/1999huye1104/wasm-faas). Made some extension and customization from [fission/fission](https://github.com/fission/fission), to manage WebAssembly functions.

- [WasmFunction/k3s-k8s](https://github.com/WasmFunction/k3s-k8s). Made some optimization about timers and reduced cold start time.
- [WasmFunction/containerd](https://github.com/WasmFunction/containerd). Made some optimization about pod sandbox and status reporting.
- [WasmFunction/kuasar](https://github.com/WasmFunction/kuasar). Made some customization for wasm-sandboxer from  the [kuasar-io/kuasar](https://github.com/kuasar-io/kuasar), to use our external WebAssembly runtime instead.
- [WasmFunction/rust-extensions](https://github.com/WasmFunction/rust-extensions). Stable dependency for [WasmFunction/kuasar](https://github.com/WasmFunction/kuasar). fixed and completed metric collecting.
- [WasmFunction/wasmkeeper](https://github.com/WasmFunction/wasmkeeper). Customized WebAssembly runtime and also a HTTP server that serves WebAssembly function. Called by  [WasmFunction/kuasar](https://github.com/WasmFunction/kuasar) and runs as the container process.


### Quick Start

**Test Environment: Ubuntu 24.04 LTS**

1. First, download the installation package.

   ```bash
   wget https://github.com/WasmFunction/WasmFunction/releases/download/v0.0.2/wasm-function.tar.gz
   ```

2. Run the installation script.

   ``` bash
   source install.sh
   # Note: During the installation of WasmEdge, the script temporarily sets the following environment variables to ensure library files load correctly:
   - `LD_LIBRARY_PATH`: `/usr/local/lib/`
   
   # If you want these environment variables to persist in future terminal sessions, you can add them to your `~/.bashrc` file as follows:
   ```bash
   echo 'export LD_LIBRARY_PATH=/usr/local/lib:$LD_LIBRARY_PATH' >> ~/.bashrc
   source ~/.bashrc
   ```

3. Run the startup script.

   ```bash
   source startup.sh
   # Note: The following environment variables are set in this script, but are only effective in the current shell session:
   - `FISSION_NAMESPACE`: Currently set to `fission`
   - `FISSION_ROUTER`: Currently set to `localhost:32046`
   
   # If you want these environment variables to persist across future sessions, you can add them to your `~/.bashrc` file as follows:
   ```bash
   echo 'export FISSION_NAMESPACE=fission' >> ~/.bashrc
   echo 'export FISSION_ROUTER=localhost:32046' >> ~/.bashrc
   source ~/.bashrc
   ```

4. Use the following command to check if the Fission components are running and to verify that containerd and Wasm-sandboxer are running in the background using `tmux`. If any of them are not running, rerun the startup script.

   ```bash
   kubectl get pods
   tmux ls
   ```



### Test

**Test Environment: Ubuntu 24.04 LTS**

1. Create the "sort" and "hello" functions.

   **Naming convention for WebAssembly environments: The environment names should follow the xx-wasm format.**
   Reason: Due to the characteristics of the Kuasar wasm-sandboxer, functions are bound to their corresponding environments in a one-to-one relationship. For example, when using the default runtime, Fission provides general pods and specialized pods. For instance, when you create a Python environment, Fission creates a general pool of pods. When a Python function is triggered, Fission selects a pod from the pool that supports the Python environment and specializes it for running the function.
However, to support efficient deployment and triggering of WebAssembly (Wasm) functions, we use Kuasar as the runtime. With Kuasar’s wasm-sandboxer feature, functions and environments are essentially fused into one entity. Hence, we adopt the xx-wasm naming convention to emphasize this uniqueness.
   We support two ways to create Wasm environments:
   - If the ```--image``` parameter specifies an image address, the environment container’s image is fetched from that address.
   - If the ```--image``` parameter specifies a local .wasm file, Kaniko is used to package the local file into an image as the container’s source image.
   - **Important Note: The Kaniko workspace is predefined as ```/var/lib/kaniko/workplace/```. The installation script automatically creates this directory. If the ```--image``` parameter specifies a local .wasm file, the file must exist in the ```/var/lib/kaniko/workplace/``` path; otherwise, the Kaniko pod will fail to build the image at the specified location.**
   ```bash
   # If the --image parameter specifies an image address
   fission env create --name sort-wasm --image docker.io/amnesia1997/sortwasm

   # If the --image parameter specifies a local `.wasm` file
   fission env create --name hello-wasm --image hello.wasm
   ```

2. Create “sort” and “hello” Functions
   Notes:
   - The ```--code``` parameter is required as per the original Fission architecture.
   - However, since we are offering a deployment and execution approach for Wasm functions using Kuasar, as mentioned earlier, the function and environment are effectively fused into a single entity. The function is packaged as part of the container image. Therefore, the ```--code``` parameter can point to an empty file or any file without impacting functionality. This will be optimized in the future.
   ```bash
   # If the --image parameter specifies an image address
   fission env create --name sort-wasm --image docker.io/amnesia1997/sortwasm

   # If the --image parameter specifies a local `.wasm` file
   fission env create --name hello-wasm --image hello.wasm
   ```
   
3. Create corresponding HTTP triggers.
   ```bash
   fission route create --url /sort --method POST --function sort
   fission route create --url /hello --method POST --function hello
   ```

4. Trigger the functions using the following commands.

   ```bash
   curl -X POST -H "Content-Type: application/json" -d '{"args":["1111", "2345", "8", "99", "11234", "12312", "1231231", "-10000"]}' http://$FISSION_ROUTER/sort
   
   curl -X POST -H "Content-Type: application/json" -d '{}' http://$FISSION_ROUTER/hello
   ```
You should see the following output.
![test result](https://github.com/WasmFunction/.github/blob/main/profile/test%20function.png?raw=true)

4. You can check the corresponding resources created in Kubernetes for the two functions under the "fission-function" namespace, and you can also use Fission commands to view the created functions and triggers.

### How to Write Your Own Wasm Functions to Run in This Project?

**Take `sort.cpp` as an example (source code generated by GPT-4).**

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

**You can compile this to a .wasm executable file using various tools, which are not covered here.**

 ``` curl -X POST -H "Content-Type: application/json" -d '{"args":["1111", "2345", "8", "99", "11234", "12312", "1231231", "-10000"]}' http://$FISSION_ROUTER/sort.```

1. When this sort function is triggered via an HTTP request, the `args` field in the request body contains a list of numbers in string format: `{"args":["1111", "2345", "8", "99", "11234", "12312", "1231231", "-10000"]}`. These arguments will be parsed as command-line arguments and passed to the WebAssembly runtime. Essentially, it will execute a command ```like:wasmedge sort.wasm 1111 2345 8 99 11234 12312 1231231 -10000```.
   Note: In the actual C program, `argv[0]` is `"sort.wasm"` (the program name), and the arguments `"1111", "2345", ...` start from `argv[1]`. Therefore, we need to process the arguments from `argv[1]` onwards and convert them to integers.
2. Two key points:
   - Command-line argument handling: `argv[0]` is the program name, and the numbers start from `argv[1]`.
   - Correspondence between the HTTP request body and the program arguments: The string list in the `args` field is parsed into integers for sorting.
3. Based on the above rules, developers can customize various functions.




The Dockerfile content used to package functions into images is unified as follows.

```dockerfile
FROM scratch
COPY YOUR_FUNCTION.WASM /
ENTRYPOINT = ["YOUR_FUNCTION.WASM"]

For example:
FROM scratch
COPY sort.wasm /
ENTRYPOINT = ["sort.wasm"]
```





### How to Compile the Project

This project mainly consists of Fission and K8s components. The K8s components include K3s, containerd, Kuasar, and Wasmkeeper.

This section will explain how to compile in the following order: Wasmkeeper, Kuasar, containerd, K3s, and Fission. By following these steps, you can compile and run the project yourself.

#### 1. Wasmkeeper

Wasmkeeper is used to create container processes and run WebAssembly instances.

- [WasmFunction/wasmkeeper: A simple http server that runs wasm functions. (github.com)](https://github.com/WasmFunction/wasmkeeper)

**Prerequisites**

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

**Build and Install**

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



#### 2.  Wasm-sandboxer (Based on Kuasar v0.3.0)

Wasm-sandboxer is used by containerd to provide sandbox management and task services, responsible for calling Wasmkeeper to create and manage container processes.

- [WasmFunction/kuasar: A multi-sandbox container runtime that provides cloud-native, all-scenario multiple sandbox container solutions. (github.com)](https://github.com/WasmFunction/kuasar)

**Prerequisites**

- Rust

**Build and Install**

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

**Run**

It's recommended to run Wasm-sandboxer in the background using tmux.

```bash
tmux new-session -d -s wasm-sandboxer 'wasm-sandboxer --listen /run/wasm-sandboxer.sock --dir /run/kuasar-wasm'
```



#### 3.  Containerd (Based on Containerd v1.5.11)

This is a modified version of containerd to support Wasm-sandboxer, and it integrates with Fission to update Pod IPs.

- [WasmFunction/containerd at fission_event (github.com)](https://github.com/WasmFunction/containerd/tree/fission_event)

**Prerequisites**

- Go

**Build and Install**

The installation of **runc** and **CNI** follows the same steps as in the original containerd. If the following method doesn't work, refer to the official containerd documentation for installation.

- [containerd/docs/getting-started.md at main · containerd/containerd (github.com)](https://github.com/containerd/containerd/blob/main/docs/getting-started.md)

1. Install **runc** to run OCI containers:

   Download the `runc.<ARCH>` binary from https://github.com/opencontainers/runc/releases , verify its sha256sum, and install it as `/usr/local/sbin/runc`.

   ```bash
   install -m 755 runc.amd64 /usr/local/sbin/runc
   ```

2. Install **CNI** plugins:

   Download the `cni-plugins-<OS>-<ARCH>-<VERSION>.tgz` archive from https://github.com/containernetworking/plugins/releases , verify its sha256sum, and extract it under `/opt/cni/bin`：

   ```bash
   mkdir -p /opt/cni/bin
   
   tar Cxzvf /opt/cni/bin cni-plugins-<OS>-<ARCH>-<VERSION>.tgz
   ```

3. Build and install **containerd**:

   ```bash
   # clone repository
   git clone https://github.com/WasmFunction/containerd.git
   cd containerd
   # *** important ***
   git checkout fission_event
   
   make bin/containerd
   
   cp bin/containerd  /usr/local/bin/
   ```

   Copy the Containerd configuration file `config.toml.kuasar` to the Containerd configuration directory.

   ```bash
   wget https://raw.githubusercontent.com/WasmFunction/WasmFunctionFiles/refs/heads/main/config.toml.kuasar
   mkdir /etc/containerd/
   cp config.toml.kuasar /etc/containerd/config.toml
   ```

 *This configuration file is based on Kuasar's Containerd configuration file, with the CNI plugin configuration replaced by K3s configuration.*

**Run**

It's recommended to run containerd in the background using tmux.

```bash
tmux new-session -d -s containerd 'ENABLE_CRI_SANDBOXES=1 containerd'
```



#### 4. K3s (Based on K3s v1.28.2)

This is an optimized version of K3s to address timer issues.

- [WasmFunction/k3s-k8s: customized kubernetes from k3s (github.com)](https://github.com/WasmFunction/k3s-k8s)

**Prerequisites**

- Go
- Docker

**Build and Install**

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

**Run**

Use `k3s_install.sh` to initialize the k3s service.

**Master Node**

```bash
wget https://raw.githubusercontent.com/WasmFunction/WasmFunctionFiles/refs/heads/main/k3s_install.sh
# init service
INSTALL_K3S_SKIP_DOWNLOAD=true sh ./k3s_install.sh --container-runtime-endpoint /run/containerd/containerd.sock

# kubeconfig set up
mkdir ~/.kube
cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
```

*If the master node does not run Wasm workloads, you can skip installing the above programs.*

```bash
# init service
INSTALL_K3S_SKIP_DOWNLOAD=true sh ./k3s_install.sh

# kubeconfig set up
mkdir ~/.kube
cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
```

**Worker Node**

After initializing the master node, on the **master node**, retrieve the `token`

```bash
cat /var/lib/rancher/k3s/server/token
```

Replace `[master ip]` and `[token]` with the actual values from the master node and run the following command to join the worker node to the cluster

```bash
INSTALL_K3S_SKIP_DOWNLOAD=true K3S_URL=https://[master ip]:6443 K3S_TOKEN=[token] sh ./k3s_install.sh --container-runtime-endpoint /run/containerd/containerd.sock
```



#### 5. Fission (Based on Fission v1.17.0)

**Please refer to the [Fission Compilation and Packaging Process](https://github.com/1999huye1104/wasm-faas/blob/main/README.md) for the build process.**

**For deployment, you can refer to the official Fission guide to create CRD resources and namespaces, but use the YAML files provided by this project, and the extended Fission CLI provided by this project (which is obtained during Fission compilation).**

```bash
# Create CRD resources
kubectl create -k "github.com/fission/fission/crds/v1?ref=v1.17.0"

# Set the default namespace to fission
export FISSION_NAMESPACE="fission"
kubectl create namespace $FISSION_NAMESPACE
kubectl config set-context --current --namespace=$FISSION_NAMESPACE

# Fetch the yaml file
wget https://raw.githubusercontent.com/WasmFunction/WasmFunctionFiles/refs/heads/main/wasm-function.yaml

# Install Fission
kubectl apply -f wasm-function.yaml

# Install CLI (if you compiled your own version, modify the URL to point to your CLI release, for example):
curl -Lo faas-cli https://github.com/1999huye1104/wasm-faas/releases/download/v1.18.14/fission-v1.18.14-linux-amd64 && chmod +x faas-cli && sudo mv fission /usr/local/bin/
```


### References

* https://www.infracloud.io/serverless-functions-kubernetes/
* https://knative.dev/docs/
* https://openwhisk.apache.org/
* https://www.openfaas.com/pricing/
* https://cn.serverless.com/
* https://kuasar.io/
