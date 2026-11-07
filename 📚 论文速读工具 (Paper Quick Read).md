# 📚 论文速读工具 (Paper Quick Read)

> 一个集成信息检索与结构化总结的个人学习助手。输入研究主题或论文文本，一键生成包含「核心要点 / 关键术语 / 延伸问题 / 真实来源」的结构化速读笔记。

![Vue 3](https://img.shields.io/badge/Vue-3.0-42b883?logo=vue.js&logoColor=white)
![Vite](https://img.shields.io/badge/Vite-5.0-646CFF?logo=vite&logoColor=white)
![Dify](https://img.shields.io/badge/Powered%20by-Dify-2563eb)
![License](https://img.shields.io/badge/license-MIT-blue)

## ✨ 核心特性

- **🤖 真实 AI 检索与总结**：对接 Dify 工作流（集成 Tavily 搜索 + DeepSeek 大模型），拒绝伪造，每一条引用均可溯源至真实学术网站。
- **📑 多标签页视图**：像浏览器一样同时打开多个笔记，自由切换，方便对比阅读。
- **🗂 智能历史记录**：基于 `localStorage` 本地存储，支持记录**置顶**、**单条删除**与一键清空。
- **📝 结构化渲染**：严格按照契约将 AI 返回的 JSON 渲染为「总览、要点、术语、问题、参考来源」五大区块。
- **🎨 现代 UI 与动效**：磨砂玻璃质感、渐变边框、入场动画、定制的“猫头鹰阅读”Loading 动效。
- ##### **🛡 健壮的容错机制**：自动拦截 401 鉴权失败、超时、节点报错，并剥离大模型的 `<think>` 思考标签。===
- **📋 一键导出**：支持将笔记复制为标准的 Markdown 格式。

## 🛠 技术栈

- **前端框架**：Vue 3 (Composition API) + Vite
- **AI 后端**：Dify (Workflow API)  //本人用Dify搭建的后端
- **检索插件**：Tavily Search API
- **大语言模型**：DeepSeek API
- **UI 动效**：原生 CSS3 (无第三方组件库)

## 🚀 本地运行指南

### 1. 克隆项目
```bash
git clone <your-repo-url>
cd paper-reader-frontend
```

### 2. 安装依赖

```
npm install
```

### 3. 配置环境变量

在项目根目录新建一个 `.env` 文件（参考 `.env.example`），填入你的 Dify API 信息：（或其他API）

```
# 是否开启本地 Mock 模拟数据（true/false）
VITE_USE_MOCK=false

# Dify 的 API 地址（通常在 Dify 后台的“访问 API”页面获取）
VITE_API_BASE=

# 你在 Dify 创建的 API 密钥 (务必以 app- 开头)
VITE_API_KEY=app-xxxxxxxxxxxxxxxxxxxxxx
```

### 4. 启动开发服务器

```
npm run dev
```

打开浏览器访问 `http://localhost:5173` 即可看到页面。

## 📁 项目目录结构

```
paper-reader-frontend/
├── public/
│   └── OWL.png          # 猫头鹰 Logo 与 Loading 动效图片
├── src/
│   ├── App.vue          # 核心页面与全部逻辑 (Vue 3 setup)
│   ├── main.js          # 入口文件
│   └── style.css        # 全局基础样式（已清空 Vite 默认样式）
├── .env                 # 本地环境变量（不提交至 Git）
├── .env.example         # 环境变量模板
├── index.html           # HTML 入口（已配置 Favicon）
├── package.json
└── vite.config.js
```

## 🔗 接口契约说明 (v1.1)

前端期望 Dify 返回的标准 JSON 结构如下（若格式不符，前端会将其拦截并提示解析失败）：

```
{
  "ok": true,
  "summary": "一句话总览（60字以内）",
  "points": [
    { "text": "核心要点内容", "source_ids": [1, 2] }
  ],
  "terms": [
    { "term": "术语名称", "explain": "通俗解释" }
  ],
  "questions": ["延伸问题1", "延伸问题2"],
  "sources": [
    { "id": 1, "title": "参考文献标题", "url": "https://..." }
  ]
}
```

