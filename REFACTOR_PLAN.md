# Iconize 图标加载机制重构计划

## 问题诊断

### 当前架构缺陷
1. **全量加载策略**：启动时将所有图标包的所有图标解析并加载到内存 (`iconPacks` 数组)
2. **外部 SVG 存储不稳定**：部分图标存储在 `.obsidian/icons/` 目录下的独立 SVG 文件，容易丢失
3. **无按需加载机制**：即使只使用 10 个图标，也会加载数千个图标到内存
4. **缺乏索引系统**：每次查找图标都需要遍历 `iconPacks` 数组
5. **无缓存分层**：已解析的图标和未解析的图标混在一起管理

### 现有代码关键问题点
- **Line 405**: `let iconPacks = []` - 全局数组存储所有图标包
- **Line 398**: `let preloadedIcons = []` - 预加载图标列表，但机制不完善
- **Line 805-836**: `getIconFromIconPack()` 和 `getSvgFromLoadedIcon()` - 线性遍历查找
- **Line 4126/4163**: `addIconToIconPack()` - 动态添加图标时直接修改全局数组

## Glyphit 架构优势分析

### 核心设计模式
1. **Index-First 架构**：启动时只加载图标索引 (元数据)，不解析 SVG
2. **三层缓存系统**：
   - **内存层 (Resolver)**: 已解析的 Icon 对象
   - **磁盘缓存层 (IconCacheStore)**: 已处理的 SVG 文件
   - **源层 (ZipSource/FolderSource)**: 压缩包或文件夹
3. **按需解析 (Lazy Loading)**：
   - `peekIcon()`: 同步查询已加载的图标
   - `resolveIcon()`: 异步按需加载未缓存的图标
   - `prefetch()`: 启动时仅预加载实际使用的图标
4. **分离式管理**：
   - `IconPackManager`: 统一管理所有图标包
   - `IndexStore`: 持久化索引到 JSON
   - `IconCacheStore`: 管理磁盘缓存
   - `IconResolver`: 处理图标解析和内存缓存

### 关键实现细节
```typescript
// 启动流程 (icon-pack-manager/index.ts:92-151)
async init() {
  // 1. 扫描 .zip 文件和目录
  // 2. 为每个图标包创建 IconPack 对象
  // 3. 加载或重建索引 (仅元数据，不解析 SVG)
  // 4. 释放源文件句柄
}

// 按需加载 (icon-pack-manager/index.ts:413-424)
async resolveIcon(iconName, color, options) {
  const located = this.findEntry(iconName); // O(1) 索引查找
  if (!located) return null;
  return this.resolver.resolve(request, options); // 缓存未命中时才解析
}

// 启动预加载 (lib/icon.ts:68-110)
async checkMissingIcons(plugin, data) {
  const referenced = getReferencedIcons(plugin, data); // 扫描实际使用的图标
  for (const iconName of referenced) {
    if (!iconPackManager.peekIcon(iconName)) {
      await iconPackManager.resolveIcon(iconName); // 仅加载使用的图标
    }
  }
}
```

## 重构方案设计

### 阶段一：建立索引系统 (不破坏现有功能)

#### 1.1 创建 IconIndexStore 类
```javascript
/**
 * 图标索引存储 - 持久化到 .obsidian/icons/.index/
 * Persistent icon index storage
 */
class IconIndexStore {
  constructor(adapter, basePath) {
    this.adapter = adapter;
    this.indexPath = `${basePath}/.index`;
  }

  /**
   * 保存图标包索引
   * Save icon pack index
   */
  async save(packName, index) {
    const path = `${this.indexPath}/${packName}.json`;
    await this.adapter.write(path, JSON.stringify(index));
  }

  /**
   * 加载图标包索引
   * Load icon pack index
   */
  async load(packName) {
    const path = `${this.indexPath}/${packName}.json`;
    if (!await this.adapter.exists(path)) return null;
    const content = await this.adapter.read(path);
    return JSON.parse(content);
  }
}
```

#### 1.2 创建轻量级索引结构
```javascript
/**
 * 图标包索引结构 (仅元数据)
 * Icon pack index structure (metadata only)
 */
const IconPackIndex = {
  name: 'lucide-icons',
  prefix: 'Li',
  fingerprint: 'zip-hash-or-mtime', // 用于检测变更
  entries: [
    { name: 'Home', filename: 'home.svg', id: 'LiHome' },
    { name: 'User', filename: 'user.svg', id: 'LiUser' }
    // 不包含 svgElement/svgContent，启动时不解析
  ]
};
```

