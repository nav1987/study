当你进行iOS编程时候，其实是进行Cocoa编程。所以你需要熟知Cocoa，知道什么时候和Cocoa打交道以及如何和Cocoa打交道，以及Cocoa期望你的某种行为。Cocoa是一个很大的框架，被分为许多小的框架，需要花一些时间和经历来熟悉它。而且，Cocoa有一些主要的规约和主要的组件，从你开始着手的时候就可以作为指南。
The Cocoa API is written mostly in Objective-C, and Cocoa itself consists mostly of
Objective-C classes, derived from the root class, NSObject. When programming iOS,
you’ll be using mostly the built-in Cocoa classes. Objective-C classes are comparable
to and compatible with Swift classes, but the other two Swift object type flavors,
structs and enums, are not matched by anything in Objective-C. Structs and enums
declared originally in Swift cannot generally be handed across the bridge from Swift
to Objective-C and Cocoa. Fortunately, some of the most important native Swift
object types are also bridged to Cocoa classes. (See Appendix A for more details
about the Objective-C language and how communications work between Swift and
Objective-C.)
This chapter introduces Cocoa’s class structure. It discusses how Cocoa is conceptu‐
ally organized, in terms of its underlying Objective-C features, and then surveys some
of the most commonly encountered Cocoa utility classes, concluding with a discus‐
sion of the Cocoa root class and its features, which are inherited by all Cocoa classes.