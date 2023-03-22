# Antipattern: voyeuristic models

"Voyeuristic Models"는 Rails 프로그래밍에서 사용할 수 있는 디자인 패턴 중 하나입니다. 이 패턴은 모델이 자신의 데이터 뿐만 아니라 다른 모델의 데이터에도 책임을 지는 것을 의미합니다. 그러나 이 패턴은 코드를 복잡하고 유지보수하기 어렵게 만들 수 있으므로 일반적으로 권장되지 않습니다. 대신에, Rails 프로그래밍에서는 "단일 책임" 원칙을 따르는 것이 좋습니다. 이 원칙에 따르면 각 모델은 자신의 데이터만 관리할 책임이 있습니다. 이를 통해 유지보수하기 쉬운 고품질 코드를 작성할 수 있으며 이해하기 쉽고 작업하기 쉬운 코드를 만들 수 있습니다.

객체는 정보를 캡슐화하여 외부에서 직접적인 접근을 막고, 메서드를 통해 동작하도록 설계되어야 합니다. 이를 통해 객체의 데이터와 동작이 외부에서 보호되고, 객체 간의 상호작용이 단순화됩니다. 이는 객체 지향 프로그래밍에서의 핵심 개념 중 하나입니다. 객체 지향 프로그래밍에서는 객체들의 상호작용을 최소화하고, 개별 객체의 독립성을 보장함으로써 코드의 유지보수성과 재사용성을 높일 수 있습니다. 또한 객체 지향 프로그래밍에서는 객체들의 행위와 상태를 분리함으로써, 코드의 가독성과 유지보수성을 향상시킬 수 있습니다. 따라서 객체는 정보를 캡슐화하고, 메서드를 통해 동작하도록 설계되어야 합니다.

## Solution: Follow the Law of Demeter

[디미터 규칙](https://tecoble.techcourse.co.kr/post/2020-06-02-law-of-demeter/)을 적용합니다.

* 각 유닛은 다른 유닛에 대한 제한된 지식만 가지고 있어야 합니다. 즉, 현재 유닛과 "밀접한" 관련이 있는 유닛만 있어야 합니다.
* 각 유닛은 친구들과만 대화해야 합니다. 낯선 사람과 이야기하지 마십시오.
* 직계 class에게만 이야기하십시오.

기존코드

```ruby
class Address < ActiveRecord::Base
  belongs_to :customer
end

class Customer < ActiveRecord::Base
  has_one :address
  has_many :invoices
end

class Invoice < ActiveRecord::Base
  belongs_to :customer
end

<%= @invoice.customer.name %>
<%= @invoice.customer.address.street %>
<%= @invoice.customer.address.city %>,
<%= @invoice.customer.address.state %>
<%= @invoice.customer.address.zip_code %>

```

* 솔루션: Wrapper Model 사용

```ruby

class Address < ActiveRecord::Base
    belongs_to :customer
end

class Customer < ActiveRecord::Base
    has_one :address
    has_many :invoices

    def street   address.street   end
    def city     address.city     end
    def state    address.state    end
    def zip_code address.zip_code end
end

class Invoice < ActiveRecord::Base
    belongs_to :customer
    def customer_name     customer.name     end
    def customer_street   customer.street   end
    def customer_city     customer.city     end
    def customer_state    customer.state    end
    def customer_zip_code customer.zip_code end
end

```

* invoice모델에서 어드레스 값을 찾으려고 할때 Customer모델을 통해서 찾고 싶지 않다면 delegate를 사용해봅니다.

변경코드

```ruby
class Address < ActiveRecord::Base
  belongs_to :customer
  end

class Customer < ActiveRecord::Base
  has_one :address
  has_many :invoices
  delegate :street, :city, :state, :zip_code, to: :address
end

class Invoice < ActiveRecord::Base
  belongs_to :customer
  delegate :name,
          :street,
          :city,
          :state,
          :zip_code,
          to: :customer,
          prefix: true
end
```

위처럼 바꾸면 다음과 같이 사용할 수 있습니다.

```ruby
@invoice.customer_name
@invoice.customer_street
@invoice.customer_city
@invoice.customer_state
@invoice.customer_zip_code

```

## Solution: Push All find() calls into Finders on the Model

```html
<html>
  <body>
    <ul>
      <% User.find(order: 'last_name').each do |user| -%>
        <li><%= user.last_name %> <%= user.first_name %></li>
      <% end %>
    </ul>
  </body>
</html>

```

위의 page는 MVC를 침해 할 뿐 아니라 로직이 어플리케이션을 통해 매우 많이 중복될 수 있습니다. 이 이슈를 처리해보면

```ruby

class UsersController < ApplicationController
  def index
    @users = User.order('last_name')
  end
end

```
로 바꿔보면

```html

<html>
  <body>
    <ul>
      <% @users.each do |user| -%>
        <li><%= user.last_name %> <%= user.first_name %></li>
      <% end %>
    </ul>
  </body>
</html>
```

로 넣을 수 있죠.

rails에는 또한 scope라는 것도 사용합니다. Ruby on Rails 커뮤니티는 이 개념을 강력하게 수용하여 scope를 포함하여 프레임워크 자체에 적용했습니다. 나중에 scope 사용에 대해 살펴보겠지만 지금은 여기에서는 scope가 모델에서 메서드를 정의하기 위한 지름길이라고 말하는 것으로 충분합니다.

scope를 사용해 위의 내용을 정리해보면 다음 코드로 써볼 수 있습니다.

```ruby
class User < ActiveRecord::Base
  scope :ordered, order('last_name')
end
```

## Scope에서 다른 모델을 침범하여 find 하는 문제 처리

scope를 사용했을 때 다른 모델의 것까지 포함해서 scope를 사용한다면

```ruby
class User < ActiveRecord::Base
    has_many :memberships
    def find_recent_active_memberships
        memberships.where(:active => true).limit(5)
        . order("last_active_on DESC")
    end
end
```

위의 코드 처럼 될텐데요. 이경우 find_recent_active_memberships 는 너무 많이 active membership이 무엇인지 알고 있게 됩니다. User모델이 Membership모델에 대해 너무 많이 알고 있게 되는 것이죠. 이에 대한 대안은 귀찮더라도 자기자신이 찾게 하는 방법을 사용하는 것입니다.

## Solution: Keep Finders on Their Own Model

1. scope를 연관 모델로 옮긴다.

```ruby

# Alternative 1
class User < ActiveRecord::Base
    has_many :memberships
    def find_recent_active_memberships
        memberships.find_recently_active
    end
end

class Membership < ActiveRecord::Base
    belongs_to :user
    def self.find_recently_active
        where(:active => true).limit(5).order("last_active_on DESC")
    end
end

```

"AssociationProxy 매직"을 사용합니다. 이 magic은 연결된 레코드 간의 원활한 상호 작용을 허용하는 Rails의 ActiveRecord 연결 동작을 나타냅니다. "belongs to" 또는 "has many" 관계와 같은 두 모델 사이에 연관이 정의되면 Rails는 두 모델 사이에서 중개자 역할을 하는 "association proxy" 객체를 생성합니다. 이 프록시 개체를 사용하면 복잡한 SQL 쿼리를 작성하거나 데이터베이스 스키마의 세부 정보를 처리하지 않고도 연결된 레코드에 쉽게 액세스하고 조작할 수 있습니다.
