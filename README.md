微信抢红包插件
===================

这个Android插件可以帮助你在微信群聊抢红包时战无不胜。当检测到红包时，插件会自动点击屏幕，和人工点击的速度不可同日而语。

你正在查看的是[**dev分支**](https://github.com/geeeeeeeeek/WeChatLuckyMoney/tree/dev)，这个分支仍在开发中，如果你希望有一个可以立即使用的插件请切换到[**stable分支**](https://github.com/geeeeeeeeek/WeChatLuckyMoney/tree/stable)。

下面的文档仅针对**dev分支**。


预期特性
-------------
1. 可以抢屏幕上显示的所有红包，而同类插件往往只能获取最新的一个红包。

2. 智能跳过已经戳过的红包，避免频繁点击影响正常使用。

3. 红包日志功能，方便查看抢过的红包内容和金额。

4. 性能优化，感受不到插件的存在，可一直后台开启，丝毫不影响日常聊天。

5. 由于这是一份教学代码，项目的文档和注释都比较完整，代码适合阅读。

	> **注：** 其中一些功能还存在一些问题，仍在调试中。


实现原理
-------------------
### 1. 抢红包流程的逻辑控制

这个插件通过一个Stage类来记录当前对应的阶段。Stage类被设计成单例并惰性实例化，因为一个Service不需要也不应该处在不同的阶段。对外暴露阶段常量和entering和getCurrentStage两个方法，分别记录和获取当前的阶段。

````java

public class Stage {

    private static Stage stageInstance;

    public static final int FETCHING_STAGE = 0, OPENING_STAGE = 1, FETCHED_STAGE = 2, OPENED_STAGE = 3;

    private int currentStage = FETCHED_STAGE;

    private Stage() {}

    public static Stage getInstance() {
        if (stageInstance == null) stageInstance = new Stage();
        return stageInstance;
    }

    public void entering(int _stage) {
        stageInstance.currentStage = _stage;
    }

    public int getCurrentStage() {
        return stageInstance.currentStage;
    }
}

````

#### 阶段说明

|阶段 | 说明|
|---|---|
|FETCHING_STAGE|正在读取屏幕上的红包，此时不应有别的操作|
|FETCHED_STAGE|已经结束一个FETCH阶段，屏幕上的红包都已加入待抢队列|
|OPENING_STAGE|正在拆红包，此时不应有别的操作|
|OPENED_STAGE|红包成功抢到，进入红包详情页面|


1. 程序以FETCHED_STAGE 开始，将屏幕上的红包加入待抢队列：

	--> FETCHED_STAGE --> FETCHING_STAGE  --> FETCHED_STAGE -->

  
2. 处理待抢队列中的红包：

	--> [CLICK] --> OPENING_STAGE --> [CLICK] --> OPENED_STAGE --> [BACK] --> FETCHED_STAGE -->（抢到）

	--> [CLICK] --> OPENING_STAGE --> [BACK] --> FETCHED_STAGE -->（没抢到）

3. 不断重复流程1和2


### 2. 屏幕内容检测和自动化点击的实现

和其他插件一样，这里使用的是Android API提供的AccessibilityService。这个类位于android.accessibilityservice包内，该包中的类用于开发无障碍服务，提供代替或增强的用户反馈。
	
AccessibilityService 服务在后台运行，等待系统在发生 AccessibilityEvent 事件时回调。这些事件指的是用户界面上显示发生的状态变化， 比如焦点变更、按钮按下等等。服务可以请求“查询当前窗口中内容”的能力。 开发辅助服务需要继承该类并实现其抽象方法。

#### 2.1 配置AccessibilityService
在这个例子中，我们需要监听的事件是当红包来或者滑动屏幕时引起的屏幕内容变化，和点开红包时窗体状态的变化，因此我们只需要在配置XML的accessibility-service标签中加入一条
````xml
android:accessibilityEventTypes="typeWindowStateChanged|typeWindowContentChanged"
````
或在onAccessibilityEvent回调函数中对事件进行一次类型判断
````java
final int eventType = event.getEventType();
if (eventType == AccessibilityEvent.TYPE_WINDOW_STATE_CHANGED
     || eventType == AccessibilityEvent.TYPE_WINDOW_CONTENT_CHANGED) {
	     // ...
}
````
 
 除此之外，由于我们只监听微信，还需要指定微信的包名
 ````xml
android:packageNames="com.tencent.mm"
````
为了获取窗口内容，我们还需要指定
````xml
android:canRetrieveWindowContent="true"
````
其他配置请看代码。

#### 2.2 获取红包所在的节点
首先，我们要获取当前屏幕的根节点，下面两种方式效果是相同的：
````java
AccessibilityNodeInfo nodeInfo = event.getSource();

AccessibilityNodeInfo nodeInfo = getRootInActiveWindow();
````

这里返回的AccessibilityNodeInfo是窗体内容的节点，包含节点在屏幕上的位置、文本描述、子节点Id、能否点击等信息。从AccessibilityService的角度来看，窗体上的内容表示为辅助节点树，虽然和视图的结构不一定一一对应。换句话说，自定义的视图可以自己描述上面的辅助节点信息。当辅助节点传递给AccessibilityService之后就不可更改了，如果强行调用引起状态变化的方法会报错。

在聊天页面，每个红包上面都有“领取红包”这几个字，我们把它作为识别红包的依据。因此，如果你收到了这四个字的文本消息，那插件会做出误判，并没有什么办法可以解决。

AccessibilityNodeInfo的API中有一个findAccessibilityNodeInfosByText方法允许我们通过文本来搜索界面中的节点。匹配是大小写敏感的，它会从遍历树的根节点开始查找。API文档中特别指出，为了防止创建大量实例，节点回收是调用者的责任，这一点会在接下来的部分中讲到。
````java
List<AccessibilityNodeInfo> node1 = nodeInfo.findAccessibilityNodeInfosByText("领取红包");
````

#### 2.3 对节点进行操作
AccessibilityNodeInfo同样暴露了一个API——performAction来对节点进行点击或者其他操作。处于安全性考虑，只有这个操作来自AccessibilityService时才会被执行。
````java
nodeInfo.performAction(AccessibilityNodeInfo.ACTION_CLICK);
````
不过，我们在调试时发现"领取红包"的mClickable属性为false，说明点击的监听加在它父辈的节点上。通过getParent获取父节点，这个节点是可以点击的。

我们还需要全局的返回操作，最方便的办法就是performGlobalAction，不过注意这个方法是API 16时才有的。
````java
performGlobalAction(GLOBAL_ACTION_BACK);
````

### 3. 获取屏幕上的所有红包

和其他插件最大的区别是，这个插件的逻辑是获取屏幕上所有的红包节点，去掉已经获取过的之后，将待抢红包加入队列，再将队列中的红包一个个打开。

#### 3.1 判断红包节点是否已被抢过
实现这一点是编写时最大的障碍。对于一般的Java对象实例来说，除非被GC回收，实例的Id都不会变化。我最初的想法是通过正则表达式匹配下面的十六进制对象id来表示一个红包。
````
android.view.accessibility.AccessibilityNodeInfo@2a5a7c; .......
````
但在测试中，队列中的部分红包没有被戳开。进一步观察发现，新的红包节点和旧的红包节点id出现了重复，且出现概率较大。由于GC日志正常，我推测AccessibilityNode可能有一个实例池的设计。获取当前窗体节点树的时候，从一个可重用的实例池中获取一个辅助节点信息 (AccessibilityNodeInfo)实例。在接下来的获取时，仍然从实例池中获取节点实例，这时可能会重用之前的实例。这样的设计是有好处的，可以防止每次返回都创建大量的实例，影响性能。AccessibilityNodeProvider的源码表明了这样的设计。

也就是说，为了标识一个唯一的红包，只用实例id肯定是不充分的。这个插件插件采用的是红包内容+节点实例id的hash来标记。因为同一屏下，同一个节点树下的红包节点id是一定不会重复的，滑动屏幕后新红包的内容和节点id同时重复的概率已经大大减小。更改标识策略后，实测中几乎没有出现误判。

#### 3.2 将新出现的红包加入待抢队列

我们维护了两个列表，分别记录待抢红包和抢过的红包标识。
````java
private List<AccessibilityNodeInfo> nodesToFetch = new ArrayList<>();

private List<String> fetchedIdentifiers = new ArrayList<>();
````

在每次读取聊天屏幕的时候，会检查这个红包是否在fetchedIdentifiers队列中，如果没有，则加入nodesToFetch队列。
````java
for (AccessibilityNodeInfo cellNode : fetchNodes) {
    String id = getHongbaoHash(cellNode);
    /* 如果节点没有被回收且该红包没有抢过 */
    if (id != null && !fetchedIdentifiers.contains(id)) {
        nodesToFetch.add(cellNode);
    }
}
````


版权说明
-------------------

本项目源自小米今年秋季发布会时演示的抢红包测试[源码](https://github.com/XiaoMi/LuckyMoneyTool)。stable分支基于此代码继续开发，dev分支重写了几乎所有的逻辑代码。应用的包名com.miui.hongbao未变。

由于插件可能会改变自然的微信交互方式，这份代码仅可用于教学目的，不得更改后用于其他用途。

项目使用MIT许可证。在上述前提下，你可以将代码用于任何用途。