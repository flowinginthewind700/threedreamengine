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
   in a browser produce the same simulation. The arithmetic is the engine's own
   too: `core/trig.ts` writes every transcendental ECMAScript leaves
   implementation-approximated, so a digest does not move when a host's libm does.
   That is what makes a physics-AI kernel testable: all 4428 unit tests run
   without a GPU, and the wasm backend is
   held to the bits of the TypeScript solver it ports. The two GPU layers are held
   to a CPU reference the same way, and neither claims determinism for itself:
   `deterministic` is false on both backends, because atomics promise no order, so
   training and replay always run on the reference tier.

1. **仿真是唯一事实来源。** 渲染层每帧把物理状态镜像到 `THREE.Object3D`，从不回写。
   于是同一个场景既能无头训练，也能在浏览器里渲染游玩，结果完全一致。
2. **关键路径都是确定性的。** 带种子的 RNG、固定步长、没有隐藏的全局状态，所以 Node
   里的 `engine.step(n)` 与浏览器里的 `engine.frame(dt)` 跑的是同一个仿真。运算也是引
   擎自己的：`core/trig.ts` 写了 ECMAScript 留给实现近似的全部超越函数，所以宿主换一
   个 libm 构建，摘要不动。这让一个物理-AI 内核变得可测：4428 个单元测试全都不需要
   GPU，而 wasm 后端要对齐它所移植的
   TS 求解器的每一个比特。两个 GPU 层用同样的方式对齐一份 CPU 参照，而且都不替自己
   声称确定性：两个后端的 `deterministic` 都是 false，因为原子操作不承诺顺序，所以
   训练与回放永远走参照档。

## Layers

