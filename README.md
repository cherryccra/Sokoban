# 推箱子（Sokoban）


## 项目实现
   项目采用UE 5.7.4 蓝图实现 ，主要功能有：
   ①推箱子关卡的正常游玩体验（将箱子推入目标地点），基于网格系统和UE5碰撞系统实现。
   ②基于标准XSB格式的自定义编辑器功能，允许玩家自定义地图并进行保存
   ③存档与关卡选择，允许玩家选择游戏自带关卡/自定义关卡进行游玩，允许玩家保存当前游戏进度与继续游戏。
   ④基本的UI交互系统，包括主菜单，关卡编辑菜单，关卡选择与管理菜单，游戏内暂停菜单等。
   
   关于项目每个蓝图的具体结构，可见项目工程/Game/Sokoban部分，主要函数说明见ProjectData.md.

## 项目结构

```
Content/Sokoban/
├─ Core/                        # 核心逻辑蓝图
│  ├─ GI_Sokoban                → 游戏实例，保存关卡index，customlevel等核心数据
│  ├─ BP_GridManager            → 网格管理器，用于管理一关内的Actor生成和网格内容
│  ├─ BP_Player                 → 玩家角色，采用俯视视角，WASD操控移动，ESC控制游戏暂停
│  ├─ BP_Box                    → 箱子类型蓝图，纯黄色，通过与玩家的hit事件实现移动
│  ├─ BP_Wall                   → 墙类型蓝图，无色
│  ├─ BP_Target                 → 目标类型蓝图，绿色
│  ├─ BP_SokobanController      → 玩家控制器，主要控制玩家的输入，玩家与箱子间的碰撞等
│  ├─ BP_SokobanGameMode        → 默认游戏模式
│  ├─ SG_Sokoban                → 游戏存档蓝图
│  ├─ E_CellType                → 地面变量类型枚举
│  └─ S_LevelData               → 关卡数据结构体，包含关卡信息，关卡星级，最佳步数等
│ 
├─ UI/                          # 界面蓝图
│  ├─ W_MainMenu                → 主菜单，包括简单的功能分类按钮以及其事件逻辑
│  ├─ W_LevelSelect             → 关卡选择界面，其由多个关卡卡片串联成的索引集组成
│  ├─ W_LevelCard               → 关卡卡片，分自定义/非自定义关卡，前者可删除
│  ├─ W_LevelEditor             → 关卡编辑器，包含自定义编辑系统（默认8x8大小），五类笔刷
│  ├─ W_EditorCell              → 编辑器单元格，可变色
│  ├─ W_Victory                 → 胜利界面，包含下一关判定与关卡index控制
│  └─ W_Pause                   → 暂停菜单，主要包含继续，返回等常规按钮与重置游戏功能
│ 
├─ Maps/                        # 关卡
│  ├─ L_MainMenu                → 主菜单关卡
│  └─ L_GamePlay                → 游戏内关卡
│ 
├─ Input/                       # 输入
│  ├─ IMC_Default               → 输入映射上下文
│  └─ Actions/
│     ├─ IA_Move                → 移动（WASD）
│     └─ IA_Pause               → 暂停（默认esc）
│ 
└─ Data/
   └─ DT_Levels                 → 数据表（提供7关配置，含星级）
```
