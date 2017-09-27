---
layout: post
title:  "Building Family Associations with Rails: part one, models"
date:   2017-09-24 18:05:28 -0400
---


I had quite a time building a family tree app. It's still not quite perfect, but it works pretty well. I couldn't find much online that addressed this, so hopefully this blog post will help other people.

The first thing I decided to do was build a main "persons" table and make use of self joins, along with two join tables: "child_parents" and "marriages". The join tables, in combination with the fact that all of these associations involved the "persons" table, made it impossible to set up inverse relationships. I ended up having to build these inverses myself (I'll talk about that in part 3.) Maybe there is a better way to set up these associations that can allow for inverses, but here is how I got it to work:

First I generated a resource "Person". The table looks like this:

```
create_table "people", force: :cascade do |t|
  t.string "given_name"
  t.string "name"
  t.integer "year_of_birth"
  t.integer "year_of_death"
  t.text "comments"
  t.integer "creator_id"
end
```

Then I generated two join models: ChildParent and Marriage. The tables looked like this:

```
create_table "child_parents", force: :cascade do |t|
  t.integer "person_id"
  t.integer "parent_id"
  t.integer "child_id"
end
	
create_table "marriages", force: :cascade do |t|
  t.integer "person_id"
  t.integer "spouse_id"
end
``` 


My models looked like this:

my main Person model:

```
class Person < ApplicationRecord

  has_many :child_parents, class_name: 'ChildParent', foreign_key: :person_id
  has_many :parents, through: :child_parents, class_name: 'Person', foreign_key: :child_id
  has_many :children, through: :child_parents, class_name: 'Person', foreign_key: :parent_id

  has_many :marriages, class_name: 'Marriage', foreign_key: :person_id
  has_many :spouses, through: :marriages, class_name: 'Person', foreign_key: :spouse_id

end

```

and my two join models:

```
class ChildParent < ApplicationRecord
  belongs_to :parent, optional: true, class_name: 'Person', foreign_key: :parent_id
  belongs_to :child, optional: true, class_name: 'Person', foreign_key: :child_id
  belongs_to :person, optional: true, class_name: 'Person', foreign_key: :person_id
end

class Marriage < ApplicationRecord
  belongs_to :spouse, optional: true, class_name: 'Person', foreign_key: :spouse_id
  belongs_to :person, optional: true, class_name: 'Person', foreign_key: :person_id
end

```


