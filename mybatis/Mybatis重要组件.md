# Mybatis重要组件介绍

1. Configuration: Mybatis所有的配置信息都会保存在Configuration。

2. SqlSession: Mybatis中最主要的顶层API，用于用户和数据库交互时的会话，提供了对数据库的增删改查操作。

3. Executor: Mybatis中的执行器，负责Sql语句的生成 ；查询和维护缓存。

4. StatementHandler: 封装了JDBC Statement操作，负责对JDBC Statement的操作。

5. ParameterHandler: 负责对用户传递的参数转换为JDBC 所对应的数据类型。

6. ResultSetHandler: 负责将JDBC返回的ResultSet结果对象转为List类型集合

7. TypeHandler: 负责Java数据类型和JDBC数据类型之间的映射和转换。

8. MappingedStatement: 维护一条select|insert|deleted|update节点的信息。（用于存储解析后mapper.xml中的sql操作标签信息）

9. SqlSource: 负责根据用户传递的parameterObject，动态的生产Sql语句，将信息封装到BoundSql对象中。

10. BoundSql: 动态生成的Sql语句及相关参数信息。
