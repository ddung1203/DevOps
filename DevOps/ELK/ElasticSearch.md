# ElasticSearch 기본

## 인덱스와 도큐먼트

인덱스는 도큐먼트를 저장하는 논리적 구분자이며, 도큐먼트는 실제 데이터를 저장하는 단위다. 일반적으로 엘라스틱을 이용해 시스템을 개발하면 하나의 프로젝트에서 하나의 클러스터를 생성한다. 그리고 클러스터 내부는 데이터 성격에 따라 여러 개의 인덱스를 생성한다.

### 도큐먼트

도큐먼트는 엘라스틱서치에서 데이터가 저장되는 기본 단위로 JSON 형태며, 하나의 도큐먼트는 여러 필드와 값을 갖는다.

```json
{
  "name": "mike",
  "age": 25,
  "gender": "male"
}
```

|MySQL|엘라스틱서치|
|---|---|
|테이블|인덱스|
|레코드|도큐먼트|
|컬럼|필드|
|스키마|매핑|

### 인덱스

인덱스는 도큐먼트를 저장하는 논리적 단위로, RDB의 테이블과 유사한 개념이다. 하나의 인덱스에 다수의 도큐먼트가 포함되는 구조이며, 동일한 인덱스에 있는 도큐먼트는 동일한 스키마를 갖는다. 또한 모든 도큐먼트는 반드시 하나의 인덱스에 포함돼야 한다.

> **인덱스의 관리 목적의 그룹핑**
> 
> 기본적으로 인덱스는 무한대의 도큐먼트를 포함할 수 있다. 하지만 인덱스가 커지면 검색 시 많은 도큐먼트를 참조해야 하기 때문에 성능이 나빠진다. 따라서 인덱스 용량 제한을 두게 한다. 기본적으로 특정 도큐먼트 개수에 도달하거나 특정 용량을 넘어서면 인덱스를 분리한다. 날짜/시간 단위로 인덱스를 분리하면 특정 날짜의 데이터를 쉽게 처리할 수 있다. 월별로 인덱스를 분리했다면 1개월 전의 인덱스만 삭제하면 된다.

## 도큐먼트 CRUD

### 인덱스 생성/확인/삭제

```js
PUT index1

GET index1

DELETE index1
```

### 도큐먼트 생성

```js
PUT index2/_doc2/1
{
  "name": "mike",
  "age": 25,
  "gender": "male"
}
```

상기를 요청하면 존재하지 않았던 index2라는 인덱스를 생성하면서 동시에 index2 인덱스에 도큐먼트를 인덱싱한다.

```js
GET index2
```

index2 인덱스는 mappings에 `age: long`, `gender, name: text` 타입으로 필드가 지정되었다. 데이터 타입을 지정하지 않아도 엘라스틱서치는 도큐먼트의 필드와 값을 보고 자동으로 지정한다.

`새로운 필드가 추가된 도큐먼트 인덱싱`
```js
PUT index2/_doc/2
{
  "name": "jane",
  "country": "france"
}
```

`데이터 타입이 잘못된 도큐먼트 인덱싱`
```js
PUT index2/_doc/3
{
  "name": "kim",
  "age": "20",
  "gender": "female"
}
```

상기의 경우 `age`를 `long` 타입으로 매핑했는데 텍스트 타입으로 입력했다. RDB라면 오류가 발생했겠지만 스키마에 유연하게 대응하는 엘라스틱서치는 타입을 변환해 저장한다.

### 도큐먼트 읽기

```js
// 도큐먼트 아이디를 사용해 조회
GET index2/_doc/1

// search의 DSL 쿼리를 이용해 도큐먼트 조회
GET index2/_search
```

### 도큐먼트 수정

도큐먼트 수정과 삭제는 개별 도큐먼트 삭제 및 수정 시 비용이 많이 들어가는 직업이다. 사용 시 주의해야 한다.

`1번 도큐먼트 업데이트`
```js
PUT index2/_doc/1
{
  "name": "kim",
  "age": 20,
  "gender": "female"
}
```

`update API를 사용한 도큐먼트 업데이트`
```js
POST index2/_update/1
{
  "doc": {
    "name": "lee"
  }
}
```

### 도큐먼트 삭제

```js
DELETE index2/_doc/2
```

## 응답 메시지

|코드|상태|해결 방법|
|---|---|---|
|200, 201|정상적으로 수행함||
|4xx|클라이언트 오류|클라이언트에서 문제점 수정|
|404|요청한 리소스가 없음|인덱스나 도큐먼트가 존재하는지 체크|
|405|요청 메소드를 지원하지 않음|API 사용법 다시 확인|
|429|busy|재선종, 노드 추가 조치|
|5xx|서버 오류|엘라스틱서치 로그 확인 후 조치|

## 벌크 데이터

REST API 콜 횟수를 줄여 성능을 높인다.

```js
POST _bulk
{"index": {"_index": "index2", "_id": "1"}}
{"name":"hong", "age": 10, "gender": "female"}
{"index": {"_index": "index2", "_id": "2"}}
{"name":"choi", "age": 90, "gender": "male"}
```

## 매핑

### 다이나믹 매핑

### 명시적 매핑

```js
PUT index3
{
  "mappings": {
    "properties": {
      "age": {"type": "short"},
      "name": {"type": "text"},
      "gender": {"type": "keyword"}
    }
  }
}
```

### 멀티 필드를 활용한 문자열 처리

**텍스트 타입**

**키워드 타입**

**멀티 필드**

