# Celebrity Trade Bot - 系统设计文档

## 文档信息
- 版本：1.0
- 最后更新：2024-09-16
- 状态：草稿

## 1. 架构概述

### 1.1 系统架构图

```
+---------------------------+
|      配置管理模块          |
+---------------------------+
           |
           v
+---------------------------+     +---------------------------+
|                           |     |                           |
|    社交媒体监控模块        | <-> |      数据存储模块         |
|                           |     |                           |
+---------------------------+     +---------------------------+
           |                                 ^
           v                                 |
+---------------------------+                |
|                           |                |
|       AI 分析模块         | --------------+
|                           |                |
+---------------------------+                |
           |                                 |
           v                                 v
+---------------------------+     +---------------------------+
|                           |     |                           |
|       通知模块            | <-> |      市场数据模块         |
|                           |     |                           |
+---------------------------+     +---------------------------+
```

### 1.2 设计理念

Celebrity Trade Bot 采用模块化设计，将系统划分为多个相对独立但紧密协作的组件。这种设计有以下优点：

1. **灵活性**：各模块可以独立开发、测试和部署
2. **可维护性**：每个模块有明确的职责边界
3. **可扩展性**：可以轻松添加新功能或替换现有模块
4. **容错性**：单个模块的故障不会导致整个系统崩溃

系统采用事件驱动架构，以社交媒体活动作为主要触发事件，驱动后续的数据收集、分析和通知流程。

### 1.3 技术选择

#### 编程语言和框架：
- **主要语言**：Python 3.8+
  - 理由：丰富的库生态系统，特别是在数据处理、API 集成和 AI 方面
  - 良好的跨平台兼容性
  - 开发效率高

#### 数据存储：
- **主要存储**：SQLite
  - 理由：轻量级，无需额外服务器
  - 支持复杂查询
  - 单文件存储，便于备份和迁移
- **配置存储**：YAML 文件
  - 理由：人类可读性好
  - 支持注释
  - 结构化数据表示简洁

#### 依赖库：
- **API 请求**：requests
- **数据处理**：pandas
- **数据库 ORM**：SQLAlchemy
- **配置管理**：PyYAML
- **异步处理**：asyncio
- **日志管理**：Python logging 模块
- **Telegram API**：python-telegram-bot

## 2. 核心组件设计

### 2.1 配置管理模块

#### 2.1.1 职责
- 加载和解析配置文件
- 提供全局配置访问接口
- 监控配置文件变化并重新加载
- 验证配置有效性

#### 2.1.2 主要类和接口
- **ConfigManager**：配置管理主类
  - `load_config()`: 加载配置文件
  - `get_config(section, key)`: 获取配置值
  - `validate_config()`: 验证配置完整性
  - `watch_config_changes()`: 监控配置文件变更

#### 2.1.3 设计细节
- 采用单例模式实现
- 支持配置热重载
- 使用环境变量覆盖敏感配置
- 配置验证机制确保必要设置存在

### 2.2 社交媒体监控模块

#### 2.2.1 职责
- 连接社交媒体 API
- 定期查询监控账号的新活动
- 检测新的社交媒体更新
- 格式化和标准化社交媒体数据
- 触发分析流程

#### 2.2.2 主要类和接口
- **SocialMediaMonitor**：抽象基类
  - `check_new_activities()`: 检查新活动
  - `process_activity(activity)`: 处理单个活动
  - `format_activity_data(raw_data)`: 格式化数据

- **TwitterMonitor**：Twitter 平台监控实现
  - 继承 SocialMediaMonitor
  - 实现 Twitter API 特定连接和查询

- **ActivityProcessor**：活动处理器
  - `is_new_activity(activity_id)`: 检查活动是否已处理
  - `mark_as_processed(activity_id)`: 标记活动为已处理
  - `trigger_analysis(activity_data)`: 触发分析流程

#### 2.2.3 设计细节
- 采用策略模式支持多种社交媒体平台
- 实现重试机制处理临时 API 失败
- 使用轮询方式定期检查新活动
- 维护已处理活动ID集合避免重复处理

### 2.3 市场数据模块

