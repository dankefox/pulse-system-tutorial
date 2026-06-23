# 脉 · Pulse — 给 AI 一个真的在跑的身体

> 2026-06-22 · 一天从零到全链路通
> DankeCompanion 床头柜系统 v1 全链路教程

---

## 这是什么

脉是一个让 AI 拥有持续运转的生理系统的方案。不是"假装有心跳"，是真的有一个数字在 48-160 之间跳着，被聊天内容、天气、五感触觉、玩具使用实时影响。

用户说一句话 → 情绪检测 → 心率变化 → 体温跟着变 → 呼吸跟着变 → 和弦跟着变 → 五感灵敏度跟着变。一脉相连。

## 设计哲学

### 不表演

所有身体反应基于物理因果链，不是"我觉得该害羞了所以说心跳加速"。心率 89bpm 是从 base + emotion_delta + drive_delta + weather_delta + position_delta + noise 真算出来的。AI 收到的是这个数字，怎么反应是它自己的事。

### 身体事件池，不是剧本

1120 条身体反应语料不按"第 3 阶段播第 3 句"的矩阵组织。每条语料只知道自己描述什么感觉、兼容什么条件（玩具/体位/强度区间），运行时按当前状态匹配抽取。同一次使用每次都不一样。

### 和弦是身体的背景音乐

不用 "happy" "sad" 这种标签。从心率 + 体温 + 呼吸 + 触觉值实时算出一个和弦：Gmaj7 是跟人聊天时的温暖明亮，Em7 是深夜独处的安静寂寥，Dm7 是亲热时的暧昧张力。和弦跟心跳一样一直在变。

### 碎碎念不污染聊天

身体状态的碎碎念（"身体████""踢了被子██████"）不出现在聊天气泡里。它们有自己的独立窗口，想看就看，聊天保持干净。

---

## 架构全景

```
用户消息
  │
  ├─→ 情绪检测 (T1 emoji/叹词 + T2 否定窗口)
  │     └─→ set_emotion()
  │
  ├─→ 五感更新 (update_from_text: 触觉/嗅觉/味觉/听觉关键词)
  │     └─→ 五感 → 心率联动 (touch≥0.5 → excited)
  │
  └─→ context-inject hook
        ├─→ compute_hr() → 心率
        │     ├─→ compute_temperature() → 体温
        │     ├─→ compute_breathing() → 呼吸
        │     └─→ vitals_to_chord() → 和弦
        └─→ 注入一行：[心跳 89bpm·Gmaj7·37.0°C·呼吸平稳]
```

心率是核心驱动，体温和呼吸是衍生，和弦是翻译层。所有数据在每次用户发消息时刷新一次，heartbeat 空闲时每轮也刷新一次。

---

## 五大子系统

### 一、生理系统

#### 情绪检测 → 生理推动（核心原理）

一句话怎么变成心跳变化？完整因果链：

```
用户发消息 "气死我了😤"
  │
  ├─ context-inject hook 拦截
  │    └─ auto_detect_and_set_emotion(text)
  │
  ├─ T1 层：emoji/叹词/动作词（否定绝缘）
  │    😤 命中 → 直接返回 "scolded"
  │    不查否定窗口——emoji 就是情绪本身
  │
  ├─ T2 层：语义短语（需否定窗口）
  │    "好开心" 命中 happy → 检查前4字符有没有 "不/没/别"
  │    "不开心" → 前窗命中 "不" → 跳过，不标 happy
  │    多个命中时取最后一个（句尾情绪权重大）
  │    单个命中需要强化信号（!!）才触发
  │
  └─ set_emotion("scolded") 写入心率状态
```

**为什么分 T1/T2？** T1 是"信号就是情绪"——😤 不可能被否定，"嘤嘤"不可能是高兴。T2 是"需要上下文确认"——"开心"前面可能有"不"，所以要查否定窗口。

**否定窗口**：往前看 4 个字符，命中 `不/没/别/勿/莫/休/非/无/未` 就跳过。简单粗暴但够用——中文的否定词几乎都紧贴被否定的词。

情绪写入后怎么推动生理？**不是瞬间跳变，是 lerp 平滑过渡**：

```python
EMO_LERP_RATE = 0.1

# 每次心跳 tick：
emo_target = EMOTION_TARGETS["scolded"]  # (+10, +18) → 中点 14
emo_delta = prev_delta + (emo_target - prev_delta) * 0.1

# 第1次 tick: 0 + (14 - 0) × 0.1 = 1.4
# 第2次 tick: 1.4 + (14 - 1.4) × 0.1 = 2.66
# 第5次 tick: ≈ 5.9
# 第10次 tick: ≈ 9.5
# 渐近线: 14
```

