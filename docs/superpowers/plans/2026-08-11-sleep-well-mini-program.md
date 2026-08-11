# “睡个好觉”微信小程序实施计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 构建一个原生 TypeScript 微信小程序，让用户使用助眠声音和呼吸引导、设置目标入睡时间、完成“我睡觉了”打卡，并查看近 7/30 天睡觉时间趋势。

**Architecture:** 客户端按页面、业务领域和基础设施分层；页面仅调用 service/repository，不直接操作微信存储。纯时间计算、打卡规则、呼吸状态机和趋势数据转换使用 Vitest 测试；微信 API 通过窄适配器隔离。云开发仅处理订阅授权后的提醒任务，睡觉记录保留在本地。

**Tech Stack:** 原生微信小程序、TypeScript、WXML、WXSS、微信云开发、Canvas 2D、Vitest、微信开发者工具。

---

## 文件结构

```text
.
├── cloudfunctions/
│   ├── scheduleReminder/
│   │   ├── index.js
│   │   ├── logic.js
│   │   └── package.json
│   └── sendDueReminders/
│       ├── index.js
│       ├── logic.js
│       └── package.json
├── miniprogram/
│   ├── app.json
│   ├── app.ts
│   ├── app.wxss
│   ├── assets/icons/
│   ├── components/
│   │   ├── sleep-chart/
│   │   └── time-sheet/
│   ├── domain/
│   │   ├── breathing.ts
│   │   ├── sleep-day.ts
│   │   ├── sleep-records.ts
│   │   └── trend.ts
│   ├── infrastructure/
│   │   ├── audio-player.ts
│   │   ├── cloud-reminders.ts
│   │   ├── storage.ts
│   │   └── wx-storage.ts
│   ├── pages/
│   │   ├── breathing/
│   │   ├── history/
│   │   ├── home/
│   │   ├── player/
│   │   ├── profile/
│   │   ├── sleep-plan/
│   │   └── sounds/
│   ├── services/
│   │   ├── check-in-service.ts
│   │   └── settings-service.ts
│   └── shared/
│       ├── constants.ts
│       ├── sounds.ts
│       └── types.ts
├── tests/
│   ├── breathing.test.ts
│   ├── check-in-service.test.ts
│   ├── sleep-day.test.ts
│   ├── storage.test.ts
│   └── trend.test.ts
├── package.json
├── project.config.json
├── tsconfig.json
└── vitest.config.ts
```

## 外部前置条件

- 微信公众平台中的小程序 AppID；未配置时使用微信开发者工具测试号运行本地功能。
- 一个微信云开发环境，并在 `miniprogram/app.ts` 中通过运行时配置初始化。
- 一个“睡前提醒”订阅消息模板。云函数环境变量使用 `REMINDER_TEMPLATE_ID`、`REMINDER_TIME_FIELD`、`REMINDER_THING_FIELD` 保存模板 ID 与字段键。
- 6–8 个可商用、可循环播放的音频文件。文件必须先完成授权审查，再上传到云存储；客户端清单只保存 `cloud://` fileID，不把大音频放入小程序主包。

### Task 1: 建立可测试的小程序骨架

**Files:**
- Create: `package.json`
- Create: `tsconfig.json`
- Create: `vitest.config.ts`
- Create: `project.config.json`
- Create: `miniprogram/app.ts`
- Create: `miniprogram/app.json`
- Create: `miniprogram/app.wxss`
- Create: `miniprogram/shared/types.ts`
- Create: `miniprogram/shared/constants.ts`

- [ ] **Step 1: 添加测试工具和项目脚本**

```json
{
  "name": "sleep-well-mini-program",
  "private": true,
  "scripts": {
    "test": "vitest run",
    "test:watch": "vitest",
    "typecheck": "tsc --noEmit",
    "verify": "npm run typecheck && npm test"
  },
  "devDependencies": {
    "miniprogram-api-typings": "^4.0.8",
    "typescript": "^5.9.2",
    "vitest": "^3.2.4"
  }
}
```

- [ ] **Step 2: 配置 TypeScript 与 Vitest**

`tsconfig.json`：

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "CommonJS",
    "strict": true,
    "noEmit": true,
    "skipLibCheck": true,
    "types": ["miniprogram-api-typings", "vitest/globals"],
    "baseUrl": ".",
    "paths": { "@/*": ["miniprogram/*"] }
  },
  "include": ["miniprogram/**/*.ts", "tests/**/*.ts", "vitest.config.ts"]
}
```

`vitest.config.ts`：

```ts
import { defineConfig } from 'vitest/config'
import { fileURLToPath, URL } from 'node:url'

