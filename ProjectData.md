数据层（具体函数实现）

### E_CellType（枚举）

| 值 | 名称 | 含义 |
|----|------|------|
| 0 | Empty | 空地 |
| 1 | Wall  | 墙壁 |
| 2 | Target | 目标点 |

### S_LevelData（结构体）

| 字段 | 类型 | 说明 |
|------|------|------|
| GridText | FString | 关卡文本（多行，用特定字符表示格子类型） |
| StarRating | int32 | 星级数（1-7） |
| ParMoves | int32 | 标准步数 |

### 关卡文本格式

双图层模型，用单字符同时表示地板层和物件层：

| 字符 | 地板层 | 物件层 | 含义 |
|------|--------|--------|------|
| `-` | 空地 | 无 | 空地 |
| `#` | 墙壁 | — | 墙壁 |
| `.` | 目标 | 无 | 目标点 |
| `$` | 空地 | 箱子 | 箱子 |
| `*` | 目标 | 箱子 | 目标上的箱子 |
| `@` | 空地 | 玩家 | 玩家 |
| `+` | 目标 | 玩家 | 目标上的玩家 |

文本中换行符作为网格行分隔符。解析在 `BP_GridManager.InitLevel` 中完成。

### DT_Levels（数据表）

行结构 `S_LevelData`，7 行：`L1` ~ `L7`。打包时通过 `Additional Asset Directories to Cook` 确保包含。

---

## 蓝图详解

### GI_Sokoban（GameInstance）

**父类**：`GameInstance`  
**职责**：全局状态管理、存档读写、数据源切换

**变量**：

| 变量 | 类型 | 说明 |
|------|------|------|
| CurrentLevelIndex | int32 | 当前关卡索引 |
| PlayMode | int32 | 数据源模式：0=连续游戏、1=选关（正式）、2=选关（自定义）、3=试玩 |
| LevelStars | TArray\<int32\> | 每关星级（预留） |
| BestMoves | TArray\<int32\> | 每关最佳步数（预留） |
| CustomLevels | TArray\<S_LevelData\> | 自定义关卡列表（SaveGame 持久化） |
| TempLevelData | S_LevelData | 临时关卡数据（编辑器试玩用） |
| bIsSinglePlay | bool | 是否为单次试玩 |
| OfficialLevels | DataTable\* | 指向 DT_Levels（硬引用，确保打包包含） |

**函数**：

| 函数 | 功能 |
|------|------|
| `SaveProgress` | 创建 SG_Sokoban → 写入 CustomLevels → SaveGameToSlot("SokobanSave") |
| `LoadProgress` | LoadGameFromSlot → Cast SG_Sokoban → 恢复 CustomLevels 到 GI |

**事件**：
- `Event Init` → LoadProgress（启动时自动恢复存档）

---

### BP_GridManager（Actor）

**父类**：`Actor`  
**职责**：网格数据管理、关卡加载、碰撞检测、胜利判定

**变量**：

| 变量 | 类型 | 说明 |
|------|------|------|
| GridState | TMap\<IntPoint, E_CellType\> | 地板层地图（静态，关卡加载后不变） |
| BoxPositions | TArray\<IntPoint\> | 箱子位置列表 |
| TargetPositions | TArray\<IntPoint\> | 目标点位置列表 |
| WallPositions | TArray\<IntPoint\> | 墙壁位置列表 |
| PlayerStartPosition | IntPoint | 玩家起始位置 |
| BoxActors | TArray\<BP_Box_C\*\> | 箱子 Actor 引用 |
| WallActors | TArray\<BP_Wall_C\*\> | 墙壁 Actor 引用 |
| TargetActors | TArray\<BP_Target_C\*\> | 目标点 Actor 引用 |
| PlayerActor | BP_Player_C\* | 玩家 Actor 引用 |
| CellSize | float | 格子大小（默认 100） |
| MoveCount | int32 | 当前步数 |
| ParMoves | int32 | 当前关卡标准步数 |
| IsMoving | bool | 是否正在执行移动动画 |
| MoveHistory | TArray\<IntPoint\> | 移动历史（用于撤销） |

**函数**：