#### 2.3.1 职责
- 连接金融数据 API
- 定期获取监控市场的最新数据
- 存储历史市场数据
- 提供市场数据查询接口

#### 2.3.2 主要类和接口
- **MarketDataManager**：市场数据管理主类
  - `update_market_data()`: 更新所有市场数据
  - `get_current_data(market_id)`: 获取最新数据
  - `get_historical_data(market_id, start_time, end_time)`: 获取历史数据
  - `format_market_summary()`: 生成市场摘要

- **MarketDataProvider**：抽象数据提供者接口
  - `fetch_current_price(symbol)`: 获取当前价格
  - `fetch_historical_data(symbol, timeframe)`: 获取历史数据

- **FinancialAPIProvider**：具体数据提供者实现
  - 继承 MarketDataProvider
  - 实现特定 API 的连接和数据获取

#### 2.3.3 设计细节
- 采用适配器模式支持不同的金融数据 API
- 实现本地缓存减少 API 调用
- 使用定时任务定期更新数据
- 支持数据规范化处理不同来源的数据格式

### 2.4 AI 分析模块

#### 2.4.1 职责
- 准备分析所需的数据
- 构建 AI 分析请求
- 调用 Gemini API
- 解析 AI 返回结果
- 生成结构化的分析结果

#### 2.4.2 主要类和接口
- **AIAnalyzer**：AI 分析主类
  - `prepare_analysis_data(activity_id)`: 准备分析数据
  - `build_prompt(activity_data, historical_data, market_data)`: 构建提示
  - `analyze(prompt)`: 调用 AI API 进行分析
  - `parse_response(ai_response)`: 解析 AI 返回结果

- **PromptBuilder**：提示构建器
  - `build_base_prompt()`: 构建基础提示模板
  - `add_activity_context(prompt, activity)`: 添加活动上下文
  - `add_historical_context(prompt, history)`: 添加历史上下文
  - `add_market_context(prompt, market_data)`: 添加市场上下文

- **ResponseParser**：响应解析器
  - `extract_recommendation(response)`: 提取交易建议
  - `extract_confidence(response)`: 提取置信度
  - `extract_reasoning(response)`: 提取分析理由
  - `validate_response(parsed_data)`: 验证解析结果

#### 2.4.3 设计细节
- 采用模板方法模式处理提示构建和结果解析
- 实现结构化输出格式，便于解析
- 支持模型响应验证和重试
- 使用异步处理避免 AI 请求阻塞主流程

### 2.5 通知模块

#### 2.5.1 职责
- 格式化通知消息
- 发送 Telegram 通知
- 处理通知失败和重试
- 记录通知历史

#### 2.5.2 主要类和接口
- **NotificationManager**：通知管理主类
  - `format_notification(analysis_result)`: 格式化通知内容
  - `send_notification(message)`: 发送通知
  - `handle_failed_notification(message, error)`: 处理失败通知
  - `record_notification(message, status)`: 记录通知历史

- **TelegramNotifier**：Telegram 通知实现
  - `initialize_bot(token, chat_id)`: 初始化 Telegram Bot
  - `send_message(chat_id, message)`: 发送消息
  - `format_rich_text(text)`: 格式化富文本

#### 2.5.3 设计细节
- 采用装饰器模式支持不同的通知格式
- 实现消息队列处理并发通知
- 支持通知失败的重试策略
- 提供通知模板自定义功能

### 2.6 数据存储模块

#### 2.6.1 职责
- 管理数据库连接
- 提供数据访问接口
- 实现数据模型映射
- 处理数据迁移和备份

#### 2.6.2 主要类和接口
- **DatabaseManager**：数据库管理主类
  - `initialize_db()`: 初始化数据库
  - `get_session()`: 获取数据库会话
  - `create_backup()`: 创建数据库备份
  - `clean_old_data(days)`: 清理过期数据

- **数据模型类**：
  - `Account`: 监控账号模型
  - `Market`: 市场产品模型
  - `SocialActivity`: 社交媒体活动模型
  - `MarketData`: 市场数据模型
  - `AnalysisResult`: 分析结果模型
  - `Notification`: 通知记录模型

