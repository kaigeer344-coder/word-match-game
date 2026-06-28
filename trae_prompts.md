# Trae 开发提示词：从「闪卡记忆」到「单词消消乐」

> 使用方式：按顺序把每个 Prompt 复制进 Trae IDE 的新会话，一次只执行一步。每步完成后保存文件、本地测试、截图，并记下 Session ID。

---

## 第 1 步：添加核心消除游戏界面

**目标**：在现有闪卡学习流程结束后，进入单词配对消除游戏。

请基于 `闪卡记忆_base.html` 进行如下修改：

1. **流程改造**
   - 当前流程：首页 → 学习 10 张闪卡 → 完成页
   - 新流程：首页 → 学习 10 张闪卡 → **消除游戏页（gameScreen）**
   - 当 `markWord()` 中 `state.learningIndex >= state.currentWords.length` 时，不再调用 `showCompletionSummary()`，改为调用 `startGame()`
   - 暂时保留 `completionScreen`，但先不显示，后续步骤会把它改造成结算页

2. **新增 gameScreen 结构**
   - 在 HTML 中 `learningScreen` 后面新增一个 `div.screen.game-screen`，id 为 `gameScreen`
   - 顶部信息栏：
     - 左侧：倒计时 `timer`，显示 `<span id="timeLeft">60</span>s`
     - 中间：连击显示 `comboDisplay`，默认隐藏，有连击时显示 `xN 连击`
     - 右侧：得分 `score-display`，显示 `<span id="gameScore">0</span> 分`
   - 顶部加一个 `timer-bar` 进度条，`timer-bar-fill` 宽度随时间减少
   - 中间是 `game-board` 区域，用 CSS Grid 布局
   - 底部暂时不放任何按钮

3. **游戏板生成逻辑**
   - 新增 `startGame()` 函数：
     - 调用 `showScreen('gameScreen')`
     - 设置 `state.gameActive = true`、`state.gameScore = 0`、`state.combo = 0`、`state.maxCombo = 0`、`state.correct = 0`、`state.wrong = 0`
     - 设置 `state.timeLeft = 60`
     - 用 `state.currentWords` 生成 20 个 tile：每个词拆成英文 tile 和中文 tile
     - 打乱顺序后渲染到 `game-board`
   - 新增 `renderBoard()` 函数：
     - 每个 tile 是一个 `div.tile`，data-id 绑定 wordId
     - 英文 tile 显示单词，`data-type="en"`
     - 中文 tile 显示释义，`data-type="cn"`
     - 所有 tile 初始背面朝上或显示内容但灰色状态（设计自选，建议初始显示文字、可点击）
   - 新增 `selectTile(tile)` 函数：
     - 记录第一次选中的 tile
     - 选中第二个 tile 时判断是否匹配（data-id 相同且 type 不同）
     - 匹配：两个 tile 添加 `matched` 类并消失（scale-out 动画），增加得分
     - 不匹配：两个 tile 添加 `shake` 动画后移除选中状态

4. **视觉与动画**
   - `.tile` 样式：白色背景、2px 浅灰边框、圆角 12px、最小高度 70px、flex 居中、文字 14px 粗体
   - `.tile.selected`：边框变绿色 `#58cc02`、背景 `#f3ffec`、轻微上移
   - `.tile.matched`：scale 从 1 到 0 的消失动画，300ms
   - `.tile.shake`：左右抖动动画，300ms
   - `.game-screen` 整体保持绿色主题，padding 20px

5. **音效**
   - 使用 Web Audio API 生成简单音效：
     - `select`：短促 600Hz → 800Hz 上升音，0.1s
     - `correct`：800Hz 清脆短音 + 1100Hz 更高短音，愉悦感
     - `wrong`：300Hz 下降音，0.15s
   - 在对应时机调用 `SFX.select()`、`SFX.correct()`、`SFX.wrong()`

6. **游戏结束判定**
   - 时间到 0：调用 `endGame()`，当前先用 `alert('时间到！得分：' + state.gameScore)` 占位
   - 所有 tile 匹配完：调用 `endGame()`，当前先用 `alert('恭喜通关！得分：' + state.gameScore)` 占位
   - 完整结算页在下一步实现

7. **约束**
   - 只修改 `闪卡记忆_base.html` 这一个文件
   - 保持现有闪卡学习逻辑不变
   - 不要引入外部库
   - 保持移动端 390px 宽度下可正常显示