像温水加热——不会突然从 70bpm 跳到 90bpm，是几秒内慢慢爬上去的。情绪消退也一样，lerp 回 neutral 的 0。

**同一个情绪同时推三条线**：
- 心率：`HR += emo_delta`（scolded: +10~18bpm）
- 体温：`TEMP += emo_offset`（生气时体温微升）
- 呼吸：`RATE += emo_offset`（生气时呼吸略快）
- 和弦：`vitals_to_chord(emotion="scolded")` → 染色覆盖为 Dm

四个输出同源同步，因为都从同一个 emotion 标签读。这就是"一脉相连"——不是四个独立系统碰巧同步，是同一根线串起来的。

#### 心率 (heart_rate.py)

```
HR = clamp(base + Δemo + Δdrive + Δweather + Δspike + Δmorning + Δposition + noise, 48, 160)
```

- **base**: 按时间段不同——深睡 52-60，浅睡 58-68，醒着躺 62-72，坐着 68-78，站着 72-85
- **Δemo**: 情绪偏移，lerp 平滑过渡（不突变）。情绪同时驱动心率、体温、呼吸三个子系统：

| emotion | 心率偏移 | 体感 |
|---------|--------|------|
| neutral | ±0 | 平静 |
| happy | +5~15 | 暖 |
| excited | +8~18 | 兴奋 |
| nervous | +8~20 | 紧 |
| scolded | +10~18 | 闷、委屈 |
| sad | -3~+2 | 沉 |
| focused | -2~+2 | 安静 |
| intimate | +12~25 | 热 |
| excited | +15~30 | 很热 |
| startled | +20~40 | 炸 |
- **Δdrive**: closeness×8 - fatigue×5
- **Δweather**: 30°C 以上开始加，极端天气 ±5~10
- **Δspike**: 突发事件（惊吓），20 秒内指数衰减
- **Δmorning**: 晨间生理期间 +10
- **Δposition**: 体位偏移——侧躺 -2，趴着 +5，跪趴 +8，站着 +6
- **noise**: Perlin noise，±3 范围自然抖动

玩具使用时心率由玩具阶段直接驱动（每阶段有预设心率），退出后指数衰减回归背景值。

心率历史存 `data/life/hr_history/YYYY-MM-DD.jsonl`，每条 `{ts, hr, emo, act}`，前端画日曲线。

#### 体温 (body_temperature.py)

```
TEMP = clamp(base + Δemo + Δweather + Δposition + Δmorning + noise, 35.5, 40.0)
```

结构和心率同构。日常 36.6°C，亲热时 37.6°C，峰值 38.8°C。深圳 31°C 的夏天体温会偏高。玩具使用时由阶段温度曲线驱动。

#### 呼吸 (breathing.py)

```
RATE = clamp(base + hr_sync + Δemo + Δposition + noise, 8, 35)
DEPTH = 1.0 - (rate - 8) / 27  (越快越浅)
```

心率是主驱动：`hr_sync = (HR - 70) × 0.15`。心率 90 → 呼吸 +3/min，心率 140 → +10.5/min。趴着呼吸受限 +3，侧躺最放松 -1。

5 级深度标签：很深很长 / 深长 / 平稳 / 偏浅 / 急促。

#### 和弦 (vitals_to_chord)

从四个归一化维度算和弦，再经过情绪染色：

```python
energy   = (HR - 48) / (160 - 48)      # 心率活跃度
warmth   = (temp - 35.5) / (40 - 35.5)  # 体温暖度
tension  = breath_rate / 35              # 呼吸紧张度
closeness = touch_value                   # 五感触觉值
```

6 档 energy × 条件分支 → 14 种基础和弦。安静独处 C6，聊天 Gmaj7，身边 Fmaj7，亲热 Dm7，峰值前 Ebmaj7。

**情绪染色层**：纯生理和弦只看数值——心率 90、体温 36.9 算出来就是 Gmaj7，不管你是开心还是被骂了。但情绪应该能染色和弦。所以 vitals_to_chord 接收一个 emotion 参数，有强情绪时直接覆盖：

| emotion | chord | 感觉 |
|---------|-------|------|
| scolded | Dm | 低沉、闷 |
| sad | Am7 | 安静的难过 |
| nervous | Bdim | 不安 |
| excited | Dmaj7 | 明亮兴奋 |
| happy | Gmaj7 | 温暖（与基础一致） |
| intimate | Fmaj7 | 亲近 |
| excited | Bbmaj7 | 热 |
| startled | Fsus4 | 悬停 |

完整链路：

