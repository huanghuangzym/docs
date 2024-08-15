# Istio multi gateway configured with same tls secret 404 error

## 问题描述

官方链接：https://istio.io/latest/docs/ops/common-problems/network-issues/#404-errors-occur-when-multiple-gateways-configured-with-same-tls-certificate

在 Istio 中，当使用 HTTPS http2 承载多路复用时，如果使用相同证书的gateway资源，是分开设置的，那么你可能会遇到404的问题
对于该问题，在alauda service mesh 产品存在如下解决方案
分别为
* **临时方案 **： 禁用http2，强制让alpn协商仅支持http1.1。 
* **推荐方案 **： 合并使用相同证书的gateway资源

## 如何排查确认
关于如何排查确认您的环境是否存在该问题
我们提供了如下脚本进行参考
```bash
#!/bin/bash

nslist=$(kubectl get ns  -o jsonpath='{.items[*].metadata.name}')
declare -A cred_map

echo "begin to check gw"
for ns in $nslist; do
  # 获取 gw资源
  #echo "begin to list gw in $ns"
  gateways=$(kubectl get gw -n $ns -o jsonpath='{.items[*].metadata.name}')
  # 获取 Gateway 资源的 YAML 文件
  for gateway in $gateways; do
    gateway_yaml=$(kubectl get gw -n $ns $gateway  -o yaml)
    gateway_json=$(kubectl get gw -n $ns $gateway  -o json)

    tls_lines=$(echo "$gateway_yaml" | grep  'credentialName:')
    secname=$(echo "$gateway_yaml" | grep  'credentialName:'|awk '{print $2}')

    if [[ -n "$tls_lines" ]]; then
      found=false
      for key in "${!cred_map[@]}"; do
        if [[ "$key" == "$secname" ]]; then
          found=true
          break
        fi
      done

      if [[ $found == true ]]; then
        echo -e "\033[31m cred already exist in other gw resource ,please must merge hosts in the  gw resource  ${cred_map[$secname]} ,and delete this gw!  \033[0m"
        hosts=$(echo "$gateway_json" | jq -r '.spec.servers[] | .hosts[]')
        # 输出 Gateway 名称和 hosts 信息
        echo -e  "\033[31m invalid gw name namespace: $gateway ,  $ns \033[0m"
        echo "Hosts: $hosts"
      else
        echo "first get secret name $secname the gw is $gateway $ns"
        cred_map["$secname"]="$gateway~$ns"
      fi


      #for key in "${!cred_map[@]}"; do
        #echo "Key: $key, Value: ${cred_map[$key]}"
      #done

      echo ""
    fi
  done
done

```
您可以将脚本复制到业务集群的master节点，保存并执行
当执行结果出现如下描述时，代表您在多个gw资源上使用了相同的证书，存在404的可能。
```bash

[root@idp-lihuang-w9x9w-9n9jv-cluster0-dt2n4 gwtls]# sh check.sh 
begin to check gw
first get secret name jiaxiurc-com the gw is drawdb-gateway drawdb

first get secret name gyssg-com the gw is ec jxb-ec

first get secret name nexus the gw is nexus-gateway nexus

 cred already exist in other gw resource ,please must merge hosts in the  gw resource  drawdb-gateway~drawdb ,and delete this gw!  
 invalid gw name namespace: authory-gateway ,  nm-edu-authory 
Hosts: rzzx-test.jiaxiurc.com
rzzx-test.jiaxiurc.com

 cred already exist in other gw resource ,please must merge hosts in the  gw resource  drawdb-gateway~drawdb ,and delete this gw!  
 invalid gw name namespace: careers-gateway ,  nm-edu-careers 
Hosts: rcyy-test.jiaxiurc.com
szrc-test-jiaxiurc.gyssg.com
rcyyapi-test.jiaxiurc.com
rcyy-test.jiaxiurc.com
szrc-test-jiaxiurc.gyssg.com
rcyyapi-test.jiaxiurc.com

 cred already exist in other gw resource ,please must merge hosts in the  gw resource  drawdb-gateway~drawdb ,and delete this gw!  
 invalid gw name namespace: careers-isv-gateway ,  nm-edu-careers-isv 
Hosts: api-szrc-test.gyssg.com
rcnlapi-test.jiaxiurc.com
company-szrc-test.gyssg.com
rcqy-test.jiaxiurc.com
m-szrc-test.gyssg.com
rcyd-test.jiaxiurc.com
school-szrc-test.gyssg.com
rcyx-test.jiaxiurc.com
api-szrc-test.gyssg.com
rcnlapi-test.jiaxiurc.com
company-szrc-test.gyssg.com
rcqy-test.jiaxiurc.com
m-szrc-test.gyssg.com
rcyd-test.jiaxiurc.com
school-szrc-test.gyssg.com
rcyx-test.jiaxiurc.com

 cred already exist in other gw resource ,please must merge hosts in the  gw resource  drawdb-gateway~drawdb ,and delete this gw!  
 invalid gw name namespace: trade-center-gateway ,  nm-edu-paycenters 
Hosts: jyzx-test.jiaxiurc.com
jyzx-test.jiaxiurc.com

 cred already exist in other gw resource ,please must merge hosts in the  gw resource  ec~jxb-ec ,and delete this gw!  
 invalid gw name namespace: databases-gateway ,  share-databases 
Hosts: *
*

first get secret name knowledge-yottacloud-cn the gw is yottacloud-confluence-gw yottacloud-confluence

 cred already exist in other gw resource ,please must merge hosts in the  gw resource  drawdb-gateway~drawdb ,and delete this gw!  
 invalid gw name namespace: cydd-gateway ,  zj-chanyedingdan 

```

