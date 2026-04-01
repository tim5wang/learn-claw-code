# 08 - 二次开发指南

> **基于 claw-code 进行定制开发** - 环境搭建、扩展开发、最佳实践  
> **更新时间:** 2026-04-01

---

## 🎯 开发场景

### 场景 1: 添加自定义工具

**适用情况:**
- 需要集成内部 API
- 需要特殊文件处理
- 需要自定义工作流

### 场景 2: 开发插件

**适用情况:**
- 需要审计/日志功能
- 需要安全检查
- 需要成本追踪

### 场景 3: 定制 UI/交互

**适用情况:**
- 需要 Web 界面
- 需要 IDE 集成
- 需要自定义终端

---

## 🛠️ 开发环境搭建

### Python 版本环境

```bash
# 1. 克隆仓库
git clone https://github.com/instructkr/claw-code.git
cd claw-code

# 2. 创建虚拟环境
python3 -m venv .venv
source .venv/bin/activate  # Linux/Mac
# .venv\Scripts\activate   # Windows

# 3. 安装依赖
pip install -r requirements.txt

# 4. 安装开发依赖
pip install -r requirements-dev.txt

# 5. 运行测试
python -m pytest tests/ -v

# 6. 运行 CLI
python -m src.main
```

### Rust 版本环境

```bash
# 1. 安装 Rust (如未安装)
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

# 2. 克隆仓库
git clone https://github.com/instructkr/claw-code.git
cd claw-code/rust

# 3. 构建
cargo build --release

# 4. 运行测试
cargo test

# 5. 运行 CLI
./target/release/claw-cli
```

---

## 📦 扩展开发示例

### 示例 1: Python 自定义工具

```python
# src/tools/my_custom_tool.py
from typing import Dict, Any
from .base import Tool

class DatabaseQueryTool(Tool):
    """数据库查询工具"""
    
    name = "database_query"
    description = "执行 SQL 查询"
    
    parameters = {
        "type": "object",
        "properties": {
            "query": {
                "type": "string",
                "description": "SQL 查询语句"
            },
            "database": {
                "type": "string",
                "description": "数据库名称",
                "enum": ["users", "orders", "products"]
            }
        },
        "required": ["query", "database"]
    }
    
    def execute(self, args: Dict[str, Any]) -> Dict[str, Any]:
        query = args["query"]
        database = args["database"]
        
        # 安全检查
        if not self._is_safe_query(query):
            return {"error": "不允许的 SQL 操作"}
        
        # 执行查询
        result = self._run_query(database, query)
        
        return {
            "success": True,
            "data": result,
            "rows_affected": len(result)
        }
    
    def _is_safe_query(self, query: str) -> bool:
        # 禁止 DROP, DELETE, TRUNCATE 等操作
        forbidden = ["DROP", "DELETE", "TRUNCATE", "ALTER"]
        return not any(cmd in query.upper() for cmd in forbidden)
    
    def _run_query(self, database: str, query: str) -> list:
        # 实际数据库连接逻辑
        import sqlite3
        conn = sqlite3.connect(f"/data/{database}.db")
        cursor = conn.cursor()
        cursor.execute(query)
        return cursor.fetchall()

# 注册工具
from .tool_pool import register_tool
register_tool(DatabaseQueryTool())
```

### 示例 2: Python 自定义插件

```python
# src/plugins/audit_logger.py
from typing import Any
from .base import Plugin
from datetime import datetime
import json

class AuditLoggerPlugin(Plugin):
    """审计日志插件"""
    
    name = "audit_logger"
    version = "1.0.0"
    
    def __init__(self, log_file: str = "/var/log/claw-audit.log"):
        self.log_file = log_file
    
    def on_session_start(self, session_id: str) -> None:
        self._log({
            "event": "session_start",
            "session_id": session_id,
            "timestamp": datetime.utcnow().isoformat()
        })
    
    def on_message(self, message: dict) -> None:
        self._log({
            "event": "message",
            "role": message["role"],
            "length": len(message.get("content", "")),
            "timestamp": datetime.utcnow().isoformat()
        })
    
    def on_tool_call(self, tool_name: str, args: dict) -> None:
        self._log({
            "event": "tool_call",
            "tool": tool_name,
            "args_keys": list(args.keys()),
            "timestamp": datetime.utcnow().isoformat()
        })
    
    def on_response(self, response: dict) -> None:
        self._log({
            "event": "response",
            "has_tool_calls": "tool_calls" in response,
            "timestamp": datetime.utcnow().isoformat()
        })
    
    def _log(self, entry: dict) -> None:
        with open(self.log_file, "a") as f:
            f.write(json.dumps(entry) + "\n")

# 注册插件
from .plugin_registry import register_plugin
register_plugin(AuditLoggerPlugin())
```

