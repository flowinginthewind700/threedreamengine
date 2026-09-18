# ThreeDream architecture

The [README](../README.md) states what the engine is and what it measures. This
document is the long form: how the layers are cut, what each solver promises,
how the browser demos are tested, and how an Unreal 5 project's own files become
a page. Everything here is implemented in this repository; where a paragraph
names a file, that file is the specification it describes.

[README](../README.md) 说明这台引擎是什么、量到了什么；本页是长文：层怎么切、每个求解器
承诺什么、浏览器演示怎么测、一个 Unreal 5 工程自己的文件如何变成一个页面。文中点到的
每个文件，就是它所描述那份规范的实现。

## Two rules

1. **The simulation is the only source of truth.** The renderer mirrors physics
   state into `THREE.Object3D`s each frame and never writes back. The same scene
   therefore runs headless for training and rendered for play, with identical
   results.
2. **Everything that matters is deterministic.** A seeded RNG, a fixed timestep,
   and no hidden global state mean `engine.step(n)` in Node and `engine.frame(dt)`
   in a browser produce the same simulation. That is what makes a physics-AI
   kernel testable: all 2616 unit tests run without a GPU, and the wasm backend is
   held to the bits of the TypeScript solver it ports. The two GPU layers are held
   to a CPU reference the same way, and neither claims determinism for itself:
   `deterministic` is false on both backends, because atomics promise no order, so
   training and replay always run on the reference tier.

1. **仿真是唯一事实来源。** 渲染层每帧把物理状态镜像到 `THREE.Object3D`，从不回写。
   于是同一个场景既能无头训练，也能在浏览器里渲染游玩，结果完全一致。
2. **关键路径都是确定性的。** 带种子的 RNG、固定步长、没有隐藏的全局状态，所以 Node
   里的 `engine.step(n)` 与浏览器里的 `engine.frame(dt)` 跑的是同一个仿真。这让一个
   物理-AI 内核变得可测：2616 个单元测试全都不需要 GPU，而 wasm 后端要对齐它所移植的
   TS 求解器的每一个比特。两个 GPU 层用同样的方式对齐一份 CPU 参照，而且都不替自己
   声称确定性：两个后端的 `deterministic` 都是 false，因为原子操作不承诺顺序，所以
   训练与回放永远走参照档。

## Layers

```
core     clock / ECS / events / engine facade     no three.js, no WASM
physics  backend interface + three solvers        no three.js
gpu      probe, shared device, scale layers       no three.js, no WASM
ai       MLP, Gaussian policy, policy-gradient    no three.js, no WASM
envs     learning tasks (drive, reach, pursuit)   physics only
render   three.js bridge + particle/soft views    the only layer importing three
assets   glTF, RGBE, glb, ue/ package reader      no three.js, no WASM
```

`src/index.ts` re-exports everything **except** `render`. Importing the barrel
must not pull three.js into a training script, so browser code imports
`ThreeRenderer` directly. Deliberate constraint, not an oversight. The two WASM
backends are in the barrel and still cost nothing to import: each loads its
binary through a dynamic `import()` inside its own factory, so a headless run
never instantiates a module it did not ask for.

`gpu/` is the layer that decides how good a browser's graphics stack is, and it
is a probe, not a build flag: `selectRenderTier` takes adapter results as
arguments and returns `webgpu`, `webgl2` or `cpu`. Nothing in it reads
`import.meta.env`, so the fallback cannot be baked in at compile time.
The same layer owns the shared `GPUDevice` (`device.ts`, refcounted, with a
recovery path for device loss), the wrapper that lets external WGSL bind
three.js's own buffers (`compute.ts`), and the two scale layers: particles and
soft bodies. Each of those has a CPU reference implementation beside the GPU
one, so the WGSL can be checked against numbers rather than against a
screenshot.

`src/index.ts` 重导出**除** `render` **之外**的全部内容。引入这个入口不能把 three.js
带进训练脚本，所以浏览器代码直接引 `ThreeRenderer`。这是刻意的约束，不是遗漏。两个
WASM 后端**在**这个入口里，引入它们依然不花代价：各自的工厂函数内部用动态
`import()` 加载二进制，所以无头运行不会实例化它没要过的模块。

`gpu/` 是判断浏览器图形栈成色的那一层，而且是探测，不是构建开关：
`selectRenderTier` 接收 adapter 探测结果作为参数，返回 `webgpu`、`webgl2` 或
`cpu`。它内部不读 `import.meta.env`，所以回退路径不可能在编译期被写死。
这一层同时持有共享的 `GPUDevice`（`device.ts`，引用计数，带设备丢失后的恢复路径）、
让外部 WGSL 直接绑定 three.js 自己那些 buffer 的封装（`compute.ts`），以及那两层规模层
—— 粒子与软体。两者都在 GPU 实现旁边放着一份 CPU 参照实现，所以 WGSL 要对齐的是一组
数字，而不是一张截图。

### The scale layers

Neither GPU layer is a fast path with a CPU stub. `particleCpu.ts` sits beside
`particleGpu.ts` and `softCpu.ts` beside `softGpu.ts`, each pair sharing one
layout module and one set of constants, and the soft pair its uniform packer
too, so the WGSL is asserted against numbers. `deterministic` is false on both
GPU backends: atomics promise no order and a driver may contract `a * b + c`
into an fma, so training and replay stay on the reference tier. What the
soft-body layer can claim instead is `raceFree`, and getting there took two
graph passes.

两个 GPU 层都不是「快路径配一个 CPU 桩」。`particleCpu.ts` 与 `particleGpu.ts` 并排，
`softCpu.ts` 与 `softGpu.ts` 并排，每对读的是同一个 layout 模块与同一套常量（软体那对
还共用同一个 uniform 打包器），于是 WGSL 要对齐的是一组数字。两个 GPU 后端的
`deterministic` 都是 false：原子操作不承诺顺序，驱动也可能把 `a * b + c` 收缩成 fma，
所以训练与回放留在参照档。软体层能声称的是 `raceFree`，而为了它多出了两趟图计算。

