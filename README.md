# 🛒 多Agent电商推荐与营销系统

> 基于 LangGraph 的企业级 Multi-Agent 电商推荐系统，采用 Supervisor 模式编排四大专业 Agent，实现从用户画像到个性化推荐的全链路智能化。

[![Python](https://img.shields.io/badge/Python-3.11%2B-blue?logo=python)](python/)
[![License](https://img.shields.io/badge/License-MIT-green)](LICENSE)

---

## 📖 目录

1. [项目简介](#-项目简介)
2. [系统架构](#-系统架构)
3. [四大核心 Agent](#-四大核心-agent)
4. [关键代码展示](#-关键代码展示)
5. [快速上手](#-快速上手)
6. [API 接口](#-api-接口)
7. [项目结构](#-项目结构)
8. [技术亮点](#-技术亮点)
9. [参考资料](#-参考资料)

---

## 🤔 项目简介

### 核心价值

通过 Multi-Agent 技术，让电商平台的推荐、文案、库存系统协同工作，为每位用户生成个性化推荐结果。

### 解决的痛点

| 痛点 | 传统做法 | 本系统做法 |
|------|---------|-----------|
| 推荐与库存脱节 | 推荐缺货商品 | 库存 Agent 实时校验，缺货自动剔除 |
| 营销文案千篇一律 | 统一广告语 | 文案 Agent 根据用户画像生成个性化文案 |
| 各系统各自为战 | 系统间互不感知 | Supervisor 统一编排，结果实时联动 |

### 技术关键词

`Multi-Agent` · `Supervisor模式` · `LangGraph` · `asyncio并行` · `Redis Feature Store` · `A/B Testing` · `Thompson Sampling` · `RAG` · `ReAct`

---

## 🏗 系统架构

### 整体架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                         用户发起推荐请求                           │
│                    {"user_id": "u001", "num_items": 5}           │
└───────────────────────────────┬─────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Supervisor 协调Agent                           │
│                  (python/orchestrator/supervisor.py)              │
│                                                                   │
│  ════════════════ Phase 1: 并行执行 ═══════════════════           │
│  ┌──────────────────────┐    ┌──────────────────────┐            │
│  │   用户画像 Agent      │    │   商品召回 Agent      │            │
│  │  user_profile_agent  │    │  product_rec_agent   │            │
│  │  ──────────────────  │    │  ────────────────── │            │
│  │  Redis → 实时行为特征 │    │  协同过滤+向量检索召回 │            │
│  │  RFM模型 → 用户分群   │    │  返回候选商品列表     │            │
│  └──────────┬───────────┘    └──────────┬──────────┘            │
│             │                           │                         │
│  ════════════════ Phase 2: 并行执行 ═══════════════════           │
│  ┌──────────────────────┐    ┌──────────────────────┐            │
│  │   LLM重排 Agent      │    │   库存决策 Agent      │            │
│  │  (product_rec再次调用)│    │   inventory_agent    │            │
│  │  ──────────────────  │    │  ────────────────── │            │
│  │  用户画像 × 商品属性  │    │  SQLite → 库存查询   │            │
│  │  LLM精排，返回TopN   │    │  过滤缺货，限购策略   │            │
│  └──────────┬───────────┘    └──────────┬──────────┘            │
│             │                           │                         │
│  ════════════════ Phase 3: 串行执行 ═══════════════════           │
│             └──────────────┬────────────┘                         │
│                            ▼                                      │
│             ┌──────────────────────────────┐                      │
│             │      结果聚合器               │                      │
│             │  库存过滤 → 排序合并 → TopN   │                      │
│             └──────────────┬───────────────┘                      │
│                            ▼                                      │
│             ┌──────────────────────────────┐                      │
│             │   营销文案 Agent              │                      │
│             │  marketing_copy_agent        │                      │
│             │  ────────────────────────── │                      │
│             │  5套Prompt模板 × 用户分群    │                      │
│             │  LLM生成 + 广告法合规校验    │                      │
│             └──────────────┬───────────────┘                      │
│                            ▼                                      │
│             ┌──────────────────────────────┐                      │
│             │   A/B 测试引擎               │                      │
│             │  用户ID哈希分桶              │                      │
│             │  Thompson Sampling 动态调优  │                      │
│             └──────────────┬───────────────┘                      │
└──────────────────────────────┬──────────────────────────────────┘
                               ▼
              ┌─────────────────────────────────┐
              │  个性化推荐响应（返回给用户）      │
              │  商品列表 + 个性化文案 + 实验分组 │
              └─────────────────────────────────┘
```

---

## 🤖 四大核心 Agent

### Agent 1：用户画像 Agent

**文件**：`python/agents/user_profile_agent.py`

**功能**：将用户历史行为数据转化为结构化用户画像，供其他 Agent 使用。

**关键技术**：
- Redis Sorted Set：存储用户行为序列，支持滑动窗口查询
- RFM 模型：Recency × Frequency × Monetary
- 用户分群：新客 / VIP / 价格敏感 / 活跃 / 流失风险

### Agent 2：商品推荐 Agent

**文件**：`python/agents/product_rec_agent.py`

**功能**：两阶段推荐——多路召回 + LLM 精排。

```
多路召回策略
  ├── 协同过滤（买了A也买了B）
  ├── 向量检索（Milvus，语义相似）
  ├── 热度策略（最近7天热卖）
  └── 新品策略（上架30天内）
        │
        ▼（去重合并）
  LLM 精排 → TopN 商品列表
```

### Agent 3：营销文案 Agent

**文件**：`python/agents/marketing_copy_agent.py`

**功能**：根据用户画像动态选择文案模板，调用 LLM 生成个性化文案，并做合规校验。

```python
TEMPLATES = {
    "new_user":        "首单专属福利，{product}立减{discount}元！",
    "vip":             "尊享会员特权，{product}专属价{price}，品质之选。",
    "price_sensitive": "今日限时抢购！{product}历史最低价，仅剩{stock}件！",
    "active":          "根据您的浏览偏好，为您精选 {product}，好评率{rating}%",
    "churn_risk":      "好久不见！{product}为您专属保留，点击领取优惠券",
}
```

### Agent 4：库存决策 Agent

**文件**：`python/agents/inventory_agent.py`

**功能**：查询实时库存，过滤缺货商品，输出限购策略和补货预警。

---

## 💻 关键代码展示

### Supervisor 并行编排

**文件**：`python/orchestrator/supervisor.py`

```python
class SupervisorOrchestrator:
    """Supervisor 编排器 — 并行分发 + 聚合模式"""

    async def recommend(self, request: RecommendationRequest) -> RecommendationResponse:
        # A/B 实验分组
        experiment = self.ab_engine.assign(request.user_id)

        # Phase 1：用户画像 + 商品召回 并行执行
        profile_result, rec_result = await asyncio.gather(
            self.user_profile_agent.run(user_id=request.user_id, context=request.context),
            self.product_rec_agent.run(user_profile=None, num_items=request.num_items * 2),
        )

        # Phase 2：LLM重排 + 库存校验 并行执行
        rerank_result, inventory_result = await asyncio.gather(
            self.product_rec_agent.run(user_profile=profile_result, num_items=request.num_items),
            self.inventory_agent.run(products=rec_result.products),
        )

        # 库存过滤
        available_ids = set(inventory_result.available_products)
        final_products = [p for p in rerank_result.products if p.product_id in available_ids]

        # Phase 3：文案生成（串行）
        copy_result = await self.marketing_copy_agent.run(
            user_profile=profile_result,
            products=final_products,
        )

        # 汇总响应
        return RecommendationResponse(
            products=final_products,
            marketing_copies=copy_result.copies,
            experiment_group=experiment.get("group", "control"),
        )
```

### A/B 测试引擎

**文件**：`python/services/ab_test.py`

```python
class ABTestEngine:
    """流量分桶 + Thompson Sampling 多臂赌博机"""

    def assign(self, user_id: str) -> dict:
        # 用户ID哈希取模保证一致性
        bucket = int(hashlib.md5(user_id.encode()).hexdigest(), 16) % 100
        
        if bucket < 60:
            return {"group": "control", "strategy": "collaborative_filter"}
        elif bucket < 80:
            return {"group": "treatment_llm", "strategy": "llm_rerank"}
        else:
            return {"group": "treatment_vector", "strategy": "vector_search"}

    def record_click(self, user_id: str, clicked: bool):
        # Thompson Sampling 更新
        group = self.assign(user_id)["group"]
        if clicked:
            self.alpha[group] += 1
        else:
            self.beta[group] += 1
```

### Agent 基类：重试 + 降级

**文件**：`python/agents/base_agent.py`

```python
class BaseAgent(ABC):
    """所有 Agent 的基类 — 模板方法模式"""
    
    MAX_RETRIES = 3
    RETRY_DELAY = 1.0

    async def run(self, **kwargs) -> AgentResult:
        try:
            return await self._retry_execute(**kwargs)
        except Exception as e:
            logger.warning(f"{self.name} fallback triggered: {e}")
            return self._fallback(**kwargs)

    async def _retry_execute(self, **kwargs) -> AgentResult:
        """指数退避重试"""
        for attempt in range(self.MAX_RETRIES):
            try:
                return await asyncio.wait_for(
                    self._execute(**kwargs),
                    timeout=self.timeout,
                )
            except asyncio.TimeoutError:
                if attempt < self.MAX_RETRIES - 1:
                    await asyncio.sleep(self.RETRY_DELAY * (2 ** attempt))
        raise RuntimeError(f"{self.name} failed after {self.MAX_RETRIES} retries")

    @abstractmethod
    async def _execute(self, **kwargs) -> AgentResult:
        """子类实现业务逻辑"""
```

---

## 🚀 快速上手

### 前置条件

- Python 3.11+
- 申请 LLM API Key（推荐 MiniMax 或阿里通义，有免费额度）

### 启动步骤

```bash
# 1. 克隆项目
git clone https://github.com/bcefghj/multi-agent-ecommerce-system.git
cd multi-agent-ecommerce-system/python

# 2. 创建虚拟环境
python -m venv .venv
source .venv/bin/activate   # Windows: .venv\Scripts\activate

# 3. 安装依赖
pip install -r requirements.txt

# 4. 配置 API Key
cp .env.example .env
# 编辑 .env，填入 LLM_API_KEY

# 5. 启动服务
python main.py
# 访问 http://localhost:8000

# 6. 测试推荐接口
curl -X POST http://localhost:8000/api/v1/recommend \
  -H "Content-Type: application/json" \
  -d '{
    "user_id": "user_001",
    "scene": "homepage",
    "num_items": 5,
    "context": {
      "recent_views": ["手机", "耳机"],
      "avg_order_amount": 500
    }
  }'
```

### Docker 一键部署

```bash
docker-compose up -d
# 服务地址: http://localhost:8000
```

---

## 📡 API 接口

### 接口列表

| 方法 | 路径 | 说明 |
|------|------|------|
| `POST` | `/api/v1/recommend` | 核心推荐接口 |
| `POST` | `/api/v1/recommend/graph` | LangGraph 状态图推荐 |
| `GET` | `/api/v1/experiments` | 查看 A/B 实验状态 |
| `GET` | `/api/v1/metrics` | 系统监控指标 |
| `GET` | `/health` | 健康检查 |

### 请求示例

```json
POST /api/v1/recommend
{
  "user_id": "user_001",
  "scene": "homepage",
  "num_items": 5,
  "context": {
    "recent_views": ["手机", "耳机"],
    "avg_order_amount": 500,
    "last_purchase_days": 7
  }
}
```

### 响应示例

```json
{
  "request_id": "a3f8c2d1-...",
  "user_id": "user_001",
  "products": [
    {
      "product_id": "P001",
      "name": "iPhone 16 Pro",
      "category": "手机",
      "price": 7999.0,
      "score": 0.95
    }
  ],
  "marketing_copies": [
    {
      "product_id": "P001",
      "copy": "根据您最近对手机的兴趣，为您精选 iPhone 16 Pro，好评率 98%。"
    }
  ],
  "experiment_group": "treatment_llm",
  "total_latency_ms": 1523.4
}
```

---

## 📁 项目结构

```
multi-agent-ecommerce-system/
├── README.md                    # 项目总览
├── plan.md                      # 项目计划
├── docker-compose.yml           # 一键部署配置
└── python/
    ├── main.py                  # FastAPI 入口
    ├── requirements.txt         # 依赖列表
    ├── .env.example             # 环境变量模板
    ├── agents/
    │   ├── base_agent.py        # 基类：重试/超时/降级
    │   ├── user_profile_agent.py      # 用户画像 Agent
    │   ├── product_rec_agent.py       # 商品推荐 Agent
    │   ├── marketing_copy_agent.py    # 营销文案 Agent
    │   └── inventory_agent.py         # 库存决策 Agent
    ├── orchestrator/
    │   ├── supervisor.py        # Supervisor 并行编排
    │   └── graph.py             # LangGraph 状态图
    ├── services/
    │   ├── ab_test.py           # A/B 测试引擎
    │   ├── feature_store.py     # Redis 实时特征服务
    │   └── metrics.py           # 监控指标
    ├── models/schemas.py        # Pydantic 数据模型
    ├── config/settings.py       # 配置管理
    └── tests/                   # 单元测试
```

---

## ✨ 技术亮点

1. **Supervisor 模式编排**：集中控制，流程清晰，支持并行执行，降低端到端延迟
2. **实时特征工程**：基于 Redis Sorted Set 的滑动窗口计算，特征更新延迟 < 100ms
3. **A/B 测试引擎**：流量分桶保证一致性，Thompson Sampling 动态优化流量分配
4. **多层可靠性保障**：超时控制、指数退避重试、降级策略，确保系统稳定性
5. **个性化文案生成**：基于用户画像的 Prompt 模板选择，提升营销效果

---

## 🔗 参考资料

| 项目 | 说明 |
|------|------|
| LangGraph | 状态机图架构的 Multi-Agent 框架 |
| LangChain | LLM 应用开发工具链 |
| Redis | 实时特征存储 |
| Milvus | 向量数据库 |
| MiniMax API | LLM 服务 |

---

## 📄 License

[MIT License](LICENSE)

---

<div align="center">

**如果这个项目对你有帮助，欢迎点个 ⭐ Star！**

</div>
