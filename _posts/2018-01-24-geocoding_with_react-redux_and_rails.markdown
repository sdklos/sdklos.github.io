---
layout: post
title:      "Geocoding with React-Redux and Rails"
date:       2018-01-24 21:36:14 -0500
permalink:  geocoding_with_react-redux_and_rails
---

I created the bones of an app for adding, viewing, updating, deleting, and searching for lost and found pets. One of the features I implemented was a geolocating feature. In this blog I will Go over the basic setup for this.

My app uses a rails backend and a react frontend that manages state using redux. When I set up my rails api I chose not to use the stripped-down api version of the framework. This is rather a full rails app that happens to contain a react app as well. Also, rather than use the react-rails gem I have two apps that in development mode communicate from two different servers.

With that out of the way, let's look at the geocoding functionality:

On the rails end, I use the geocoder gem

```
*/Gemfile*


gem 'geocoder'
```

Then I create an Address model with the following structure:

```
*/app/db/schema.rb*

  create_table "addresses", force: :cascade do |t|
    t.float "latitude"
    t.float "longitude"
    t.datetime "created_at", null: false
    t.datetime "updated_at", null: false
    t.string "address"
  end
```
To set up the model with geocoding, here's what it looks like (the geocoder gem automatically looks for the latitude and longitude columns):

```
**
*/app/models/address.rb*

class Address < ApplicationRecord

  validates :address, presence: true
  has_many :pets

  geocoded_by :address
  after_validation :geocode, :if => :address_changed?

  reverse_geocoded_by :latitude, :longitude
  after_validation :reverse_geocode
	
end
```


I also create a Pet model with a foreign key for an address

```
create_table "pets", force: :cascade do |t| 
  t.integer "address_id"
end
```

and here's how I define the model to accept nested attributes for address:

```
*/app/models/pet.rb*

Class Pet < ApplicationRecord

  belongs_to :address, optional: true
	
	  def address_attributes=(address_attributes)
      address = Address.find_or_create_by(address_attributes)
      if address.save
        self.address = address
      end
  end
	
	
end
```
You'll have noticed I have not set up any validations yet for my address model. Ultimately I plan to do this on the client side, although right now I am just relying on the autocomplete to supply a valid address string. And on that note, let's check out the client side.

Right now my front end is living in */app/client*. 

For this side of the project I make use of the 'react-places-autocomplete' middleware; I also use material-ui components:

```
*console, client directory*

npm install react-places-autocomplete --save

npm install --save material-ui

```

I created a stateless component called SearchByLocation

```
*/app/client/src/components/SearchByLocation.js*

import React, { Component } from 'react';
import PlacesAutocomplete from 'react-places-autocomplete';

class SearchByLocation extends Component {

  onChange = (payload) => this.props.handleChange(payload)


  render() {
    const inputProps = {
      value: this.props.value,
      onChange: this.onChange,
      placeholder: this.props.placeholder,
    }

    const shouldFetchSuggestions = ({ value }) => value.length > 2

    const AutocompleteItem = ({formattedSuggestion}) => (
      <div>
        <strong>
          {formattedSuggestion.mainText}
        </strong>{' '}
        <small>{formattedSuggestion.secondaryText}</small>
      </div>
    )

    return(
      <div>
        <PlacesAutocomplete
          onSelect={this.onChange}
          renderSuggestion={AutocompleteItem}
          onEnterKeyDown={this.onChange}
          inputProps={inputProps}
          shouldFetchSuggestions={shouldFetchSuggestions} />
      </div>
    )
  }
}

export default SearchByLocation;
```
I use this component in two places: In my form, where pets are added to the database; and on my main page, where pets can be searched for by address.

Let's look at the form first. It is another stateless component:

```
*/app/client/src/components/PetForm.js*

import React, { Component } from 'react';
import SearchByLocation from '../components/SearchByLocation.js';
import RaisedButton from 'material-ui/RaisedButton';

export default class PetForm extends Component {

    handleSubmit = event => {
    event.preventDefault()
    const formState = Object.assign({}, this.props.formState)
    const pet = {pet: formState}
    pet['pet']['address_attributes'] = {}
    pet['pet']['address_attributes']['address'] = this.props.formState.address_string
    delete pet.pet.address_string
    this.props.submitPet(pet, pet.pet.id)
  }
	
```
My handleSubmit function goes through a lot of trouble to turn a formState into params that rails will accept. I am sure there is a better way to do this, but this is a 'hack' that I have devised for the time being.

More relevant parts of my form:

