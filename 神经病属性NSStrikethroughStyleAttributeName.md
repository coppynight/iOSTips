@(神经病Apple)

问题的起源是工作中遇到的一个app，保密原因隐去了。

该属性在iOS8.1 iOS10.3与其他版本下，分别有不同的表现，真是一个活泼的属性。

### 给Label特定文本划删除线

```objectivec
NSMutableAttributedString *attStr = [[NSMutableAttributedString alloc] initWithString:@"aaabbb"];

[attStr addAttribute:NSStrikethroughStyleAttributeName value:@(NSUnderlinePatternSolid | NSUnderlineStyleSingle) range:NSMakeRange(1, 3)];

```
这段代码，大多数情况下都是完美优雅的方案。

### iOS8.1下
NSStrikethroughStyleAttributeName属性必须从index=0的位置开始（模拟器可以复现），否则无法生效，并且后来的属性会覆盖掉NSStrikethroughStyleAttributeName，所以必须最后设置，如果想要在文本段中间位置划线，代码如下，

```objectivec
NSMutableAttributedString *attStr = [[NSMutableAttributedString alloc] initWithString:@"aaabbb"];
[attStr addAttribute:NSStrikethroughStyleAttributeName value:@(NSUnderlineStyleNone) range:NSMakeRange(0, 1)];
[attStr addAttribute:NSStrikethroughStyleAttributeName value:@(NSUnderlinePatternSolid | NSUnderlineStyleSingle) range:NSMakeRange(1, 3)];
```
是的，拼一段NSUnderlineStyleNone在前面

### iOS10.3下，根本不会响应这个属性
模拟器能正常渲染，真机可复现，解决方案如下，

```objectivec
NSMutableAttributedString *attStr = [[NSMutableAttributedString alloc] initWithString:@"aaabbb"];
[attStr addAttribute:NSStrikethroughStyleAttributeName value:@(NSUnderlinePatternSolid | NSUnderlineStyleSingle) range:NSMakeRange(1, 3)];
[attStr addAttribute:NSBaselineOffsetAttributeName value:@(0) range:NSMakeRange(1, 3)];
```
增加NSBaselineOffsetAttributeName，Baseline就是渲染字体时的基准线。

### 到最后通用的解决方案就变成了

```
NSMutableAttributedString *attStr = [[NSMutableAttributedString alloc] initWithString:@"aaabbb"];
[attStr addAttribute:NSStrikethroughStyleAttributeName value:@(NSUnderlineStyleNone) range:NSMakeRange(0, 1)];
[attStr addAttribute:NSStrikethroughStyleAttributeName value:@(NSUnderlinePatternSolid | NSUnderlineStyleSingle) range:NSMakeRange(1, 3)];
[attStr addAttribute:NSBaselineOffsetAttributeName value:@(0) range:NSMakeRange(1, 3)];
```
    