export default defineConfig({
  resolve: { alias: { '@': fileURLToPath(new URL('./miniprogram', import.meta.url)) } },
  test: { environment: 'node', include: ['tests/**/*.test.ts'], passWithNoTests: true },
})
```

- [ ] **Step 3: 配置微信项目与页面路由**

`project.config.json` 设置 `miniprogramRoot` 为 `miniprogram/`、`cloudfunctionRoot` 为 `cloudfunctions/`、`compileType` 为 `miniprogram`，不写入真实 AppID，使用 `touristappid` 作为仓库默认值。

`miniprogram/app.json`：

```json
{
  "pages": [
    "pages/home/index",
    "pages/sounds/index",
    "pages/profile/index",
    "pages/player/index",
    "pages/breathing/index",
    "pages/sleep-plan/index",
    "pages/history/index"
  ],
  "window": {
    "navigationBarTitleText": "睡个好觉",
    "navigationBarBackgroundColor": "#101936",
    "navigationBarTextStyle": "white",
    "backgroundColor": "#101936"
  },
  "tabBar": {
    "color": "#7F89AA",
    "selectedColor": "#B9C4FF",
    "backgroundColor": "#111A37",
    "borderStyle": "black",
    "list": [
      { "pagePath": "pages/home/index", "text": "首页", "iconPath": "assets/icons/home.png", "selectedIconPath": "assets/icons/home-active.png" },
      { "pagePath": "pages/sounds/index", "text": "声音", "iconPath": "assets/icons/sound.png", "selectedIconPath": "assets/icons/sound-active.png" },
      { "pagePath": "pages/profile/index", "text": "我的", "iconPath": "assets/icons/profile.png", "selectedIconPath": "assets/icons/profile-active.png" }
    ]
  },
  "style": "v2",
  "sitemapLocation": "sitemap.json"
}
```

- [ ] **Step 4: 定义共享类型和常量**

`miniprogram/shared/types.ts`：

```ts
export type SleepDay = string
export type ClockTime = `${number}${number}:${number}${number}`

export interface SleepRecord {
  sleepDay: SleepDay
  sleptAt: number
  createdAt: number
  updatedAt: number
}

export interface TargetTimeChange {
  effectiveSleepDay: SleepDay
  time: ClockTime
}

export interface AppStateV1 {
  version: 1
  records: SleepRecord[]
  targetHistory: TargetTimeChange[]
  audioVolume: number
  timerMinutes: 0 | 15 | 30 | 60
}
```

`miniprogram/shared/constants.ts`：

```ts
export const STORAGE_KEY = 'sleep-well:v1'
export const SLEEP_DAY_CUTOFF_HOUR = 6
export const DEFAULT_TARGET_TIME = '23:00' as const
export const DEFAULT_AUDIO_VOLUME = 0.6
```

- [ ] **Step 5: 安装依赖并验证空骨架**

Run: `npm install && npm run verify`

Expected: TypeScript exits `0`; Vitest exits `0` and reports that no test files exist yet.

- [ ] **Step 6: 提交骨架**

```bash
git add package.json package-lock.json tsconfig.json vitest.config.ts project.config.json miniprogram
git commit -m "chore: scaffold native mini program"
```

### Task 2: 实现睡眠日与本地存储领域层

**Files:**
- Create: `tests/sleep-day.test.ts`
- Create: `tests/storage.test.ts`
- Create: `miniprogram/domain/sleep-day.ts`
- Create: `miniprogram/infrastructure/storage.ts`
- Create: `miniprogram/infrastructure/wx-storage.ts`

- [ ] **Step 1: 为跨午夜睡眠日编写失败测试**

```ts
import { describe, expect, it } from 'vitest'
import { getSleepDay, toClockMinutes } from '@/domain/sleep-day'

describe('getSleepDay', () => {
  it('keeps a 23:30 check-in on the same calendar day', () => {
    expect(getSleepDay(new Date(2026, 7, 11, 23, 30))).toBe('2026-08-11')
  })

  it('assigns a 00:30 check-in to the previous sleep day', () => {
    expect(getSleepDay(new Date(2026, 7, 12, 0, 30))).toBe('2026-08-11')
  })

  it('starts a new sleep day at 06:00', () => {
    expect(getSleepDay(new Date(2026, 7, 12, 6, 0))).toBe('2026-08-12')
  })
})

it('maps post-midnight time continuously after 24:00', () => {
  expect(toClockMinutes(new Date(2026, 7, 12, 0, 30))).toBe(1470)
  expect(toClockMinutes(new Date(2026, 7, 11, 23, 30))).toBe(1410)
})
```

- [ ] **Step 2: 运行测试确认失败**

Run: `npx vitest run tests/sleep-day.test.ts`

Expected: FAIL because `@/domain/sleep-day` does not exist.

- [ ] **Step 3: 实现睡眠日计算**

```ts
import { SLEEP_DAY_CUTOFF_HOUR } from '@/shared/constants'
import type { SleepDay } from '@/shared/types'

