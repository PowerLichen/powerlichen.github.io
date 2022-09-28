---
title : DRF에서 논리삭제 구현
date : 2022-09-29 01:32:00 +0900
categories : [Backend, Django]
tags : [drf, django]
img_path: /assets/img/2022-09-29/
---

프로젝트 진행 중에 특이한 요구사항이 발생했다.  
Django ORM을 활용하면 손쉽게 데이터를 삭제할 수 있다.  
하지만 이런 데이터가 추후 데이터를 쌓고 분석하는데 사용하려면 사용자에게만 삭제된 것 처럼 보이는 추가 구현이 필요해졌다.

# 개요
## 물리삭제와 논리삭제
데이터베이스에서 데이터를 삭제하는 방법은 `물리삭제`와 `논리삭제`가 있다.

**물리삭제**는 SQL의 DELETE를 사용하여 DB 내의 데이터를 삭제하는 방법이다.

**논리삭제**는 SQL의 UPDATE를 사용하여 삭제 여부를 나타내는 속성(FLAG)을 변경하여 삭제된 것처럼 표현하는 것이다.

사용자가 탈퇴하는 상황의 경우, 일반적으로는 개인정보보호를 위해 물리적인 삭제를 진행할 것이다.

하지만 주문정보와 같은 사용자의 행동 데이터의 경우, 논리적으로 삭제한 뒤에 일정 주기마다 통계 데이터로 활용할 수 있을 것이다.

예시와 같이 논리 삭제는 실제 서비스를 비롯해 여러 부분에 쓰일 수 있지만, 기존에 구현한 로직에 변경이 필요하다.

## 목표
본 게시물에서 목표로 하는 구현은 다음과 같다.

- `커피` 모델은 하나의 `원두` 모델을 외래키로 가진다.
- `원두` 모델의 삭제는 `삭제 플래그`를 True로 바꾸는 것이다.
- `커피` 모델이 가리키는 `원두` 모델이 삭제될 경우, `커피` 모델은 변경되지 않는다.
다만, API 호출 시에는 SET_NULL과 유사하게 처리해야 한다.


# DRF에서의 구현 문제
기존 Django 코드에서 논리 삭제로 변경할 때 유의해야 할 점은 다음과 같다.

- 조회 시 삭제되지 않은 row만 조회가 되어야 한다.
- 삭제된 row를 참조하는 다른 테이블도 처리가 필요하다.

## 삭제 플래그 필터링 문제
### Django Manager
Django는 기본적으로 모든 Model에 objects라는 클래스 변수를 선언한다.

```python
class Comment(models.Model):
    ...
    objects = models.Manager()
```

여기 objects가 선언되어있기 때문에, instance를 호출할 수 있는 것이다.
또한, 기본 Manager의 `queryset`은 Model의 모든 데이터를 반환하도록 작성되어 있다.

그렇다면, 저 Manager를 원하는 방식대로 바꾼다면 우리가 원하는 답을 찾아낼 수 있다.

### Custom Manager를 이용한 구현
DEL_FL가 False인 row만 호출되도록 바꿔주자

```python
# models.py
class ActiveManager(models.Manager):
    def get_queryset(self):
        return super().get_queryset().filter(DEL_FL=False)

class Comment(models.Model):
    desc = models.CharField(max_length=100)
    DEL_FL = models.BooleanField(default=False)

    all_objects = models.Manager()
    objects = ActiveManger()
```

이렇게 작성하면 기존에 objects로 호출했던 모든 API들이 삭제된 인스턴스로 접근하는 것을 막을 수 있다.

여기서 all_objects를 따로 추가한 이유는 다음과 같다.
Manager 재정의 시, 해당 모델에서 `처음 정의된 Manager`가 Model의 default Manager가 된다.

모델과 관련된 작업을 하는 타 라이브러리의 경우, 모델명과 매니저를 제대로 알 수 없으므로 `Model._default_manager`를 사용하는데, 이때 우리가 임의로 만든 Manager가 Default가 된다면 문제가 생길 수 있다.

## 삭제된 테이블 참조 문제
### 외래키 문제점
다음과 같은 Model을 가정해보자

```python
class 원두(models.Model):
    성분 = ...
    출신지 = ...
    가격 = ...

class 커피(models.Model):
    원두 = ForeignKey(원두)
    시럽 = ...
    가격 = ...
```

커피는 원두를 외래키로 가지고 있다.
이때, ForeignKey를 사용할때는 일반적으로 on_delete라는 옵션을 사용한다. 해당 옵션에는 다음과 같은 값이 들어갈 수 있다.

- models.CASCADE: 외래키가 참조하는 값이 삭제될 때, 외래키를 포함하는 row도 삭제한다.
- models.SET_NULL: 외래키가 참조하는 값이 삭제될 때, 외래키 값을 null로 바꿔준다.