`softIslands.ts` is a union-find that also emits the node order the kernels
dispatch against: each island's nodes consecutively, padded out to a multiple of
the 64-lane workgroup, so "is this island asleep" costs one load per workgroup
rather than one per node. `softColoring.ts` then colors the constraint graph
first-fit in ascending edge order: one u32 mask per node, `MAX_COLORS = 32`,
and a graph that would need more is refused rather than silently mis-colored. A
color is a set of constraints that share no node, so every write inside one
batch lands on a distinct node and the whole solve is race-free without a single
atomic. Both passes run identically on the two tiers, and `tests/soft_gpu.test.ts`
asserts the resulting `SoftPlan` field for field: that equality, not the
naming, is what "island grouping and constraint coloring pass a determinism
check" means.

`softIslands.ts` 是 union-find，同时产出 kernel 实际 dispatch 用的节点顺序：每个
island 的节点连续排布，并补齐到 64 lane workgroup 的倍数，于是「这个 island 睡着了
吗」是每个 workgroup 一次 load，而不是每个节点一次。`softColoring.ts` 再按边的升序做
first-fit 着色 —— 一个节点一个 u32 掩码，上限 `MAX_COLORS = 32`，需要更多颜色的图会被
拒绝，而不是被悄悄错着色。一个 color 是一组互不共享节点的约束，所以同一批里的每次写入
都落在不同节点上，整个求解无竞争，而且不用一个原子操作。两趟在两档上的算法完全相同，
`tests/soft_gpu.test.ts` 逐字段断言产出的 `SoftPlan` —— 「island 分组与约束着色通过
确定性对照」说的是这个相等，不是命名。

A step is `5 + iterations * colors` dispatches: 69 for a cloth at 8 iterations
and 8 colors, a number that follows the graph's maximum degree and not the node
count. `publish` writes into a buffer three.js already allocated, and
`render/soft.ts` blits it straight into the mesh's position attribute: a plain
`BufferAttribute(itemSize=3)`, neither a storage attribute nor
`DynamicDrawUsage`, because the first would break the byte-for-byte
correspondence and the second would overwrite it with a stale CPU array every
frame. 10k nodes cost 1.40 MiB and 2.3-3.2 ms/step, 20k cost 2.81 MiB and
3.9-6.3, and 1k costs 0.7-1.6: the ladder is about 0.7 ms of fixed cost (69
dispatch submissions and a queue flush, which the node count does not move)
plus roughly 0.16 us a node, so the bottom rung is nearly all submission and
the top one is mostly solve. Each number is the median of the 20 timed chunks a
160-step rung produces, printed with its p95 beside it, because the mean over
the four chunks a 30-step rung used to produce was a number one contended chunk
could triple: the same rung on the same machine read 2.97 and then 10.07
ms/step four minutes apart, in a run whose 20k rung came out *faster* than its
10k one. Those are the numbers `scripts/bench_gpu_soft.mjs` exists to print,
and they are a floor in device terms (headless Chromium hands `requestAdapter()`
whichever GPU the driver stack prefers, which on a laptop with no Vulkan ICD for
the discrete card is the integrated one) and not a floor from one run to the
next, which is what the p95 column is there to show.

一步是 `5 + iterations * colors` 个 dispatch：8 次迭代、8 个 color 的布料是 69 个，
这个数字跟着图的最大度数走，不跟节点数走。`publish` 写进 three.js 已经分配好的那块
buffer，`render/soft.ts` 把它直接 blit 进 mesh 的 position attribute —— 普通的
`BufferAttribute(itemSize=3)`，既不是 storage attribute 也不是 `DynamicDrawUsage`，
因为前者会破坏逐字节对应，后者会每帧用陈旧的 CPU 数组把刚 blit 进去的位置盖掉。
1 万节点 1.40 MiB、2.3–3.2 ms/步，2 万节点 2.81 MiB、3.9–6.3 ms/步，1k 是
0.7–1.6 ms/步：整条阶梯约等于 0.7 ms 的固定开销（69 次 dispatch 提交加一次队列
flush，节点数推不动它）加上每节点约 0.16 us，所以底部那一档几乎全是提交，顶部那一档
才主要是求解。每个数字都是 160 步一档产出的 20 个计时 chunk 的中位数，旁边同时打印
p95 —— 因为以前 30 步一档只有四个 chunk，而四个样本的均值是一个被抢占的 chunk 就能
翻三倍的数字：同一档在同一台机器上相隔四分钟分别读到 2.97 与 10.07 ms/步，那一轮里
20k 还比 10k「更快」。这些正是 `scripts/bench_gpu_soft.mjs` 要打印的数字，它们在设备
意义上是下限（无头 Chromium 会把 `requestAdapter()` 交给驱动栈偏好的那块 GPU，在一台
独显没有 Vulkan ICD 的笔记本上，那就是集显），但在轮次与轮次之间不是 —— 而那正是 p95
那一列要显示的东西。

The tier below that one has a number too, because "the fallback works" and "the
fallback is usable at scale" are two different claims and only the second one is
an acceptance criterion. `scripts/bench_cpu_soft.ts` runs the same ladder, the
same scene, seed and iteration count through `src/gpu/softCpu.ts`, which is the
shipped fallback rather than a restatement of it, and the same code the `webgl2`
and `cpu` tiers both simulate on. It reads 1.5 / 7.9 / 15.9 / 32.1 ms/step at
1k / 5k / 10k / 20k nodes: about 1.6 us a node with no fixed cost worth naming,
roughly ten times the device tier's marginal 0.16 us. So 10k nodes cost a whole
60 Hz frame before anything is drawn, and the fallback's stated target is ~2k
nodes at 60 Hz (about 3.2 ms/step, which leaves the rest of the frame for
drawing and for the engine) rather than the 20k the device tier holds. Its p95
sits within a percent of its p50, and printing both is the point: with no driver
submission and no queue flush inside the timed region, that spread is the
machine's own noise floor, so whatever is wider than it in the GPU table above
belongs to the device path.

