---
layout:     post
title:      Flink SQL 之 MySQL Catalog
subtitle:   
date:       2020-11-25
author:     danner
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - Flink
    - SQL
    - catalog
---

Flink 1.11

1.11 版本为止，真正能实现的 Catalog 只有 `HiveCatalog` 和 `PostgresCatalog`。JDBC catalog 提供了接⼝连接到各种关系数据库，使得 Flink 能够**⾃动检索表**，**不⽤⽤户⼿动输⼊和修改**。 `MySQL` 没有实现，本文带大家实现下。

在实现 `MySQL catalog` 前先看看 `PostgresCatalog`，是 `AbstractJdbcCatalog` 实现类。

![](https://vendanner.github.io/img/Flink/catalog.jpg)

### PostgresCatalog

```java
// org.apache.flink.connector.jdbc.catalog.PostgresCatalog
public class PostgresCatalog extends AbstractJdbcCatalog {
  // 只读 catalog(hiveCatalog 读写)，只实现了几个函数
  // databaseExists，listDatabases，getDatabase，listTables，getTable，tableExists
  public List<String> listTables(String databaseName) throws DatabaseNotExistException, CatalogException {
    ...
    // 连接数据库，通过元数据获取所有表
    try (Connection conn = DriverManager.getConnection(baseUrl + databaseName, username, pwd)) {
      PreparedStatement ps = conn.prepareStatement("SELECT schema_name FROM information_schema.schemata;");
      ResultSet rs = ps.executeQuery();
      List<String> schemas = new ArrayList<>();

      while (rs.next()) {
        String pgSchema = rs.getString(1);
        if (!builtinSchemas.contains(pgSchema)) {
          schemas.add(pgSchema);
        }
      }

      List<String> tables = new ArrayList<>();
      for (String schema : schemas) {
        PreparedStatement stmt = conn.prepareStatement(
          "SELECT * \n" +
            "FROM information_schema.tables \n" +
            "WHERE table_type = 'BASE TABLE' \n" +
            "    AND table_schema = ? \n" +
            "ORDER BY table_type, table_name;");

        stmt.setString(1, schema);
        ResultSet rstables = stmt.executeQuery();

        while (rstables.next()) {
          // position 1 is database name, position 2 is schema name, position 3 is table name
          tables.add(schema + "." + rstables.getString(3));
        }
      }
      return tables;
    } 
  }
  // 获取表结构，自动创建 CatalogBaseTable
  public CatalogBaseTable getTable(ObjectPath tablePath) throws TableNotExistException, CatalogException {
    ...
		PostgresTablePath pgPath = PostgresTablePath.fromFlinkTableName(tablePath.getObjectName());

		String dbUrl = baseUrl + tablePath.getDatabaseName();
		try (Connection conn = DriverManager.getConnection(dbUrl, username, pwd)) {
			DatabaseMetaData metaData = conn.getMetaData();
			Optional<UniqueConstraint> primaryKey = getPrimaryKey(
				metaData,
				pgPath.getPgSchemaName(),
				pgPath.getPgTableName());

			PreparedStatement ps = conn.prepareStatement(
				String.format("SELECT * FROM %s;", pgPath.getFullPath()));
      // 第一步：得到表结构
			ResultSetMetaData rsmd = ps.getMetaData();

			String[] names = new String[rsmd.getColumnCount()];
			DataType[] types = new DataType[rsmd.getColumnCount()];

			for (int i = 1; i <= rsmd.getColumnCount(); i++) {
				names[i - 1] = rsmd.getColumnName(i);
        // 第二步：fromJDBCType：flink 类型和 jdbc 类型转换
				types[i - 1] = fromJDBCType(rsmd, i);
				if (rsmd.isNullable(i) == ResultSetMetaData.columnNoNulls) {
					types[i - 1] = types[i - 1].notNull();
				}
			}
      // 第三步：表结构生成 Flink 内部的 TableSchema
			TableSchema.Builder tableBuilder = new TableSchema.Builder()
				.fields(names, types);
			primaryKey.ifPresent(pk ->
				tableBuilder.primaryKey(pk.getName(), pk.getColumns().toArray(new String[0]))
			);
			TableSchema tableSchema = tableBuilder.build();
      // 第四步：添加连接属性
			Map<String, String> props = new HashMap<>();
			props.put(CONNECTOR.key(), IDENTIFIER);
			props.put(URL.key(), dbUrl);
			props.put(TABLE_NAME.key(), pgPath.getFullPath());
			props.put(USERNAME.key(), username);
			props.put(PASSWORD.key(), pwd);
      // schema + 连接属性 创建表
			return new CatalogTableImpl(
				tableSchema,
				props,
				""
			);
```

简单来说，`PostgresCatalog` 就是个转接口：**Postgres 表转换成 Flink 表(相当于自动生成 DDL)**。

- databaseExists：数据库是否存在
- listDatabases：列出所有数据库
- getDatabase：获取数据库
- listTables：列出所有表
- getTable：获取表
- tableExists：表是否存在



### MySQL catalog

#### Register catalog

``` java 
// JdbcCatalog catalog = new JdbcCatalog(name, defaultDatabase, username, password, baseUrl);
// tableEnv.registerCatalog("mypg", catalog);

// org.apache.flink.connector.jdbc.catalog.JdbcCatalog
public JdbcCatalog(
      String catalogName,
      String defaultDatabase,
      String username,
      String pwd,
      String baseUrl) {
  super(catalogName, defaultDatabase, username, pwd, baseUrl);

  internal =
          JdbcCatalogUtils.createCatalog(
                  catalogName, defaultDatabase, username, pwd, baseUrl);
}

// org.apache.flink.connector.jdbc.catalog.JdbcCatalogUtils
/** Create catalog instance from given information. */
public static AbstractJdbcCatalog createCatalog(
      String catalogName,
      String defaultDatabase,
      String username,
      String pwd,
      String baseUrl) {
  // 通过 url 获取方言，MySQL 方言 flink 已实现
  JdbcDialect dialect = JdbcDialects.get(baseUrl).get();
  
  // 通过方言创建 Catalog，jdbc 方式只实现 Postgres
  if (dialect instanceof PostgresDialect) {
      return new PostgresCatalog(catalogName, defaultDatabase, username, pwd, baseUrl);
  } else {
      throw new UnsupportedOperationException(
              String.format("Catalog for '%s' is not supported yet.", dialect));
  }
}
```

以上代码可知，添加一个 Jdbc Catalog ，需要在 `createCatalog` 函数中添加判断

```java
else if (dialect instanceof MySQLDialect) {
      return new MySQLCatalog(catalogName, defaultDatabase, username, pwd, baseUrl);
  }
```

至此，创建 MySQL Catalog 时就会帮我映射到 MySQLDialect (没添加之前是会报错)。

#### Create catalog

实现 MySQL Catalog 代码，照抄 `PostgresCatalog`，不同之处加注释。

```java
public class MySQLCatalog extends AbstractJdbcCatalog {

  private static final Logger LOG = LoggerFactory.getLogger(MySQLCatalog.class);

  public MySQLCatalog(String catalogName,
                      String defaultDatabase,
                      String username,
                      String pwd,
                      String baseUrl) {
      super(catalogName, defaultDatabase, username, pwd, baseUrl);
  }

  // ------ MySQL default objects that shouldn't be exposed to users ------

  private static final Set<String> builtinDatabases =
          new HashSet<String>() {
              {
                  add("information_schema");
                  add("mysql");
                  add("performance_schema");
                  add("sys");
              }
          };

  @Override
  public List<String> listDatabases() throws CatalogException {
      List<String> mysqlDatabases = new ArrayList<>();

      try (Connection conn = DriverManager.getConnection(defaultUrl, username, pwd)) {

          PreparedStatement ps = conn.prepareStatement("SELECT SCHEMA_NAME FROM INFORMATION_SCHEMA.SCHEMATA;");

          ResultSet rs = ps.executeQuery();

          while (rs.next()) {
              String dbName = rs.getString(1);
              if (!builtinDatabases.contains(dbName)) {
                  mysqlDatabases.add(rs.getString(1));
              }
          }

          return mysqlDatabases;
      } catch (Exception e) {
          throw new CatalogException(
                  String.format("Failed listing database in catalog %s", getName()), e);
      }
  }

  @Override
  public CatalogDatabase getDatabase(String databaseName)
          throws DatabaseNotExistException, CatalogException {
      if (listDatabases().contains(databaseName)) {
          return new CatalogDatabaseImpl(Collections.emptyMap(), null);
      } else {
          throw new DatabaseNotExistException(getName(), databaseName);
      }
  }

  @Override
  public List<String> listTables(String databaseName)
          throws DatabaseNotExistException, CatalogException {
      if (!databaseExists(databaseName)) {
          throw new DatabaseNotExistException(getName(), databaseName);
      }

      // 获取 databas 下所有表
      try (Connection conn = DriverManager.getConnection(baseUrl + databaseName, username, pwd)) {
          PreparedStatement stmt =
                  conn.prepareStatement("SELECT TABLE_NAME FROM information_schema.`TABLES` WHERE TABLE_SCHEMA = ?;");

          stmt.setString(1, databaseName);

          ResultSet rs = stmt.executeQuery();

          List<String> tables = new ArrayList<>();

          while (rs.next()) {
              tables.add(rs.getString(1));
          }

          return tables;
      } catch (Exception e) {
          throw new CatalogException(
                  String.format("Failed listing database in catalog %s", getName()), e);
      }
  }

  @Override
  public CatalogBaseTable getTable(ObjectPath tablePath)
          throws TableNotExistException, CatalogException {
      if (!tableExists(tablePath)) {
          throw new TableNotExistException(getName(), tablePath);
      }

      String dbUrl = baseUrl + tablePath.getDatabaseName();
      try (Connection conn = DriverManager.getConnection(dbUrl, username, pwd)) {
          DatabaseMetaData metaData = conn.getMetaData();

          // MySQL 没有 schema 概念，直接传 null
          Optional<UniqueConstraint> primaryKey =
                  getPrimaryKey(metaData,null,tablePath.getObjectName());

          PreparedStatement ps =
                  conn.prepareStatement(String.format("SELECT * FROM %s;", tablePath.getObjectName()));

          ResultSetMetaData rsmd = ps.getMetaData();

          // 列名称
          String[] names = new String[rsmd.getColumnCount()];
          // 列类型(flink中)
          DataType[] types = new DataType[rsmd.getColumnCount()];

          for (int i = 1; i <= rsmd.getColumnCount(); i++) {
              names[i - 1] = rsmd.getColumnName(i);
              types[i - 1] = fromJDBCType(rsmd, i);
              if (rsmd.isNullable(i) == ResultSetMetaData.columnNoNulls) {
                  types[i - 1] = types[i - 1].notNull();
              }
          }

          TableSchema.Builder tableBuilder = new TableSchema.Builder().fields(names, types);
          primaryKey.ifPresent(
                  pk ->
                          tableBuilder.primaryKey(
                                  pk.getName(), pk.getColumns().toArray(new String[0])));

          TableSchema tableSchema = tableBuilder.build();

          Map<String, String> props = new HashMap<>();
          props.put(CONNECTOR.key(), IDENTIFIER);
          props.put(URL.key(), dbUrl);
          props.put(TABLE_NAME.key(), tablePath.getObjectName());
          props.put(USERNAME.key(), username);
          props.put(PASSWORD.key(), pwd);
          // 返回 CatalogTableImpl 与 create table sql 所做的事情是一致的
          return new CatalogTableImpl(tableSchema, props, "");
      }catch (Exception e) {
          throw new CatalogException(
                  String.format("Failed getting table %s", tablePath.getFullName()), e);
      }
  }

  public static final String MYSQL_TINYINT = "TINYINT";
  public static final String MYSQL_SMALLINT = "SMALLINT";
  public static final String MYSQL_TINYINT_UNSIGNED = "TINYINT UNSIGNED";
  public static final String MYSQL_INT = "INT";
  public static final String MYSQL_MEDIUMINT = "MEDIUMINT";
  public static final String MYSQL_SMALLINT_UNSIGNED = "SMALLINT UNSIGNED";
  public static final String MYSQL_BIGINT = "BIGINT";
  public static final String MYSQL_INT_UNSIGNED = "INT UNSIGNED";
  public static final String MYSQL_BIGINT_UNSIGNED = "BIGINT UNSIGNED";
  public static final String MYSQL_FLOAT = "FLOAT";
  public static final String MYSQL_DOUBLE = "DOUBLE";
  public static final String MYSQL_NUMERIC = "NUMERIC";
  public static final String MYSQL_DECIMAL = "DECIMAL";
  public static final String MYSQL_BOOLEAN = "BOOLEAN";
  public static final String MYSQL_DATE = "DATE";
  public static final String MYSQL_TIME = "TIME";
  public static final String MYSQL_DATETIME = "DATETIME";
  public static final String MYSQL_CHAR = "CHAR";
  public static final String MYSQL_VARCHAR = "VARCHAR";
  // mysql text 类型实质是 varchar(13618)
  public static final String MYSQL_TEXT = "TEXT";


  // Converts MySQL type to Flink {@link DataType}
  private DataType fromJDBCType(ResultSetMetaData metadata, int colIndex) throws SQLException {
      String mysqlType = metadata.getColumnTypeName(colIndex);

      int precision = metadata.getPrecision(colIndex);
      int scale = metadata.getScale(colIndex);

      switch (mysqlType) {
          case MYSQL_TINYINT:
              if (1 == precision) {
                  return DataTypes.BOOLEAN();
              }
              return DataTypes.TINYINT();
          case MYSQL_SMALLINT:
              return DataTypes.SMALLINT();
          case MYSQL_TINYINT_UNSIGNED:
              return DataTypes.SMALLINT();
          case MYSQL_INT:
              return DataTypes.INT();
          case MYSQL_MEDIUMINT:
              return DataTypes.INT();
          case MYSQL_SMALLINT_UNSIGNED:
              return DataTypes.INT();
          case MYSQL_BIGINT:
              return DataTypes.BIGINT();
          case MYSQL_INT_UNSIGNED:
              return DataTypes.BIGINT();
          case MYSQL_BIGINT_UNSIGNED:
              return DataTypes.DECIMAL(20,0);
          case MYSQL_FLOAT:
              return DataTypes.FLOAT();
          case MYSQL_DOUBLE:
              return DataTypes.DOUBLE();
          case MYSQL_NUMERIC:
              return DataTypes.DECIMAL(precision,scale);
          case MYSQL_DECIMAL:
              return DataTypes.DECIMAL(precision,scale);
          case MYSQL_BOOLEAN:
              return DataTypes.BOOLEAN();
          case MYSQL_DATE:
              return DataTypes.DATE();
          case MYSQL_TIME:
              return DataTypes.TIME(scale);
          case MYSQL_DATETIME:
              return DataTypes.TIMESTAMP(scale);
          case MYSQL_CHAR:
              return DataTypes.CHAR(precision);
          case MYSQL_VARCHAR:
              return DataTypes.CHAR(precision);
          case MYSQL_TEXT:
              return DataTypes.STRING();
          default:
              throw new UnsupportedOperationException(
                      String.format("Doesn't support mysql type '%s' yet", mysqlType));
      }
  }

  @Override
  public boolean tableExists(ObjectPath tablePath) throws CatalogException {
      List<String> tables = null;
      try {
          tables = listTables(tablePath.getDatabaseName());
      } catch (DatabaseNotExistException e) {
          return false;
      }

      return tables.contains(tablePath.getObjectName());
  }
}
```







## 参考资料

[FLIP-93: JDBC catalog and Postgres catalog](https://cwiki.apache.org/confluence/display/FLINK/FLIP-93%3A+JDBC+catalog+and+Postgres+catalog)

[【Flink 1.11】Flink JDBC Connector：Flink 与数据库集成最佳实践](https://developer.aliyun.com/article/776069)