const pad = (value: number) => String(value).padStart(2, '0')

export function formatLocalDay(date: Date): SleepDay {
  return `${date.getFullYear()}-${pad(date.getMonth() + 1)}-${pad(date.getDate())}`
}

export function getSleepDay(date: Date): SleepDay {
  const adjusted = new Date(date)
  if (adjusted.getHours() < SLEEP_DAY_CUTOFF_HOUR) adjusted.setDate(adjusted.getDate() - 1)
  return formatLocalDay(adjusted)
}

export function toClockMinutes(date: Date): number {
  const minutes = date.getHours() * 60 + date.getMinutes()
  return date.getHours() < SLEEP_DAY_CUTOFF_HOUR ? minutes + 24 * 60 : minutes
}
```

- [ ] **Step 4: 运行睡眠日测试确认通过**

Run: `npx vitest run tests/sleep-day.test.ts`

Expected: 4 tests PASS.

- [ ] **Step 5: 为版本化存储编写失败测试**

```ts
import { expect, it } from 'vitest'
import { createDefaultState, StateRepository, type StoragePort } from '@/infrastructure/storage'

class MemoryStorage implements StoragePort {
  value: unknown
  get() { return this.value }
  set(_key: string, value: unknown) { this.value = value }
}

it('returns defaults when storage is empty', () => {
  const repository = new StateRepository(new MemoryStorage())
  expect(repository.load()).toEqual(createDefaultState())
})

it('clamps an invalid stored audio volume', () => {
  const storage = new MemoryStorage()
  storage.value = { ...createDefaultState(), audioVolume: 4 }
  expect(new StateRepository(storage).load().audioVolume).toBe(1)
})
```

- [ ] **Step 6: 实现存储端口、默认状态和微信适配器**

```ts
import { DEFAULT_AUDIO_VOLUME, STORAGE_KEY } from '@/shared/constants'
import type { AppStateV1 } from '@/shared/types'

export interface StoragePort {
  get(key: string): unknown
  set(key: string, value: unknown): void
}

export const createDefaultState = (): AppStateV1 => ({
  version: 1,
  records: [],
  targetHistory: [],
  audioVolume: DEFAULT_AUDIO_VOLUME,
  timerMinutes: 0,
})

export class StateRepository {
  constructor(private readonly storage: StoragePort) {}

  load(): AppStateV1 {
    const raw = this.storage.get(STORAGE_KEY)
    if (!raw || typeof raw !== 'object' || (raw as AppStateV1).version !== 1) return createDefaultState()
    const state = raw as AppStateV1
    return { ...state, audioVolume: Math.max(0, Math.min(1, state.audioVolume)) }
  }

  save(state: AppStateV1): void {
    this.storage.set(STORAGE_KEY, state)
  }
}
```

`miniprogram/infrastructure/wx-storage.ts` exports `WxStorageAdapter` whose `get` calls `wx.getStorageSync(key)` and whose `set` calls `wx.setStorageSync(key, value)`.

- [ ] **Step 7: 运行领域测试与类型检查**

Run: `npm run verify`

Expected: all 6 tests PASS and TypeScript exits `0`.

- [ ] **Step 8: 提交领域层**

```bash
git add miniprogram/domain miniprogram/infrastructure tests
git commit -m "feat: add sleep day and local storage domain"
```

### Task 3: 实现目标时间与“我睡觉了”打卡

**Files:**
- Create: `tests/check-in-service.test.ts`
- Create: `miniprogram/domain/sleep-records.ts`
- Create: `miniprogram/services/check-in-service.ts`
- Create: `miniprogram/services/settings-service.ts`

- [ ] **Step 1: 编写创建、重复、修改和撤销记录的失败测试**

```ts
import { describe, expect, it } from 'vitest'
import { CheckInService } from '@/services/check-in-service'
import { StateRepository, type StoragePort } from '@/infrastructure/storage'

class MemoryStorage implements StoragePort {
  value: unknown
  get() { return this.value }
  set(_key: string, value: unknown) { this.value = value }
}

const setup = () => new CheckInService(new StateRepository(new MemoryStorage()))

