---
title: IdMaker
toc: true
date: 2021-09-26 14:19:56
thumbnail: /images/2021/岁月的童话2.jpg
tags:
  - other
  - blog
categories:
  - other

---



<!--more-->


```sql
create table `id_maker`
(
    `business` varchar(255)            NOT NULL COMMENT '业务',
    `next_id`  decimal(64, 0) unsigned NOT NULL default 1 COMMENT '下一次id',
     PRIMARY KEY (`business`) USING BTREE
) ENGINE = InnoDB
  DEFAULT CHARSET = utf8mb4 COMMENT ='idMaker';
```
```sql
insert into id_maker values ('default',1);
insert into id_maker values ('eat_food_00',1);
insert into id_maker values ('eat_food_01',101);
insert into id_maker values ('eat_food_02',201);
insert into id_maker values ('eat_food_03',301);
insert into id_maker values ('eat_food_04',401);
```

```sql
begin;
select next_id from id_maker where business = 'default';
update id_maker set next_id = next_id+100 where business = 'default';
commit;
```

```sql
begin;
select next_id from id_maker where business = 'default';
update id_maker set next_id = next_id+500 where eat_food_01 = 'default';
commit;
```


```php
class IdMaker
{
  public $idMaker = 'default';
  public $maxId = '0';
  public $nextId = '0';
  public function __construct()
  {
    $this->idMaker = 'eat_food_'.rand_mt(0,4);
    $this->updateIdMaker();
  }

  public function getNextId()
  {
    if($nextId>$maxId){
        $this->updateIdMaker();
    }
    return $this->nextId++;
  }

  protected function updateIdMaker()
  {
    // 通过 idMaker 获取最新的
    $this->nextId = $maxId;
    $this->maxId = $nextId + 100;
  }
}
```
