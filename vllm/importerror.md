

## cuTensorMapEncodeTiled函数
`cuTensorMapEncodeTiled` 是 NVIDIA CUDA 中的一个函数，主要用于处理 Tensor Memory Accelerator (TMA) 相关的操作，特别是在 Hopper 架构（如 sm_90+）的设备上。该函数用于创建和编码分块（tiled）的张量描述符，以便在 CUDA 内核中使用。

### 支持的 CUDA 版本
根据搜索结果，`cuTensorMapEncodeTiled` 是在 CUDA 12.x 版本中引入的，特别是针对 Hopper 架构（如 NVIDIA H100 GPU）的支持。具体来说：
- 该函数是 CUDA 12.x 的一部分，用于支持 TMA 功能，这是 Hopper 架构的一个重要特性。
- 它通常与 `cuTensorMapEncodeIm2col` 等函数一起使用，用于构建张量描述符并传递给 CUDA 内核。

### 相关背景
- **TMA（Tensor Memory Accelerator）**：TMA 是一种内存引擎，支持从全局内存到共享内存的异步数据拷贝，特别适用于 GEMM（矩阵乘法）等操作，能够显著减少寄存器压力。
- **Hopper 架构**：这是 NVIDIA 的最新 GPU 架构之一，支持 TMA 和其他高级特性，适用于高性能计算和深度学习任务。

`cuTensorMapEncodeTiled` 是 CUDA 12.x 版本中引入的函数，主要用于 Hopper 架构的设备（如 sm_90+），支持 TMA 功能。如果你使用的是 CUDA 12.x 及更高版本，并且设备支持 Hopper 架构，则可以调用该函数。

那么问题来了，测试环境当中的gpu型号是 Ada Lovelace 架构,为啥也会去引用该函数?
```
Wed Jan 15 14:01:36 2025
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 550.142                Driver Version: 550.142        CUDA Version: 12.4     |
|-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
|                                         |                        |               MIG M. |
|=========================================+========================+======================|
|   0  NVIDIA GeForce RTX 4090        On  |   00000000:00:0B.0 Off |                  Off |
| 30%   29C    P8             32W /  450W |    1461MiB /  24564MiB |      0%      Default |
|                                         |                        |                  N/A |
+-----------------------------------------+------------------------+----------------------+

+-----------------------------------------------------------------------------------------+
| Processes:                                                                              |
|  GPU   GI   CI        PID   Type   Process name                              GPU Memory |
|        ID   ID                                                               Usage      |
|=========================================================================================|
|    0   N/A  N/A      4467      C   /opt/conda/bin/python                        1454MiB |
+-----------------------------------------------------------------------------------------+

```


## vllm如何使用

```python
https://github.com/vllm-project/vllm/blob/main/vllm/_custom_ops.py

import contextlib
import importlib
from typing import TYPE_CHECKING, List, Optional, Tuple, Union

import torch
import torch.library

import vllm.envs as envs
from vllm.logger import init_logger
from vllm.platforms import current_platform
from vllm.scalar_type import ScalarType

logger = init_logger(__name__)

if not current_platform.is_tpu() and not current_platform.is_hpu():
    try:
        import vllm._C
    except ImportError as e:
        logger.warning("Failed to import from vllm._C with %r", e)


```
这里会去调用vllm的c模块，去校验需要加载的函数，应该也仅仅是校验而已,理论上是不是删除该校验也能运行

