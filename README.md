# BTC-Price-Visualization

基于 Vue.js 的比特币价格可视化项目，展示比特币历史价格走势及重要事件标记。

## 功能特性

- 实时显示比特币价格走势图
- 支持历史数据回溯（最近8年数据）
- 重要事件标记和展示
- 价格变化动画效果
- 响应式设计，适配不同屏幕尺寸
- 支持数据导出功能

## 数据需求

### 价格数据
- 数据来源：Binance API
- 更新频率：日级别
- 数据格式：
  ```json
  {
    "date": "时间戳",
    "price": "收盘价"
  }
  ```

### 事件数据
- 格式要求：
  ```json
  {
    "date": "事件发生时间",
    "title": "事件标题",
    "description": "事件描述",
    "type": "事件类型（positive/negative/neutral）"
  }
  ```

## 项目设置
```
npm install
```

### 开发环境编译和热重载
```
npm run serve
```

### 生产环境编译和压缩
```
npm run build
```

### 代码检查和修复
```
npm run lint
```

## 环境要求

- Node.js >= 14.0.0
- Vue.js >= 3.0.0
- D3.js >= 7.0.0

## API 限制

- Binance API 调用限制：
  - 权重：1200/分钟
  - IP限制：2400次/小时

## 开发注意事项

1. 本地开发需要配置 `.env.local` 文件
2. 确保遵守 API 使用限制
3. 大数据量渲染时注意性能优化

## 部署说明

1. 确保生产环境的 API 配置正确
2. 建议使用 CDN 加速静态资源
3. 启用 GZIP 压缩

## 自定义配置
更多配置信息请参考 [Vue CLI 配置参考](https://cli.vuejs.org/config/).

## 贡献指南

1. Fork 本仓库
2. 创建特性分支
3. 提交更改
4. 发起 Pull Request

## 许可证

MIT License
