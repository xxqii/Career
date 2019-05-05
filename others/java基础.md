1. HashMap的put(key, value)方法具体实现？

   * 计算key的hashCode值(hashCode高16位^低16位)；
   * 判断tab是否为null，如果为null，则调用resize方法初始化tab数组；
   * 计算key对应的下标i（n-1 & hash）；
   * 如果tab[i]为null，则直接根据key-value生成newNode，插入tab[i]位置；
   * 如果tab[i]和key相等(hashCode和值都相等)，则将tab[i]节点保存到变量e中；
   * 如果tab[i]是一个TreeNode节点，则调用putTreeVal方法，将key-value插入红黑树中（key存在返回旧的TreeNode保存到变量e，否则直接插入key-value）；
   * 如果tab[i]是链表结构，则遍历链表，直到找到key对应的节点或者链表末尾（如果找到key，将节点存储到变量e中，否则插入新节点）；
   * 如果变量e（如果key存在，保存key对应的节点信息；如果key不存在，值为null）的值不为null，根据参数onlyIfAbsent来更新对应key的值(true：不更新；false：更新)
   * 调用afterNodeAccess方法；
   * 判断节点数量是否到达阈值，如果达到则调用reSize扩容；
   * 调用afterNodeInsertion(evict)方法

   注意：在HashMap中定义了三个方法：void afterNodeAccess(Node<K,V> p) { }；void afterNodeInsertion(boolean evict) { }；void afterNodeRemoval(Node<K,V> p) { }，这三个方法主要在LinkedHashMap里面控制节点的访问顺序和删除策略。在HashMap里面实现为null。