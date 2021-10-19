# K3s on Jetson Board with GPU Allocation 

Created by Jack Liang

>Disclaimer: Due to internet blockade in China (and maybe running inside SSH/Jetson board), downloading remains unreliable. If any curl, apt, git, etc commands fail or hang, TRY again. And again. If the repeating of waiting and trying does’t work, change the mirrors or resort to SCP.
 
Go through Set up Networking in Jetson Board. 

First `git clone https://github.com/jetsonhacks/jetsonUtilities`, then `cd` into the directory and `python3 jetsonInfo.py`.

Output:
```
NVIDIA Jetson Xavier NX (Developer Kit Version)
L4T 32.4.3 [ JetPack 4.4 ]
Ubuntu 18.04.4 LTS
Kernel Version: 4.9.140-tegra
...
```
Getting to know our Jetson board is important for this exercise. Because at the point of writing of this guide, https://github.com/NVIDIA/k8s-device-plugin which is for exposing the number of GPU's on each node in a k8s cluster and keeping track of the health of GPUs, has only recently started officially supporting ARM64 (https://github.com/NVIDIA/k8s-device-plugin/commit/cde4e1a1dcdf75f60eb5de488a72007ad61d8443). Not only are our Jetson boards on ARM64, they are also Jetson Xavier NX boards, which is Tegra architecture. We will refer back to these details later on; getting to know these details prior to starting off avoids all the kerfuffle in preparing for this guide. 

