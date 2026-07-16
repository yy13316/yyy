# 量化天机决策系统 - 实时行情后端

从纯前端模拟数据升级到**真实实时行情驱动的量化分析系统**。

---

## 架构概览

```
┌──────────────────────────────────────────────────────────┐
│                    浏览器 (前端 React)                      │
│  ┌─────────────┐    ┌─────────────┐    ┌──────────────┐ │
│  │  仪表盘UI    │◄───│ WebSocket   │◄───│  量化分析展示 │ │
│  │             │    │  实时更新    │    │              │ │
│  └─────────────┘    └──────┬──────┘    └──────────────┘ │
└─────────────────────────────┼────────────────────────────┘
                              │
                         ws://localhost:3001
                              │
┌─────────────────────────────┼────────────────────────────┐
│                    后端服务 (Node.js)                      │
│  ┌─────────────┐    ┌─────┴─────┐    ┌────────────────┐ │
│  │  Express    │    │ WebSocket │    │  量化计算引擎   │ │
│  │  HTTP API   │    │ 推送服务   │    │  (MA/MACD/RSI) │ │
│  └──────┬──────┘    └───────────┘    └────────────────┘ │
│         │                                                  │
│  ┌──────┴─────────────────────────────────────────────┐   │
│  │         行情数据获取层 (自动降级)                      │   │
│  │  腾讯财经 ──► 新浪财经 ──► 东方财富(备用)            │   │
│  └────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────┘
```

---

## 快速启动

### 1. 安装依赖

```bash
cd quant-server
npm install
```

需要：Node.js 18+

### 2. 启动服务

```bash
# 开发模式（热重载）
npm run dev

# 生产模式
npm run build
npm start
```

服务启动后：
- HTTP API: http://localhost:3001
- WebSocket: ws://localhost:3001

### 3. 验证

```bash
# 健康检查
curl http://localhost:3001/api/health

# 查询股票（002222 福晶科技）
curl http://localhost:3001/api/stock/002222

# 查询贵州茅台
curl http://localhost:api/stock/600519

# 大盘指数
curl http://localhost:3001/api/index
```

---

## API 接口

### HTTP REST API

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/api/health` | 健康检查 |
| GET | `/api/index` | 大盘指数（上证/深证/创业板/科创50） |
| GET | `/api/stock/:code` | 单只股票完整分析（行情+技术指标+买卖信号） |
| GET | `/api/stock/:code/simple` | 仅实时行情 |
| GET | `/api/stock/:code/kline?period=day&count=60` | K线历史数据 |
| POST | `/api/stock/batch` | 批量查询（最多50只） |

### WebSocket 实时推送

连接 `ws://localhost:3001` 后：

```javascript
// 订阅股票
ws.send(JSON.stringify({
  type: 'subscribe',
  codes: ['002222', '600519']
}));

// 接收实时推送
ws.onmessage = (event) => {
  const msg = JSON.parse(event.data);
  if (msg.type === 'tick') {
    console.log('实时数据:', msg.data);
  }
};
```

推送频率：交易中每 5 秒，非交易时间暂停。

---

## 数据来源与降级策略

系统内置**三级降级**，确保数据可用性：

| 优先级 | 数据源 | 延迟 | 说明 |
|--------|--------|------|------|
| 1 | 腾讯财经 | ~500ms | 首选，最稳定 |
| 2 | 新浪财经 | ~500ms | 腾讯失败时自动切换 |
| 3 | 东方财富 | ~800ms | 新浪也失败时使用 |

K线历史数据始终来自东方财富（数据最全）。

---

## 量化计算引擎

### 技术指标（7项加权）

| 指标 | 权重 | 计算方法 |
|------|------|----------|
| 均线系统 | 20% | MA5/10/20/60 多头排列 |
| 资金流向 | 20% | 买卖盘压力比 |
| MACD | 15% | DIF/DEA/MACD 金叉死叉 |
| 量价关系 | 15% | 放量涨/缩量的判断 |
| KDJ | 10% | J值超买超卖 |
| RSI | 10% | 14日相对强弱 |
| 布林带 | 10% | 上下轨突破 |

### 融合权重（8-Agent模拟）

| 维度 | 权重 | 说明 |
|------|------|------|
| 技术量化 | 50% | 上述7项技术指标综合 |
| 资金+舆情 | 20% | 买卖盘压力（资金10%+舆情10%模拟） |
| 奇门周期 | 15% | 可接入奇门遁甲计算模块 |
| 六爻+干支 | 15% | 可接入六爻计算模块 |

---

## 前端对接

### 方式一：WebSocket（推荐，实时更新）

将 `src/utils/useRealtimeData.ts` 复制到前端项目：

```typescript
import { useRealtimeData } from './utils/useRealtimeData';

function TradingPanel() {
  const { data, connected, lastUpdate } = useRealtimeData(['002222']);
  
  // data[0] 包含：quote（行情）、analysis（分析）、plan（交易计划）
  return (
    <div>
      <p>连接状态: {connected ? '已连接' : '断开'}</p>
      <p>最后更新: {lastUpdate}</p>
      <p>信号: {data[0]?.analysis.overallSignal}</p>
      <p>价格: ¥{data[0]?.quote.price}</p>
    </div>
  );
}
```

### 方式二：HTTP轮询（备用）

```typescript
import { fetchStockAnalysis } from './utils/useRealtimeData';

const data = await fetchStockAnalysis('002222');
```

### 方式三：代理配置（开发环境）

在 `vite.config.ts` 中添加代理：

```typescript
export default {
  server: {
    proxy: {
      '/api': {
        target: 'http://localhost:3001',
        changeOrigin: true,
      },
      '/ws': {
        target: 'ws://localhost:3001',
        ws: true,
      },
    },
  },
};
```

---

## 生产部署

### 使用 PM2

```bash
npm install -g pm2
npm run build
pm2 start dist/index.js --name quant-server
pm2 startup
pm2 save
```

### Docker

```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install --production
COPY dist/ ./dist/
EXPOSE 3001
CMD ["node", "dist/index.js"]
```

```bash
docker build -t quant-tianji-server .
docker run -d -p 3001:3001 --name quant-server quant-tianji-server
```

---

## 与QMT对接（进阶）

如果您有银河证券QMT，可以将QMT作为**数据源**输入到本系统：

```
QMT（本地）──导出JSON──► 后端服务 ──► 前端展示
              
在QMT中定时将行情数据写入：
quant-server/data/qmt_feed.json

后端检测到该文件存在时优先读取，
实现"QMT数据 + 本系统量化引擎"的组合。
```

详见 `docs/QMT_INTEGRATION.md`。

---

## 免责声明

> **本系统仅用于算法实验、准确率验证，不构成投资建议。**
> 
> 行情数据来自免费公开接口，可能存在延迟（通常500ms-2s），不适合高频交易。
> 量化模型的信号基于历史技术指标计算，不保证未来收益。