#### 2.6.3 设计细节
- 使用 ORM 框架（SQLAlchemy）映射数据模型
- 实现数据库迁移管理
- 支持定期数据清理和备份
- 采用连接池优化数据库访问性能

## 3. 数据设计

### 3.1 数据库模式

#### 3.1.1 Account 表
```sql
CREATE TABLE accounts (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    platform_type TEXT NOT NULL,
    username TEXT NOT NULL,
    display_name TEXT,
    description TEXT,
    status TEXT DEFAULT 'active',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(platform_type, username)
);
```

#### 3.1.2 Market 表
```sql
CREATE TABLE markets (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    market_type TEXT NOT NULL,
    ticker TEXT NOT NULL,
    name TEXT,
    description TEXT,
    status TEXT DEFAULT 'active',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(market_type, ticker)
);
```

#### 3.1.3 SocialActivity 表
```sql
CREATE TABLE social_activities (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    account_id INTEGER NOT NULL,
    platform_type TEXT NOT NULL,
    activity_id TEXT NOT NULL,
    activity_type TEXT NOT NULL,
    published_at TIMESTAMP NOT NULL,
    content TEXT,
    media_urls TEXT,
    engagement_data TEXT,
    raw_data TEXT,
    processed BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (account_id) REFERENCES accounts(id),
    UNIQUE(platform_type, activity_id)
);
```

#### 3.1.4 MarketData 表
```sql
CREATE TABLE market_data (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    market_id INTEGER NOT NULL,
    timestamp TIMESTAMP NOT NULL,
    price REAL NOT NULL,
    price_change REAL,
    price_change_percent REAL,
    volume REAL,
    high_24h REAL,
    low_24h REAL,
    additional_data TEXT,
    raw_data TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (market_id) REFERENCES markets(id),
    UNIQUE(market_id, timestamp)
);
```

#### 3.1.5 AnalysisResult 表
```sql
CREATE TABLE analysis_results (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    activity_id INTEGER NOT NULL,
    analyzed_at TIMESTAMP NOT NULL,
    recommendation_type TEXT NOT NULL,
    target_market_id INTEGER NOT NULL,
    confidence_score REAL NOT NULL,
    reasoning TEXT,
    supporting_info TEXT,
    raw_analysis TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (activity_id) REFERENCES social_activities(id),
    FOREIGN KEY (target_market_id) REFERENCES markets(id)
);
```

#### 3.1.6 Notification 表
```sql
CREATE TABLE notifications (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    analysis_result_id INTEGER NOT NULL,
    sent_at TIMESTAMP,
    status TEXT NOT NULL,
    content TEXT,
    error_message TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (analysis_result_id) REFERENCES analysis_results(id)
);
```

### 3.2 数据流图

```
+-----------+      +----------------+      +-----------------+
| 配置文件   | ---> | ConfigManager  | ---> | 账号和市场配置   |
+-----------+      +----------------+      +-----------------+
                          |
                          v
+-------------+      +----------------+      +----------------+
| Twitter API | ---> | TwitterMonitor | ---> | 社交媒体活动数据 |
+-------------+      +----------------+      +----------------+
                                                    |
+-------------+      +----------------+             |
| 金融数据 API | ---> | MarketManager  | --------+   |
+-------------+      +----------------+         |   |
                                               |   |
                                               v   v
                                          +----------------+
                                          |  AIAnalyzer    |
                                          +----------------+
                                                  |
                                                  v
+-------------+      +----------------+      +----------------+
| Gemini API  | <--> |  PromptBuilder | <--- | 分析数据准备    |
+-------------+      +----------------+      +----------------+
       |                                            ^
       v                                            |
+----------------+      +----------------+          |
| ResponseParser | ---> | 分析结果数据   | ---------+
+----------------+      +----------------+
                              |
                              v
+----------------+      +----------------+
| TelegramBot API| <--- | NotifyManager  |
+----------------+      +----------------+
```

## 4. 接口设计

### 4.1 外部接口

