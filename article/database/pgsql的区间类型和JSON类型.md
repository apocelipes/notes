## Ranges区间类型

种类：

- daterange 日期区间，`[2024-04-01, 2024-05-01)`
- int4range, int8range 4字节整数或者8字节整数的离散区间
- numrange 实数的连续区间
- tsrange, tstzrange 时间区间

pg会默认把所有区间转换成对应的左闭右开区间。

操作：

```sql
-- 判断两个区间是否有重叠部分，包括闭区间的边界
'[2024-04-01, 2024-05-01)'::daterange && '[2024-03-01, 2024-04-02)'::daterange;
-- true

-- 判断某个元素或者区间是否包含在另一个区间内
select 10::bigint <@ '[9,11)'::int8range;           -- true
select '[9,11)'::int4range @> 10;                   -- true
select '[9,10)'::int8range <@ '[9,11)'::int8range;  -- true

-- lower返回左边界
select lower('[9,11)'::int8range);  -- 9
select lower('(9,11)'::int8range);  -- 10
-- lower始终会去到在range内的元素，因此9是开放边界的时候会取10

-- upper返回右边界
select upper('[9,11)'::int8range);  -- 11
select upper('[9,11]'::int8range);  -- 12
-- upper只取右边的边界，因为pg内部会把右闭区间转换成对应的开区间，因此[9,11] -> [9,12)，得到12

-- 创建range
<range-type>(左边界，有边界，'[]');  -- 第三个参数是边界的开闭情况，例子里的是两边都是闭合

-- 第一个区间是否在第二的左侧，两者直接不能有重叠
select int8range(1,10) << int8range(100,110);  -- true

-- 第一个区间是否在第二的右侧，两者直接不能有重叠
select int8range(100,110) >> int8range(1,10);  -- true

-- 第一个区间是否没有延伸到第二个的右侧或者左侧
select int8range(1,20) &< int8range(18,20);  -- true
select int8range(7,20) &> int8range(5,10);   -- true

-- 两个range是否相邻
select '[1, 10)'::int4range -|- '(10, 20)'::int4range;  -- false
select '[1, 10)'::int4range -|- '[10, 20)'::int4range;  -- true

-- 创建ranges上的GiST索引，可以加速上述查询
CREATE INDEX reservation_idx ON reservation USING GIST (during);

-- 合并两个区间，产生一个最小的包含两个区间的新区间
range_merge('[1,2)'::int4range, '[3,4)'::int4range) → [1,4)

-- r1+r2产生并集，r1*r2产生交集，r1-r2产生差集
-- isempty()检测区间是否为空
```

编程语言支持情况：

直接使用数据库驱动得到的都是字符串
python：django里有field对象支持
golang：不支持，但可以被当作字符串值；也可以用lower和upper，左闭右开；或者按github.com/go-gorm/datatypes的方式自定义字段类型

## JSON类型

还有个JSONB是二进制的JSON，提供的操作差不多但比文本类型更紧凑。

```sql
select '{"dataset":[{"info":{"data":"uuu"},"data":"aaa"},{"info":2,"data":"bbb"}],"name":"textt"}'::json->'dataset'->0#>'{info,data}'::text[];
```

`->`返回json或者jsonb对象，后面跟字段的名字或者数组的索引。
`#>`后面跟数组，数组里的东西会被拼接成json path，比如'info.data'，返回路径指向的字段的值。
`json_array_length(json)`返回数组的长度。

将查询结果转换成json对象：`select row_to_json(ret) from (查询) as ret`。每条记录都会变成一个json对象。

jsonb性能比json快，因为是已经解析过的二进制数据。jsonb还会删除重复的字段只保留其中一条。jsonb上可以创建gin索引，json上只能创建函数索引。

编程语言支持：

数据库驱动一样默认当成字符串处理。
python: django支持
golang: github.com/go-gorm/datatypes
