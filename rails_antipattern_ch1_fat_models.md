# Antipattern: fat models

문제: 😣 목적과 달리 커져가는 모델

클래스를 바꾸기 위해서는 하나의 이유가 있어야 합니다. 이는 "단일 책임 원칙"의 일부이며, 소프트웨어의 유지보수성과 확장성을 높이는 중요한 개념 중 하나입니다. 즉, 클래스가 많은 책임을 가지면 코드가 복잡해지고 예측하기 어렵게 됩니다. 이를 해결하기 위해, 클래스는 가능한 한 자신의 책임에만 집중하고 다른 클래스의 책임에 대해서는 적극적으로 개입하지 않아야 합니다. 이렇게 하면 클래스 간의 결합도가 낮아지고 응집도가 높아져 유지보수성과 확장성이 향상됩니다.

"Fat Model"은 모델이 비지니스 로직, 데이터 접근, 뷰 로직등 여러가지 역할을 한꺼번에 수행하려고 모든 것을 다 집어 넣을 때 만들어 집니다. 이로 인해 복잡도가 높아지고 단위 테스트 작성도 어려워집니다.

만약, 온라인 스토어에서 주문하는 Order model을 작성해보려고 할때, 여러 state에 따라 order를 찾고, xml, json, pdf로 저장하는 기능도 가지고 있다고 봤을때, 다음과 같은 코드가 됩니다.

```ruby
# app/models/order.rb

class Order < ActiveRecord::Base
    def self.find_purchased
        # ...
    end

    def self.find_waiting_for_review
    end

    def self.find_waiting_for_sign_off
    end

    def self.advanced_search(fields, options = {})
    end

    def self.simple_search(terms)
    end

    def to_xml
    end

    def to_json
    end

    def to_csv
    end

    def to_pdf
    end
end
```

한번에 보면 보기 쉽고, 바로바로 만들기는 좋을지 몰라도 이 코드가 점차 비대해 지면서 관리와 가독성에서 문제가 생깁니다. 앞으로 이 코드를 다른 클래스들로 분산시켜서 어떻게 하면 관리를 좋게 할 것인지 고민해 보도록 합니다.

## Solution: 새로운 클래스에 책임을 전가 (위임:Delegate)

지난번 Solution에서 처럼 클래스의 method를 기능별로 나누면 가독성이 높아진다. Converation mothod에 집중해서 보도록합니다.

```ruby
# app/models/order.rb
class Order < ActiveRecord::Base
    def to_xml
    end

    def to_json
    end

    def to_csv
    end

    def to_pdf
    end
end
```

Order class에서 주문에 관련된 역할을 값을 계산, 주문 아이템들을 관리같은 것들이다. 위의 method들은 잘 맞지 않는 method들이다.

### Keepin' It Classy

로버트 세실 마틴은 SRP(The Single Responsibility Principle)을 주장했습니다. SRP의 규칙은 클래스가 변경되는 이유는 한가지 이상일 수 없다는 것입니다. SRP는 객체 지향 디자인의 기본 원칙으로 클래스에 여러 기능에 대한 책임이 있을 경우 하나의 책임을 변경하면 다른 기능에 영향을 미치고 코드를 관리유지 테스트 하기 힘들고 어렵습니다.

그런 관점에서 method들을 하나의 책임 아래로 묶어 놓아 봅니다.

```ruby
# app/models/order.rb
class Order < ActiveRecord::Base
    def converter
        OrderConverter.new(self)
    end
end

# app/models/order_converter.rb
class OrderConverter
    attr_reader :order
    def initialize(order)
        @order = order
    end

    def to_xml
    end

    def to_json
    end

    def to_csv
    end

    def to_pdf
    end
end
```

위처럼 코드를 변경하게 되면, pdf로 export하고자 한다면, `@order.converter.to_pdf` 라고 쓰면 되겠죠. 객체지향관점에서는 이를 composition이라고 합니다. Rails에서는 association method(`has_one, has_many, belongs_to`)를 사용해 자동으로 database baked model를 만듭니다.

### 디미터 규칙을 깨나요???? 아니?? 다른 방법이 있쥐..

이렇게 delegate를 하게 되면 연관 연관 연관 하게 되여 Voyeuristic model에서 나온 디미터 규칙을 깰수밖에 없는데, Rails에서는 다음과 같은 것을 지원합다.

```ruby
# app/models/order.rb
class Order < ActiveRecord::Base
    delegate :to_xml, :to_json, :to_csv, :to_pdf, :to => :converter
    def converter
        OrderConverter.new(self)
    end
end
```

이렇게 하면 `@order.to_pdf`를 그대로 사용할 수 있게 되는 것이죠.

### Crying All the way to the bank