#### 1.3 修改图标包初始化流程
```javascript
/**
 * 重构后的初始化流程
 * Refactored initialization flow
 */
async function initializeIconPacks(plugin) {
  const iconPacksPath = plugin.getSettings().iconPacksPath;
  const indexStore = new IconIndexStore(plugin.app.vault.adapter, iconPacksPath);
  
  const listing = await plugin.app.vault.adapter.list(iconPacksPath);
  
  for (const zipFile of listing.files.filter(f => f.endsWith('.zip'))) {
    const packName = zipFile.replace(/\.zip$/, '');
    
    // 尝试加载已有索引
    let index = await indexStore.load(packName);
    
    // 检查索引是否过期
    const fingerprint = await getZipFingerprint(plugin, zipFile);
    if (!index || index.fingerprint !== fingerprint) {
      // 重建索引（仅扫描文件名，不解析 SVG）
      index = await buildLightweightIndex(plugin, packName, zipFile);
      await indexStore.save(packName, index);
    }
    
    // 注册到全局索引，但不加载图标内容
    iconPacks.push({
      name: packName,
      prefix: index.prefix,
      index: index, // 仅元数据
      source: { type: 'zip', path: zipFile } // 源文件路径，按需打开
    });
  }
}
```

### 阶段二：实现按需加载机制

#### 2.1 创建 IconResolver 类
```javascript
/**
 * 图标解析器 - 三层缓存架构
 * Icon resolver with three-tier cache
 */
class IconResolver {
  constructor(plugin) {
    this.plugin = plugin;
    this.memoryCache = new Map(); // 已解析的图标对象
    this.diskCachePath = `${plugin.getSettings().iconPacksPath}/.cache`;
  }

  /**
   * 同步查询 - 仅返回已加载到内存的图标
   * Synchronous peek - returns only memory-cached icons
   */
  peek(iconId) {
    return this.memoryCache.get(iconId);
  }

  /**
   * 异步解析 - 按需加载
   * Async resolve - load on demand
   */
  async resolve(iconId, options = {}) {
    // 1. 内存缓存命中
    if (this.memoryCache.has(iconId)) {
      return this.memoryCache.get(iconId);
    }

    // 2. 查找索引条目
    const entry = this.findEntry(iconId);
    if (!entry) return null;

    // 3. 尝试从磁盘缓存加载
    const cacheKey = `${entry.packName}/${entry.filename}`;
    const cached = await this.loadFromDiskCache(cacheKey);
    if (cached) {
      this.memoryCache.set(iconId, cached);
      return cached;
    }

    // 4. 从源文件解析（压缩包或文件夹）
    const svgContent = await this.extractFromSource(entry);
    const icon = {
      name: entry.name,
      prefix: entry.prefix,
      svgElement: extractSvg(svgContent), // 现有的 SVG 处理逻辑
      iconPackName: entry.packName
    };

    // 5. 写入缓存
    if (options.persist !== false) {
      await this.saveToDiskCache(cacheKey, icon);
    }
    this.memoryCache.set(iconId, icon);

    return icon;
  }

  /**
   * 从索引查找条目 (O(1) 查找)
   * Find entry from index (O(1) lookup)
   */
  findEntry(iconId) {
    const split = nextIdentifier(iconId);
    const prefix = iconId.substring(0, split);
    
    const pack = iconPacks.find(p => p.prefix === prefix);
    if (!pack) return null;

    const entry = pack.index.entries.find(e => e.id === iconId);
    return entry ? { ...entry, packName: pack.name } : null;
  }

  /**
   * 从源文件提取 SVG
   * Extract SVG from source
   */
  async extractFromSource(entry) {
    const pack = iconPacks.find(p => p.name === entry.packName);
    if (!pack) return null;

    if (pack.source.type === 'zip') {
      // 使用现有的 ZIP 解析逻辑
      const zipContent = await this.plugin.app.vault.adapter.readBinary(pack.source.path);
      const zip = await JSZip.loadAsync(zipContent);
      const file = zip.file(entry.filename);
      return file ? await file.async('text') : null;
    } else {
      // 文件夹模式
      const filePath = `${pack.source.path}/${entry.filename}`;
      return await this.plugin.app.vault.adapter.read(filePath);
    }
  }

  /**
   * 磁盘缓存加载
   * Load from disk cache
   */
  async loadFromDiskCache(cacheKey) {
    const path = `${this.diskCachePath}/${cacheKey}`;
    if (!await this.plugin.app.vault.adapter.exists(path)) {
      return null;
    }
    const content = await this.plugin.app.vault.adapter.read(path);
    return JSON.parse(content);
  }

  /**
   * 磁盘缓存保存
   * Save to disk cache
   */
  async saveToDiskCache(cacheKey, icon) {
    const path = `${this.diskCachePath}/${cacheKey}`;
    const dir = path.substring(0, path.lastIndexOf('/'));
    
    // 确保目录存在
    if (!await this.plugin.app.vault.adapter.exists(dir)) {
      await this.plugin.app.vault.adapter.mkdir(dir);
    }
    
    await this.plugin.app.vault.adapter.write(path, JSON.stringify(icon));
  }
}
```

