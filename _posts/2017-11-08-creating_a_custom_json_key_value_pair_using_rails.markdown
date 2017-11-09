---
layout: post
title:      "Creating a Custom JSON Key/Value Pair Using Rails"
date:       2017-11-09 02:22:04 +0000
permalink:  creating_a_custom_json_key_value_pair_using_rails
---


This is a quick demonstration of creating a custom attribute for Rails' very useful ActiveModel::Serializer implementation

The model in question is a Product, that has database columns name, price, inventory, and description. The inventory attribute is an integer. Let's say you want the model to return a string, either "Available" or "Sold Out" depending on this inventory value. There are a number of ways this can be done, if you want to include it in the object's JSON, you can do this:

```
class ProductSerializer < ActiveModel::Serializer
  attributes :id, :name, :description, :price, :inventory, :availability

  def availability
    if self.object.inventory == 0
      "Sold Out"
    else
      "Available"
    end
  end
end
```

The data can then be retrieved using jQuery

```
$.get("/products/" + id + ".json", function(data) {

  var availability = data["availability"]
	
});
```
