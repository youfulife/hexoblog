title: python-mysql-zh
date: 2017-02-05 23:49:40
tags: python
categories: 编程
---

这两天看了很多关于mysql中文乱码的问题，除了创建table的时候设置为utf8编码以及修改mysql配置文件的方法外，很少有人提关于python库中中文乱码的处理办法，尤其是records库的中文乱码问题。

文中的代码在CentOS或者Ubuntu操作系统python3的环境下都测试没问题。

基于python3使用pymysql来读取mysql中的内容，在connect中一定要加入charset参数，否则中文在ubuntu或者centos下读出来显示一堆问号。

```
# -*- coding: utf-8 -*-
import pymysql
import config

if __name__ == '__main__':
    db = pymysql.connect(config.mysql_host, config.mysql_user, config.mysql_pass, config.mysql_db, charset='utf8')
    cursor = db.cursor()
    sql = "select name from user"
    cursor.execute(sql)
    for row in cursor.fetchall():
        print(row)
    db.close()
```

[records](https://github.com/kennethreitz/records)库是requests作者 [kennethreitz](https://github.com/kennethreitz) 写的一个非常方便的针对各种数据库进行数据处理的python库，只不过文档和网上的相关内容很少，尤其是中文的情况，如果不知道正确的使用方法很容易出现乱码。

```
mysql4read = 'mysql://{user}:{passwd}@{host}:3306/{db}'.format(host=host, user=user, passwd=pass, db=db)
db = records.Database(mysql4read, connect_args={"charset": "utf8"})
sql = "select name from user"
for row in db.query(sql).as_dict():
    print(row)
```

可以看到一定要在创建db对象的时候传入connect_args参数，否则中文很容易出现乱码。

