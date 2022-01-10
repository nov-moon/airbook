# 代码规范

## Page
* 点击事件统一放在controller，最好不要在page里面写回调逻辑。
* 一般情况下，点击事件的注册用Widget的扩展方法onTab。
* 在controller的dispose中回收资源，如TextController回收、监听的取消等。
* 刷新统一用update。
* 布局时，使用Container，以Container的属性代替Align、Padding、ColoredBox、ClipPath等属性，减少Widget嵌套层级。
* 定义Widget尽量提供const构造方法。
* 使用Widget尽量使用const关键字。


## Model
* 属性声明使用'?'号，尽量不声明late。
* Model里面只写已用，不用的注释掉。

## 通用
* 属性、方法命名尽量私有化。
* 属性、方法、类命名采用大驼峰命名法。
* function的返回类型要明确声明，不要省略。