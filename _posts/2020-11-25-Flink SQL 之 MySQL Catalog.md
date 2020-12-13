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

在实现 `MySQL catalog` 前先看看 `PostgresCatalog`，两者都是 `AbstractJdbcCatalog` 实现类。

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
      // 得到表结构
			ResultSetMetaData rsmd = ps.getMetaData();

			String[] names = new String[rsmd.getColumnCount()];
			DataType[] types = new DataType[rsmd.getColumnCount()];

			for (int i = 1; i <= rsmd.getColumnCount(); i++) {
				names[i - 1] = rsmd.getColumnName(i);
        // fromJDBCType：flink 类型和 jdbc 类型转换
				types[i - 1] = fromJDBCType(rsmd, i);
				if (rsmd.isNullable(i) == ResultSetMetaData.columnNoNulls) {
					types[i - 1] = types[i - 1].notNull();
				}
			}
      // 表结构生成 Flink 内部的 TableSchema
			TableSchema.Builder tableBuilder = new TableSchema.Builder()
				.fields(names, types);
			primaryKey.ifPresent(pk ->
				tableBuilder.primaryKey(pk.getName(), pk.getColumns().toArray(new String[0]))
			);
			TableSchema tableSchema = tableBuilder.build();

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
		} 
	}
}
```

简单来说，`PostgresCatalog` 就是转接口：Postgres 表转换成 Flink 表(相当于自动生成 DDL)。

### MySQL catalog

#### Create catalog

``` java 

```











## 参考资料

[【Flink 1.11】Flink JDBC Connector：Flink 与数据库集成最佳实践]()