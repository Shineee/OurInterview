# View 面试题

* 事件分发的整体链路：从手指触碰到 View.onTouchEvent（源码级责任链）

  > **入口**：InputManager 通过 InputChannel 把事件投递到应用进程 → ViewRootImpl 的 DecorView（ViewGroup）开始分发。
  >
  > **三个核心方法（谁有、干什么）**：
  > - **dispatchTouchEvent**：所有 View/ViewGroup 都有，负责分发。
  > - **onInterceptTouchEvent**：**仅 ViewGroup** 有，决定是否拦截（拦截则走自己的 onTouchEvent）。
  > - **onTouchEvent**：消费事件，返回 true 表示处理。
  >
  > **责任链逻辑（简化 ViewGroup.dispatchTouchEvent）**：
  > 1. 检查 onInterceptTouchEvent → 若拦截，自己 onTouchEvent 处理。
  >  2. 未拦截 → 按 z-order 倒序遍历子 View，调子 View 的 dispatchTouchEvent；子返回 true 则本层记录 mFirstTouchTarget 并结束分发（后续事件直接走该 target）。
  >  3. 无子消费 → 自己 onTouchEvent。
  >
  > **DOWN 的特殊性**：首次 DOWN 决定后续事件流向（找到消费者）；MOVE/UP 走已建立的 TouchTarget；若 DOWN 无人消费，后续事件不再分发给该层级（onTouchEvent 也不收）。
  >
  > 设计动机：责任链让父容器有机会"先问要不要拦"（滚动冲突），又让叶子优先消费，平衡了容器与子 View 的控制权。

* onTouchEvent 与 OnTouchListener、performClick 的关系

  > View.dispatchTouchEvent 顺序：若设置了 OnTouchListener 且 onTouch 返回 true，**不再调 onTouchEvent**；否则走 onTouchEvent（内部若 clickable/longClickable 会触发 performClick → OnClickListener）。
  >
  > 这解释了"设置了 OnTouchListener 返回 true 后点击事件收不到"——因为短路了 onTouchEvent。
  >
  > 无障碍：performClick 会走 AccessibilityNodeInfo 播报，自定义 View 应在合适的 click 时机调用 performClick 而非直接 listener（保证 TalkBack 支持）。

* 滑动冲突的两类场景与解决思路（嵌套滑动）

  > **外部拦截 vs 内部拦截**：
  > - **外层需拦截（如 ViewPager 横向 + 内部竖向列表，希望外层处理横向）**：父容器 onInterceptTouchEvent 根据 dx/dy 方向判断拦截（外层拦截法）。
  > - **内层需优先（如内层自定义 View 要消费横滑）**：内层 dispatchTouchEvent 向父 requestDisallowInterceptTouchEvent(true)（内部拦截法）。
  >
  > **现代方案：NestedScrolling**（NestedScrollView/RecyclerView/CoordinatorLayout）：子 View 把未消费的滚动量通过 nested scroll 协议先给父（dispatchNestedPreScroll/Scroll），父处理不完的再由子消费——避免了手写冲突判断。AppBarLayout+RecyclerView 联动就是这套协议。

* ACTION_CANCEL 什么时候来？为什么实现"可取消的按下"要注意 CANCEL

  > 当父容器中途拦截（onInterceptTouchEvent 从 false 变 true）时，给子 View 发 **ACTION_CANCEL**（子的按下被打断）；或手势被系统识别为其他手势/窗口失焦时。
  >
  > 陷阱：只在 ACTION_UP 里清理状态（如复位按压效果、移除回弹），漏了 CANCEL 会导致"按到一半被打断后 UI 卡在按下态"。正确：UP 与 CANCEL 都做收尾清理。

* 事件分发中 onTouchEvent 返回 false 的后续行为

  > 若子 View onTouchEvent 返回 false（不消费 DOWN），父 ViewGroup 会继续尝试下一个子 View，最终父自己的 onTouchEvent 兜底。一旦 DOWN 被某层消费（返回 true），后续 MOVE/UP 就固定发给该层（mFirstTouchTarget），中间层再拦截会发 CANCEL 给它。
  >
  > 面试陷阱："false 就不分发了"不准确——是"这一层不消费，交给上级/兄弟尝试"。

* 面试官视角

  > **考察点**：是否读过 ViewGroup.dispatchTouchEvent 源码，能否讲清 TouchTarget、DOWN 定流向、CANCEL 触发条件与 NestedScrolling。
  >
  > **加分**：TouchTarget 建立机制、requestDisallowInterceptTouchEvent、嵌套滚动协议、无障碍 performClick。
  >
  > **减分**：把三个方法记混（以为 onInterceptTouchEvent 所有 View 都有）；说不清 MOVE 走哪里；不知道 CANCEL。

