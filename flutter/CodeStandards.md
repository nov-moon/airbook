# 代码规范

## Page
* 点击事件统一放在controller，最好不要在page里面写回调逻辑。
* 一般情况下，点击事件的注册用Widget的扩展方法onTab。
* 在controller的dispose中回收资源，如TextController回收、监听的取消等。
* 刷新统一用update。


## Model

* 属性声明不使用'?'号，必须声明late。
* FormJson给默认值：number => 0；string => ''。
* Model里面只写已用，不用的注释掉。

## 通用

* function的返回类型要明确声明，不要省略。