exec进去验证
```
oot@inferapi-test-vllm-api-predictor-default-test-deployment-7x4vmw:/vllm-workspace# python3
Python 3.12.6 (main, Sep 10 2024, 00:05:17) [GCC 9.4.0] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> import torch
>>> torch.cuda.is_available()
I0115 01:01:33.769391      41 logs.go:84] [core][Channel #1] Channel created
I0115 01:01:33.769670      41 logs.go:84] [core][Channel #1] original dial target is: "/etc/vcuda/vcuda.sock"
I0115 01:01:33.769798      41 logs.go:84] [core][Channel #1] parsed dial target is: resolver.Target{URL:url.URL{Scheme:"", Opaque:"", User:(*url.Userinfo)(nil), Host:"", Path:"/etc/vcuda/vcuda.sock", RawPath:"", OmitHost:false, ForceQuery:false, RawQuery:"", Fragment:"", RawFragment:""}}
I0115 01:01:33.769830      41 logs.go:84] [core][Channel #1] fallback to scheme "passthrough"
I0115 01:01:33.769868      41 logs.go:84] [core][Channel #1] parsed dial target is: passthrough:////etc/vcuda/vcuda.sock
I0115 01:01:33.769885      41 logs.go:84] [core][Channel #1] Channel authority set to "%2Fetc%2Fvcuda%2Fvcuda.sock"
I0115 01:01:33.770915      41 logs.go:84] [core][Channel #1] Resolver state updated: {
  "Addresses": [
    {
      "Addr": "/etc/vcuda/vcuda.sock",
      "ServerName": "",
      "Attributes": null,
      "BalancerAttributes": null,
      "Metadata": null
    }
  ],
  "Endpoints": [
    {
      "Addresses": [
        {
          "Addr": "/etc/vcuda/vcuda.sock",
          "ServerName": "",
          "Attributes": null,
          "BalancerAttributes": null,
          "Metadata": null
        }
      ],
      "Attributes": null
    }
  ],
  "ServiceConfig": null,
  "Attributes": null
} (resolver returned new addresses)
I0115 01:01:33.771154      41 logs.go:84] [core][Channel #1] Channel switches to new LB policy "pick_first"
I0115 01:01:33.771363      41 logs.go:84] [core][pick-first-lb 0xc00011e8d0] Received new config {
  "shuffleAddressList": false
}, resolver state {
  "Addresses": [
    {
      "Addr": "/etc/vcuda/vcuda.sock",
      "ServerName": "",
      "Attributes": null,
      "BalancerAttributes": null,
      "Metadata": null
    }
  ],
  "Endpoints": [
    {
      "Addresses": [
        {
          "Addr": "/etc/vcuda/vcuda.sock",
          "ServerName": "",
          "Attributes": null,
          "BalancerAttributes": null,
          "Metadata": null
        }
      ],
      "Attributes": null
    }
  ],
  "ServiceConfig": null,
  "Attributes": null
}
I0115 01:01:33.771456      41 logs.go:84] [core][Channel #1 SubChannel #2] Subchannel created
I0115 01:01:33.771497      41 logs.go:84] [core][Channel #1] Channel Connectivity change to CONNECTING
I0115 01:01:33.771555      41 logs.go:84] [core][Channel #1] Channel exiting idle mode
I0115 01:01:33.771634      41 logs.go:84] [core][Channel #1 SubChannel #2] Subchannel Connectivity change to CONNECTING
I0115 01:01:33.771678      41 logs.go:84] [core][Channel #1 SubChannel #2] Subchannel picks a new address "/etc/vcuda/vcuda.sock" to connect
I0115 01:01:33.772021      41 logs.go:84] [core][pick-first-lb 0xc00011e8d0] Received SubConn state update: 0xc00011ea50, {ConnectivityState:CONNECTING ConnectionError:<nil>}
I0115 01:01:33.773769      41 logs.go:84] [core][Channel #1 SubChannel #2] Subchannel Connectivity change to READY
I0115 01:01:33.773831      41 logs.go:84] [core][pick-first-lb 0xc00011e8d0] Received SubConn state update: 0xc00011ea50, {ConnectivityState:READY ConnectionError:<nil>}
I0115 01:01:33.773849      41 logs.go:84] [core][Channel #1] Channel Connectivity change to READY
I0115 01:01:33.781335      41 logs.go:84] [core][Channel #1] Channel Connectivity change to SHUTDOWN
I0115 01:01:33.781385      41 logs.go:84] [core][Channel #1] Closing the name resolver
I0115 01:01:33.781427      41 logs.go:84] [core][Channel #1] ccBalancerWrapper: closing
I0115 01:01:33.781494      41 logs.go:84] [core][Channel #1 SubChannel #2] Subchannel Connectivity change to SHUTDOWN
I0115 01:01:33.781536      41 logs.go:84] [core][Channel #1 SubChannel #2] Subchannel deleted
I0115 01:01:33.781573      41 logs.go:84] [transport][client-transport 0xc000308488] Closing: rpc error: code = Canceled desc = grpc: the client connection is closing
I0115 01:01:33.781717      41 logs.go:84] [core][Channel #1] Channel deleted
True
>>> torch.randn(5,5).cuda()
tensor([[-0.4785, -0.5469,  0.8985,  0.8653,  1.1294],
        [-0.3128, -0.6142, -1.8687,  0.0550,  0.4594],
        [-0.8542,  0.7507, -0.6035,  0.5503, -0.4111],
        [ 1.3127,  0.7197, -0.3040, -2.2406, -0.2125],
        [-0.8676, -0.4034,  1.0511,  1.0231,  0.7546]], device='cuda:0')
>>>

```


ps 
环境上修改了 /usr/local/lib/python3.12/dist-packages/vllm/_custom_ops.py 貌似也可以绕过去,不推荐，仅自己尝试

# 总结
影响范围很小，如果没有使用该函数，理论不存在任何影响。目前已知的是vllm会在启动时存在校验
