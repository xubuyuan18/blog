---
title: 我的毕设
tags:
  - K8s
  - kvm
  - llm
categories: AI
abbrlink: 26728
date: 2026-04-10 11:43:50
---

# 三节点 K8s + Ollama 本地大模型 + AMD GPU：我的毕设全记录

> 一个毕业生如何在 Fedora 笔记本上搭出一套企业内网私有化 AI 知识库

---

## 缘起

去年选题的时候，我就想："现在大模型这么火，你能不能做一个能在企业内部用的 AI 知识库？数据不能上公有云，全得跑在局域网里。"

我想了想——这不就是 K8s 搭集群 + 本地跑模型 + 做个网页的事吗？行，干。

后来的几个月，我把一台 AMD 笔记本（16 核 22GB 内存，RX 6700S 独显）折腾成了三节点 Kubernetes 集群，接上 Ollama 本地大模型，写了套 RAG 检索增强生成的知识库，还配了 Prometheus/Grafana 监控。最后跑通了完整闭环：上传文档 → 向量检索 → 大模型回答。

这篇博客就是整个项目的全记录。

---

## 一句话说清楚这个项目

> 在公司局域网里搭一个三节点的 AI 知识库——员工上传文档，AI 从文档里找答案，全程数据不出公司。

---

## 整体架构

```
┌────────────────────────────────────────────────────────┐
│                   企业内网 10.10.10.0/24                 │
│                                                        │
│  ┌──────────────────┐  ┌──────────────────┐            │
│  │  k8s-master      │  │  k8s-rag         │            │
│  │  Ubuntu 22.04 VM │  │  Ubuntu 22.04 VM │            │
│  │  4GB / 2 vCPU    │  │  6GB / 2 vCPU    │            │
│  │  k3s server      │  │  k3s agent       │            │
│  │  集群控制平面      │  │  Flask RAG :8080 │            │
│  │                  │  │  FAISS + SQLite  │            │
│  └────────┬─────────┘  └────────┬─────────┘            │
│           │                     │                      │
│  ┌────────┴─────────────────────┴─────────┐            │
│  │         k8sbr0 网桥                     │            │
│  │                                        │            │
│  │  ┌──────────────────────────────────┐  │            │
│  │  │  fedora (物理机, 22GB / 16核)      │ │             │
│  │  │  Ollama 本地大模型 :11434          │  │            │
│  │  │  RX 6700S GPU (8GB) · ROCm       │  │            │
│  │  │  Prometheus :9090 · Grafana :3000│  │            │
│  │  └──────────────────────────────────┘  │            │
│  └────────────────────────────────────────┘            │
│                                                         │
│  用户浏览器 ──→ RAG Web :8080 ──→ Ollama :11434           │
└─────────────────────────────────────────────────────────┘
```

三台机器的分工：

| 节点 | 类型 | 内存 | 干什么 |
|------|------|------|--------|
| k8s-master | 虚拟机 | 4GB | K8s 控制平面，管理整个集群 |
| k8s-rag | 虚拟机 | 6GB | 跑 Flask Web 服务、FAISS 向量库、SQLite 数据库 |
| fedora | 物理机 | 22GB | 跑 Ollama 大模型 + RX 6700S GPU + Prometheus/Grafana 监控 |

---

## RAG 知识库是怎么工作的

RAG 全称 Retrieval Augmented Generation（检索增强生成），是这个项目的核心。

### 原理就是用三步走：

```
第一步：文档向量化
  上传的文档 → 切片成小块 → 调用 Ollama 把每块变成一串数字（向量）
  → 存进 FAISS 向量数据库

第二步：语义检索
  用户提问 → 同样向量化 → 去 FAISS 里找最相似的几个文档片段

第三步：生成回答
  把搜到的文档片段 + 用户问题拼成一段 prompt → 发给 Ollama
  → 大模型基于文档内容生成答案 → 返回给用户
```

通俗版：

> 你把公司制度文档喂给它，然后问"请假要几天？"，它会先去文档里把相关内容翻出来，再让 AI 帮你总结成一段通顺的话。答案后面还会标注"根据哪几段文档"，防止 AI 瞎编。

### 技术栈

| 层 | 用了什么 |
|----|----------|
| 前端 | 原生 HTML/CSS/JS（无框架，单文件 Flask 渲染） |
| 后端 API | Flask + Python |
| 向量数据库 | FAISS（Facebook 开源，轻量级，适合原型验证） |
| 向量化 | Ollama `/api/embeddings` |
| 大模型推理 | Ollama `/api/chat` + qwen2:1.5b |
| 业务数据库 | SQLite（存用户、会话、聊天记录） |
| GPU 加速 | AMD ROCm 7.1.1 + RX 6700S |
| 容器编排 | k3s（轻量级 Kubernetes） |
| 监控 | Prometheus + Grafana + node_exporter + rocm_exporter |

---

## 为什么要用 K8s？

这个问题的答案很抽象

其实这个项目规模只有 3 节点，裸跑 docker 也能用。但 K8s 给了三个关键能力：

