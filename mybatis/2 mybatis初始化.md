# mybatis初始化
1. SqlSessionFactoryBuilder接收一个input数据流（xml主配置文件流），将此流委托给XMLConfigBuilder进行解析获取Configuration对象
    1. XMLConfigBuilder拿到流对象后，会创建一个XPathParser对象，由它来进行具体的xml解析工作
    2. 调用XMLConfigBuilder.parse()方法进行解析
    3. 首先读取根节点（configuration节点）
    4. 以configuration节点作为根节点读取properties节点，其内部的propertie节点会先被加载，然后加载properties的url或resource指定位置的配置文件（相同属性覆盖之前的），在之后使用创建SqlSessionFactoryBuilder时指定的properties对象中的配置进行覆盖
2. 以Configuration做为参数创建DefaultSqlSessionFactory对象供用户使用