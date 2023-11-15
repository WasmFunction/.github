## WasmFunction 👋

WasmFunction is a Kubernetes-native serverless FaaS solution built on top of WebAssembly.

![WasmFunction Architecture](https://github.com/WasmFunction/.github/blob/main/profile/wasmfunction.png?raw=true)

We focuses on:

- Seamlessly using WebAssembly in Kubernetes like OCI Container
- Reduce Kubernetes container cold start time
- Build a WebAssembly-based FaaS platform on top of the optimized Kubernetes

Our solution aims to achieve comparable cold start performance without container pre-warming.

### Contributing

Out solution involves development and optimization of projects and applications at different levels.

- [1999huye1104/wasm-faas](https://github.com/1999huye1104/wasm-faas). Made some extension and customization from [fission/fission](https://github.com/fission/fission), to manage WebAssembly functions.

- [WasmFunction/k3s-k8s](https://github.com/WasmFunction/k3s-k8s). Made some optimization about timers and reduced cold start time.
- [WasmFunction/containerd](https://github.com/WasmFunction/containerd). Made some optimization about pod sandbox and status reporting.
- [WasmFunction/kuasar](https://github.com/WasmFunction/kuasar). Made some customization for wasm-sandboxer from  the [kuasar-io/kuasar](https://github.com/kuasar-io/kuasar), to use our external WebAssembly runtime instead.
- [WasmFunction/rust-extensions](https://github.com/WasmFunction/rust-extensions). Stable dependency for [WasmFunction/kuasar](https://github.com/WasmFunction/kuasar). fixed and completed metric collecting.
- [WasmFunction/wasmkeeper](https://github.com/WasmFunction/wasmkeeper). Customized WebAssembly runtime and also a HTTP server that serves WebAssembly function. Called by  [WasmFunction/kuasar](https://github.com/WasmFunction/kuasar) and runs as the container process.

You can try it out by following steps from [K8s-files/how-to-setup.md](https://github.com/WasmFunction/K8s-files/blob/main/how-to-setup.md) to set up the environment.

### References

* https://www.infracloud.io/serverless-functions-kubernetes/
* https://knative.dev/docs/
* https://openwhisk.apache.org/
* https://www.openfaas.com/pricing/
* https://cn.serverless.com/
* https://kuasar.io/