当出现例如 cred already exist in other gw resource ,please must merge hosts in the  gw resource  drawdb-gateway~drawdb ,and delete this gw! 描述代表您有不正确的使用

1. ** 如果很幸运 **: 脚本执行完无上述输出，那么代表您的使用正确,可以忽略下面的内容
2. ** 如果很不幸 **: 脚本执行完发现您的环境存在该问题，下面我们聊下该如何解决


## 禁止http2方案

### 前提条件

当您使用的acp版本 小于 3.16.1 时，推荐使用此方案
该方案会在https alpn协商协议时，禁用掉http2
由于asm发布的每个版本包含细微差别，我们给的方案是同时去调整istiooperator和网关deploy两个资源，确保可以解决您的问题

### 如何实施

#### 1 调整istiooperator yaml
执行如下命令，获取网关安装使用的istiooperator资源
```bash
kubectl get istiooperators.install.istio.io  -n istio-system
```
按照如下示例修改istiooperator的yaml

```yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  labels:
    asm.cpaas.io/component: gateway
    asm.cpaas.io/gateway-type: ingress
    asm.cpaas.io/owning-resource: edge-gw
    asm.cpaas.io/owning-resource-ns: testhl
  name: edge-gw
  namespace: istio-system
spec:
  components:
    ingressGateways:
    - enabled: true
      k8s:
        hpaSpec:
          maxReplicas: 1
          minReplicas: 1
        #在如下段，增加asm.cpaas.io/h2-enabled: "no"  和 asm.cpaas.io/h2-workaround404Issue: "no"配置
        podAnnotations:
          asm.cpaas.io/h2-enabled: "no"  
          asm.cpaas.io/h2-workaround404Issue: "no"
          proxy.istio.io/config: |-
            proxyStatsMatcher:
              inclusionRegexps:
              - .*downstream_cx_total
        replicaCount: 1
```
#### 2 调整网关deploy的yaml
按照如下实例，调整网关部署deployment的yaml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: edge-gw
  namespace: istio-system
spec:
  progressDeadlineSeconds: 600
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: edge-gw
      istio: testhl-edge-gw
  strategy:
    rollingUpdate:
      maxSurge: 100%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      annotations:
        #在这里增加下面两个annotation
        asm.cpaas.io/h2-enabled: "no"
        asm.cpaas.io/h2-workaround404Issue: "no"
        istio.io/rev: 1-22
        prometheus.io/path: /stats/prometheus
        prometheus.io/port: "15020"
        prometheus.io/scrape: "true"
        proxy.istio.io/config: |-
          proxyStatsMatcher:
            inclusionRegexps:
            - .*downstream_cx_total
        sidecar.istio.io/inject: "false"
      labels:
        chart: gateways