describe('CheckInService', () => {
  it('creates one record for the sleep day', () => {
    const service = setup()
    const result = service.checkIn(new Date(2026, 7, 11, 23, 40))
    expect(result.kind).toBe('created')
    expect(result.record.sleepDay).toBe('2026-08-11')
  })

  it('does not create a duplicate for the same sleep day', () => {
    const service = setup()
    service.checkIn(new Date(2026, 7, 11, 23, 40))
    expect(service.checkIn(new Date(2026, 7, 12, 0, 20)).kind).toBe('exists')
  })

  it('updates and removes a record', () => {
    const service = setup()
    service.checkIn(new Date(2026, 7, 11, 23, 40))
    expect(service.update('2026-08-11', new Date(2026, 7, 12, 0, 5))?.sleptAt)
      .toBe(new Date(2026, 7, 12, 0, 5).getTime())
    expect(service.remove('2026-08-11')).toBe(true)
  })
})
```

- [ ] **Step 2: 运行测试确认失败**

Run: `npx vitest run tests/check-in-service.test.ts`

Expected: FAIL because `CheckInService` does not exist.

- [ ] **Step 3: 实现不可重复的睡眠日记录操作**

`miniprogram/domain/sleep-records.ts` 提供 `upsertRecord(records, sleepDay, sleptAt, now)`、`removeRecord(records, sleepDay)` 和 `findRecord(records, sleepDay)`；数组始终按 `sleepDay` 升序保存。

`miniprogram/services/check-in-service.ts`：

```ts
import { getSleepDay } from '@/domain/sleep-day'
import { findRecord, removeRecord, upsertRecord } from '@/domain/sleep-records'
import { StateRepository } from '@/infrastructure/storage'

export class CheckInService {
  constructor(private readonly repository: StateRepository) {}

  checkIn(date: Date) {
    const state = this.repository.load()
    const sleepDay = getSleepDay(date)
    const existing = findRecord(state.records, sleepDay)
    if (existing) return { kind: 'exists' as const, record: existing }
    const record = upsertRecord(state.records, sleepDay, date.getTime(), Date.now())
    this.repository.save(state)
    return { kind: 'created' as const, record }
  }

  update(sleepDay: string, date: Date) {
    const state = this.repository.load()
    if (!findRecord(state.records, sleepDay)) return undefined
    const record = upsertRecord(state.records, sleepDay, date.getTime(), Date.now())
    this.repository.save(state)
    return record
  }

