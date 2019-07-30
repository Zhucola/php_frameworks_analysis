** PageCache还是需要走到PHP里面取做cache，应该使用nginx的proxy-cache来做PageCache **
** 不管是yii的PageCache还是nginx的proxy-cache，都是缓存了所有的数据，如果涉及到动态的信息(用户信息、订单详情等)，需要做动态处理，推荐ajax ** 
## 目录
* [附加事件处理器](#附加事件处理器)
* [类级别事件处理器](#类级别事件处理器)

# 执行流程
- asd
- asd
