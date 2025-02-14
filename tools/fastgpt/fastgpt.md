# 本地部署

https://doc.fastgpt.cn/docs/development/docker/

## 准备docker
```bash
# 安装 Docker
curl -fsSL https://get.docker.com | bash -s docker --mirror Aliyun
systemctl enable --now docker
# 安装 docker-compose
curl -L https://github.com/docker/compose/releases/download/v2.20.3/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose
# 验证安装
docker -v
docker-compose -v
# 如失效，自行百度~
```

## 下载 docker-compose.yml
```bash
mkdir fastgpt
cd fastgpt
curl -O https://raw.githubusercontent.com/labring/FastGPT/main/projects/app/data/config.json

# pgvector 版本(测试推荐，简单快捷)
curl -o docker-compose.yml https://raw.githubusercontent.com/labring/FastGPT/main/files/docker/docker-compose-pgvector.yml
# milvus 版本
# curl -o docker-compose.yml https://raw.githubusercontent.com/labring/FastGPT/main/files/docker/docker-compose-milvus.yml
# zilliz 版本
# curl -o docker-compose.yml https://raw.githubusercontent.com/labring/FastGPT/main/files/docker/docker-compose-zilliz.yml
```

## 配置

config.json 修改

```diff
--- old/config.json     2025-01-27 16:51:39.473188301 +0800
+++ config.json 2025-01-27 16:03:53.954796988 +0800
@@ -11,16 +11,16 @@
   },
   "llmModels": [
     {
-      "provider": "OpenAI", // 模型提供商，主要用于分类展示，目前已经内置提供商包括：https://github.com/labring/FastGPT/blob/main/packages/global/core/ai/provider.ts, 可 pr 提供新的提供商，或直接填写 Other
-      "model": "gpt-4o-mini", // 模型名(对应OneAPI中渠道的模型名)
-      "name": "gpt-4o-mini", // 模型别名
-      "maxContext": 125000, // 最大上下文
-      "maxResponse": 16000, // 最大回复
-      "quoteMaxToken": 120000, // 最大引用内容
+      "provider": "DeepSeek", // 模型提供商，主要用于分类展示，目前已经内置提供商包括：https://github.com/labring/FastGPT/blob/main/packages/global/core/ai/provider.ts, 可 pr 提供新的提供商，或直接填写 Other
+      "model": "deepseek-chat", // 模型名(对应OneAPI中渠道的模型名)
+      "name": "deepseek-chat", // 模型别名
+      "maxContext": 12500, // 最大上下文
+      "maxResponse": 4096, // 最大回复
+      "quoteMaxToken": 12000, // 最大引用内容
       "maxTemperature": 1.2, // 最大温度
       "charsPointsPrice": 0, // n积分/1k token（商业版）
       "censor": false, // 是否开启敏感校验（商业版）
-      "vision": true, // 是否支持图片输入
+      "vision": false, // 是否支持图片输入
       "datasetProcess": true, // 是否设置为文本理解模型（QA），务必保证至少有一个为true，否则知识库会报错
       "usedInClassify": true, // 是否用于问题分类（务必保证至少有一个为true）
       "usedInExtractFields": true, // 是否用于内容提取（务必保证至少有一个为true）
@@ -115,39 +115,14 @@
   ],
   "vectorModels": [
     {
-      "provider": "OpenAI",
-      "model": "text-embedding-3-small",
-      "name": "text-embedding-3-small",
-      "charsPointsPrice": 0,
-      "defaultToken": 512,
-      "maxToken": 3000,
-      "weight": 100
-    },
-    {
-      "provider": "OpenAI",
-      "model": "text-embedding-3-large",
-      "name": "text-embedding-3-large",
-      "charsPointsPrice": 0,
-      "defaultToken": 512,
-      "maxToken": 3000,
-      "weight": 100,
-      "defaultConfig": {
-        "dimensions": 1024
-      }
-    },
-    {
-      "provider": "OpenAI",
-      "model": "text-embedding-ada-002", // 模型名（与OneAPI对应）
-      "name": "Embedding-2", // 模型展示名
-      "charsPointsPrice": 0, // n积分/1k token
-      "defaultToken": 700, // 默认文本分割时候的 token
-      "maxToken": 3000, // 最大 token
-      "weight": 100, // 优先训练权重
-      "defaultConfig": {}, // 自定义额外参数。例如，如果希望使用 embedding3-large 的话，可以传入 dimensions:1024，来返回1024维度的向量。（目前必须小于1536维 度）
-      "dbConfig": {}, // 存储时的额外参数（非对称向量模型时候需要用到）
-      "queryConfig": {} // 参训时的额外参数
+      "model": "m3e",
+      "name": "M3E（测试使用）",
+      "price": 0.1,
+      "defaultToken": 500,
+      "maxToken": 1800
     }
-  ],
+  ]
+  ,
   "reRankModels": [],
   "audioSpeechModels": [
     {
```

docker-compose.yml 修改
```diff
--- old/docker-compose.yml      2025-01-27 16:51:45.458547321 +0800
+++ ./docker-compose.yml        2025-01-27 16:10:06.250770190 +0800
@@ -92,12 +92,13 @@
     restart: always
     environment:
       # 前端访问地址: http://localhost:3000
-      - FE_DOMAIN=
+      - FE_DOMAIN=http://0.0.0.0:3000
       - DEFAULT_ROOT_PSW=1234
-      - OPENAI_BASE_URL=http://oneapi:3000/v1
+      - OPENAI_BASE_URL=http://localhost:3001/v1
+      - CHAT_API_KEY=sk-fastgpt
       # 数据库最大连接数
       - DB_MAX_LINK=30
@@ -144,6 +145,15 @@
       MYSQL_DATABASE: oneapi
     volumes:
       - ./mysql:/var/lib/mysql
+  m3e:
+    container_name: m3e
+    image: stawky/m3e-large-api
+    ports:
+      - 6008:6008
+    networks:
+      - fastgpt
+    restart: always
+
   oneapi:
     container_name: oneapi
     image: ghcr.io/songquanpeng/one-api:v0.6.7
```

### 拉取m3e
```bash
docker pull stawky/m3e-large-api:latest
```
## 启动
```bash
# 启动容器
docker-compose up -d
# 等待10s，OneAPI第一次总是要重启几次才能连上Mysql
sleep 10
# 重启一次oneapi(由于OneAPI的默认Key有点问题，不重启的话会提示找不到渠道，临时手动重启一次解决，等待作者修复)
docker restart oneapi
```

### oneapi 添加各种大模型
oneapi 可以通过 ip:3001 访问，默认账号 root/123456

登录 - 渠道 - 添加新的渠道
- m3e
  - 类型 : 自定义渠道
  - Base Url : http://192.168.33.129:6008
  - 名称 : m3e
  - 分组 : default
  - 模型 : m3e
  - 密钥 : sk-aaabbbcccdddeeefffggghhhiiijjjkkk
- deepseek
  - 类型 : DeepSeek
  - 名称 : m3e
  - 分组 : default
  - 模型 : deepseek-chat
  - 密钥 : 从deepseek获取
  - 代理 : https://api.deepseek.com

#### 从oneapi获取令牌
可以直接使用 Inital Root Token的令牌

也可以创建令牌，但选择模型时注意将deepseek 和 m3e都添加

填写 docker-compose.yml后重新启动镜像

# 访问fastgpt
通过 ip:3000 访问，默认账号 root，密码为docker-compose.yml环境变量里设置的 DEFAULT_ROOT_PSW。

