
---

# 通过 VSCode 的 Jupyter 插件远程连接 Kubernetes 中的 Jupyter Notebook Server

## 1. 前提条件
- **Kubernetes 集群**：已部署并运行。
- **Jupyter Notebook Server**：已部署在 Kubernetes 中。
- **VSCode**：已安装并配置好 Jupyter 插件。
- **kubectl**：已配置并可以访问 Kubernetes 集群。

---

## 2. 创建 Service 以通过 NodePort 暴露 Jupyter Notebook Server

### 2.1 创建 Service 的 YAML 文件
创建一个 `jupyter-service.yaml` 文件，内容如下：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: jupyter-notebook-service
spec:
  type: NodePort
  ports:
  - port: 8889
    targetPort: 8889
  selector:
    app: jupyter-notebook  # 请修改确保与 Jupyter Notebook Server 的 Pod 标签匹配
```

### 2.2 部署 Service
使用 `kubectl` 部署 Service：

```bash
kubectl apply -f jupyter-service.yaml
```

### 2.3 获取 NodePort 访问地址
1. 获取 Kubernetes 节点的 IP 地址：
   ```bash
   kubectl get nodes -o wide
   ```
   假设节点 IP 为 `192.168.1.100`。

2. 获取 Service 的 `NodePort`：
   ```bash
   kubectl get svc jupyter-notebook-service
   ```
   输出示例：
   ```
   NAME                     TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
   jupyter-notebook-service NodePort    10.96.123.45    <none>        8889:30000/TCP   10m
   ```
   - **NodePort**：`30000`（示例）。

Jupyter Notebook Server 的访问地址为：
```
http://192.168.1.100:30000
```

---

## 3. 获取 Jupyter Notebook Server 的 Token

### 3.1 查找 Jupyter Notebook Server 的 Pod 名称
使用以下命令查找 Jupyter Notebook Server 的 Pod 名称：

```bash
kubectl get pods
```

输出示例：
```
NAME                              READY   STATUS    RESTARTS   AGE
testhl-0   1/1     Running   0          10m
```

### 3.2 获取 Token
使用 `kubectl exec` 进入 Pod 并获取 Token：

```bash
kubectl exec -it testhl-0 -- jupyter notebook list
```

输出示例：
```
Currently running servers:
http://localhost:8888/?token=abc123def456ghi789 :: /home/jovyan
```
- **Token**：`abc123def456ghi789`（示例）。

---

## 4. 配置 VSCode 的 Jupyter 插件

### 4.1 安装 Jupyter 插件
在 VSCode 中安装 Jupyter 插件：
1. 打开 VSCode。
2. 进入扩展视图。
3. 搜索并安装 **Jupyter** 插件。

### 4.2 配置远程连接
1. 打开 VSCode，按下 `Ctrl+Shift+P`（或 `Cmd+Shift+P`）打开命令面板。
2. 搜索并选择 `Jupyter: Specify Jupyter Server for Connections`。
3. 选择 `Existing`，然后输入 Jupyter Notebook Server 的访问地址和 Token：
   ```
   http://192.168.1.100:30000/?token=abc123def456ghi789
   ```

### 4.3 验证连接
1. 在 VSCode 中打开一个 Jupyter Notebook 文件（`.ipynb`）。
2. 选择远程 Jupyter Notebook Server 的内核。
3. 运行代码单元，验证是否成功连接到远程服务器。

---

## 5. 常见问题及解决方法

### 5.1 连接失败
- **检查网络**：确保本地网络可以访问 Kubernetes 节点的 IP 和端口。
- **检查 Token**：确保输入的 Token 正确。
- **检查服务状态**：使用 `kubectl get pods` 和 `kubectl logs <pod-name>` 检查 Jupyter Notebook Server 的状态和日志。


---

## 6. 总结
通过以上步骤，你可以使用 VSCode 的 Jupyter 插件远程连接部署在 Kubernetes 中的 Jupyter Notebook Server，并通过 NodePort 对外暴露服务。如果需要更详细的配置或优化，可以参考 [Kubernetes 官方文档](https://kubernetes.io/docs/) 或 [VSCode Jupyter 插件文档](https://code.visualstudio.com/docs/datascience/jupyter-notebooks)。

--- 

