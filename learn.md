# [关系代数](https://blog.csdn.net/QuinnNorris/article/details/70739094)
关系代数是关系型数据库操作的理论基础，是一种过程化查询语言，关系代数支持并、差、笛卡尔积、投影和选择等基本运算，这些运算以一个或两个关系为输入，产生一个新的关系作为结果。
| 名称      |    英文 | 符号  | 说明  |
|:-------- |:--------:|:--------:| :--: |
| 选择     | select |  σ   |类似于 SQL 中的 where
| 投影     | project |  Π  |类似于 SQL 中的 select
| 并       |    union | ∪  |类似于 SQL 中的 union
| 集合差   |    set-difference | -  |SQL中没有对应的操作符
| 笛卡儿积 |    Cartesian-product | ×  |不带 on 条件的 inner join
| 更名     |    rename | ρ  |类似于 SQL 中的 as
| 集合交   |    intersection | ∩  |SQL中没有对应的操作符
| 自然连接 |    natural join | ⋈  |类似于 SQL 中的 inner join
| 赋值     |    assignment | ←  |  
# calcite处理流程
# Step0: 生成SQL解析器
Calcite 使用 JavaCC 做 SQL 解析，JavaCC 根据 Calcite 中定义的 [Parser.jj](https://github.com/apache/calcite/blob/master/core/src/main/codegen/templates/Parser.jj) 文件，生成一系列的 java 代码。
## [fmpp]()
FMPP是一种使用[FreeMarker]()模板的通用文本文件预处理工具。
FMPP最常用的应用是生成静态网站，源代码，配置文件等。
它可以将数据从数据库，CSV，XML和JSON等源插入到生成的文件中。
Calcite基于FreeMarker生成代码,之后编译运行,fmpp例子请看系统代码 [fmpp基本例子](https://gitee.com/leinenglian/apache-calcite-tutorial/blob/master/calcite-tutorial-2-parser/parser-1-fmpp-tutorial/README.md)
## [javacc]()
Javacc是用来生产词法分析器(lexical analysers)和语法解析器(parser)。
编译器和解释器结合词法分析器和语法解析器来编译源程序文件。
javacc使用递归下降语法解析，LL(k)。其中，第一个L表示从左到右扫描输入； 第二个L表示每次都进行最左推导(在推导语法树的过程中每次都替换句型中最左的非终结符为终结符。类似还有最右推导)；k表示的是每次向前探索(lookahead)k个终结符。[参考](http://www.yunshouce.com/g62/diyccompiler/5.html)
[JavaCC基本例子](https://gitee.com/leinenglian/apache-calcite-tutorial/blob/master/calcite-tutorial-2-parser/parser-2-javacc-tutorial/README.md)
## [生产解析器](https://gitee.com/leinenglian/apache-calcite-tutorial/blob/master/calcite-tutorial-2-parser/parser-3-calcite-tutorial/README.md)
# Step1: SQL 解析阶段（SQL–>SqlNode）
使用 Calcite 进行 Sql 解析的代码如下：
```
  String sql = "select id, cast(score as int), 'hello' from T where id < ?";
  SqlParser parser = SqlParser.create(sql, SqlParser.Config.DEFAULT);
  SqlNode sqlNode = parser.parseQuery();
```
SQL解析器把 SQL 转换成 AST 的数据结构（这里是 SqlNode 类型）。
以上SQL中， 1. id, score, T 等为SqlIdentifier 2. cast()为SqlCall 3. int 为SqlDataTypeSpec 4. 'hello' 为SqlLiteral 5. '?' 为SqlDynamicParam
# Step2: SqlNode 验证（SqlNode–>SqlNode）
经过上面的第一步，会生成一个 SqlNode 对象，它是一个**未经验证**的抽象语法树，下面就进入了一个语法检查阶段，语法检查前需要知道元数据信息，这个检查会包括表名、字段名、函数名、数据类型的检查。进行语法检查的实现如下：
```
  SqlTypeFactoryImpl factory = new SqlTypeFactoryImpl(RelDataTypeSystem.DEFAULT);
  CalciteCatalogReader calciteCatalogReader = new CalciteCatalogReader(
          CalciteSchema.from(rootSchema),
          CalciteSchema.from(rootSchema).path(null),
          factory,
          new CalciteConnectionConfigImpl(new Properties()));
  
   //note: 校验（包括对表名，字段名，函数名，字段类型的校验。）
  SqlValidator validator = SqlValidatorUtil.newValidator(SqlStdOperatorTable.instance(), calciteCatalogReader, factory,SqlConformanceEnum.DEFAULT);
  SqlNode validateSqlNode = validator.validate(sqlNode);
```
Calcite 本身是不管理和存储元数据的，在检查之前，需要先把元信息注册到 Calcite 中，一般的操作方法是实现 SchemaFactory，由它去创建相应的 Schema，在 Schema 中可以注册相应的元数据信息。
```
 SchemaPlus rootSchema = Frameworks.createRootSchema(true);
 rootSchema.add("USERS", new AbstractTable() { //note: add a table
        @Override
        public RelDataType getRowType(final RelDataTypeFactory typeFactory) {
            RelDataTypeFactory.Builder builder = typeFactory.builder();
            builder.add("ID", new BasicSqlType(new RelDataTypeSystemImpl() {}, SqlTypeName.INTEGER));
            builder.add("NAME", new BasicSqlType(new RelDataTypeSystemImpl() {}, SqlTypeName.CHAR));
            builder.add("AGE", new BasicSqlType(new RelDataTypeSystemImpl() {}, SqlTypeName.INTEGER));
            return builder.build();
        }
});
```
# Step3: 语义分析（SqlNode–>RelNode/RexNode）  
经过第二步之后，这里的 SqlNode 就是经过语法校验的 SqlNode 树，接下来这一步就是将 SqlNode 转换成 RelNode/RexNode，也就是生成相应的逻辑计划（Logical Plan），示例的代码实现如下：
```
  // create the rexBuilder
  final RexBuilder rexBuilder =  new RexBuilder(new SqlTypeFactoryImpl(RelDataTypeSystem.DEFAULT));
  
  // init the planner
  // 这里也可以注册 VolcanoPlanner，这一步 planner 并没有使用
  HepProgramBuilder builder = new HepProgramBuilder();
  RelOptPlanner planner = new HepPlanner(builder.build());
  
  //note: init cluster: An environment for related relational expressions during the optimization of a query.
  final RelOptCluster cluster = RelOptCluster.create(planner, rexBuilder);
  
  //note: init SqlToRelConverter
  final SqlToRelConverter.Config config = SqlToRelConverter.configBuilder()
          .withConfig(SqlToRelConverter.Config.DEFAULT)
          .withTrimUnusedFields(false)
          .withConvertTableAccess(false)
          .build(); //note: config
  // 创建 SqlToRelConverter 实例，cluster、calciteCatalogReader、validator 都传进去了，SqlToRelConverter 会缓存这些对象
  final SqlToRelConverter sqlToRelConverter = new SqlToRelConverter(new DogView(), validator, calciteCatalogReader, cluster, StandardConvertletTable.INSTANCE, config);
  // convert to RelNode
  RelRoot root = sqlToRelConverter.convertQuery(validateSqlNode, false, true);
  root = root.withRel(sqlToRelConverter.flattenTypes(root.rel, true));
  final RelBuilder relBuilder = config.getRelBuilderFactory().create(cluster, null);
  root = root.withRel(RelDecorrelator.decorrelateQuery(root.rel, relBuilder));
  RelNode relNode = root.rel;
```
此过程中SqlToRelConverter会将SqlNode转化成RelNode, 这其中主要涉及了以下几个数据结构

1. RelNode 关系表达式， 主要有TableScan, Project, Sort, Join等。如果SQL为查询的话，所有关系达式都可以在SqlSelect中找到, 如 where和having 对应的Filter, selectList对应Project, orderBy、offset、fetch 对应着Sort, From 对应着TableScan/Join等等。示例Sql最后会生成如下RelNode树:
```
LogicalProject
    LogicalFilter
        LogicalTableScan
```
2. RexNode 行表达式， 如RexLiteral(常量), RexCall(函数)， RexInputRef(输入引用)等， 还是这一句SQL select id, cast(score as int), 'hello' from T where id < ?, 其中id 为RexInputRef, cast为RexCall, 'hello' 为RexLiteral等

3. Traits 转化特征，存在于RelNode中，目前有三种Traits: Convention、RelCollation、 RelDistribution。 

# Step4: 优化阶段（RelNode–>RelNode）  
终于来来到了第四阶段，也就是 Calcite 的核心所在，优化器进行优化的地方。此过程是利用Planner进行RBO和CBO优化，请参考后面关于RBO和CBO的介绍。 

# 参考  

1.[Apache Calcite 处理流程详解（一）](http://matt33.com/2019/03/07/apache-calcite-process-flow/#Step3-%E8%AF%AD%E4%B9%89%E5%88%86%E6%9E%90%EF%BC%88SqlNode%E2%80%93-gt-RelNode-RexNode%EF%BC%89)  

2.[Apache Calcite 处理流程详解（二）](http://matt33.com/2019/03/17/apache-calcite-planner/#VolcanoPlanner)  

3.[Calcite 若干概念说明](https://zhuanlan.zhihu.com/p/65701467)  

4.[基于calcite做sql优化](https://www.cnblogs.com/niutao/articles/13982876.html)
