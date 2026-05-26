# TPM 可信启动 Demo 演示脚本

## 演示目标

展示 K8s 集群中，前端如何通过 TPM PCR 校验确保**只与运行在可信环境中的后端通信**。当后端从可信节点迁移到普通节点时，前端自动检测到环境不可信并拒绝通信。

---

## 环境准备

### 集群拓扑（3 节点）

| 节点 | 角色 | 特性 | 标签 |
|------|------|------|------|
| **node-tpm** | TPM 可信节点 | 有 `/dev/tpm0`（华为云 QingTian TPM 2.0） | `node-type: tpm` |
| **node-1** | 普通节点 | 无 TPM | `node-type: regular` |
| **node-2** | 普通节点 | 无 TPM | `node-type: regular` |

### 前提条件

- K8s 1.24+ 集群已就绪，3 节点均 Ready
- 镜像已推送到 Registry（`tpm-llm`, `tpm-backend`, `tpm-frontend`）
- 从可信节点采集过真实 PCR 基线值（见 Step 0）

---

## Step 0：采集真实 PCR 黄金基线

> **目的：** 获取可信节点的真实 PCR 值，作为前端校验的基准线。

**操作：**
```bash
# 在 node-tpm 上读取 PCR 1 和 PCR 4 的真实值
kubectl debug node/node-tpm -it --image=alpine -- \
  cat /sys/class/tpm/tpm0/pcr-sha256 | grep -E "^(1|4):"
```

> 如果没有 kubectl debug，也可以先部署一个带 `USE_MOCK_TPM=false` + `PCR_INDICES=1,4` 的临时 backend pod 到 node-tpm，然后 `curl localhost:8000/attestation` 获取。

**预期结果：**
```json
{
  "pcrs": {
    "1": "sha256:a1b2c3d4...(真实值)",
    "4": "sha256:e5f6a7b8...(真实值)"
  },
  "trusted": true,
  "mock": false
}
```

**记录这两个值**，后续配置到前端 `PCR_GOLDEN_BASELINE` 环境变量中。

**✅ 验证指标：** `mock: false` 表示读取的是真实 TPM；记录下 PCR 1 和 PCR 4 的 sha256 值。

---

## Step 1：部署 LLM 服务

> LLM 是后端的依赖，先启动。

**操作：**
```bash
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tpm-llm
  namespace: tpm-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: tpm-llm
  template:
    metadata:
      labels:
        app: tpm-llm
    spec:
      containers:
        - name: tpm-llm
          image: your-registry.com/tpm/tpm-llm:v1.0
          ports:
            - containerPort: 8080
          resources:
            requests:
              cpu: "2"
              memory: 4Gi
            limits:
              cpu: "4"
              memory: 8Gi
          env:
            - name: LLAMA_ARG_MODEL
              value: /models/Qwen3.5-0.8B-Q4_K_M.gguf
            - name: LLAMA_ARG_ALIAS
              value: qwen3.5
            - name: LLAMA_ARG_CTX_SIZE
              value: "1024"
            - name: LLAMA_ARG_REASONING
              value: "off"
          readinessProbe:
            httpGet:
              path: /v1/models
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 10
---
apiVersion: v1
kind: Service
metadata:
  name: tpm-llm
  namespace: tpm-app
spec:
  selector:
    app: tpm-llm
  ports:
    - name: http
      port: 8080
      targetPort: 8080
EOF
```

**验证：**
```bash
kubectl get pods -n tpm-app -l app=tpm-llm
kubectl logs -n tpm-app -l app=tpm-llm --tail=10
```

**✅ 验证指标：**
- Pod 状态：`Running`，READY `1/1`
- 日志中出现 `llama server listening on http://0.0.0.0:8080` 或类似启动成功信息
- `curl` 测试：`kubectl exec -n tpm-app deploy/tpm-llm -- wget -q -O- http://localhost:8080/v1/models` 返回模型列表

---

## Step 2：部署 Backend 到 TPM 可信节点

> **关键：** 后端通过 nodeSelector 调度到有 TPM 的节点，启用真实 TPM 读取。

