

# `SpringCloudStream-kafka`









## 事件驱动和消息驱动

消息驱动和事件驱动很类似，都是先有一个事件，然后产生一个相应的消息，再把消息放入**消息队列**，由需要的项目获取。他们的区别是消息是谁产生的

**消息驱动**：鼠标管自己点击不需要和系统有过多的交互，消息由系统（第三方）循环检测，来捕获并放入消息队列。消息对于点击事件来说是被动产生的，高内聚。

**事件驱动**：鼠标点击产生点击事件后要向系统发送消息 “我点击了” 的消息，消息是主动产生的。再发送到消息队列中。事件往往会将事件源包装起来。



























































