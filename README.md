# HTML 链接下载工具 / HTML Links Downloader Tool

一个基于 Vue 3 的 Web 工具，用于批量下载多个网页的 HTML 内容并打包成 ZIP 文件。

A Vue 3 based web tool for batch downloading HTML content from multiple URLs and packaging them into a ZIP file.

## 功能特性 / Features

- 📝 支持批量输入多个 URL（每行一个）
- 📦 自动将所有页面的 HTML 内容打包成 ZIP 文件
- 🔄 实时显示下载进度
- 📊 详细的处理日志
- 🌐 支持 CORS 代理选项
- 🌓 自动适配深色/浅色主题

## 使用方法 / Usage

1. 在文本框中输入要下载的链接列表（每行一个 URL）
2. 如果遇到 CORS 问题，可以勾选"使用 CORS 代理"选项
3. 点击"下载 ZIP"按钮
4. 等待处理完成后，ZIP 文件会自动下载

## 本地开发 / Local Development

```bash
# 安装依赖
npm install

# 启动开发服务器
npm run dev

# 构建生产版本
npm run build

# 预览生产构建
npm run preview
```

## 技术栈 / Tech Stack

- Vue 3
- Vite
- JSZip
- 原生 Fetch API

## 部署 / Deployment

本项目配置了 GitHub Actions，会自动将 main 分支的代码构建并部署到 GitHub Pages。

This project is configured with GitHub Actions to automatically build and deploy the main branch to GitHub Pages.

## License

ISC