```


## gateway合并方案

### 前提条件

当您使用的acp版本 大于等于 3.16.1 时，推荐此方案
此方案也是istio社区默认推荐方案

依然举个直观的例子
### 错误样例
给一个典型的错误的yaml样例
```yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: default2
  namespace: istio-system
spec:
  selector:
    istio: ingressgateway
  servers:
  - hosts:
    - "asm2.test.com"
    tls:
      mode: SIMPLE
      credentialName: "testhl"
    port:
      name: https
      number: 443
      protocol: HTTPS
---
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: default
  namespace: istio-system
spec:
  selector:
    istio: ingressgateway
  servers:
  - hosts:
    - "asm1.test.com"
    tls:
      mode: SIMPLE
      credentialName: "testhl"
    port:
      name: https
      number: 443
      protocol: HTTPS
---
#这种也是错误的，虽然在同一个gw资源，但是不在同一个hosts段落
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: error-3
  namespace: istio-system
spec:
  selector:
    istio: ingressgateway
  servers:
  - hosts:
    - "asm1.test.com"
    tls:
      mode: SIMPLE
      credentialName: "testhl"
    port:
      name: https-2
      number: 443
      protocol: HTTPS
  - hosts:
    - "asm2.test.com"
    tls:
      mode: SIMPLE
      credentialName: "testhl"
    port:
      name: https
      number: 443
      protocol: HTTPS
---
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: default2
  namespace: bus-system
spec:
  gateways:
  - istio-system/default2
  hosts:
  - asm2.test.com
  http:
  - route:
    - destination:
        host: asm-0.testhl.svc.cluster.local
        port:
          number: 80
  ...
---
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: default
  namespace: bus-system
spec:
  gateways:
  - istio-system/default
  hosts:
  - asm1.test.com
  ...

```
在这个例子当中，我们使用同一个证书，签发了asm1.test.com和asm2.test.com
然后分别在两个gateway资源进行了定义使用.

下面看看如何修改成正确的yaml吧
```yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: default2
  namespace: istio-system
spec:
  selector:
    istio: ingressgateway
  servers:
  - hosts:
    - "asm2.test.com"
    - "asm1.test.com"
    tls:
      mode: SIMPLE
      credentialName: "testhl"
    port:
      name: https
      number: 443
      protocol: HTTPS
---
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: default1
  namespace: istio-system
spec:
  gateways:
  - istio-system/default2
  hosts:
  - asm2.test.com
  ...
---
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: default2
  namespace: istio-system
spec:
  gateways:
  - istio-system/default2
  hosts:
  - asm1.test.com
  ...
```

也可以写成泛域名的形式
```yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: default2
  namespace: istio-system
spec:
  selector:
    istio: ingressgateway
  servers:
  - hosts:
    - "*.test.com"
    tls:
      mode: SIMPLE
      credentialName: "testhl"
    port:
      name: https
      number: 443
      protocol: HTTPS
```


**修改步骤：**

1. **合并 Gateway 资源的spec.servers.hosts段：** 将所有使用同一secret证书的gateway资源进行合并到spec的同一个server段(在同一个gw里，但是分开在不同server也不可以)，将hosts段写在一起或者直接写成泛域名格式
2. **检查 vs 资源的spec.gateway配置：** 修改所有相关的vs资源当中的gateways字段，选中新合并的gateway资源，可以进行夸ns引用
3. **检查 vs 资源的route dest配置：** 确保vs的destination使用的都是k8s的fqdn写法

当您执行完上述步骤，可以重复执行检查脚本，确认没有遗漏的输出


**总结：**

通过以上方法，您可以有效地解决 Istio 在 HTTPS 多路复用下 404的问题。请根据您的具体情况选择合适的解决方法，并进行相应的配置调整。

