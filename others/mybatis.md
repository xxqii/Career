1. mybatis简介？
   MyBatis是一个持久层ORM框架，对jdbc的操作进行封装。使用MyBatis开发者只需要关注SQL本身，不再需要处理驱动注册、Connection创建、Statement创建、设置参数、结果集检索等复杂流程。

2. #{}和${}的区别？
   #{}是预编译处理，MyBatis在遇到#{}的时候会替换为？，然后底层调用PrepareStatement的set方法来设置参数；${}是字符串替换。
   使用#{}可以预防SQL注入。
3. MyBatis每个xml配置文件都由一个Dao接口与之对应，Dao接口里的方法可以重载吗？
   Dao接口即Mapper接口，里面的方法不能重载，当接口调用时，接口权限定名+方法名拼接字符串作为key值，可唯一定位一个MappedStatement。
   Dao接口的工作原理时JDK动态代理，MyBatis运行时会使用JDK动态代理为Dao接口生成proxy对象，代理对象会拦截接口方法，转而执行MappedStatement所代表的sql，然后将sql执行结果返回。

4. MyBatis动态sql？
   Mybatis提供了9种动态sql标签：trim|where|set|foreach|if|choose|when|otherwise|bind。
   其执行原理为，从sql参数对象中计算表达式的值，根据表达式的值动态拼接sql，以此来完成动态sql的功能。
5. 