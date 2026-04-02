## RecyclerView？

**关键成员变量**
mLayout: RecyclerView.LayoutManager类型，负责布局。
mState: RecyclerView.State类型。保存了很多状态位。其中mState.mLayoutStep有三种状态STEP_START，STEP_LAYOUT，STEP_ANIMATIONS。满足一个状态转移过程：**STEP_START -> dispatchLayoutStep1 -> STEP_LAYOUT -> dispatchLayoutStep2 -> State.STEP_ANIMATIONS -> dispatchLayoutStep3 -> STEP_START.**
mAdapter: RecyclerView.Adapter类型。负责ViewHolder的创建和数据绑定。
mRecycler: RecyclerView.Recycler类型。负责VH的复用，也就是缓存池，是缓存机制的核心。

**3大layout方法作用**
dispatchLayoutStep1: 本方法的作用主要有三点：
1.处理Adapter更新;
2.决定是否执行ItemAnimator;
3.保存ItemView的动画信息。本方法也被称为preLayout(预布局)，当Adapter更新了，这个方法会保存每个ItemView的旧信息(oldViewHolderInfo)
dispatchLayoutStep2: 在这个方法里面，真正进行children的测量和布局。
dispatchLayoutStep3: 这个方法的作用执行在dispatchLayoutStep1方法里面保存的动画信息。

**onMeasure过程**分3种情况：

1. **当mLayout为空的时候**，会按照specMode测量尺寸，但是onLayout阶段不会布局，也就是不会有数据显示。所以这个mLayout也是我们每次初始化Rv必传的参数。
2. **当LayoutManager开启了自动测量**。常用的LinearLayoutManager是开启的。会依次执行dispatchLayoutStep1、dispatchLayoutStep2，如果需要二次测量，还需要执行dispatchLayoutStep2。
   dispatchLayoutStep1和动画有关，第一次加载数据，是不会执行动画的。这个方法先按下不表。
   dispatchLayoutStep2真正负责layoutChildren. 先通过mAdapter获取itemCount，然后调用mLayout.onLayoutChildren(mRecycler, mState)。onLayoutChildren由各个RecyclerView.LayoutManager子类实现，比如我们常用的LinearLayoutManager。
3. **当没有开启自动测量。** 如果mHasFixedSize为true(也就是调用了setHasFixedSize方法)，将直接调用LayoutManager的onMeasure方法进行测量。如果mHasFixedSize为false，同时此时如果有数据更新，先处理数据更新的事务，然后调用LayoutManager的onMeasure方法进行测量。

**onLayout过程**
onLayout主要调用dispatchLayout。
dispatchLayout主要保证RV必须经历的3大步骤，dispatchLayoutStep1、dispatchLayoutStep2、dispatchLayoutStep3。
RecyclerView跟其他ViewGroup不同的地方在于，如果开启了自动测量，在measure阶段，已经将Children布局完成了；如果没有开启自动测量，则在layout阶段才布局Children。

**onDraw**draw干了三件事：

1. 调用super.draw方法。这里主要做了两件事：
   a. 将Children的绘制分发给ViewGroup;
   b. 将分割线的绘制分发给ItemDecoration。
2. 如果需要的话，调用ItemDecoration的onDrawOver方法。通过这个方法，我们在每个ItemView上面画上很多东西。
3. 如果RecyclerView调用了setClipToPadding,会实现一种特殊的滑动效果--每个ItemView可以滑动到padding区域。

RV的绘制流程看起来比较简单，但具体在LayoutManager里实现的layoutChildren是比较复杂的。直接看一下LinearLayoutManager的实现。

**LinearLayoutManager.onLayoutChildren！！！！！！**
关联成员：
mAnchorInfo: LinearLayoutManager.AnchorInfo类型的数据类。用于保存锚点信息。锚点信息在LLM执行layout时候，可以提供参考。

**1. 先更新锚点信息，mAnchorInfo的计算**三种方式：

1. 第一种计算方式，表示含义有两种：1.RecyclerView被重建，期间回调了onSaveInstanceState方法，所以目的是为了恢复上次的布局；2.RecyclerView调用了scrollToPosition之类的方法，所以目的是让
   RecyclerView滚到准确的位置上去。所以，锚点的信息根据上面的两种情况来计算。
