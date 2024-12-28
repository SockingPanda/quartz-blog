---
title: 基于 React Flow 的 Obsidian 任务管理插件
date: 2024-12-09 01:48
tags:
  - 探索历程
status: 活动
public: true
---

---
（2024-12-09) -->

任务管理迫在眉睫，急需一种方式能够快速添加任务、图形化展示任务。于是今天项目创建了！！

>[!abstract]- 比较完善的设计方案
>**1. 核心任务模型**
>每个任务存储为单独的 `.md` 文件，包含以下属性：
>
>```yaml
>---
>title: [任务标题]
>due: [截止日期]  # 格式 YYYY-MM-DD
>priority: [高/中/低]
>status: [未开始/进行中/已完成/阻塞]
>context: [多选标签，例如 家里, 有网络]
>dependencies: [前置任务1, 前置任务2]  # 可选
>subtasks:
>  - 收集参考文献: 1  # 子任务权重
>  - 完成数据分析: 3
>  - 撰写初稿: 2
>progress: [x/x]  # 默认 0/1，表示任务完成度
>manual_weight: [权重数值]  # 可选，用于覆盖自动计算权重
>parent: [主任务标题]  # 可选，仅子任务需要
>---
> 备注
[>任务的详细描述或背景信息]
>```
>
> **2. Context 管理**
>
> **集中式数据管理**
>通过 CSV 文件或表格定义所有可选 context 类别和子项：
>
>```csv
>类别,子项
>环境,家里;学校;办公室
>网络,有网络;无网络
>协作,独立完成;需要协作
>时间,白天;晚上;周末
>```
>
>**实现方式**
>- Templater 动态读取 CSV 数据源生成多选框，确保一致性。
>- 插件支持从 CSV 文件中读取 context，并应用到筛选逻辑中。
>
>
> **3. 动态展示与排序**
>
> **视图逻辑**
>1. **时间视图**：按截止时间分为今日任务、即将到期任务、长期任务、超期任务。
>2. **上下文视图**：支持通过多选上下文筛选，匹配的任务优先展示，未匹配任务排后。
>3. **权重视图**：按手动权重或动态计算权重排序。
>
> **综合排序公式**
>
>动态权重优先计算：
>```
>权重 = 优先级 × 优先级权重 + 1/剩余天数 × 时间权重
>```
>
>- 手动权重 `manual_weight` 优先于自动权重。
>
>
>
>**4. 任务关系**
>
>**前置任务**
>- 属性：`dependencies`
>- 逻辑：
>    - 任务状态为“未开始”，若前置任务未完成则显示为“阻塞”。
>    - 前置任务完成后自动解除阻塞。
>
> **子任务与权重**
>- 属性：`subtasks`
>- 子任务权重默认值为 1，可在主任务中配置。
>- 主任务完成度公式：
>    
>    ```
>    主任务完成度 = Σ(子任务完成度 × 子任务权重) / Σ(子任务权重)
>    ```
>    
>
>
>**5. 任务完成度**
>
>**完成度联动规则**
>
>1. **完成度影响任务状态**：
>    - `progress` 达到 `1` 时，状态自动变为“已完成”。
>2. **任务状态影响完成度**：
>    - 手动将任务状态改为“已完成”，`progress` 的分子自动更新为分母。
>
>**展示**
>
>完成度以 `x/x` 格式表示，并动态显示在任务列表中。
>
>
>**6. 插件的功能规划**
>
>**插件范围**
>
>- **任务展示**：
>    - 按视图动态生成任务列表。
>    - 支持按上下文、时间、权重筛选和排序。
>- **交互功能**：
>    - 支持上下文多选筛选，自动调整排序。
>    - 支持动态更新任务的状态和完成度联动。
>- **任务关系**：
>    - 可视化任务依赖和子任务完成情况。
>- **任务创建**:
>	- 生成表单，填入默认数据（存在多选）
>		- 通过任务箭头延伸创建（固定前置任务）
>		- 通过tab创建（固定父任务）
>		- 双击空白处创建（完整新任务）
>- **任务的详细编辑**:
>	- 双击任务弹出表单修改
>	- 其他方式修改

>[!abstract]- **插件功能概述**
>
>**1. 模板创建**
>
>- **功能**：基于模板快速生成标准化的任务文件。
>- **实现**：
>    - 提供简单直观的任务创建界面。
>    - 支持多选上下文，通过 Context 数据源动态生成选项。
>    - 自动填充默认属性（如 `progress: 0/1`，`manual_weight: null`）。
>- **技术栈**：插件配置 TypeScript + React，支持交互式表单。
>
>**2. 任务展示**
>
>- **功能**：以 React Flow 为基础，图形化展示任务关系。
>- **特点**：
>    - 主任务与子任务通过节点和连线可视化。
>    - 支持任务的展开、收起、查看详情。
>    - 动态展示完成度（如任务的背景做进度条）和状态（背景、边框颜色标记）。
>- **实现**：
>    - 数据解析：读取任务文件，生成节点和关系数据。
>    - 交互功能：
>        - 点击节点查看任务详情。
>        - 拖动调整布局，保存节点位置。
>        - 动态更新完成度和状态联动。
>- **技术栈**：React + React Flow + Obsidian Plugin API。
>
>
>
>**3. 视图转换**
>
>- **功能**：支持多种任务展示视图的切换。
>- **视图类型**：
>    1. **时间视图**：按截止时间分组展示任务。
>    2. **上下文视图**：按当前上下文筛选和排序任务。
>    3. **权重视图**：按动态计算权重排序任务。
>    4. **图形视图**：基于 React Flow 展示任务关系。
>- **交互**：
>    - 用户通过菜单或快捷键快速切换视图。
>    - 支持视图的个性化配置和布局保存。