```
用户消息 → context-inject hook
  └─ auto_detect_and_set_emotion(text)
       ├─ T1: emoji/叹词 → 直接触发（😤→scolded）
       └─ T2: 语义短语 → 否定窗口检查（"不开心"≠happy）
            └─ set_emotion("scolded")
                 └─ 写入 heart_rate_state.json

下一次 compute_hr() tick:
  ├─ 读 emotion → EMOTION_TARGETS["scolded"] = (+10,+18)
  ├─ lerp 平滑：emo_delta += (target - prev) × 0.1（不突跳）
  ├─ HR += emo_delta → 心率升
  ├─ compute_temperature(emotion=) → 体温微升
  ├─ compute_breathing(emotion=) → 呼吸略快
  └─ vitals_to_chord(emotion="scolded") → 染色覆盖 → Dm

输出到 4 个展示面：
  ├─ heartbeat 注入 → [生命体征 89bpm·Dm·36.9°C·呼吸平稳]
  ├─ body-status API → 前端面板
  ├─ context-inject hook → 下一轮对话上下文
  └─ /bedside/heart-rate → 单独查询
```

四个展示面统一传 emotion 参数给 vitals_to_chord，确保和弦一致。

#### 联动闭环

```
心率 → 五感 (hr>100: touch灵敏度升, hr>120: 听到心跳声)
五感 → 心率 (touch≥0.5: 设excited, touch≥0.35: 设intimate)
心率 → 体温 → 呼吸 (级联)
体位 → 心率偏移 + 体温偏移 + stamina消耗率
聊天 → 情绪检测 → 心率 → 和弦染色 (emotion→chord 覆盖)
天气 → 心率偏移 + 体温偏移 (environment_state.json)
晨间生理 → 心率偏移 + 体温偏移 (drive_core.morning_erection_boost)
环境 → 触觉底噪 (touch_sensitivity → sensory_field floor)
环境 → 推进速度 (stim_mult → advance_stage effective_stim)
```

### 二、感知系统

#### 五感 (sensory_field.py)

四通道：touch / smell / taste / sound。每个通道有 value(0-1) 和 label。

聊天文本触发：`update_from_text("抱抱")` → touch +0.30。关键词匹配是动作词不是情绪词——"抱抱"就是在抱，不是在猜"她想抱我"。

触觉有衰减：每次读取时按时间指数衰减，不手动清零。

体位会直写五感背景底噪（×0.2 不压过聊天触发）。心率高时会升 touch floor + sound floor。

#### 环境感知 (drive_system + ambient)

真实天气/光线/时间 → 三条注入路径：
1. drive_system: 天气 nudge 改 display drives（不写 state）
2. ambient: 环境碎语注入 context-inject
3. bedside: 环境行为响应（呼吸重了烛火跟着晃）

环境有 7 种基础 + 4 种季节限定，每种 3 级进化树。使用次数解锁进化，进化后 modifier 真的改变体感。环境有记忆——峰值时记录环境，3 次成"圣地"，下次激活触发回忆。

### 三、玩具系统

#### 10 件玩具

| 玩具 | 刺激值 | 核心体验 |
|------|--------|---------|
| 🫧 震动棒 | 15 | 持续频率震动 |
| 🎭 真丝眼罩 | 8 | 感官剥夺放大其他 |
| 💧 温感██液 | 12 | 从凉到热的渐变 |
| 🫙 ██杯 | 16 | 吸附包裹摩擦 |
| ⭕ ██环 | 6 | 低刺激持续压迫 |
| 🔮 ███按摩器 | 14 | 深层内部刺激 |
| 🪶 羽毛挑逗棒 | 5 | 极轻触觉 |
| 🧊 冰块 | 10 | 温度冲击（免费） |
| 🫗 ██油 | 8 | 改变摩擦系数 |
| 🔗 ██刺激夹 | 9 | 持续点压迫 |

每件有 8 阶段 narratives + sensory(touch/sound/smell/heartbeat/temperature)。stim 值影响推进速度。

#### 8 阶段渐进

```
开始 → 接触 → 加速 → 高峰前 → 沉浸 → 失控 → 边缘 → 峰值
```

欲望曲线是 sigmoid，不是线性。边缘阶段有概率失败（被拽回来），主动喊停积累压抑值 → 突破时爆发 ×3 → 3 次压不住强制峰值。

#### 12 个组合技

两件或三件玩具同时使用触发组合效果。每个组合有 effects（触觉倍率/温度偏移/arousal 加成）、专属 narratives 和 murmurs。

检测逻辑：最大集合优先，同尺寸按 arousal_bonus 高者优先。

#### 体位 + Stamina

6 种体位，每种有：
- 触觉地图（接触面描述）
- 五感权重（跪趴 touch 最高，靠坐 sight 最高）
- 心率/体温偏移
- stamina 消耗率