### 示例 3: Rust 自定义工具

```rust
// crates/tools/src/my_tool.rs
use crate::{Tool, Schema, ToolResult};
use serde_json::Value;

pub struct DatabaseQueryTool;

impl Tool for DatabaseQueryTool {
    fn name(&self) -> &str {
        "database_query"
    }
    
    fn description(&self) -> &str {
        "执行 SQL 查询"
    }
    
    fn parameters(&self) -> Schema {
        Schema::Object {
            properties: vec![
                ("query".to_string(), Schema::String),
                ("database".to_string(), Schema::String),
            ],
            required: vec!["query".to_string(), "database".to_string()],
        }
    }
    
    fn execute(&self, args: Value) -> ToolResult<Value> {
        let query = args["query"].as_str().ok_or("Invalid query")?;
        let database = args["database"].as_str().ok_or("Invalid database")?;
        
        // 安全检查
        if !self.is_safe_query(query) {
            return Err("不允许的 SQL 操作".into());
        }
        
        // 执行查询
        let result = self.run_query(database, query)?;
        
        Ok(json!({
            "success": true,
            "data": result,
            "rows_affected": result.as_array().map(|a| a.len()).unwrap_or(0)
        }))
    }
}

impl DatabaseQueryTool {
    fn is_safe_query(&self, query: &str) -> bool {
        let forbidden = ["DROP", "DELETE", "TRUNCATE", "ALTER"];
        !forbidden.iter().any(|cmd| query.to_uppercase().contains(cmd))
    }
    
    fn run_query(&self, database: &str, query: &str) -> Result<Value, Box<dyn std::error::Error>> {
        // 实际数据库查询逻辑
        Ok(json!([]))
    }
}
```

### 示例 4: Rust 自定义插件

```rust
// crates/plugins/src/audit_logger.rs
use crate::{Plugin, PluginResult};
use claw_runtime::Message;
use serde_json::json;
use std::fs::OpenOptions;
use std::io::Write;

pub struct AuditLoggerPlugin {
    log_file: String,
}

impl AuditLoggerPlugin {
    pub fn new(log_file: String) -> Self {
        Self { log_file }
    }
}

impl Plugin for AuditLoggerPlugin {
    fn name(&self) -> &str {
        "audit_logger"
    }
    
    fn version(&self) -> &str {
        "1.0.0"
    }
    
    fn on_message(&self, msg: &Message) -> PluginResult<()> {
        self.log(json!({
            "event": "message",
            "role": msg.role,
            "length": msg.content.len(),
        }))?;
        Ok(())
    }
    
    fn on_tool_call(&self, tool: &str, args: &serde_json::Value) -> PluginResult<()> {
        self.log(json!({
            "event": "tool_call",
            "tool": tool,
        }))?;
        Ok(())
    }
}

impl AuditLoggerPlugin {
    fn log(&self, entry: serde_json::Value) -> std::io::Result<()> {
        let mut file = OpenOptions::new()
            .create(true)
            .append(true)
            .open(&self.log_file)?;
        writeln!(file, "{}", entry)?;
        Ok(())
    }
}
```

---

## 🔌 集成现有系统

### 集成内部 API

