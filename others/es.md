1. 写数据底层原理？
   数据写入内存buffer，然后每隔1s refresh到os cache，到了os cache数据就可以被搜索到了（近实时搜索，数据写入1s后才能被搜索），数据写入buffer的同时会顺序写translog文件，translog文件每隔5s fsync到磁盘，translog大到一定程度或者默认每隔30min，会出发commit操作，将os cache中的数据都fsync到磁盘文件，然后清理translog文件。
2. 删除、更新数据底层原理？
   如果是删除操作，commit的时候会生成一个.del文件，里面讲某个document标识为deleted状态，那么搜索的时候根据.del文件就知道这个doc是否被删除了。如果是更新操作就将旧的文档标识为deleted状态，然后写入一条数据。
   buffer每refresh一次就产生一个segment file，默认情况下每秒产生一个segment file，这样下来文件会越来越多，此时会定期执行merge，每次merge会将多个segment file合并成一个，同时将标记为deleted的doc给物理删除掉，然后将新的segment file写入磁盘。