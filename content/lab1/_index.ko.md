---
title: Amazon SageMaker 추천 모델 개발하기
weight: 30
pre: "<b>2. </b>"
---


이번 단계에서는 SageMaker를 이용하여 구매 이력으로부터 상품을 추천하눈 간단한 추천 모델을 만듭니다.


## SageMaker Notebook 생성하기


[Create a Notebook Instance](https://docs.aws.amazon.com/ko_kr/sagemaker/latest/dg/howitworks-create-ws.html)를 참고하여 노트북 인스턴스를 생성합니다.

1. Amazon SageMaker 콘솔로 이동합니다. [link](https://console.aws.amazon.com/sagemaker/)

2. 노트북 인스턴스를 선택한 후 노트북 인스턴스 생성을 선택합니다.

3. 노트북 인스턴스 생성 페이지에서 `노트북 인스턴스 이름`과 `노트북 인스턴스 유형`을 선택합니다.

```
노트북 인스턴스 이름: sg-notebook-reco
노트북 인스턴스 유형: ml.p2.xlarge
```

![pic](./images/lab1-1.png)

4. 권한 및 암호화에서 IAM 역할을 선택합니다. IAM 역할이 없으면 새로 생성합니다.

![pic](./images/lab1-2.png)

5. `노트북 인스턴스 생성`을 클릭하여 인스턴스를 생서합니다.

6. 노트북 인스턴스의 상태가 `Pending`에서 `InService`로 바뀌면 `Jupyter 열기`를 클릭하여 노트북에 접속합니다.


## 데이터 Explore / Clean / Prepare

이번 실습에서는 SageMaker에서 제공하는 Example 중 gluon_recommender_system을 참고하여 추천 모델을 만들기 위한 데이터 탐색 / 클리닝 / 전처리 과정을 거칩니다.
앞서 접속한 SageMaker 노트북에서 SageMaker Example 탭을 선택하면 Introduction to Applying Machine Learning의 gluon_recommender_system의 내용과 동일합니다.
실습에서 사용한 SageMaker 노트북은 [Github summit_2020_demo](https://github.com/elbanic/summit_2020_demo/notebooks)에서 다운로드할 수 있습니다.

![pic](./images/lab1-3.png)

1. 우측 상단의 New -> conda_mxnet_p36을 선택하여 노트북을 생성합니다. 노트북 이름은 `ecommerce_recommender_system`로 저장합니다.

2. S3 버킷과 Prefix를 지정합니다.

```
bucket = 'summit-2020-db-ml'
prefix = 'sagemaker/DEMO-gluon-recsys'

import sagemaker
role = sagemaker.get_execution_role()
```

3. 필요한 라이브러리를 import 합니다.

```
import os
import mxnet as mx
from mxnet import gluon, nd, ndarray
from mxnet.metric import MSE
import pandas as pd
import numpy as np
import sagemaker
from sagemaker.mxnet import MXNet
import boto3
import json
import matplotlib.pyplot as plt
```

4. 실습 1에서 저장했던 데이터를 다운로드합니다.

```
!mkdir /tmp/recsys/
!aws s3 cp s3://summit-2020-db-ml/ecommerce-behavior-data/org/2019-Oct.csv /tmp/recsys/
!aws s3 cp s3://summit-2020-db-ml/ecommerce-behavior-data/org/2019-Nov.csv /tmp/recsys/
```

5. 데이터를 Pandas Dataframe으로 변환합니다. 두 개의 파일을 읽어서 dataframe으로 합칩니다.

```
df_Oct = pd.read_csv('/tmp/recsys/2019-Oct.csv', delimiter=',',error_bad_lines=False)
df_Nov = pd.read_csv('/tmp/recsys/2019-Nov.csv', delimiter=',',error_bad_lines=False)
df = pd.concat([df_Oct,df_Nov], ignore_index=True)
df.head()
```

6. 학습에 필요한 컬럼만 가져옵니다. 'category_id', 'category_code', 'brand'는 학습에 사용되지는 않지만 후애 모델 검증에 사용됩니다.
'view, cart, remove_from_cart, purchase' 네 가지 타입은 학습에 사용될 주요 피쳐이기 때문에 디지털화합니다.

```
df = df[['user_id', 'product_id', 'event_type', 'category_id', 'category_code', 'brand']]
df['event_type_digit'] = df['event_type'].apply(lambda x: 4 if x=='purchase' else 3 if x=='cart' else 2 if x=='remove_from_cart' else 1)
```

7. 데이터가 어떻게 분포하는지 확인합니다.

```
users = df['user_id'].value_counts()
products = df['product_id'].value_counts()

quantiles = [0, 0.01, 0.02, 0.03, 0.04, 0.05, 0.1, 0.25, 0.5, 0.75, 0.9, 0.95, 0.96, 0.97, 0.98, 0.99, 1]
print('users\n', users.quantile(quantiles))
print('products\n', products.quantile(quantiles))
```

8. 너무 낮은 빈도의 데이터는 머신 러닝 학습에는 불필요한 노이즈가 될 수 있습니다.
5개 이상 트랜잭션이 있었던 user와 product만 남깁니다.

```
users = users[users >= 5]
products = products[products >= 5]

reduced_df = df.merge(pd.DataFrame({'user_id': users.index})).merge(pd.DataFrame({'product_id': products.index}))
```

9. Test용 데이터셋과 Train용 데이터셋을 나눕니다.


```
test_df = reduced_df.groupby('user_id').last().reset_index()

train_df = reduced_df.merge(test_df[['user_id', 'product_id']], 
                            on=['user_id', 'product_id'], 
                            how='outer', 
                            indicator=True)
train_df = train_df[(train_df['_merge'] == 'left_only')]
```

10. 각 데이터셋과 batch 사이즈를 설정합니다.

```
batch_size = 1024

train = gluon.data.ArrayDataset(nd.array(train_df['user'].values, dtype=np.float32),
                                nd.array(train_df['item'].values, dtype=np.float32),
                                nd.array(train_df['event_type_digit'].values, dtype=np.float32))
test  = gluon.data.ArrayDataset(nd.array(test_df['user'].values, dtype=np.float32),
                                nd.array(test_df['item'].values, dtype=np.float32),
                                nd.array(test_df['event_type_digit'].values, dtype=np.float32))

train_iter = gluon.data.DataLoader(train, shuffle=True, num_workers=4, batch_size=batch_size, last_batch='rollover')
test_iter = gluon.data.DataLoader(train, shuffle=True, num_workers=4, batch_size=batch_size, last_batch='rollover')
```


## 모델 학습하기





## 모델 평가하기





실습에서 사용한 SageMaker 노트북은 [Github summit_2020_demo](https://github.com/elbanic/summit_2020_demo/notebooks)에서 다운로드할 수 있습니다.


---
<p align="center">
© 2020 Amazon Web Services, Inc. 또는 자회사, All rights reserved.
</p>
