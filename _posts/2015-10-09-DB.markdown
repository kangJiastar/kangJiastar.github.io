---
layout: post
title:  "数据库"
date:   2015-10-09 13:26:30
categories: jekyll update
---

##SQLite3

#SQL语句定义

SQL语句是对关系数据库进行定义和操作的语句。

 - 注意：
 
    1.要以`;`结尾。
    
    2.SQLite是无类型的，但是为了程序员交流最好写上类型
      

#增删改查（CRUD） create、read、update、delete。

 - 创建表
 
  `CREATE TABLE IF NOT EXISTS t_dog(name text,age integer);`
  
  1.有主键限制的创建`PRIMARY KEY`,主键包含了唯一性。主键会自增长，不能设置为字符串
  
`CREATE TABLE IF NOT EXISTS t_pig(id integer PRIMARY KEY)`

2.`NOT NULL`标识某个字段为非空

`CREATE TABLE IF NOT EXISTS t_dog(name text NOT NULL,age integer);`

3.`UNIQUE`字段名称唯一，如果有一个叫“李四”，不能出现第二个“李四”

`CREATE TABLE IF NOT EXISTS t_dog(name text NOT NULL UNIQUE,age integer);`

4.`DEFAULT`设置默认值,可以根据设置改变

`CREATE TABLE IF NOT EXISTS t_cat(name text NOT NULL UNIQUE,age integer DEFAULT 1);`
  
 - 删除表
 
`DROP TABLE IF EXISTS t_dog;`

 - 增（插入）
 
`INSERT INTO t_shop (name , price ,left_count) VALUES ('手机',2001.0,500);`

注：使用单引号，属性可以不写全
`INSERT INTO t_shop (name,left_count) VALUES ('风扇', 600);`

 - 删
 
1.与AND

`DELETE FROM t_shop WHERE price <= 1202 AND left_count < 100`

2.或OR

`DELETE FROM t_shop WHERE price <= 1202 OR left_count < 100`

 - 改
 
`UPDATE t_shop SET price = 900 ,left_count = 200;`

 - 查
 
`SELECT * FROM t_shop`

`SELECT * FROM t_shop WHERE price > 800`

1.默认是升序

`SELECT * FROM t_shop ORDER BY price`

2.降序

`SELECT * FROM t_shop ORDER BY price DESC;`

3.升序

`SELECT * FROM t_shop ORDER BY price ASC;`

4.先按照price升序，然后按照left_count降序

`SELECT * FROM t_shop ORDER BY price ASC , left_count DESC`

5.模糊查询

`SELECT * FROM t_shop WHERE price like ‘%800%’`

###常见操作
 - 计算记录的数量
 
 1.`SELECT count(name) FROM t_shop `
 
 2.常用（只计算一次，同样是统计记录的数量）
 
`SELECT count(*) 剩余数量 FROM t_shop `

 - 分页
 
LIMIT 0,2 等价于 LIMIT 2

`SELECT * FROM t_shop ORDER BY price DESC LIMIT 0 ,2`

###FMDB
 - executeQuery:查询数据
`- (FMResultSet *)executeQuery:(NSString*)sql, ...;`
 - executeUpdate:除查询数据以外的其他操作
`- (BOOL)executeUpdate:(NSString*)sql, ...;`

{% highlight objc %}
 // 1.打开数据库
    NSString *path = [[NSSearchPathForDirectoriesInDomains(NSDocumentDirectory, NSUserDomainMask, YES) lastObject] stringByAppendingPathComponent:@"shops.sqlite"];
    _db = [FMDatabase databaseWithPath:path];
    [_db open];
    
// 2.创表
[_db executeUpdate:@"create table if not exists t_collects (id integer primary key autoincrement,createat text not null, access_token text not null,workid integer not null,list blob,userId integer not null,Fid integer not null unique );"];

//根据字典保存到数据库
+ (void)saveCollect:(NSDictionary *)dict
{
    if (dict == nil) {
        return;
    }
    NSNumber *userId = [YQUserInfoModel takeUserToken].ID;
    FMResultSet *set = nil;
    set = [_db executeQuery:@"select * from t_collects where userId = ? and workid = ?;",userId,dict[@"ID"]];
    NSData *data = [NSJSONSerialization dataWithJSONObject:dict options:0 error:nil];
    if ([set next]) {
        [_db executeUpdate:@"UPDATE t_collects SET list = ? WHERE workid = ? ;",data,dict[@"ID"]];
    }else{
        [_db executeUpdate:@"insert into t_collects (userId,list,createat,workid,access_token,Fid) values (?, ?, ?, ?, ?, ?);",userId,data,dict[@"CreateAt"],dict[@"ID"],@"collect",dict[@"Fid"]];
    }
    [set close];
}

//从数据库取出全部数据
+ (NSMutableArray *)cacheCollects
{
    NSNumber *userId = [YQUserInfoModel takeUserToken].ID;
    FMResultSet *set = nil;
    set = [_db executeQuery:@"select * from t_collects where userId = ? order by Fid desc;",userId];
    NSMutableArray *listArr = [NSMutableArray array];
    while([set next]) {
        NSData *data = [set dataForColumn:@"list"];
        NSDictionary *dict = [NSJSONSerialization JSONObjectWithData:data options:0 error:nil];
        YQWorkList *list = [YQWorkList modelWithKeyValues:dict];
        [listArr addObject:list];
    }
    [set close];
    return listArr;
}
//根据主键查询信息
+ (YQWorkList *)searchCollectsWithID:(NSInteger )ID
{
    FMResultSet *set = nil;
    set = [_db executeQuery:@"select * from t_collects where workid = ?;",@(ID)];
    YQWorkList *list = nil;
    if ([set next]) {
        NSData *data = [set dataForColumn:@"list"];
        NSDictionary *dict = [NSJSONSerialization JSONObjectWithData:data options:0 error:nil];
        list = [YQWorkList modelWithKeyValues:dict];
    }
    [set close];
    return list;
}

//删除数据库
+ (void)deleteCollectsWithID:(NSInteger)ID
{
    [_db executeUpdate:@"delete from t_collects where workid = ?;",@(ID)];
}

+ (void)deleteCollects
{
    [_db executeUpdate:@"delete from t_collects ;"];
}

{% endhighlight %}

###[将图片保存到数据库][jekyll - DB]

	
[jekyll - DB]:     http://m.blog.csdn.net/blog/wei78008023/44903541

