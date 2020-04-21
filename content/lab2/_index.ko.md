---
title: Aurora에서 SageMaker 모델 실행하기
weight: 40
pre: "<b>3. </b>"
---

이번 단계에서는 Aurora에서 SQL을 이용하여 SageMaker의 모델을 사용하는 실습입니다.

이 페이지는 [Aurora MySQL에서 기계 학습(ML) 사용](https://docs.aws.amazon.com/ko_kr/AmazonRDS/latest/AuroraUserGuide/mysql-ml.html)을 참고하여 작성되었습니다.

## Amazon RDS에서 Amazon SageMaker에 대한 IAM 액세스 권한 설정


## Amazon SageMaker에 대한 IAM 액세스 권한 설정


## Aurora 기계 학습 서비스 호출을 위한 SQL 권한 부여

데모에서는 admin 유저를 사용하고 있습니다. 만약 데이터베이스 유저에게 Aurora 기계 학습 서비스 호출 권한이 없다면 아래와 같이 추가해 줍니다.

```
GRANT INVOKE SAGEMAKER ON *.* TO 'user1'@'%' WITH GRANT OPTION;
```

## Aurora MySQL에서 다른 AWS 서비스로의 네트워크 통신 활성화



## Amazon SageMaker를 사용하여 자체 ML 모델 실행

```
CREATE FUNCTION recommend_score (
       user_id BIGINT(20),
       product_id BIGINT(20))
RETURNS double
       alias aws_sagemaker_invoke_endpoint
       endpoint name 'sagemaker-mxnet-2020-04-05-09-40-05-472';

CREATE FUNCTION recommend_score (
       user_id BIGINT(20))
RETURNS double
       alias aws_sagemaker_invoke_endpoint
       endpoint name 'sagemaker-mxnet-2020-04-05-09-40-05-472';
       
SELECT *, recommend_score(user_id,product_id) recommend_score FROM inputs;

CREATE TABLE inputs (
    user_id varchar(20) PRIMARY KEY,
    product_id varchar(20)
);

INSERT INTO inputs
VALUES (512505687,44600062);
```


```

CREATE TABLE IF NOT EXISTS inputs
    SELECT U.user_id, P.product_id
    FROM users AS U
    CROSS JOIN products AS P limit 100000;

CREATE FUNCTION recommend_score (
       user_id BIGINT(20),
       product_id BIGINT(20))
RETURNS double
       alias aws_sagemaker_invoke_endpoint
       endpoint name 'sagemaker-mxnet-2020-04-06-17-26-29-027';

CREATE TABLE IF NOT EXISTS predict
    SELECT *, recommend_score(user_id,product_id) recommend_score FROM inputs;

SELECT * FROM predict ORDER BY recommend_score DESC limit 10;

summit-2020-rds-sg

INSERT INTO inputs VALUES (112505680,14600062);


CREATE TABLE review (
  product_title varchar(100),
  review_body longtext,
  brand varchar(30) 
)

LOAD DATA LOCAL INFILE '/home/ec2-user/mart-final.txt' INTO TABLE review
FIELDS TERMINATED BY '|';

SELECT * FROM review limit 10;

SELECT product_title,brand,
       aws_comprehend_detect_sentiment(review_body, 'en') AS sentiment,
       aws_comprehend_detect_sentiment_confidence(review_body, 'en') AS confidence
  FROM review WHERE brand='sony';


1	asics
2	casio
3	philips
4	tissot
5	sony
6	ikea
7	nike
8	fossil
9	panasonic
10	bosch
11	caon
  
  
CREATE TABLE result
  SELECT A.user_id, A.product_id, A.recommend_score, B.product_name 
  FROM predict AS A 
      JOIN products AS B 
      ON A.product_id=B.product_id 
  WHERE product_name='sony'
  ORDER BY A.recommend_score DESC;
  
```

---
<p align="center">
© 2020 Amazon Web Services, Inc. 또는 자회사, All rights reserved.
</p>