위의 기술은 클래스를 분리하는 기본적인 기술인데, 이를 기반으로 좀더 다른 것들을 해볼 수 있습니다.

```ruby
# app/models/bank_account.rb
class BankAccount < ActiveRecord::Base
    validates :balance_in_cents, :presence => true
    validates :currency, :presence => true

    def balance_in_order_currency(currency)
    end

    def balance
        balance_in_cents / 100
    end

    def balance_equal?(other_bank_account)
        balance_in_cents == other_bank_account.balance_in_order_currency(currency)
    end
end
```

은행 계좌에서의 기능은 transfer, deposit, withdraw open, close등이 있습니다. 이 기능에 필요한 method로 balance in dallars, comparing the balances, converting the balance to another currency등이 있죠. 이것만으로도 너무 많은 일을 하게 됩니다. 일단은 나눠야 겠죠.
Rails에서는 이를 위해 `composed_of` method를 제공합니다.

`composed_of`는 3가지 main option을 받습니다. 새로운 객체를 참조할 method이름, 객체 클래스의 이름(`:class_name`), 매핑할 db의 컬럼 이름(`:mapping`)

```ruby
# app/models/bank_account.rb
class BankAccount < ActiveRecord::Base
    validates :balance_in_cents, :presence => true
    validates :currency, :presence => true

    composed_of :balance,
                :class_name => 'Money',
                :mapping => [%w(balance_in_cents amount_in_cent), %w(currency currency)]
end

# app/models/money.rb
class Money
    include Comparable
    attr_accessor :amount_in_cents :currency

    def initialize(amount_in_cents, currency)
        self.amount_in_cents = amount_in_cents
        self.currency = currency
    end

    def in_currency(other_currency)
        # currency exchange logic ...
    end

    def amount
        balance_in_cents / 100
    end

    def <=>(other_money)
        amount_in_cents == other_money.in_order_currency(currency).amount_in_cents
    end
end
```

`@bank_account.balance.in_currency(:usd)` 이런 방식으로 사용할 수 있습니다. 비교할 때는 `@bank_account.balance > @other_bank_account.balance`로 사용할 수 있습니다.

> ### 주의
>
> composed_of는 구성된 개체를 제자리에서 수정할 수 없습니다. 그렇게 하면 사용 중인 Rails 버전에 따라 상위 레코드를 저장할 때 변경 사항이 지속되지 않거나 고정 객체 예외가 발생합니다.

## Solution: 모듈의 사용

Order 객체를 다르게 분리 할 수 있는 방법을 생각해 봅니다.

```ruby
# app/models/order.rb

class Order < ActiveRecord::Base
    def self.find_purchased
        # ...
    end

    def self.find_waiting_for_review
    end

    def self.find_waiting_for_sign_off
    end

    def self.advanced_search(fields, options = {})
    end

    def self.simple_search(terms)
    end

    def to_xml
    end

    def to_json
    end

    def to_csv
    end

    def to_pdf
    end
end
```

### Divide and Conquer

```ruby
# app/models/order.rb
class Order < ActiveRecord::Base
    extend OrderStateFinders
    extend OrderSearchers
    include OrderExporters
end

# lib/order_state_finders.rb
module OrderStateFinders
    def find_purchased
        # ...
    end

    def find_waiting_for_review
    end

    def find_waiting_for_sign_off
    end

    def find_waiting_for_sign_on
    end
end

# lib/order_searchers.rb
module OrderSearchers
    def advanced_search(fields, options = {})
    end

    def simple_search(terms)
    end
end

# lib/order_exporters.rb
module OrderExporters
    def to_xml
    end

    def to_json
    end

    def to_csv
    end

    def to_pdf
    end
end
```

`include`와 `extend`는 다른 데, `include`는 instance method로 사용되는 것이고, `extend`는 class method로 사용됩니다.

### Move and Shake



### Solution: Reduce the Size of Large Transaction Blocks

Active Record는 저장 프로세스의 일부로 내장된 트랜잭션을 제공하기 때문에 큰 트랜잭션 블록은 컨트롤러에 있든 모델에 있든 종종 필요하지 않습니다. 이러한 기본 제공 트랜잭션은 모든 콜백 및 유효성 검사를 포함하여 전체 저장 프로세스를 자동으로 래핑합니다. 이러한 기본 제공 트랜잭션을 활용하면 코드 복잡성을 크게 줄일 수 있습니다.

```ruby
class Account < ActiveRecord::Base
    def create_account!(account_params, user_params)
        transaction do
            account = Account.create!(account_params)
            first_user = User.new(user_params)
            first_user.admin = true
            first_user.save!
            self.users << first_user
            account.save!
            Mailer.deliver_confirmation(first_user)
            return account
        end
    end
end
```

