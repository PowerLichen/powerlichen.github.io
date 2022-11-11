---
title : DRF에서 복합키로 instance 찾기
date : 2022-11-11 23:47:00 +0900
categories : [Backend, Django]
tags : [drf, django]
img_path: /assets/img/2022-11-11/
image:
    path: composite-example.png
---

# 개요
Django Rest Framework의 API에서 DB의 데이터에 접근하는 메소드는 두 가지가 있다. `UPDATE`와 `DELETE`가 그 예시이다.
해당 메소드의 코드를 보면, 다음과 같이 데이터에 접근하는 것을 볼 수 있다.
```python
...
	instance = self.get_object()
...
```
처음부터 `get_object` 메소드를 호출하여 데이터를 불러오고 있다.
이 메소드를 간단하게 요약하자면 다음과 같다.

1. `get_queryset()`을 통해 전체 쿼리셋을 가져온다.
2. 요청된 request에 `pk`라는 이름으로 지정된 파라미터를 가져온다.
3. 위에서 가져온 `pk`를 통해 전체 쿼리셋을 필터링한다.
4. 필터링 후 나온 객체(object)를 반환한다.

하지만 개발을 하다보면 `기본키`로 데이터를 특정하는 것이 아닌, `복합키`를 이용할 때도 있다.
어떻게 하면 DRF에서 구현할 수 있을까?

# 구현
## 복합키 설정
DB를 사용할 때, 두 개 이상의 컬럼을 키로 지정하는 것은 자주있는 일이다. Django ORM에서도 해당 기능을 지원한다.

```python
class CoffeeBean(models.Model):
    origin = models.CharField(max_length=100)
    date = models.DateField('get_bean_date', auto_now_add=True)
    price = models.IntegerField()

    class Meta:
        constraints = [
            models.UniqueConstraint(
                fields= ['origin', 'date'],
                name = 'origin-date composite key'
            )
        ]
```

`models.UniqueConstraint`를 통해 복합키 설정을 지정할 수 있다.
예시에서는 `origin`,`date` 속성이 복합키로 지정되었다.

## View 수정하기
개요에서 데이터 접근은 `get_object`에서 이루어진다고 설명했다.
메소드의 상세 코드는 다음과 같다.

```python
def get_object(self):
    # 1. 전체 쿼리셋 가져오기
    queryset = self.filter_queryset(self.get_queryset())

    # 2. 기본적으로 지정된 필터링 키워드 가져오기.
    # drf의 APIView들의 기본값은 'pk'
    lookup_url_kwarg = self.lookup_url_kwarg or self.lookup_field

    assert lookup_url_kwarg in self.kwargs, (
        ...
    )
    # 3. 필터링 키워드로 dict 만들기 및 쿼리셋 필터링
    filter_kwargs = {self.lookup_field: self.kwargs[lookup_url_kwarg]}
    obj = get_object_or_404(queryset, **filter_kwargs)

    # May raise a permission denied
    self.check_object_permissions(self.request, obj)
    
    # 4. 결과 객체 반환
    return obj
```

코드에서 객체를 찾기 위한 filtering은 `lookup_field`를 통해 이루어진다. `filter_kwargs` 변수는 단일 값만 할당하고 있으므로 메소드 전체를 오버라이딩 하는 방법을 사용해야 한다.

위에 예시로 든 `CoffeeBean` 모델은 `origin` `date` `price`를 속성으로 가지며, 이중 `origin-date`가 복합 키이다.

그렇다면 get_object의 `filter_kwargs`를 다음과 같이 변경하면 된다는 것을 생각해볼 수 있다.

```
filter_kwargs = {
	origin = <Value>
    date = <Value>
}
```

여기서 `<Value>`에 들어갈 값은 API 요청의 param이나 query, body를 통해 받은 값이 될 것이다.

그렇다면 원하는 결과를 얻기 위해 변경해야 할 부분은 원본코드의 `lookup_url_kwarg` `filter_kwargs`가 될 것이다.
기본 코드를 그대로 가져온 뒤, 커스텀 해 보자

```python
# get_object 메소드 오버라이딩
def get_object(self):
    ...
    lookup_url_kwarg = ('origin', 'date')
    values = (self.request.data[kwarg] for kwarg in lookup_url_kwarg)

    filter_kwargs = dict(zip(lookup_url_kwarg, values))
    ...

    return obj
```

코드를 하나씩 설명하면 다음과 같다.

```python
lookup_url_kwarg = ('origin', 'date')
```
1. 변경할 키 값의 이름을 튜플로 선언한다. 

```python
values = (self.request.data[key] for key in lookup_url_kwarg)
```
2. 값을 튜플로 선언한다.`self.request.data[kwarg]` 부분은 해당 예시에서 request의 `body`를 통해 key를 받았기 때문에 사용되었다.

>`param`으로 값을 받았다면, `self.kwargs[key]`를 사용.
>`query`으로 값을 받았다면, `self.request.query_params('origin',None)`를 사용하도록 하자.

```python
filter_kwargs = dict(zip(lookup_url_kwarg, values))
```
3. python zip 함수를 이용해 key와 value를 딕셔너리로 변환하였다.



### 전체 코드

```python
def get_object(self):
    queryset = self.filter_queryset(self.get_queryset())
    lookup_url_kwarg = ('origin', 'date')
    values = (self.request.data[kwarg] for kwarg in lookup_url_kwarg)

    filter_kwargs = dict(zip(lookup_url_kwarg, values))
    obj = get_object_or_404(queryset, **filter_kwargs)

    self.check_object_permissions(self.request, obj)

    return obj
```