---

* measure / layout / draw 三步流程与 MeasureSpec 的传递规则

  > **measure**：父 View 结合自身 MeasureSpec + 子 LayoutParams 生成子的 MeasureSpec（ getChildMeasureSpec 规则：父约束 + 子要求 → 三种模式 EXACTLY/AT_MOST/UNSPECIFIED），子 measure(w,h) 后 setMeasuredDimension 存下测量尺寸。
  >
  > **layout**：父 layout 确定子的 left/top/right/bottom（相对父），子 onLayout 再递归安排自己的子（View 的 onLayout 是空，ViewGroup 实现）。
  >
  > **draw**：背景 → 自身内容（onDraw）→ 子 View（dispatchDraw）→ 前景/滚动条；硬件加速下走 RenderNode 录制 DisplayList 而非直接 Canvas 绘制指令。
  >
  > **MeasureSpec 传递**（EXACTLY=精确值；AT_MOST=wrap_content 上限；UNSPECIFIED=ScrollView 高度等不限）：这是"为什么 wrap_content 在 LinearLayout 权重下有时失效"的根源——需正确重写 onMeasure 处理 AT_MOST。

* invalidate / postInvalidate / requestLayout / forceLayout 的区别

  > - **invalidate**：只重绘（draw），不重新 measure/layout；主线程调。子 View invalidate 会向上标记 PFLAG_INVALIDATED，最终 ViewRootImpl 调度下一帧重画该区域。
  > - **postInvalidate**：非主线程用，内部 handler post 到主线程 invalidate。
  > - **requestLayout**：重新 measure+layout+draw；沿父链向上传递 PFLAG_FORCE_LAYOUT，触发整棵树（受影响部分）重新测量布局。频繁 requestLayout（如滚动中）代价高。
  > - **forceLayout**：只标记本 View 需重测，不向上传递；一般配合手动 measure 用。
  >
  > 陷阱：改变 View 尺寸/位置必须 requestLayout；只改颜色/内容 invalidate 即可；二者混用错会"不刷新"或"整树重测卡顿"。

* Choreographer 与 VSYNC：一帧是怎么被调度的

  > **模型**：应用调用 invalidate/requestLayout 后不立即绘制，而是**等 VSYNC 信号**（硬件垂直同步脉冲，60Hz≈16.6ms）触发 Choreographer 的 doFrame。
  >
  > **doFrame 三类回调顺序**：INPUT → ANIMATION → TRAVERSAL（measure/layout/draw）。TRAVERSAL 里 ViewRootImpl.performTraversals 走完整绘制管线。
  >
  > **同步屏障**：scheduleTraversals 会 postSyncBarrier，确保 VSYNC 到来的 async doFrame 消息优先于普通 Handler 消息（链接 Asynchronous 第1页）。
  >
  > 掉帧=一帧内没完成 TRAVERSAL，错过本 VSYNC 等下一帧（Jank）。三缓冲在连丢两帧时提升吞吐但增加延迟。

* 硬件加速与软件绘制的差异，何时回退软件

  > **软件**：CPU 逐指令画 Canvas（Skia），支持部分老 API；复杂时 CPU 压力大。
  > **硬件加速**：主线程把绘制录成 **DisplayList（RenderNode）**，交给 RenderThread 在 GPU 执行；减少主线程绘制耗时、支持动画高效重放。
  >
  > 回退场景：某些 Canvas API 硬件不支持（复杂 Path、特殊 PorterDuff 模式历史上踩坑），可 setLayerType(LAYER_TYPE_SOFTWARE) 强制软件层。
  >
  > 排查过度绘制：开发者选项"调试 GPU 过度绘制"色块图（蓝1x/绿2x/红3x+）；systrace 看 RenderThread/GPU 帧耗时。

* View 缓存：缓存层级与 RecyclerView 预取（GapWorker）

  > **View 树缓存四级**：mLayoutCache（刚被 layout 的）→ mDrawingCache/RenderNode 缓存 → Scrap（RecyclerView ViewHolder 回收池）→ 自定义缓存。
  >
  > **RecyclerView 预取**：滚动时利用主线程空闲（IdleHandler/GapWorker）提前 inflate/bind 即将进入屏幕的 item（按滚动方向预测），减少滑动中 inflate 造成的掉帧。设置 setItemViewCacheSize、共享 RecycledViewPool（跨列表）可进一步优化。
  >
  > 面试追问：为什么 RecyclerView 比 ListView 流畅——四级回收 + ViewHolder 复用 + 预取 + 局部刷新（notifyItemChanged 避免整列表重绘）。