```
core     clock / ECS / events / engine / math     no three.js, no WASM
physics  backend interface + four solvers         no three.js
gpu      probe, shared device, scale layers       no three.js, no WASM
ai       MLP, Gaussian policy, policy-gradient    no three.js, no WASM
envs     learning tasks (drive, reach, pursuit)   physics only
render   three.js bridge + rigid/articulated/
         particle/soft views                  the only layer importing three
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

All four implement `PhysicsBackend`, so envs and the trainer stay agnostic:

| Backend | File | Notes |
|---|---|---|
| `builtin` | `src/physics/builtin.ts` | Pure TS, deterministic, zero dependencies. Default, and the normative spec. |
| `wasm` | `src/physics/wasm.ts` | First-party Rust kernel. Bit-identical to `builtin`, 2.6x-4.9x faster in Node. |
| `rapier` | `src/physics/rapier.ts` | Third-party Rapier WASM. Same interface, different solver: ~3% off steady-state speeds, and different trajectories. |
| `mujoco` | `src/physics/mujoco.ts` | Third-party MuJoCo WASM. Same interface, plus the machine: a robot description compiles into links and joints whose poses come back as the solver's own kinematics. Non-deterministic, and its collision filter is a superset of the others'. |

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

`mujoco` is the one that carries a machine rather than a pile of bodies. A
`RobotDescription` -- the tree `assets/robot.ts` reads URDF and MJCF into -- is
written back out as MJCF by `assets/mjcf.ts` and compiled by MuJoCo itself, so
the forward kinematics, the joint limits and the mass matrix are the solver's and
not this engine's: `linkPose`, `jointPosition` and `sitePose` read them back, and
`tests/mujoco_backend.test.ts` checks them link by link against the chain the
description spells. Bodies made through `createBody` live in the same world and
join the model by a recompile, which is batched: creates and state writes mark it
dirty, and the next read pays for all of them once. The poses the file was
authored to start from travel with it: `MjcfScene.keyframes` goes into every
recompile, so a model's keyframe indices do not change meaning under a caller
that adds a body. Two claims it does not make.
`deterministic` is `false`, because the WASM build is compiled with SIMD whose
reduction order the browser picks, so an environment that needs a replay must not
land here by accident. And its collision filter joins `group` and `mask` with
*or* where the other backends join them with *and*, which makes it a superset: a
pair the others separate, this one may still report. Both are in the suite, so
neither is something a reader has to find out at runtime.

Its solids come from outside. A description names a mesh and points at a file;
MuJoCo reads that file out of a file system attached to the compiler, so
`meshBytes` carries the bytes in, keyed by the name a geom uses, and the mount
lives exactly as long as the compile that consumed it -- MuJoCo copies a mesh
into the model it builds. The bookkeeping is by file rather than by name,
because two `<mesh>` declarations may point at one STL (one part at two scales)
and the compiler then opens it twice off one buffer. A file with no bytes is
refused at construction naming every link that draws or collides with it, all of
them in one message rather than one per compile. What arrives is real geometry
and not a placeholder, which the suite proves physically: a plate 4cm thick
dropped from 0.6 comes to rest at 0.019724, its own half thickness, where a
backend that had kept only the bounds would rest it at 0.2 and one that had
loaded nothing would still be falling. Not every solid owes bytes: MJCF's
`ellipsoid` is three semi-axes and nothing on disk, and this reader keeps it an
ellipsoid instead of rounding it up to the sphere that contains it -- proven the
same way, since dropped on its shortest axis it rests at 0.019325, where a sphere
of the longest would rest at 0.05.

Getting a pack into that model is most of what loading costs, and the shape of
this code follows two measurements. The bindings' own way across the boundary,
`MjVFS.addBuffer`, copies a buffer element by element at about 2 MiB/s, which
makes the 17 MB of STL in the SO-101 pack an 8.7 s compile before MuJoCo reads a
line of it; writing the same bytes into the file system the compiler reads from
is a `memcpy` at about 774 MiB/s, and the same pack then compiles in 0.75 s. So
a mount is a numbered directory under `/threedream-mesh`, written just before a
compile and emptied just after, with the VFS kept as the fallback for a build
that exports no file system. Numbered, because that file system is shared by
every world in a page -- and so, one level up, is the solver: `@mujoco/mujoco`
is a MODULARIZE build, so a factory call per world is a second solver per world,
35 ms and 25 MB each and `destroy()` returns none of it, where one instance
carried by every world makes the second world cost 1-2 ms and 0.2 MB. A trainer
that wants the machine without its shells says so with `compileVisualSolids:
false`: the visual geoms are already inert, since a shell carries `contype="0"`,
but MuJoCo loads their vertices anyway, and on the workcell that is 891 ms and
411 MiB of the compile against 75 ms and nothing a resident-set sample can see,
for a trajectory identical to five decimals. A link the file weighed by its geoms
keeps them whatever that option says, because there the mass is in what it draws.

Those solids are a fetched pack rather than a drawing this repository made.
`scripts/fetch_robots.ts` reads `scripts/robot_kit.json`, downloads a robot
description at a pinned commit from its publisher's own repository, checks every
file it receives against the blob sha1 GitHub publishes for that path at that
commit, and writes `demo/public/models/arm-lab/manifest.json` and a licence file
beside the bytes; a re-run against an unchanged spec fetches nothing, and the run
refuses any file under the delivery root that no manifest row accounts for. So
the arm the embodied demos train against is the SO-101 as The Robot Studio
published it -- MJCF, solver settings, collision primitives and nineteen STL
solids -- and the provenance and licence of each of them are on the record in
[free-assets.md](free-assets.md).

四个后端都实现 `PhysicsBackend`，因此 env 与 trainer 与之无关：

| 后端 | 文件 | 说明 |
|---|---|---|
| `builtin` | `src/physics/builtin.ts` | 纯 TS、确定性、零依赖。默认，也是规范实现。 |
| `wasm` | `src/physics/wasm.ts` | 自研 Rust 内核。与 `builtin` 逐位一致，Node 下快 2.6x-4.9x。 |
| `rapier` | `src/physics/rapier.ts` | 第三方 Rapier WASM。接口相同，求解器不同：稳态速度差约 3%，轨迹也不同。 |
| `mujoco` | `src/physics/mujoco.ts` | 第三方 MuJoCo WASM。接口相同，另外还带着一台机器：机器人描述被编译成连杆与关节，位姿以求解器自己的运动学返回。非确定性，且它的碰撞过滤是其他后端的超集。 |

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

`mujoco` 是唯一带着一台机器、而不是一堆刚体的后端。`RobotDescription` —— 也就是
`assets/robot.ts` 把 URDF 与 MJCF 读成的那棵树 —— 由 `assets/mjcf.ts` 重新写成
MJCF，再交给 MuJoCo 自己编译，于是正运动学、关节限位与质量矩阵都属于求解器，而不
属于本引擎：`linkPose`、`jointPosition` 与 `sitePose` 把它们读回来，
`tests/mujoco_backend.test.ts` 逐连杆对照描述自己拼出的那条链做校验。通过
`createBody` 创建的刚体住在同一个世界里，靠一次重编译并入模型，而重编译是批量的：
创建与状态写入只负责标脏，下一次读取统一付一次代价。文件写下来的那些起始姿态跟着它
一起走：`MjcfScene.keyframes` 会被带进每一次重编译，于是一个模型的 keyframe 下标不会
因为调用方加了一个刚体就变了意思。有两个承诺它不做。
`deterministic` 是 `false`，因为 WASM 构建带 SIMD，归约顺序由浏览器决定，所以需要
回放的环境不能意外落到这里。它的碰撞过滤用「或」连接 `group` 与 `mask`，
而其他后端用「与」，因此它是超集：别的后端分开的一对，它可能仍然上报。两条都写在
测试里，而不是留给读者在运行时发现。

它的实体来自外面。描述只说出一个网格叫什么、指向哪个文件，而 MuJoCo 是从挂在编译器上
的文件系统里读那个文件的，所以字节由 `meshBytes` 交进来、按 geom 用的那个名字索引，
挂载的寿命就等于消费它的那一次编译 —— MuJoCo 会把网格拷进它建出来的模型里。记账按文件
而不是按名字，因为两个 `<mesh>` 声明可以指向同一份 STL（同一个零件的两个 scale），编译器
于是从一个 buffer 上把它打开两次。没有字节的文件在构造时就被拒绝，点名每一个画着它或拿它
碰撞的 link，并且一次说全，而不是一次编译说一个。进来的是真几何、不是替身，这一条用物理
证明：一块 4cm 厚的板从 0.6 落下，停在 0.019724 —— 它自己的半厚；只留下包围盒的后端会把
它停在 0.2，什么都没装上的后端会让它一直掉下去。也不是每个实体都欠字节：MJCF 的
`ellipsoid` 是三个半轴、盘上什么都没有，这个读取器把它留成椭球，而不是抹成装得下它的那个
球 —— 证明方式相同：让它最短的那根轴朝下落，停在 0.019325，而按最长轴抹成球的后端会把它
停在 0.05。

把一个 pack 装进那个模型，是加载开销的大头，而这段代码长成这样是跟着两个测量走的。绑定
自己那条过界的路 —— `MjVFS.addBuffer` —— 按元素拷贝一个 buffer，速率约 2 MiB/s，于是
SO-101 pack 那 17 MB 的 STL，在 MuJoCo 读到第一行之前就变成 8.7 秒的编译；把同样的字节
写进编译器读取的那个文件系统是一次 `memcpy`，约 774 MiB/s，同一个 pack 于是 0.75 秒编译
完。所以挂载就是 `/threedream-mesh` 下一个带编号的目录，编译前写入、编译后清空，VFS 留给
不导出文件系统的构建做回退。带编号，是因为那个文件系统由一个页面里的所有世界共用 —— 而
再往上一层，求解器也是共用的：`@mujoco/mujoco` 是 MODULARIZE 构建，一个世界一次 factory
调用就是一个世界一份求解器，每份 35 ms、25 MB，而 `destroy()` 一点都不还；由所有世界共持
一个实例时，第二个世界只花 1–2 ms、0.2 MB。只想要这台机器、不想要它外壳的训练方，说一声
`compileVisualSolids: false` 就行：visual geom 本来就是惰性的，因为外壳带着 `contype="0"`，
但 MuJoCo 照样把它们的顶点装进去，而在这台工作台上那就是编译里的 891 ms 与 411 MiB，对比
75 ms 与一个常驻集采样看不出来的增量，轨迹则到小数点后五位完全一致。至于文件靠 geom 称重
的那种 link，无论这个选项怎么说都保留它的实体，因为它的质量就在它画的东西里。

这些实体是取回来的一份 pack，而不是本仓库照着照片画的东西。`scripts/fetch_robots.ts`
读 `scripts/robot_kit.json`，按钉住的 commit 从发布者自己的仓库下载一份机器人描述，
把收到的每个文件与 GitHub 为该路径在该 commit 上公布的 blob sha1 对一遍，再把
`demo/public/models/arm-lab/manifest.json` 与一份许可证文件写在字节旁边；对着未变的
spec 重跑什么都不下，而交付根目录下只要有一个文件没有对应的 manifest 行，这一次运行
就拒绝通过。于是具身 demo 用来训练的那条臂，就是 The Robot Studio 发布的那个 SO-101
—— MJCF、求解器设置、碰撞图元与十九个 STL 实体 —— 而它们各自的出处与许可证都记录在
[free-assets.md](free-assets.md) 里。

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

A trained policy has a deployment half, and it is deliberately not the training
class: `src/ai/inference.ts` turns a snapshot into actions without importing the
trainer, the critic or the GAE math, so a page that only plays a policy ships
none of it. `decodeAction` is the one implementation of "mean plus scaled
Gaussian noise", and the sampling stream lives in the inference object rather
than in a kernel -- which is what lets a second kernel exist without a second
policy appearing beside it.

`src/ai/onnx.ts` is that second kernel. It writes a policy out as a `.onnx` file
and reads one back, the protobuf emitted and parsed here rather than through a
code generator, and `OnnxPolicyInference` runs the trunk through ONNX Runtime
Web. The runtime is imported dynamically, so a caller who never asks for ONNX
never instantiates its WASM. The graph carries the trunk only: `actionScale` and
the per-dimension `logSigma` travel in `metadata_props`, and the reader refuses
an artifact whose metadata disagrees with its own graph, so a policy fetched
over HTTP fails at load and names the field instead of failing at step 4000 with
a NaN. An export from a stack we do not own carries no such metadata and gets
its head handed over explicitly.

Two kernels do not agree to the bit, and nothing here pretends they do.
`Mlp.gemv` accumulates one output in f64 and narrows once per layer; ONNX
Runtime's MLAS blocks the same product to fit SIMD registers and reduces in its
own order. `tests/onnx.test.ts` measures the gap on real trunks -- 9.5e-7
open-loop on 6-64-64-2, 1.4e-6 worst over three closed-loop episodes -- and pins
it with an order of magnitude of headroom, so another BLAS does not turn the
suite red and a rescaled weight still does. Because the decode is shared,
`sampled` mode draws the identical noise sequence on both paths and the only
difference between the two trajectories is the trunk's last bit. That is the
difference between "two implementations that mostly agree" and "one policy with
two kernels", and `replay.ts` is where it becomes an assertion: `recordRollout`
and `recordRolloutAsync` share one plan, and `compareTraces` reports the largest
deviation per quantity plus the first step that broke a tolerance the caller
stated.

Cost is measured rather than assumed, and it is not a reason to move the
reference path: a batch-1 `session.run` costs about seven times a TypeScript
forward pass on the narrow trunk, and the order flips on a wide one, where a
blocked GEMM finally has enough work to block. `ai/batch.ts` gathers a chunk's
rows into one flat buffer, which is the shape that batched call wants.

A demonstration is the other way in, and it arrives as a dataset rather than as
a reward. `src/ai/dataset.ts` is the LeRobot v3.0 layout: the five files that
make a dataset at the smallest, the bookkeeping columns the layout owns and a
caller never supplies, and the per-dimension statistics -- including the five
quantiles, which lerobot estimates from a 5000-bin histogram and this writer
computes over every frame. It returns a map of path to bytes rather than a
directory, because the recording happens in a browser: Node puts the same map on
a filesystem, a page streams it to a mirror or to IndexedDB, and neither choice
is made in the engine. The columnar container underneath it is
`src/assets/parquet.ts`.

That map leaves a page as one file. `src/assets/zip.ts` writes it as an archive
and reads one back, which is the difference between a dataset a visitor can
download and one that only ever existed in memory: the five files become a
single `.zip`, and the page on the other side unpacks it into the same map
`readDataset` takes. It writes to be reproducible -- names sorted, one fixed
timestamp, no extra fields -- so the same recording twice is the same bytes, and
a published artifact can be told apart from a different one. The central
directory is the authority and the local header only a witness, which is what
lets it read an archive carrying a self-extracting stub in front of it, and one
whose sizes arrived in a data descriptor after the payload. Deflate is injected
the way PNG's codecs are, so `src/` still imports nothing from `node:`.

The container reads what a real lerobot writes, not only what this writer
produces: Snappy-compressed pages and RLE_DICTIONARY-encoded ones, which is how a
dataset recorded on an arm arrives. That is a claim about somebody else's bytes,
so it is checked against somebody else's implementation --
`scripts/lerobot_dataset_check.ts` hands a corpus to an installed lerobot and
reports, one row a dataset, what each side made of the same bytes and what
lerobot makes of the bytes written back. Two implementations written by the same
hand agree perfectly and prove nothing, which is why the foreign side is a
process and not a fixture. The five exact quantities are held to a tolerance; the
five quantiles are held to numpy's, because lerobot answers from a 5000-bin
per-episode histogram and clamps the outer two to `min - 1e-10`, and the distance
between the two estimators is printed beside the verdict rather than asserted.

`src/ai/imitation.ts` fits a policy to that recording. It is the other trainer's
gradient with the advantage replaced by a constant, so the diagonal-Gaussian
head and the sign conventions that were each paid for with a measured failure
over there are inherited rather than re-derived, and what gets minimised is the
log density: a squared-error fit would leave the head's sigma at whatever it was
initialised to, and the policy would then be sampled from a width that means
nothing. The trunk runs Adam (`src/ai/adam.ts`), because a supervised gradient
is exact for the batch it came from and the parameters it moves differ in scale
by orders of magnitude; the head keeps its own ascent at a rate of its own.
Episodes are held out whole rather than frame by frame, and the epoch with the
lowest held-out loss is the one kept. The normalisation the training happens in
is folded into the first and last layers before the policy leaves, so the
artifact that deploys -- a `PolicySnapshot`, or the ONNX graph written from it
-- carries no mean and standard deviation a page would have to remember to
apply.

训练好的策略有部署的那一半，而且刻意不是训练时那个类：`src/ai/inference.ts` 把一份
snapshot 变成动作，不 import trainer、critic 与 GAE 那套数学，所以一个只播放策略的页面
一样都不会打包进去。`decodeAction` 是「均值加缩放高斯噪声」唯一的实现，采样流住在推理
对象里而不是某个内核里 —— 这正是第二个内核可以存在、而不必在它旁边多出第二个策略的原因。

`src/ai/onnx.ts` 就是那个第二内核。它把策略写成 `.onnx` 文件再读回来，protobuf 在这里
自己写、自己解，不经代码生成器；`OnnxPolicyInference` 让 trunk 走 ONNX Runtime Web。运行时
是动态 import 的，所以没要 ONNX 的调用方一次它的 WASM 都不会实例化。图里只有 trunk：
`actionScale` 与每一维的 `logSigma` 走 `metadata_props`，而读取器拒绝一份元数据与自己的图
对不上的产物，于是从 HTTP 取回来的策略在加载时就失败并点名字段，而不是跑到第 4000 步给出
一个 NaN。别人那套栈导出的文件没有这份元数据，就显式把 head 交给它。

两个内核不会逐位相同，这里也没有任何东西假装它们相同。`Mlp.gemv` 在 f64 里累加一个输出、
每层收窄一次；ONNX Runtime 的 MLAS 为了塞进 SIMD 寄存器把同一个乘积分块，按自己的顺序
归约。`tests/onnx.test.ts` 在真实 trunk 上量出这个差 —— 6-64-64-2 开环 9.5e-7，三段闭环
里最差 1.4e-6 —— 并带着一个数量级的余量钉住它：换一套 BLAS 不会让测试变红，重新缩放一个
权重仍然会。因为 decode 是共享的，`sampled` 模式在两条路径上抽出完全相同的噪声序列，两条
轨迹之间唯一的差别是 trunk 的最后一个比特。这就是「两个大致相同的实现」与「一个策略、两个
内核」的区别，而 `replay.ts` 是它变成断言的地方：`recordRollout` 与 `recordRolloutAsync`
共用一份 plan，`compareTraces` 报告每个量的最大偏差，以及第一个越过调用方给定容差的那一步。

代价是量出来的，不是假设出来的，而且它不构成把参照路径搬走的理由：窄 trunk 上一次 batch-1
的 `session.run` 大约是 TypeScript 前向的七倍开销，宽 trunk 上这个次序会翻过来，因为分块
GEMM 终于有了够多的活可分。`ai/batch.ts` 把一个 chunk 的行收进一整块连续缓冲，那正是批量
调用想要的形状。

示教是另一条入口，而且它是以数据集的形式到达的，不是以奖励的形式。`src/ai/dataset.ts`
就是 LeRobot v3.0 的排布：一个数据集最小的那五个文件、由排布自己拥有因而调用方永远不
会去提供的簿记列，以及每一维的统计量 —— 其中五个分位数 lerobot 用一张 5000 桶的直方图
估计，这里在全部帧上直接算。它交出的是一张「路径 → 字节」的表而不是一个目录，因为录制
发生在浏览器里：Node 把同一张表落到文件系统上，页面把它流到镜像或 IndexedDB，这两种
选择都不在引擎里做。底下那层列式容器住在 `src/assets/parquet.ts`。

这张表以「一个文件」的形式离开页面。`src/assets/zip.ts` 把它写成归档、也把它读回来，这就
是「一个访客能下载的数据集」与「一个只存在于内存里的数据集」之间的区别：五个文件变成一个
`.zip`，另一侧的页面把它解回 `readDataset` 要的那张表。它是照着可复现写的 —— 名字排序、
统一一个固定时间戳、不带 extra 字段 —— 所以同一份录制跑两次得到同一串字节，一个已发布的
产物才和另一个区分得开。中央目录是权威，本地头只是证人，这也是它读得了「前面带着自解压
壳」的归档、和「尺寸写在 payload 之后的 data descriptor 里」的归档的原因。deflate 按 PNG
那套注入进来，所以 `src/` 依然不 import 任何 `node:`。

这层容器读的是真实 lerobot 写出来的东西，不只是这个写方产出的东西：Snappy 压缩的页、
RLE_DICTIONARY 编码的页 —— 一份在机械臂上录下来的数据集就是这样到达的。这是关于别人字节
的断言，所以要拿别人的实现来核：`scripts/lerobot_dataset_check.ts` 把一个语料库交给装好的
lerobot，逐个数据集报告一行 —— 两侧对同一份字节各读出了什么，以及写回之后的那份字节
lerobot 又读出什么。同一只手写出来的两个实现会完美地互相同意，因而什么也证明不了，这正是
对面那一侧是一个进程而不是一个 fixture 的原因。五个精确量按容差钉住；五个分位数按 numpy
的那一个钉住，因为 lerobot 是从一张 5000 桶的逐 episode 直方图回答的，而且把最外两个夹到
`min - 1e-10`；两个估计量之间的距离写在结论旁边，是量出来的，不是断言出来的。

`src/ai/imitation.ts` 把策略拟合到这份录制上。它就是另一个 trainer 的梯度、把优势换成
一个常数，所以对角高斯头与那些「各自被一次实测失败买下来」的符号约定是继承来的，不是
重新推的；被最小化的是对数密度：换成平方误差，头的 sigma 就会停在它被初始化到的那个
值上，于是策略将从一个什么都不表示的宽度里被采样出来。trunk 走 Adam（`src/ai/adam.ts`），
因为一个监督梯度对它所在的那个 batch 是精确的，而它要动的参数在尺度上相差几个数量级；
头保留自己的上升方向与自己的步长。留出是按整段 episode 而不是按帧，留下的是留出损失
最低的那个 epoch。训练所在的那个归一化，在策略离开之前被折进首末两层，于是部署出去的
产物 —— 一份 `PolicySnapshot`，或由它写出的 ONNX 图 —— 不带任何需要页面记得去应用的
均值与标准差。

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
| `npm test` | 4428 unit tests in 143 files, headless, no GPU needed | ~25s |
| `npm run test:coverage` | same suite under v8, floor enforced by `vitest.config.ts` | ~31s |
| `npm run test:e2e` | 67 Playwright tests over 9 specs, two projects: SwiftShader WebGL2 and ANGLE/Vulkan WebGPU | ~2.8 min |
| `npm run test:rust` | 68 native Rust tests for the solver | ~1s warm |
| `npm run check:wasm` | assertions over the shipped wasm kernel: ABI, provenance, behaviour | ~1s |
| `node scripts/bench_gpu_particles.mjs` | the M3 ladder at 1k/10k/50k/100k particles, 160 steps a rung: per-step cost as p50/p95/mean over 20 chunk samples, draw calls, blit size | ~6s |
| `node scripts/bench_gpu_soft.mjs` | the M4 ladder at 1k/5k/10k/20k nodes, 160 steps a rung: the same distribution, plus dispatches, colors and stretch | ~3s |
| `npx tsx scripts/bench_cpu_soft.ts` | the same M4 ladder on the fallback tier: `softCpu.ts`, single-threaded, no GPU, p50/p95 a step at 1k-20k nodes and a fitted us/node | ~10s |
| `npm run sim-env` | compose a task document against a machine and a level and run it headless: the sizes a policy is built at, how much of a level came and how much was pruned, every note the composition settled for, the cost of one control step, and with `--episodes n` a training run and a greedy score after it | 1.5s to measure / ~2.3 min for 320 episodes |
| `npm run bench:env` | the env factory's two ladders: one instance over a growing world (1/8/32 loose bodies) timed against MuJoCo WASM compiled from the same composed scene, and a fleet of independent envs (1-8) stepped round-robin for the linearity M11 stands on | ~1.5 min |
| `npx tsx scripts/profile_boot.ts` | what one page boot costs, from four sources no single one of which can see all of it: the heap curve and its allocation floor from CDP `Performance.getMetrics`, the GC pause distribution from a `v8` trace, the frames of the load itself from a sampler installed before any page code runs, and the bytes left resident on the GPU and wasm sides. `--runs n` reports the median and the spread across boots, `--measure ms` bounds how long the page gets to finish counting its own scene | ~1.5 min a boot on `ue-fps ?level=mainmap`, ~4.5 min for three |
| `npm run diagrams` / `npm run diagrams:check` | rasterise `docs/diagrams/*.svg` to the committed PNGs, or gate them against the manifest's hashes | ~5s / ~0s |
| `node scripts/capture_shots.mjs` | the screenshots in the README and the share cards: 14 captures of the built pages served at their deployment subpath, each gated on the page's own tier report | ~40s |
| `node scripts/check_live_demo.mjs` | boot a *deployed* page headless and read the verdict the page publishes, with every request it made: the check that the mirror two hops away still serves what the bundle asks for | ~1 min a page |
| `npx tsx scripts/robot_zoo_check.ts` | read, write and re-read every robot model in the opt-in zoo under `thirdparty/rosclaw`, one row each: the command behind the claim that the URDF and MJCF readers hold on models other people wrote. Not a gate step, because the zoo is ignored; exits 0 when it is absent | ~3s |
| `npx tsx scripts/lerobot_dataset_check.ts` | hand every dataset under `thirdparty/lerobot-datasets` to an installed lerobot, one row each: what this reader made of the bytes, what lerobot made of them, and what lerobot makes of the bytes written back. With no corpus on disk it asks lerobot to write one. The five quantiles are held to numpy and the distance to lerobot's histogram estimator is printed beside the verdict rather than asserted. Not a gate step, because the second implementation is a python process the suite does not own; exits 0 when lerobot is absent and carries the interpreter's own reason | ~10s |

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
bytes), and so is the imported project's own half of the extract: the level, the
rifle, the arms and their animations, the targets and the one weapon `SoundWave`
-- 26 files, 6.5 MB, published to the asset mirror with the rest of the
delivery. `AbandonedFactory` ships no licence at all, so
`thirdparty/AbandonedFactory/` and `demo/public/ue/shooter/pack/` are both
ignored and the town's 534 MB never enters this repository; a deployed page
reads those bytes from the mirror that carries them, `mc0700/td-ue-fps-pack`,
pinned by revision in `demo/assetBase.ts`, so `?level=mainmap` resolves without
this repository redistributing anything it was not granted. The rule and the
survey behind it are in [free-assets.md](free-assets.md).

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
名字，不是字节），导入产物里属于工程自己的那一半也是提交的：关卡、步枪、手臂及其动画、
靶子、那一个武器 `SoundWave` —— 26 个文件、6.5 MB，和交付里其余的部分一起发布到资产镜像。
`AbandonedFactory` 根本没有许可证，所以 `thirdparty/AbandonedFactory/` 与
`demo/public/ue/shooter/pack/` 都被 ignore，那座城的 534 MB 从不进入本仓库；部署出去的
页面从承载这些字节的镜像 `mc0700/td-ue-fps-pack` 去读，版本在 `demo/assetBase.ts` 里钉住，
于是 `?level=mainmap` 能解析，而本仓库没有再分发任何它没被授权的东西。规则与背后的调研在
[free-assets.md](free-assets.md)。

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
src/core/      clock.ts digest.ts ecs.ts engine.ts events.ts rng.ts trig.ts
               profile.ts
               (every transcendental the spec leaves implementation-approximated,
               written in the arithmetic it pins, so a digested simulation answers
               the same bits on every host; and the arithmetic of a measurement --
               nearest-rank percentiles, a heap curve that says in so many bytes
               that it is a floor, a frame report against the engine's own fixed
               step -- owned here rather than rewritten by each script that samples,
               so "what is the p95" is one question with one answer)
src/physics/   types.ts builtin.ts wasm.ts rapier.ts mujoco.ts reference.ts
               components.ts
src/gpu/       capabilities.ts device.ts compute.ts particle*.ts soft*.ts
src/ai/        mlp.ts policy.ts trainer.ts baseline.ts, then the half that
               deploys a trained policy: inference.ts (the reference kernel and
               the decode both paths share), replay.ts (a rollout pinned by
               digest, and the comparison that holds two kernels to each other),
               batch.ts (a chunk's rows gathered into one flat buffer), onnx.ts
               (the same policy through ONNX Runtime, plus the writer and reader
               of the artifact that carries it), and the half that learns from a
               recording instead of a reward: dataset.ts (the LeRobot v3.0
               layout, written and read back), adam.ts (the update rule a
               supervised gradient earns), imitation.ts (behaviour cloning, with
               its normalisation folded into the layers that leave)
src/envs/      types.ts drive.ts reach.ts pursuit.ts, then the two that make a
               task a document instead of a file: spec.ts (the seven things a task
               may state, each a closed list of kinds rather than an expression
               language, and every way a document can fail to be a task, reported
               together) and factory.ts (a level's boxes, a machine's description
               and one of those documents composed into a `TaskEnv`, with the
               pruning and the degenerate solids it had to settle for counted out
               loud)
src/assets/    glTF document/mesh/skin/animation/material, rgbe (HDR sky, sun
               bearing), glb (the re-pack path a fetched prop goes through),
               xml (the text both robot formats are written in), robot (the one
               description shape they are read into, and the pose arithmetic
               both readers fold with), urdf, mjcf (reader and writer), stl (the
               solids both of them point at, reader and writer both), parquet
               (the columnar container a dataset lives in, reader and writer
               both), zip (the one file that map leaves a page in, reader and
               writer both), textureLedger (which textures a level asked for and
               which of them arrived, so "this level is textured" is a question
               with an answer instead of a hope), and ue/: the uasset package
               reader, property tags,
               Oodle bulk data, compressed buffers, mesh descriptions, textures,
               material graphs, sound waves and cues
src/render/    scene.ts articulatedScene.ts (a machine drawn link by link
               from the description a robot file compiled into) particles.ts
               soft.ts assets.ts rig.ts levelScene.ts surface.ts (the level a
               project's placements are drawn into, and the material each of its
               sections gets, which is also where every texture request is
               announced to that level's ledger) sceneBytes.ts (what such a scene
               costs the card, deduplicated by uuid so a kit shared by three
               hundred meshes is counted once, and naming what it could not size)
rust/          physics (solver) / physics-wasm (ABI) / gpu (wgpu skeleton)
wasm/pkg/      the shipped wasm kernel, rebuilt by `npm run build:wasm`
scripts/       gate.ts (the CI graph, run locally) build_inpage.ts (the bundle the
               host site serves) publish_assets.ts (the asset mirror)
               publish_docs.ts (the public docs mirror) train_headless.ts
               check_wasm_artifact.mjs check_live_demo.mjs bench_*.mjs
               sim_env.ts (a document on disk composed, measured and trained
               headless) bench_env_factory.ts (the same composition's two ladders:
               a growing world against the reference, a growing fleet for
               linearity)
               profile_boot.ts (what one page boot costs, measured rather than
               argued about: the heap curve and its allocation floor, the GC pause
               distribution, the frames of the load itself, and the bytes left
               resident on the GPU and wasm sides)
               ue_texture_encode.ts capture_shots.mjs render_diagrams.mjs
               ue_extract.ts ue_redirects.ts
               ue_redirects/ (the committed name tables, names not bytes)
               audit_unreal_reference.mjs clone_unreal_reference.sh
               robot_zoo_check.ts (the same idea, over the robot zoo)
               lerobot_dataset_check.ts (the same idea, against a real lerobot)
               cc0_kit.json fetch_cc0.ts robot_kit.json fetch_robots.ts
docs/          this file, feasibility study, development plan, free-assets.md
               (the licence survey behind the scene kit), diagrams/ (SVG sources
               and the PNGs the README shows, gated by render_diagrams.mjs),
               demo assets
demo/          index (trainer), physics-check, shared-device, particles, soft,
               fps, ue-fps, arm-lab: each a .html + .ts pair (fps adds fps/ for the match
               itself, ue-fps adds ue/ for the level, its weapons, its sound, and
               sceneKit.ts + kitview.ts: the CC0 layout planner that runs in
               bare Node, and the view code that dresses the level with it, and
               arm-lab adds arm-lab/ for the workcell: the document it reads, the
               expert that demonstrates on it, the episode both hosts fly, and
               the recording that exports, trains and hands a policy back), plus
               pages.ts and nav.ts (the shared nav) and public/ (favicon, share
               cards, trained policies, ue/ assets the import tool wrote,
              models/arm-lab/: the robot kit, fetched by fetch_robots.ts with
              its manifest.json and LICENSE.md, and cc0/: the scene kit, with
              its manifest.json and LICENSE.md; the
               ue/shooter/ extract ships except for pack/, which is ignored:
               bytes from a pack that ships no licence, reproducible by the two
               commands above and read from a mirror of their own)
tests/         143 files, 4428 tests
e2e/           demo, wasm physics, particles, soft, the UE level's digest, the
               arm lab (WebGL); shared device and the GPU halves of particles and
               soft (WebGPU)
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
