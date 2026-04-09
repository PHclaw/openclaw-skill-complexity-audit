# openclaw-skill-complexity-audit

## 简介

代码复杂度评分与圈复杂度分析 Skill。扫描代码库，识别高复杂度函数，给出重构建议。

## 触发场景

- "代码太复杂"、"too complex"
- "圈复杂度"、"cyclomatic complexity"
- "代码审计"、"code audit"
- "重构建议"、"refactor"
- "代码质量"、"maintainability"

## 核心流程

### 第一步：工具选择

| 语言 | 工具 | 命令 |
|------|------|------|
| Python | `radon` | `pip install radon && radon cc -a src/` |
| JavaScript/TS | `eslint --rule 'complexity: error'` | ESLint 内置 |
| Go | `gocyclo` | `go install github.com/fzipp/gocyclo/...@latest` |
| Java | `pmd` | 下载 PMD |
| 多语言 | `sonarqube` | 需要 server |

### 第二步：复杂度评级

| CC（圈复杂度）| 评级 | 含义 | 建议 |
|---------------|------|------|------|
| 1-10 | ✅ 简单 | 低风险，易测试 | 可接受 |
| 11-20 | ⚠️ 中等 | 风险增加 | 建议拆分 |
| 21-50 | 🔶 复杂 | 高风险，难测试 | 必须拆分 |
| > 50 | 🔴 非常复杂 | 极高风险 | 紧急重构 |

**其他指标：**
- **LOC（代码行数）**：函数 > 50 行建议拆分
- **NBC（嵌套复杂度）**：嵌套 > 3 层建议重构
- **Halstead 体积**：程序体积指标

### 第三步：分析 Python 示例

```python
# 高复杂度示例（CC=15）
def process_order(order_data):
    if not order_data:
        return None
    if order_data.get('type') == 'online':
        if order_data.get('payment') == 'card':
            if verify_card(order_data['card']):
                if check_stock(order_data['items']):
                    if not is_fraud(order_data['user']):
                        ship_order(order_data)
                    else:
                        flag_fraud(order_data)
                else:
                    notify_out_of_stock(order_data)
        elif order_data.get('payment') == 'wallet':
            if deduct_wallet(order_data):
                ship_order(order_data)
    elif order_data.get('type') == 'offline':
        if check_store_stock(order_data['store']):
            reserve_order(order_data)
    return order_data
```

**重构后（CC=3）：**
```python
def process_order(order_data):
    if not order_data:
        return None
    if order_data['type'] == 'online':
        return _process_online_order(order_data)
    elif order_data['type'] == 'offline':
        return _process_offline_order(order_data)

def _process_online_order(order_data):
    handlers = {
        'card': _handle_card_payment,
        'wallet': _handle_wallet_payment,
    }
    handler = handlers.get(order_data.get('payment'))
    if not handler:
        raise ValueError(f"Unknown payment: {order_data['payment']}")
    return handler(order_data)
```

### 第四步：输出格式

```
## 复杂度审计报告

### 概览
- 扫描文件数：42
- 高复杂度函数 (>10)：8 个
- 极高复杂度 (>20)：2 个

### TOP 5 高复杂度函数

| 文件 | 函数 | CC | LOC | 建议 |
|------|------|-----|-----|------|
| orders.py | process_order | 15 | 60 | 拆分为多个小函数 |
| auth.py | authenticate | 23 | 85 | 拆分为策略模式 |
| parser.py | parse_config | 8 | 30 | 可接受 |

### 修复建议

#### orders.py::process_order (CC: 15)
**问题**：嵌套过深（5层），分支过多
**策略**：卫语句 + 策略模式
**重构后**：CC 降至 4

```python
def process_order(order_data):
    if not order_data:
        return None
    return (_process_online if order_data['type'] == 'online'
            else _process_offline)(order_data)
```

### 总体建议
1. 优先重构 CC > 20 的函数
2. 函数长度控制在 50 行以内
3. 嵌套层数不超过 3 层
```

## 重构模式速查

| 模式 | 适用场景 | 效果 |
|------|----------|------|
| **提取函数** | 嵌套深、分支多 | CC 降低 3-10 |
| **卫语句** | if 嵌套过深 | CC 降低 2-5 |
| **策略模式** | 多分支处理同类逻辑 | CC 降低 5-15 |
| **命令模式** | 操作步骤多 | CC 降低 5-10 |
| **表驱动** | 大型 switch/if-else | CC 降低 5-20 |

## 注意事项

- 不要为了降低 CC 而过度拆分（简单函数过多也难维护）
- 工具测的是圈复杂度，不是代码行数，两者都要看
- 测试覆盖率要跟上：CC 高的函数必须有完整测试