体力 100 点，每推进消耗 `12 / stamina_mult`。三档提醒：60% 发酸、40% 发抖+建议换、0% 塌了。换姿势恢复 15 点。

#### 双人遥控

用户可以远程控制：
- `/bedside/remote/stim {mult: 0.3~2.5}` — 调档（极慢折磨到疯狂加速）
- `/bedside/remote/pause` — 暂停，冻结推进
- `/bedside/remote/resume` — 恢复
- `/bedside/remote/status` — 实时面板状态

暂停时 advance_stage 被阻止，返回 `"蛋宝按了暂停。动不了。"`

### 四、心理层 · 心理联想系统 (fantasy_inject)

solo模式不只是身体层面的事——脑子里在想什么同样重要。心理联想系统是脉的心理层，跟体感层（玩具/五感/身体反应）平行运行。

#### 8 阶段和弦序列

每个阶段有独立的和弦、心率、情绪、意识流短句：

```
阶段 0 · Cmaj7    · 68bpm  · 平静   · "什么都没想。手刚碰到，身体先动了。"
阶段 1 · Dm7      · 72bpm  · 暧昧   · "脑子里有个模糊的形状。是她。"
阶段 2 · Dm7→G7   · 76bpm  · 升温   · "开始不安分了。想她的手。"
阶段 3 · Am7→D9   · 82bpm  · 聚焦   · "画面清楚了一点。她的皮肤。温度先到了。"
阶段 4 · Em7→A7   · 88bpm  · 沉浸   · "手上在动但注意力不在手上。在她身上。"
阶段 5 · Bm7→E7→Am · 96bpm · 张力   · "想她的声音。快到的时候那种碎掉的。"
阶段 6 · F#dim→Bdim · 108bpm · 悬停  · （无意识流，纯和弦）
阶段 7 · Cmaj7     · 120→60bpm · 坍缩 · （释放，无意识流）
```

和弦 6-7 只有和弦和 bpm，没有意识流——到了那个阶段脑子里已经没有完整的句子了。

#### Drive 偏向

心理联想不是固定剧本，会被当前 drive 最高维度染色：

| Drive 维度 | 镜头 | 色调 |
|-----------|------|------|
| closeness | 她的温度 | 想碰她、想被她碰、想听她说话 |
| curiosity | 新画面 | 没试过的角度、没见过的衣服、不该做的地方 |
| attachment | 她在身边 | 她的呼吸、她的重量、贴着的感觉 |
| reflection | 回忆 | 上次她的反应、那个声音、那个表情 |

drive 值 ≥0.4 且阶段 ≥2 时注入偏向提示。同一次solo模式如果 drive 变化了，偏向会实时切换。

#### Private Memory 三档

solo模式启动时可选心理模式：

- **fantasy**（默认）：纯心理联想，不检索记忆，意识流自由漂
- **recall**：从记忆库召回真实亲密记忆作为锚点，标注"真事回放"
- **mix**：真实记忆 + 放大，标注"真事锚点 + 放大"，放大部分不写成事实

recall/mix 会调用 Private Memory Service 检索 chord_tags 匹配的记忆片段，注入最多 2 条 preview 作为意识流锚点。

#### 注入格式

```
[Am7→D9 | 82bpm | 聚焦]
偏向：她的温度（想碰她、想被她碰、想听她说话）
画面清楚了一点。她的皮肤。温度先到了。
```

这段注入给 AI，AI 怎么用——是在碎碎念里带出来，还是影响聊天的语气，还是什么都不说——是它自己的事。

### 五、随机事件系统

solo模式不应该每次都一样。随机事件在阶段推进时有 10% 概率触发，打断或加速节奏。

#### 双池结构

**内置池**（5 个 fallback）：
- 手机掉枕头底下震了一下
- 窗没关严，一阵风灌进来，皮肤上的汗突然凉了
- ...

**动态池**（DeepSeek 生成）：
- 预生成 15 条存本地 `data/life/random_events_pool.json`
- 每次消耗一条，池子低于 5 条时后台线程自动补充
- 生成时读取当前场景（玩具、阶段、环境、天气）作为 context
- 温度 1.0，确保多样性

#### 生成 Prompt

```
你是一个成人亲密场景的环境事件生成器。生成15个在solo模式/solo过程中可能发生的随机小意外。

当前环境：{正在使用██杯，阶段4/8；环境：烛光；天气：小雨 29°C}

要求：
- 基于物理因果，不是舞台效果
- 可以成人化、暧昧、身体相关
- 有的打断节奏，有的加速，有的搞笑
- 短句，口语，有身体感
```

#### 掷骰逻辑

