## MyBatis

### 配置

```xml
    <!--配置数据源-->
	<bean id="dataSource" class="com.alibaba.druid.pool.DruidDataSource" parent="dataSourceBase" init-method="init" destroy-method="close">
        <property name="url" value="${gamehall.url}" />
        <property name="username" value="${gamehall.username}" />
        <property name="password" value="${gamehall.password}" />
    </bean>
	<!--配置sqlSessionFactory-->
    <bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
        <property name="dataSource" ref="dataSource" />
    </bean>
	<!--配置MapperScanner-->
    <bean id="mapperScannerConfigurer" class="org.mybatis.spring.mapper.MapperScannerConfigurer">
        <property name="basePackage" value="com.mirror.game.dao.gamehall" />
        <property name="sqlSessionFactoryBeanName" value="sqlSessionFactory" />
    </bean>
	<!--配置事务管理器-->
    <bean id="transactionManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
        <property name="dataSource" ref="gamehallDataSource"/>
    </bean>
```

