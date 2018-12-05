# Linux性能优化

## 准备工作
- 准备一台linux服务器，本文以ubuntu服务器为例

## Linux系统常用的性能分析指标

### 系统平均负载
通过系统平均负载的值，我们可以直观的感受到系统的负荷是高还是低。参考**阮一峰**的[理解Linux系统负荷](http://www.ruanyifeng.com/blog/2011/07/linux_load_average_explained.html)一文，文中阮一峰的举例非常形象的表达了什么是系统负载

假设最简单的情况，你的电脑只有一个CPU，所有的运算都必须由这个CPU来完成。
不妨把这个CPU想象成一座大桥，桥上只有一根车道，所有车辆都必须从这根车道上通过。（很显然，这座桥只能单向通行。）

系统负荷为0，意味着大桥上一辆车也没有。
![负荷0](http://www.ruanyifeng.com/blogimg/asset/201107/bg2011073004.png)


