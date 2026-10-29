# Fragment 面试题

* Fragment 生命周期与宿主 Activity 的回调顺序（必背细节）

  > Fragment 依附于 FragmentActivity，回调顺序（以 activity 启动含 fragment 为例）：
  > - activity.onCreate 内执行 fragment 事务 → fragment.onAttach → fragment.onCreate → fragment.onCreateView → fragment.onViewCreated → fragment.onStart → activity.onStart → fragment.onResume → activity.onResume
  >
  > 注意：fragment 的 onResume 在 activity.onResume **之前**（onAttach..onResume 都嵌套在 activity 对应回调内部完成，如果事务在 onCreate 提交）。销毁时顺序对称：activity.onPause → fragment.onPause → ... → fragment.onDetach。
  >
  > **onAttach~onCreateView 之前**：拿不到宿主 View；**onCreateView 返回的 View** 是 fragment 的容器视图；**onViewCreated** 适合做 findViewById/绑定；**onResume** 后才真正可见可交互。
  >
  > 追问："为什么 onCreateView 可以返回 null？"ViewPager 的非 UI Fragment（如只做数据）可返回 null；但普通 UI Fragment 返回 null 会在后续 NPE。

* Fragment 与 Activity 状态恢复的联动（setRetainInstance 已废弃背景）

  > 系统重建 Activity 时，FragmentManager 会按 tag/id 从 saved state **自动重建** fragment，并调用 onSaveInstanceState/onViewStateRestored。所以 fragment 里临时状态也要自己存（arguments 或 onSaveInstanceState）。
  >
  > **setRetainInstance(true)**（已废弃）：曾用于"配置变更时保留 fragment 实例不销毁重建"（旋转不丢内存中数据）。废弃原因：与 Navigation/多窗口/嵌套 fragment 状态管理冲突，官方推荐 **ViewModel** 承担"配置变更存活"职责，fragment 该重建就重建。
  >
  > 面试考点：区分"配置变更保留（ViewModel）"与"进程死亡恢复（SavedStateHandle/onSaveInstanceState）"两层，fragment 自身只是视图容器不承担长数据持有。

* 嵌套 Fragment 与 childFragmentManager、parentFragmentManager 的区别

  > - **childFragmentManager**：管理子 Fragment（嵌在本 Fragment 的容器里）。子 fragment 的生命周期跟随父 fragment（父 destroy 子 destroy）。
  > - **parentFragmentManager**：管理"把我放进来"的那层事务（与宿主 activity 的 fm 有时等价，嵌套时是上一层 fragment 的 child fm）。
  > - **activity.supportFragmentManager**：宿主 Activity 的顶级 fm。
  >
  > 陷阱：嵌套 ViewPager + Fragment 时用错 fm 会导致 child 泄漏或 state 错乱；`requireParentFragment()` 可拿父。getChildFragmentManager 获取的 fm 在父 destroyView 后仍可能短暂存在（view 销毁但 fragment 实例还在）——"view 生命周期"与"fragment 生命周期"分离是嵌套场景高频 bug 源。

* 为什么 Fragment 不能在 onCreate 里直接操作 View / getContext 为 null 的时机

  > onCreateView 之前 View 未创建；onDestroyView 之后 View 已销毁但 fragment 实例可能还在（返回栈中 hide/replace 后）。中间态用 **isDetached/isAdded/getView** 判断。
  >
  > getContext() 在 onAttach 后有效，onDetach 后置 null——异步回调回来时 fragment 可能已 detach，直接 requireContext() 会崩；用 isAdded() 守卫或 lifecycleScope 自动取消。

* fragment.arguments 传参为什么优于构造函数传参

  > 系统重建 fragment 时调用**无参构造** + 恢复 arguments；带参构造在重建时找不到匹配构造器会崩或参数丢失。所以官方要求：参数放 arguments Bundle，用 newInstance 工厂方法封装。
  >
  > 面试追问：ViewModel 为什么可以带参？——用 SavedStateHandle 或 Navigation 的 args 解析后 set，不依赖构造。

* 面试官视角

  > **考察点**：是否理解 fragment 是"依附 activity 的视图+控制器"，生命周期嵌套关系与 view/fragment 生命周期分离。
  >
  > **加分**：onResume 前后顺序、setRetainInstance 废弃与 ViewModel 替代、childFragmentManager 语义、无参构造恢复机制。
  >
  > **减分**：把 fragment 当独立 activity；用带参构造；不知道 onDestroyView 后实例可能还活着。

