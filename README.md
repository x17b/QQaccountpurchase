# QQ账户购买与自动化管理工具

## 项目概述

本项目提供了一套完整的QQ账户购买、验证及自动化管理解决方案，包含从账户获取到日常维护的全套工具链。系统采用Python 3.10+开发，支持多平台运行，并提供了丰富的API接口供开发者集成。

## 功能特性

- QQ账户批量采购接口
- 自动化验证系统
- 账户信息加密存储
- 多协议登录支持
- 行为模拟引擎
- 分布式任务调度

## 系统架构

```mermaid
graph TD
    A[供应商API] --> B[采购模块]
    B --> C[验证系统]
    C --> D[账户数据库]
    D --> E[任务调度器]
    E --> F[行为模拟器]
    F --> G[QQ协议接口]
```

## 快速开始

### 环境准备

```bash
# 克隆仓库
git clone https://github.com/yourrepo/qq-account-manager.git
cd qq-account-manager

# 创建虚拟环境
python -m venv venv
source venv/bin/activate  # Linux/Mac
venv\Scripts\activate  # Windows

# 安装依赖
pip install -r requirements.txt
```

### 配置文件

创建`config.yaml`文件：

```yaml
database:
  host: 127.0.0.1
  port: 3306
  user: qqadmin
  password: securepass
  name: qq_accounts

api:
  purchase_endpoint: https://api.qqprovider.com/v3
  api_key: your_purchase_key
  rate_limit: 10  # 每秒请求数

security:
  encryption_key: your_256bit_encryption_key
  proxy_list: 
    - http://proxy1.example.com:8080
    - socks5://proxy2.example.com:1080
```

## 核心模块详解

### 1. 账户采购模块

```python
class QQAccountPurchaser:
    def __init__(self, api_endpoint, api_key):
        self.session = requests.Session()
        self.api_endpoint = api_endpoint
        self.session.headers.update({
            'Authorization': f'Bearer {api_key}',
            'Content-Type': 'application/json'
        })
    
    def get_account_pricing(self, quantity, account_type='standard'):
        """获取账户价格信息"""
        params = {
            'quantity': quantity,
            'type': account_type
        }
        response = self.session.get(
            f'{self.api_endpoint}/pricing',
            params=params
        )
        return self._handle_response(response)
    
    def purchase_accounts(self, quantity, payment_method='crypto'):
        """批量购买QQ账户"""
        data = {
            'quantity': quantity,
            'payment_method': payment_method,
            'callback_url': 'https://yourdomain.com/callback'
        }
        response = self.session.post(
            f'{self.api_endpoint}/purchase',
            json=data
        )
        return self._handle_response(response)
    
    def _handle_response(self, response):
        """统一处理API响应"""
        if response.status_code == 200:
            return response.json()
        elif response.status_code == 429:
            raise RateLimitError("API rate limit exceeded")
        else:
            raise APIError(f"API request failed: {response.text}")
```

### 2. 账户验证系统

```python
class QQAccountValidator:
    def __init__(self, proxy_manager=None):
        self.proxy_manager = proxy_manager or ProxyManager()
        self.captcha_solver = CaptchaSolver()
    
    async def validate_account(self, qq_number, password):
        """异步验证QQ账户有效性"""
        connector = aiohttp.TCPConnector(ssl=False)
        proxy = self.proxy_manager.get_random_proxy()
        
        async with aiohttp.ClientSession(connector=connector) as session:
            # 模拟登录流程
            login_data = {
                'uin': qq_number,
                'pwd': password,
                'verifycode': await self._get_verify_code(session, proxy),
                'remember_uin': '1'
            }
            
            try:
                async with session.post(
                    'https://xui.ptlogin2.qq.com/cgi-bin/xlogin',
                    data=login_data,
                    proxy=proxy['url']
                ) as response:
                    if response.status == 200:
                        text = await response.text()
                        return '登录成功' in text
                    return False
            except Exception as e:
                logging.error(f"Validation failed for {qq_number}: {str(e)}")
                return False
    
    async def _get_verify_code(self, session, proxy):
        """获取验证码"""
        captcha_url = f'https://captcha.qq.com/getimage?uin={qq_number}&aid=549000912'
        async with session.get(captcha_url, proxy=proxy['url']) as resp:
            image_data = await resp.read()
            return self.captcha_solver.solve(image_data)
```

