---
title : Django에서 다중 업데이트 구현
date : 2022-08-22 17:35:00 +0900
categories : [Backend, Django]
tags : [drf, django]
img_path: /assets/img/2022-08-22/
---

# 개요
최근 작업을 진행하면서 동시에 여러 개체를 수정해야 하는 상황이 생겼다.  
프론트 개발자의 요청 상황은 다음과 같다.

- `post` 모델의 여러 행이 가진 태그를 동시에 수정하고 싶다.

특이한 기능이니까 `extra-action`을 사용해야 겠다는 점은 알 것 같다.  
하지만 어떻게 하면 구현할 수 있을까?

DRF의 `ListModelMixin`의 `list` 메소드는 다음과 같이 구현되어 있다.
```python
    def list(self, request, *args, **kwargs):
        ...
        if page is not None:
            serializer = self.get_serializer(page, many=True)
            return self.get_paginated_response(serializer.data)
        ...
```

여기서 보면 `many=True`를 통해서 무언가를 하는 듯 한데...  
이걸 이용할 수는 없을까?

# ListSerializer
ListSerializer는 한번에 여러 개체를 직렬화하기 위한 시리얼라이저이다.
해당 시리얼라이저는 직접 사용할 필요는 없다.
직렬화 시 `many=True` 옵션을 사용하여 이를 호출할 수 있다!
## ListSerializer 다중 수정 동작 커스터마이징
기본 문서에 따르면 `Serializer`와 `ListSerializer` 간의 구조는 다음과 같다.

```python
class CustomListSerializer(serializers.ListSerializer):
    ...

class CustomSerializer(serializers.Serializer):
    ...
    class Meta:
        list_serializer_class = CustomListSerializer
```
이제 `ListSerializer`의 `update` 메소드를 오버라이딩 하면 된다.

다만 여기서 주의할 사항이 몇 가지 있다.
- 인스턴스 리스트 중 update할 인스턴스를 어떻게 구별하는가?
- insert 문제: 존재하지 않는 인스턴스라면 어떻게 처리하는가?
- delete 문제: 수정 대상이 아닌 인스턴스는 어떻게 처리하는가?
- 한 인스턴스에 대한 요청이 여러개라면, 어떤 순서로 처리하는가?

기본 문서에서는 인스턴스 구별을 위해 기본 시리얼라이저에 명시적인 id 필드를 추가하였다.  
이를 반영한 코드는 다음과 같다.

```python
class BookListSerializer(serializers.ListSerializer):
    def update(self, instance, validated_data):
        # Maps for id->instance and id->data item.
        book_mapping = {book.id: book for book in instance}
        data_mapping = {item['id']: item for item in validated_data}

        # Perform creations and updates.
        ret = []
        for book_id, data in data_mapping.items():
            book = book_mapping.get(book_id, None)
            if book is None:
                ret.append(self.child.create(data))
            else:
                ret.append(self.child.update(book, data))

        # Perform deletions.
        for book_id, book in book_mapping.items():
            if book_id not in data_mapping:
                book.delete()

        return ret

class BookSerializer(serializers.Serializer):
    # We need to identify elements in the list using their primary key,
    # so use a writable field here, rather than the default which would be read-only.
    id = serializers.IntegerField()
    ...

    class Meta:
        list_serializer_class = BookListSerializer
```


# 요구 사항에 맞는 구현
개요에서 설명한 상황을 다시 생각해보자

> `post` 모델의 여러 행이 가진 태그를 동시에 수정하고 싶다.

우선 수정 대상 구분은 `ModelSerializer`의 `fields` 지정으로 해결할 수 있다.  
또한, 작성되지 않은 데이터에 대해서는 insert나 delete도 불필요하다.  
인스턴스의 요청이 여러개라면 모두 반영하는 것을 기본으로 한다.

위 사항을 고려하여 구현을 해 보았다.

```python
class PostListSerializer(serializers.ListSerializer):
    def update(self, instance, validated_data):
        data_mapping = {item['id']: item for item in validated_data}

        ret = list()
        for one in instance:
            if one.id not in data_mapping:
                continue
            ret.append(self.child.update(one, data_mapping[one.id]))

        return ret


class PostTagMultipleUpdateSerializer(serializers.ModelSerializer):
    class Meta:
        model = Post
        fields = ['id', 'tags']
        list_serializer_class = PostListSerializer
```

## 코드 설명
```python
data_mapping = {item['id']: item for item in validated_data}
```
`validated_data`는 요청에서 넘겨받은 데이터이다.  
즉, 변경 타겟에 대한 정보라 할 수 있다.

변경 대상 확인을 위해 `id`를 `key`로 가지는 딕셔너리로 선언했다.

```python
ret = list()
```
반환할 데이터 리스트를 선언했다.

```python
for one in instance:
    if one.id not in data_mapping:
        continue
```
인스턴스의 `id`가 `data_mapping`에 없다면 변경 대상이 아니다.  
continue 처리한다.

```python
ret.append(self.child.update(one, data_mapping[one.id]))
```
업데이트를 수행한다.

여기서 child는 `PostTagMultipleUpdateSerializer`를 의미한다.
해당 구조는 `many=True` 옵션에서 실행되는 클래스메소드인 `many_init` 메소드 코드를 보면 확인할 수 있다.

[DRF 공식 문서 참조](https://www.django-rest-framework.org/api-guide/serializers/#customizing-listserializer-initialization)