Second, install/run K3s:
`curl -sfL https://get.k3s.io | sh -s - --docker` (refer to https://k3s.io/ but add the “--docker”). This also installs kubectl, so no need to install kubectl separately. The “--docker” part allows K3s to access local docker images instead of only pulling from registries. After this is done, we should see our K3s node with `sudo k3s kubectl get node`.

## Docker Compose to K8s 
Next, we want to convert the docker-compose files to K8s YAML files. Install Kompose then create an .env file from the environment variables you use for the docker-compose files. Run `docker-compose -f docker-compose.file.of.conversion.interest.yml config > docker-compose-output.yml` to create a temporary docker-compose file with all environment variables substituted in from the .env file. Modify the docker-compose-output.yml as follows: Remove any instances of "device_requests" that's for allocation of GPU when running Docker Compose (this is now useless in K3s for requesting GPU resource, plus conversion won’t work with “device_requests”) and change the docker-compose version from “3” to “3.7” → 

Now run `kompose convert -f docker-compose-output.yml --volumes hostPath`. Notice all the K8s service and deployment files generated from the Docker Compose services.
To learn more about the generated YAML files, visit https://www.youtube.com/watch?v=qmDzcu5uY1I. 
The “--volumes hostPath” option maps the K3s pod volumes to our Jetson board. We will now use a K8s deployment file to create a K3s node.
   
Let’s run `sudo k3s kubectl apply -f ./example-deployment.yaml`. You should see the deployment running with `sudo k3s kubectl get pods`.  You can also see the logs by `sudo k3s kubectl logs -f xxx`. 

## Solving Node Disk Pressure Taint 
Before we go any further, let’s take a detour and talk about the potential disk pressure problem. If your Jetson board is already loaded with docker images, chances are that upon restarting the Jetson board,
you will see "Taints: node.kubernetes.io/disk-pressure:NoSchedule” in the description of `sudo k3s kubectl describe node`. To resolve, first `df -h` to see which drive mount is almost full. According to https://kubernetes.io/docs/concepts/scheduling-eviction/node-pressure-eviction/ in human terms, when the mounted drive at which `kubelet` is installed in gets less than 10% disk space, pods will be hard-evicted or killed un-gracefully. To amend this issue, we need to move kubelet to a more hospitable habitat. Perform the following: 

```
mkdir -p /folder-which-drive-mount-has-more-space
sudo cp -r /var/lib/kubelet/ /folder-which-drive-mount-has-more-space
sudo mv /var/lib/kubelet /var/lib/kubelet-bak
sudo ln -s /folder-which-drive-mount-has-more-space /var/lib/kubelet
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```
If the `systemctl` restart doesn’t work, simply restart the Jetson board by power. `sudo k3s kubectl describe node` and you should see everything normal again.  

## Granting GPU Access
Back to our regular scheduled guide. Right now if you deploy the YAML file again then run `jtop` immediately, watch and you’ll see there's no GPU activity whatsoever. What we now need to do is request GPU resource from within the K3s pod. But first, let’s make sure that we can communicate with the Jetson board GPU. `docker pull jitteam/devicequery` then `sudo k3s kubectl run -i -t nvidia --image=jitteam/devicequery --restart=Never` → 
```
CUDA Device Query (Runtime API) version (CUDART static linking)

Detected 1 CUDA Capable device(s)

Device 0: "Xavier"
  CUDA Driver Version / Runtime Version          10.2 / 10.0
  CUDA Capability Major/Minor version number:    7.2
  Total amount of global memory:                 7770 MBytes (8147574784 bytes)
  ( 6) Multiprocessors, ( 64) CUDA Cores/MP:     384 CUDA Cores
  GPU Max Clock rate:                            1109 MHz (1.11 GHz)
  Memory Clock rate:                             1109 Mhz
  Memory Bus Width:                              256-bit
  L2 Cache Size:                                 524288 bytes
  Maximum Texture Dimension Size (x,y,z)         1D=(131072), 2D=(131072, 65536), 3D=(16384, 16384, 16384)
  Maximum Layered 1D Texture Size, (num) layers  1D=(32768), 2048 layers
  Maximum Layered 2D Texture Size, (num) layers  2D=(32768, 32768), 2048 layers
  Total amount of constant memory:               65536 bytes
  Total amount of shared memory per block:       49152 bytes
  Total number of registers available per block: 65536
  Warp size:                                     32
  Maximum number of threads per multiprocessor:  2048
  Maximum number of threads per block:           1024
  Max dimension size of a thread block (x,y,z): (1024, 1024, 64)
  Max dimension size of a grid size    (x,y,z): (2147483647, 65535, 65535)
  Maximum memory pitch:                          2147483647 bytes
  Texture alignment:                             512 bytes
  Concurrent copy and kernel execution:          Yes with 1 copy engine(s)
  Run time limit on kernels:                     No
  Integrated GPU sharing Host Memory:            Yes
  Support host page-locked memory mapping:       Yes
  Alignment requirement for Surfaces:            Yes
  Device has ECC support:                        Disabled
  Device supports Unified Addressing (UVA):      Yes
  Device supports Compute Preemption:            Yes
  Supports Cooperative Kernel Launch:            Yes
  Supports MultiDevice Co-op Kernel Launch:      Yes
  Device PCI Domain ID / Bus ID / location ID:   0 / 0 / 0
  Compute Mode:
     < Default (multiple host threads can use ::cudaSetDevice() with device simultaneously) >

deviceQuery, CUDA Driver = CUDART, CUDA Driver Version = 10.2, CUDA Runtime Version = 10.0, NumDevs = 1
Result = PASS
```
If the result looks like this with a “PASS” at the end, we should be good to go. Additionally, we can run `tegrastats` (a replacement for `nvidia-smi`, which is not available for the Jetson platform due to Jetson runs on the integrated SoC Tegra; `nvidia-smi` only supports standalone GPU’s) and see some stats that show "GPU" and "GR3D_FREQ" information → 

```
RAM 4071/7770MB (lfb 171x4MB) SWAP 0/3885MB (cached 0MB) CPU [9%@1420,7%@1420,9%@1420,7%@1420,12%@1420,6%@1420] EMC_FREQ 0% GR3D_FREQ 0% AO@61.5C GPU@62C PMIC@100C AUX@62C CPU@64C thermal@62.25C VDD_IN 5608/5483 VDD_CPU_GPU_CV 1160/1089 VDD_SOC 2640/2643
RAM 4071/7770MB (lfb 171x4MB) SWAP 0/3885MB (cached 0MB) CPU [2%@1420,2%@1420,6%@1420,7%@1420,3%@1420,1%@1420] EMC_FREQ 0% GR3D_FREQ 0% AO@62C GPU@62C PMIC@100C AUX@62C CPU@63.5C thermal@62.25C VDD_IN 5408/5476 VDD_CPU_GPU_CV 1041/1084 VDD_SOC 2640/2643
```
Moreover, we can run `sudo k3s kubectl exec -it xxx -- python3` then
```
from tensorflow.python.client import device_lib
device_lib.list_local_devices()
```
which yields: 
```
Created TensorFlow device (/device:GPU:0 with 539 MB memory) -> physical GPU (device: 0, name: Xavier, pci bus id: 0000:00:00.0, compute capability: 7.2)
[name: "/device:CPU:0"
device_type: "CPU"
memory_limit: 268435456
locality {
}
incarnation: 10468017588030091916
, name: "/device:XLA_CPU:0"
device_type: "XLA_CPU"
memory_limit: 17179869184
locality {
}
incarnation: 3593462647012768773
physical_device_desc: "device: XLA_CPU device"
, name: "/device:XLA_GPU:0"
device_type: "XLA_GPU"
memory_limit: 17179869184
locality {
}
incarnation: 6550798316105121592
physical_device_desc: "device: XLA_GPU device"
, name: "/device:GPU:0"
device_type: "GPU"
memory_limit: 565473280
locality {
  bus_id: 1
  links {
  }
}
incarnation: 2436860410790867085
physical_device_desc: "device: 0, name: Xavier, pci bus id: 0000:00:00.0, compute capability: 7.2"
]
```
And even: 
```
from tensorflow.python.client import device_lib
def get_available_gpus():
  local_device_protos = device_lib.list_local_devices()
  return [x.name for x in local_device_protos if x.device_type == 'GPU']
get_available_gpus()
```
which yields: 
```
2021-09-22 13:26:21.806527: I tensorflow/core/common_runtime/gpu/gpu_device.cc:1402] Created TensorFlow device (/device:GPU:0 with 539 MB memory) -> physical GPU (device: 0, name: Xavier, pci bus id: 0000:00:00.0, compute capability: 7.2)
['/device:GPU:0']
```
All the above confirms to us that we are indeed seeing and communicating with the GPU from within the K3s pod, but requesting GPU resource for the pod we need to add “resources” to "containers" in the deployment YAML file, such as:

```
containers:
...
  image: local_image
  imagePullPolicy: Never
  name: example
  resources:
    limits:
      nvidia.com/gpu: "1"
```
When we deploy the deployment YAML file, we will get “0/1 nodes are available: 1 Insufficient nvidia.com/gpu“ for the pod description message. To be able to actually request GPU resource, we need to run https://github.com/NVIDIA/k8s-device-plugin. First, take a look at the GitHub repository README, notice that k8s-device-plugin needs to run in the nvidia-docker runtime. Our boards are already on nvidia-docker, but nvidia-docker version will tell you we are on version 2.0.3, way behind the current, at the time of writing, 2.6.0. But whatever you do DO NOT update. Our boards can not handle higher versions. Currently if you `kubectl create -f https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/v0.9.0/nvidia-device-plugin.yml` (SCP the file into the board if link is blocked in China) you will get “OCI runtime create failed” which happens usually because of incompatible platform (AMD64 vs ARM64). If we use the latest ARM64 image (Docker Hub) by changing the “image” in nvidia-device-plugin.yml to point to “nvcr.io/nvidia/k8s-device-plugin:devel-ubuntu20.04”, we will get “Failed to initialize NVML: could not load NVML library.” due to the NVML library is again only for dedicated GPU’s and does not support Jetson/Tegra (But it’s coming: 
https://gitlab.com/nvidia/kubernetes/device-plugin/-/merge_requests/61 ”Evan Lezar @elezar (Owner) Jun 25th 2021 Hi @pmundt (and others) thanks for looking into this. We have finally been able to spend some cycles looking into this. We are definitely keen to get Tegra support for the device plugin going forward.” So check for new updates in the repository.) 

Alas, currently all thanks to Pablo Rodriguez Quesada of Wind River (https://github.com/NVIDIA/k8s-device-plugin/issues/132 / https://blogs.windriver.com/wind_river_blog/2020/06/nvidia-k8s-device-plugin-for-wind-river-linux/), by following his instructions exactly (Notice that it’s checking out “1.0.0-beta6" which is old, but master has already started supporting ARM64 so the first patch might raise conflicts, so keepin' it safe.): 
```
$ git clone -b 1.0.0-beta6 https://github.com/nvidia/k8s-device-plugin.git
$ cd k8s-device-plugin
$ wget https://labs.windriver.com/downloads/0001-arm64-add-support-for-arm64-architectures.patch
$ wget https://labs.windriver.com/downloads/0002-nvidia-Add-support-for-tegra-boards.patch
$ wget https://labs.windriver.com/downloads/0003-main-Add-support-for-tegra-boards.patch
$ git am 000*.patch
$ docker build -t nvidia/k8s-device-plugin:1.0.0-beta6 -f docker/arm64/Dockerfile.ubuntu16.04 .
$ sudo k3s kubectl apply -f nvidia-device-plugin.yml
```
We will be able to deploy k8s-device-plugin. Now that we `sudo k3s kubectl describe node`, we will see "nvidia.com/gpu 0 0" under "Allocated resources:", where it wasn't there before.  Now deploy the deployment YAML file again with “resources” added to "containers" as in above and run `jtop` immediately, watch and you will see some GPU activity at the start, whereas if you don’t add “resources” to "containers" there would be no GPU activity shown at all. Also, by `sudo k3s kubectl describe node` now we will see "nvidia.com/gpu 1 1".  

## Epilogue
Currently, in order to share GPU resource to more than one Docker Compose service in K8s/K3s, we must put these services in the same K8s deployment YAML file, since multi-deployment GPU sharing/concurrency is currently not supported with k8s-device-plugin. So deploying two deployments together with GPU resource `limits: nvidia.com/gpu: "1"` will yield “0/1 nodes are available: 1 Insufficient nvidia.com/gpu.” for one of them and will remain so until the one with GPU is deleted. 

Helpful links for resolving the GPU concurrency issue: 
- https://github.com/NVIDIA/k8s-device-plugin/issues/169
- https://aws.amazon.com/blogs/opensource/virtual-gpu-device-plugin-for-inference-workload-in-kubernetes/
- https://github.com/Deepomatic/shared-gpu-nvidia-k8s-device-plugin