---

* add / replace / show / hide 四种事务操作的本质区别

  > - **add(containerViewId, fragment)**：把 fragment 的 View 加入容器，原可见 fragment 不移除（叠加）。多个 add 会层叠，需配合 hide。
  > - **replace**：先 remove 容器内已有 fragment（若在回退栈中则 detach），再 add 新的。视觉上"替换"。
  > - **show / hide**：只改 fragment View 的 visibility（GONE/VISIBLE），**不触发 onCreateView/onDestroyView**，onHiddenChanged 回调。适合底部导航栏等"保留状态、切换显示"场景。
  >
  > 性能差异：show/hide 保留 View 树与状态，切换快但内存占用高；replace 走完整销毁重建（回退栈中可能 detach 保实例），省内存但重 onCreateView。
  >
  > 现代实践：单 Activity + Navigation + 单 fragment 容器 + replace，状态交给 ViewModel；多 tab 用 show/hide 或 ViewPager2（各有取舍）。

* addToBackStack 的语义与常见误解

  > addToBackStack(name) 把**这一组事务**压入 FragmentManager 的回退栈：按返回键时**逆序回放**撤销操作（add 变 remove、show 变 hide、replace 恢复前一个）。
  >
  > 误解1："add 不需要 back stack"——不 addBackStack 的 add/replace，返回键不会回到前一个 fragment，直接走 activity finish。
  >
  > 误解2："replace 总是 detach 保实例"——有回退栈时被替换 fragment 走 onDestroyView 但实例保留（view 销毁）；无回退栈直接 remove 销毁。
  >
  > popBackStack 是异步的（内部 enqueue），立即执行后续事务要注意顺序。

* FragmentTransaction commit 的三种时机：commit / commitAllowingStateLoss / commitNow

  > - **commit()**：加入主线程消息队列异步执行；在 Activity onSaveInstanceState 之后调用抛 IllegalStateException（"Can not perform this action after onSaveInstanceState"）——高频线上崩溃。
  > - **commitAllowingStateLoss()**：允许状态丢失，不抛异常但可能"这次切换在重建后丢失"。是止血不是根治。
  > - **commitNow() / commitNowAllowingStateLoss()**：同步执行完事务（不能 addToBackStack）。
  >
  > 根治思路：把事务时机约束在生命周期内（lifecycleScope.launchWhenResumed / 在 STARTED 前），或用 Navigation 组件封装避免手写时机错误。

* show/hide 的 onHiddenChanged 与默认选中第一个 tab 的坑

  > 用 show/hide 做底部导航：初次 add 第一个 fragment 后，后续 tab 切换 show 新的 + hide 旧的。注意 onHiddenChanged 只在显隐变化时回调，**初次 add 不触发**——懒加载初始化要额外处理（isHidden 判断 + 第一次手动触发）。
  >
  > 另一坑：只 show 不 hide 会层叠；只 hide 不 save state 可能丢滚动位置（hide 保留 View，通常状态在）。
  >
  > 替代：ViewPager2 + FragmentStateAdapter（自带懒加载语义，Fragment 的 setUserVisibleHint 已废弃，改用 resume 时机判断）。

* replace + 回退栈下，返回键的真实行为（追问链）

  > 事务 [replace(R.id.container, B).addToBackStack("t")] 提交时：A 走 onDestroyView（实例保留在 fm，View 销毁），B add 进容器。按返回：popBackStack 异步撤销——B remove，A 走 onCreateView 重建 View（onCreateView→onViewCreated→onStart→onResume）。
  >
  > 若 A 依赖"重建后恢复滚动位置"，依赖 viewHierarchyState 自动恢复（需 view id 稳定）；数据则从 ViewModel 重读。
  >
  > 面试官可能追问："A 的 onDestroyActivity 之外还有什么没释放？"——A 的 View 已 destroy，但 A 实例、其持有的非 View 引用还在栈里；若 A 持有 Activity Context 引用（如 getContext 长持有），在长回退栈下是泄漏隐患。