---

## 第 2 步：完善计分、连击、计时和结算页

**目标**：让消除游戏有完整的得分、连击、计时反馈和结算展示。

请基于上一步修改后的文件继续：

1. **计分系统**
   - 基础匹配得分：每次正确匹配 +10 分
   - 连击加成：当前 combo 为 N 时，本次得分为 `10 * (1 + N * 0.5)`，即 combo 越高得分越多
   - `state.combo`：连续正确匹配次数，匹配错误时清零
   - `state.maxCombo`：记录本局最高连击
   - 每次得分变化时，用短暂放大动画更新 `gameScore` 显示

2. **连击显示**
   - `comboDisplay` 在有 combo ≥ 2 时显示 `x2 连击`、`x3 连击` 等
   - 使用 `.active` 类触发放大出现动画，1.5s 后淡出
   - combo 达到 5、10、15 时，显示特殊颜色（金色）和文字强调

3. **计时系统**
   - 总时间 60 秒
   - 正确匹配：+2 秒
   - 错误匹配：-2 秒
   - 使用 `setInterval` 每秒更新 `timeLeft` 和 timer-bar 宽度
   - 当 `timeLeft <= 10` 时：
     - timer 文字变红色
     - timer-bar-fill 变红色
     - 每秒播放一次急促的 `SFX.tick()` 警告音（800Hz 短促脉冲，0.05s）

4. **结算页改造**
   - 将 `completionScreen` 改造成 `resultScreen`（id 仍可用 resultScreen）
   - 显示内容：
     - 大标题：根据表现显示「太棒了」「还不错」「继续加油」
     - 星级：3 颗星，根据正确率点亮（≥90% 3星，≥70% 2星，≥50% 1星）
     - 数据卡片：得分、最大连击、正确率
     - 单词回顾列表：显示本轮 10 个词 + 学习时标记状态（认识/模糊/不认识）
     - 两个按钮：「返回首页」（调用 goHome）、「再来一局」（调用 startTodayTask）
   - `endGame()` 函数：
     - 清除 timer interval
     - 设置 `state.gameActive = false`
     - 计算正确率 `state.correct / (state.correct + state.wrong) * 100`
     - 调用 `showScreen('resultScreen')` 并填充数据

5. **音效补充**
   - `SFX.tick()`：时间警告音
   - `SFX.victory()`：游戏结束时播放胜利音效（三个上行音符：523Hz → 659Hz → 784Hz，各 0.15s）
   - `SFX.defeat()`：时间到且表现差时播放低沉音效（300Hz → 200Hz，0.3s）

6. **动画增强**
   - 得分更新时，分数数字短暂放大 1.3 倍再恢复
   - 星级点亮时逐个弹出动画
   - 结算页进入时有 fade-in + 上移动画

7. **约束**
   - 只修改 `闪卡记忆_base.html`
   - 所有状态用 `state` 对象管理
   - 时间 interval 在游戏结束或返回首页时必须清除，避免内存泄漏

---

## 第 3 步：加入经济系统（金币、宝石、等级）

**目标**：为游戏增加长期成长动力：金币、宝石、等级。

1. **数据结构**
   - 新增 `EconomySystem` 对象，管理：
     - `coins`：金币
     - `gems`：宝石
     - `totalScore`：累计总得分
     - `level`：当前等级对象 `{lv, title, score}`
     - `totalDays`：累计学习天数
     - `totalAttempts` / `totalCorrect`：累计答题次数和正确次数
   - localStorage 键名：`wordmatch_economy`
   - 新增 `const ECONOMY_KEY = 'wordmatch_economy'`

2. **等级表**
   - 至少 5 个等级：
     - Lv.1 新手上路：0 分
     - Lv.2 初窥门径：500 分
     - Lv.3 渐入佳境：1500 分
     - Lv.4 词汇达人：3500 分
     - Lv.5 单词大师：7000 分
   - `getLevel()` 根据 totalScore 返回当前等级
   - `getNextLevel()` 返回下一等级信息，用于进度条

3. **奖励规则**
   - 每次正确匹配：+2 金币
   - 每局结算：
     - 基础奖励 10 金币
     - 每颗星额外 +5 金币
     - 正确率 100% 额外 +10 金币
   - 宝石：
     - 获得 3 星奖励 1 宝石
     - 首次达到新等级奖励 3 宝石
   - 累计数据：每局结束后更新 totalScore、totalAttempts、totalCorrect、totalDays

