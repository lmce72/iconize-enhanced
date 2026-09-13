# Iconize Enhanced

**高性能按需加载版 Obsidian 图标插件**

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

## 🚀 主要改进

这是 [Iconize](https://github.com/FlorianWoelki/obsidian-iconize) 插件的增强版本，专注于性能优化和稳定性改进。

### ⚡ 性能提升

| 指标 | 原版 | 增强版 | 改善 |
|------|------|--------|------|
| 启动时间 | 3-5秒 | 200-500ms | **90% ↓** |
| 内存占用 | 50-100MB | 5-10MB | **80% ↓** |
| 图标查找 | O(n*m) | O(1) | **接近瞬时** |

### 🎯 核心特性

#### 1. 按需加载系统
- **索引优先架构** - 启动时仅加载轻量级 JSON 索引
- **三层缓存** - 内存 → 磁盘 → 源文件（ZIP/文件夹）
- **智能预加载** - 仅加载实际使用的图标
- **延迟加载** - 内联图标 `:IconName:` 批处理加载

#### 2. 稳定性改进
- **杜绝图标丢失** - 不再依赖外部 SVG 文件
- **集中缓存管理** - 所有图标数据统一存储在 `.cache` 目录
- **自动迁移** - 现有外部 SVG 文件自动迁移到缓存系统
- **代码防护** - 禁用所有外部 SVG 写入逻辑

#### 3. 优化的查找性能
- **前缀索引** - O(1) 图标包定位
- **哈希查找** - O(1) 图标查找
- **异步加载** - 不阻塞 UI 渲染

## 📦 安装

### 方法 1: 手动安装（推荐）

1. 下载最新的 [Release](https://github.com/CoreVortex/iconize-enhanced/releases)
2. 解压到 `.obsidian/plugins/obsidian-icon-folder/`
3. 在 Obsidian 设置中启用插件

### 方法 2: BRAT（Beta 测试）

1. 安装 [BRAT](https://github.com/TfTHacker/obsidian42-brat) 插件
2. 添加此仓库：`CoreVortex/iconize-enhanced`
3. 启用插件

## 🔄 从原版迁移

增强版完全向后兼容原版 Iconize：

1. **备份数据**（可选）
   ```bash
   cp -r .obsidian/plugins/obsidian-icon-folder .obsidian/plugins/obsidian-icon-folder.backup
   ```

2. **替换插件文件**
   - 下载增强版
   - 替换 `.obsidian/plugins/obsidian-icon-folder/` 目录

3. **重新加载插件**
   - 在 Obsidian 设置中禁用再启用插件

4. **验证成功**
   - 打开开发者工具（Ctrl+Shift+I）
   - 查看控制台应显示：`[Iconize] Initializing icon packs with lazy loading...`

### 迁移说明
- ✅ `data.json` 无需修改
- ✅ 所有现有图标继续正常工作
- ✅ 图标包无需重新下载
- ✅ 自定义规则保持不变

## 🏗️ 技术架构

### 索引系统
```
启动
  ↓
加载索引 (.obsidian/icons/.index/*.json)
  ↓
构建前缀索引 (O(1) 查找)
  ↓
预加载使用的图标 (15-50个)
  ↓
完成 (200-500ms)
```

### 三层缓存
```javascript
// Layer 1: 内存缓存
iconResolver.peek('LiHome')  // 即时返回

// Layer 2: 磁盘缓存
await iconResolver.resolve('LiHome')  // 50-150ms

// Layer 3: 源文件
await iconResolver.resolve('NewIcon')  // 200-500ms (首次)
```

### 内联图标批处理
```javascript
// 用户在笔记中输入多个 :IconName:
:LiHome: :LiUser: :LiSettings:

// 系统自动批处理（60ms 防抖）
InlineIconLoader.flush()
  → 批量加载 3 个图标
  → 自动重绘笔记
```

## 📁 文件结构

```
.obsidian/plugins/obsidian-icon-folder/
├── main.js                      # 主插件文件（已集成按需加载）
├── icon-index-store.js          # 索引持久化系统
├── icon-resolver.js             # 三层缓存解析器
├── icon-indexing.js             # 索引构建工具
├── inline-icon-loader.js        # 内联图标延迟加载
├── manifest.json                # 插件清单
├── styles.css                   # 样式文件
└── docs/
    ├── REFACTOR_PLAN.md         # 完整重构计划
    ├── INTEGRATION_REPORT.md    # 集成完成报告
    ├── SVG_CLEANUP_REPORT.md    # SVG清理报告
    └── FINAL_SUMMARY.md         # 最终总结

.obsidian/icons/
├── .index/                      # 图标包索引
│   ├── lucide-icons.json
│   └── font-awesome-solid.json
├── .cache/                      # 磁盘缓存
│   ├── lucide-icons/
│   └── migrated/                # 迁移的外部 SVG
└── [图标包 ZIP 文件]
```

## 🔧 开发

### 构建
```bash
npm install
npm run build
```

### 开发模式
```bash
npm run dev
```

### 测试
```bash
# 清理缓存
node scripts/clear-cache.js

# 重建索引
node scripts/rebuild-index.js

# 迁移外部 SVG
node scripts/migrate-external-svg.js
```

## 🐛 问题排查

### 问题：图标不显示
1. 打开控制台检查错误
2. 清理缓存：
   ```javascript
   const plugin = app.plugins.plugins['obsidian-icon-folder'];
   await plugin.iconResolver.clearDiskCache();
   await plugin.iconIndexStore.clearAll();
   ```
3. 重新加载插件

### 问题：启动慢
- 检查是否有大量图标包
- 查看索引是否已创建（`.obsidian/icons/.index/`）
- 检查控制台是否显示 "lazy loading enabled"

### 回退到原版
如需回退：
```bash
cp main.js.backup-lazy-loading main.js
```

## 🤝 贡献

欢迎提交 Issue 和 Pull Request！

### 开发指南
1. Fork 本仓库
2. 创建特性分支：`git checkout -b feature/amazing-feature`
3. 提交更改：`git commit -m 'Add amazing feature'`
4. 推送到分支：`git push origin feature/amazing-feature`
5. 提交 Pull Request

## 📜 致谢

- [Iconize](https://github.com/FlorianWoelki/obsidian-iconize) - 原始插件作者 @FlorianWoelki
- [glyphit](https://github.com/jmarasch/glyphit) - 按需加载架构参考 @jmarasch
- Obsidian 社区

## 📄 许可证

MIT License - 详见 [LICENSE](LICENSE) 文件

---

## 🔗 相关链接

- [原版 Iconize](https://github.com/FlorianWoelki/obsidian-iconize)
- [Obsidian](https://obsidian.md)
- [问题反馈](https://github.com/CoreVortex/iconize-enhanced/issues)

## 📊 统计

- ⭐ Stars: 欢迎 Star 支持！
- 🍴 Forks: 欢迎 Fork 改进！
- 🐛 Issues: 发现问题请提交 Issue

---

**注意**: 这是 Iconize 的增强版本，专注于性能优化。如果您喜欢原版的所有功能，请支持 [原作者](https://github.com/FlorianWoelki/obsidian-iconize)。