#### 2.2 启动时预加载实际使用的图标
```javascript
/**
 * 启动预加载 - 仅加载 data.json 中引用的图标
 * Startup prefetch - load only referenced icons
 */
async function prefetchUsedIcons(plugin) {
  const resolver = plugin.iconResolver;
  const data = plugin.getData();
  const referencedIcons = new Set();

  // 1. 收集所有引用的图标
  Object.values(data).forEach(value => {
    if (typeof value === 'string' && value.startsWith(':') === false) {
      referencedIcons.add(value);
    } else if (typeof value === 'object' && value.iconName) {
      referencedIcons.add(value.iconName);
    }
  });

  // 2. 收集规则中的图标
  plugin.getSettings().rules.forEach(rule => {
    if (rule.icon) referencedIcons.add(rule.icon);
  });

  // 3. 批量预加载
  const missing = [];
  for (const iconId of referencedIcons) {
    try {
      const icon = await resolver.resolve(iconId);
      if (!icon) missing.push(iconId);
    } catch (error) {
      console.error(`[Iconize] Failed to prefetch icon ${iconId}:`, error);
      missing.push(iconId);
    }
  }

  if (missing.length > 0) {
    new Notice(`[Iconize] ${missing.length} icons could not be loaded`, 5000);
  }
}
```

#### 2.3 内联图标的延迟加载
```javascript
/**
 * 内联图标加载器 - 用于笔记中的 :IconName: 语法
 * Inline icon loader - for :IconName: syntax in notes
 */
class InlineIconLoader {
  constructor(plugin) {
    this.plugin = plugin;
    this.pending = new Set();
    this.failed = new Set();
    this.timer = null;
  }

  /**
   * 请求图标（非阻塞）
   * Request icon (non-blocking)
   */
  requestIcon(iconId) {
    if (this.failed.has(iconId) || this.pending.has(iconId)) {
      return;
    }

    // 已加载
    if (this.plugin.iconResolver.peek(iconId)) {
      return;
    }

    this.pending.add(iconId);
    this.schedule();
  }

  /**
   * 批量处理延迟加载
   * Batch process deferred loading
   */
  schedule() {
    if (this.timer) return;
    
    this.timer = setTimeout(async () => {
      this.timer = null;
      await this.flush();
    }, 60); // 60ms 批处理延迟
  }

  async flush() {
    if (this.pending.size === 0) return;

    const icons = [...this.pending];
    this.pending.clear();

    let loaded = 0;
    for (const iconId of icons) {
      try {
        const icon = await this.plugin.iconResolver.resolve(iconId, { persist: true });
        if (icon) {
          loaded++;
        } else {
          this.failed.add(iconId);
        }
      } catch (error) {
        console.error(`[Iconize] Failed to load inline icon ${iconId}:`, error);
        this.failed.add(iconId);
      }
    }

    if (loaded > 0) {
      // 重新渲染打开的笔记
      this.plugin.app.workspace.getLeavesOfType('markdown').forEach(leaf => {
        if (leaf.view.previewMode) {
          leaf.view.previewMode.rerender(true);
        }
      });
    }
  }
}
```

### 阶段三：废弃外部 SVG 存储

#### 3.1 迁移策略
```javascript
/**
 * 将外部 SVG 文件迁移到 ZIP 缓存
 * Migrate external SVG files to ZIP cache
 */
async function migrateExternalSvgFiles(plugin) {
  const iconsPath = `${plugin.getSettings().iconPacksPath}`;
  const files = await plugin.app.vault.adapter.list(iconsPath);
  
  const svgFiles = files.files.filter(f => f.endsWith('.svg') && !f.includes('/.'));
  
  if (svgFiles.length === 0) return;

  console.log(`[Iconize] Migrating ${svgFiles.length} external SVG files...`);

  for (const svgFile of svgFiles) {
    try {
      const content = await plugin.app.vault.adapter.read(svgFile);
      const filename = svgFile.split('/').pop();
      const iconName = filename.replace('.svg', '');
      
      // 解析图标前缀（如 fab_odnoklassniki.svg -> Fab）
      const prefix = extractPrefixFromFilename(filename);
      const iconId = `${prefix}${capitalize(iconName.replace(/_/g, ''))}`;
      
      // 保存到磁盘缓存
      await plugin.iconResolver.saveToDiskCache(`migrated/${filename}`, {
        name: iconName,
        prefix: prefix,
        svgElement: extractSvg(content),
        iconPackName: 'migrated'
      });

      // 删除原文件
      await plugin.app.vault.adapter.remove(svgFile);
    } catch (error) {
      console.error(`[Iconize] Failed to migrate ${svgFile}:`, error);
    }
  }

  new Notice('[Iconize] External SVG files migrated to cache', 3000);
}
```

