# 宠物图片资源获取指南

## 问题

班级宠物园网站 (kuketang.cn/pet-garden/) 有访问限制，无法直接爬取宠物图片资源。

---

## 替代方案

### 方案 1: 免费图标库

#### Iconify (推荐)
- 网址: https://icon-sets.iconify.design/
- 搜索: cat, dog, rabbit, hamster, bird, fish
- 格式: SVG (可直接使用)
- 许可: 多数免费

#### Flaticon
- 网址: https://www.flaticon.com/packs/pet-icons
- 格式: PNG, SVG
- 许可: 免费需署名

#### Icons8
- 网址: https://icons8.com/icons/set/pet
- 格式: PNG, SVG
- 许可: 免费版有限制

---

### 方案 2: AI 生成

使用 DALL-E / Midjourney / Stable Diffusion 生成可爱的宠物图片：

```
提示词示例:
- "cute cartoon cat icon, simple design, kawaii style, white background"
- "adorable pixel art dog character, game asset, 64x64"
- "cute rabbit mascot logo, flat design, minimal"
```

---

### 方案 3: 开源游戏资源

#### OpenGameArt
- 网址: https://opengameart.org/art-search?keys=pet
- 许可: CC0 / CC-BY

#### itch.io
- 网址: https://itch.io/game-assets/free/tag-pets
- 格式: PNG, Sprite sheets
- 许可: 多数免费

#### Kenney Assets
- 网址: https://kenney.nl/assets
- 搜索: animals, pets
- 许可: CC0 (完全免费)

---

### 方案 4: 自己绘制 SVG

创建简单的卡通宠物 SVG：

```svg
<!-- 猫咪示例 -->
<svg viewBox="0 0 100 100">
  <circle cx="50" cy="55" r="35" fill="#FFB366"/>
  <!-- 耳朵 -->
  <polygon points="25,35 35,10 45,35" fill="#FFB366"/>
  <polygon points="55,35 65,10 75,35" fill="#FFB366"/>
  <!-- 眼睛 -->
  <circle cx="38" cy="50" r="5" fill="#333"/>
  <circle cx="62" cy="50" r="5" fill="#333"/>
  <!-- 鼻子 -->
  <ellipse cx="50" cy="60" rx="4" ry="3" fill="#FF6B6B"/>
  <!-- 嘴巴 -->
  <path d="M 40 68 Q 50 75 60 68" stroke="#333" fill="none" stroke-width="2"/>
</svg>
```

---

## 推荐的宠物类型

根据常见电子宠物游戏，建议准备以下类型：

| 类型 | 变体 | 数量 |
|------|------|------|
| 🐱 猫 | 不同颜色/品种 | 5-10 |
| 🐕 狗 | 不同品种 | 5-10 |
| 🐰 兔子 | 不同颜色 | 3-5 |
| 🐹 仓鼠 | 不同颜色 | 3-5 |
| 🐦 鸟 | 不同品种 | 3-5 |
| 🐟 鱼 | 不同颜色 | 3-5 |
| 🐉 龙 | 神话宠物 | 1-3 |

每种宠物还需要不同等级/状态的变化：
- 正常状态
- 开心状态
- 饥饿状态
- 生病状态
- 睡觉状态

---

## 快速开始: 使用 Emoji

最简单的方案是使用 Emoji 作为宠物图标：

```javascript
const PET_EMOJIS = {
  cat: ['🐱', '😺', '😸', '😹', '😻', '😼', '😽', '🙀', '😿', '😾'],
  dog: ['🐶', '🐕', '🦮', '🐕‍🦺', '🐩', '🐾'],
  rabbit: ['🐰', '🐇'],
  hamster: ['🐹'],
  bird: ['🐦', '🐧', '🦆', '🦅', '🦉', '🦜'],
  fish: ['🐟', '🐠', '🐡', '🦈'],
  dragon: ['🐉', '🦎'],
  fantasy: ['🦄', '🐲', '🦋', '🦊']
};
```

---

## 下一步

1. 确定需要的宠物类型和数量
2. 选择一个资源获取方案
3. 下载/创建图片资源
4. 整理到项目的 `/public/images/pets/` 目录

需要我帮你获取特定类型的图片资源吗？