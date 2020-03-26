---
title: Amazon Aurora Database 만들기
weight: 20
pre: "<b>1. </b>"
---


## Aurora DB

1. 순서를 가이드합니다.



## MySQL 셋업 및 테이블 생성


```
mysql -h summit-db-cluster-instance-1.csz1mbf6avao.ap-northeast-2.rds.amazonaws.com -P 3306 -u admin -p
Enter password:
```

```
CREATE USER 'user1'@'%' IDENTIFIED BY 'password1';

GRANT SELECT, INSERT, UPDATE, DELETE, CREATE, DROP, RELOAD, PROCESS, REFERENCES, INDEX, ALTER, SHOW DATABASES, CREATE TEMPORARY TABLES, LOCK TABLES, EXECUTE, REPLICATION SLAVE, REPLICATION CLIENT, CREATE VIEW, SHOW VIEW, CREATE ROUTINE, ALTER ROUTINE, CREATE USER, EVENT, TRIGGER ON *.* TO 'user1'@'%' WITH GRANT OPTION;
```

```
CREATE TABLE products (
    product_id BIGINT PRIMARY KEY,
    category_id VARCHAR(30),
    category_code VARCHAR(30),
    brand VARCHAR(30),
    price FLOAT,
    product_name VARCHAR(30)
);

LOAD DATA LOCAL INFILE '/home/ec2-user/data/prod/products.csv' INTO TABLE products
FIELDS TERMINATED BY ','
OPTIONALLY ENCLOSED BY '"'
IGNORE 1 LINES;
```

```



```




---
<p align="center">
© 2020 Amazon Web Services, Inc. 또는 자회사, All rights reserved.
</p>