### 阶段四：优化查找性能

#### 4.1 创建前缀索引
```javascript
/**
 * 前缀到图标包的映射 (O(1) 查找)
 * Prefix-to-pack mapping (O(1) lookup)
 */
const prefixIndex = new Map();

function buildPrefixIndex() {
  prefixIndex.clear();
  iconPacks.forEach(pack => {
    prefixIndex.set(pack.prefix, pack);
  });
}

function getIconPackByPrefix(prefix) {
  return prefixIndex.get(prefix);
}
```

#### 4.2 优化 getIconFromIconPack 函数
```javascript
/**
 * 重构后的图标查找 (使用索引，O(1) 复杂度)
 * Refactored icon lookup (indexed, O(1) complexity)
 */
const getIconFromIconPack = (iconPackName, iconPrefix, iconName) => {
  // 1. 先检查内存缓存
  const iconId = `${iconPrefix}${iconName}`;
  const cached = iconResolver.peek(iconId);
  if (cached) return cached;

  // 2. 通过前缀快速定位图标包
  const pack = prefixIndex.get(iconPrefix);
  if (!pack || pack.name !== iconPackName) {
    return undefined;
  }

  // 3. 在索引中查找（不触发加载）
  const entry = pack.index.entries.find(e => e.id === iconId);
  return entry ? { ...entry, pending: true } : undefined; // 标记为待加载
};
```

## 影响评估

### 不会破坏的功能
- ✅ 现有的图标显示逻辑 (文件浏览器、标签页、标题等)
- ✅ data.json 中的图标引用
- ✅ 自定义规则匹配
- ✅ 图标拾取器 (IconPickerModal)
- ✅ 图标下载和安装流程

### 需要适配的功能
- 🔄 图标拾取器需要支持异步加载预览 (目前是同步渲染)
- 🔄 `getAllLoadedIconNames()` 需要改为 `getAllIndexedIconNames()`
- 🔄 图标包设置页面需要显示"已索引但未加载"的状态

### 性能提升预期
| 指标 | 当前 (全量加载) | 重构后 (按需加载) | 改善 |
|------|----------------|------------------|------|
| 启动时间 | ~3-5s (加载所有图标) | ~200-500ms (仅加载索引) | **90% ↓** |
| 内存占用 | ~50-100MB (1000+ 图标) | ~5-10MB (仅使用的图标) | **80% ↓** |
| 首次图标显示 | 即时 (已全部加载) | 50-150ms (磁盘缓存) | 略慢 |
| 后续图标显示 | 即时 | 即时 (内存缓存) | 相同 |

## 实施步骤

### Week 1: 基础架构
1. ✅ 分析 glyphit 源码，理解架构设计
2. ⏳ 创建 `IconIndexStore` 类
3. ⏳ 实现轻量级索引构建逻辑
4. ⏳ 编写索引持久化和加载代码

### Week 2: 解析器实现
1. ⏳ 创建 `IconResolver` 类
2. ⏳ 实现三层缓存逻辑
3. ⏳ 实现 `peek()` 和 `resolve()` 方法
4. ⏳ 编写单元测试

### Week 3: 集成和迁移
1. ⏳ 重构 `initializeIconPacks()` 流程
2. ⏳ 实现 `prefetchUsedIcons()` 启动预加载
3. ⏳ 创建 `InlineIconLoader` 延迟加载器
4. ⏳ 实现外部 SVG 文件迁移逻辑

### Week 4: 优化和测试
1. ⏳ 构建前缀索引优化查找
2. ⏳ 适配图标拾取器异步加载
3. ⏳ 全面测试各场景
4. ⏳ 性能基准测试和调优

## 风险控制

### 回退机制
- 保留旧版本 `main.js` 作为 `main.js.backup-lazy-loading`
- 提供配置选项切换加载模式：`"iconLoadingStrategy": "eager" | "lazy"`
- 启动时检测索引损坏，自动回退到全量加载

### 数据安全
- 迁移外部 SVG 前先备份到 `.obsidian/icons/.backup/`
- 索引构建失败时不删除原有数据
- 磁盘缓存写入失败不影响内存缓存使用

### 兼容性
- 保持 `iconPacks` 全局变量存在（改为索引对象）
- `getIconFromIconPack()` 函数签名不变，内部改为按需加载
- data.json 格式完全兼容，无需迁移

## 参考文档
- Glyphit 源码: `/tmp/glyphit/src/icon-pack-manager/`
- Iconize 原始代码: `.obsidian/plugins/obsidian-icon-folder/main.js`
- Obsidian API: https://github.com/obsidianmd/obsidian-api

---

**最后更新**: 2026-09-13  
**状态**: 计划阶段 - 等待用户审批
