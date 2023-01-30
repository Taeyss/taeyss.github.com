## <center>SwiftSVG <center/>
```SwiftSVG```做了一件很简单的事情，读取svg文件，解析svg文件，最终将svg转换成我们的```CALayer```，系统就可以自动渲染出对应的图案。所以最重要的其实就是解析svg和转化上。

我们先看看他的大致项目的组成结构：
- Attributes (元素默认支持的属性，以协议的形式声明)
- Base Eelements （基础的元素，以协议的形式声明）
- Cache （缓存）
- Elements （具体的元素类型）
- Helpers （一些辅助的实现类和扩展）
- Iterators （自定义的迭代器）
- Parser （XML解析）
- Extensions （对外提供的扩展，方便使用）

## 重点介绍
### Attributes
主要声明了6个协议，这些协议都是为元素提供基础的属性操作。我们以其中的```Fillable```为例，其他的基本实现思路都是相同的
```
public protocol Fillable { }
extension Fillable where Self : SVGShapeElement {
    var fillAttributes: [String : (String) -> ()] {
        return [
            "color": self.fill,
            "fill": self.fill,
            "fill-opacity": self.fillOpacity,
            "fill-rule": self.fillRule,
            "opacity": self.fillOpacity,
        ]
    }
    func fill(fillColor: String) {
        guard let colorComponents = self.svgLayer.fillColor?.components else {
            return
        }
        guard let fillColor = UIColor(svgString: fillColor) else {
            return
        }
        self.svgLayer.fillColor = fillColor.withAlphaComponent(colorComponents[3]).cgColor
    }
}
```
这里的协议，描述了他所支持的元素的属性，这里很重要的思想是，他在声明支持的属性的时候，字典的值是一个只有一个字符串入参的函数，这里这个类型的函数后面还会有。这里函数的好处是，在处理各个元素的属性的时候，不用关心具体的方法名称，只需要去调用方法就行了，调用的地方是没有耦合的。

