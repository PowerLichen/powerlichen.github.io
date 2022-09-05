---
title : Django ORM의 참조에 사용하는 related_name
date : 2022-09-05 22:48:00 +0900
categories : [Backend, Django]
tags : [drf, django]
img_path: /assets/img/2022-09-05/
---

# 개요
Django를 사용하다 보면 연관된 객체를 가져와야 할 상황이 자주 일어난다.

예를 들어 다음과 같은 모델을 생각해 보자

> Post - (id, title, context)  
> Comment = (id, post, context, author)

위와 같은 구조에서 `Comment` 객체는 `Post` 객체를 ForeignKey로 가지고 있다.  
이를 통해서 `Comment`는 다음과 같이 Post에 정참조로 접근할 수 있다.

```python
instance = Comment.objects.get(id=1)
instance.post.title
```

그렇다면 `Post`가 자신을 참조하는 Comment들을 어떻게 가져올 수 있을까?

이 때 사용하는 것이 바로 `related_name`이다.

# related_name
`related_name`은 위와 같은 역참조 상황에서 사용할 수 있다.

개요에서 설명한 것 처럼 Post-Comment 관계를 생각해보자.  
여기서 Comment의 코드는 다음과 같다.

```python
class Comment(models.Model):
    ...
    post = models.ForeignKey(
        Post,
        on_delete=models.CASCADE,
        related_name="related_comments"
    )
    ...
```

역참조는 기본적으로 `<모델 이름 소문자>_set` 형태로 사용할 수 있다.  
하지만 ForeignKey의 옵션에 `related_name`을 포함하여 해당 명칭으로 참조할 수도 있다.

```python
instance = Post.objects.get(id=1)

# related_name 비선언
instance.comment_set.all()

# related_name을 related_comments으로 선언
instance.related_comments.all()
```

## 역참조에서 커스텀 매니저 사용
`related_name`을 통한 역참조에서는 해당 모델의 기본 매니저를 사용한다.  
이는 `manager` 옵션을 통해 다른 관리자를 사용할 수 있다.

```python
class Comment(models.Model):
    post = models.ForeignKey(Post, on_delete=models.CASCADE)
    ...
    objects = models.Manager()  # 기본 매니저
    active_objects = ActiveManager() # 커스텀 매니저

instance = Post.objects.get(id=1)
instance.comment_set(manager="active_objects").all()
```

## related_query_name
`related_query_name`도 ForeignKey에서 사용할 수 있는 옵션 중 하나이다.
이는 역참조 필터에서 사용할 이름으로, 기본 값은 `related_name`과 같다.

```python
# John이 댓글을 남긴 게시글만 필터링
Post.objects.filter(comment_set__author="John")
```
