# 代码规范

## Page
* 点击事件统一放在controller，最好不要在page里面写回调逻辑。


## Model

* 属性声明不使用'?'号，必须声明late。
* FormJson给默认值：number => 0；string => ''。

## 通用

* function的返回类型要明确声明，不要省略。