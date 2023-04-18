# rails autosave

먼저 Rails ActiveRecord를 간단히 정리하고 Rails의 Autosave를 이야기 하고자 합니다.

## Rails ActiveRecord

Rails에서 ActiveRecord는 데이터베이스와의 상호작용을 위해 사용됩니다. 이것은 모델 인스턴스가 데이터베이스에 저장된 후에만 변경 사항이 저장된다는 것을 의미합니다. ActiveRecord는 매우 강력한 ORM(Object-Relational Mapping)으로, 데이터베이스와 객체 간의 변환을 쉽게 수행할 수 있게 해줍니다.

ActiveRecord는 모델과 연결된 테이블을 사용하여 데이터를 저장하고 검색합니다. 모델은 데이터베이스 테이블과 1:1로 매핑되며, 각각의 속성(column)은 모델의 속성으로 매핑됩니다. 예를 들어, 게시물(Post) 모델은 posts 테이블과 매핑되며, title, content 등의 속성은 각각 Post 모델의 title, content 속성과 매핑됩니다.

ActiveRecord에는 다양한 기능이 있습니다. 예를 들어, 검증(validation) 기능을 사용하여 모델이 데이터베이스에 저장되기 전에 모델이 유효한지 확인할 수 있습니다. 또한, 관계 설정을 통해 두 개 이상의 모델 간에 관계를 설정할 수 있으며, 이를 통해 객체 지향 프로그래밍의 장점을 활용할 수 있습니다.

ActiveRecord를 사용하면 SQL 질의문을 작성하지 않아도 데이터베이스와 상호작용할 수 있으며, ORM을 통해 객체 간의 관계를 쉽게 처리할 수 있습니다. 이러한 이유로 Rails에서 ActiveRecord는 매우 중요한 역할을 합니다.

## Autosave

Rails는 ActiveRecord를 사용하여 데이터베이스와 상호 작용합니다. 이것은 모델 인스턴스가 데이터베이스에 저장된 후에만 변경 사항이 저장된다는 것을 의미합니다. 이것은 때로는 불편할 수 있습니다. autosave 기능은 이 문제를 해결해줍니다.

autosave는 모델에서 변경 사항을 저장하는 방법입니다. 이것은 일반적으로 has_many와 같은 관계에서 사용됩니다. 예를 들어, 만약 게시물이 많은 댓글을 가지고 있다면, 댓글이 저장될 때마다 게시물을 저장하는 것이 좋습니다.

autosave를 사용하려면, 관련 모델에 autosave: true를 추가하면 됩니다. 예를 들어, 게시물 모델에서 댓글 모델을 has_many로 관계를 설정하려면 다음과 같이 하면 됩니다.

```ruby
class Post < ActiveRecord::Base
  has_many :comments, autosave: true
end
```

이제 댓글이 저장될 때마다 게시물도 자동으로 저장됩니다.

autosave는 새로운 모델 인스턴스를 만들 때도 사용할 수 있습니다. 예를 들어, 만약 게시물이 댓글을 가지고 있지 않은 경우 댓글을 만들 때 게시물도 자동으로 생성하려면 다음과 같이 하면 됩니다.

```ruby
class Comment < ActiveRecord::Base
  belongs_to :post, autosave: true
end
```

이제 댓글을 만들 때마다 게시물도 자동으로 생성됩니다.

## autosave 주의사항

물론 autosave가 항상 좋은 것은 아닙니다. 관련 모델이 많고 복잡한 경우에는 성능 문제가 발생할 수 있습니다. 따라서 autosave를 사용할 때는 주의해야 합니다.

Rails에서 autosave를 해제하려면, 관련 모델에 `autosave: false`를 추가하면 됩니다. 예를 들어, 게시물 모델에서 댓글 모델을 has_many로 관계를 설정하려면 다음과 같이 하면 됩니다.

```ruby
class Post < ActiveRecord::Base
  has_many :comments, autosave: false
end
```

### autosave가 저장되지 않는다면

Rails에서 autosave를 설정하지 않으면 모델 인스턴스가 데이터베이스에 저장된 후에만 변경 사항이 저장됩니다. 예를 들어, 게시물 모델에서 댓글 모델을 has_many로 관계를 설정하고, 댓글 모델에서 게시물 모델을 belongs_to로 관계를 설정한 경우, 댓글을 저장할 때는 게시물이 자동으로 저장되지 않습니다. 따라서, 댓글을 저장한 후에는 게시물을 수동으로 저장해야 합니다. 좀 불편할 수 있죠.


### 결론

* autosave: true 일 경우, save할때 연결된 모델의 데이터까지 같이 저장하게 됩니다.
* autosave: false 일 경우, save할때 연결된 모델의 데이터를 저장하지 않습니다.
* autosave가 명시되지 않는 경우는 새로 연결된 모델의 데이터는 저장하지만, 기존에 연결된 모델의 데이터는 저장하지 않습니다.

따라서 위의 상황을 모델 구조에 따라 잘 적용해야 할 것같습니다.