目前在尝试做的是，设计 React Flow 部分的内容，包括节点设计、连线设计、交互设计、颜色变化等等～

>[!abstract]- React Flow 部分设计
>**1. 节点设计**
>
>**节点基本结构**
>
>- **形状**：矩形。
>- **锚点位置**：矩形的四条边中心，共 4 个锚点（上、下、左、右）。
>- **任务信息展示**：
>    - 标题：任务名称。
>    - 进度：任务完成度（如 `2/5`）。
>    - 状态：通过颜色或标识展示任务状态（如“进行中”、“已完成”）。
>
>**节点示例**
>
>```javascript
>const nodes = tasks.map(task => ({
>  id: task.id,
>  data: {
>    title: task.title,
>    progress: task.progress,
>    status: task.status,
>  },
>  style: {
>    backgroundColor: calculateColor(task.due), // 根据剩余时间设置颜色
>    width: 150,
>    height: 100,
>    border: '1px solid #000',
>  },
>  position: { x: task.x, y: task.y },
>  sourcePosition: 'right',
>  targetPosition: 'left',
>}));
>```
>
>
>
>**2. 连线设计**
>
>**连线类型**
>
>1. **父子关系**：
>    
>    - **连线样式**：实心线。
>    - **方向**：从父任务指向子任务。
>    - **逻辑**：点击父节点锚点，显示/收起所有子任务。
>2. **前后置关系**：
>    
>    - **连线样式**：虚线。
>    - **方向**：从前置任务指向后置任务。
>    - **逻辑**：
>        - 父任务的后置任务等效于所有子任务均指向此后置任务。
>
>**连线示例**
>
>```javascript
>const edges = tasks.flatMap(task => {
>  const childEdges = task.subtasks?.map(sub => ({
>    id: `${task.id}-${sub.id}`,
>    source: task.id,
>    target: sub.id,
>    type: 'default', // 实心线
>    animated: false,
>    style: { strokeWidth: 2 },
>  })) || [];
>
>  const dependencyEdges = task.dependencies?.map(dep => ({
>    id: `${dep}-${task.id}`,
>    source: dep,
>    target: task.id,
>    type: 'dashed', // 虚线
>    animated: true,
>    style: { strokeDasharray: '5,5', strokeWidth: 1 },
>  })) || [];
>
>  return [...childEdges, ...dependencyEdges];
>});
>```
>
>
>
>**3. 节点交互**
>
>**点击锚点收缩/展开子任务**
>
>- **交互效果**：
>    - 点击父节点的锚点，切换显示/隐藏子任务。
>    - 隐藏子任务后，所有子任务的连线逻辑合并到父任务。
>- **实现逻辑**：
>    - 父任务维护一个 `collapsed` 状态。
>    - 更新子任务显示状态时，重新计算父节点连线逻辑。
>
>**交互示例代码**
>
>```javascript
>const toggleSubtasks = (nodeId) => {
>  setNodes(prevNodes =>
>    prevNodes.map(node =>
>      node.id === nodeId ? { ...node, data: { ...node.data, collapsed: !node.data.collapsed } } : node
>    )
>  );
>
>  setEdges(prevEdges => {
>    const relatedEdges = edges.filter(edge => edge.source === nodeId);
>    if (nodes.find(node => node.id === nodeId).data.collapsed) {
>      // 展开子任务
>      return [...prevEdges, ...relatedEdges];
>    } else {
>      // 收起子任务
>      return prevEdges.filter(edge => edge.source !== nodeId);
>    }
>  });
>};
>```
>
>
>
>**4. 特殊连线规则**
>
>**父任务与后置任务**
>
>- 父任务的后置任务等效于所有子任务均指向后置任务。
>- **实现逻辑**：
>    - 当父任务的子任务收起时，将后置任务的连线直接附加到父任务上。
>
>**示例代码**
>
>```javascript
>const updateEdgesForCollapsed = (parentNode) => {
>  const childTasks = nodes.filter(node => node.parent === parentNode.id);
>  const childToDependencyEdges = edges.filter(edge => childTasks.some(child => edge.source === child.id));
>
>  const parentToDependencyEdges = childToDependencyEdges.map(edge => ({
>    ...edge,
>    source: parentNode.id,
>  }));
>
>  setEdges(prevEdges => [
>    ...prevEdges.filter(edge => !childToDependencyEdges.includes(edge)),
>    ...parentToDependencyEdges,
>  ]);
>};
>```
>
>
>
>**5. 颜色动态变化**
>
>**逻辑**
>
>- **父子任务颜色联动**：当子任务完成度更新时，父任务颜色动态改变以反映整体进度。

