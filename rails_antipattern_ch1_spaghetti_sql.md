# Antipattern: Spaghetti SQL

ActiveRecord는 믿을 수 없을 정도로 강력한 프레임워크이며 많은 새로운 Rails 개발자가 이를 활용하지 못하고 있습니다. ActiveRecord 연결의 놀라운 유용성을 활용하지 않는 예를 봅시다.


```ruby
class PetsController < ApplicationController
    def show
        @pet = Pet.find(params[:id])
        @toys = Toy.where(:pet_id => @pet.id, :cute => true)
    end
end

```

Toy에서 찾는 것을 Pet콘트롤러에서 하는 잘못된 경우가 되는 것이죠. 이를 좀더 고쳐보면 다음과 같습니다.

```ruby
class PetsController < ApplicationController
    def show
        @pet = Pet.find(params[:id])
        @toys = Toy.find_cute_for_pet(@pet)
    end
end

class Toy < ActiveRecord::Base
    def self.find_cute_for_pet(pet)
        where(:pet_id => pet.id, :cute => true)
    end
end

```

## Solution: Use Your Active Record Associations and Finders Effectively

대부분 프로그래머의 실수는 모든 언어와 프레임워크의 기본 도구들을 완전히 사용하지 않는 다는 것입니다. Rails는 그런 기능이 많은데, 이를 얻으려면 배워야 하는 것이 많죠. Active Record Associations는 기능이 많습니다 위의 코드를 Active Record Associations를 이용해 refactoring해봅니다.

```ruby
class PetsController < ApplicationController
    def show
        @pet = Pet.find(params[:id])
        @toys = @pet.find_cute_toys
    end
end

class Pet < ActiveRecord::Base
    has_many :toys

    def find_cute_toys
        self.toys.where(:cute => true)
    end
end

```

`has_many`라는 기능을 통해, Pet#toys라는 Proxy Array를 얻을 수가 있습니다.

좀 찝찝한 것은 cute라는 컬럼의 값을 가지고 있는 것인데요. assocation에 있는 기능은 다음과 같이
어떤 선언에 블럭을 주고 기능을 확장해 나갈 수 있습니다.

```ruby
class PetsController < ApplicationController
    def show
        @pet = Pet.find(params[:id])
        @toys = @pet.toys.cute
    end
end

class Pet < ActiveRecord::Base
    has_many :toys do
        def cute
            where(:cute => true)
        end
    end
end

```

`pet.toys.cute`라고 호출하면 Rails는 이를 하나로 묶어서 호출합니다. 이는 하나의 SQL 호출로 만들어주며, scope와 같은 역할을 합니다.


Pet과 같이 Toy를 Assocation하여 부르는 Ower라는 모델이 있다고 하면 다음같이 선언하겠죠.

```ruby

class Owner < ActiveRecord::Base
    has_many :toys do
        def cute
            where(:cute => true)
        end
    end
end

```

이렇게 중복된다면 다음과 같이 변경하여 처리할 수 있습니다.


```ruby

module ToyAssocationMethods
    def cute
        where(:cute => true)
    end
end

class Pet < ActiveRecord::Base
    has_many :toys, :extend => ToyAssocationMethods
end


class Owner < ActiveRecord::Base
    has_many :toys, :extend => ToyAssocationMethods
end

```

이런 경우는 좀 특수해서 흔치 않겠지만, 복잡한 경우 유용하게 쓰일 것같습니다.

### 다시 정리

위의 코드들을 정리해서 다시 작성해 보면 다음과 같겠죠.

```ruby
class PetsController < ApplicationController
    def show
        @pet = Pet.find(params[:id])
        @toys = @pet.toys.cute
    end
end

class Toy < ActiveRecord::Base
    def self.cute
        where(:cute => true)
    end
end

class Pet < ActiveRecord::Base
    has_many :toys
end

```

scope기능을 사용해 다음과 같이 변경할 수 있습니다.

```ruby
class PetsController < ApplicationController
    def show
        @pet = Pet.find(params[:id])
        @toys = @pet.toys.cute.paginate(params[:page])
    end
end

class Toy < ActiveRecord::Base
    scope :cute, where(:cute => true)
end

class Pet < ActiveRecord::Base
    has_many :toys
end

```

그러면 하나의 SQL 호출로 toys를 불러낼 수 있겠죠.