下面那一档同样有数字，因为「回退能用」与「回退在规模上可用」是两条不同的结论，而只有
第二条是验收项。`scripts/bench_cpu_soft.ts` 用同一条阶梯、同一个场景、同一个 seed 与
同样的迭代次数跑 `src/gpu/softCpu.ts`：那是实际交付的回退路径，不是它的复述，也正是
`webgl2` 与 `cpu` 两档共用的那份仿真代码。它在 1k / 5k / 10k / 20k 节点上读到
1.5 / 7.9 / 15.9 / 32.1 ms/步，每节点约 1.6 us，固定开销小到不值得命名，大约是设备档
每节点 0.16 us 的十倍。于是 1 万节点在还没开始画之前就已经吃掉一整个 60 Hz 帧，回退档
给出的明确目标是 60 Hz 下约 2k 节点（约 3.2 ms/步，把这一帧剩下的预算留给绘制与引擎
其余部分），而不是设备档站得住的 2 万。它的 p95 与 p50 相差在百分之一以内，而把两个都
打出来正是重点：计时区间里没有驱动提交也没有队列 flush，这个离散度就是机器本身的噪声
底，所以上面那张 GPU 表里比它更宽的部分，属于设备路径。

The most expensive lesson in the layer is one the unit tests could not catch.
The color selector originally read `global_invocation_id.z`, on the theory that
a dispatch dimension is a free channel into the kernel. It is not: dispatch
dimensions are concurrent rather than sequential, so `(x, 1, color + 1)` runs
every color at once and re-creates exactly the race the coloring exists to
remove, while the last color never solves at all. The shader compiled, the
dispatches succeeded, the mesh moved, and the unit tests drive the CPU
reference, so no tolerance ever complained. What caught it was a CPU/GPU
comparison on a real device, which is why `e2e/soft_gpu.spec.ts` exists. The
selector now hangs off a binding: each color owns one 256-byte slot of
`batchBuf` and sees only its own 8 bytes, so the kernel reads `batchBuf[0u]`
and cannot index a neighbour's slot even if the shader text is wrong.

这一层最贵的一课，单元测试抓不到。color 选择器原本读 `global_invocation_id.z`，理由是
「dispatch 维度是通往 kernel 的一条免费通道」。它不是：dispatch 的维度是并发的，不是
顺序的，于是 `(x, 1, color + 1)` 会同时跑所有 color，把着色本该消除的那个竞争原样重建
出来，而最后一个 color 根本不会被解。shader 编译通过、dispatch 成功、网格在动，单元
测试驱动的是 CPU 参照，所以没有任何容差会报警。抓到它的是真设备上的一次 CPU/GPU 对照
—— 这正是 `e2e/soft_gpu.spec.ts` 存在的理由。现在选择器挂在 binding 上：每个 color
独占 `batchBuf` 里一个 256 字节的槽位，只看得见自己那 8 字节，于是 kernel 读
`batchBuf[0u]`，即使 shader 文本写错也索引不到邻居的槽位。

### Physics backends

All three implement `PhysicsBackend`, so envs and the trainer stay agnostic:

| Backend | File | Notes |
|---|---|---|
| `builtin` | `src/physics/builtin.ts` | Pure TS, deterministic, zero dependencies. Default, and the normative spec. |
| `wasm` | `src/physics/wasm.ts` | First-party Rust kernel. Bit-identical to `builtin`, 2.6x-4.9x faster in Node. |
| `rapier` | `src/physics/rapier.ts` | Third-party Rapier WASM. Same interface, different solver: ~3% off steady-state speeds, and different trajectories. |

`wasm` is the backend whose claim is unusual. Not "same interface" but "same
bits": `src/physics/reference.ts` defines one canonical scene plus a digest over
raw IEEE-754 bit patterns, and every runner has to reproduce
`REFERENCE_GOLDEN_DIGEST` in Node, in a browser, and on a replay of the same
seed. Rounding the digest inputs would hide a one-ulp divergence, which is the
only kind of divergence a determinism gate exists to catch. The Rust side is
three crates under `rust/`: the solver, with no wasm-bindgen dependency, so
`cargo test` runs it natively; a cdylib that is nothing but the ABI; and a wgpu
skeleton for the GPU milestones. `rapier` stays as a cross-check against a
solver nobody here controls.

Swapping backends has caught real bugs: Rapier's `RigidBodyDesc.setAdditionalMass`
is lazy, folded in only at the next `world.step()`, so a first-frame impulse used
to land on a body roughly 400x too light. `tests/rapier_backend.test.ts` asserts
both the impulse-on-first-step behaviour and the cross-backend agreement.

三个后端都实现 `PhysicsBackend`，因此 env 与 trainer 与之无关：

| 后端 | 文件 | 说明 |
|---|---|---|
| `builtin` | `src/physics/builtin.ts` | 纯 TS、确定性、零依赖。默认，也是规范实现。 |
| `wasm` | `src/physics/wasm.ts` | 自研 Rust 内核。与 `builtin` 逐位一致，Node 下快 2.6x-4.9x。 |
| `rapier` | `src/physics/rapier.ts` | 第三方 Rapier WASM。接口相同，求解器不同：稳态速度差约 3%，轨迹也不同。 |

`wasm` 是承诺最特殊的那个后端。不是「接口相同」，而是「比特相同」：
`src/physics/reference.ts` 定义一个规范场景，外加一个基于 IEEE-754 原始比特模式的
摘要，所有运行方都必须复现 `REFERENCE_GOLDEN_DIGEST` —— 在 Node 里、在浏览器里、
以及同一 seed 的回放里。对摘要输入做舍入会掩盖 1 ulp 的偏差，而那正是确定性关卡
唯一要抓的偏差类型。Rust 侧是 `rust/` 下的三个 crate：不依赖 wasm-bindgen 的求解器
（因此 `cargo test` 能原生跑它）、只负责 ABI 的 cdylib，以及为 GPU 里程碑准备的
wgpu 骨架。`rapier` 保留下来，作为对「这里没人能控制的求解器」的交叉验证。

换后端真抓到过 bug：Rapier 的 `RigidBodyDesc.setAdditionalMass` 是惰性的，要到下一次
`world.step()` 才生效，于是第一帧的冲量曾经打在一个轻了约 400 倍的刚体上。
`tests/rapier_backend.test.ts` 同时断言「首帧冲量」行为和两个后端的一致性。

### The learner