* 面试官视角

  > **考察点**：绘制管线是否到 MeasureSpec/RenderNode/Choreographer 层级，invalidate 与 requestLayout 是否真懂。
  >
  > **加分**：同步屏障与 VSYNC 关系、GapWorker 预取、硬件加速回退场景、过度绘制色块图。
  >
  > **减分**：把 invalidate 当"会重新测量"；不知道一帧要等 VSYNC；说 RecyclerView 快是因为"用了缓存"一句话。

---

* 自定义 View 的完整步骤与常见陷阱

  > **步骤**：
  > 1. 继承 View/ViewGroup，处理构造函数（代码/布局各路径，统一到带 AttributeSet 的，读取 styleable 属性并 recycle）。
  > 2. onMeasure：处理 wrap_content（AT_MOST 时给出合理默认值，否则默认等于父上限）；必要时支持 padding。
  > 3. onDraw（View）或 onLayout+dispatchDraw（ViewGroup）：绘制内容。
  > 4. 交互：onTouchEvent 处理手势；必要时 ValueAnimator 驱动重绘。
  > 5. 性能：Paint/Path/Rect 预创建复用（避免 onDraw new）；复杂计算移出 onDraw。
  >
  > **常见陷阱**：
  > - wrap_content 失效（没处理 AT_MOST）。
  > - 在 onDraw 里 new 对象 → 内存抖动掉帧。
  > - 忘记处理 padding。
  > - 多线程改属性不 postInvalidate → 界面不刷新或线程安全问题。
  > - 自定义属性未 declare-styleable 导致布局 inflate 崩。

* Window / WindowManager / DecorView / Dialog 的关系

  > **Window**（抽象）：PhoneWindow 实现，代表"一块可显示区域"的策略与 DecorView 持有者；Activity.attach 时创建 PhoneWindow。
  >
  > **WindowManager**（接口，实现 WMS 客户端）：addView(view, params) 把 View 树附着到窗口系统（经 ViewRootImpl + SurfaceFlinger 合成）。
  >
  > **DecorView**：PhoneWindow 的顶级 FrameLayout，含标题栏与内容区（android.R.id.content），是 View 树根。
  >
  > **Dialog**：内部独立 PhoneWindow + WindowManager.addView，独立于 Activity 的 View 树；Activity destroy 未 dismiss Dialog → 窗口泄漏（WindowLeaked）。
  >
  > 面试追问：悬浮窗（TYPE_APPLICATION_OVERLAY）需要 SYSTEM_ALERT_WINDOW 权限，本质是 WindowManager.addView 的独立窗口，与 Activity 解耦。

* 为什么 onCreate 里 view.getWidth() 是 0？拿到宽高的正确时机

  > 宽高在 layout 后才有；onCreate/onResume 时还未 measure/layout。正确方式：
  > - view.post { } （挂到主线程队列，通常在 layout 后执行）。
  > - ViewTreeObserver.OnGlobalLayoutListener（首次布局完成回调，注意 remove 防泄漏）。
  > - view.doOnLayout {}（KTX）。
  >
  > 追问：转屏/键盘弹出宽高变化——依赖 OnGlobalLayoutListener 或 OnApplyWindowInsetsListener 处理 insets，而非缓存一次性值。

* 过度绘制与布局层级优化的真实手段（不是"减少嵌套"一句）

  > 1. 移除无用背景（根布局/父容器 background 常被子盖住）——最常见立竿见影。
  > 2. merge / ViewStub：merge 减少一层；ViewStub 延迟加载复杂区域。
  > 3. ConstraintLayout 扁平化替代深层嵌套（注意其 measure 复杂度，深约束也会慢）。
  > 4. 减少 clipPath/圆角阴影层层叠加；用 CardView/Elevation 时评估 overdraw。
  > 5. 离屏缓冲/RenderEffect（API 31+）按需。
  > 排查：Layout Inspector 层级 + GPU 过度绘制色块 + systrace measure 耗时。

* 面试官视角

  > **考察点**：自定义 View 的工程完整度与 Window 体系认知，是否在真实项目调过层级/过度绘制。
  >
  > **加分**：Dialog 窗口泄漏、ViewTreeObserver 时机、ViewStub/merge、Elevation 过度绘制评估。
  >
  > **减分**：wrap_content 不处理；在 onDraw new Paint；不知道 DecorView 是啥。
