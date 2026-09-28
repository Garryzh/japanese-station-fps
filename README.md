# JAPANESE STATION — TACTICAL FPS

Three.js r170 + WebGL2 的第一人称射击 Demo。所有模型、贴图、HDR 环境光和音效都有程序化 fallback，**一个 `index.html` 就能直接玩**。以后可以把 GLB、HDR 和音频文件放进 `assets/`，直接替换掉对应的 fallback。

---

## 一、项目结构

```
japanese-station-fps/
├── index.html                  ← 游戏本体（HTML + CSS + JS 模块，约 3700 行）
├── README.md
└── assets/                     ← 可选的外部资源（缺了会自动用 fallback）
    ├── models/                 station.glb / platform.glb / train.glb / rifle.glb / pistol.glb /
    │                           shotgun.glb / enemy.glb / vending_machine.glb / convenience_store.glb /
    │                           props.glb / sakura.glb
    ├── hdr/                    station_night.hdr
    ├── textures/
    │   ├── floor/              basecolor.jpg / normal.jpg / roughness.jpg
    │   ├── concrete/           同上
    │   ├── metal/              同上
    │   └── wood/               同上
    └── audio/                  rifle.mp3 / pistol.mp3 / shotgun.mp3 / train_loop.mp3 … （完整列表见下）
```

## 二、技术栈

- Three.js r170（通过 CDN Import Map 加载，版本锁死在 0.170.0，不会被未来的更新改坏）
- WebGL 2、PBR（MeshStandard / MeshPhysical，透射玻璃，清漆）、PMREM 环境光、ACES Filmic 色调映射、sRGB 输出
- EffectComposer：RenderPass → 武器单独渲染 Pass（避免枪穿墙）→ UnrealBloomPass → 调色 / 暗角 / 胶片颗粒 / 色差 → OutputPass
- GLTFLoader + RGBELoader + LoadingManager + SkeletonUtils + AnimationMixer
- PointerLockControls、Raycaster、Box3 碰撞 + A* 导航图
- 粒子系统：THREE.Points + BufferGeometry + ShaderMaterial，全部使用对象池
- 音频：Web Audio API 实时合成全部音效 + THREE.PositionalAudio 3D 空间音频 + SpeechSynthesis 日语广播

## 三、运行方式

**方式 A（推荐）：用本地 HTTP 服务器**。想加载 `assets/` 里的 GLB、HDR 或音频就必须用这种方式：

```bash
cd japanese-station-fps
python -m http.server 8080
```

然后在浏览器打开 `http://localhost:8080`（建议用最新版 Chrome 或 Edge，并开启硬件加速）。

**方式 B：直接双击 `index.html`**。游戏可以正常运行，但浏览器的 `file://` 安全限制会拦截本地资源，所以只会使用程序化 fallback。

两种方式都需要联网，用来从 jsdelivr 加载 Three.js。

## 四、操作说明