### 3. 账户数据库模型

```python
from sqlalchemy import Column, Integer, String, Text, DateTime
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker
from cryptography.fernet import Fernet

Base = declarative_base()

class QQAccount(Base):
    __tablename__ = 'qq_accounts'
    
    id = Column(Integer, primary_key=True)
    qq_number = Column(String(20), unique=True, nullable=False)
    encrypted_password = Column(Text, nullable=False)
    purchase_date = Column(DateTime, nullable=False)
    last_verified = Column(DateTime)
    status = Column(String(20), default='new')
    account_level = Column(Integer, default=1)
    extra_data = Column(Text)
    
    def __init__(self, qq_number, password, encryption_key):
        self.qq_number = qq_number
        self.encrypted_password = self._encrypt_password(password, encryption_key)
        self.purchase_date = datetime.now()
    
    def _encrypt_password(self, password, key):
        fernet = Fernet(key)
        return fernet.encrypt(password.encode()).decode()
    
    def get_password(self, encryption_key):
        fernet = Fernet(encryption_key)
        return fernet.decrypt(self.encrypted_password.encode()).decode()

class AccountManager:
    def __init__(self, engine):
        self.Session = sessionmaker(bind=engine)
    
    def add_account(self, qq_number, password, encryption_key):
        session = self.Session()
        account = QQAccount(qq_number, password, encryption_key)
        session.add(account)
        session.commit()
        return account
    
    def get_accounts_by_status(self, status, limit=100):
        session = self.Session()
        return session.query(QQAccount).filter_by(status=status).limit(limit).all()
```

## 高级功能实现

### 分布式任务调度

```python
from celery import Celery
from kombu import Queue

app = Celery('qq_tasks',
             broker='redis://localhost:6379/0',
             backend='redis://localhost:6379/1')

app.conf.task_queues = (
    Queue('validation', routing_key='task.validation'),
    Queue('purchase', routing_key='task.purchase'),
    Queue('maintenance', routing_key='task.maintenance'),
)

app.conf.task_routes = {
    'tasks.validate_account': {'queue': 'validation'},
    'tasks.purchase_accounts': {'queue': 'purchase'},
    'tasks.run_maintenance': {'queue': 'maintenance'},
}

@app.task(bind=True, max_retries=3)
def validate_account(self, qq_number, encrypted_password):
    """Celery任务：验证QQ账户"""
    try:
        account_manager = AccountManager(engine)
        key = current_app.config['ENCRYPTION_KEY']
        password = account_manager.get_password(encrypted_password, key)
        
        validator = QQAccountValidator()
        loop = asyncio.get_event_loop()
        is_valid = loop.run_until_complete(
            validator.validate_account(qq_number, password)
        
        return {'qq_number': qq_number, 'valid': is_valid}
    except Exception as exc:
        self.retry(exc=exc)
```

### 行为模拟引擎

```python
class BehaviorSimulator:
    def __init__(self, account_manager):
        self.account_manager = account_manager
        self.actions = {
            'login': self.simulate_login,
            'message': self.send_random_message,
            'space': self.browse_qzone,
            'game': self.play_mini_game
        }
        self.delay_range = (1, 5)  # 行为间隔秒数
    
    def run_random_behavior_sequence(self, qq_account, duration=300):
        """运行随机行为序列"""
        end_time = time.time() + duration
        behaviors = list(self.actions.keys())
        
        while time.time() < end_time:
            behavior = random.choice(behaviors)
            self.actions[behavior](qq_account)
            time.sleep(random.uniform(*self.delay_range))
    
    def simulate_login(self, qq_account):
        """模拟登录行为"""
        protocol = random.choice(['PC', 'Mobile', 'Web'])
        print(f"[{qq_account.qq_number}] 模拟{protocol}端登录")
        # 实际实现会调用QQ协议接口
    
    def send_random_message(self, qq_account):
        contacts = self._get_contacts(qq_account)
        if contacts:
            receiver = random.choice(contacts)
            message = self._generate_random_message()
            print(f"[{qq_account.qq_number}] 发送消息给{receiver}: {message}")
            # 实际实现会调用QQ消息接口
    
    def _get_contacts(self, qq_account):
        """获取联系人列表（模拟）"""
        return [f'user_{random.randint(10000, 99999)}' for _ in range(5)]
    
    def _generate_random_message(self):
        """生成随机消息"""
        messages = [
            "你好啊！",
            "在吗？",
            "最近怎么样？",
            "吃饭了吗？",
            "周末有空吗？"
        ]
        return random.choice(messages)
```