```
  handleAddressChange = address => {
    var addressChange = {}
    addressChange['address_string'] = address
    this.props.setFormState(addressChange)
  }
	
	render() {
	  return (
		
		 . . .
		 
      <div>
			   <SearchByLocation
           handleChange={this.handleAddressChange}
           value={this.props.formState.address_string}
           placeholder="Address" 
				 />
      </div>
			
			<div className="input">
          <RaisedButton
            type="button"
            onClick={this.handleSubmit.bind(this)}
            label="Submit Pet"
          />
        </div>
	
```
What are these props that are getting added to the PetForm? Let's look at the container:

```
*/app/client/src/containers/NewPetContainer.js*

import React, { Component } from 'react';
import PetForm from '../components/PetForm.js';
import { bindActionCreators } from 'redux';
import { addPet, setFormState, clearFormState } from '../actions/index';
import { connect } from 'react-redux';

componentDidMount(){
    this.props.clearFormState()
  }

  render() {
  return (
    <PetForm
      template="new"
      formState={this.props.formState}
      setFormState={this.props.setFormState}
      submitPet={this.props.addPet}
      breeds={this.props.breeds}
       />
  )}
}

const mapStateToProps = state => {
  return {
    formState: state.formState,
    breeds: state.breeds,
    isLoading: state.loading
  }
}
const mapDispatchToProps = (dispatch) => {
  return bindActionCreators({
    addPet: addPet,
    setFormState: setFormState,
    clearFormState: clearFormState
  }, dispatch);
};

export default connect(mapStateToProps, mapDispatchToProps)(NewPetContainer);
```
Let's take a look at what's going on in our actions and reducer:

```
*/app/client/src/actions/index.js*

import fetch from 'isomorphic-fetch';

export function addPet(pet, id) {
  return (dispatch) => {
    dispatch({ type: 'LOADING'})
    return fetch('/pets', {
      method: "POST",
      body: JSON.stringify(pet),
      headers: {
        'Accept': 'application/json',
        "Content-Type": "application/json"
      }
    }).then(response => {
      if (!response.ok) {throw response}
      return response.json()
    })
      .then(pet => {
      dispatch({type: 'SET_ACTIVE_PET', payload: pet })
      window.location.assign(`/pets/${pet.id}`)
    }).catch(error => {
      error.text().then(errorMessage => {
        console.log(errorMessage)
      })
    })
  }
}

export function setFormState(formState) {
  return {
    type: 'SET_FORM_STATE',
    payload: formState
  }
}

export function clearFormState() {
  return {
    type: 'CLEAR_FORM_STATE'
  }
}

```

And my reducers (right now all my reducers are in one file. Eventually I will split them):

```
*/app/client/src/reducers/managePets.js*

const formState = {
  id: null,
  status: '',
  address_string: '',
  name: '',
  pet_type: '',
  primary_breed: '',
  primary_color: '',
  age: '',
  contact_phone: ''
}

const initialState = {
  ...
	
  formState: formState

  ...
}

export default function managePets(state = initialState, action) {

  switch(action.type) {
	
	  case 'SET_FORM_STATE':
      const formChange = action.payload
      const newFormState = Object.assign({}, state.formState, formChange)
      const setFormState = Object.assign({}, state, {formState: newFormState})
      return setFormState
    case 'CLEAR_FORM_STATE':
      const clearedFormState = { id: null,
        status: '',
        address_string: '',
        name: '',
        pet_type: '',
        primary_breed: '',
        primary_color: '',
        age: '',
        contact_phone: ''}
      const stateWithClearedFormState = Object.assign({}, state, { formState: clearedFormState })
      return stateWithClearedFormState
	  default:
      return state;
	}
};
 
```

You'll notice my addPet calls a second argument, a pet id. Obviously, when creating a new pet this variable is null. I did this so that I could use the same submit function for my form whether in editing or creating mode. This is the same reason my location autocomplete in the form has a value of "address_string" while the parameters that are ultimately passed to rails are address_attributes.address . Just more hacks to get this thing to work.

As for this action "SET_ACTIVE_PET", this relates to rendering the pet after it has been created.

Here's what's happening on the controller side of this action:

```
*/app/config/routes.rb*

Rails.application.routes.draw do

  resources :addresses, defaults: {format: :json}
end

```
```
*/app/controllers/pets_controller.rb*

class PetsController < ApplicationController

  def create
    @pet = Pet.new(pet_params)
    if @pet.save
      render json: @pet
    else
      render json: @pet.errors, status: :unprocessable_entity
    end
  end
	
	private
	
	  def pet_params
      params.require(:pet).permit( ... :address_attributes => [:address])
    end
	
end	
	
```

Let's say I want to search for pets by address and radius. Here's what it looks like:

Here's a rather unwieldy PetsContainer that does just that:

