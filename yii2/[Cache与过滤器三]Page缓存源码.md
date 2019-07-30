**PageCache还是需要走到PHP里面取做cache，应该使用nginx的proxy-cache来做PageCache**
**不管是yii的PageCache还是nginx的proxy-cache，都是缓存了所有的数据，如果涉及到动态的信息(用户信息、订单详情等)，需要做动态处理，推荐ajax** 

## 目录
* [执行流程](#执行流程)
* [源码分析](#源码分析)

# 执行流程
- 项目初始化，路由解析
- 控制器初始化，走到base\Controller里面的runAction方法，执行beforeAction事件
- 加载过滤器行为，将PageCache的beforeAction事件注入，并且执行
- 判断requestedRoute(就是真实uri)拼的键是否在缓存中存在，存在则直接获取缓存数据，并且将beforeAction返回false，就是直接走到response组件的send了，不会再继续加载控制器方法了
- 如果缓存不存在，将页面响应信息存缓存

# 源码分析