## 部署方案

### Docker 部署

```dockerfile
FROM python:3.10-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

# 加密密钥环境变量
ENV ENCRYPTION_KEY your_encryption_key_here

CMD ["gunicorn", "-w 4", "-b :5000", "app:app"]
```

### Kubernetes 部署配置

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: qq-account-manager
spec:
  replicas: 3
  selector:
    matchLabels:
      app: qq-account
  template:
    metadata:
      labels:
        app: qq-account
    spec:
      containers:
      - name: manager
        image: yourrepo/qq-account-manager:latest
        ports:
        - containerPort: 5000
        env:
        - name: ENCRYPTION_KEY
          valueFrom:
            secretKeyRef:
              name: qq-secrets
              key: encryption-key
        resources:
          limits:
            cpu: "1"
            memory: 1Gi
          requests:
            cpu: "0.5"
            memory: 512Mi
---
apiVersion: v1
kind: Service
metadata:
  name: qq-account-service
spec:
  selector:
    app: qq-account
  ports:
    - protocol: TCP
      port: 80
      targetPort: 5000
```

## 安全注意事项

1. **数据加密**：所有账户密码必须使用AES-256加密存储
2. **访问控制**：实施严格的API访问权限控制
3. **日志审计**：记录所有敏感操作日志
4. **网络隔离**：账户管理服务应部署在独立网络区域
5. **合规使用**：确保遵守QQ用户协议和相关法律法规

## 性能优化建议

```python
# 使用连接池优化数据库访问
from sqlalchemy.pool import QueuePool

engine = create_engine(
    'mysql+pymysql://user:pass@localhost/db',
    poolclass=QueuePool,
    pool_size=10,
    max_overflow=20,
    pool_timeout=30
)

# 异步IO优化网络请求
async def batch_validate_accounts(accounts):
    semaphore = asyncio.Semaphore(10)  # 并发限制
    
    async def validate_with_semaphore(account):
        async with semaphore:
            return await validator.validate_account(account)
    
    tasks = [validate_with_semaphore(acc) for acc in accounts]
    return await asyncio.gather(*tasks, return_exceptions=True)
```

## 开发者指南

### 扩展新功能

1. 创建新的行为模块：

```python
from abc import ABC, abstractmethod

class QQBehavior(ABC):
    @abstractmethod
    def execute(self, qq_account):
        pass

class GroupJoinBehavior(QQBehavior):
    def execute(self, qq_account):
        """实现加群行为逻辑"""
        print(f"{qq_account.qq_number} 正在加入群组")
        # 实际加群实现...
```

2. 注册到行为工厂：

```python
class BehaviorFactory:
    behaviors = {
        'group_join': GroupJoinBehavior,
        # 其他行为...
    }
    
    @classmethod
    def create_behavior(cls, behavior_type):
        if behavior_type not in cls.behaviors:
            raise ValueError(f"未知行为类型: {behavior_type}")
        return cls.behaviors[behavior_type]()
```

### API文档

```markdown
### POST /api/accounts/purchase

购买新的QQ账户

**请求参数**:
```json
{
  "quantity": 10,
  "account_type": "premium",
  "payment_method": "crypto"
}
```

**响应示例**:
```json
{
  "order_id": "ORD-123456",
  "total_price": 150.00,
  "currency": "USDT",
  "estimated_delivery": "2025-06-21T12:00:00Z"
}
```
```

## 贡献指南

欢迎通过GitHub提交Pull Request贡献代码。请确保：

1. 遵循PEP 8代码风格
2. 为新功能添加单元测试
3. 更新相关文档
4. 不包含任何敏感信息

## 许可证

本项目采用MIT许可证。详情请见LICENSE文件。

---

**免责声明**：本项目仅供技术研究使用，开发者不对任何滥用行为负责。使用前请确保遵守相关服务条款和法律法规。
