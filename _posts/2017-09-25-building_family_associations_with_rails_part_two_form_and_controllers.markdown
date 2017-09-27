---
layout: post
title:  "Building Family Associations with Rails: part two, form and controllers"
date:   2017-09-25 21:12:34 -0400
---


Here is how I built my form/controllers for my family tree app:

My new path:

```
  def new
    @person = Person.new
  end

```

It is pretty basic. As is the form.

```
<%= form_for @person do |f| %>

  <%= f.text_field :given_name, :placeholder => "Given Name" %>
  <%= f.text_field :name, :placeholder => "Family Name" %>
  <%= f.text_field :year_of_birth, :placeholder => "Year of Birth (four digits)" %>
  <%= f.text_field :year_of_death, :placeholder => "Year of Death (four digits)" %>
  <%= f.text_area :comments, :placeholder => "Comments" %>
  <%= f.hidden_field :creator_id, :value => current_user.id %>
```

Then I added this form as a partial for the other associations, setting the value of each field to nil for a clean edit view.

 I built the unpersisted relation objects inside of the form rather than in the controller:

```
<%= f.fields_for :parents, @person.parents.build do |parent_fields| %>
  <%= render 'fields', :f => parent_fields %>
<% end %>
```

Etc. I also added a collection select for each association, for relating database items that have already been persisted:

```
<%= f.label :parent_ids, "Select Parents" %>
  <%= f.collection_check_boxes(:parent_ids, Person.alphabetize, :id, :display) do |b| %>
    <div><%= b.check_box + b.text %></div>
<% end %>
```

Here is how my nested attributes are accepted:

```
class Person < ApplicationRecord
  include Relationships
  include PersonDisplay::InstanceMethods
  extend PersonDisplay::ClassMethods
  extend Validation
	
	accepts_nested_attributes_for :parents, :spouses, :children, :reject_if => :reject_person_attributes?
	
  def reject_person_attributes?(attributes)
    !Person.attributes_are_valid?(attributes)
  end
	
end
```

```
module Validation

  def attributes_are_valid?(attributes)
    object = self.new(attributes)
    object.valid?
  end
	
end
```

I used the following validations on my Person class:

```
validates :name, :given_name, presence: true
validates_uniqueness_of :given_name, :scope => [:name, :year_of_birth, :year_of_death ]
validates :year_of_birth, allow_blank: true, numericality: {
                                            only_integer: true,
                                            greater_than_or_equal_to: 1400,
                                            less_than_or_equal_to: Date.today.year
                                          }
validates :year_of_death, allow_blank: true, numericality: {
                                            only_integer: true,
                                            greater_than_or_equal_to: 1400,
                                            less_than_or_equal_to: Date.today.year
                                          }
```

My strong parameters

```
def person_params

    params.require(:person).permit(:name, :given_name, :year_of_birth, :year_of_death, :comments, :creator_id, :parent_ids   => [], :child_ids => [], :spouse_ids => [], :city_ids => [], :parents_attributes => [:name, :given_name, :year_of_birth, :year_of_death, :comments, :creator_id], :children_attributes => [:name, :given_name, :year_of_birth, :year_of_death, :comments, :creator_id], :spouses_attributes => [:name, :given_name, :year_of_birth, :year_of_death, :comments, :creator_id], :cities_attributes => [:name, :state_id, :comments])
		
end
```

And finally, my create method:

```
class PeopleController < ApplicationController

  def create
    @person = Person.new(person_params)
    @person.persist_relationships
      respond_to do |format|
        if @person.save
          format.html { redirect_to @person, notice: 'Person was successfully created.'}
          format.json {render action: 'show', status: :created, location: @person}
        else
          format.html {render action: 'new'}
          format.json {render json: @person.errors, status: :unprocessable_entity}
        end
      end
  end
	
end
```

In part 3 of this series, I will explain this persist_relationships method, and other methods for establishing familial relationships
