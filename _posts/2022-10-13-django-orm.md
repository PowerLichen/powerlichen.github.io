---
title : Django ORM의 Q와 F에 대해서
date : 2022-10-13 21:32:00 +0900
categories : [Backend, Django]
tags : [drf, django]
img_path: /assets/img/2022-10-13/
---

# Q() objects
`django.db.models.Q`는 데이터베이스 관련 작업에 사용할 수 있는 SQL 조건문을 의미한다.
Q와 연산자를 통해 조건을 결합할 수 있다.

Django ORM에서 `filter`를 여러개 연결하는 것은 **AND** 조건을 계속 추가하는 것과 같다.
또한, `filter`의 여러 패러미터도 SQL 쿼리의 **AND**로 처리된다.

`Q()`는 여기서 더 나아가, `OR`, `NOT`, `XOR`을 처리할 수 있다.

```python
# question이 Who 또는 What으로 시작되도록 필터링
Q(question__startswith='Who') | Q(question__startswith='What')

# question이 Who로 시작되거나, pub_date 년도가 2005인 row만 필터링
Q(question__startswith='Who') | ~Q(pub_date__year=2005)
```

## Q를 활용한 복잡한 예시
검색이라는 기능을 예시로 들어보자.
검색 API는 패러미터로 다음과 같은 값을 받는다

- name: 작성자명
- keyword: 검색할 단어
- date_start: 시작 날짜
- date_end: 종료 날짜

위 네가지 값은 전부 들어올 수도 있고, 들어오지 않을 수도 있다.
이를 기본 filter만 사용해 나타낸 것은 다음과 같다.

```python
...
instance = Post.objects
if name:
    intance = instance.filter(user__username=name)
if keyword:
    intance = instance.filter(context__contains=keyword)
if date_start:
    intance = instance.filter(created__gte=date_start)
if date_end:
    intance = instance.filter(created__lte=date_end)
...
```

하지만 여기서 다음과 같은 조건식이 필요하다면 어떻게 해야할까?
(name & keyword) | (date_start & date_end)

```python
Qs = [Q()] * 4
if name:
    Qs[0] = Q(user__username=name)
if keyword:
    Qs[1] = Q(context__contains=keyword)
if date_start:
    Qs[2] = Q(created__gte=date_start)
if date_end:
    Qs[3] = Q(created__lte=date_end)

instance = Post.objects.filter(
    (Qs[0] & Qs[1]) | (Qs[2] & Qs[3])
)
```


# F() expressions
`django.db.models.F`는 모델의 필드, annotate된 열의 값을 의미한다.
파이썬으로 데이터를 가져오는 것이 아니라, 그 연산에 해당하는 쿼리를 만들어낸다.

다음과 같은 예를 생각해보자
```python
reporter = Reporters.objects.get(name='Tintin')
reporter.stories_filed += 1
reporter.save()
```

이 코드는 `name(pk)`이 Tintin인 행의 `stories_filed`를 1 증가시켰다.  
하지만 Django ORM에서는 이렇게 표현할 수 있다.

```python
reporter = Reporters.objects.get(name='Tintin')
reporter.stories_filed = F('stories_filed') + 1
reporter.save()
```
두번째 줄의 `F('stories_filed') + 1` 연산은 파이썬의 연산을 수행하는 것이 아니다.

Django가 F() 객체를 만나면 연산자 오버라이딩을 통해 SQL 쿼리를 수행한다.  
이 코드에서는 stories_filed를 단순히 1 증가시켰다.

여기서 중요한 점은, 데이터를 굳이 python 환경으로 가져오지 않았다는 점이다.  
만약 모든 `Reporters` 모델의 `stories_filed`를 1씩 증가시키려면, F를 이용한 다음과 같은 코드 한줄이면 충분하다.
```python
Reporter.objects.update(stories_filed=F('stories_filed') + 1)
```

## F()의 장점
첫 번째로, 데이터베이스에서 쿼리 처리를 통해 성능을 높일 수 있다.

두 번째로, 작업에 필요한 쿼리를 줄일 수 있다.

마지막으로, Race Condition 문제를 피할 수 있다.

Django는 데이터베이스 정보를 메모리로 가져와 처리한다.  
만약 여러 요청이 동시에 하나의 객체로 접근을 한다면 문제가 발생할 것이다.
`F()`는 이런 요청을 데이터베이스 단위로 처리하면서 문제를 해결할 수 있다.

## F() 사용 예시
### 쿼리 필터링
예를 들어 Blog-Entry 구조를 생각해보자.
여기서 작성자 이름이 블로그 이름과 같은 항목을 필터링 할 수 있다.
```python
Entry.objects.filter(authors__name=F('blog__name'))
```

또한, 날짜를 기반으로도 확인이 가능하다.
생성 후 마지막 수정 사이의 시간이 3일 이상인 항목을 필터링 할 수 있다.
```python
from datetime import timedelta
...
Entry.objects.filter(modified_date__gt=F('created_date') + timedelta(days=3))
```

### 어노테이션
다음과 같이 동적인 필드를 만들어 낼 수 있다.

```python
company = Company.objects.annotate(
    chairs_needed=F('num_employees') - F('num_chairs')
)
```