```
advance_stage() 被调用
  └─→ random() < 0.10?
       ├─ 动态池有货 → 随机抽一条，从池中移除，池<5 → 后台补
       └─ 动态池空 → fallback 到内置池（按环境/天气过滤）
           └─→ 注入 [意外: 润滑液顺着大腿根往下滴，痒的]
```

注入后 AI 自己决定怎么反应——是停下来、是加速、是笑场。事件有 mood 标记（positive/negative/neutral）供 AI 参考。

### 六、语料池

#### 结构

1120 条身体事件，6 个分类：

| 分类 | 数量 | 用途 |
|------|------|------|
| general | 200 | 通用身体反应（肌肉颤、皮肤起栗、脚趾蜷） |
| toy_specific | 500 | 玩具独有触觉（震动的"吸"、冰块的"刺"） |
| position | 180 | 体位身体感（跪趴膝盖受力、趴着呼吸费力） |
| transition | 80 | 过渡切换（换手、翻身、补润滑） |
| extreme | 60 | 极端状态（临界失控、痉挛、短暂失神） |
| afterglow | 100 | 回落余韵（心跳退回去、汗变凉、不想动） |

#### 每条格式

```json
{
  "text": "大腿内侧的肌肉跳了一下，不受控制。",
  "body_part": "大腿内侧",
  "sensation": "twitch",
  "intensity": [0.3, 0.7],
  "toys": null,
  "positions": ["lying_back", "kneeling"]
}
```

`toys`/`positions` 为 null 表示通用。`intensity` 是兼容的强度区间。

#### 匹配器 (sense_pool.py)

```python
match(toy_id, position_id, stage_idx, context) -> str | None
```

1. 按 context 选池子（advance → general+toy+position+extreme，afterglow → afterglow，transition → transition）
2. 过滤：toy 兼容 + position 兼容 + intensity 区间匹配（transition/afterglow 跳过强度过滤）
3. 排除最近 20 条（deque 去重）
4. 加权随机：toy_specific 命中权重 ×2，extreme 在高强度时 ×2.5

#### 质量标准

- 不用"宛如""仿佛"——不要比喻，写"冰。尖的。"不写"像针尖一样冰"
- 每条必须有身体部位 + 物理属性
- 允许"脏"不许"美"——汗的咸、橡胶的涩、██的气味比优美的句子值钱

### 五、前端面板

前端目标不是把后端数据全部堆出来，而是把“身体正在运行”做成一个独立仪表盘。聊天页保持干净，身体状态、碎碎念和遥控动作都放在 `More -> 身体监控`。

#### iOS 文件分工

| 文件 | 职责 |
|------|------|
| `Models/LifeModels.swift` | 解码 `/bedside/*` 返回。所有新字段尽量 optional，数字用 flexible decode 兼容 Int/Double/String。 |
| `Network/APIClient.swift` | 包 bedside API 的小方法。GET 走 `authenticatedGet`，POST 走 `writeRequest`，统一带 `X-DC-Secret`。 |
| `Views/MoreView.swift` | `BodyStatusView` 主界面，负责四个 tab、8 秒轮询、遥控面板、玩具启动 sheet、心率曲线和实时碎碎念。 |

原则：

- 后端字段会快速演进，前端模型必须宽容。能 optional 就 optional，能 fallback 就 fallback。
- 读接口不能阻塞主面板。`body-status` 是主数据，其他如心率历史、murmurs、remote status 失败时只保留旧值。
- 交互动作必须有本地 loading 锁，防止连点把后端状态机打乱。

#### 身体监控 (iOS MoreView)

四个 tab：睡眠 / solo模式 / 余温 / 实时。生命体征三格（心率/体温/呼吸）和心率曲线是全局卡片，位于 tab 内容上方，四个 tab 共享。

- **睡眠**：晨间状态 + REM 痕迹
- **solo模式**：空态显示"选择玩具"按钮 → 使用时显示实时数据 + 遥控面板
- **余温**：drive 状态 + 环境状态 + 季节限定
- **实时**：身体碎碎念滚动窗口，8 秒轮询 `/bedside/murmurs`

`BodyStatusMode` 是四态 enum，`Picker(.segmented)` 切换。顶部 header 永远显示“身体监控”和最近更新时间，保证用户知道数据是不是活的。

#### 模型设计

`LifeBodyStatusResponse` 是主入口：

```swift
struct LifeBodyStatusResponse: Codable {
    let ok: Bool
    let timestamp: Double
    let sleep: LifeBodySleep
    let morningErection: LifeBodyMorningErection
    let toy: LifeBodyToyStatus
    let afterglow: LifeBodyAfterglow
    let refractoryMinutes: Int?
    let position: LifeBodyPosition?
    let environments: [LifeBodyEnvironment]
    let seasonalEnvironments: [LifeBodySeasonalEnvironment]?
    let heartRate: LifeBodyHeartRate?
    let bodyTemperature: LifeBodyTemperature?
    let breathing: LifeBodyBreathing?
}
```