1. **资源隔离**：创建 namespace + ResourceQuota，不同业务用不同配额，不会互相抢资源
2. **声明式管理**：`kubectl create role` 一条命令就配好权限，不用每台机器手改
3. **可扩展**：如果将来公司再加 5 台服务器，`kubeadm join` 一条命令直接扩容

论文里选了 k3s 而不是完整 kubeadm，因为 k3s 只要 50MB 二进制文件，特别适合实验环境。它是 CNCF 官方认证的 K8s 发行版，API 完全兼容。

---

## AMD 显卡跑 AI 模型的坑

这个项目最难的部分不是写代码，是让 AMD 显卡跑起来。

Ollama 默认支持 NVIDIA CUDA，但我的笔记本是 AMD RX 6700S（Navi 23，8GB 显存）。要让 Ollama 用上这块卡，需要装 ROCm 7.1.1。

踩过的坑：

```text
1. rocBLAS 没装 → Ollama 只识别 CPU → 装上后识别 GPU
2. HSA_OVERRIDE_GFX_VERSION 不设 → GPU 不可见 → 设为 10.3.0
3. /opt/rocm/lib 符号链接缺失 → Ollama 找不到库 → 手动 ln -s
4. 装完 ROCm 不重启 → 内核模块没加载 → reboot
```

最终这条日志让我觉得一切都值了：

```text
inference compute library=ROCm compute=gfx1030
  name=ROCm0 description="AMD Radeon RX 6700S"
  type=discrete total="8.0 GiB" available="7.9 GiB"
```

那一刻我真的理解了：**环境配通本身就是工程能力**。

---

## 常用命令速查

### 集群管理

```bash
# SSH 到控制节点
ssh ubuntu@10.10.10.138

# 查看全部节点
sudo kubectl get nodes -o wide

# 查看命名空间
sudo kubectl get ns

# 创建资源配额
kubectl create quota test-quota -n dify-namespace --hard=cpu=2,pods=10,memory=4Gi

# RBAC 权限验证
kubectl auth can-i get configmaps --as=test-user -n dify-namespace
```

### Ollama

```bash
# 查看已安装模型
ollama list

# 拉取新模型
ollama pull qwen2:7b

# 查看 GPU 识别状态
journalctl -u ollama -n 30 | grep -E "ROCm|RX 6700S"

# 查看 GPU 实时指标
curl http://127.0.0.1:9400/metrics | grep amd_gpu
```

### 恢复系统

```bash
# 关机重启后一键恢复所有服务
recover

# 查看全系统状态仪表盘
status
```

---

## 整个项目的时间线

```text
Day 1   选型调研：K8s 还是 Docker Compose？
Day 2   装 KVM + 创建 2 台 Ubuntu 虚拟机
Day 3   搭 k3s 集群，3 节点全部 Ready
Day 4   装 Ollama，踩 AMD GPU 的坑
Day 5   写 RAG 核心：chunk_text → embedding → FAISS → chat
Day 6   写 Flask Web + HTML 前端
Day 7   加登录系统、管理员账号管理
Day 8   加多会话持久化
Day 9   装 Prometheus + Grafana + 自己写 rocm_exporter
Day 10  反复调试、截图、写论文
```

---

## 我学到了什么

### 1. 不要被"高大上"吓到

刚看论文题目时觉得 K8s + AI + GPU 好像很复杂。实际做下来发现，每块拆开都是可攻克的：K3s 一条命令装好、Ollama 一条命令拉模型、FAISS 两行代码建索引。关键是想清楚 **数据怎么流转**。

### 2. 环境配置比写代码更耗时

这个项目 60% 的时间在配环境：
- KVM 虚拟机网络桥接
- AMD ROCm 驱动适配
- Ollama 找到 GPU
- 各种 systemd 服务文件

代码反而是最简单的部分。

### 3. 先吹牛再落地，发现不对劲赶紧改

论文初稿写得挺大：Dify 平台、kubeadm 集群、3 台 2C2G 虚拟机、GPU 占用率 65%。然后开始动手做，发现根本不是那么回事。

2C2G 虚拟机连 K8s 都跑不稳，Dify 镜像太大拉不下来，GPU 调度也远比想象中复杂。好在及时意识到了问题，果断调整方案：换成 k3s 轻量 K8s、自己用 Flask + FAISS 撸了一个 RAG 引擎、把模型推理搬到物理机上用 ROCm 加速。

改完之后项目反而更扎实了——因为每一行代码、每一个配置都是亲手调的，不再依赖现成平台。**现在回头想，早期那些不切实际的设想反而是好事——它让我在落地过程中被迫理解了每一层的原理。**

### 4. 原型系统的价值在于"跑通"

这个系统不是生产环境能用的产品，但它证明了：

> 一个毕业生用一台笔电，真的可以在内网环境里跑通 K8s + GPU + RAG 的完整流程。

这就是毕业设计的价值。

---

## 附：项目文件

```text
~/ai_k8s_status.sh          状态仪表盘脚本
~/ai_k8s_recover.sh          关机后恢复脚本
~/基于 Kubernetes 的...md   论文文件

http://10.10.10.175:8080    RAG 知识库网页
http://10.115.234.102:9090  Prometheus
http://10.115.234.102:3000  Grafana (admin/admin)
```

---

*2026年4月，by 一个即将毕业的计算机系学生*