이외에도 여러 값이 존재하지만, 위와 같은 케이스만 고려해보자.

기존 코드의 경우, `원두`가 삭제되면 외래키를 가진 `커피`는 다음과 같은 작업을 한다.
- CASCADE 일 경우, 삭제된 `원두`를 외래키로 가진 `커피`도 삭제한다.
- SET_NULL 일 경우, 삭제된 `원두`를 외래키로 가진 `커피`의 외래키를 Null로 변경한다.

위 사항을 고려해보면, `related_name`를 사용한 역참조로 데이터를 수정, 삭제할 수 있다.

하지만 본 게시글의 목표에서는 `커피` 모델의 `원두` 참조값을 그대로 두면서, SET_NULL과 같은 형태로 API를 제공해야한다.
이에 대한 구현은 다음과 같다.

# 변형 논리삭제 구현하기
## Destroy 기능 수정
기존의 DELETE method를 다음과 같이 변경한다.

```python
# views.py
...
def destroy(self, request, *args, **kwargs):
    return super().update(request, *args, **kwargs)
...
```

DELETE 작업이 삭제플래그(DEL_FL)를 갱신하는 UPDATE 작업이 되었으므로, update를 호출하도록 오버라이딩 한다.

UpdateModelMixin의 update 메소드는 다음과 같다.

```python
def update(self, request, *args, **kwargs):
    partial = kwargs.pop('partial', False)
    instance = self.get_object()
    serializer = self.get_serializer(instance, data=request.data, partial=partial)
    serializer.is_valid(raise_exception=True)
    self.perform_update(serializer)

    if getattr(instance, '_prefetched_objects_cache', None):
        instance._prefetched_objects_cache = {}

    return Response(serializer.data)
```

update 메소드의 구조는 다음과 같다.
인스턴드 가져오기 -> 직렬화(Serializer) -> valid 검사 -> perform_update 호출

여기서 `perform_update`는 serializer의 save() 메소드를 호출한다.
`serializer의 save()` 메소드는 instance가 있을 때 `serializer의 update()`를 호출한다.

그렇다면, serializer의 update() 메소드를 커스텀하면 DEL_FL를 처리할 수 있다.

```python
from rest_framework.exceptions import NotAcceptable
...
class BasicDestroySerializer(serializers.Serializer):
    def update(self, instance, validated_data):
        if instance.DEL_FL == True:
            raise NotAcceptable
        instance.DEL_FL = True
        instance.save()
        return instance
```

커스텀한 update 메소드의 내용은 다음과 같다.
- 이미 삭제된 경우, NotAcceptable Exception을 발생시킨다.
- 아니라면 DEL_FL를 True로 변경한다. (True/False인 이유는 model에서 BooleanField로 선언했기 때문)
- 인스턴스의 save()를 호출하고 반환한다.

## 외래 키 처리
삭제된 row를 참조하는 테이블은 삭제되는 것이 아니라, 해당 부분이 null로 처리되어야 한다.

간단하게 떠올릴 수 있는 방법은 삭제된 테이블을 참조하는 부분을 전부 찾아 null로 UPDATE 하는 것이다.
하지만 해당 포스트에서 `논리 삭제 구현의 목적`은 삭제된 데이터 활용을 위해서였다.

이를 기반으로 떠올린 방법은 DRF의 SerializerMethodField를 사용하는 방법이었다.


```python
class CoffeeSerializer(serializers.ModelSerializer):
    원두 = serializers.SerializerMethodField()
    
    class Meta:
        model = 커피
        fields = ['원두', '시럽', '가격']
    
    class get_원두(self, obj):
        if obj.원두 == None:
            return None
        if obj.원두.DEL_FL == True:
            return None
        return obj.원두
```

이 Serializer는 GET 요청 시 사용한다.
`커피` 모델의 `원두`를 SerializerMethodField로 재정의 하고 메소드를 만든다.
SerializerMethodField는 기본적으로 get_<변수명>과 연결된다.
선언 시 method_name 옵션을 줄 경우 다른 메소드도 사용할 수 있지만, 기본적으로 주어지는 것을 쓰기로 했다.

메소드 내용은 다음과 같다.
- 존재하지 않는 원두를 참조할 경우, None을 반환한다.
- 존재하지만 삭제되어 DEL_FL이 True일 경우, None을 반환한다.
- 존재하는 원두라면 원두 오브젝트를 반환한다.

여기서 `get_원두` 메소드가 인자로 받은 obj는 모델(커피)의 instance를 의미한다.
그래서 obj.원두를 호출할 경우, 기존의 외래키로 연결된 `커피`의 `원두` 인스턴스를 반환한다.

위와 같이 구현할 시, 원두가 삭제된 경우 `null`을 반환하는 models.SET_NULL 작업을 유사하게 나타낼 수 있다.

