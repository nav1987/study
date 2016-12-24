　　当你进行iOS编程时候，其实是进行Cocoa编程。所以你需要熟知Cocoa，知道什么时候和Cocoa打交道以及如何和Cocoa打交道，以及Cocoa期望你的某种行为。Cocoa是一个很大的框架，被分为许多小的框架，需要花一些时间和经历来熟悉它。而且，Cocoa有一些主要的规约和主要的组件，从你开始着手的时候就可以作为指南。　　　　

　　Cocoa API大多数使用Objective-C写的，Cocoa本身包含了大多数的Objective-C类从根类NSObject中继承而来。在进行iOS编程的时候，在大多数的情况下你将使用内置的Cocoa类。Objective-C类和Swift类兼容，但是Swift类中的其它两个类型结构体和枚举，在Objective-C中没有匹配的。在Swift中声明的结构体和枚举不能从Swift桥接到Objective-C和Cocoa中。幸运的是一些在Swift中重要的Swift对象类型可以桥接到Cocoa类。（附录A中详细介绍Swift和Objective-C之间的通信）  
　　这一章介绍Cocoa类结构。将讨论Cocoa如何从理论上组织起来的，并且考虑到底层Objective-C的特点，接着考察了一些最常用的Cocoa工具类，包含被许多类继承的Cocoa根类的特点。
# 子类化 #
　　Cocoa高效的提供类一仓库的对象，这些对象可以按照一定的方式执行。例如UIButton知道如何绘制自己，以及用户触碰它的时候如何响应；UITextField知道如何显示可以编辑的文本，如何唤出键盘。以及如何接收键盘输入。
　　

　　大多数情况下，Cocoa提供对象的默认行为或者默认显示不是你想要的，你想定制它。这并不意味着你需要继承它。Cocoa中的类通常会具有一些方法你可以调用，一些属性你可以设置，准确来说如果需要定制实体，这些是你的第一选择。通常要研究一些文档来看一看Cocoa类实例是否已经可以实现你的需求。例如，UIButton的类文档中已经显示你可以设置按钮的标题，标题颜色，内部图片，背景图片，还有许多其他的属性和行为，不需要继承。  

  　　另外，任何内置的类更愿意使用代理来定制它们的行为。例如，你不需要通过继承UIApplication来响应应用启动完成，因为代理机制提供了完成这一需求的方式，UIApplicationDelegate中的application(_:didFinishLaunchingWithOptions:)来完成这一操作。这就是为何模板中给我们一个AppDelegate类，而不是UIApplication的子类，AppDelegate采用了UIApplicationDelegate协议。  

　　实际上，在Cocoa中进行编码，继承是很罕见的方式。知道什么时候去继承很相当灵活，但是通用的规则是你不应该继承除非你被显示要求这样。（在一些文档中，显示禁止） 


　　然而，在有些时候，设置属性和调用方法或者使用代理并不能满足你定制实例的要求。在这种情况下，Cocoa类可能提供内部调用的方法，就像实例完成的那样，这样就允许你通过集成覆盖的方式定制类的行为。你并不拥有Cocoa内部类的源代码，但是你容然可以继承它，创建一个新类，行为像原先的类那样，除了你提供的一些修改。  


　　一些Cocoa Touch类按照惯例进行继承，的确存在这样的例子。例如，经常使用的UIViewController，不进行继承很罕见，一个iOS应用没有至少一个UIViewController子类，在实际过程中是不可能的。  


　　另外一个例子是UIView。Cocoa Touch中包含了大量的内置的UIView的子类，按成需要来完成自身的绘制，（UIButton, UITextField），你很少需要继承它们。相反，你需要创建自己的UIView子类，它将以某种完全新的方式来绘制自己你不需要明确的绘制一个UIView，当一个UIView需要绘制的时候，draw(_:)方法会被调用，所以这个视图可以绘制自己。所以以定制的方式绘制UIView是通过继承UIView并且实现里面的draw(_:)方法来完成。就像文档中说的那样，子类需要覆盖这一方法来实现他们的绘制代码。文档说为了绘制你自己的内容你需要继承UIView。  

　　例如，假设你想让我们的窗口来绘制水平线。在Cocoa中没有现成的水平线接口插件。我们需要自己来完成绘制水平线。我们来试一下：  

1. 在我们的空窗口例子项目中，选择文件-》新建，指定iOS → Source → Cocoa Touch Class，继承UIView，名字叫MyHorizLine。XCode创建了MyHorizLine.swift。确保它是应用的目标。
2. 在MyHorizLine.swift。中替换一下内容：
```swift
required init?(coder aDecoder: NSCoder) {
super.init(coder:aDecoder)
self.backgroundColor = .clear
}
override func draw(_ rect: CGRect) {
let c = UIGraphicsGetCurrentContext()!
c.move(to:CGPoint(x: 0, y: 0))
c.addLine(to:CGPoint(x: self.bounds.size.width, y: 0))
c.strokePath()
} 
```
3. 编辑storyboard。在Object library中找到UIView，将这个视图对象拖到画布上。你可以调整它让它更窄一些。
4. 完成以上之后，选中刚才拖动的UIView，使用Identity inspector将类改为MyHorizLine。

　　在模拟其中运行应用，你将在看着在MyHorizLine实例的上部绘制了水平线。我们的实例自己绘制了水平线，因为我们继承UIView来完成了这些。

在这个例子中，没有处理的UIView没有绘制功能。所以没有必要调用父类的绘制方法UIView中draw(_:)的默认实现没有做任何事情。你同样可以继承UIView的子类来改变原先的绘制行为。例如，UILabel为我们提供了两个方法来完成此操作。drawText(in:) 和 textRect(forBounds:limitedToNumberOfLines:)明确的告诉我们：这个方法可以被子类重写如果想改变标签的而绘制方式。隐含告诉我们这些方法会在标签绘制的时候被Cocoa自动调用。因此我们可以继承UILabel，在子类的中实现这些方法来更改如何绘制方法。

　　这里有个例子，是我的一个应用中用到的，我继承了UILabel来定制矩形边框绘制，内容离边框有一定的距离，通过重写drawText(in:)来完成的。文档中告诉我们，在你重写的方法中你可以进一步配置当前的绘图环境，然后再调用父类方法来完成真正的绘制。我们来试一下：  

1. 在空的窗体项目中，创建一个新的类文件，MyBoundedLabel继承UILabel。
2. 在MyBoundedLabel.swift中在类的定义中插入如下代码：
···swift
let context = UIGraphicsGetCurrentContext()!
context.stroke(self.bounds.insetBy(dx: 1.0, dy: 1.0))
super.drawText(in: rect.insetBy(dx: 5.0, dy: 5.0))
}
···

3. 编辑storyboard，添加一个UILabel到界面上，选中它，在Identity inspector中将类设置为MyBoundedLabel。

构建运行应用，你将看到矩形如何被绘制，以及标签文本向内部偏移。