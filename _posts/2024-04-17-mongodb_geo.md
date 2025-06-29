---
title: MongoDB와 지리검색 
layout: post
tag: [mongodb]
toc: true
---

[배달앱은 어떻게 내 주변의 맛집을 찾을까?](https://inf.run/3BEWe) 파트1에 대한 개인적인 정리 글 입니다.

## MongoDB에서 지리공간 데이터 다루는 방법

MongoDB는 다양한 형태의 데이터를 저장하고 검색할 수 있는 유연성을 제공하며, 지리공간 데이터의 처리도 예외는 아닙니다. MongoDB에서 지리공간 데이터를 다루기 위해, 첫 번째 단계는 적절한 인덱스를 생성하는 것입니다. 지리공간 인덱스를 사용하면, 효율적으로 위치 기반 데이터를 쿼리할 수 있습니다.

### 인덱스 
[MongoDB Geospatial Indexes](https://www.mongodb.com/docs/manual/core/indexes/index-types/index-geospatial/)   

MongoDB에서 제공하는 지리공간 인덱스 유형은 2dsphere와 2d가 있습니다. 
#### 2dsphere 
* 개념: 2dsphere 인덱스는 지구와 같은 구형 모델을 기반으로 공간 데이터를 처리합니다. 이 유형의 인덱스는 지리적 위치 데이터를 위해 설계되었으며, 위도와 경도를 사용하는 좌표계에서 효과적입니다.
* 사용 사례: 2dsphere 인덱스는 어떠한 형태의 지리공간 객체(포인트, 라인, 폴리곤)도 지원하며, 이를 이용해 거리 측정, 면적 찾기, 경계 내/경계 교차 등의 복잡한 공간 쿼리를 수행할 수 있습니다.
* 장점: 실제 지구와 같은 구면 좌표계에 최적화되어 있어 정확하고 복잡한 지리공간 쿼리를 지원합니다.

#### 2d
* 개념: 2d 인덱스는 평면적인 좌표계를 사용하여 공간 데이터를 처리합니다. 주로 간단한 평면 지도 또는 카테시안 평면 위의 데이터에 사용됩니다.
* 사용 사례: 이 인덱스는 주로 간단한 2차원 위치 검색에 사용되며, X와 Y 좌표를 사용하는 데이터에 적합합니다. 거리 측정 및 직사각형 범위 내의 객체를 찾는 데 주로 활용됩니다.
* 장점: 2d 인덱스는 평면 데이터에 대한 간단하고 빠른 쿼리 수행이 가능하며, 주로 간단한 거리 측정과 범위 쿼리에 유용합니다.

#### 차이점
* 좌표계 : 2dsphere는 구형 좌표계를, 2d는 평면 좌표계를 사용합니다.
* 지원하는 데이터 유형 : 2dsphere는 다양한 지리공간 데이터 유형(포인트, 라인, 폴리곤 등)을 지원하는 반면, 2d는 주로 2차원 포인트에 최적화되어 있습니다.
* 적용 가능한 쿼리 유형 : 2dsphere는 보다 복잡하고 다양한 지리공간 쿼리를 지원하지만, 2d는 간단한 2차원 범위와 거리 측정에 초점을 맞춥니다.


### 데이터 타입
MongoDB는 Point와 Polygon을 포함한 여러 지리공간 데이터 형식을 지원합니다. Point는 단일 위치를 나타내며, Polygon은 다각형 영역을 표현합니다. 

#### Point
Point는 위도와 경도를 하나로 묶은 자료구조입니다. MongoDB에서는 geo json 양식을 사용합니다. 
![image](assets/img/posts/geo-point.png)
```json
{
  "type": "Point",
  "coordinates": [127.027667, 37.498563]
}
```
longitude, latitude 순서

Point 데이터를 활용한 검색은 위치 기반 서비스에서 매우 흔하게 사용됩니다. 사용자의 현재 위치에서 가장 가까운 매장을 찾거나, 특정 범위 내의 관심 지점(POI)을 조회하는 것이 예입니다.

#### Polygon
`point` 점들이 모인 선 `linestring`이고,  `linestring` 선들이 모인 면이 `polygon` 입니다.
![image](assets/img/posts/geo-polygon.png)
```json
{
  "type": "Polygon",
  "coordinates": [
    [[0,0], [0,2], [2,2], [2,0], [0,0]],
    [[1,0], [1.5,0.5], [1,1], [0.5,0.5], [1,0]]
  ]
}
```

> ##### 💡 Right Hand Rule
`polygon`을 그릴 때는, 겉 부분은 시계방향으로 안쪽 부분은 반시계 방향으로 그려야 하는 규칙이 있다.

Polygon을 사용한 검색은 사용자가 정의한 특정 지역 내의 데이터를 찾는 데 사용됩니다. 예를 들어, 특정 지역 내의 이벤트나 광고를 찾기 위해 사용할 수 있습니다.

$geoWithin과 $geoIntersects 연산자를 사용하여 Polygon 내부 또는 교차하는 위치의 데이터를 검색할 수 있습니다.





### Google S2

[S2 공식문서](https://s2geometry.io/)

![image](assets/img/posts/google-s2.png)


Google S2 라이브러리는 지구를 셀로 나누어 공간 데이터를 효율적으로 색인하고 검색할 수 있는 기능을 제공합니다. 이 라이브러리는 지리공간 검색에서 고유한 접근 방식을 사용하여, 지구 표면을 구형 트리로 나타내며, 이는 S2 셀이라고 불리는 작은 영역으로 분할됩니다. 

| Google s2 를 한 마디로 말하자면 동그란 지구 표면 위에 힐버트 커브를 그려놓은 것

![image](assets/img/posts/geo-graph.png)


S2 인덱스 생성 전 Query Performance Summary 
![image](assets/img/posts/index-before.png)

S2 인덱스 생성 후 Query Performance Summary
![image](assets/img/posts/index-after.png)

* excecution time : **421ms -> 3ms**
* examined documents : **80000 -> 465**