앞에서 언급했듯이 연결 동작을 확장하는 처음 두 가지 방법 중 하나를 실제로 사용하고 싶을 때가 있습니다. 블록 및 :extend 형식을 모두 사용하면 finder 메서드 내에서 기본 프록시 개체의 세부 정보에 직접 액세스할 수 있습니다. [CollectionProxy](https://api.rubyonrails.org/classes/ActiveRecord/Associations/CollectionProxy.html) 참조


## Solution: Learn and Love the scope Method

복잡한 finder를 사용하기 위해 어떻게 해야 하는지 고민해봅시다.

```ruby
class RemoteProcess < ActiveRecord::Base
    def self.find_top_running_processes(limit = 5)
        find(:all,
            :conditions => "state = 'Running'",
            :order => "percent_cpu desc",
            :limit => limit)
    end

    def self.find_top_system_processes(limit = 5)
        find(:all,
        :conditions => "state = 'Running' and (owner in ('root', 'mysql')",
        :order => "percent_cpu desc",
        :limit => limit)
    end
end
```

평범한 finder 코드이지만, 계속해서 이런 코드가 생성된다면, 이런 remote process의 조합을 보게 될 듯합니다. 이를 scope를 통해 클린한 코드로 변경해 보려고 합니다.



```ruby
class RemoteProcess < ActiveRecord::Base
    scope :running, where(:state => 'Running')
    scope :system, where(:owner => ['root', 'mysql'])
    scope :sorted, order("percent_cpu desc")
    scope :top, lambda {|1| limit(1)}

end

RemoteProcess.running.sorted.top(10)
RemoteProcess.running.system.sorted.top(5)
```

코드 사이즈가 작아지는 것뿐 아니라, 그 자체로도 좋은 코드가 됩니다. 즉, 하나의 라인으로 하나의 SQL call을 할 수 있죠.
가독성 면에서, 사실 클래스 메쏘드로 만드는 것이 더 좋을 수 있습니다. 그리고 두 라인은 demeter법칙을 깨는 것같은 냄새가 나기도 합니다.

```ruby
class RemoteProcess < ActiveRecord::Base
    scope :running, where(:state => 'Running')
    scope :system, where(:owner => ['root', 'mysql'])
    scope :sorted, order("percent_cpu desc")
    scope :top, lambda {|1| limit(1)}
    def self.find_top_running_processes(limit = 5)
        running.sorted.top(limit)
    end

    def self.find_top_running_system_processes(limit = 5)
        running.system.sorted.top(limit)
    end
end

```

이렇게 하면 잘 작성된 함수안에서 scope를 사용한 것임을 알 수 있습니다.

이제 기본 스코프 사용법을 간단히 살펴봤으니 고급 검색 메서드 작성의 원래 문제로 돌아가 보겠습니다. 이 경우 온라인 음악 데이터베이스에 대한 고급 검색 구현을 만들어야 합니다. 이것은 고급 검색이 올바르게 작동하도록 하기 위해 건너뛰어야 하는 후프의 전형적인 예입니다. 다음 방법의 복잡성과 크기로 인해 확실한 유지 관리 문제가 발생했으며 이를 정리하는 방법을 찾아야 했습니다.

```ruby
class Song < ActiveRecord::Base
    def self.search(title, artist, genre,
                    published, order, limit, page)
        condition_values = {:title => "%#{title}%",
                            :artist => "%#{artist}%",
                            :genre => "%#{genre}%"}
        case order
        when "name":    order_clause = "name DESC"
        when "lenth":   order_clause = "duration DESC"
        when "genre":   order_clause = "genre DESC"
        else
            order_clause = "album DESC"
        end

        joins      = []
        conditions = []
        conditions << "(title LIKE ':title')" unless title.blank?
        conditions << "(artist LIKE ':artist')" unless artist.blank?
        conditions << "(genre LIKE ':genre')" unless genre.blank?

        unless published.blank?
            conditions << "(published_on == :true OR
                            published_on IS NOT NULL)"
        end

        find_opts = { :conditions => [conditions.join(" AND "),
                                    condition_values],
                      :joins => joins.join(' '),
                      :limit => limit,
                      :order => order_clause}

        page = 1 if page.blank?
        paginate(:all, find_opts.merge(:page => page,
                                        :per_page => 25))
    end
end

```

이런 condition에서의 조합을 scope 방법으로 풀어보면 다음과 같습니다.


```ruby
class Song < ActiveRecord::Base
    def self.top(number)
        limit(number)
    end

    def self.matching(column, value)
        where(["#{column} like ?", "%#{value}%"])
    end

    def self.published
        where("published_on is not null")
    end

    def self.order(col)
        sql = case col
            when "name": "name desc",
            when "length": "duration desc",
            when "genre": "genre desc",
            else "album desc"
            end
        order(sql)
    end

    def self.search(col)
        finder = matching(:title, title)
        finder = finder.matching(:artist, artist)
        finder = finder.matching(:genre, genre)
        finder = finder.published unless published.blank?
        return finder
    end
end

Song.search('fool', 'billy', "rock", true).
    order("length").
    top(10).
    paginate(:page => 1)
```

위의 구현은 노래 검색 작업을 결과 정렬, 제한 및 페이지 매기기 작업과 분리합니다. 이러한 작업은 서로 다르며 원래 ActiveRecord::Base#find의 제한 때문에 단일 메서드로만 구현되었습니다. 이제 이러한 메서드가 find 메서드에서 리팩토링되었으므로 애플리케이션 전체에서 재사용할 수 있습니다.

pagination은 MVC 원칙을 위반하는 것으로 보입니다. 콘트롤러는 언제 pagination이 적용될지 아닐지 책임을 가지고 있죠.

search, matching 메쏘드들은 내부에서만 사용되는 메쏘드들이고, 외부에서는 전혀 사용되지 않습니다. search 메쏘드 대신 where로 사용을 해도 될 것같죠.


```ruby
class Song < ActiveRecord::Base
    has_many :uploads
    has_many :users, :through => :uploads

    def self.search(title, genre, published)
        finder =        where(["title   like ?", "%#{title}%"])
        finder = finder.where(["artist  like ?", "%#{artist}%"])
        finder = finder.where(["genre like ?", "%#{genre}%"])
        unless published.blank?
            finder = finder.where("published_on is not null")
        end
        finder
    end
end

```

위처럼 하면, search메쏘드 안에서, matching, published song을 찾는 로직을 깔끔히 숨길 수 있죠. 이건 scope객체를 리턴하는 방식입니다.

Scope는 Ruby와 같은 언어의 유연성과 결합된 약간의 독창성으로 수행할 수 있는 작업을 보여줍니다. 복잡한 SQL 쿼리를 깨끗하고 아름다운 방식으로 표현하는 수단을 제공하므로 읽기 쉽고 유지 관리 가능한 코드를 작성할 수 있습니다.


## Solution: Use a Full-Text Search Engine

### Simplify Search


```ruby
# app/models/user.rb
class User < ActiveRecord::Base

end
```