尝试分析了一下我的任务，发现其实对每个任务的任务cost没有那么了解，就是概念非常模糊，所以感觉可以尝试在任务的属性里面加一个预期 cost 和 实际 cost。

这个 cost 能不能更详细呢？更细致化一下？比如，预期 cost 是一个实际的值，实际的 cost 则是时间段的列表/集合。

创建的时候输入预期 cost。单独做一个界面去显示、用时间轴的形式去选择、添加、调整各个实际的时间段区间。
大致应该是这个样子：

![Pasted image 20241209090919.png](https://cdn.sockingpanda.com/4e4338efdfef324a373a8734b5fb4869.png)

选择任务区间的时候能够高亮时间轴，伸出右侧的侧边栏，侧边栏里面能够显示任务的基本属性，提供部分属性的修改能力。

---
（2024-12-10) -->

大致知道怎么去设计这个节点流了～
这是目前的情况：

![Pasted image 20241210014305.png](https://cdn.sockingpanda.com/d73cf0332188dc0bfd7884933ceef52a.png)

显示还是有点问题的：
- edge
	- 虚线：子任务
	- 实线：依赖关系
- 状态
	- 已完成：绿色
	- 进行中：橘色
	- 未开始：灰色
	- 阻塞：红色

这里应该是 “完成数据分析” 是 “阻塞”，“收集参考文献” 是 “已完成”的，有点小 bug。

之后准备实践这个：[自定义活动节点](https://reactflow.dev/examples/edges/simple-floating-edges)
这样节点之间的连线就更为灵活了！


右侧的选项卡部分：
- 里面的参数暂时没有写全
- 状态流转没有写好
- 父任务完成度无法手动更改（仅自动计算，以节点背景进度条的形式呈现
- 子任务需要一个 check 框，方便切换状态
- 时间段部分
	- 多一行新建，两个按钮，点击左侧的记录当前时间作为开始时间，此时右侧的才能点击；点击右侧的时候记录当前时间作为结束时间。
	- 时间段可手动修改

还有新建任务（新增node）、通过锚点新增任务之类的……

嘛，起来以后先把上面的小问问题修好，尽可能把node、edge 单独拉一个新的自定义出来，方便使用。

title: [任务标题]
due: [截止日期] # 格式 YYYY-MM-DD
priority: [高/中/低]
status: [未开始/进行中/已完成/阻塞]
context: [多选标签，例如 家里, 有网络]
dependencies: [前置任务1, 前置任务2] # 可选
subtasks:
- 收集参考文献: 1 # 子任务权重
- 完成数据分析: 3
- 撰写初稿: 2
progress: [x/x] # 默认 0/1，表示任务完成度（只有末端节点能够操作、修改）
manual_weight: [权重数值] # 可选，用于覆盖自动计算权重
parent: [主任务标题] # 可选，仅子任务需要

修改了一下逻辑，虚线它是会动的，由前置任务流向后置任务。
GPT 老师了解了我的子任务权重，来了一个根据权重去配置 edge 的宽度和箭头的大小，这个想法非常棒！之后也会应用上。
![Pasted image 20241210110911.png](https://cdn.sockingpanda.com/05c42a182ba940528ff2a98d49aad18b.png)

现在想来，一个节点需要展示的信息还是很足的：
- title
- 进度（进行中、已完成、未开始、阻塞）
	- 0%
		- 未开始
		- 阻塞
	- 1%～99%
		- 进行中
	- 100%
		- 已完成
- 截止时间
- 优先级

截止时间和优先级应该是结合来看，来确定边框颜色，截至时间近到一定程度将会跳动 或者 边框转圈圈。

---
（2024-12-11) -->

更新了一波，目前支持
- 分子、分母一个一个加～
- 父任务的进度自动根据权重、子任务进度计算
- 任务状态自动变更
- 颜色随任务状态自动变更
- 
![Pasted image 20241211001403.png](https://cdn.sockingpanda.com/e69b95a38a68fcf8bd08f6fd40ff01fa.png)

后续会把进度用 background 进度条来表示
状态用颜色
优先级用边框渐变色

---
（2024-12-12) -->

![Pasted image 20241212053133.png](https://cdn.sockingpanda.com/7ed42b722dddb45119297a769cf387c8.png)

新增了一个 map 记录节点之间的依赖关系以及父子关系，还能够通过这个 map 直接修改 tasks 的值。

还需改进：
- 自动计算进度 和 状态转移，都需要适配连锁计算。（进度 --> 连锁parent，状态转移 --> 连锁dependince）
- 将update部分代码转移到 tasksService 中


目前修复了 progress 修改后对应文件所在的文件树的更新～

我希望它能够在父节点的卡片中点击 `完成` 后，能够遍历所有它的子树，将它们全部设置为完成。