### Base Eelements
主要声明了三个协议：
```SVGElement```、```SVGContainerElement```和```SVGShapeElement```
```SVGElement```协议很简单，他是最基础的元素协议，声明了所有元素的最基础的属性和方法，一个抽象协议
```
public protocol SVGElement {
    /// 元素的标准化名称（path、line、group等）
    static var elementName: String { get }
    /// 元素支持的属性，注意每个属性的值是一个函数，并且有个字符串的入参
    var supportedAttributes: [String : (String) -> ()] { get set }
    /// 元素解析完成，注意container参数是该元素的父级元素
    func didProcessElement(in container: SVGContainerElement?)
}

```
```SVGContainerElement```声明的是一种容器元素，遵循这个协议的元素，可以将其他元素添加到他所创建的layer上。同时可以看到这个元素协议已经开始继承多个实际的元素属性协议，可以知道该协议对象已经是一个具体的协议对象，不再是一个抽象协议。比如具体的```group```元素，就会遵循这个协议。
```
public protocol SVGContainerElement: SVGElement, DelaysApplyingAttributes, Fillable, Strokable, Transformable, Stylable, Identifiable {
    /// 可以将其他元素的layer添加到该layer层上
    var containerLayer: CALayer { get set }
}

```
```SVGShapeElement```声明的是形状元素，一般除了容器元素，剩下的都是形状元素。即会生成相应形状的layer的元素。
```
public protocol SVGShapeElement: SVGElement, Fillable, Strokable, Transformable, Stylable, Identifiable {
    /// 生成的layer对象
    var svgLayer: CAShapeLayer { get set }
}
extension SVGShapeElement {
    /// layer的大小，在解析每个形状元素后，将每个layer的大小进行合并
    var boundingBox: CGRect? {
        return self.svgLayer.path?.boundingBox
    }
}
```
这里声明的3个协议，定义了所有元素的会产生的行为，真正的面向协议编程。
### Elements
这里创建了支持的具体元素类：```SVGCircle```、```SVGEllipse```、```SVGPolyline```等等。其中大部分是形状元素，```SVGRootElement```和```SVGGroup```是容器元素。\
这里需要说明的一点是，因为这里的元素都是遵循```SVGElement```的，所以在实现```supportedAttributes```属性时，会传入下面这种函数：
```
  internal func radius(r: String) {
        guard let r = CGFloat(lengthString: r) else {
            return
        }
        self.circleRadius = r
    }
    internal func yCenter(y: String) {
        guard let y = CGFloat(lengthString: y) else {
            return
        }
        self.circleCenter.y = y
    }
```
即在解析到元素所支持的属性的时候，将解析到的属性值通过这种函数调用的方式，传入到对应的元素内部。
### Iterators
在进行元素的属性解析的时候，现在有这样两种场景，你会怎么去进行解析？
1. 
```
points="20,100 40,60 70,80 100,20"
```
2. 
``` 
d="M31.38,50.08 C31.1365386,49.979982 30.8634614,49.979982 30.62,50.08 C30.4988151,50.1306419 30.3872009,50.2016691 30.29,50.29 C30.1072663,50.4816338 30.0036835,50.735233 30,51 C29.9949422,51.0665707 29.9949422,51.1334293 30,51.2 C30.0107984,51.2626849 30.0310278,51.323373 30.06,51.38 C30.0818883,51.4437243 30.1121536,51.5042548 30.15,51.56 C30.1869597,51.6123582 30.2270319,51.6624485 30.27,51.71 C30.3651037,51.8010406 30.4772487,51.8724056 30.6,51.92 C30.8420399,52.0269768 31.1179601,52.0269768 31.36,51.92 C31.6690463,51.7932621 31.8944082,51.520589 31.960688,51.1932068 C32.0269678,50.8658246 31.925413,50.5269659 31.69,50.29 C31.5987206,50.2036745 31.4940236,50.1327508 31.38,50.08"
```
如果给你这样一个字符串，你会如何将他转换为一系列的点的坐标？第一种情况，你观察之后，可能第一会想到我先将这个字符串用空格进行分割，然后再用逗号分割，就能获取到每个点的坐标。但是第二种情况呢，他每一个字母还有不同的，可能M开头的我需要后面1个坐标，但是C开头的我需要3个坐标，所以如果还是使用字符串分割的话就会比较麻烦，并且更容易出错，也不利于维护。
所以在swift里，还可以有一种更“优雅”的方式实现，就是使用自定义迭代器```IteratorProtocol```和序列化```Sequence```。具体的实现细节不讲，最后的效果就是可以用一个字符串初始化一个迭代器，就可以用for循环去读取了。
第一个迭代器在```CoordinateLexer```类，第二个比较复杂的是```path```的迭代器，在```PathDLexer```，他们都被定义成结构体。
### Parser
在SVG的文件解析过程中，虽然目前只支持一种XML的解析，但是还是将解析行为定义为了一个协议```SVGParser```。另外一个```SVGParserSupportedElements```结构体，初始化了目前支持解析的元素，他这里比较有意思的点是，所有支持的元素初始化在一个字段对象里，但是值是一个返回元素的函数，只有在我们解析到这个元素，获取到这个函数，执行之后才会创建对应的元素。
```
public typealias ElementGenerator = () -> SVGElement
public static var allSupportedElements: SVGParserSupportedElements {
        let supportedElements: [String : ElementGenerator] = [
            SVGCircle.elementName: {
                let returnElement = SVGCircle()
                returnElement.supportedAttributes = [
                    "cx": unown(returnElement, SVGCircle.xCenter),
                    "cy": unown(returnElement, SVGCircle.yCenter),
                    "r": unown(returnElement, SVGCircle.radius),
                ]
                returnElement.supportedAttributes.add(returnElement.identityAttributes)
                returnElement.supportedAttributes.add(returnElement.fillAttributes)
                returnElement.supportedAttributes.add(returnElement.strokeAttributes)
                returnElement.supportedAttributes.add(returnElement.styleAttributes)
                returnElement.supportedAttributes.add(returnElement.transformAttributes)
                return returnElement
            }
        ]
        return SVGParserSupportedElements(tags: supportedElements)
    }
```
最后的xml解析的实现类```NSXMLSVGParser```，就比较简单，用系统的xml解析库解析xml文件，自定义了一个栈的数据结构来存储解析过程中的元素。
### Cache
使用```NSCache```做了一个自定义的内存缓存，存储calayer对象

### 总结
1. 大量的使用了面向协议编程的思想，通过协议来描述元素的行为，类与类之间的耦合很低，并且在后期需要增加新的元素，也更容易进行扩展。
2. 多处使用函数作为属性值进行存储，在需要的地方执行，也降低了耦合和硬编码
3. 通过自定义迭代器，可以更加优雅的实现某些功能。比如之前在oc中实现对一个数组进行遍历，比如：10个元素，初始位置为6，则迭代顺序为6574839210。
4. 能使用struct尽量用struct
5. 通过协议的继承，可以尽量让协议更加抽象和独立，防止一个协议定义过多的方法和属性
6. 其他swift的语法糖的使用