| 函数 | 输入 | 输出 | 功能 |
|------|------|------|------|
| `InitLevel` | GridText(FString) | — | 解析关卡文本 → 填充 GridState/位置数组 → 生成 Actor |
| `GetLevelData` | LevelIndex(int32) | GridText(FString) | 根据 PlayMode 从不同数据源读取关卡文本 |
| `CheckWin` | — | — | 遍历 BoxPositions，全部在 TargetPositions 中即为胜利 → 创建 W_Victory |
| `Reset` | — | — | 重新调用 InitLevel 重置关卡 |
| `GridToWorld` | GridX(int32), GridY(int32) | WorldPos(Vector) | 网格坐标转世界坐标 |
| `BoxMove` | DirX(int32), DirY(int32), PushedBox(BP_Box_C\*) | bBoxCanMove(bool), TargetPos(IntPoint) | 箱子推动逻辑与碰撞检测 |

**GetLevelData 数据源路由**：

| PlayMode | 数据源 | 说明 |
|----------|--------|------|
| 0 | DT_Levels[0] | 连续游戏，从第一关开始 |
| 1 | DT_Levels[CurrentLevelIndex] | 选关模式（正式关卡） |
| 2 | GI.CustomLevels[CurrentLevelIndex] | 选关模式（自定义关卡） |
| 3 | GI.TempLevelData | 编辑器试玩 |

---

### BP_Player（Character）

**父类**：`Character`  
**职责**：玩家输入、移动执行、碰撞检测、暂停触发

**变量**：

| 变量 | 类型 | 说明 |
|------|------|------|
| MoveSpeed | float | 移动速度 |
| BoxStartPos | Vector | 箱子推动起始位置 |
| GridManagerRef | BP_GridManager_C\* | 网格管理器引用（关卡中手动指向） |

**函数**：

| 函数 | 功能 |
|------|------|
| `PlayerMove` | WASD 输入处理 → 调用 BoxMove → 执行移动动画 |
| `GetPushDirection` | 从碰撞法线计算推动方向（DirX/DirY） |

**事件**：
- `EnhancedInputAction IA_Move`（Triggered）→ PlayerMove
- `EnhancedInputAction IA_Pause`（Triggered）→ 创建 W_Pause → 暂停游戏
- `事件命中` → GetPushDirection → 推箱子

---

### BP_Box（StaticMeshActor）

**父类**：`StaticMeshActor`

| 变量 | 类型 | 说明 |
|------|------|------|
| IsMoving | bool | 动画锁 |
| GridManagerRef | BP_GridManager_C\* | 网格管理器引用 |
| BoxStartPos | Vector | 动画起始位置 |

**功能**：接受推动指令，执行 Timeline 平移动画。InitLevel 中动态生成。

---

### BP_Wall / BP_Target

**BP_Wall**：StaticMeshActor，墙壁装饰，InitLevel 动态生成。  
**BP_Target**：StaticMeshActor，目标点标记，InitLevel 动态生成。

---

### BP_SokobanController（PlayerController）

**职责**：游戏启动时添加 IMC_Default 映射、设置输入模式为游戏模式、隐藏鼠标。

---

### SG_Sokoban（SaveGame）

**变量**：`CustomLevels`（TArray\<S_LevelData\>）

存档位置：`Saved/SaveGames/SokobanSave.sav`

---

## UI 界面

### W_MainMenu

| 按钮 | 功能 |
|------|------|
| 开始游戏 | Set PlayMode=0 → OpenLevel(L_GamePlay) |
| 关卡编辑器 | CreateWidget(W_LevelEditor) → AddToViewport |
| 关卡选择 | CreateWidget(W_LevelSelect) → AddToViewport |
| 退出游戏 | 退出应用程序 |

### W_LevelSelect

**流程**：
1. `Event Construct` → GetDataTableRowNames(DT_Levels) → ForEach 行名
2. 每个行名：CreateWidget(W_LevelCard) → SetupCard(Index, "关卡X", StarRating, bCustom=false) → AddChild(LevelContainer)
3. ForEach 完成后 → Get GI.CustomLevels → ForEach 自定义关卡
4. 每个自定义：CreateWidget(W_LevelCard) → SetupCard(Index, "自定义X", 0, bCustom=true) → AddChild

### W_LevelCard