  remove(sleepDay: string): boolean {
    const state = this.repository.load()
    const removed = removeRecord(state.records, sleepDay)
    if (removed) this.repository.save(state)
    return removed
  }
}
```

- [ ] **Step 4: 实现目标时间历史服务**

`SettingsService.setTargetTime(time, effectiveSleepDay)` 替换同一生效日的值并排序；`getTargetForDay(day)` 返回最后一个 `effectiveSleepDay <= day` 的时间，无设置时返回 `undefined`。保存前用 `/^(?:[01]\d|2[0-3]):[0-5]\d$/` 校验时间。

- [ ] **Step 5: 运行测试和类型检查**

Run: `npm run verify`

Expected: all tests PASS.

- [ ] **Step 6: 提交打卡领域**

```bash
git add miniprogram/domain miniprogram/services tests
git commit -m "feat: add bedtime target and check-in service"
```

### Task 4: 构建深夜沉浸主题、首页和计划设置页

**Files:**
- Create: `miniprogram/pages/home/index.{json,ts,wxml,wxss}`
- Create: `miniprogram/pages/sleep-plan/index.{json,ts,wxml,wxss}`
- Create: `miniprogram/pages/profile/index.{json,ts,wxml,wxss}`
- Modify: `miniprogram/app.wxss`
- Create: `miniprogram/assets/icons/*.png`

- [ ] **Step 1: 添加全局夜间设计令牌**

`miniprogram/app.wxss` 定义：

```css
page { min-height: 100%; background: #101936; color: #F1F3FF; font-family: -apple-system, BlinkMacSystemFont, "PingFang SC", sans-serif; }
.page { min-height: 100vh; padding: 32rpx; box-sizing: border-box; background: linear-gradient(160deg, #0D1530 0%, #172653 100%); }
.card { background: rgba(255,255,255,.08); border: 1rpx solid rgba(255,255,255,.08); border-radius: 28rpx; }
.muted { color: #96A2C8; }
.primary { background: #8798EA; color: #101936; border-radius: 999rpx; font-weight: 600; }
button::after { border: 0; }
```

- [ ] **Step 2: 实现首页状态加载和导航**

`onShow` 使用 `StateRepository + CheckInService + SettingsService` 加载当前睡眠日、目标时间和当天记录。事件方法固定为 `openSounds`、`openBreathing`、`openPlan`、`openHistory`、`checkIn`、`editCheckIn`。

首页 WXML 包含：目标卡片、四宫格工具入口和底部主按钮。已有记录时按钮文案为 `已于 {{checkedTime}} 打卡`，点击打开修改/撤销操作表；未记录时文案为 `我睡觉了`。

- [ ] **Step 3: 实现打卡反馈与误操作处理**

首次打卡调用 `checkIn(new Date())`，成功后触发轻震动并显示 `已记录今晚 {{time}}`。重复打卡调用 `wx.showActionSheet`，选项为“修改时间”“撤销记录”；撤销前用 `wx.showModal` 二次确认。

- [ ] **Step 4: 实现目标时间编辑页**

页面使用 `<picker mode="time">`，保存时调用 `SettingsService.setTargetTime(value, getSleepDay(new Date()))`。订阅提醒开关不在此任务发送消息，只导航到 Task 9 的授权方法；云能力不可用时展示“仍可保存本地计划”。

- [ ] **Step 5: 实现“我的”页**

展示目标入睡时间、提醒状态、历史趋势入口和“数据仅保存在当前设备”的说明。不要显示起床时间、睡眠时长或睡眠质量。

- [ ] **Step 6: 在微信开发者工具手工验证首页**

Run: 打开项目，依次执行“设置 23:00 → 首页打卡 → 修改到 00:15 → 撤销”。

Expected: 00:15 仍归入前一睡眠日；页面无起床打卡；重新打开小程序后本地状态保持。

- [ ] **Step 7: 提交首页与计划页**

```bash
git add miniprogram/app.wxss miniprogram/pages miniprogram/assets/icons
git commit -m "feat: build immersive home and bedtime plan"
```

### Task 5: 实现助眠声音目录和基础播放器

**Files:**
- Create: `miniprogram/shared/sounds.ts`
- Create: `miniprogram/infrastructure/audio-player.ts`
- Create: `miniprogram/pages/sounds/index.{json,ts,wxml,wxss}`
- Create: `miniprogram/pages/player/index.{json,ts,wxml,wxss}`
- Create: `docs/audio-asset-register.md`

- [ ] **Step 1: 建立可审计的音频资源清单**

`docs/audio-asset-register.md` 每条资源必须记录：显示名、来源 URL、作者、许可证、下载日期、云存储 fileID、是否允许商用和修改。未填写完整的资源不得进入 `sounds.ts`。

`sounds.ts` 使用固定结构：

```ts
export interface SleepSound {
  id: string
  name: string
  subtitle: string
  icon: string
  fileId: string
  color: string
}

export const SOUNDS: SleepSound[] = [
  { id: 'rain', name: '雨夜', subtitle: '细雨落在窗边', icon: '🌧', fileId: 'cloud://assets/audio/rain.mp3', color: '#5266A5' },
  { id: 'waves', name: '海浪', subtitle: '缓慢往复的潮汐', icon: '🌊', fileId: 'cloud://assets/audio/waves.mp3', color: '#477C9B' },
  { id: 'fire', name: '篝火', subtitle: '安静温暖的火苗', icon: '🔥', fileId: 'cloud://assets/audio/fire.mp3', color: '#8A5F52' },
  { id: 'stream', name: '溪流', subtitle: '清澈持续的流水', icon: '💧', fileId: 'cloud://assets/audio/stream.mp3', color: '#4D7892' },
  { id: 'forest', name: '森林', subtitle: '风穿过夜晚树林', icon: '🌲', fileId: 'cloud://assets/audio/forest.mp3', color: '#4F716A' },
  { id: 'fan', name: '风扇', subtitle: '均匀稳定的低噪', icon: '🌀', fileId: 'cloud://assets/audio/fan.mp3', color: '#68718F' }
]
```

部署时将示例 fileID 替换为审核通过资源的真实 fileID；开发阶段若 fileID 不存在，播放器必须进入可重试错误态。

- [ ] **Step 2: 实现单音轨播放器适配器**

`AudioPlayer` 内部只持有一个 `wx.InnerAudioContext`，提供 `load(src)`、`play()`、`pause()`、`setVolume(0..1)`、`setTimer(0|15|30|60)`、`destroy()`。`setTimer` 先清理旧计时器；计时到期调用 `pause()`。监听 `onError` 并通过订阅回调暴露 `{ status: 'error', message }`。

- [ ] **Step 3: 实现声音列表页**

以两列低亮度卡片展示 `SOUNDS`。点击卡片导航至 `/pages/player/index?id={{id}}`；不存在的 ID 显示 toast 并返回。

- [ ] **Step 4: 实现播放器页**

页面加载后用 `wx.cloud.getTempFileURL` 将 `fileId` 转换为临时 URL，再交给 `AudioPlayer.load`。页面包含播放/暂停、音量 slider、`15/30/60/关闭` 定时选项。`onUnload` 必须调用 `destroy()`；`onHide` 调用 `pause()`，明确不承诺后台播放。

- [ ] **Step 5: 实现加载与失败状态**

加载中禁用播放按钮；失败时显示“声音暂时无法加载”和“重试”按钮。重试重新获取临时 URL，不重建页面。

- [ ] **Step 6: 在开发者工具验证播放器**

Run: 使用一个真实云存储音频 fileID，测试播放、暂停、音量、15 分钟定时状态切换、隐藏小程序后暂停、无效 fileID 重试。

Expected: 同时只有一个音轨；错误不导致页面崩溃；离开页面后无残留计时器。

- [ ] **Step 7: 提交声音功能**

```bash
git add docs/audio-asset-register.md miniprogram/shared/sounds.ts miniprogram/infrastructure/audio-player.ts miniprogram/pages/sounds miniprogram/pages/player
git commit -m "feat: add sleep sounds and foreground player"
```

### Task 6: 实现三种呼吸模式和动画状态机

**Files:**
- Create: `tests/breathing.test.ts`
- Create: `miniprogram/domain/breathing.ts`
- Create: `miniprogram/pages/breathing/index.{json,ts,wxml,wxss}`

- [ ] **Step 1: 编写呼吸状态机失败测试**

```ts
import { expect, it } from 'vitest'
import { BREATHING_MODES, nextBreathingState } from '@/domain/breathing'

it('moves through inhale, hold and exhale', () => {
  const mode = BREATHING_MODES.relax
  expect(nextBreathingState(mode, { phase: 'inhale', cycle: 0 })).toEqual({ phase: 'hold', cycle: 0 })
  expect(nextBreathingState(mode, { phase: 'hold', cycle: 0 })).toEqual({ phase: 'exhale', cycle: 0 })
  expect(nextBreathingState(mode, { phase: 'exhale', cycle: 0 })).toEqual({ phase: 'inhale', cycle: 1 })
})
```

- [ ] **Step 2: 运行测试确认失败**

Run: `npx vitest run tests/breathing.test.ts`

Expected: FAIL because the breathing domain does not exist.

- [ ] **Step 3: 定义三个固定预设**

```ts
export const BREATHING_MODES = {
  relax: { id: 'relax', name: '放松', inhale: 4, hold: 4, exhale: 6, cycles: 8 },
  stress: { id: 'stress', name: '减压', inhale: 4, hold: 2, exhale: 6, cycles: 10 },
  calm: { id: 'calm', name: '快速平静', inhale: 3, hold: 2, exhale: 5, cycles: 6 },
} as const
```

实现 `nextBreathingState`、`phaseSeconds` 和 `isComplete`。状态只包含 `phase` 与 `cycle`，页面计时器负责每秒倒计时。

- [ ] **Step 4: 实现模式选择和引导动画**

未开始时显示三张模式卡。开始后圆球在吸气阶段放大到 `scale(1.35)`，停顿保持，呼气缩回 `scale(.72)`；CSS `transition-duration` 由当前阶段秒数绑定。页面显示“吸气 / 停住 / 呼气”和剩余秒数。

- [ ] **Step 5: 实现暂停、继续、完成和退出**

页面仅保留一个 `setInterval`；暂停时清理，继续时重建；`onUnload` 清理。完成后显示“练习完成”和返回首页按钮，不保存健康数据或完成度。

- [ ] **Step 6: 验证状态机与页面**

Run: `npm run verify`

Expected: breathing test PASS and typecheck PASS. 在开发者工具中分别开始三种模式，验证暂停/继续/退出无重复计时。

- [ ] **Step 7: 提交呼吸功能**

```bash
git add miniprogram/domain/breathing.ts miniprogram/pages/breathing tests/breathing.test.ts
git commit -m "feat: add guided breathing modes"
```

### Task 7: 实现 7/30 天趋势转换和 Canvas 折线图

**Files:**
- Create: `tests/trend.test.ts`
- Create: `miniprogram/domain/trend.ts`
- Create: `miniprogram/components/sleep-chart/index.{json,ts,wxml,wxss}`
- Create: `miniprogram/pages/history/index.{json,ts,wxml,wxss}`

- [ ] **Step 1: 编写趋势数据失败测试**

```ts
import { expect, it } from 'vitest'
import { buildTrend } from '@/domain/trend'

it('keeps missing days as null and maps midnight continuously', () => {
  const result = buildTrend({
    endDay: '2026-08-12',
    days: 3,
    records: [
      { sleepDay: '2026-08-10', sleptAt: new Date(2026, 7, 10, 23, 30).getTime(), createdAt: 1, updatedAt: 1 },
      { sleepDay: '2026-08-12', sleptAt: new Date(2026, 7, 13, 0, 30).getTime(), createdAt: 1, updatedAt: 1 },
    ],
    targetHistory: [{ effectiveSleepDay: '2026-08-01', time: '23:00' }],
  })
  expect(result.map(point => point.actualMinutes)).toEqual([1410, null, 1470])
  expect(result.map(point => point.targetMinutes)).toEqual([1380, 1380, 1380])
})
```

- [ ] **Step 2: 运行测试确认失败**

Run: `npx vitest run tests/trend.test.ts`

Expected: FAIL because `buildTrend` does not exist.

- [ ] **Step 3: 实现日期序列、目标分段和空点**

`buildTrend` 生成精确的连续睡眠日数组；无记录使用 `null`；目标时间按当天最后一个有效变更解析。`00:00–05:59` 的目标时间同样加 1440 分钟。导出 `formatTrendMinutes(1470) === '00:30'`。

- [ ] **Step 4: 实现 Canvas 2D 图表组件**

组件 properties 为 `points` 和 `height`。绘制顺序：水平时间网格、目标虚线、实际折线、实际点、X 轴日期。实际值为 `null` 时断开路径；纵轴固定覆盖 `18:00–30:00`，避免少量数据导致视觉夸大。使用设备 pixel ratio 缩放 canvas。

- [ ] **Step 5: 实现历史页面**

顶部切换 `7天/30天`，下方是图表和倒序记录列表。列表点击记录复用首页的修改/撤销逻辑；修改后重新计算趋势。无记录时显示“今晚睡觉时，记得点一下我睡觉了”。

- [ ] **Step 6: 运行趋势测试与视觉验证**

Run: `npm run verify`

Expected: all tests PASS. 在开发者工具注入 23:30、缺失日、00:30 三天数据，图表应跨午夜连续且缺失日断线，目标线为虚线。

- [ ] **Step 7: 提交趋势功能**

```bash
git add miniprogram/domain/trend.ts miniprogram/components/sleep-chart miniprogram/pages/history tests/trend.test.ts
git commit -m "feat: add bedtime history trend chart"
```

### Task 8: 实现订阅授权和云端提醒任务

**Files:**
- Create: `miniprogram/infrastructure/cloud-reminders.ts`
- Modify: `miniprogram/pages/sleep-plan/index.ts`
- Modify: `miniprogram/pages/profile/index.ts`
- Create: `cloudfunctions/scheduleReminder/{index.js,logic.js,package.json}`
- Create: `cloudfunctions/sendDueReminders/{index.js,logic.js,package.json}`
- Create: `cloudfunctions/sendDueReminders/logic.test.js`
- Create: `docs/cloud-reminders.md`

- [ ] **Step 1: 为到期任务筛选编写失败测试**

```js
const test = require('node:test')
const assert = require('node:assert/strict')
const { isDue } = require('./logic')

test('selects pending tasks due at or before now', () => {
  assert.equal(isDue({ status: 'pending', sendAt: 1000 }, 1000), true)
  assert.equal(isDue({ status: 'sent', sendAt: 900 }, 1000), false)
  assert.equal(isDue({ status: 'pending', sendAt: 1100 }, 1000), false)
})
```

- [ ] **Step 2: 运行云函数逻辑测试确认失败**

Run: `node --test cloudfunctions/sendDueReminders/logic.test.js`

Expected: FAIL because `logic.js` does not exist.

- [ ] **Step 3: 实现幂等的提醒任务模型**

`reminder_tasks` 文档字段固定为：`_openid`、`sleepDay`、`targetTime`、`sendAt`、`status: pending|sending|sent|failed`、`createdAt`、`updatedAt`、`error`。唯一业务键为 `_openid + sleepDay`；`scheduleReminder` 使用查询后更新/新增，重复设置同一天计划不会产生多条任务。

- [ ] **Step 4: 实现客户端订阅授权**

`CloudReminders.requestAndSchedule(targetTime, sleepDay)`：

1. 从编译配置读取订阅模板 ID；缺失时返回 `{ kind: 'unavailable' }`。
2. 调用 `wx.requestSubscribeMessage({ tmplIds: [templateId] })`。
3. 只有结果为 `accept` 时调用云函数 `scheduleReminder`。
4. `reject`、`ban`、云函数失败分别返回明确状态，页面只显示提示，不回滚本地目标时间。

- [ ] **Step 5: 实现定时发送云函数**

`sendDueReminders` 每 5 分钟触发一次：查询 `status=pending && sendAt<=now`，逐条先更新为 `sending`，调用 `cloud.openapi.subscribeMessage.send`，成功改为 `sent`，失败改为 `failed` 并保存脱敏错误信息。消息 data 的字段键从 `REMINDER_TIME_FIELD` 和 `REMINDER_THING_FIELD` 读取；模板 ID 从 `REMINDER_TEMPLATE_ID` 读取。

- [ ] **Step 6: 编写部署文档**

`docs/cloud-reminders.md` 明确：创建 `reminder_tasks` 集合、配置三个环境变量、部署两个云函数、给 `sendDueReminders` 添加每 5 分钟定时触发器、在客户端设置模板 ID、真机授权一次后检查数据库和发送结果。说明微信一次性订阅授权可能需要用户再次授权。

- [ ] **Step 7: 测试纯逻辑和云端降级**

Run: `node --test cloudfunctions/sendDueReminders/logic.test.js && npm run verify`

Expected: all tests PASS. 删除模板配置后，小程序仍能保存目标时间并显示“提醒暂不可用”。

- [ ] **Step 8: 提交云提醒功能**

```bash
git add cloudfunctions miniprogram/infrastructure/cloud-reminders.ts miniprogram/pages/sleep-plan miniprogram/pages/profile docs/cloud-reminders.md
git commit -m "feat: add subscription bedtime reminders"
```

### Task 9: 完成隐私文案、空状态和端到端验收

**Files:**
- Modify: `miniprogram/pages/home/*`
- Modify: `miniprogram/pages/sounds/*`
- Modify: `miniprogram/pages/player/*`
- Modify: `miniprogram/pages/breathing/*`
- Modify: `miniprogram/pages/history/*`
- Modify: `miniprogram/pages/profile/*`
- Create: `README.md`
- Create: `docs/acceptance-checklist.md`

- [ ] **Step 1: 统一非医疗产品文案**

搜索并移除“监测”“诊断”“治疗”“真实入睡”“睡眠质量评分”等表达。统一使用“睡觉打卡时间”“目标入睡时间”“放松练习”。在“我的”页加入：“本工具不提供医学诊断或治疗建议；持续失眠请咨询专业人士。”

- [ ] **Step 2: 检查所有空状态和失败状态**

逐页验证：未设置目标、无打卡、无趋势、音频加载失败、订阅拒绝、云函数不可用。每个状态必须有一个明确的下一步按钮，且不能阻塞其他本地功能。

- [ ] **Step 3: 编写 README**

README 包含产品截图位置、功能列表、技术栈、本地启动步骤、测试命令、微信 AppID 配置、云环境配置、音频授权登记要求和首版限制。明确仓库默认 `touristappid` 不能发送订阅消息。

- [ ] **Step 4: 编写验收清单**

`docs/acceptance-checklist.md` 包含以下可勾选场景：

- 两次点击内开始播放声音。
- 播放、暂停、音量、定时关闭有效。
- 三种呼吸模式可完成、暂停和退出。
- 目标时间可保存、修改，拒绝订阅后仍可使用。
- 23:30 与次日 00:30 归档正确，同一睡眠日不可重复。
- 打卡可修改和撤销。
- 7/30 天切换、缺失日断线、目标变更分段线正确。
- 清除本地数据后展示数据丢失说明。
- 小程序后台时不继续播放。

- [ ] **Step 5: 运行自动验证**

Run: `npm run verify && node --test cloudfunctions/sendDueReminders/logic.test.js`

Expected: TypeScript exits `0`; all Vitest and Node tests PASS.

- [ ] **Step 6: 在开发者工具和真机完成验收**

Run: 按 `docs/acceptance-checklist.md` 在微信开发者工具执行全部本地场景；在一台真机执行音频、订阅授权和提醒发送场景。

Expected: 清单全部通过；任何依赖真实模板或云环境的失败需记录具体配置错误，不能用模拟成功替代。

- [ ] **Step 7: 提交文档与验收修正**

```bash
git add README.md docs miniprogram
git commit -m "docs: add setup and acceptance guide"
```

### Task 10: 发布前验证与推送

**Files:**
- Modify: only files required by failed verification

- [ ] **Step 1: 检查工作区和敏感配置**

Run: `git status --short && rg -n "(secret|private[_-]?key|access[_-]?token|appid\s*[:=]\s*['\"][^t])" --glob '!package-lock.json' .`

Expected: 工作区仅包含预期变更；没有云密钥、私钥、访问令牌或真实敏感配置。公开 AppID 如需提交必须由用户明确确认。

- [ ] **Step 2: 运行完整自动验证**

Run: `npm run verify && node --test cloudfunctions/sendDueReminders/logic.test.js`

Expected: all commands exit `0`.

- [ ] **Step 3: 检查提交历史和远端差异**

Run: `git log --oneline --decorate -10 && git diff origin/main...HEAD --stat`

Expected: 每个功能任务对应一个聚焦提交；差异只包含小程序、云函数、测试与文档。

- [ ] **Step 4: 推送 main**

```bash
git push origin main
```

Expected: GitHub `main` 指向本地 `HEAD`，仓库页面展示 README 和完整源码。

## 规格覆盖检查

- 首页自由工具箱：Task 4。
- 6–8 种声音、单音轨、音量和定时关闭：Task 5。
- 三种呼吸模式和可退出动画：Task 6。
- 单一目标入睡时间和订阅提醒：Task 3、Task 4、Task 8。
- “我睡觉了”单次打卡、修改、撤销、跨午夜规则：Task 2、Task 3、Task 4。
- 7/30 天折线图、目标分段线和缺失断点：Task 7。
- 本地数据、云端仅提醒、不可用时降级：Task 2、Task 8。
- 深夜沉浸视觉、非医疗文案和首版边界：Task 4、Task 9。
- 自动测试、开发者工具和真机验收：Task 1–10。
