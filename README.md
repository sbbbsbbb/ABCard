# ChatGPT Business / Plus 自动开通

全自动注册 ChatGPT 账号 + 开通 Business 或 Plus 套餐（首月免费），支持 Web UI 操作。

## 功能

- **自动注册** — 临时邮箱创建、OTP 验证、账号注册一条龙
- **首月免费** — Business (`team-1-month-free`) 或 Plus (`plus-1-month-free`)
- **自动支付** — Xvfb + Chrome 自动填写 Stripe 表单、绕过 hCaptcha
- **Web UI** — 粘贴卡片信息即可操作，支持选择已有账号或手动输入 Token
- **计划选择** — 支持 Business (团队版 5席位 $0) 和 Plus (个人版 $0)

## 配置

复制配置模板并填写你的凭证:

```bash
cp config.example.json config.json
```

编辑 `config.json` 填写:
- `mail.worker_domain` / `mail.admin_token` / `mail.email_domain` — 临时邮箱服务
- `card` — 信用卡信息
- `billing` — 账单地址
- `captcha.client_key` — YesCaptcha API Key (可选, API 模式才需要)
- `proxy` — 代理地址

或通过环境变量设置:
```bash
export YESCAPTCHA_KEY="your-key"  # 可选
```

## 快速开始

### 1. 安装依赖

```bash
pip install -r requirements.txt
sudo apt-get install -y xvfb  # 虚拟显示
```

### 2. 安装浏览器

```bash
playwright install chromium
```

### 3. 启动 Web UI

```bash
# 启动 Xvfb 虚拟显示
Xvfb :99 -screen 0 1920x1080x24 -ac &
export DISPLAY=:99

# 启动 UI
streamlit run ui.py --server.address 0.0.0.0 --server.port 8503
```

浏览器访问 `http://localhost:8503`。

### 4. 开发者模式

显示日志、Stripe 响应、高级配置：

```bash
streamlit run ui.py --server.port 8503 -- --dev
```

## Web UI 使用

1. **选择计划**: Business (团队版) 或 Plus (个人版)
2. **选择账号来源**：新注册 / 选择已有账号 / 手动输入 Token
3. **粘贴卡片信息**：支持键值对和纯文本格式自动识别
4. **填写账单地址**：国家/姓名/地址/城市/州/邮编
5. **点击执行**：进度条显示当前步骤，完成后显示结果

## 代码调用

```python
import os, subprocess
from browser_payment import BrowserPayment
from auth_flow import AuthFlow
from mail_provider import MailProvider
from config import Config

# 启动 Xvfb
subprocess.Popen(["Xvfb", ":99", "-screen", "0", "1920x1080x24", "-ac"])
os.environ["DISPLAY"] = ":99"

# 注册账号
cfg = Config()
cfg.proxy = "http://proxy:port"
af = AuthFlow(config=cfg)
mp = MailProvider(cfg.mail.worker_domain, cfg.mail.admin_token, cfg.mail.email_domain)
auth = af.run_register(mp)

# 运行支付
bp = BrowserPayment(proxy=cfg.proxy, headless=False, slow_mo=80)
result = bp.run_full_flow(
    session_token=auth.session_token,
    access_token=auth.access_token,
    device_id=auth.device_id,
    card_number="4242424242424242",
    card_exp_month="03",
    card_exp_year="32",
    card_cvc="123",
    billing_name="John Doe",
    billing_country="US",
    billing_zip="63640",
    billing_line1="123 Main St",
    billing_city="Springfield",
    billing_state="MO",
    billing_currency="USD",
    workspace_name="MyWorkspace",  # Business 模式下用
    chatgpt_proxy=cfg.proxy,
    plan_type="plus",  # "team" 或 "plus"
)
print(f"Success: {result['success']}")
```

## 项目结构

```
auto_bindcard/
├── browser_payment.py     # 核心: 浏览器支付 (Checkout + Stripe 填表 + hCaptcha)
├── auth_flow.py           # API 注册 (10 步)
├── mail_provider.py       # 临时邮箱 + OTP
├── config.py              # 配置管理
├── ui.py                  # Web UI (Streamlit)
├── database.py            # SQLite 数据库层
├── code_manager.py        # 兑换码管理 (生成/验证/计次)
├── admin_cli.py           # 兑换码管理 CLI
├── payment_flow.py        # API 模式支付 (备用)
├── http_client.py         # HTTP 客户端 (curl_cffi)
├── stripe_fingerprint.py  # Stripe 设备指纹
├── browser_challenge.py   # hCaptcha 绕过策略
├── captcha_solver.py      # YesCaptcha 打码服务
├── logger.py              # 日志 & 结果持久化
├── main.py                # CLI 入口
├── test_all.py            # 单元测试
├── config.example.json    # 配置模板
├── requirements.txt
└── README.md
```

## 兑换码管理

使用前需要先生成兑换码:

```bash
# 生成 10 个一次性兑换码
python3 admin_cli.py generate 10

# 生成 5 个可用 3 次的兑换码
python3 admin_cli.py generate 5 --uses 3

# 生成带过期时间的兑换码 (30天)
python3 admin_cli.py generate 1 --uses 99 --expires 30 --note "VIP"

# 查看所有兑换码
python3 admin_cli.py list

# 查看单个兑换码详情
python3 admin_cli.py info XXXX-XXXX-XXXX

# 查看执行历史
python3 admin_cli.py history XXXX-XXXX-XXXX
```

用户打开 Web UI 后需要输入兑换码才能使用。**失败不扣次数**，仅成功时消耗额度。
```

## 环境要求

| 组件 | 版本/说明 |
|------|----------|
| Python | 3.10+ |
| Chrome | Playwright 内置 (chromium) |
| Xvfb | 虚拟显示 (`--headless=new` 不可用) |
| 代理 | 必须为美国 IP |

## 已知限制

1. **不支持 `--headless=new`** — HeadlessChrome UA 被检测，Stripe 不加载。必须用 Xvfb。
2. **3DS 验证** — 如果卡触发 3D Secure，无法自动完成。
3. **卡片被拒** — 如果 Stripe 返回 "支付被拒"，是卡本身的问题，不是代码问题。