**操作：**
```bash
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tpm-backend
  namespace: tpm-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: tpm-backend
  template:
    metadata:
      labels:
        app: tpm-backend
    spec:
      # ★ 关键：调度到 TPM 节点
      nodeSelector:
        node-type: tpm
      containers:
        - name: tpm-backend
          image: your-registry.com/tpm/tpm-backend:v1.0
          ports:
            - containerPort: 8000
          env:
            # ★ 启用真实 TPM（非 mock）
            - name: USE_MOCK_TPM
              value: "false"
            - name: PCR_INDICES
              value: "1,4"
            - name: PCR_GOLDEN_BASELINE
              value: "1:sha256:<Step0采集的真实PCR1值>;4:sha256:<Step0采集的真实PCR4值>"
            - name: ENCLAVE_MODE
              value: "true"
            - name: LLM_BASE_URL
              value: http://tpm-llm:8080/v1/chat/completions
            - name: LLM_MODEL
              value: qwen3.5
            - name: SYSTEM_PROMPT
              value: "你是TPM可信医疗助手，运行在安全环境中。"
          # 需要访问宿主机 TPM 设备
          volumeMounts:
            - name: tpm-device
              mountPath: /dev/tpm0
              readOnly: true
          resources:
            requests:
              cpu: "500m"
              memory: 256Mi
          readinessProbe:
            httpGet:
              path: /health
              port: 8000
            initialDelaySeconds: 3
            periodSeconds: 10
      volumes:
        - name: tpm-device
          hostPath:
            path: /dev/tpm0
            type: CharDevice
---
apiVersion: v1
kind: Service
metadata:
  name: tpm-backend
  namespace: tpm-app
spec:
  selector:
    app: tpm-backend
  ports:
    - name: http
      port: 8000
      targetPort: 8000
EOF
```

**验证：**
```bash
# 确认 Pod 调度到了 tpm 节点
kubectl get pods -n tpm-app -l app=tpm-backend -o wide

# 测试 attestation 接口
kubectl exec -n tpm-app deploy/tpm-frontend -- \
  wget -q -O- http://tpm-backend:8000/attestation
```

**✅ 验证指标：**
- Pod 的 `NODE` 列显示 `node-tpm`
- `/attestation` 返回 `"mock": false`（真实 TPM）
- `/attestation` 返回的 PCR 值与 Step 0 采集的基线**一致**
- Backend 日志：`[TPM] PCR read successful, indices=[1,4]`（无报错）

---

## Step 3：部署 Frontend 到普通节点

> 前端配置了与后端相同的黄金基线，用于独立校验。

**操作：**
```bash
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tpm-frontend
  namespace: tpm-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: tpm-frontend
  template:
    metadata:
      labels:
        app: tpm-frontend
    spec:
      # 前端部署在普通节点（无需 TPM）
      nodeSelector:
        node-type: regular
      containers:
        - name: tpm-frontend
          image: your-registry.com/tpm/tpm-frontend:v1.0
          ports:
            - containerPort: 3000
          env:
            - name: BACKEND_URL
              value: http://tpm-backend:8000
            - name: NEXT_PUBLIC_API_BASE
              value: http://tpm-backend:8000
            - name: NODE_ENV
              value: production
            # ★ 黄金基线 — 格式：逗号分隔（前端用）
            - name: PCR_GOLDEN_BASELINE
              value: "1:sha256:<Step0采集的真实PCR1值>,4:sha256:<Step0采集的真实PCR4值>"
          resources:
            requests:
              cpu: "200m"
              memory: 256Mi
          readinessProbe:
            httpGet:
              path: /
              port: 3000
            initialDelaySeconds: 5
            periodSeconds: 10
---
apiVersion: v1
kind: Service
metadata:
  name: tpm-frontend
  namespace: tpm-app
spec:
  type: LoadBalancer
  selector:
    app: tpm-frontend
  ports:
    - name: http
      port: 80
      targetPort: 3000
EOF
```

**验证：**
```bash
# 确认 Pod 调度到了普通节点
kubectl get pods -n tpm-app -l app=tpm-frontend -o wide

# 获取前端访问地址
kubectl get svc -n tpm-app tpm-frontend
```