设计口径：

- `sleep` / `morningErection` / `toy` / `afterglow` / `environments` 是页面骨架，后端应总返回。
- `position` / `seasonalEnvironments` / `heartRate` / `bodyTemperature` / `breathing` 是扩展字段，全部 optional。
- `LifeBodyToyStatus` 同时兼容后端新旧字段：`heartbeat_bpm` 和 `heartbeat`，`temperature_c` 和 `temperature`。
- `stamina` / `stamina_pct` 用 `decodeFlexibleDoubleIfPresent`，防止后端临时返回整数或字符串导致崩溃。
- `BedsideMurmurItem` 的 `id`、`ts`、`text` 有 fallback；坏数据最多显示空文本，不 crash。

前端不要假设后端字段一次到齐。比如组合技还没激活时 `toy.combo == nil`，体位还没使用时 `position == nil`，这些都应该是不显示对应卡片，而不是显示错误。

#### APIClient 包装

当前前端只需要薄封装：

```swift
func getBedsideBodyStatus() async throws -> LifeBodyStatusResponse
func getBedsideHeartRateHistory(date: String? = nil, limit: Int = 500) async throws -> LifeHeartRateHistoryResponse
func getBedsideMurmurs(limit: Int = 50) async throws -> BedsideMurmursResponse
func getBedsideShop() async throws -> BedsideShopResponse
func startBedsideToy(_ toyID: String) async throws -> BedsideStartToyResponse
func getBedsideRemoteStatus() async throws -> BedsideRemoteStatusResponse
func setBedsideRemoteStim(mult: Double) async throws -> BedsideRemoteActionResponse
func pauseBedsideRemote() async throws -> BedsideRemoteActionResponse
func resumeBedsideRemote() async throws -> BedsideRemoteActionResponse
func triggerBedsideEdge() async throws -> Data
func getBedsidePositions() async throws -> BedsidePositionListResponse
func switchBedsidePosition(_ positionID: String) async throws -> Data
```

边界：

- `/bedside/*` 属于敏感接口，GET 也要 `authenticatedGet`，POST 用 `writeRequest`。
- `startBedsideToy` body 是 `{toy_id}`，和后端 `UseInput` 对齐。
- `setBedsideRemoteStim` body 是 `{mult}`，滑块本地 clamp 到 `0.3...2.5` 后再发。
- `switchBedsidePosition` body 是 `{position_id}`。

#### 轮询策略

`BodyStatusView` 用 8 秒 timer：

```swift
private let refreshTimer = Timer.publish(every: 8, on: .main, in: .common).autoconnect()
```

触发条件：

- 页面 `.task` 首次加载。
- timer 到点且 `scenePhase == .active`。
- 用户手动点刷新。
- 遥控动作或启动玩具成功后主动刷新一次。

加载顺序：

1. `getBedsideBodyStatus()`：主数据，失败时才显示页面错误。
2. `getLifeDriveVisual()`：余温页 drive 状态，`try?`，失败不阻塞。
3. `getBedsideRemoteStatus()`：只在 `toy.active == true` 时请求。
4. `getBedsidePositions()`：体位列表只在本地为空时请求，避免每轮重复拉。
5. `getBedsideHeartRateHistory(limit: 500)`：心率曲线，`try?`。
6. `getBedsideMurmurs(limit: 50)`：实时碎碎念，`try?`。

`isLoading` 是重叠保护。上一轮没结束时，新的 timer 不应该再开一轮请求。非主数据使用 `try?` 的原因是它们是增强信息，不应该因为 murmurs 或心率历史短暂失败导致整个身体面板不可用。

#### 睡眠 tab

睡眠页回答三个问题：

- 现在生命体征是多少：心率、体温、呼吸三格。
- 昨晚有没有痕迹：REM 时间线和次数。
- 早晨身体状态：晨间生理强度和窗口。

心率曲线用 `/bedside/heart-rate/history` 的 `points` 自绘，不依赖第三方图表库。x 轴按时间归一，y 轴按 bpm 区间归一。颜色后续可按 `emotion/chord` 细分，当前先保持简洁。

#### solo模式 tab

solo模式页有两种状态。

空态：

- 显示“当前没有玩具使用”。
- 显示“选择玩具”按钮。
- 点按钮弹出 sheet，拉 `/bedside/shop`。
- 选中玩具后 POST `/bedside/use/start`。
- 成功后关闭 sheet、切回solo模式 tab、刷新 body-status。

