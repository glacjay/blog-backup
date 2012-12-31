---
categories:
  - Database
comments: true
layout: post
published: true
status: publish
tags:
  - oracle
  - sql
title: Oracle 建库 SQL 脚本
type: post
---

``` sql
-- 首先是删除该数据库中该用户名下的所有表、序列与触发器，
-- 其中触发器是通过表格级联删除的。
declare
  cursor usertables is
    select *
    from user_tables
    where table_name not like 'BIN$%';
  cursor usersequences is
    select *
    from user_sequences;
begin
  for next_row in usertables loop
    execute immediate
      'drop table ' || next_row.table_name ||
      ' cascade constraints';
  end loop;
  for next_row in usersequences loop
    execute immediate
      'drop sequence ' || next_row.sequence_name;
  end loop;
end;
/

-- 然后，就是一砣……的建表语句啦，
-- 比方说下面就是两个含外键的表。
create table "A" (
  "ID" integer primary key,
  "NAME" nvarchar2(64) not null         -- nvarchar2类型支持UTF8
);
-- Oracle中的表名与字段名最好写成这样引号加全大写的形式。

create table "B" (
  "ID" integer primary key,
  "A" integer not null references "B" on delete cascade,
  "VALUE" nvarchar2(64) not null
);

-- 接下来是第三部分，为所有的表创建对应的序列和触发器，以实现自增主键。
declare
  cursor usertables is
    select cols.table_name, cols.column_name
    from all_constraints cons, all_cons_columns cols
    where cols.owner = (
      select sys_context('USERENV', 'SESSION_USER')
      from dual)
    and cols.table_name not like 'BIN$%'
    and cons.constraint_type = 'P'
    and cons.constraint_name = cols.constraint_name
    and cons.owner = cols.owner
begin
  for nextrow in usertables loop
    -- 1000以下的留作初始测试数据用，见下
    execute immediate
      'create sequence "' || nextrow.table_name ||
      '_PK_SEQ" start with 1000';
    execute immediate
      'create or replace trigger "' || nextrow.table_name ||
      '_PK_TRG" before insert on "' || nextrow.table_name ||
      '" for each row begin if :new."' || nextrow.column_name ||
      '" is null then select "' || nextrow.table_name ||
      '_PK_SEQ".nextval into :new."' || nextrow.column_name ||
      '" from dual; end if; end;';
  end loop;
end;
/

-- 最后当然就是测试数据啦。
insert into "A" (ID, NAME) values (1, 'first name');
insert into "B" (ID, AID, VALUE) values (1, 1, 'Jay');
insert into "B" (ID, AID, VALUE) values (2, 1, '杰');
```