`src/ai/trainer.ts` is a batched policy-gradient method with GAE advantages and a
learned critic. Deliberately not PPO: no importance ratios, no replay buffer, so
the whole update is readable end to end while still being a real policy-gradient
method. Its header comment records three design decisions that were each forced by
a measured failure, because those failures are silent: entropy stays healthy,
training returns look plausible, and greedy evaluation simply never improves.

`src/ai/trainer.ts` 是带 GAE 优势估计与学习式 critic 的批量策略梯度方法。刻意不做
PPO：没有重要性采样比，也没有 replay buffer，整个更新过程可以从头读到尾，同时仍然是
一个真正的策略梯度方法。文件头注释记录了三个「各自被一次实测失败逼出来」的设计决定，
因为这类失败是无声的：熵保持健康、训练回报看着合理，但贪婪评估就是不涨。

Environments follow a gymnasium-style contract (`reset` -> `step` -> `observe`)
over typed fixed-size buffers, because allocation inside a training loop starts to
dominate cost once episodes run in the thousands.

环境遵循 gymnasium 风格的契约（`reset` -> `step` -> `observe`），使用带类型的定长缓冲
区，因为一旦 episode 上千，训练循环里的内存分配就开始主导开销。

| Env | Task | Obs | Act |
|---|---|---|---|
| `DriveEnv` | Steer a body to a randomised goal under real dynamics | 6 | 2 |
| `ReachEnv` | Push a puck into a goal through contact | 14 | 2 |
| `PursuitEnv` | Close on a moving target in the FPS arena | arena | aim + move |

`ReachEnv`'s 14 observations are 6 planar positions (agent / puck / goal), 4 unit
direction vectors (agent->puck, puck->goal) and 4 planar velocities. Contact tasks
are second-order: knowing where the puck is without knowing which way it is already
sliding leaves the policy a step behind. Positions are normalised by arena size so
the policy transfers across scales.

`ReachEnv` 的 14 维观测是 6 个平面位置（agent / puck / goal）、4 个单位方向向量
（agent->puck、puck->goal）和 4 个平面速度。接触类任务是二阶的：只知道 puck 在哪而不
知道它正往哪滑，策略就永远慢一拍。位置按场地尺寸归一化，所以策略能跨尺度迁移。

## Headless training

```bash
npm run train
npm run train -- --episodes 4000 --seed 3 --out artifacts/policy.json
npm run train -- --help
```

Flags: `--episodes N` `--seed N` `--lr N` `--batch N` `--hidden a,b` `--every N`
`--out PATH`. The script trains `ReachEnv`; the same loop drives the browser demo,
so a policy that trains here plays there.

参数：`--episodes N` `--seed N` `--lr N` `--batch N` `--hidden a,b` `--every N`
`--out PATH`。该脚本训练 `ReachEnv`；同一个循环也在驱动浏览器演示，所以这里训练出来
的策略就能在那里玩。

## Writing your own task

```ts
import { createEngine, createBuiltinPhysics, vec3 } from './src/index.js';

const physics = createBuiltinPhysics({ gravity: vec3(0, -9.81, 0) });
const engine = createEngine({ physics });

engine.addGround();
const box = physics.createBody({
  shape: { kind: 'box', halfExtents: vec3(0.25, 0.25, 0.25) },
  position: vec3(0, 3, 0),
  mass: 1,
  restitution: 0.3,
});

engine.step(120); // two seconds at 60 Hz
console.log(physics.getBodyState(box)!.position); // -> [0, 0.246, 0]
```

Implement `LearningEnvironment` over such a world and the trainer learns it with no
changes. `src/envs/drive.ts` is the shortest useful example.

在这样的世界上实现 `LearningEnvironment`，trainer 无需任何改动就能学它。
`src/envs/drive.ts` 是最短且有用的示例。

## Testing and the gate

The project is developed test-first: a spec exists for every module under `src/`,
and `tests/tdd.test.ts` fails the build the moment a new module lands without one.

| Command | What it runs | Cost |
|---|---|---|
| `npm run gate` | every gate below in one serial, fail-fast run; `gate:fast` drops the coverage pass and the wasm rebuild | ~4 min / ~3.5 min |
| `npm test` | 2616 unit tests in 93 files, headless, no GPU needed | ~28s |
| `npm run test:coverage` | same suite under v8, floor enforced by `vitest.config.ts` | ~31s |
| `npm run test:e2e` | 65 Playwright tests over 8 specs, two projects: SwiftShader WebGL2 and ANGLE/Vulkan WebGPU | ~2.8 min |
| `npm run test:rust` | 68 native Rust tests for the solver | ~1s warm |
| `npm run check:wasm` | assertions over the shipped wasm kernel: ABI, provenance, behaviour | ~1s |
| `node scripts/bench_gpu_particles.mjs` | the M3 ladder at 1k/10k/50k/100k particles, 160 steps a rung: per-step cost as p50/p95/mean over 20 chunk samples, draw calls, blit size | ~6s |
| `node scripts/bench_gpu_soft.mjs` | the M4 ladder at 1k/5k/10k/20k nodes, 160 steps a rung: the same distribution, plus dispatches, colors and stretch | ~3s |
| `npx tsx scripts/bench_cpu_soft.ts` | the same M4 ladder on the fallback tier: `softCpu.ts`, single-threaded, no GPU, p50/p95 a step at 1k-20k nodes and a fitted us/node | ~10s |
| `npm run diagrams` / `npm run diagrams:check` | rasterise `docs/diagrams/*.svg` to the committed PNGs, or gate them against the manifest's hashes | ~5s / ~0s |
| `node scripts/capture_shots.mjs` | the screenshots in the README and the share cards: 12 captures of the built pages served at their deployment subpath, each gated on the page's own tier report | ~40s |

The two e2e projects exist because the pages need two different GPUs. `demo`,
`physics-check`, `particles` and `soft` only need *a* GL context, and headless
Chromium has none, so they run on SwiftShader: that the render layer works
without hardware is the property worth testing. The GPU halves of `particles`
and `soft` need a real `GPUDevice`, which means ANGLE's Vulkan backend with the
WebGPU service enabled. Same browser, different flags, and a flag set that works
for one silently downgrades the other. `shared-device` runs under both: with an
adapter it gates the three claims, without one it records what the page owes a
reader whose browser has no WebGPU at all: the claims left at `not run` rather
than painted as failures, and the fallback tier still presenting. The tests that
assume *no* adapter skip themselves on a machine that has one, rather than
reporting a tier the machine did not produce.