2. 第二种计算方法，从Children上面来计算锚点信息。这种计算方式也有两种情况：1. 如果当前有拥有焦点的Child，那么有当前有焦点的Child的位置来计算锚点；2. 如果没有child拥有焦点，那么根据布局方向(此时布局方向由mLayoutFromEnd来决定)获取可见的第一个ItemView或者最后一个ItemView。
3. 如果前面两种方式都计算失败了，那么采用第三种计算方式，也就是默认的计算方式。

**2. 调用detachAndScrapAttachedViews对所有itemView进行回收**
涉及到缓存机制，后续再说。

**3. 根据锚点的布尔值mLayoutFromEnd（填充方向）和锚点（填充位置）信息，去用fill填充children**需要知道的是，不管是哪个方向，都需要两次fill。以mLayoutFromEnd==true为例，先从锚点向start填充，再从锚点向end填充。fill方法内，会计算可用空间，然后循环调用**layoutChunk**来完成单个child的填充动作。layoutChunk步骤如下：

1) 调用LayoutState的**next**方法获得一个ItemView。千万别小看这个next方法，RecyclerView缓存机制的起点就是从这个方法开始，可想而知，这个方法到底为我们做了多少事情。
2) 如果RecyclerView是第一次布局Children的话(layoutState.mScrapList == null为true)，会先调用addView，将View添加到RecyclerView里面去。
3) 调用measureChildWithMargins方法，测量每个ItemView的宽高。注意这个方法测量ItemView的宽高考虑到了两个因素：1.margin属性；2.ItemDecoration的offset。
4) 调用layoutDecoratedWithMargins方法，布局ItemView。这里也考虑上面的两个因素的。

**缓存机制！！！！！！！**
**四级缓存**
一级: mAttachedScrap和mChangedScrap。均在Recycler中。其中mAttachedScrap存储的是当前还在屏幕中的ViewHolder，mChangedScrap存储的是数据被更新的ViewHolder,比如说调用了Adapter的notifyItemChanged方法。
二级: mCachedViews。默认大小为2，通常用来存储预取的ViewHolder，同时在回收ViewHolder时，也会可能存储一部分的ViewHolder，这部分的ViewHolder通常来说，意义跟一级缓存差不多。
三级: ViewCacheExtension。自定义缓存,通常用不到。
四级: RecyclerViewPool。根据ViewType来缓存ViewHolder，每个ViewType的数组大小为5，可以动态的改变。

**VH的状态**VH自身有很多状态机isInvalid: 对应FLAG_INVALID，表示当前ViewHolder是否已经失效。通常来说，在3种情况下会出现这种情况：1.调用了Adapter的notifyDataSetChanged方法；2. 手动调用RecyclerView的invalidateItemDecorations方法；3. 调用RecyclerView的setAdapter方法或者swapAdapter方法。isRemoved: 对应FLAG_REMOVED，表示当前的ViewHolder是否被移除。通常来说，数据源被移除了部分数据，然后调用Adapter的notifyItemRemoved方法。isBound: 对应FLAG_BOUND，表示当前ViewHolder是否已经调用了onBindViewHolder。isTmpDetached: 对应FLAG_TMP_DETACHED，表示当前的ItemView是否从RecyclerView(即父View)detach掉。通常来说有两种情况下会出现这种情况：

1. 手动了RecyclerView的detachView相关方法；
2. 在从mHideViews里面获取ViewHolder,会先detach掉这个ViewHolder关联的ItemView。mHideViews先按下不表。
   isScrap: 无Flag来表示该状态，用mScrapContainer是否为null来判断。表示是否在mAttachedScrap或者mChangedScrap数组里面，进而表示当前ViewHolder是否被废弃。
   isUpdated: 对应FLAG_UPDATE。表示当前ViewHolder是否已经更新。通常来说，在3种情况下会出现情况：
3. isInvalid方法存在的三种情况；
4. 调用了Adapter的onBindViewHolder方法；
5. 调用了Adapter的notifyItemChanged方法。

**复用和回收***复用*前面提到的layout一个child的layoutChunk方法，获取用到的VH是用的LayoutState.next方法。next方法里，先忽略从scrapList获取VH的逻辑。主要逻辑是通过recycler.getViewForPosition获取下一个itemView。最终会调用到tryGetViewHolderForPositionByDeadline。过程大致如下：