```python
# src/tools/internal_api.py
import requests
from .base import Tool

class InternalAPITool(Tool):
    """内部 API 调用工具"""
    
    name = "internal_api"
    description = "调用公司内部 API"
    
    def __init__(self, base_url: str, api_key: str):
        self.base_url = base_url
        self.session = requests.Session()
        self.session.headers.update({
            "Authorization": f"Bearer {api_key}",
            "Content-Type": "application/json"
        })
    
    def execute(self, args: dict) -> dict:
        endpoint = args["endpoint"]
        method = args.get("method", "GET")
        data = args.get("data")
        
        url = f"{self.base_url}/{endpoint}"
        
        response = self.session.request(
            method=method,
            url=url,
            json=data if method in ["POST", "PUT", "PATCH"] else None
        )
        
        return {
            "status_code": response.status_code,
            "data": response.json(),
            "headers": dict(response.headers)
        }
```

### 集成数据库

```python
# src/tools/database.py
from sqlalchemy import create_engine, text
from .base import Tool

class DatabaseTool(Tool):
    """数据库操作工具"""
    
    name = "database"
    description = "数据库读写操作"
    
    def __init__(self, connection_string: str):
        self.engine = create_engine(connection_string)
    
    def execute(self, args: dict) -> dict:
        operation = args["operation"]
        
        with self.engine.connect() as conn:
            if operation == "query":
                result = conn.execute(text(args["sql"]))
                return {"rows": [dict(row) for row in result]}
            
            elif operation == "insert":
                result = conn.execute(text(args["sql"]), args["params"])
                conn.commit()
                return {"rows_affected": result.rowcount}
            
            elif operation == "update":
                result = conn.execute(text(args["sql"]), args["params"])
                conn.commit()
                return {"rows_affected": result.rowcount}
```

### 集成消息队列

```python
# src/tools/message_queue.py
import pika
from .base import Tool

class MessageQueueTool(Tool):
    """消息队列工具"""
    
    name = "message_queue"
    description = "发送/接收消息队列消息"
    
    def __init__(self, rabbitmq_url: str):
        self.connection = pika.BlockingConnection(
            pika.URLParameters(rabbitmq_url)
        )
        self.channel = self.connection.channel()
    
    def execute(self, args: dict) -> dict:
        action = args["action"]
        queue = args["queue"]
        message = args.get("message")
        
        if action == "publish":
            self.channel.basic_publish(
                exchange='',
                routing_key=queue,
                body=message
            )
            return {"status": "published"}
        
        elif action == "consume":
            method_frame, header_frame, body = self.channel.basic_get(
                queue=queue,
                auto_ack=True
            )
            if method_frame:
                return {
                    "message": body.decode(),
                    "delivery_tag": method_frame.delivery_tag
                }
            return {"message": None}
```

---

## 🧪 测试指南

### Python 测试

```python
# tests/test_my_tool.py
import unittest
from src.tools.my_custom_tool import DatabaseQueryTool

class TestDatabaseQueryTool(unittest.TestCase):
    
    def setUp(self):
        self.tool = DatabaseQueryTool()
    
    def test_safe_query(self):
        """测试安全查询"""
        result = self.tool.execute({
            "query": "SELECT * FROM users",
            "database": "users"
        })
        self.assertTrue(result["success"])
    
    def test_unsafe_query(self):
        """测试不安全查询被拒绝"""
        result = self.tool.execute({
            "query": "DROP TABLE users",
            "database": "users"
        })
        self.assertIn("error", result)
    
    def test_parameter_validation(self):
        """测试参数验证"""
        with self.assertRaises(KeyError):
            self.tool.execute({
                "query": "SELECT * FROM users"
                # 缺少 database 参数
            })

if __name__ == "__main__":
    unittest.main()
```

### Rust 测试

```rust
// crates/tools/tests/test_my_tool.rs
use claw_tools::my_tool::DatabaseQueryTool;
use claw_tools::Tool;
use serde_json::json;

#[test]
fn test_safe_query() {
    let tool = DatabaseQueryTool;
    let result = tool.execute(json!({
        "query": "SELECT * FROM users",
        "database": "users"
    }));
    assert!(result.is_ok());
}

#[test]
fn test_unsafe_query() {
    let tool = DatabaseQueryTool;
    let result = tool.execute(json!({
        "query": "DROP TABLE users",
        "database": "users"
    }));
    assert!(result.is_err());
}

#[test]
fn test_parameter_validation() {
    let tool = DatabaseQueryTool;
    let result = tool.execute(json!({
        "query": "SELECT * FROM users"
        // 缺少 database 参数
    }));
    assert!(result.is_err());
}
```