멀티 필드는 단일 필드 입력에 대해 여러 하위 필드를 정의하는 기능으로, 이를 위해 fields라는 매핑 파라미터가 사용된다. fields는 하나의 필드를 여러 용도로 사용할 수 있게 만들어준다. 문자녈의 경우 전문 검색이 필요하면서 정렬도 필요한 경우가 있다. 또한 처음 데이터 스키마를 잡는 시점에서는 키워드 타입으로 충분히 처리가 가능한 범주형 데이터였지만 데이터가 늘어나면서 전문 검색이 필요해지는 경우도 생긴다. 이런 경우 텍스트와 키워드를 동시에 지원해야 한다.

```js
PUT multifield_index
{
  "mappings": {
    "properties": {
      "message": {"type": "text"},
      "contents": {
        "type": "text",
        "fields": {
          "keyword": {"type": "keyword"}
        }
      }
    }
  }
}
```

상기의 경우 `contents` 필드는 멀티 타입으로 설정했다. `contents` 필드는 텍스트 타입이면서 키워드 타입을 갖는다.

## 인덱스 템플릿

설정이 동일한 복수의 인덱스를 만들 때 사용한다.

`인덱스 템플릿 생성`
```js
PUT _index_template/test_template
{
  "index_patterns": ["test_*"],
  "priority": 1,
  "template": {
    "settings": {
      "number_of_shards": 3,
      "number_of_replicas": 1
    },
    "mappings": {
      "properties": {
        "name": {"type": "text"},
        "age": {"type": "short"},
        "gender": {"type": "keyword"}
      }
    }
  }
}
```

상기 적용 후 인덱스의 매핑을 확인해보면 다이내믹 매핑이 아닌 `test_template`에 정의되었던 매핑값이 적용된다.

만약 형변환이 불가능한 경우 `400` 에러와 함께 에러 이류를 뱉는다.

**템플릿 삭제**

```js
DELETE _index_template/test_template
```

**템플릿 우선순위**

### 다이내틱 템플릿

로그 시스템의 구조를 알지 못하기 때문에 필드 타입을 정확히 정의하기 힘들고, 필드 개수를 정할 수 없는 경우가 있다.

```js
PUT dynamic_index1
{
  "mappings": {
    "dynamic_templates": [
      {
        "my_string_fields": {
          "match_mapping_type": "string",
          "mapping"; {"type": "keyword"}
        }
      }
    ]
  }
}
```

`my_string_fields`는 임의로 정의한 다이내믹 템플릿의 이름이다. 다이내믹 템플릿 이름 밑으로 2개의 설정이 있다.

`match_mapping_type`은 조건문 혹은 매핑 트리거다. 여기서는 string 타입 데이터가 있으면 조건에 만족하여 `mapping`에 해당하는 실제 매핑을 적용한다.

```js
PUT dynamic_index1/_doc1/1
{
  "name": "mr. kim",
  "age": 40
}
```

상기의 경우, `name` 필드가 여기에 해당한다. `age` 필드는 사용자가 정의한 다이내믹 템플릿 조건에 만족하지 않아서 시스템의 기본 다이내믹 매핑에 의해 숫자 `long` 타입으로 매핑된다.

```js
PUT dynamic_index2
{
  "mappings": {
    "dynamic_templates": [
      {
        "my_long_fields": {
          "match": "long_*",
          "unmatch": "*_text",
          "mapping"; {"type": "long"}
        }
      }
    ]
  }
}
```

상기의 경우에는 `match`는 정규표현식을 이용해 필드명을 검사할 수 있다. match는 조건에 맞는 경우 mapping에 의해 필드들은 모두 숫자 타입을 갖는다. unmatch는 조건에 맞는 경우 mapping에서 제외한다.

## 분석기

### 구성

- 캐릭터 필터: 입력받은 문자열을 변경하거나 불필요한 문자들을 제거한다.
- 토크나이저: 문자열을 토큰으로 분리한다. 분리할 때 토큰의 순서나 시작, 끝 위치도 기록한다.
- 토큰 필터: 분리된 초큰들의 필터 작업을 한다. 대소문자 구분, 형태소 분석 등의 작업이 가능하다.

**역인덱싱**

**분석기 API**

```js
POST _analyze
{
  "analyzer": "stop",
  "text": "The 10 most loving dog breeds."
}
```

```js
{
  "tokens": [
    {
      "token": "most",
      "start_offset": 7,
      "end_offset": 11,
      "type": "word",
      "position": 1
    },
    {
      "token": "loving",
      "start_offset": 12,
      "end_offset": 18,
      "type": "word",
      "position": 2
    },
    {
      "token": "dog",
      "start_offset": 19,
      "end_offset": 22,
      "type": "word",
      "position": 3
    },
    {
      "token": "breeds",
      "start_offset": 23,
      "end_offset": 29,
      "type": "word",
      "position": 4
    }
  ]
}
```

상기 결과는 stop 분석기에 의해 `the`, `10` 등은 토큰으로 지정되지 않았다.

**분석기 종류**

standard: 기본값. 영문법을 기준으로 한 스탠다드 토크나이저와 소문자 변경 필터, 스톱 필터가 포함되어 있다.

simple: 문자만 토큰화한다. 공백, 숫자, -, ' 같은 문자는 토큰화하지 않는다.

whitespace: 공백을 기준으로 구분하여 토큰화한다.

stop: 불용어 제거. simple 분석기와 비슷하지만 스톱 필터가 포함되었다.

