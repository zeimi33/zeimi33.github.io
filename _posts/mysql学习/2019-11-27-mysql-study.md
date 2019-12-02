---
layout: post
title: mysql学习
category: dump
description: 南大通用mysql培训
---

# mysql添加命令
+ mysqlcommend.h中添加

+ 在ql_yacc.yy添加 token(yacc.yy要在cmake阶段是编译才会产生.h和.cc文件)
```
%token ME_SYM
```

# 语法解析
+ 在show_para中添加 ME_SYM |（每个命令都要以或的方式并存） 
```
{
    LEX *lex =lex
    lex->sql_command=SQLCOM_SHOW_ME;
}
```

# 修改 com_status_vars :sql/mysqld.cc
```
起一个名字 并且 关联命令
```

# sql_parse :dispatch_commend
```
switch (sql->mysql_commend):{

}
```

## plugin
+ 进入plug.h
+ 进入deamon_example.cc