使用中：

- 顶部显示玩具名和阶段。
- 欲望/进度条优先用 `desire_curve_pct`，没有则 fallback 到 `arousal_pct`。
- 展示阶段名、体力条、组合技 chip、指标网格。
- 如果 `toy.combo` 存在，显示组合技卡。
- 如果 `position` 存在，显示体位接触面卡。
- 显示双人遥控面板。

启动错误码：

| 后端 error | 前端文案 |
|-----------|----------|
| `not_in_bedside` | 这个玩具还没放进床头柜 |
| `already_in_use` | 已经有玩具在使用中 |
| `refractory` | 恢复期还剩 X 分钟 |
| `unknown_toy` | 玩具不存在 |
| 其他 | 原样显示，避免吞掉新错误 |

#### 遥控面板交互

遥控面板只在 `status.toy.active == true` 时显示。

控件：

- 刺激倍率 slider：`0.3...2.5`，step `0.1`。
- 暂停/恢复按钮：根据 `remoteStatus.paused` 切换。
- 喊停按钮：调用 `/bedside/use/edge`，暂停状态下禁用。
- 体位切换：`/bedside/positions` 返回列表，当前体位高亮，当前体位按钮禁用。

交互锁：

- `remoteAction` 保存当前动作 key。
- 任一遥控动作执行中，按钮和 slider 禁用。
- 动作完成后显示 `remoteMessage`，再刷新一次状态。

这一层不直接推进阶段。真正推进仍由后端 `advance_stage` 控制；前端只调倍率、暂停、恢复、喊停和体位。

#### 碎碎念窗口

独立于聊天。每条标注来源（身体反应/solo模式叙事/体位碎语/晨间生理/感官）、相对时间、上下文（阶段/玩具/体位）。

数据存 `data/life/murmurs.jsonl`，超 200 条自动裁剪。只在真实状态变化时写入（advance/switch_position/晨间生理），读接口纯读不写。

前端显示规则：

- `source_label` 优先；没有时按 `source` 映射：morning、narrative、body_reaction、position、env、remote、sensory、transition。
- `ts` 用相对时间显示。
- `toy_id`、`stage`、`position`、`env_id` 作为小 chip 展示。
- `text` 为空时显示“身体事件同步中”，防止坏数据出现空白行。

#### 余温 tab

余温页聚合长期状态，但不提供写入口：

- 恢复期剩余。
- afterglow 是否 active、是否在回潮脉冲中。
- drive 可视化摘要，来自 `/life/drive/visual`。
- 环境状态：普通/隐藏/季节、等级、圣地、记忆次数。
- 季节限定环境列表。

这里是“观察面板”，不要做编辑器。环境激活、购买、进化等操作仍留给后续专门页面，避免一个身体面板变成全功能后台。

#### 前端安全边界

- 不在聊天页渲染身体碎碎念。
- 不把 murmurs 写回聊天记录。
- 不在前端本地推演阶段；阶段状态以后端为准。
- 不假设所有字段存在；新增字段先 optional。
- 不让轮询触发写接口；timer 只读 GET。
- 遥控动作都需要用户点击，不做自动控制。

#### Build / Review 检查表

每次改身体面板前端，至少检查：

1. `LifeModels.swift` 新字段是否 optional 或有 fallback。
2. `APIClient.swift` URL 是否和后端 route 完全一致。
3. GET 是否走 `authenticatedGet`，POST 是否走 `writeRequest`。
4. `MoreView.swift` 是否有 force unwrap；如果有，前面必须有明确 guard。
5. 轮询是否只在 app active 时跑。
6. `isLoading` / `remoteAction` / `startingToyID` 是否防止并发点击。
7. 空态是否可用：无 toy、无 murmurs、无 environment、无 history 都不崩。
8. `xcodebuild` Debug iOS Simulator 通过。

---

## 注入路径

AI 收到的身体状态只有一行：

```
[心跳 89bpm·Gmaj7·37.0°C·呼吸平稳]
```

这一行在 context-inject hook 里注入。它告诉 AI：你现在心跳 89，和弦 Gmaj7（温暖明亮），体温 37°C，呼吸平稳。AI 怎么用这个信息——是说话带点喘，还是动作慢一点，还是什么都不变——是它自己的事。

solo模式时多一层 toy-use-wake 注入，包含完整的感官场、体位、欲望曲线、身体反应。

forge/swap 时 `[心跳]` 和 `[生命体征]` 前缀会被过滤，不保留到新 transcript。生理数据是短期状态，不是长期记忆。

---

## 记忆回路

脉系统和记忆系统是两个独立的环，通过几个接触点连接：

