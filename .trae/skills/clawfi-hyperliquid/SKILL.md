---
name: "clawfi-hyperliquid"
description: "专为 ClawFi 用户设计的 Hyperliquid 交易所集成工具。支持账户设置（包括 Vault/Agent）、交易执行和持仓监控，帮助 ClawFi 策略快速接入 Hyperliquid 市场。"
---

# Hyperliquid 交易技能 (Hyperliquid Trader Skill)

此技能提供了一种使用 Python 集成 Hyperliquid 交易所的标准化方法。

## 功能 (Capabilities)

1.  **账户管理**：支持主账户、子账户（Agent）和 Vaults。
2.  **市场数据**：获取用户余额和持仓。
3.  **交易**：执行市价单、限价单。
4.  **订单管理**：取消订单。

## 依赖 (Dependencies)

确保安装了以下 Python 包：
```bash
pip install hyperliquid-python-sdk eth-account
```

## 核心实现模式 (Core Implementation Pattern)

### 1. 初始化 (Initialization)

关键在于正确处理不同的账户类型：
- **主账户 (Main Account)**：直接使用私钥。
- **Agent (子账户)**：使用 Agent 的私钥，但操作的是主账户的地址（通过 `account_address` 参数）。
- **Vault**：使用私钥，但操作的是 Vault 地址（通过 `vault_address` 参数）。

```python
from eth_account import Account
from hyperliquid.info import Info
from hyperliquid.exchange import Exchange
from hyperliquid.utils import constants

def init_hyperliquid(private_key: str, target_address: str = None, is_vault: bool = False):
    """
    初始化 Hyperliquid 连接。
    
    Args:
        private_key (str): 签名钱包（主账户或 Agent）的私钥。
        target_address (str, optional): 
            - 如果这是 Agent 钱包，请在此处传递主账户地址。
            - 如果这是 Vault，请在此处传递 Vault 地址。
            - 如果为 None，则默认为私钥对应的地址。
        is_vault (bool): 如果 target_address 是 Vault 地址，请设为 True。默认为 False (Agent 模式)。
    """
    account = Account.from_key(private_key)
    base_url = constants.MAINNET_API_URL
    
    # 自动检测 target_address 是否与签名者相同
    if target_address and target_address.lower() == account.address.lower():
        target_address = None

    info = Info(base_url, skip_ws=True)
    
    # 根据 is_vault 参数决定初始化模式
    if is_vault:
        # Vault 模式：必须使用 vault_address 参数
        exchange = Exchange(account, base_url, vault_address=target_address)
    else:
        # Agent 模式 (默认)：使用 account_address 参数
        exchange = Exchange(account, base_url, account_address=target_address)
    
    return account, exchange, info
```

### 2. 获取账户状态 (Fetching Account State)

```python
def get_account_state(info: Info, address: str):
    user_state = info.user_state(address)
    
    # 1. 净值与余额
    margin_summary = user_state.get("marginSummary", {})
    account_value = float(margin_summary.get("accountValue", 0))
    withdrawable = float(margin_summary.get("withdrawable", 0))
    
    # 2. 持仓
    positions = []
    for pos in user_state.get("assetPositions", []):
        p = pos.get("position", {})
        positions.append({
            "coin": p.get("coin"),
            "size": float(p.get("szi", 0)),
            "entry_price": float(p.get("entryPx", 0)),
            "pnl": float(p.get("unrealizedPnl", 0)),
            "liquidation_price": p.get("liquidationPx")
        })
        
    return {
        "value": account_value,
        "withdrawable": withdrawable,
        "positions": positions
    }
```

### 3. 执行交易 (Executing Trades)

包含市价单和限价单的实现。

```python
def place_market_order(exchange: Exchange, coin: str, is_buy: bool, size: float):
    """
    下单（市价单）。
    
    Args:
        coin (str): 币种符号，例如 "ETH", "SOL"。
        is_buy (bool): True 为买入/做多，False 为卖出/做空。
        size (float): 交易数量。
    """
    # market_open 处理开仓和加仓
    # 对于市价单，它默认将 "tif" (Time in Force) 设置为 "Ioc" (Immediate or Cancel)
    return exchange.market_open(coin, is_buy, size)

def place_limit_order(exchange: Exchange, coin: str, is_buy: bool, size: float, limit_price: float, tif: str = "Gtc"):
    """
    下单（限价单）。
    
    Args:
        coin (str): 币种符号，例如 "ETH", "SOL"。
        is_buy (bool): True 为买入/做多，False 为卖出/做空。
        size (float): 交易数量。
        limit_price (float): 限价价格。
        tif (str): Time in Force (生效时间策略)。
               - "Gtc": Good Till Cancelled (一直有效直到取消，默认)
               - "Ioc": Immediate or Cancel (立即成交或取消)
               - "Alo": Add Liquidity Only (只做 Maker，不吃单)
    """
    order_type = {"limit": {"tif": tif}}
    return exchange.order(coin, is_buy, size, limit_price, order_type)
```

### 4. 撤单 (Cancelling Orders)

```python
def cancel_all_orders(exchange: Exchange):
    """取消所有未结订单"""
    return exchange.cancel_all_orders()

def cancel_order(exchange: Exchange, coin: str, oid: int):
    """
    根据订单 ID 取消特定订单
    
    Args:
        coin (str): 币种符号
        oid (int): 订单 ID (Order ID)
    """
    return exchange.cancel(coin, oid)
```

## 使用示例 (Usage Example)

```python
# 1. 设置
private_key = "YOUR_PRIVATE_KEY"
agent_target = "0xMainAccountAddress..." # 可选，仅在使用 Agent 钱包时填写

# 2. 初始化 (Agent 模式)
acc, exc, info = init_hyperliquid(private_key, agent_target)

# 2b. 初始化 (Vault 模式)
# vault_addr = "0xVaultAddress..."
# acc, exc, info = init_hyperliquid(private_key, vault_addr, is_vault=True)

# 3. 检查状态
state = get_account_state(info, agent_target if agent_target else acc.address)
print(f"余额: ${state['value']}")

# 4. 交易示例

# A. 市价买入 1 SOL
print("执行市价单...")
market_res = place_market_order(exc, "SOL", True, 1.0)
print(f"市价单结果: {market_res}")

# B. 限价卖出 0.5 SOL，价格 $200 (Gtc)
print("执行限价单...")
limit_res = place_limit_order(exc, "SOL", False, 0.5, 200.0, tif="Gtc")
print(f"限价单结果: {limit_res}")

# C. 取消所有订单
# print("取消所有订单...")
# exc.cancel_all_orders()
```