* 面试官视角

  > **考察点**：四种事务的生命周期后果与回退栈回放机制，commit 时机崩溃的根因。
  >
  > **加分**：show/hide 不销毁 View 的取舍、commit 异步性与 state loss 崩溃根治、replace 在回退栈中 onDestroyView 而非销毁。
  >
  > **减分**：混用四种操作无章法；只会 commitAllowingStateLoss 止血；不知道返回键是"逆序回放事务"。

---

* Fragment 与 Activity/Fragment 之间通信的推荐方案及失效边界

  > 1. **共享 ViewModel（首选）**：同一 ViewModelStoreOwner（activity 级 activityViewModels）内的 fragment 共享 StateFlow/LiveData。失效边界：仅限同宿主；跨宿主或进程死亡需 SavedStateHandle。
  > 2. **Fragment Result API**：`Fragment-result`（setFragmentResult / setFragmentResultListener），本质是 FragmentManager 内部事件总线，按 requestKey 订阅。适合一次性结果回传（如选完颜色回传父 fragment），比 EventBus 轻、有生命周期安全。
  > 3. **导航参数（Navigation SafeArgs）**：direction 传参类型安全，本质仍是 Bundle + 类型封装。
  > 4. **接口回调 + onAttach/onDetach 绑定解绑**：传统方式，fragment 定义 listener 接口，宿主实现；onDetach 须置空防泄漏。
  > 5. **EventBus/SharedFlow 全局**：跨模块解耦，但需防粘性事件与忘记注销。
  >
  > 选型：同宿主数据共享 → ViewModel；一次性结果 → Fragment Result API；跨页面 → Navigation/事件总线。避免 fragment 直接 findFragmentByTag 强转调方法（脆弱且时序难控）。

* ViewPager2 + Fragment 的正确用法与懒加载语义演进

  > ViewPager2 基于 RecyclerView 实现，adapter 必须是 **FragmentStateAdapter**。要点：
  > - offscreenPageLimit：预加载页数（默认 RecyclerView 预取 + 1）；设为 OFFSCREEN_PAGE_LIMIT_DEFAULT 用系统默认，不要乱设 MAX 防内存爆。
  > - setUserVisibleHint 已废弃；判断"可见性"用 `Fragment.onResume`/`onPause`（跟随 view 生命周期）或 lifecycle 观察 RESUMED。
  > - 用户滑动中途切 tab 可能导致 fragment 重建；关键数据放 ViewModel 不放 fragment 字段。
  > - 与 NestedScroll 冲突：ViewPager2 内嵌 RV，外层再套 RV 需正确处理嵌套滚动（RecyclerView 内嵌 ViewPager2 横向是常见布局）。

* FragmentStateAdapter 如何管理 Fragment 生命周期（与 FragmentPagerAdapter 对比）

  > - **FragmentPagerAdapter**（废弃）：destroyItem 只 detach 不 remove，fragment 实例常驻内存，适合少量固定页面；大数据量会堆积泄漏。
  > - **FragmentStateAdapter**：destroyItem 走 remove，靠 saved state 恢复；真正按需创建销毁，适合动态页面列表。
  >
  > FragmentStateAdapter 内部用 FragmentManager 的 tag 管理（itemId + position），getItemId/getItemPosition 决定 diff 行为；数据源变化时调 notifyDataSetChanged 或用 ListAdapter 风格的 diff。

* 预测式返回（Predictive Back）与 Fragment 返回栈

  > Android 13+ 预测式返回手势需要应用开启 android:enableOnBackInvokedCallback。Navigation 组件 2.4+ 支持；手写 popBackStack 需改用 OnBackPressedCallback 注册，旧 onBackPressed 已废弃。
  > Fragment 回退栈在预测式返回下的动画与状态：系统会询问是否有可返回的条目（FragmentManager.getBackStackEntryCount > 0 时消费手势）。面试若被问"预测式返回对 Fragment 影响"，核心是：回调 API 从 onBackPressed 改为 OnBackPressedCallback + Enabled 状态控制。

* 面试官视角

  > **考察点**：通信方案选型是否有边界意识；ViewPager2 生命周期语义是否踩过 setUserVisibleHint 的坑。
  >
  > **加分**：Fragment Result API、FragmentStateAdapter vs PagerAdapter 内存模型、预测式返回回调迁移。
  >
  > **减分**：只会 EventBus；推荐 FragmentPagerAdapter 做大列表；不知道 setUserVisibleHint 已废弃。