**✅ 验证指标：**
- Pod 的 `NODE` 列显示 `node-1` 或 `node-2`
- Service 分配到外部 IP/端口
- 前端 Pod 状态 `Running`，READY `1/1`

---

## Step 4：验证可信通信（正常场景）

> 前端部署到普通节点，后端在 TPM 节点 — 系统正常工作。

### 4.1 打开浏览器访问前端

```bash
# 获取前端地址
FRONTEND_IP=$(kubectl get svc -n tpm-app tpm-frontend -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo "访问 http://${FRONTEND_IP}"
```

### 4.2 观察前端页面状态

**✅ 验证指标（前端页面上应可见）：**

| 指标 | 预期值 | 含义 |
|------|--------|------|
| 连接状态指示灯 | 🟢 绿色 / "可信环境" | PCR 校验通过 |
| PCR 1 状态 | ✅ match | 前端基线与后端返回一致 |
| PCR 4 状态 | ✅ match | 前端基线与后端返回一致 |
| 整体信任判定 | `trusted: true` | 后端 trusted + 前端独立校验均通过 |

### 4.3 测试加密对话

在前端输入框输入健康问题（如："我最近头疼，可能是什么原因？"），发送。

**✅ 验证指标：**

| 指标 | 预期值 | 含义 |
|------|--------|------|
| 消息发送 | 成功发出，无报错 | HPKE 密钥协商 + 加密通信正常 |
| 回复生成 | LLM 返回回复内容 | 后端→LLM 链路畅通 |
| 浏览器控制台 | 无 `trusted: false` 日志 | 校验流程未中断 |
| Backend 日志 | `[HPKE] decrypt success` + `[LLM] stream complete` | 加解密和 LLM 调用均成功 |
| Frontend 日志 | `[API /attestation] ← trusted=true mock=false` | 前端确认可信 |

### 4.4 抓包验证（可选，增强说服力）

```bash
# 在前端 Pod 内抓包观察
kubectl exec -n tpm-app deploy/tpm-frontend -- \
  tcpdump -i eth0 -A -s 0 port 8000 | head -100
```

**✅ 验证指标：**
- HTTP 请求体中的 `ct` 字段是 base64 编码的密文，**不可读**
- 无法从网络流量中提取明文健康数据
- 响应也是加密的 NDJSON chunk

---

## Step 5：迁移 Backend 到普通节点

> **核心演示：** 将后端从 TPM 节点迁移到没有 TPM 的普通节点。

### 5.1 修改 nodeSelector

```bash
kubectl patch deployment tpm-backend -n tpm-app --type=json -p='[
  {"op": "replace", "path": "/spec/template/spec/nodeSelector/node-type", "value": "regular"}
]'
```

### 5.2 等待滚动更新完成

```bash
kubectl rollout status deployment/tpm-backend -n tpm-app --timeout=120s
```

**✅ 验证指标：**
- 新 Pod 的 `NODE` 列显示 `node-1` 或 `node-2`（不再是 node-tpm）
- Pod 状态 `Running`，READY `1/1`

### 5.3 确认后端行为变化

```bash
# 检查新后端的 attestation
kubectl exec -n tpm-app deploy/tpm-frontend -- \
  wget -q -O- http://tpm-backend:8000/attestation
```

**✅ 验证指标：**

| 指标 | 预期值 | 含义 |
|------|--------|------|
| `/dev/tpm0` 可达 | ❌ 不可用 | 普通节点无 TPM 设备 |
| PCR 读取 | 失败或返回 mock 值 | 无法读取真实 PCR |
| `mock` 字段 | `true`（降级到 mock）或接口报错 | TPM 不可用的降级行为 |

> **注意：** 具体行为取决于后端在无 TPM 时的降级策略：
> - 如果 `USE_MOCK_TPM=false` + `ENCLAVE_MODE=true`：后端可能**启动失败**或 `/attestation` 返回错误
> - 如果后端有 graceful degradation：返回 mock PCR 值（与黄金基线不匹配）

---