---

## 📚 最佳实践

### 1. 安全优先

```python
# ✅ 好的做法
def execute(self, args):
    # 1. 参数验证
    self.validate_params(args)
    
    # 2. 权限检查
    if not self.has_permission(args):
        return {"error": "Permission denied"}
    
    # 3. 输入清理
    safe_input = self.sanitize(args["input"])
    
    # 4. 执行操作
    result = self._do_work(safe_input)
    
    # 5. 审计日志
    self.log_action(args, result)
    
    return result
```

### 2. 错误处理

```python
# ✅ 好的做法
def execute(self, args):
    try:
        result = self._do_work(args)
        return {"success": True, "data": result}
    except PermissionError as e:
        return {"success": False, "error": f"权限错误：{e}"}
    except ValidationError as e:
        return {"success": False, "error": f"验证错误：{e}"}
    except Exception as e:
        # 记录详细日志
        self.logger.error(f"工具执行失败：{e}", exc_info=True)
        return {"success": False, "error": "内部错误，请联系管理员"}
```

### 3. 性能优化

```python
# ✅ 好的做法
class CachedTool(Tool):
    def __init__(self):
        self.cache = {}
        self.cache_ttl = 300  # 5 分钟
    
    def execute(self, args):
        cache_key = self._make_cache_key(args)
        
        # 检查缓存
        if cache_key in self.cache:
            cached_time, cached_result = self.cache[cache_key]
            if time.time() - cached_time < self.cache_ttl:
                return cached_result
        
        # 执行实际操作
        result = self._do_work(args)
        
        # 更新缓存
        self.cache[cache_key] = (time.time(), result)
        
        return result
```

### 4. 配置管理

```python
# ✅ 好的做法
# config.yaml
tools:
  database:
    connection_string: "postgresql://user:pass@localhost/db"
    pool_size: 10
  internal_api:
    base_url: "https://api.internal.com"
    api_key: "${API_KEY}"  # 从环境变量读取

# 加载配置
from pydantic_settings import BaseSettings

class ToolConfig(BaseSettings):
    database_url: str
    api_key: str
    
    class Config:
        env_file = ".env"
```

---

## 🚀 部署指南

### Docker 部署

```dockerfile
# Dockerfile
FROM python:3.11-slim

WORKDIR /app

# 安装依赖
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# 复制代码
COPY src/ ./src/
COPY config/ ./config/

# 设置环境变量
ENV PYTHONPATH=/app
ENV CLAW_CONFIG_PATH=/app/config

# 运行
CMD ["python", "-m", "src.main"]
```

```yaml
# docker-compose.yml
version: '3.8'

services:
  claw-code:
    build: .
    volumes:
      - ./data:/app/data
      - ./logs:/app/logs
    environment:
      - OPENAI_API_KEY=${OPENAI_API_KEY}
      - DATABASE_URL=postgresql://user:pass@db:5432/claw
    depends_on:
      - db
  
  db:
    image: postgres:15
    environment:
      - POSTGRES_PASSWORD=pass
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  postgres_data:
```

### Kubernetes 部署

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: claw-code
spec:
  replicas: 3
  selector:
    matchLabels:
      app: claw-code
  template:
    metadata:
      labels:
        app: claw-code
    spec:
      containers:
      - name: claw-code
        image: your-registry/claw-code:latest
        env:
        - name: OPENAI_API_KEY
          valueFrom:
            secretKeyRef:
              name: claw-secrets
              key: openai-api-key
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
```

---

## 📖 参考资源

| 资源 | 链接 |
|------|------|
| GitHub 仓库 | https://github.com/instructkr/claw-code |
| Python 文档 | https://docs.python.org/3/ |
| Rust 文档 | https://doc.rust-lang.org/ |
| oh-my-codex | https://github.com/Yeachan-Heo/oh-my-codex |
| Discord 社区 | https://instruct.kr/ |

---

[← 返回上一章](/claw-code/07-rust-rewrite.md) | [回到首页](/)