- 如果是preLayout阶段，也就是**dispatchLayout1**过程中，则从**mChangedScrap**里获取；前面提到只有当ItemAnimator不为空，被changed的ViewHolder会放在mChangedScrap数组里面。这里可以理解为change动画前后的VH是不同的，所以当预布局时，从mChangedScrap缓存里面去，而正式布局时，不会从mChangedScrap缓存里面去，这就保证了动画前后相同位置上是不同的VH。关于细节后续再提。
- 如果没有获取到，则分别从**mAttachedScrap、 mHiddenViews、mCachedViews**获取ViewHolder。如果获取的ViewHolder是无效的，得做一些清理操作，然后重新放入到缓存里面，具体对应的缓存就是mCacheViews（二级）和RecyclerViewPool（四级）。回收操作后续讲。
- 前面是通过position获取VH，如果position没有获取到VH，则用viewType来获取。前面position能获取到，viewType也会是正确的。
  首先会判断Adapter的hasStableIds的返回结果，如果是true，则优先通过ViewType和id两个条件来寻找（从mAttachedScrap和mCachedViews里）。id是在adapter的实现类里覆写getItemId方法获取的。
  如果为false，首先会在ViewCacheExtension里面找，如果还没有找到的话，最后会在RecyclerViewPool里面来获取ViewHolder。**RecyclerViewPool用SparseArray维护了每个ViewType对应的VH数组，每个数组最大size是5。**这里就是根据viewType去找取一个VH。
  如果以上的复用步骤都没有找到合适的ViewHolder，最后就会调用**Adapter的onCreateViewHolder**方法来创建一个新的ViewHolder。

*回收*

1. 回收到scrap。主要看scrapView的调用时机。

   1) 在getScrapOrHiddenOrCachedHolderForPosition方法里面，如果从mHiddenViews获得一个ViewHolder的话，会先将这个ViewHolder从mHiddenViews数组里面移除，然后调用Recycler的scrapView方法将这个ViewHolder放入到scrap数组里面，并且标记FLAG_RETURNED_FROM_SCRAP和FLAG_BOUNCED_FROM_HIDDEN_LIST两个flag。
   2) 在LayoutManager里面的scrapOrRecycleView方法也会调用Recycler的scrapView方法。而有两种情形下会出现如此情况：1. 手动调用了LayoutManager相关的方法;2. RecyclerView进行了一次布局(调用了requestLayout方法)
2. 回收到mCacheViews。主要看recycleViewHolderInternal的调用时机。

   1) 在重新布局回收了。这种情况主要出现在调用了Adapter的notifyDataSetChange方法,并且此时Adapter的hasStableIds方法返回为false。从这里看出来，为什么notifyDataSetChange方法效率为什么那么低，同时也知道了为什么重写hasStableIds方法可以提高效率。因为notifyDataSetChange方法使得RecyclerView将回收的ViewHolder放在二级缓存，效率自然比较低。
   2) 在复用时，从一级缓存里面获取到ViewHolder，但是此时这个ViewHolder已经不符合一级缓存的特点了(比如Position失效了，跟ViewType对不齐)，就会从一级缓存里面移除这个ViewHolder，从添加到mCacheViews里面
   3) 当调用removeAnimatingView方法时，如果当前ViewHolder被标记为remove,会调用recycleViewHolderInternal方法来回收对应的ViewHolder。调用removeAnimatingView方法的时机表示当前的ItemAnimator已经做完了。
3. 回收到mHiddenViews。
   个ViewHolder回收到mHiddenView数组里面的条件比较简单，如果当前操作支持动画，就会调用到RecyclerView的addAnimatingView方法，在这个方法里面会将做动画的那个View添加到mHiddenView数组里面去。通常就是动画期间可以会进行复用，因为mHiddenViews只在动画期间才会有元素。
4. 回收到RecyclerViewPool。
   RecyclerViewPool跟mCacheViews,都是通过recycleViewHolderInternal方法来进行回收，所以情景与mCacheViews差不多，只不过当不满足放入mCacheViews时，才会放入到RecyclerViewPool里面去。

推荐参考：https://juejin.cn/post/6958962329220284453