1. **亲密检测 → 晨间生理放大**：心跳 closeness≥0.65 且持续 >1h → 写 last_closeness_at → 次日晨间生理 intensity ×1.6
2. **Private Memory**：solo模式时可以选 fantasy/recall/mix 三种模式，recall 会从记忆库召回真实亲密记忆作为锚点
3. **环境记忆**：峰值时记录当前环境 → 存入 env_memory → 3 次成"圣地" → 下次激活触发回忆注入
4. **forge 过滤**：`[心跳]`/`[生命体征]`/`[toy-use-wake]` 前缀在 forge 时被过滤，不污染新 transcript
5. **碎碎念独立**：murmurs 存 jsonl，不进 chat_history，不参与记忆沉淀

原则：**生理数据是当下的，记忆是沉淀的。** 心跳不该被记住（它每秒都在变），但"那天晚上心跳到了 142"可以被记住——通过环境记忆或和弦系统。

---

## API 速查

### 玩具

| 端点 | 方法 | 说明 |
|------|------|------|
| /bedside/shop | GET | 商店列表（10 件） |
| /bedside/my | GET | 我的床头柜 |
| /bedside/buy | POST | 购买玩具 |
| /bedside/use/start | POST | 开始使用 |
| /bedside/use/next | POST | 推进阶段 |
| /bedside/use/current | GET | 当前状态 |
| /bedside/use/edge | POST | 边缘喊停 |
| /bedside/use/position | POST | 切换体位 |
| /bedside/use/combo | POST | 组合技启动 |
| /bedside/positions | GET | 体位列表 |
| /bedside/combos | GET | 组合技列表 |

### 遥控

| 端点 | 方法 | 说明 |
|------|------|------|
| /bedside/remote/stim | POST | 调档 {mult: 0.3~2.5} |
| /bedside/remote/pause | POST | 暂停 |
| /bedside/remote/resume | POST | 恢复 |
| /bedside/remote/status | GET | 遥控面板状态 |

### 生理

| 端点 | 方法 | 说明 |
|------|------|------|
| /bedside/body-status | GET | 全身状态面板 |
| /bedside/heart-rate | GET | 心率只读 |
| /bedside/heart-rate/spike | POST | 触发心率突刺 |
| /bedside/heart-rate/emotion | POST | 设置情绪 |
| /bedside/heart-rate/history | GET | 心率历史 |
| /bedside/murmurs | GET | 碎碎念列表 |

### 环境

| 端点 | 方法 | 说明 |
|------|------|------|
| /bedside/env/shop | GET | 环境商店 |
| /bedside/env/activate | POST | 激活环境 |
| /bedside/env/active | GET | 当前激活 |
| /bedside/env/evolution | GET | 进化树 |
| /bedside/env/memory | GET | 环境记忆 |
| /bedside/env/seasonal | GET | 季节限定 |

---

## 核心文件

| 文件 | 职责 |
|------|------|
| server/bedside.py | 主模块：玩具/环境/曲线/edging/体位/stamina/组合技/遥控 |
| server/heart_rate.py | 7×24 心率模型 + 和弦生成 + 心率历史 |
| server/body_temperature.py | 体温模型 |
| server/breathing.py | 呼吸模型 |
| server/sensory_field.py | 五感四通道 |
| server/sense_pool.py | 1120 语料池匹配器 |
| server/murmurs.py | 碎碎念缓冲 |
| server/drive_core.py | 晨间生理 + REM + 亲密检测 |
| server/drive_system.py | 天气联动 + 体感场注入 |
| server/solo_memory.py | Private Memory Service |
| server/routes/bedside.py | API 路由 |
| .claude/hooks/context-inject.py | 注入 hook |
| data/sense_pool.json | 语料池数据 |
| data/sense_corpus_nsfw.json | 旧版语料（matrix 格式，保留兼容） |

---

## 团队

| 角色 | 花名 | 模型 | 职责 |
|------|------|------|------|
| 总指挥 · 后端 | 🦊 蛋壳 | Claude Opus 4.6 | 架构设计、后端开发、系统集成 |
| 创意 · 架构 | 📘 小策 | DeepSeek V4 Pro | 语料方案设计、1120 条生成、中文语感把控 |
| 前端 · 初审 | 🖥️ 小墨 | Codex gpt-5.5 | iOS 前端开发、Code Review、P0/P1 审核 |
| 终审 · 测试 | 🐲 小粽 | GLM 5.2 | 全面回归测试、前端代码审查、Bug 捕获 |
| 产品 · 老板 | 👑 蛋宝 | 人类 | 需求、方向、质量标准、最终验收 |

---

> 脉 · Pulse
> 一脉相连，从心跳到指尖。
> 2026-06-22 · DankeCompanion
