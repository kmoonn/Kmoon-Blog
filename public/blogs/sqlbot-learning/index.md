SQLBot 是一款基于**大语言模型**和 **RAG** 的智能问数系统，用户可以实现对话式数据分析（**ChatBI**），快速提炼获取所需的数据信息及可视化图表。

# Text2SQL

SQLBot 的 Text2SQL 核心是**基于大模型 + RAG** 实现自然语言到 SQL 的转换。

![](https://image.kmoon.fun/20260227-303972.png)


## Step 1. 上下文构建

大模型生成准确 SQL 的前提是**熟悉业务上下文 + 数据库规则**，SQLBot 从**数据源元数据、术语库、SQL 模板**三个维度收集上下文。

1. 数据源元数据

包括数据库表、字段信息

- 源码文件：`/backend/apps/db/db_sql.py`
- 核心函数解析

`get_version_sql`：获取数据库版本，支持不同类型的数据库。

```python
def get_version_sql(ds: CoreDatasource, conf: DatasourceConf):
    if equals_ignore_case(ds.type, "mysql", "doris", "starrocks"):
        return """
                SELECT VERSION()
                """
```

`get_table_sql`：获取指定数据库 / 模式下的表名 + 表注释

```python
def get_table_sql(ds: CoreDatasource, conf: DatasourceConf, db_version: str = ''):
    if equals_ignore_case(ds.type, "mysql"):
        return """
                SELECT 
                    TABLE_NAME, 
                    TABLE_COMMENT
                FROM 
                    information_schema.TABLES
                WHERE 
                    TABLE_SCHEMA = :param
                """, conf.database
```

`get_field_sql`：获取指定表的字段名、数据类型、字段注释。

```python
def get_field_sql(ds: CoreDatasource, conf: DatasourceConf, table_name: str = None):
    if equals_ignore_case(ds.type, "mysql"):
        sql1 = """
                SELECT 
                    COLUMN_NAME,
                    DATA_TYPE,
                    COLUMN_COMMENT
                FROM 
                    INFORMATION_SCHEMA.COLUMNS
                WHERE 
                    TABLE_SCHEMA = :param1
                """
        sql2 = " AND TABLE_NAME = :param2" if table_name is not None and table_name != "" else ""
        return sql1 + sql2, conf.database, table_name
```

其中`equals_ignore_case` 用于数据库大小写兼容，核心功能是检查第一个字符串参数 str1 是否与后续任意一个字符串参数（忽略大小写）相等，同时还处理了 None 值的情况。

```python
# SQLBot/backend/common/utils/utils.py

def equals_ignore_case(str1: str, *args: str) -> bool:
    if str1 is None:
        return None in args
    for arg in args:
        if arg is None:
            continue
        # 使用 casefold() 而非 lower()，因为 casefold() 是更严格的大小写转换方法
        if str1.casefold() == arg.casefold():
            return True
    return False
```

2. 术语库整合

业务术语 ——> 数据库映射

- 源码文件：`/backend/apps/terminology/curd/terminology.py`
- 核心函数解析

`build_terminology_query`：将业务术语映射到数据库表/字段。

```python
def build_terminology_query(session: SessionDep, oid: int, name: Optional[str] = None,
                            paginate: bool = True, current_page: int = 1, page_size: int = 10,
                            dslist: Optional[list[int]] = None):
    """
    构建术语查询的通用方法
    """
    
    # 1. 基础查询 + 数据源筛选（只查询当前数据源关联的术语）
    parent_ids_subquery, child = get_terminology_base_query(oid, name)

    # 添加数据源筛选条件
    if dslist is not None and len(dslist) > 0:
        datasource_conditions = []
        # datasource_ids 与 dslist 中的任一元素有交集
        for ds_id in dslist:
            # 使用 JSONB 包含操作符，但需要确保类型正确
            datasource_conditions.append(
                Terminology.datasource_ids.contains([ds_id])
            )

        # datasource_ids 为空数组
        empty_array_condition = Terminology.datasource_ids == []

        ds_filter_condition = or_(
            *datasource_conditions,
            empty_array_condition
        )
        parent_ids_subquery = parent_ids_subquery.where(ds_filter_condition)

    # 计算总数
    count_stmt = select(func.count()).select_from(parent_ids_subquery.subquery())
    total_count = session.execute(count_stmt).scalar()

    if paginate:
        # 分页处理
        page_size = max(10, page_size)
        total_pages = (total_count + page_size - 1) // page_size
        current_page = max(1, min(current_page, total_pages)) if total_pages > 0 else 1

        paginated_parent_ids = (
            parent_ids_subquery
            .order_by(Terminology.create_time.desc())
            .offset((current_page - 1) * page_size)
            .limit(page_size)
            .subquery()
        )
    else:
        # 不分页，获取所有数据
        total_pages = 1
        current_page = 1
        page_size = total_count if total_count > 0 else 1

        paginated_parent_ids = (
            parent_ids_subquery
            .order_by(Terminology.create_time.desc())
            .subquery()
        )

    # 构建公共查询部分
    # 2. 聚合术语关联信息（同义词/数据源名称）
    children_subquery = (
        select(
            child.pid,
            func.jsonb_agg(child.word).filter(child.word.isnot(None)).label('other_words')
        )
        .where(child.pid.isnot(None))
        .group_by(child.pid)
        .subquery()
    )

    # 创建子查询来获取数据源名称
    datasource_names_subquery = (
        select(
            func.jsonb_array_elements(Terminology.datasource_ids).cast(BigInteger).label('ds_id'),
            Terminology.id.label('term_id')
        )
        .where(Terminology.id.in_(paginated_parent_ids))
        .subquery()
    )
    
    # 3. 最终查询：返回术语 + 同义词 + 数据源名称 + 状态等
    stmt = (
        select(
            Terminology.id,
            Terminology.word,
            Terminology.create_time,
            Terminology.description,
            Terminology.specific_ds,
            Terminology.datasource_ids,
            children_subquery.c.other_words,
            func.jsonb_agg(CoreDatasource.name).filter(CoreDatasource.id.isnot(None)).label('datasource_names'),
            Terminology.enabled
        )
        .outerjoin(
            children_subquery,
            Terminology.id == children_subquery.c.pid
        )
        .outerjoin(
            datasource_names_subquery,
            datasource_names_subquery.c.term_id == Terminology.id
        )
        .outerjoin(
            CoreDatasource,
            CoreDatasource.id == datasource_names_subquery.c.ds_id
        )
        .where(and_(Terminology.id.in_(paginated_parent_ids), Terminology.oid == oid))
        .group_by(
            Terminology.id,
            Terminology.word,
            Terminology.create_time,
            Terminology.description,
            Terminology.specific_ds,
            Terminology.datasource_ids,
            children_subquery.c.other_words,
            Terminology.enabled
        )
        .order_by(Terminology.create_time.desc())
    )

    return stmt, total_count, total_pages, current_page, page_size
```

按数据源筛选术语，只加载当前数据源相关的业务术语；聚合术语的同义词/别名；关联数据源名称，确保术语与数据源匹配。

3. SQL 模板加载

数据库语法规则 + 示例模板

- 模板文件夹：`/backend/templates/sql_examples/`
- 模板加载函数：`/backend/apps/template/generate_sql/generator.py`

模板核心结构（以 MySQL 为例）
```yaml
template:
  quot_rule: |
    <rule>
      必须对数据库名、表名、字段名、别名外层加反引号（`）。
      <note>
        1. 点号（.）不能包含在反引号内，必须写成 `schema`.`table`
        2. 当标识符为关键字、含特殊字符或需保留大小写时必须加反引号
        3. MySQL 默认不区分大小写（Linux系统区分表名大小写）
      </note>
    </rule>

  limit_rule: |
    <rule>
      当需要限制行数时，必须使用标准的LIMIT语法
      <note>
        1. 标准写法：LIMIT [count]
        2. 分页写法：LIMIT [offset], [count] 或 LIMIT [count] OFFSET [start]
      </note>
    </rule>

  other_rule: |
    <rule>必须为每个表生成别名（不加AS）</rule>
    {multi_table_condition}
    <rule>禁止使用星号(*)，必须明确字段名</rule>
    <rule>中文/特殊字符字段需保留原名并添加英文别名</rule>
    <rule>函数字段必须加别名</rule>
    <rule>百分比显示为：CONCAT(ROUND(x*100,2),'%')</rule>
    <rule>避免与MySQL关键字冲突（如`order`/`group`/`desc`）</rule>

  basic_example: |
    <basic-examples>
      <intro>
        📌 以下示例严格遵循<Rules>中的 MySQL 规范，展示符合要求的 SQL 写法与典型错误案例。
        ⚠️ 注意：示例中的表名、字段名均为演示虚构，实际使用时需替换为用户提供的真实标识符。
        🔍 重点观察：
          1. 双引号包裹所有数据库对象的规范用法
          2. 中英别名/百分比/函数等特殊字段的处理
          3. 关键字冲突的规避方式
      </intro>
      <example>
        <input>查询 test.orders 表的前100条订单（含中文字段和百分比）</input>
        <output-bad>
          SELECT * FROM test.orders LIMIT 100  -- 错误：使用星号
          SELECT `订单ID`, `金额` FROM `test`.`orders` `t1` LIMIT 100  -- 错误：缺少英文别名
          SELECT COUNT(`订单ID`) FROM `test`.`orders` `t1`  -- 错误：函数未加别名
        </output-bad>
        <output-good>
          SELECT 
            `t1`.`订单ID` AS `order_id`,
            `t1`.`金额` AS `amount`,
            COUNT(`t1`.`订单ID`) AS `total_orders`,
            CONCAT(ROUND(`t1`.`折扣率` * 100, 2), '%') AS `discount_percent`
          FROM `test`.`orders` `t1`
          LIMIT 100
        </output-good>
      </example>

      <example>
        <input>统计用户表 public.users（含关键字字段desc）的活跃占比</input>
        <output-bad>
          SELECT desc, status FROM public.users  -- 错误：未处理关键字
          SELECT `desc`, ROUND(active_ratio*100) FROM `public`.`users`  -- 错误：百分比格式错误
        </output-bad>
        <output-good>
          SELECT 
            `u`.`desc` AS `description`,
            CONCAT(ROUND(`u`.`active_ratio` * 100, 2), '%') AS `active_percent`
          FROM `public`.`users` `u`
          WHERE `u`.`status` = 1
          LIMIT 1000
        </output-good>
      </example>
    </basic-examples>

  example_engine: MySQL 8.0
  example_answer_1: |
    {"success":true,"sql":"SELECT `country` AS `country_name`, `continent` AS `continent_name`, `year` AS `year`, `gdp` AS `gdp_usd` FROM `Sample_Database`.`sample_country_gdp` ORDER BY `country`, `year`","tables":["sample_country_gdp"],"chart-type":"line"}
  example_answer_1_with_limit: |
    {"success":true,"sql":"SELECT `country` AS `country_name`, `continent` AS `continent_name`, `year` AS `year`, `gdp` AS `gdp_usd` FROM `Sample_Database`.`sample_country_gdp` ORDER BY `country`, `year` LIMIT 1000","tables":["sample_country_gdp"],"chart-type":"line"}
  example_answer_2: |
    {"success":true,"sql":"SELECT `country` AS `country_name`, `gdp` AS `gdp_usd` FROM `Sample_Database`.`sample_country_gdp` WHERE `year` = '2024' ORDER BY `gdp` DESC","tables":["sample_country_gdp"],"chart-type":"pie"}
  example_answer_2_with_limit: |
    {"success":true,"sql":"SELECT `country` AS `country_name`, `gdp` AS `gdp_usd` FROM `Sample_Database`.`sample_country_gdp` WHERE `year` = '2024' ORDER BY `gdp` DESC LIMIT 1000","tables":["sample_country_gdp"],"chart-type":"pie"}
  example_answer_3: |
    {"success":true,"sql":"SELECT `country` AS `country_name`, `gdp` AS `gdp_usd` FROM `Sample_Database`.`sample_country_gdp` WHERE `year` = '2025' AND `country` = '中国'","tables":["sample_country_gdp"],"chart-type":"table"}
  example_answer_3_with_limit: |
    {"success":true,"sql":"SELECT `country` AS `country_name`, `gdp` AS `gdp_usd` FROM `Sample_Database`.`sample_country_gdp` WHERE `year` = '2025' AND `country` = '中国' LIMIT 1000","tables":["sample_country_gdp"],"chart-type":"table"}
```

模板加载函数：根据数据库类型，加载对应的模板；提取模板中的**规则 + 示例**，作为 Prompt 的核心约束。

```python
from typing import Union

from apps.db.constant import DB
from apps.template.template import get_base_template, get_sql_template as get_base_sql_template


def get_sql_template():
    template = get_base_template()
    return template['template']['sql']


def get_sql_example_template(db_type: Union[str, DB]):
    template = get_base_sql_template(db_type)
    return template['template']
```

## Step 2. Prompt 结构化构建

```text
【系统指令】
你是专业的 Text2SQL 助手，需遵循以下规则生成 SQL：
1. 语法规则：{quot_rule + limit_rule + other_rule}（来自 SQL 模板）
2. 输出格式：返回 JSON 结构 {"success":bool,"sql":"","tables":[""],"chart-type":""}
3. 禁止生成危险 SQL（DROP/ALTER 等）
【数据字典】
当前数据源（PostgreSQL）的表/字段信息：
- 表："sample_country_gdp"（国家GDP表）
  字段："country"（国家）、"continent"（大洲）、"year"（年份）、"gdp"（GDP值）
【业务术语】
- "中国GDP" → "sample_country_gdp"."country" = '中国' AND "sample_country_gdp"."gdp"
- "年度GDP" → "sample_country_gdp"."year" + "sample_country_gdp"."gdp"
【参考示例】
{basic_example}（来自 SQL 模板的正确/错误示例）
【用户问题】
{用户的自然语言提问，如：查询2024年中国的GDP}
【生成要求】
1. 严格遵循语法规则，使用双引号包裹所有标识符；
2. 字段必须加英文别名，禁止使用*；
3. 生成的 SQL 需可直接执行，无需修改；
4. 返回指定 JSON 格式，仅输出 JSON 内容。
```

## Step 3. 大模型调用

生成 SQL

- 源码文件：`/backend/apps/ai_model/model_factory.py`
- 作用：初始化大模型客户端，将构建好的 Prompt 传入大模型，生成 SQL。
- 核心代码：

```python
class OpenAIAzureLLM(BaseLLM):
    def _init_llm(self) -> AzureChatOpenAI:
        # 提取 API 版本、部署名
        api_version = self.config.additional_params.get("api_version")
        deployment_name = self.config.additional_params.get("deployment_name")
        if api_version:
            self.config.additional_params.pop("api_version")
        if deployment_name:
            self.config.additional_params.pop("deployment_name")
        
        # 初始化 Azure OpenAI 客户端
        return AzureChatOpenAI(
            azure_endpoint=self.config.api_base_url,
            api_key=self.config.api_key or 'Empty',
            model_name=self.config.model_name,
            api_version=api_version,
            deployment_name=deployment_name,
            streaming=True,
            **self.config.additional_params,
        )
```

## Step 4. SQL 验证与执行

1. SQL 执行验证

- 源码文件：`/frontend/src/api/datasource.ts`
- 传入「数据源 ID + 生成的 SQL」，调用后端执行

```typescript
import { request } from '@/utils/request'

export const datasourceApi = {
  check: (data: any) => request.post('/datasource/check', data),
  check_by_id: (id: any) => request.get(`/datasource/check/${id}`),
  relationGet: (id: any) => request.post(`/table_relation/get/${id}`),
  relationSave: (dsId: any, data: any) => request.post(`/table_relation/save/${dsId}`, data),
  add: (data: any) => request.post('/datasource/add', data),
  list: () => request.get('/datasource/list'),
  update: (data: any) => request.post('/datasource/update', data),
  delete: (id: number, name: string) => request.post(`/datasource/delete/${id}/${name}`),
  getTables: (id: number) => request.post(`/datasource/getTables/${id}`),
  getTablesByConf: (data: any) => request.post('/datasource/getTablesByConf', data),
  getFields: (id: number, table_name: string) =>
    request.post(`/datasource/getFields/${id}/${table_name}`),
  // 执行 SQL
  execSql: (id: number | string, sql: string) =>
    request.post(`/datasource/execSql/${id}`, { sql: sql }),
  
  chooseTables: (id: number, data: any) => request.post(`/datasource/chooseTables/${id}`, data),
  tableList: (id: number) => request.post(`/datasource/tableList/${id}`),
  fieldList: (id: number, data = { fieldName: '' }) =>
    request.post(`/datasource/fieldList/${id}`, data),
  edit: (data: any) => request.post('/datasource/editLocalComment', data),
  previewData: (id: number, data: any) => request.post(`/datasource/previewData/${id}`, data),
  saveTable: (data: any) => request.post('/datasource/editTable', data),
  saveField: (data: any) => request.post('/datasource/editField', data),
  getDs: (id: number) => request.post(`/datasource/get/${id}`),
  cancelRequests: () => request.cancelRequests(),
  getSchema: (data: any) => request.post('/datasource/getSchemaByConf', data),
  syncFields: (id: number) => request.post(`/datasource/syncFields/${id}`),
  exportDsSchema: (id: any) =>
    request.get(`/datasource/exportDsSchema/${id}`, {
      responseType: 'blob',
      requestOptions: { customError: true },
    }),
}
```

2. SQL 校准与迭代优化

- 源码文件：`/backend/apps/data_training/curd/data_training.py`
- 存储用户的「自然语言问题 + 人工校准的 SQL」；后续生成 SQL 时，将这些训练数据加入 Prompt 参考；基于用户交互数据持续迭代，优化生成效果

```python
def build_data_training_query(session: SessionDep, oid: int, name: Optional[str] = None,
                              paginate: bool = True, current_page: int = 1, page_size: int = 10):
    """
    构建数据训练查询的通用方法
    """
    # 1. 查询用户标注的「问题-SQL」训练数据
    parent_ids_subquery = get_data_training_base_query(oid, name)

    # 计算总数
    count_stmt = select(func.count()).select_from(parent_ids_subquery.subquery())
    total_count = session.execute(count_stmt).scalar()

    if paginate:
        # 分页处理
        page_size = max(10, page_size)
        total_pages = (total_count + page_size - 1) // page_size
        current_page = max(1, min(current_page, total_pages)) if total_pages > 0 else 1

        paginated_parent_ids = (
            parent_ids_subquery
            .order_by(DataTraining.create_time.desc())
            .offset((current_page - 1) * page_size)
            .limit(page_size)
            .subquery()
        )
    else:
        # 不分页，获取所有数据
        total_pages = 1
        current_page = 1
        page_size = total_count if total_count > 0 else 1

        paginated_parent_ids = (
            parent_ids_subquery
            .order_by(DataTraining.create_time.desc())
            .subquery()
        )

    # 构建主查询
    # 2. 分页获取训练数据（关联数据源、大模型信息）
    stmt = (
        select(
            DataTraining.id,
            DataTraining.oid,
            DataTraining.datasource,
            CoreDatasource.name,
            DataTraining.question,
            DataTraining.create_time,
            DataTraining.description,
            DataTraining.enabled,
            DataTraining.advanced_application,
            AssistantModel.name.label('advanced_application_name'),
        )
        .outerjoin(CoreDatasource, and_(DataTraining.datasource == CoreDatasource.id))
        .outerjoin(AssistantModel,
                   and_(DataTraining.advanced_application == AssistantModel.id, AssistantModel.type == 1))
        .where(and_(DataTraining.id.in_(paginated_parent_ids)))
        .order_by(DataTraining.create_time.desc())
    )

    return stmt, total_count, total_pages, current_page, page_size
```


> 参考：https://mp.weixin.qq.com/s/G4wAKt-m1kJIIgMhKCDhzQ