**按照具体动作看内存作用**
列表项有4项，分别展示1、2、3、4四个数字。
*1). 更改数据4变为5，notifyItemChange发生了什么？*
4、2、1倒叙放进attachedScrap里，3放进changedScrap里。然后create和bind一个新的vh，作为5的容器。然后1、2、5、4的layout顺序，将vh添加到视图中。验证了发生变化的vh会添加到changedScrap里。但是有两个问题没搞懂，一个是为什么放到changedScrap里的3数据已经变成了5，另外是所有itemChange都会create吗？
*2). 移除3，notifyItemRemoved发生了什么？*
4、3、2、1倒叙放进attachedScrap里，然后1、2、4重新添加到布局里，3直接删除。

列表项有4项，分别展示1、2、3、4四个数字。
*3). 列表共有10条数据，铺满用7条。滑动过程，缓存的变化？*
向上滑动，第1个vh滑出屏幕，放入cachedViews；第8条进入屏幕。cachedViews默认大小是2，所以当第2个vh滑出时，也放进去；第9条进入屏幕。此时向下滑，会复用cachedViews里的vh，取出第2条；同时第9条滑出屏幕，放入cachedView。
当cachedView空间不足（=2），则会利用mRecyclerPool。mRecyclerPool会按照itemType存储vh。作用和cachedViews相似，只不过需要重新bindData。
因此，总而言之，cachedViews和mRecyclerPool是用于滑动期间复用vh的。

参考：https://mp.weixin.qq.com/s/SqjGeGW2c-BhmO5kW7kSrA


### RecyclerView的局限性

#### 布局与 UI 方面的局限

**1. 高度测量与自适应问题**

* **局限性**：RecyclerView 本身是一个高度可滚动的容器，它**不支持 `wrap_content` 与动态高度自适应**。如果将一个包含大量数据的 RecyclerView 放在 ScrollView 中，并希望它根据内容高度自动撑开，会出问题——要么完全不显示，要么高度计算为 0。
* **根本原因**：RecyclerView 的设计初衷是高效的回收复用，它需要明确的测量约束（如 `match_parent` 或固定高度）来确定可滚动的范围。如果使用 `wrap_content`，它需要测量所有子项才能确定自身高度，这完全违背了回收机制的初衷，会导致极其严重的性能问题。
* **常见问题场景**：**嵌套 RecyclerView**（如垂直滑动父列表，内部嵌套水平滑动子列表）或**将 RecyclerView 放入 ScrollView**。这会导致内部 RecyclerView 完全失去回收复用效果，一次性创建所有不可见 Item，造成严重卡顿。

**2. 复杂布局的实现成本**

**局限性**：实现一个**瀑布流（StaggeredGridLayout）**中 Item 位置动态交换，或者实现**吸顶效果**（Sticky Header），RecyclerView 本身不直接支持。需要开发者自己通过 `ItemDecoration` 进行复杂的逻辑计算，或者引入第三方库（如 `StickyHeaderDecoration`），增加了实现难度和维护成本。


#### 性能与数据更新方面的局限

1. **全局刷新导致视觉闪烁**
   * **常见问题**：新手开发者习惯在数据变化后调用 `notifyDataSetChanged()`。这个方法的成本很高，它会**禁用所有动画**，导致 RecyclerView 短暂白屏/闪烁，并且所有可见的 Item 都会重新创建和绑定，浪费性能。
   * **应对方式**：必须使用更精准的 `notifyItemInserted`、`notifyItemRemoved` 等方法，或者使用 `DiffUtil` 自动计算差异并派发精确的更新。
2. **复杂数据更新的性能瓶颈**
   * **局限性**：`DiffUtil` 虽然是神器，但在计算**万级数据量**的差异时，它的 `DiffUtil.calculateDiff()` 方法**会在主线程中执行**，如果 diff 算法复杂，依然可能造成 UI 卡顿。
   * **根本原因**：它使用的是 Eugene W. Myers 的差异算法，时间复杂度为 O(N)。对于超大数据集，可以考虑将 diff 计算放到后台线程（`ListAdapter` 默认支持后台计算吗？实际上 `AsyncListDiffer` 是主线程计算的，需要配合 `Executor` 自己实现后台 diff）。



**3. 滚动性能受 Item 复杂度影响**

* **常见问题**：如果 Item 的布局嵌套过深（如 `ConstraintLayout` 过度约束、多层 `LinearLayout`），或者 `onBindViewHolder` 中执行了耗时操作（如 IO 读取、位图解码），即使有缓存机制，**首次绑定和滑动中的绑定阶段**依然会造成卡顿。
