---
layout: post
title:      "Using Handlebars with Rails: Part 1, setting up"
date:       2017-12-01 13:42:21 +0000
permalink:  using_handlebars_with_rails_part_1_setting_up
---


I had some trouble at first figuring out how to use Handlebars with Rails' ActiveModel::Serializer. Watching [this video](https://youtu.be/PT_C2211_QE) helped me get started. But beside from that, my googling efforts were little help to me. So from start to finish, here is what I ended up doing:

(note: I am using rails 5.1.3 here)

Onto our trusty Family Tree Builder.

the gems in question:

```
gem 'active_model_serializers'

gem 'handlebars_assets'
```

After running bundle install you generate your serializers in the console, ie "rails g serializer person"

This will create a file in the 'app/serializers' directory. Here is what my ActiveModel::Serializer file for my Person model looks like, after I have also generated serializers for city and state:

```
class PersonSerializer < ActiveModel::Serializer
  attributes :id, :name, :given_name, :year_of_birth, :year_of_death, :creator_id
  has_many :children
  has_many :parents
  has_many :spouses

  has_many :cities
  has_many :states
end
```

For creating an api response with these serialized objects, your controller looks like this:

```
  def show
    @person = Person.find(params[:id])
    respond_to do |format|
      format.html { render :show }
      format.json { render json: @person }
    end
  end
```

You can view this json yourself in your browser: For a person with id "212", for example, you can enter '/people/212.json' in your browser and see something like this:

```
{
id: 212,
name: "Draper",
given_name: "Don",
year_of_birth: 1926,
year_of_death: null,
creator_id: 1,
children: [
{
id: 215,
name: "Draper",
given_name: "Sally",
year_of_birth: 1954,
year_of_death: null,
creator_id: 1
}
],
parents: [
{
id: 213,
name: "Whitman",
given_name: "Abigail",
year_of_birth: null,
year_of_death: null,
creator_id: 1
},
{
id: 214,
name: "Whitman",
given_name: "Archibald",
year_of_birth: null,
year_of_death: null,
creator_id: 1
}
],
spouses: [
{
id: 216,
name: "Hofstadt",
given_name: "Betty",
year_of_birth: 1932,
year_of_death: 1969,
creator_id: 1
},
{
id: 237,
name: "Calvet",
given_name: "Megan",
year_of_birth: 1940,
year_of_death: null,
creator_id: 1
}
],
cities: [
{
id: 3,
name: "New York"
},
{
id: 14,
name: "Ossining"
},
{
id: 7,
name: "Philadelphia"
},
{
id: 4,
name: "Los Angeles"
}
]
}


```

As for Handlebars, here is the initial setup:

I created a folder for handlebars templates in my 'app/assets/javascripts' directory. 

so my application.js file looks like this:

```
//= require jquery
//= require jquery_ujs
//= require handlebars
//= require_tree ./templates
//= require_tree
```

In your folder app/assets/javascripts I used javascript files corresponding to my controllers, such as "people.js". If your Rails automatically generated coffeescript files for your ActiveRecord models, make sure to delete them or simply change their extension from .coffee to .js. If you have a file named both "people.js" and "people.coffee" rails will not compile the file with the .js extension.

In the next post, I will show how to create javascript model objects from a jquery response.




