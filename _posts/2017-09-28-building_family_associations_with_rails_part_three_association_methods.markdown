---
layout: post
title:  "Building Family Associations with Rails: part three, association methods"
date:   2017-09-28 20:58:51 +0000
---

As mentioned in the previous post, there's a method I call in my People controller called persist_relationships:

```
def create
    @person = Person.new(person_params)
    respond_to do |format|
      if @person.save
        @person.persist_relationships
        format.html { redirect_to @person, notice: 'Person was successfully created.'}
        format.json {render action: 'show', status: :created, location: @person}
      else
        format.html {render action: 'new'}
        format.json {render json: @person.errors, status: :unprocessable_entity}
      end
    end
```

Inverse relationships are really hard to do on has_many, through: relationships. This is even more difficult when there's a self join involved, as in my case. So I wrote a method to create these inverses, as well as to associate children with spouses, etc. Here's what it looks like:

```
module Relationships

def persist_relationships
    #give all self's parents self as child
    parents = self.parents
    spouses = self.spouses
    children = self.children
    parents.each do |parent|
      unless parent.children.include?(self)
        parent.children << self
        parent.save
      end
    #give all self's parents other parents as spouse
      parent.spouses += parents
      parent.spouses.uniq
      parent.save
    end
    #give all self's spouses self as spouse
    spouses.each do |spouse|
      unless spouse.spouses.include?(self)
        spouse.spouses << self
        spouse.save
      end
    # give all self's spouses self's children as children
      spouse.children += children
      spouse.children.uniq
      spouse.save
    end
    #give all self's children self as parent
    children.each do |child|
      unless child.parents.include?(self)
        child.parents << self
        child.save
      end
    #give all self's children spouses as parent
      child.parents += spouses
      child.parents.uniq
      child.save
    end
  end
	
end

```

This is a rather large and unwieldy method. For refactoring purposes I may break it up some day.

I have other methods that I make use of on the show page, to indicate additional familial relationships:

```
def siblings
    siblings = []
    if self.parents
      a = self.parents.map {|parent| parent.id}
      Person.all.each do |person|
        person.parents.each do |parent|
          if a.include?(parent.id) && person != self
            siblings << person
          end
        end
      end
    end
    siblings.uniq
  end
```

```
def grandparents
    grandparents = []
    if self.parents
      self.parents.each do |parent|
        parent.parents.each do |grandparent|
          grandparents << grandparent
        end
      end
    end
    grandparents.uniq
  end
```

```
def grandchildren
    grandchildren = []
    if self.children
      self.children.each do |child|
        child.children.each do |grandchild|
          grandchildren << grandchild
        end
      end
    end
    grandchildren.uniq
  end
```

```
def aunts_and_uncles
    aunts_and_uncles = []
    if self.parents
      self.parents.each do |parent|
        if parent.siblings
          parent.siblings.each do |sibling|
            aunts_and_uncles << sibling
          end
        end
      end
    end
    aunts_and_uncles.uniq
  end
```

```
def nephews_and_nieces
    nephews_and_nieces = []
    if self.siblings
      self.siblings.each do |sibling|
        sibling.children.each do |child|
          nephews_and_nieces << child
        end
      end
    end
    nephews_and_nieces.uniq
  end
```

```
def cousins
    cousins = []
    if self.aunts_and_uncles
      self.aunts_and_uncles.each do |aunt_uncle|
        aunt_uncle.children.each do |cousin|
          cousins << cousin
        end
      end
    end
    cousins.uniq
  end
end
```

I have one final method that I use for spouses. I had a problem with my spouse join table, that I may have solved, that would lead to a person having themselves added as a spouse. This method may not be needed anymore, but until I have time to test it out, it's a hack I came up with for the show page:

```
module PersonDisplay

  module InstanceMethods
	
	  def spouses_for_display
      spouses_for_display = []
      safe_array = self.spouses
      spouses_for_display = safe_array.reject{|s| s == self}
    end
		
	end
		
end

```