```
*/app/client/src/containers/PetsContainer.js*

import React, { Component } from 'react';
import { bindActionCreators } from 'redux';
import { connect } from 'react-redux';
import { updateAddress, updateRadius, submitLocationQuery } from '../actions/index';
import SearchByLocation from '../components/SearchByLocation.js'
import RaisedButton from 'material-ui/RaisedButton';
import {Tabs, Tab} from 'material-ui/Tabs';

class PetsContainer extends Component {

  componentWillUpdate(nextProps) {
    this.filterPetsForDisplay(nextProps)
  }
	
	handleLocation = event => {
    this.props.submitLocationQuery(this.props.queryParams)
  }

  handleRadiusChange = event => {
    var radius = event.target.value
    this.props.updateRadius(radius)
  }
	
	filterPetsForDisplay = (props) => {
    [conditional logic related to filters, a topic I am not covering in this post]
      } else {
        const petsForDisplay = props.pets
        return petsForDisplay
      }
  }
	
	render() {
	
	  const petsForDisplay = this.filterPetsForDisplay(this.props)

      return (
			
			 . . .
			 
			 <Tabs>
			   <Tab label="Search By Location">
              <div>
                <TextField                  
                  name="radius"
                  floatingLabelText="Enter a radius in miles"
                  floatingLabelFixed={true}
                  value={this.props.queryParams.radius}
                  onChange={this.handleRadiusChange}
                />
              <SearchByLocation               
                handleChange={this.props.updateAddress}
                value={this.props.queryParams.address}
                placeholder="Enter an Address to Search"/>
            </div>
            <div>
              <RaisedButton label="Search By Location" onClick={this.handleLocation.bind(this)}/>
            </div>
          </Tab>
			</Tabs>
			
const mapStateToProps = state => {
  return {
    pets: state.pets,
    queryParams: state.queryParams
		
		. . .
  };
}

const mapDispatchToProps = (dispatch) => {
  return bindActionCreators({
    updateAddress: updateAddress,
    updateRadius: updateRadius,
    submitLocationQuery: submitLocationQuery,
    
		...
  }, dispatch);
};

export default connect(mapStateToProps, mapDispatchToProps)(PetsContainer);

```

The relevant actions:

```
*/app/client/src/actions/index.js*

  export function submitLocationQuery(queryParams) {
  return(dispatch) => {
    dispatch({ type: 'LOADING_PETS' });
    return fetch(`/pets/query?address=${queryParams.address}"&radius=${queryParams.radius}`)
    .then(response => response.json())
    .then(pets => {
    dispatch({type: 'QUERY_PETS', payload: pets })})
  };
}

export function updateAddress(address) {
  return {
    type: 'UPDATE_ADDRESS',
    payload: address
  }
}

export function updateRadius(radius) {
  return {
    type: 'UPDATE_RADIUS',
    payload: radius
  }
}


```

And reducers:

```
*/app/client/src/reducers/ManagePets.js*

const initialState = {
  ...
  queryParams: {
    address: '',
    radius: 15
  }
  ...
}

export default function managePets(state = initialState, action) {
  switch(action.type) {
    case 'QUERY_PETS':
      const filteredPetsState = Object.assign({}, state, {filtered_pets: action.payload})
      return filteredPetsState
    case 'UPDATE_ADDRESS':
      const addressQueryState = Object.assign({}, state.queryParams, {address: action.payload})
      const addressUpdateState = Object.assign({}, state, {queryParams: addressQueryState})
      return addressUpdateState
    case 'UPDATE_RADIUS':
      const radiusQueryState = Object.assign({}, state.queryParams, {radius: action.payload})
      const radiusUpdateState = Object.assign({}, state, {queryParams: radiusQueryState})
        return radiusUpdateState
				    default:
      return state;
	}

}


```

Here's how my controller is handling this action:

```
*/app/controllers/pets_controller.rb*

class PetsController < ApplicationController

  def query
    @pets = Pet.perform_query(params)
    render json: @pets, status: 200
  end
	
end

```

This perform query makes use of the geocoder middleware:


```
*/app/models/pet.rb*

Class Pet < ApplicationRecord

  belongs_to :address, optional: true
	
	  def self.perform_query(params)
      pets_with_addresses = Pet.all.select {|pet|
        pet.address != nil}
      @pets = pets_with_addresses.select {|pet| pet.address.is_near(params[:address], params[:radius])}
  end
	
	
end
```

this method calls a method on my Address model:


```
**
*/app/models/address.rb*

class Address < ApplicationRecord

  def is_near(address, radius)
    if self.latitude && self.longitude
      if self.distance_from(address) <= radius.to_i
        return true
      else
        return false
      end
    end
  end
	
end

That's about it!