#### 4.1.1 Twitter API v2
- **认证方式**：OAuth 2.0
- **主要端点**：
  - `GET /2/users/by/username/:username` - 根据用户名获取用户 ID
  - `GET /2/users/:id/tweets` - 获取用户最新推文
  - 请求参数：
    - `max_results`: 每次请求返回结果数量，最大 100
    - `start_time`: ISO 8601 格式的开始时间
    - `tweet.fields`: 指定返回的推文字段，如 created_at,text,public_metrics
  - 响应格式：JSON
  - 错误处理：HTTP 状态码 + 错误详情 JSON

#### 4.1.2 金融数据 API
- **认证方式**：API Key
- **主要功能**：
  - 获取当前价格
  - 获取历史价格数据
  - 获取市场摘要信息
- **请求格式**：将根据用户提供的具体 API 进行适配
- **响应格式**：假设 JSON
- **错误处理**：将根据具体 API 规范进行适配

#### 4.1.3 Gemini API
- **认证方式**：API Key
- **请求格式**：
  ```json
  {
    "contents": [
      {
        "parts": [
          {"text": "你的提示内容..."}
        ]
      }
    ],
    "generationConfig": {
      "temperature": 0.4,
      "topK": 32,
      "topP": 1,
      "maxOutputTokens": 2048
    }
  }
  ```
- **响应格式**：
  ```json
  {
    "candidates": [
      {
        "content": {
          "parts": [
            {"text": "AI 生成的响应内容..."}
          ]
        },
        "finishReason": "STOP",
        "index": 0
      }
    ]
  }
  ```

#### 4.1.4 Telegram Bot API
- **认证方式**：Bot Token
- **主要端点**：
  - `POST /bot{token}/sendMessage` - 发送消息
- **请求格式**：
  ```json
  {
    "chat_id": "用户或群组ID",
    "text": "消息内容",
    "parse_mode": "MarkdownV2"
  }
  ```
- **响应格式**：JSON，包含发送状态和消息信息
- **错误处理**：HTTP 状态码 + 错误详情 JSON

### 4.2 内部接口

#### 4.2.1 模块间通信
模块间通信采用直接函数调用方式，各模块提供明确的公共 API。

#### 4.2.2 事件通知机制
使用简单的观察者模式实现事件通知：
- **EventBus**：中央事件总线
  - `subscribe(event_type, callback)`: 订阅事件
  - `publish(event_type, data)`: 发布事件
- 主要事件类型：
  - `new_social_activity`: 检测到新的社交媒体活动
  - `market_data_updated`: 市场数据更新
  - `analysis_completed`: 分析完成
  - `notification_sent`: 通知已发送

## 5. 部署与运行

### 5.1 部署架构
系统设计为单体应用，可部署在以下环境：
- 个人电脑
- 小型服务器
- 云平台（如 AWS EC2, DigitalOcean Droplet 等）

### 5.2 运行要求
- Python 3.8+
- SQLite 3
- 网络连接访问各 API
- 至少 512MB RAM
- 至少 1GB 可用磁盘空间

### 5.3 运行模式
- **守护进程模式**：在服务器后台持续运行
- **命令行模式**：用于测试和调试
- **调度任务模式**：通过 cron 等定时调度运行

## 6. 安全设计

### 6.1 敏感数据处理
- API 密钥存储在环境变量或加密配置文件中
- 数据库文件设置适当权限
- 不在日志中记录敏感信息

### 6.2 错误处理
- 所有外部 API 调用都有错误处理和重试机制
- 关键错误会通过 Telegram 通知用户
- 详细日志便于故障排查

### 6.3 未来交易功能的安全考虑
- 交易操作需要额外确认
- 设置交易限额和风险控制参数
- 交易操作完整日志记录

## 7. 后续扩展计划

### 7.1 支持更多社交媒体平台
- 添加 Truth Social 支持
- 考虑增加 Reddit, Discord 等平台

### 7.2 增强 AI 分析能力
- 改进提示工程，提高分析质量
- 支持多模型比较和验证
- 添加结果置信度评估机制

### 7.3 自动交易功能
- 集成主流加密货币交易所 API
- 实现交易策略配置
- 添加风险控制和止损机制

### 7.4 用户界面
- 添加简单的 Web 界面
- 提供历史通知和建议查询
- 可视化市场和分析数据