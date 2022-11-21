---
title : DRF의 추가 기능 구현하기
date : 2022-11-21 23:47:00 +0900
categories : [Backend, Django]
tags : [drf, django]
img_path: /assets/img/2022-11-21/
---

# 개요
DRF에서 제공하는 ViewSet은 기본적인 CRUD 기능 대부분을 제공한다.
하지만 기능을 구현하다 보면 추가적인 기능을 필요로 한다.

프로젝트를 진행하면서 생긴 요청 사항 중 이런 기능이 있었다.
- 게시판에 다른 사람이 등록한 `옷 조합(이하 코디)`를 볼 수 있다.
- 이를 내 `코디` 리스트에 복사하고 싶다.
- 복사된 `코디`는 내 마음대로 수정할 수 있다.

이런 요청 사항을 따르려면, `코디`의 id 가져오는 것이 아니라 데이터 전체를 복사하는 구조가 필요하다.

# 추가 기능을 구현하기
## @action
DRF의 데코레이터 중에 `action`이라는 데코레이터가 있다.  
메소드에 `action` 데코레이터를 사용하면 해당 기능은 커스텀 기능으로서 작동한다.

필요한 패러미터는 다음과 같다.

- `methods`: 해당 action이 실행될 HTTP 메소드. 기본값은 GET.
- `detail`: 필수 값. 단일 호출과 목록 호출을 구분한다.  
True는 단일 호출 요청  
False는 목록 호출 요청  
- `url_path`: 해당 action을 사용하기 위한 URL 세그먼트. 기본값은 메소드 이름.  
*예시) /api/codi/1/__copy__/*
- `url_name`: Django 내부에서 사용할 이름. reverse를 통해 `URL Name`을 호출 시 사용된다.
- `kwargs`: 추가 설정 사항을 적용할 수 있다.  
ViewSet에 선언된 `*_classes` 내용들을 오버라이드 한다.

## 복사 기능 구현
복사 기능에 대한 간단한 구현은 다음과 같다.

복사 기능은 instance를 기반으로 생성 작업을 하기 때문에, 기존의 코드를 재사용하기 위해 partial_update를 사용하였다.

또한, 시리얼라이저의 `update` 메소드를 오버라이드 하여 복사 기능을 구현하였다.

```python
#serializers.py
class CodiDupSerializer(serializers.ModelSerializer):
    class Meta:
        model = Codi
        fields = ["id"]
    
    def update(self, instance, validated_data):
        instance.pk = None
        instance.save()
        return instance

#Views.py
@action(detail=True, methods=['post'], url_path='dup', serializer_class=CodiDupSerializer)
def dup_create(self, request, *args, **kwargs):
    super().partial_update(request, *args, **kwargs)
```