위의 코드는 `create_account!`안에서 많은 것을 다룹니다.

* 어카운트의 첫번째 유저생성
* 어드민 유저 만들기
* 어카운트에 사용자 추가하기
* 어카운트 저장하기
* 컴펌 메일 보내기

이를 한 트랜젝션에서 수행하게 되는 건 하나라도 실패하면 모두 revert하기 위해서죠. 이를 Rails에서는 validate와 다른 built-in 함수로 할 수 있습니다.

### Order up

validation callback의 순서는 다음과 같습니다.

```ruby
before_validation
before_validation :on => :create
after_validation
after_validation :on => :create
before_save
before_create
after_create
after_save
```

### 콘트롤러

`create_account!`를 호출 하는 콘트롤러는 다음과 같습니다.

```ruby
class AccountsController < ApplicationController
    def create
        @account = Account.create_account!(params[:account], params[:user])
        redirect_to @account,
            :notice => "Your account was successfully created"
    rescue
        render :action => :new
    end
end
```

Chap4에서 콘트롤러의 create 액션의 개선에 대해 이야기 하겠지만, 위의 코드는 이 컨트롤러에서 create가 성공한다면 어카운트 페이지로, 실패한다면 다시 생성하는 처음 패이지로 가는 코드가 됩니다.

이제 컨트롤러와 모델의 create함수가 다 준비 되었으니 refactoring을 해보려고 합니다.

### Nested Attributes

빌트인 함수인 save, create함수는 하나의 hash attribute값을 받습니다. Account모델에 대해 하나의 setter를 만들어야 한다는 얘기죠.
Rails에서는 `accepts_nested_attributes_for`를 제공합니다. 다음과 같이 사용하죠.

```ruby
accepts_nested_attributes_for :users
```

이를 사용하면 form도 이에 맞춰 바꾸게 됩니다.


```erb
<$= form_for(@account) do |form| -%>
    <%= form.label :name, "Account name" %>
    <%= form.text_field :name %>
    <% fields_for :user, User.new do |user_form| -%>
        <%= user_form.label :name, "User name" %>
        <%= user_form.text_field :name %>
        <%= user_form.label :email %>
        <%= user_form.text_field :email %>
        <%= user_form.label :password %>
        <%= user_form.password_field :password %>
    <% end %>
    <%= form.submit 'Create', :disable_with => "Please wait..." %>
<% end %>
```

이 코드를 보면, 어카운트 필드 안에 유저를 위한 필드가 있죠. users_attributes 라는 키를 가지고 있는 어카운트 필드들 안에 유저 필드를 위한 hash가 있는 것이죠.

### 어드민 유저 만들기

어드민으로 사용자를 생성하는 것은 다음과 같이 `before_create` 콜백에서 처리할 수 있습니다.

```ruby
class Account < ActiveRecord::Base
    before_create :make_admin_user

    private

    def make_admin_user
        self.users.first.admin = true
    end
end
```

### 컴펌 메일 보내기

save가 성공한 후 email을 보내게 되어있습니다. 생성에 성공했다는 콜백에서 email을 보내면 되겠죠.


```ruby
class Account < ActiveRecord::Base
    after_create :send_confirmation_email

    private

    def send_confirmation_email
        Mailer.confirmation(users.first).deliver
    end
end
```


### 모두 합쳐보기


```ruby
class Account < ActiveRecord::Base
    accepts_nested_attributes_for :users

    before_create :make_admin_user
    after_create :send_confirmation_email

    private

    def make_admin_user
        self.users.first.admin = true
    end

    def send_confirmation_email
        Mailer.confirmation(users.first).deliver
    end
end

class AccountsController < ApplicationController
    def create
        @account = Account.new params[:account]

        if @account.save
            flash[:notice] = "Your account was successfully created."
            redirect_to account_url(@account)
        else
            render :action => :new
        end
    end
end
```

이렇게 되면 콘트롤러는 매우 적게 커스텀으로 수정되며, 기본 제공되는 것과 비슷하게 되었죠. 큰 기능적 변경은 모델레벨에서 진행됩니다.
컨트롤러와 뷰에 대한 변경은 모델에 대한 변경을 용이하게 하기 위해 이루어졌습니다. 그러나 컨트롤러에 대한 변경 사항은 컨트롤러가 원래 버전에서 개선되었기 때문에 부수적인 이점입니다.

기본 제공하는 프로세스와 맞지 않는 경우들도 필연적으로 생길 수 있습니다. 이 때는 모델이 잘못 구성되어 있을 수 있죠. 작업중인 엔티티를 평가하고 record의 life cycle을 고민해봐야 할것같습니다.
