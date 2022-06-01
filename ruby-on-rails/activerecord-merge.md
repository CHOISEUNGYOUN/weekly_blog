# ActiveRecord#Merge 를 사용하여 모델간 범위 지정하기.

ActiveRecord를 몇년간 사용했지만 여진히 빠르게 변하고 있음을 체감하던 중 `ActiveRecord#merge`라는 새로운 기능을 발견하게 되었다. 이 기능은 이름과 기능 그자체에 대한 연관성이 모호한 이유로 ActiveRecord에서 가장 덜 사용되고 있는 기능이다. 하지만 이 기능은 모델간 조인(Join)할 시 단순히 범위를 합치는 기능이라는 점에서 연관성이 있다고 생각한다.

이를 이해하기 위해 우선 아래 두 모델의 관계와 범위를 살펴보자.

```rb
class Author < ActiveRecord::Base
  has_many :books
end
```

```rb
class Book < ActiveRecord::Base
  belongs_to :author

  scope :available, ->{ where(available: true) }
end
```

위 예시에서 두 테이블을 합쳐서 그중에서 책을 구할 수 있는(available) 작가들을 찾는 쿼리를 작성한다고 생각해보자. `ActiveRecord#merge`를 사용하지 않는다면 아래와 같이 사용할 수 있을것이다.

```rb
Author.joins(:books).where("books.available = ?", true)
```
```sql
SELECT "authors".* FROM "authors" INNER JOIN "books" ON "books"."author_id" = "authors"."id" WHERE "books"."available" = 't'
```

하지만 `ActiveRecord#merge`를 사용한다면 훨씬 더 명확하고 `available` 범위를 중복해서 작성할 필요가 없게 된다.
```rb
Author.joins(:books).merge(Book.available)
```
```sql
SELECT "authors".* FROM "authors" INNER JOIN "books" ON "books"."author_id" = "authors"."id" WHERE "books"."available" = 't'
```

위 결과에서 볼 수 있듯이 SQL 쿼리문은 정확하게 동일하게 생셩된다. `ActiveRecord#merge`는 모델에 선언된 범위를 활용하고 있기 때문에 코드 중복을 줄여주는 좋은 방법이다.

Original Source:
[Using Named Scopes Across Models with ActiveRecord#Merge](https://gorails.com/blog/activerecord-merge)
