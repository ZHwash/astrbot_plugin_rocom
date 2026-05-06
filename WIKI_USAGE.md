# Wiki 本地查询功能使用说明

## 功能概述

本插件现已支持使用本地数据库查询洛克王国的精灵和技能信息，并以精美的图片格式返回结果。

## 可用指令

### 1. `/洛克wiki <精灵名>`

查询精灵的详细信息，包括：
- 基本信息（编号、名称、属性、形态）
- 种族值（HP、攻击、魔攻、防御、魔抗、速度）
- 特性与属性克制关系
- **进化路线**（包含每个阶段的图片和编号）
- **技能列表**（包含完整的技能详情：属性、类别、威力、PP、效果）

**示例：**
```
/洛克wiki 迪莫
/洛克wiki 火花
/洛克wiki 水蓝蓝
```

### 2. `/洛克技能 <技能名>`

查询技能的详细信息，包括：
- 技能名称
- 属性类型
- 技能类别（物理/魔法/变化）
- 威力
- PP 消耗
- 技能效果描述

**示例：**
```
/洛克技能 圣光斩
/洛克技能 烈焰冲锋
/洛克技能 催眠粉
```

## 功能特性

### 智能匹配
- **精确匹配优先**：如果输入的精灵名/技能名完全匹配，直接显示结果
- **模糊搜索**：支持部分匹配，自动查找相关结果
- **多结果选择**：当有多个匹配时，会列出候选项供你选择

### 精美渲染
- 深度还原 WeGame 视觉风格
- 自适应宽度的高质量图片
- 完整的属性克制、进化链等信息展示

### 数据来源
- 数据来自 BiliGame 洛克王国 WIKI (https://wiki.biligame.com/rocom/)
- 遵循 CC BY-NC-SA 4.0 协议

## 配置说明

在插件配置中可以调整以下参数：

- `wiki_db_path`: 数据库文件路径（默认: `wiki-local.db`）
- `wiki_search_limit`: 搜索结果数量限制（默认: 5）
- `wiki_enable_fuzzy_search`: 是否启用模糊搜索（默认: true）

## 注意事项

1. **首次使用前**：确保已运行过 wiki 数据爬取脚本，生成了 `wiki-local.db` 数据库文件
2. **图片资源**：确保 `wiki/output/images/` 目录下有完整的精灵和技能图片
3. **路径配置**：如果移动了插件目录，需要更新数据库路径配置

## 故障排除

### 问题：提示 "Wiki 数据库服务未初始化"
**解决**：检查 `wiki-local.db` 文件是否存在，路径配置是否正确

### 问题：图片显示为默认图标
**解决**：检查 `wiki/output/images/pets/` 和 `wiki/output/images/skills/` 目录下是否有对应的图片文件

### 问题：查询结果为空
**解决**：
- 尝试使用更准确的精灵名/技能名
- 检查数据库中是否有该数据
- 可以运行 `wiki/src/build_wiki_db.py` 重新构建数据库

### 问题：进化路线缺少图片或编号
**已修复**：现在会自动为进化链中的每个阶段查询对应的宠物信息和图片

### 问题：技能列表为空或只有名称
**已修复**：现在会自动查询每个技能的详细信息（属性、类别、威力、PP、效果）

## 技术实现

### 数据适配层改进

#### 1. 进化路线处理
- 从 `evolution_stages` 中读取每个阶段的名称
- 为每个阶段调用 `db_service.get_pet_info()` 查询完整信息
- 获取每个阶段的 ID（作为 No. 编号）和 sprite_image_local（图片路径）
- 使用 `_resolve_wiki_path()` 正确解析图片路径

#### 2. 技能列表处理
- 从 [skills](file://h:\code\astrbot_plugin_rocom\wiki\src\db_service.py#L287-L287) 字段读取技能名称数组（JSON 格式）
- 对每个技能名称调用 `db_service.get_skill_info()` 查询详细信息
- 提取技能的 element（属性）、category（类别）、power（威力）、cost（PP）、effect（效果）
- 如果查询失败，使用技能名称作为占位符

#### 3. 图片路径处理
- 使用 [_resolve_wiki_path()](file://h:\code\astrbot_plugin_rocom\main.py#L3543-L3552) 方法解析相对路径
- 检查文件是否存在，不存在时使用默认图标
- Renderer 会自动将 `{{_res_path}}` 前缀的资源内联为 base64

### 使用的模板
- [render/pet-wiki/index.html](file://h:\code\astrbot_plugin_rocom\render\pet-wiki\index.html) - 宠物百科页面
- [render/skill-wiki/index.html](file://h:\code\astrbot_plugin_rocom\render\skill-wiki\index.html) - 技能百科页面

---

**版本**: v2.7.0  
**作者**: bvzrays & 熵增项目组  
**仓库**: https://github.com/Entropy-Increase-Team/astrbot_plugin_rocom