4. **Header 改造**
   - Header 右侧显示：
     - 等级徽章：`Lv.1 新手上路`
     - 金币：`🪙 128`
     - 宝石：`💎 5`
   - 使用 `.header-economy` 容器，三个 `.economy-badge`
   - 更新函数 `updateEconomyUI()` 在以下时机调用：
     - 页面加载
     - 每局结束后
     - 获得奖励时

5. **结算页奖励展示**
   - 在 resultScreen 增加 `reward-strip`：
     - 显示本局获得的金币和宝石
     - 用胶囊形标签展示
   - 如果升级，显示「升级！Lv.1 → Lv.2」横幅

6. **音效**
   - `SFX.coin()`：获得金币时的清脆叮当声（1000Hz + 1300Hz，短促）
   - `SFX.levelUp()`：升级时的庆祝音阶（523 → 659 → 784 → 1047Hz）

7. **约束**
   - 只修改 `闪卡记忆_base.html`
   - 经济数据持久化到 localStorage
   - 如果 localStorage 数据损坏，使用默认值并重新初始化

---

## 第 4 步：加入道具系统和商店

**目标**：让游戏更有策略性，玩家可以用金币购买道具。

1. **道具设计**
   - 三种道具：
     - **提示（hint）**：高亮显示一对可匹配的 tile，持续 1 秒，消耗 10 金币
     - **加时（+10s）**：立即增加 10 秒倒计时，消耗 15 金币
     - **重排（shuffle）**：重新打乱剩余未匹配的 tile 位置，消耗 20 金币
   - 道具数量存在 localStorage，键名：`wordmatch_powerups`
   - 初始每种道具 3 个

2. **游戏内道具栏**
   - 在 gameScreen 底部新增 `powerups` 区域
   - 三个按钮：`提示 <span class="count" id="hintCount">3</span>`、`+10 <span class="count" id="timeCount">2</span>`、`重排 <span class="count" id="shuffleCount">1</span>`
   - 按钮样式：白色背景、绿色边框、圆角 14px、带阴影
   - 数量不足或游戏未开始时按钮 disabled

3. **道具逻辑**
   - `useHint()`：
     - 找到当前未匹配的两个可配对 tile
     - 给它们添加 `.hint` 类（金色边框闪烁），1 秒后移除
     - 消耗 1 个 hint
   - `useTimeBonus()`：
     - `state.timeLeft += 10`
     - 更新 timer 显示
     - 消耗 1 个 timeBonus
   - `useShuffle()`：
     - 只重新打乱当前未匹配的 tile 的 DOM 顺序
     - 保持 tile 内容不变
     - 消耗 1 个 shuffle

4. **商店系统**
   - 新增一个 `shopModal`：
     - 标题：「🛒 道具商店」
     - 显示当前金币余额
     - 三个商品卡片：提示 x1（10 金币）、+10s x1（15 金币）、重排 x1（20 金币）
     - 每个商品显示图标、名称、价格、购买按钮
     - 金币不足时购买按钮 disabled
     - 底部「关闭」按钮
   - 在首页底部或结算页添加「商店」入口按钮
   - `openShop()` / `closeShop()` 控制 modal 显示

5. **音效**
   - `SFX.powerup()`：使用道具时的轻快音效
   - `SFX.buy()`：购买成功时的「叮」声
   - `SFX.deny()`：金币不足时的低沉提示音

6. **约束**
   - 只修改 `闪卡记忆_base.html`
   - 商店 modal 使用现有 modal 样式或新建一致样式
   - 道具数量持久化

---

## 第 5 步：数据统计页和个人中心

**目标**：让玩家看到自己的学习成果，建立长期使用的动力。

1. **底部导航栏**
   - 在页面底部新增固定导航栏 `bottomNav`
   - 三个 Tab：「首页」「统计」「设置」
   - 图标可用 emoji：🏠 📊 ⚙️
   - 当前 Tab 高亮绿色，其他灰色
   - 点击切换对应 screen

2. **统计页（statsScreen）**
   - 显示内容：
     - 顶部 3 个数据卡：累计学习天数、已掌握/总词数、当前等级
     - 中间环形图：新词 / 学习中 / 已掌握 / 已遗忘 占比（用 SVG 绘制）
     - 词库进度条：CET-4 已掌握百分比
     - 下方 4 宫格：今日待复习、历史正确率、最高连击、累计金币
   - 数据来源：`MemorySystem.getLearningStats()` 和 `EconomySystem.data`
   - 添加「返回首页」按钮