**变量**：LevelIndex(int32)、IsCustom(bool)

**控件**：Btn_Card（主按钮）、Txt_LevelName、Txt_Stars、Btn_Delete（删除按钮，仅自定义显示）

**函数 `SetupCard(Index, DisplayName, Stars, bCustom)`**：
- 设 LevelIndex / IsCustom
- 设 Txt_LevelName = DisplayName
- 设 Txt_Stars = ★×Stars
- bCustom=true 时显示 Btn_Delete

**Btn_Card OnClicked**：
→ Cast GI → Set CurrentLevelIndex(LevelIndex) → Branch(IsCustom) → Set PlayMode(1正式/2自定义) → OpenLevel(L_GamePlay)

**Btn_Delete OnClicked**（仅自定义）：
→ Cast GI → Get CustomLevels → RemoveIndex(LevelIndex) → SaveProgress → RemoveFromParent

### W_Victory

**Event Construct**：
→ Cast GI → Get PlayMode → PlayMode=0 且 CurrentLevelIndex < DT 总行数时显示 Btn_NextLevel

**Btn_NextLevel OnClicked**：
→ Cast GI → CurrentLevelIndex+1 → OpenLevel(L_GamePlay)

**Btn_BackMenu OnClicked**：
→ OpenLevel(L_MainMenu)

### W_Pause

| 按钮 | 功能 |
|------|------|
| 继续 | RemoveFromParent + SetGamePaused(false) |
| 重置 | Reset → RemoveFromParent + SetGamePaused(false) |
| 返回主菜单 | OpenLevel(L_MainMenu) |
| 退出游戏 | 退出应用程序 |

### W_LevelEditor（关卡编辑器）

**变量**：

| 变量 | 类型 | 说明 |
|------|------|------|
| GridWidth / GridLength | int32 | 网格尺寸 |
| FloorLayer / ObjectLayer | TArray\<int32\> | 双层网格数据 |
| CurrentBrush | int32 | 当前画刷类型（0-4） |
| CellWidgets | TArray\<W_EditorCell_C\*\> | 单元格控件引用 |
| GridText | FString | 生成的关卡文本 |

**画刷按钮**：Btn_Wall(0)、Btn_Floor(1)、Btn_Target(2)、Btn_Box(3)、Btn_Player(4)

**函数 `GenerateGridText`**：
- 遍历 FloorLayer + ObjectLayer → 生成 GridText 字符串
- 同时写入 GI.TempLevelData

**主要事件**：
- `事件构造`：检查尺寸合法性 → ForLoop 生成网格 → 创建 W_EditorCell → 绑定 OnCellClicked
- `HandlePaint(X, Y)`：根据 CurrentBrush 修改 FloorLayer/ObjectLayer → SetCellColor
- `Btn_GenText` → GenerateGridText → 更新 Txt_XSB 预览
- `Btn_Save` → GenerateGridText → Cast GI → CustomLevels.Add → SaveProgress
- `Btn_TestPlay` → GenerateGridText → Set PlayMode=3 → OpenLevel(L_GamePlay)
- `Btn_BackEd` → OpenLevel(L_MainMenu)

### W_EditorCell

**变量**：CellX(int32)、CellY(int32)

**事件**：Btn_Cell OnClicked → Call OnCellClicked(X=CellX, Y=CellY)  
**函数**：`SetCellColor(ColorType)` → 根据类型切换按钮颜色

---

## 输入系统

使用 UE5 Enhanced Input 系统：

| 输入动作 | 按键 | 用途 |
|----------|------|------|
| IA_Move | WASD | 玩家四方向移动 |
| IA_Pause | Esc | 打开暂停菜单 |

BP_SokobanController 的事件开始运行时添加 IMC_Default 映射上下文。

---

## 存档系统

- **存储**：SG_Sokoban（SaveGame 子类），仅存 CustomLevels
- **写盘**：GI.SaveProgress → CreateSaveGameObject(SG_Sokoban) → 写入 → SaveGameToSlot("SokobanSave", 0)
- **读盘**：GI.EventInit → LoadProgress → LoadGameFromSlot → 恢复 CustomLevels
- **触发时机**：编辑器保存关卡、关卡选择中删除自定义关卡