Coverage is a separate command, not a flag on `npm test`: instrumenting the
training inner loop costs ~10x wall time, and folding that into the fast loop
would destroy the red/green cadence the tests exist to provide. So the coverage
run skips the two convergence tests (`COVERAGE=1`, `tests/trainer.test.ts`) that
cost 180s of the 185s and contribute ~0.3% of branch coverage, plus the one
soft-body scale spec (`tests/soft_cpu.test.ts`) whose assertion is a wall clock
that instrumented code cannot meet: `npm test` still runs all three, and
`tests/tdd.test.ts` asserts that asymmetry. The floor sits a few points under
what the suite actually measures (93/82/93/94 against 98.7/93.5/98.9/99.1), which
is the part that matters: a threshold set far below current reality is
decoration, while one set at it makes every refactor a fight.

`.github/workflows/ci.yml` is one file with five jobs: `verify` (typecheck +
tests + diagram gate + build), `coverage`, `e2e` and `rust` run in parallel as
gates, and `deploy` depends on all four. A second workflow would hand out a
shippable bundle from a red `main`, because Actions cannot express `needs:` across
workflow files, so the dependency lives in one graph, and
`tests/ci_workflow.test.ts` asserts it (along with the SHA-pinning of every
action and the fact that no job holds a publish permission).