3. **设置/个人中心页（settingsScreen）**
   - 显示：
     - 头像区域（可用简单 SVG 或 emoji）
     - 等级称号
     - 等级进度条
     - 金币、宝石数量
     - 「打开商店」按钮
     - 「清除学习数据」按钮（红色，带 confirm 确认）
   - 整体居中布局，保持可爱风格

4. **Header 改造**
   - Header 右侧保留进度显示，移除或简化经济徽章（因为底部导航和设置页已展示）
   - 或者 Header 右侧只放一个个人中心入口按钮，点击切换到设置页

5. **音效**
   - `SFX.tab()`：切换 Tab 时的轻短点击音

6. **约束**
   - 只修改 `闪卡记忆_base.html`
   - 底部导航在首页、统计、设置页显示，在学习/游戏/结算页隐藏
   - 统计页数据为空时显示 0 占位，不报错

---

## 第 6 步：每日连胜、新手引导和最终打磨

**目标**：增加每日活跃机制，优化首次体验，修复细节问题。

1. **每日连胜系统**
   - 新增 `StreakSystem`：
     - 记录最后访问日期 `lastVisitDate`
     - 连续访问天数 `streak`
     - 判断逻辑：
       - 今天已访问：不处理
       - 昨天访问过：streak + 1
       - 昨天没访问：streak 重置为 1
       - 首次访问：streak = 1
   - 连胜奖励：
     - streak 1：5 金币
     - streak 3：15 金币 + 1 宝石
     - streak 7：50 金币 + 3 宝石
     - streak 30：200 金币 + 10 宝石
   - 新增 `streakModal`：
     - 显示「连续学习 X 天」
     - 显示奖励
     - 「领取奖励」按钮
   - 页面加载时检测连胜，如果是新的一天且 streak ≥ 1，弹出 modal

2. **新手引导**
   - 首次访问时显示 onboarding overlay
   - 3 页引导：
     - 第 1 页：欢迎来到闪卡记忆
     - 第 2 页：先学闪卡，再玩消除游戏
     - 第 3 页：完成任务获得金币和等级
   - 底部圆点指示器 +「下一步」「开始学习」按钮
   - 完成后设置 `localStorage.setItem('wordmatch_onboarded', 'true')`

3. **动画和交互打磨**
   - 首页 level-btn 选中时有轻微上浮和阴影变化
   - 学习卡片翻转更平滑（3D flip）
   - tile 消除时增加粒子/星光效果（CSS）
   - combo 达成时屏幕中央闪过 combo 文字
   - 所有按钮点击时有轻微缩放反馈

4. **响应式修复**
   - 确保在 390px 宽手机上无横向滚动
   - 游戏 board 在不同词数下自适应列数（10词用 4列，tile 自动换行）
   - 底部导航不遮挡内容，主容器加 `padding-bottom`

5. **Bug 修复**
   - 修复快速点击 tile 导致的重复判定
   - 修复返回首页后再次开始游戏时状态未重置
   - 修复 timer interval 未清除的潜在问题
   - 修复音频在 iOS Safari 上需要用户手势解锁的问题

6. **音效补充**
   - `SFX.streak()`：连胜奖励音
   - `SFX.complete()`：每日任务完成音

7. **最终检查**
   - 所有 localStorage 键名统一前缀 `wordmatch_`
   - 所有屏幕切换正常
   - 游戏可完整跑通一局
   - 刷新页面后数据不丢失

---

## 开发节奏建议

| 步骤 | 预计 Trae 会话数 | 交付物 |
|---|---|---|
| 第 1 步：核心消除游戏 | 1 个 Session | 可玩的配对游戏 |
| 第 2 步：计分/连击/结算 | 1 个 Session | 完整单局体验 |
| 第 3 步：经济系统 | 1 个 Session | 金币/等级/奖励 |
| 第 4 步：道具/商店 | 1 个 Session | 可购买道具 |
| 第 5 步：统计/设置 | 1 个 Session | 3 Tab 导航 |
| 第 6 步：连胜/引导 | 1 个 Session | 完整产品 |

**总计 6 个 Session**，每步都能拿到独立的 Session ID 和截图，非常适合写参赛帖。