## Step 6：验证前端拒绝不可信通信

> **高潮部分：** 前端检测到后端不再可信，自动拒绝通信。

### 6.1 刷新前端页面

```bash
# 重新打开浏览器访问前端
echo "刷新 http://${FRONTEND_IP}"
```

### 6.2 观察前端页面变化

**✅ 验证指标（前端页面上应可见）：**

| 指标 | 预期值 | 含义 |
|------|--------|------|
| 连接状态指示灯 | 🔴 红色 / "不可信环境" | PCR 校验失败 |
| PCR 1 状态 | ❌ mismatch 或 missing | 值不匹配或无法获取 |
| PCR 4 状态 | ❌ mismatch 或 missing | 值不匹配或无法获取 |
| 整体信任判定 | `trusted: false` | 前端独立校验失败 |
| 通信状态 | "无法与后端建立安全连接" 或类似提示 | 前端拒绝发起加密通信 |
| 输入框 | 禁用或发送按钮不可点击 | 业务功能被阻断 |

### 6.3 尝试发送消息

在前端输入框尝试输入并发送消息。

**✅ 验证指标：**

| 指标 | 预期值 | 含义 |
|------|--------|------|
| 消息发送 | ❌ 被前端拦截，不发出请求 | 前端在发起请求前就拒绝了 |
| 浏览器控制台 | `[API /attestation] ← trusted=false` | 前端日志记录校验失败 |
| Network 面板 | 无 `/key-exchange` 和 `/chat` 请求 | 加密通信流程根本未启动 |
| Backend 日志 | 无新的请求记录 | 后端没有收到任何对话请求 |

### 6.4 查看前端日志确认

```bash
kubectl logs -n tpm-app -l app=tpm-frontend --tail=20
```

**✅ 验证指标：**
- 日志中出现：`[API /attestation] ← trusted=false mock=true` 或 `trusted=false mock=false`
- 无 `/key-exchange` 相关日志
- 无 `/chat` 相关日志

---

## Step 7：恢复 — 迁移 Backend 回 TPM 节点

> 证明系统是动态可恢复的。

```bash
# 迁移回 TPM 节点
kubectl patch deployment tpm-backend -n tpm-app --type=json -p='[
  {"op": "replace", "path": "/spec/template/spec/nodeSelector/node-type", "value": "tpm"}
]'

# 等待就绪
kubectl rollout status deployment/tpm-backend -n tpm-app --timeout=120s
```

**刷新前端页面。**

**✅ 验证指标：**

| 指标 | 预期值 | 含义 |
|------|--------|------|
| 连接状态 | 🟢 恢复绿色 / "可信环境" | PCR 校验重新通过 |
| 对话功能 | 恢复正常 | 加密通信链路重建 |
| 前端日志 | `[API /attestation] ← trusted=true` | 信任恢复 |

---

## 演示总结对照表

| 阶段 | 后端位置 | TPM 可用 | PCR 匹配 | 前端状态 | 能否通信 |
|------|----------|----------|----------|----------|----------|
| Step 4 | node-tpm | ✅ | ✅ | 🟢 可信 | ✅ 正常对话 |
| Step 6 | node-1/2 | ❌ | ❌ | 🔴 不可信 | ❌ 被拒绝 |
| Step 7 | node-tpm | ✅ | ✅ | 🟢 可信 | ✅ 恢复正常 |

---

## 关键技术点（旁白/解说词参考）

1. **前端独立校验**：PCR 黄金基线存在前端自己的环境变量中（`PCR_GOLDEN_BASELINE`），通过 Next.js API route 运行时读取，**不依赖后端返回的任何信任声明**
2. **双重校验**：信任判定 = `后端 trusted` AND `前端独立 PCR 校验全部 match`，任一失败即不可信
3. **零信任原则**：后端返回的 `pcrStatus` 被前端忽略，前端用自己持有的基线值独立比对
4. **加密通信**：通过校验后才建立 HPKE 密钥协商（X25519），健康数据全程加密传输
5. **动态调度感知**：K8s 节点调度变更后，前端在下次 attestation 请求时立即感知环境变化