| 按键 | 功能 | 按键 | 功能 |
|---|---|---|---|
| WASD | 移动 | SHIFT | 奔跑 |
| SPACE | 跳跃 | CTRL / C | 蹲下（建议用 C，避免 Ctrl+W 关掉标签页） |
| 鼠标左键 | 射击 | 鼠标右键 | 瞄准（ADS） |
| R | 换弹 | 1 / 2 / 3 / 滚轮 | 步枪 / 手枪 / 霰弹枪 |
| ESC | 解锁鼠标（暂停） | F3 或 ` | 调试面板（FPS / Draw Calls / Triangles） |
| M | 静音 | | |

任务目标：消灭 5 名敌人。完成后会显示 KILLS / ACCURACY / HEADSHOTS / TIME，并可以重新开始。开局约 14 秒后第一列电车进站，之后每 50 秒一班。

## 五、如何替换 GLB

把文件放进 `assets/models/`，文件名要和 `index.html` 里 `ASSETS.models` 的配置一致。

| 文件 | 替换内容 | 注意 |
|---|---|---|
| `station.glb` | 车站建筑外壳（地面、墙、天花板） | 单位为米，原点对齐场景中心，大厅范围约 X −32~32、Z −33~20。碰撞体会按每个 Mesh 的包围盒自动生成 |
| `platform.glb` | 站台边缘、轨道、站台门 | 同上 |
| `train.glb` | 单节车厢 | 长度方向为 X 轴，会自动缩放到 20m，并复制成 4 节 |
| `enemy.glb` | 敌人 | 会自动缩放到 1.8m 高，面朝 +Z。动画按名称自动匹配 idle / walk / run / attack(shoot) / hit / death / reload |
| `vending_machine.glb` / `convenience_store.glb` / `sakura.glb` / `props.glb` | 对应道具 | 自动缩放并摆放 |

加载后会自动开启 castShadow 和 receiveShadow，把非 PBR 材质升级为 MeshStandardMaterial，修正贴图的颜色空间，检测动画和骨骼，并缓存模型。

## 六、如何替换 HDR

把 `.hdr` 文件放到 `assets/hdr/station_night.hdr`。它会经过 PMREMGenerator 处理后写入 `scene.environment`，只作为 PBR 反射和环境照明使用，不会设为背景。如果太亮，调 `index.html` 里的 `this.scene.environmentIntensity = 0.6`，或者调 `R.toneMappingExposure`。可以用 Poly Haven 上的 CC0 夜景 HDRI，1k 或 2k 分辨率就够了。

## 七、如何替换枪械

把 `rifle.glb` / `pistol.glb` / `shotgun.glb` 放进 `assets/models/`。如果模型朝向或大小不对，改 `index.html` 顶部的 `WEAPON_GLB_FIX`：

```js
const WEAPON_GLB_FIX = {
  rifle: { length: 0.86, rotation: [0, 0, 0], offset: [0, 0, 0] },  // 枪口需朝 -Z
  ...
};
```

- 模型里名称包含 `muzzle` 或 `flash` 的节点会被当作枪口；如果没有，就取包围盒最前端。
- 名称包含 `mag` 的节点会参与换弹动画。

## 八、如何替换音效

把文件放进 `assets/audio/`，文件名如下，mp3、ogg、wav 都可以，但扩展名要和配置里写的一致：

```
rifle.mp3  pistol.mp3  shotgun.mp3  enemy_shot.mp3  reload_out.mp3  reload_in.mp3  bolt.mp3
impact_metal.mp3  impact_concrete.mp3  impact_wood.mp3  impact_glass.mp3
train_loop.mp3  brake.mp3  station_ambience.mp3  announce.mp3  departure_melody.mp3  store_bgm.mp3  footstep.mp3
```

没提供的音效会继续用 Web Audio 实时合成。日语广播优先用系统自带的日语 TTS：Windows 需要在 设置 → 时间和语言 → 语音 里添加日语语音。如果没有日语语音，会播放合成的广播音效。

## 九、如何调整画质

开始菜单和暂停菜单里都可以选 LOW / MEDIUM / HIGH / ULTRA，选择会被记住。具体参数在 `index.html` 的 `QUALITY` 配置里：

| 档位 | 内容 |
|---|---|
| LOW | 关闭阴影、Bloom、MSAA，减少粒子，玻璃不使用透射，像素比 0.85 |
| MEDIUM | 1024 阴影、Bloom、2x MSAA |
| HIGH | 2048 软阴影、Bloom、4x MSAA、RectAreaLight 站台灯、透射玻璃、色差 |
| ULTRA | 4096 阴影、地面强清漆反射（SSR 视觉近似）、最多樱花粒子、像素比最高 2.0 |

## 十、如何修改敌人数量

改 `index.html` 顶部的 `GAME.enemyCount`（1–8）。同一个配置里还可以调：

- `enemyHP`、`enemyDamage`
- `enemyVision`（视野距离）、`enemyFOV`、`enemyHearing`（听觉距离）、`enemyAttackRange`

出生点和巡逻路线在 `EnemyManager` 构造函数的 `S` 数组里。

## 十一、如何修改武器伤害

改 `index.html` 顶部的 `WEAPONS` 配置：

```js
rifle:   { damage:34, headMult:3.0, rpm:680, mag:30, reserve:120, spread:0.010, recoil:0.016, ... },
pistol:  { damage:42, headMult:2.6, rpm:400, mag:12, ... },
shotgun: { damage:16, headMult:1.8, pellets:9, spread:0.055, ... },
```