Who runs that graph is a separate fact from what it checks, and both are worth
stating. `npm run gate` runs the same commands in one serial, fail-fast order,
and `tests/gate.test.ts` pins the workflow's list and the gate's to each other in
both directions, so a check added to one and forgotten in the other fails the
unit suite. What a runner installs and a working copy already has (`npm ci`, the
pinned Rust toolchain, Playwright's Chromium) is a precondition that prints its
own fix, and two steps write into `.gate-tmp/` rather than the tree: the diagram
render, because re-rasterising over `docs/diagrams/` would replace committed PNGs
with this machine's fonts, and the wasm snapshot, because the rebuild that
follows overwrites `wasm/pkg`. A full run is about four minutes with warm
caches, and the wasm rebuild has to come back byte-identical to the committed
artifact.

Shipping is then a hand's work rather than a job's: after a green gate,
`npm run build:inpage -- --host <checkout>` writes the bundle into the host
site's static tree under `/threedream/app/`, and `scripts/publish_assets.ts`
puts the bulk bytes on the asset mirror. Nothing publishes itself from a push;
`deploy` still states the contract for whoever has a runner to give it -- build
the bundle into a scratch host, upload it, and only behind all four gates.

The `rust` gate exists because the wasm kernel ships as bytes in the repository,
and no other job can tell whether those bytes still match `rust/`: TypeScript
checks `src/physics/wasm.ts` against the `WasmBindings` interface it declares,
not against the binary. So the job runs the native solver tests, builds the wgpu
crate for `wasm32` (the M0 claim that the pinned version still compiles), checks
the committed artifact, then rebuilds it and requires identical bytes. wasm-pack
comes from a checksummed tarball rather than `curl | sh`: that installer is a
mutable ref on a third-party repo, and this is the tool minting the binary the
determinism claim rests on.

Byte-exactness needs one more thing pinned, and it is not in `Cargo.toml`:
wasm-pack fetches the `wasm-bindgen` *CLI* itself, and a prebuilt release asset
stamps the artifact's `producers` section with the tag's git hash where a
`cargo install` fallback stamps the bare version. Same CLI version, identical
behaviour, 12 different bytes, which the freshness check cannot tell apart from
real drift. `scripts/check_wasm_artifact.mjs` therefore pins the stamp
(`WASM_BINDGEN_CLI`), so a source-built CLI fails provenance with the reason
printed instead of turning up as an unexplained hash mismatch.

本项目采用测试先行：`src/` 下每个模块都有对应 spec，一旦出现没有测试的新模块，
`tests/tdd.test.ts` 会让构建失败。

覆盖率是独立命令而非 `npm test` 的参数：给训练内层循环插桩会让墙钟时间涨约 10 倍，
把它塞进快速循环会毁掉测试本该提供的红/绿节奏。因此覆盖率运行会跳过那两个收敛测试
（`COVERAGE=1`，`tests/trainer.test.ts`）—— 它们占了 185s 里的 180s，却只贡献约 0.3%
的分支覆盖率 —— 外加软体那一条规模 spec（`tests/soft_cpu.test.ts`），它断言的是一个
插桩后的代码无法满足的墙钟；`npm test` 三条照跑，这条不对称由 `tests/tdd.test.ts`
保证。下限压在实测值之下几个点（实测 98.7 / 93.5 / 98.9 / 99.1，下限 93 / 82 / 93 /
94）—— 这才是关键：远低于现状的阈值只是装饰，而贴着现状设阈值则会让每次重构都变成
搏斗。

`.github/workflows/ci.yml` 是单文件五任务：`verify`（类型检查 + 测试 + 图关卡 +
构建）、`coverage`、`e2e`、`rust` 四个关卡并行，`deploy` 依赖这四者。单独的
第二个工作流会在关卡变红时仍然交出可发布产物，因为 Actions 无法跨工作流文件表达
`needs:`，所以依赖关系收敛在一个图里，并由 `tests/ci_workflow.test.ts` 断言（连同
每个 action 的 SHA 锁定、以及没有任何任务持有发布权限这一事实）。

谁来跑这张图，与它检查什么是两件不同的事，而两件都值得说清楚。`npm run gate` 以串行、
失败即停的顺序跑同一批命令，`tests/gate.test.ts` 把 workflow 的清单与 gate 的清单双向
锁定：一处加了、另一处漏掉的那条检查会让单元测试失败。runner 需要安装、而工作副本本来
就有的东西（`npm ci`、锁定的 Rust 工具链、Playwright 的 Chromium）是前置条件，报错信息
里直接给出修复命令；有两步写进 `.gate-tmp/` 而不是仓库树：图的渲染（重新栅格化到
`docs/diagrams/` 会用本机字体替换已提交的 PNG），以及 wasm 快照（紧随其后的重建会覆盖
`wasm/pkg`）。缓存已热时跑完一遍约 4 分钟，而那次 wasm 重建必须与已提交产物逐字节一致。

交付于是由人来做，而不是由 job 来做：关卡全绿之后，`npm run build:inpage -- --host
<checkout>` 把产物写进 host 站点 `/threedream/app/` 下的静态目录，
`scripts/publish_assets.ts` 把大块字节放上资产镜像。没有任何东西会因为一次 push 就自己
发布；`deploy` 仍然为任何有 runner 可用的人保留同一份契约 —— 把产物构建进一个临时
host、上传，且只在四道关卡之后。

`rust` 关卡存在的原因：wasm 内核以字节形式随仓库交付，而没有别的任务能判断这些字节
是否还与 `rust/` 一致 —— TypeScript 检查的是 `src/physics/wasm.ts` 与它自己声明的
`WasmBindings` 接口，不是那个二进制。所以这个任务跑原生求解器测试、为 `wasm32` 构建
wgpu crate（即 M0 那条「锁定版本仍能编译」的断言）、校验已提交的产物，然后重建并要求
字节完全一致。wasm-pack 从带校验和的 tarball 安装，而不是 `curl | sh`：安装脚本是
第三方仓库上的可变 ref，而它正是铸造整个确定性承诺所依赖的那个二进制的工具。

要让「字节完全一致」成立，还有样东西必须锁定，而它不在 `Cargo.toml` 里：wasm-pack 会
自己去取 `wasm-bindgen` *CLI*，预编译的 release 产物在 `producers` 段里打上 tag 的 git
哈希，`cargo install` 兜底则只打版本号。CLI 版本相同、行为完全相同，字节却差 12 个 ——
新鲜度检查分不出这与真正的漂移。所以 `scripts/check_wasm_artifact.mjs` 把这个印记也
锁定（`WASM_BINDGEN_CLI`）：源码编译出来的 CLI 会在 provenance 关卡失败并打印原因，
而不是表现为一次无法解释的哈希不匹配。

## Unreal 5 content, read as bytes

Current feasibility study: [Rust + wasm + WebGPU + three.js](feasibility-rust-wasm-webgpu.md).
Development plan: [ThreeDream roadmap](development-plan.md).

`thirdparty/UnrealEngine` is a git submodule pointing at Epic's **private**
repository, kept as an architecture reference. The repo stores only the gitlink;
no Unreal source is committed. Cloning this project leaves that path empty.
Fetching it requires a GitHub account that has accepted Epic's EULA and been
granted access:

```bash
bash scripts/clone_unreal_reference.sh          # branch: release (default)
bash scripts/clone_unreal_reference.sh 5.5      # or another branch
```

It is a multi-GB shallow clone and the script retries transient network errors.
Nothing in `src/`, `tests/` or the demo depends on it, so a plain
`npm install && npm test` works without it.

The engine parses `.uasset` files itself. `src/assets/ue/` holds a package reader
(summary, name, import and export tables), the tagged-property decoder, Oodle
bulk data, mesh descriptions, textures and the material graph walker. Semantics
come from the reference source: an array of tagged structs carries one inner tag
whose size covers every element, material parameters are addressed by kind and
name because `UMaterialInstance` keeps its arrays apart, and each rule has a
unit test rather than a comment.

`scripts/ue_extract.ts` is the import tool that turns a project's content into
assets a page can load, and it is where that code is driven from. The project the
UE level page plays is `thirdparty/demo`: LeaffyL's UE5.5 FPS Demo, 652 MB of
`.umap` and `.uasset`. `--content` defaults to its `Content/`, so every command
below runs as written:

```bash
npx tsx scripts/ue_extract.ts level --name ue55_fps --map 关卡/关卡_测试.umap
npx tsx scripts/ue_extract.ts meshes    --dir 静态网格体 --out demo/public/ue/fps-demo
npx tsx scripts/ue_extract.ts materials --dir 材质       --out demo/public/ue/fps-demo
npx tsx scripts/ue_extract.ts data --dir 数据资产 --name weapons --out demo/public/ue/fps-demo
npx tsx scripts/ue_extract.ts data --dir 武器蓝图 --name pickups --out demo/public/ue/fps-demo
npx tsx scripts/ue_extract.ts data --dir 玩家蓝图 --name player  --out demo/public/ue/fps-demo
npx tsx scripts/ue_extract.ts sounds --encode vorbis --out demo/public/ue/fps-demo \
  --dir 音效,Gun_Sound_Essentials/Wavs,Footstep_Sounds_Pro/Wavs/Concrete_footsteps,Footstep_Sounds_Pro/Wavs/Metal_footsteps,Footstep_Sounds_Pro/Wavs/Dirt_footsteps
npx tsx scripts/ue_extract.ts cues --dir 音效,武器蓝图 --name cues --out demo/public/ue/fps-demo
```

A level also names meshes no project ships, because `/Engine/BasicShapes/Cube`
comes with the editor rather than with the game: `meshes --builtin all` writes
those from the descriptions `src/assets/ue/builtinMesh.ts` generates, and the
first-person template's prototyping meshes and materials (`LevelPrototyping`,
`Target`, `FPWeapon`, read out of `thirdparty/FPS-Shooter-Unreal/Content`) sit
beside them in `demo/public/ue/`. The page loads both sets, so a name resolves
whichever side of that line it came from.

