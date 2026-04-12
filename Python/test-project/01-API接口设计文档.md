# API接口设计文档

## 文档信息

- **版本**: v1.0
- **日期**: 2026-04-12
- **环境**: Python + FastAPI + MySQL + Elasticsearch

---

## 目录

1. [接口规范](#1-接口规范)
2. [认证接口](#2-认证接口)
3. [用户管理接口](#3-用户管理接口)
4. [数据源管理接口](#4-数据源管理接口)
5. [数据采集接口](#5-数据采集接口)
6. [数据处理接口](#6-数据处理接口)
7. [全文检索接口](#7-全文检索接口)
8. [数据分析接口](#8-数据分析接口)
9. [可视化接口](#9-可视化接口)
10. [系统监控接口](#10-系统监控接口)

---

## 1. 接口规范

### 1.1 基础信息

| 项目 | 说明 |
|------|------|
| 基础URL | `http://localhost/api/v1` |
| 协议 | HTTP/1.1 (开发) / HTTPS (生产) |
| 字符编码 | UTF-8 |
| 请求格式 | JSON |
| 响应格式 | JSON |

### 1.2 通用响应格式

```json
{
  "code": 200,
  "message": "success",
  "data": {},
  "timestamp": 1712900000,
  "request_id": "req_abc123"
}
```

### 1.3 状态码定义

| 状态码 | 说明 |
|--------|------|
| 200 | 成功 |
| 400 | 请求参数错误 |
| 401 | 未认证 |
| 403 | 无权限 |
| 404 | 资源不存在 |
| 422 | 验证错误 |
| 429 | 请求过于频繁 |
| 500 | 服务器内部错误 |

### 1.4 认证方式

使用 JWT Bearer Token:

```
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

---

## 2. 认证接口

### 2.1 用户登录

**POST** `/auth/login`

**请求体:**
```json
{
  "username": "admin",
  "password": "password123",
  "captcha": "a1b2c3"
}
```

**响应:**
```json
{
  "code": 200,
  "message": "登录成功",
  "data": {
    "access_token": "eyJhbGciOiJIUzI1NiIs...",
    "refresh_token": "eyJhbGciOiJIUzI1NiIs...",
    "token_type": "bearer",
    "expires_in": 1800,
    "user": {
      "id": 1,
      "username": "admin",
      "nickname": "管理员",
      "roles": ["admin"]
    }
  }
}
```

### 2.2 刷新Token

**POST** `/auth/refresh`

**请求头:**
```
Authorization: Bearer {refresh_token}
```

**响应:**
```json
{
  "code": 200,
  "message": "刷新成功",
  "data": {
    "access_token": "eyJhbGciOiJIUzI1NiIs...",
    "expires_in": 1800
  }
}
```

### 2.3 用户登出

**POST** `/auth/logout`

**响应:**
```json
{
  "code": 200,
  "message": "登出成功",
  "data": null
}
```

### 2.4 获取验证码

**GET** `/auth/captcha`

**响应:**
```json
{
  "code": 200,
  "message": "success",
  "data": {
    "captcha_id": "cap_abc123",
    "captcha_image": "data:image/png;base64,iVBORw0KGgo...",
    "expire_in": 300
  }
}
```

---

## 3. 用户管理接口

### 3.1 获取用户列表

**GET** `/users`

**查询参数:**
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| page | int | 否 | 页码，默认1 |
| page_size | int | 否 | 每页数量，默认20 |
| keyword | string | 否 | 搜索关键词 |
| status | int | 否 | 状态：0-禁用 1-启用 |

**响应:**
```json
{
  "code": 200,
  "message": "success",
  "data": {
    "total": 100,
    "page": 1,
    "page_size": 20,
    "items": [
      {
        "id": 1,
        "username": "admin",
        "nickname": "管理员",
        "email": "admin@example.com",
        "phone": "13800138000",
        "status": 1,
        "roles": ["admin"],
        "created_at": "2024-01-01T00:00:00Z",
        "updated_at": "2024-01-01T00:00:00Z"
      }
    ]
  }
}
```

### 3.2 创建用户

**POST** `/users`

**请求体:**
```json
{
  "username": "newuser",
  "password": "Pass1234!",
  "nickname": "新用户",
  "email": "user@example.com",
  "phone": "13900139000",
  "roles": ["user"],
  "status": 1
}
```

### 3.3 更新用户

**PUT** `/users/{id}`

**请求体:**
```json
{
  "nickname": "更新后的昵称",
  "email": "new@example.com",
  "roles": ["user", "editor"],
  "status": 1
}
```

### 3.4 删除用户

**DELETE** `/users/{id}`

**响应:**
```json
{
  "code": 200,
  "message": "删除成功",
  "data": null
}
```

### 3.5 修改密码

**PUT** `/users/{id}/password`

**请求体:**
```json
{
  "old_password": "oldPass123!",
  "new_password": "newPass456!"
}
```

---

## 4. 数据源管理接口

### 4.1 获取数据源列表

**GET** `/datasources`

**查询参数:**
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| page | int | 否 | 页码 |
| page_size | int | 否 | 每页数量 |
| type | string | 否 | 类型：mysql/postgresql/api/file |
| status | int | 否 | 状态 |

**响应:**
```json
{
  "code": 200,
  "message": "success",
  "data": {
    "total": 10,
    "items": [
      {
        "id": 1,
        "name": "生产数据库",
        "type": "mysql",
        "host": "192.168.1.100",
        "port": 3306,
        "database": "production",
        "username": "readonly",
        "status": 1,
        "created_at": "2024-01-01T00:00:00Z"
      }
    ]
  }
}
```

### 4.2 创建数据源

**POST** `/datasources`

**请求体:**
```json
{
  "name": "测试数据库",
  "type": "mysql",
  "host": "192.168.1.101",
  "port": 3306,
  "database": "test_db",
  "username": "test_user",
  "password": "db_password",
  "description": "测试环境数据库",
  "status": 1
}
```

### 4.3 测试数据源连接

**POST** `/datasources/{id}/test`

**响应:**
```json
{
  "code": 200,
  "message": "连接成功",
  "data": {
    "connected": true,
    "latency_ms": 15,
    "version": "8.0.35"
  }
}
```

### 4.4 更新数据源

**PUT** `/datasources/{id}`

### 4.5 删除数据源

**DELETE** `/datasources/{id}`

---

## 5. 数据采集接口

### 5.1 获取采集任务列表

**GET** `/collect/tasks`

**查询参数:**
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| page | int | 否 | 页码 |
| page_size | int | 否 | 每页数量 |
| status | string | 否 | 状态：pending/running/completed/failed |

**响应:**
```json
{
  "code": 200,
  "message": "success",
  "data": {
    "total": 50,
    "items": [
      {
        "id": 1,
        "name": "每日用户数据同步",
        "datasource_id": 1,
        "task_type": "sql",
        "cron": "0 2 * * *",
        "status": "completed",
        "last_run": "2024-04-11T02:00:00Z",
        "next_run": "2024-04-12T02:00:00Z",
        "success_count": 100,
        "fail_count": 0
      }
    ]
  }
}
```

### 5.2 创建采集任务

**POST** `/collect/tasks`

**请求体:**
```json
{
  "name": "订单数据同步",
  "datasource_id": 1,
  "task_type": "sql",
  "source_config": {
    "sql": "SELECT * FROM orders WHERE updated_at >= :last_sync_time",
    "primary_key": "id",
    "batch_size": 500
  },
  "target_config": {
    "table": "orders",
    "mapping": {
      "id": "order_id",
      "user_id": "user_id",
      "amount": "amount",
      "created_at": "created_at"
    }
  },
  "schedule": {
    "type": "cron",
    "cron": "0 */6 * * *"
  },
  "status": 1
}
```

### 5.3 执行采集任务

**POST** `/collect/tasks/{id}/execute`

**请求体:**
```json
{
  "sync_mode": "incremental",
  "start_time": "2024-04-01T00:00:00Z",
  "end_time": "2024-04-11T23:59:59Z"
}
```

**响应:**
```json
{
  "code": 200,
  "message": "任务已提交",
  "data": {
    "task_id": "task_abc123",
    "status": "running",
    "progress": 0,
    "estimated_time": "5m"
  }
}
```

### 5.4 获取采集任务日志

**GET** `/collect/tasks/{id}/logs`

**响应:**
```json
{
  "code": 200,
  "message": "success",
  "data": {
    "total": 100,
    "items": [
      {
        "id": 1,
        "level": "info",
        "message": "开始执行数据采集",
        "timestamp": "2024-04-11T02:00:00Z"
      }
    ]
  }
}
```

### 5.5 API数据采集

**POST** `/collect/api`

**请求体:**
```json
{
  "name": "第三方API数据",
  "url": "https://api.example.com/v1/data",
  "method": "GET",
  "headers": {
    "Authorization": "Bearer token123",
    "Content-Type": "application/json"
  },
  "params": {
    "page": 1,
    "limit": 100
  },
  "pagination": {
    "type": "page",
    "page_param": "page",
    "total_path": "data.total"
  },
  "data_path": "data.items",
  "target_table": "api_data"
}
```

---

## 6. 数据处理接口

### 6.1 获取数据处理任务

**GET** `/process/tasks`

**响应:**
```json
{
  "code": 200,
  "message": "success",
  "data": {
    "total": 20,
    "items": [
      {
        "id": 1,
        "name": "数据清洗任务",
        "process_type": "clean",
        "input_table": "raw_orders",
        "output_table": "clean_orders",
        "status": "completed",
        "processed_count": 10000,
        "created_at": "2024-04-11T00:00:00Z"
      }
    ]
  }
}
```

### 6.2 创建数据处理任务

**POST** `/process/tasks`

**请求体:**
```json
{
  "name": "用户数据统计",
  "process_type": "aggregate",
  "config": {
    "input_table": "orders",
    "output_table": "user_statistics",
    "operations": [
      {
        "type": "group_by",
        "columns": ["user_id"]
      },
      {
        "type": "aggregate",
        "aggregations": {
          "total_amount": "SUM(amount)",
          "order_count": "COUNT(*)",
          "avg_amount": "AVG(amount)"
        }
      }
    ]
  },
  "schedule": {
    "type": "interval",
    "minutes": 60
  }
}
```

### 6.3 数据清洗配置

**POST** `/process/clean`

**请求体:**
```json
{
  "input_table": "raw_data",
  "output_table": "clean_data",
  "rules": [
    {
      "field": "email",
      "rule": "email_format",
      "action": "remove_invalid"
    },
    {
      "field": "phone",
      "rule": "phone_format",
      "pattern": "^1[3-9]\\d{9}$",
      "action": "normalize"
    },
    {
      "field": "age",
      "rule": "range",
      "min": 0,
      "max": 150,
      "action": "set_null"
    },
    {
      "field": "name",
      "rule": "trim"
    },
    {
      "field": "duplicates",
      "rule": "unique",
      "keys": ["email", "phone"],
      "action": "keep_first"
    }
  ]
}
```

### 6.4 数据转换

**POST** `/process/transform`

**请求体:**
```json
{
  "input_table": "source_table",
  "output_table": "target_table",
  "transformations": [
    {
      "field": "price",
      "type": "cast",
      "to": "decimal(10,2)"
    },
    {
      "field": "category",
      "type": "map",
      "mapping": {
        "1": "电子产品",
        "2": "服装",
        "3": "食品"
      }
    },
    {
      "field": "tags",
      "type": "split",
      "separator": ",",
      "output_type": "array"
    },
    {
      "field": "created_at",
      "type": "format",
      "from_format": "timestamp",
      "to_format": "YYYY-MM-DD HH:mm:ss"
    }
  ]
}
```

### 6.5 批量数据导入

**POST** `/process/import`

**请求体 (multipart/form-data):**
```
file: [CSV/Excel文件]
table: target_table
header_row: 1
delimiter: ,
encoding: utf-8
```

**响应:**
```json
{
  "code": 200,
  "message": "导入任务已创建",
  "data": {
    "task_id": "import_abc123",
    "status": "processing",
    "total_rows": 10000,
    "processed_rows": 0,
    "estimated_time": "2m"
  }
}
```

### 6.6 数据导出

**POST** `/process/export`

**请求体:**
```json
{
  "table": "orders",
  "format": "csv",
  "filters": {
    "created_at": {
      "gte": "2024-01-01",
      "lte": "2024-12-31"
    }
  },
  "columns": ["id", "user_id", "amount", "status", "created_at"],
  "order_by": "created_at DESC"
}
```

**响应:**
```json
{
  "code": 200,
  "message": "导出任务已创建",
  "data": {
    "task_id": "export_abc123",
    "download_url": "/api/v1/files/export_abc123.csv",
    "expires_at": "2024-04-12T00:00:00Z"
  }
}
```

---

## 7. 全文检索接口

### 7.1 执行搜索

**POST** `/search`

**请求体:**
```json
{
  "query": "关键词",
  "indices": ["articles", "documents"],
  "fields": ["title^2", "content", "tags"],
  "filters": {
    "category": "技术",
    "created_at": {
      "gte": "2024-01-01",
      "lte": "2024-12-31"
    }
  },
  "sort": [
    { "_score": "desc" },
    { "created_at": "desc" }
  ],
  "highlight": {
    "fields": ["title", "content"],
    "pre_tags": ["<mark>"],
    "post_tags": ["</mark>"]
  },
  "page": 1,
  "page_size": 20
}
```

**响应:**
```json
{
  "code": 200,
  "message": "success",
  "data": {
    "total": 150,
    "page": 1,
    "page_size": 20,
    "took_ms": 25,
    "items": [
      {
        "_index": "articles",
        "_id": "1",
        "_score": 8.5,
        "_source": {
          "title": "Python <mark>关键词</mark>教程",
          "content": "这是一篇关于<mark>关键词</mark>的教程...",
          "author": "张三",
          "category": "技术",
          "tags": ["python", "教程"],
          "created_at": "2024-04-10T10:00:00Z"
        },
        "highlight": {
          "title": ["Python <mark>关键词</mark>教程"],
          "content": ["这是一篇关于<mark>关键词</mark>的教程..."]
        }
      }
    ],
    "aggregations": {
      "categories": [
        { "key": "技术", "count": 80 },
        { "key": "产品", "count": 40 },
        { "key": "设计", "count": 30 }
      ]
    }
  }
}
```

### 7.2 自动补全/建议

**GET** `/search/suggest`

**查询参数:**
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| q | string | 是 | 输入前缀 |
| index | string | 否 | 索引名称 |
| size | int | 否 | 返回数量，默认10 |

**响应:**
```json
{
  "code": 200,
  "message": "success",
  "data": {
    "suggestions": [
      { "text": "关键词1", "score": 10 },
      { "text": "关键词2", "score": 8 },
      { "text": "关键词3", "score": 6 }
    ]
  }
}
```

### 7.3 索引管理

**POST** `/search/indices`

**请求体:**
```json
{
  "name": "articles",
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 0,
    "analysis": {
      "analyzer": {
        "ik_smart": {
          "type": "custom",
          "tokenizer": "ik_smart"
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "title": { "type": "text", "analyzer": "ik_smart" },
      "content": { "type": "text", "analyzer": "ik_smart" },
      "author": { "type": "keyword" },
      "category": { "type": "keyword" },
      "tags": { "type": "keyword" },
      "created_at": { "type": "date" }
    }
  }
}
```

### 7.4 文档操作

**POST** `/search/{index}/docs`

**请求体 (添加/更新文档):**
```json
{
  "id": "doc_123",
  "title": "文档标题",
  "content": "文档内容...",
  "author": "作者名",
  "tags": ["标签1", "标签2"],
  "created_at": "2024-04-11T12:00:00Z"
}
```

**DELETE** `/search/{index}/docs/{id}`

删除指定文档。

### 7.5 批量索引

**POST** `/search/{index}/bulk`

**请求体:**
```json
{
  "actions": [
    { "index": { "_id": "1" } },
    { "title": "文章1", "content": "内容1" },
    { "index": { "_id": "2" } },
    { "title": "文章2", "content": "内容2" },
    { "delete": { "_id": "3" } }
  ]
}
```

---

## 8. 数据分析接口

### 8.1 执行SQL查询

**POST** `/analytics/query`

**请求体:**
```json
{
  "sql": "SELECT * FROM orders WHERE amount > :min_amount",
  "params": {
    "min_amount": 100
  },
  "limit": 1000,
  "export": false
}
```

**响应:**
```json
{
  "code": 200,
  "message": "success",
  "data": {
    "columns": ["id", "user_id", "amount", "status", "created_at"],
    "rows": [
      [1, 1001, 199.99, "completed", "2024-04-10T10:00:00Z"],
      [2, 1002, 299.99, "completed", "2024-04-10T11:00:00Z"]
    ],
    "total": 500,
    "execution_time_ms": 45
  }
}
```

### 8.2 统计分析

**POST** `/analytics/statistics`

**请求体:**
```json
{
  "table": "orders",
  "fields": ["amount", "quantity"],
  "group_by": ["category", "status"],
  "metrics": ["count", "sum", "avg", "min", "max", "std"],
  "filters": {
    "created_at": {
      "gte": "2024-01-01",
      "lte": "2024-12-31"
    }
  }
}
```

**响应:**
```json
{
  "code": 200,
  "message": "success",
  "data": {
    "summary": {
      "total_count": 10000,
      "total_amount": 1500000.50
    },
    "groups": [
      {
        "category": "电子产品",
        "status": "completed",
        "count": 3000,
        "sum_amount": 800000.00,
        "avg_amount": 266.67,
        "min_amount": 10.00,
        "max_amount": 5000.00,
        "std_amount": 150.25
      }
    ]
  }
}
```

### 8.3 时间序列分析

**POST** `/analytics/timeseries`

**请求体:**
```json
{
  "table": "orders",
  "time_field": "created_at",
  "interval": "1d",
  "range": {
    "from": "2024-01-01",
    "to": "2024-04-11"
  },
  "metrics": {
    "order_count": "COUNT(*)",
    "total_amount": "SUM(amount)",
    "avg_amount": "AVG(amount)"
  },
  "group_by": ["category"]
}
```

**响应:**
```json
{
  "code": 200,
  "message": "success",
  "data": {
    "series": [
      {
        "timestamp": "2024-04-01",
        "category": "电子产品",
        "order_count": 50,
        "total_amount": 15000,
        "avg_amount": 300
      }
    ]
  }
}
```

### 8.4 数据透视表

**POST** `/analytics/pivot`

**请求体:**
```json
{
  "table": "sales",
  "rows": ["region", "city"],
  "columns": ["product_category"],
  "values": {
    "amount": "SUM",
    "quantity": "SUM"
  },
  "filters": {
    "year": 2024
  }
}
```

### 8.5 相关性分析

**POST** `/analytics/correlation`

**请求体:**
```json
{
  "table": "user_behavior",
  "fields": ["page_views", "time_spent", "conversion_rate", "purchase_amount"],
  "method": "pearson"
}
```

**响应:**
```json
{
  "code": 200,
  "message": "success",
  "data": {
    "correlation_matrix": {
      "page_views": { "time_spent": 0.75, "conversion_rate": 0.45, "purchase_amount": 0.30 },
      "time_spent": { "page_views": 0.75, "conversion_rate": 0.60, "purchase_amount": 0.40 },
      "conversion_rate": { "page_views": 0.45, "time_spent": 0.60, "purchase_amount": 0.80 },
      "purchase_amount": { "page_views": 0.30, "time_spent": 0.40, "conversion_rate": 0.80 }
    }
  }
}
```

---

## 9. 可视化接口

### 9.1 获取图表列表

**GET** `/charts`

**响应:**
```json
{
  "code": 200,
  "message": "success",
  "data": {
    "total": 30,
    "items": [
      {
        "id": 1,
        "name": "月度销售额趋势",
        "type": "line",
        "datasource": "orders",
        "created_at": "2024-04-01T00:00:00Z"
      }
    ]
  }
}
```

### 9.2 创建图表

**POST** `/charts`

**请求体:**
```json
{
  "name": "产品类别占比",
  "type": "pie",
  "datasource": "orders",
  "config": {
    "title": {
      "text": "产品类别销售占比",
      "left": "center"
    },
    "series": [
      {
        "type": "pie",
        "radius": ["40%", "70%"],
        "data_query": {
          "sql": "SELECT category, SUM(amount) as value FROM orders GROUP BY category",
          "name_field": "category",
          "value_field": "value"
        }
      }
    ]
  }
}
```

### 9.3 获取图表数据

**GET** `/charts/{id}/data`

**响应:**
```json
{
  "code": 200,
  "message": "success",
  "data": {
    "title": "产品类别销售占比",
    "series": [
      {
        "type": "pie",
        "data": [
          { "name": "电子产品", "value": 500000 },
          { "name": "服装", "value": 300000 },
          { "name": "食品", "value": 200000 }
        ]
      }
    ]
  }
}
```

### 9.4 获取仪表盘列表

**GET** `/dashboards`

### 9.5 创建仪表盘

**POST** `/dashboards`

**请求体:**
```json
{
  "name": "销售数据大屏",
  "description": "实时监控销售数据",
  "layout": [
    {
      "chart_id": 1,
      "x": 0,
      "y": 0,
      "w": 6,
      "h": 4
    },
    {
      "chart_id": 2,
      "x": 6,
      "y": 0,
      "w": 6,
      "h": 4
    }
  ],
  "refresh_interval": 60
}
```

### 9.6 获取仪表盘详情

**GET** `/dashboards/{id}`

**响应:**
```json
{
  "code": 200,
  "message": "success",
  "data": {
    "id": 1,
    "name": "销售数据大屏",
    "description": "实时监控销售数据",
    "charts": [
      {
        "id": 1,
        "name": "月度销售额趋势",
        "type": "line",
        "data": { ... },
        "position": { "x": 0, "y": 0, "w": 6, "h": 4 }
      }
    ],
    "refresh_interval": 60
  }
}
```

---

## 10. 系统监控接口

### 10.1 系统状态

**GET** `/system/status`

**响应:**
```json
{
  "code": 200,
  "message": "success",
  "data": {
    "status": "healthy",
    "version": "1.0.0",
    "uptime": "5d 3h 20m",
    "components": {
      "mysql": { "status": "up", "latency_ms": 5 },
      "redis": { "status": "up", "latency_ms": 2 },
      "elasticsearch": { "status": "up", "latency_ms": 15 },
      "rabbitmq": { "status": "up", "latency_ms": 8 }
    }
  }
}
```

### 10.2 资源使用情况

**GET** `/system/resources`

**响应:**
```json
{
  "code": 200,
  "message": "success",
  "data": {
    "cpu": {
      "usage_percent": 35.5,
      "cores": 4
    },
    "memory": {
      "total_gb": 8,
      "used_gb": 5.2,
      "free_gb": 2.8,
      "usage_percent": 65
    },
    "disk": {
      "total_gb": 160,
      "used_gb": 80,
      "free_gb": 80,
      "usage_percent": 50
    }
  }
}
```

### 10.3 任务队列状态

**GET** `/system/queues`

**响应:**
```json
{
  "code": 200,
  "message": "success",
  "data": {
    "queues": [
      {
        "name": "data_collect",
        "pending": 10,
        "processing": 2,
        "completed": 1000,
        "failed": 5
      },
      {
        "name": "data_process",
        "pending": 5,
        "processing": 1,
        "completed": 500,
        "failed": 2
      }
    ]
  }
}
```

### 10.4 系统日志

**GET** `/system/logs`

**查询参数:**
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| level | string | 否 | 日志级别：debug/info/warning/error |
| start_time | string | 否 | 开始时间 |
| end_time | string | 否 | 结束时间 |
| limit | int | 否 | 返回数量 |

**响应:**
```json
{
  "code": 200,
  "message": "success",
  "data": {
    "total": 1000,
    "items": [
      {
        "timestamp": "2024-04-11T12:00:00Z",
        "level": "info",
        "message": "数据采集任务完成",
        "source": "collect_service",
        "task_id": "task_123"
      }
    ]
  }
}
```

### 10.5 告警列表

**GET** `/system/alerts`

**响应:**
```json
{
  "code": 200,
  "message": "success",
  "data": {
    "total": 5,
    "items": [
      {
        "id": 1,
        "level": "warning",
        "title": "磁盘空间不足",
        "message": "磁盘使用率超过85%",
        "component": "disk",
        "created_at": "2024-04-11T10:00:00Z",
        "acknowledged": false
      }
    ]
  }
}
```

---

## 附录

### A. 接口调用示例

**Python示例:**
```python
import requests

# 登录获取Token
response = requests.post("http://localhost/api/v1/auth/login", json={
    "username": "admin",
    "password": "password123"
})
token = response.json()["data"]["access_token"]

# 使用Token调用API
headers = {"Authorization": f"Bearer {token}"}
response = requests.get("http://localhost/api/v1/users", headers=headers)
print(response.json())
```

**curl示例:**
```bash
# 登录
curl -X POST http://localhost/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"password123"}'

# 获取用户列表
curl -X GET http://localhost/api/v1/users \
  -H "Authorization: Bearer YOUR_TOKEN"
```

### B. 错误码对照表

| 错误码 | 说明 | 处理建议 |
|--------|------|----------|
| 400001 | 参数验证失败 | 检查请求参数格式 |
| 401001 | Token过期 | 使用refresh_token刷新 |
| 401002 | Token无效 | 重新登录 |
| 403001 | 权限不足 | 联系管理员获取权限 |
| 429001 | 请求过于频繁 | 降低请求频率 |
| 500001 | 数据库错误 | 联系系统管理员 |