A level can also name a whole pack its own project never shipped. `mainmap.umap`
in `thirdparty/FPS-Shooter-Unreal` places 3,571 actors, 3,384 of them with a
mesh, and 3,367 of those point at `/Game/AbandonedFactory/...`: a folder that
repository does not contain, because the pack is a marketplace set the project
expected whoever opened it to already have. Two flags make the town readable,
both in the import tool and neither in the page. `--mount` adds a content root a
`/Game/` reference may resolve against, walked after the project's own so a mount
can add packages but never shadow one, which is the precedence Unreal's own
content library resolves a reference with. `--redirects` obeys a name table,
because the pack that does exist spells its meshes a folder deeper and an edition
prefix further along (`sm_WoodPallet_01_01` is
`WoodPallet_01/Meshes/sm_en_WoodPallet_01_01`); `scripts/ue_redirects.ts` mints
that table by folding the prefix away and requiring the fold to land on exactly
one package, so it wrote 119 redirects and reported the 15 names nothing carries
instead of guessing at either. `--max-texture 1024` is the third of the set: it
downscales through `downscalePng` at import time. The town samples 145 images;
the same command with the flag at 0 writes them as authored, 2,027 MiB of 4k
albedo and normals with one ground concrete at 41 MiB, and at 1024 it writes the
258 MiB a page can stream. Those two totals are one measurement, not an estimate.

```bash
npx tsx scripts/ue_redirects.ts \
  --level demo/public/ue/shooter/level/mainmap.json \
  --content thirdparty/AbandonedFactory/Content \
  --project thirdparty/FPS-Shooter-Unreal/Content \
  --out scripts/ue_redirects/officiallyutso-fps-shooter.json

npx tsx scripts/ue_extract.ts level --name mainmap --map mainmap.umap \
  --content thirdparty/FPS-Shooter-Unreal/Content \
  --mount thirdparty/AbandonedFactory/Content \
  --redirects scripts/ue_redirects/officiallyutso-fps-shooter.json \
  --out demo/public/ue/shooter
```

Those two write the level and its name table; the geometry and the surfaces come
from `meshes` and `materials`, which the level run prints as its own `next:` line
with the folders it actually named, the mounts and the flag carried along. The
page loads four roots, so those two runs happen once per content root: the
project's into `demo/public/ue/shooter`, the pack's into
`demo/public/ue/shooter/pack` with `--content thirdparty/AbandonedFactory/Content`.

A mesh found through a mount carries the level's own spelling as an `aliases`
entry beside its name, so the page resolves an actor's `meshName` without ever
learning that a redirect happened. The name table is committed (it is names, not
bytes) and the pack is not, nor is its extract: `AbandonedFactory` ships no
licence at all, so `thirdparty/AbandonedFactory/` and `demo/public/ue/shooter/`
are both ignored and `?level=mainmap` 404s on a fresh clone until a contributor
who has the pack runs that import. That is the honest shape of it. The level, the
rifle, the arms and their animations, the targets and the one weapon `SoundWave`
are the Apache-2.0 project's own and ship; the town it stands in is somebody's
unlicensed marketplace set and does not. The rule and the survey behind it are in
[free-assets.md](free-assets.md).

Output lands under `demo/public/ue/` with a `manifest.json` per kind: glTF meshes,
PNG textures, glTF materials plus the parameters each one resolved to, the map's
actors as placements, and one playable file per `SoundWave` beside a `cues.json`
recording which `MetaSoundSource` or weapon `Blueprint` names which waves. For the
set that ships: 20 actors, 8 of them mesh actors, every one resolved; 163 waves
read out of Oodle-compressed packages, of which 8 cues join 28 to an event; 4
weapons bound to their own fire sound through their data asset's `武器类` field.
The manifest is the contract the demo loads against, so what ships is reproducible
from the source content instead of copied by hand.

Sound is the last link, and deliberately the only part that touches a device:
`demo/ue/audio.ts` builds the bank in bare Node, where `tests/ue_audio.test.ts`
asserts every join, and `UeAudio` plays it from events the simulation already
produced. It cannot change a match, which is why `?steps=600` digests the same
with audio running as with audio never built.

`thirdparty/UnrealEngine` 是一个指向 Epic **私有**仓库的 git submodule，留作架构参考。
仓库里只存 gitlink，**不提交任何 Unreal 源码**。克隆本项目后该目录是空的；拉取它需要
一个已接受 Epic EULA 并获得访问权限的 GitHub 账号（命令同上）。这是一次数 GB 的浅克隆，
脚本会对瞬时网络错误重试。`src/`、`tests/` 和演示都不依赖它，所以直接
`npm install && npm test` 就能跑通。

引擎自己解析 `.uasset`。`src/assets/ue/` 里有包读取（summary / name / import / export
四张表）、标签属性解码、Oodle bulk data、网格描述、贴图与材质图遍历。语义全部对着参考
源码实现 —— 例如一个 tagged struct 数组只带一个内层标签、其 size 覆盖所有元素；材质
参数按 kind + name 寻址，因为 `UMaterialInstance` 的各类参数数组是分开的 —— 每条规则
都有单元测试守着，而不是只有一句注释。

`scripts/ue_extract.ts` 是把这些能力用起来的导入工具，把工程内容转成页面能加载的资源。
UE level 那页跑的工程是 `thirdparty/demo` —— LeaffyL 的 UE5.5 FPS Demo，652 MB 的
`.umap` 与 `.uasset` —— `--content` 默认就指它的 `Content/`，所以上面那些命令照抄即可运行。

关卡还会点到工程里根本没有的网格：`/Engine/BasicShapes/Cube` 是随编辑器发的，不是随游戏
发的，于是 `meshes --builtin all` 用 `src/assets/ue/builtinMesh.ts` 生成的描述把它写出来；
第一人称模板自带的 prototyping 网格与材质（`LevelPrototyping`、`Target`、`FPWeapon`，从
`thirdparty/FPS-Shooter-Unreal/Content` 读出）放在 `demo/public/ue/` 里同一层。页面两组都
加载，于是一个名字不论来自哪一边都能解析。

关卡还会点到一整个自己工程根本没发的资源包。`thirdparty/FPS-Shooter-Unreal` 里的
`mainmap.umap` 摆了 3571 个 actor，其中 3384 个带网格，而 3367 个指向
`/Game/AbandonedFactory/...` —— 那个仓库里并没有这个目录，因为它是商城资源包，工程默认
打开它的人手上已经有。两个开关把这座城读出来，都在导入工具里，页面一个都不用。
`--mount` 增加一个 `/Game/` 引用可以解析到的内容根，排在工程自己的根之后，所以 mount
只能增加包、不会覆盖包 —— 这正是 Unreal 自己的 content library 解析引用时的优先级。
`--redirects` 服从一张名字表：现存的那个包把网格写在深一层的目录里、名字还多个版本前缀
（`sm_WoodPallet_01_01` 其实是 `WoodPallet_01/Meshes/sm_en_WoodPallet_01_01`），
`scripts/ue_redirects.ts` 就把前缀折掉、并要求折完只落在唯一一个包上，于是它写出了 119 条
重定向，同时把 15 个谁都没有的名字如实报出来，两边都不猜。`--max-texture 1024` 是这组里
的第三个：导入时用 `downscalePng` 降采样。这座小镇一共采样 145 张图，同一条命令把这个
flag 取 0 时按原样写出 2,027 MiB 的 4k 漫反射与法线（单是一张地面混凝土就 41 MiB），取
1024 时写出页面流式加载得动的 258 MiB。这两个总数是一次实测，不是估算。

经 mount 找到的网格，会在自己的名字旁边多带一条 `aliases`，也就是关卡原来那种拼写，所以
页面解析 actor 的 `meshName` 时完全不需要知道曾经发生过重定向。名字表是提交的（它是
名字，不是字节），资源包不提交，导入产物也不提交：`AbandonedFactory` 根本没有许可证，
所以 `thirdparty/AbandonedFactory/` 与 `demo/public/ue/shooter/` 都被 ignore，
`?level=mainmap` 在新克隆上会 404，直到某个手上有这个包的贡献者把上面这套导入跑一遍。
这才是它诚实的样子：关卡、步枪、手臂及其动画、靶子、那一个武器 `SoundWave` 都是
Apache-2.0 工程自己的，随仓库发布；它站着的那座城是别人没有授权的商城资源，不发布。规则
与背后的调研在 [free-assets.md](free-assets.md)。

产物落在 `demo/public/ue/`，每类一份 `manifest.json`：glTF 网格、PNG 贴图、glTF 材质及其
解析出的参数、地图里的 actor（以 placement 的形式），以及每个 `SoundWave` 一个可播放文件，
外加一份记录「哪个 `MetaSoundSource` 或武器蓝图点了哪些 wave」的 `cues.json`。发布出去的
这一套：20 个 actor，其中 8 个是网格 actor，全部解析成功；从 Oodle 压缩包里读出 163 个
wave，其中 8 条 cue 把 28 个接到了事件上；4 把武器通过各自数据资产的 `武器类` 字段绑到
自己的射击音。manifest 就是 demo 加载时的契约，所以发布出去的资产可以从源内容复现，而
不是手工拷来的字节。

声音是最后一环，也是刻意做成唯一碰设备的部分：`demo/ue/audio.ts` 在纯 Node 里把 bank 建
出来（每一条 join 都由 `tests/ue_audio.test.ts` 断言），`UeAudio` 只播放仿真已经产生的事件。
它改不了对局，所以 `?steps=600` 的 digest 在开着音频与从没建过音频两种情况下完全相同。

## Layout

```
src/core/      clock.ts digest.ts ecs.ts engine.ts events.ts rng.ts
src/physics/   types.ts builtin.ts wasm.ts rapier.ts reference.ts components.ts
src/gpu/       capabilities.ts device.ts compute.ts particle*.ts soft*.ts
src/ai/        mlp.ts policy.ts trainer.ts baseline.ts
src/envs/      types.ts drive.ts reach.ts pursuit.ts
src/assets/    glTF document/mesh/skin/animation/material, rgbe (HDR sky, sun
               bearing), glb (the re-pack path a fetched prop goes through), and
               ue/: the uasset package reader, property tags, Oodle bulk data,
               compressed buffers, mesh descriptions, textures, material graphs,
               sound waves and cues
src/render/    scene.ts particles.ts soft.ts assets.ts rig.ts
rust/          physics (solver) / physics-wasm (ABI) / gpu (wgpu skeleton)
wasm/pkg/      the shipped wasm kernel, rebuilt by `npm run build:wasm`
scripts/       gate.ts (the CI graph, run locally) build_inpage.ts (the bundle the
               host site serves) publish_assets.ts (the asset mirror)
               publish_docs.ts (the public docs mirror) train_headless.ts
               check_wasm_artifact.mjs bench_*.mjs ue_texture_encode.ts
               capture_shots.mjs render_diagrams.mjs ue_extract.ts ue_redirects.ts
               ue_redirects/ (the committed name tables, names not bytes)
               audit_unreal_reference.mjs clone_unreal_reference.sh
               cc0_kit.json fetch_cc0.ts
docs/          this file, feasibility study, development plan, free-assets.md
               (the licence survey behind the scene kit), diagrams/ (SVG sources
               and the PNGs the README shows, gated by render_diagrams.mjs),
               demo assets
demo/          index (trainer), physics-check, shared-device, particles, soft,
               fps, ue-fps: each a .html + .ts pair (fps adds fps/ for the match
               itself, ue-fps adds ue/ for the level, its weapons, its sound, and
               sceneKit.ts + kitview.ts: the CC0 layout planner that runs in
               bare Node, and the view code that dresses the level with it), plus
               pages.ts and nav.ts (the shared nav) and public/ (favicon, share
               cards, trained policies, ue/ assets the import tool wrote, and
               cc0/: the scene kit, with its manifest.json and LICENSE.md; the
               ue/shooter/ extract is ignored: it is bytes from a pack that
               ships no licence, reproducible by the two commands above)
tests/         93 files, 2616 tests
e2e/           demo, wasm physics, particles, soft (WebGL); shared device and
               the GPU halves of particles and soft (WebGPU)
.github/       ci.yml: four gates (verify / coverage / e2e / rust) then the bundle
               build; `npm run gate` runs the same four on the machine at hand
thirdparty/    UnrealEngine (submodule, opt-in), demo (LeaffyL's UE5.5 FPS Demo,
               the project the ue-fps page plays), FPS-Shooter-Unreal (the
               first-person template whose prototyping assets sit beside it),
               AbandonedFactory (the mount mainmap resolves against; ignored, it
               ships no licence), blender, ooz, rosclaw (MIT; the execution
               runtime M9 reads for closed-loop arm control, ignored like the
               rest: no build, test or page fetches anything from it)
```
