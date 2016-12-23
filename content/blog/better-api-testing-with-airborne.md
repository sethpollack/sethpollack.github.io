+++
title = "Better API testing with Airborne"
date = "2014-10-26"
categories = [
  "Frisby.js",
  "Jasmine",
  "Node.js",
  "Ruby",
  "Airborne.rb",
  "RSpec"
]
highlight = "true"
+++

At work we are in the process of rewriting our e-commerce site. We are redoing it in stages, starting with a node wrapper around our existing API's that are written in Classic ASP.  I was tasked with writing a test suite for the API's and came across a popular framework called [Frisby.js](http://frisbyjs.com/) which claims to make "testing API endpoints easy, fast, and fun".

<!--more-->

## Frisby Basics ##

Frisby is basically a wrapper around jasmine and request with some really useful custom matchers for testing JSON responses.

You start out with a `frisby.create` which initializes a new firsby object taking a describe string as a parameter which is followed by a method chain including an http verb with the url as a param, some expectations, and finally a `.toss()` method that kicks off the tests, giving you something that looks like this:.

```javascript
var frisby = require('frisby');
frisby.create('Get Brightbit Twitter feed')
  .get('https://api.twitter.com/1/statuses/user_timeline.json?screen_name=brightbit')
  .expectStatus(200)
  .expectHeaderContains('content-type', 'application/json')
  .expectJSON('0', {
    place: function(val) { expect(val).toMatchOrBeNull("Oklahoma City, OK"); }, // Custom matcher callback
    user: {
      verified: false,
      location: "Oklahoma City, OK",
      url: "http://brightb.it"
    }
  })
  .expectJSONTypes('0', {
    id_str: String,
    retweeted: Boolean,
    in_reply_to_screen_name: function(val) { expect(val).toBeTypeOrNull(String); }, // Custom matcher callback
    user: {
      verified: Boolean,
      location: String,
      url: String
    }
  })
.toss();

// from the frisby docs
```

## Shortcomings ##

Frisby provides two callback methods `afterJSON` and `after` which get called once the test is completed. For example if you want to create a cart, add an item, edit the cart, and clear the cart, you would need to use the `after` callback to start a new frisby chain:

```javascript
var frisby = require('frisby');

frisby.create('get new cart')
  .post(url + '/carts')
  //some some expectations
  .afterJSON(function(json) {
    var cartId = json.cartId;

    frisby.create('add item to cart')
      .post(url + '/carts/' + cartId, {
        "itemId": itemId,
        "quantity": qty
      }, {
        json: true
      })
      //some some expectations
      .after(function() {

        frisby.create('edit cart')
          .put(url + '/carts/' + cartId, {
            "itemId": itemId,
            "quantity": qty
          }, {
            json: true
          })
          //some expectations
          .after(function() {

            frisby.create('clear cart')
              .delete(url + '/carts/' + cartId + "/items/" + itemId)
              //some expectations
              .toss();
        }).toss();
    }).toss();
}).toss();
```

Which means that any complex tests requiring a series of API calls to setup a test are going to get messy real fast. It also means that decoupling setup code from test code is going to be a big challenge, ultimately ending up with an uncomfortable amount of duplicate code or needlessly running the same tests over.

In my code, I wrapped each step in its own function and passed a callback, like this:

```javascript
function testCart(){
  return createNewCart(function(cartId){
    return addItemToCart(cartId, {itemId: itemId, Qty: qty}, function(){
      return editCart(cartId,{itemId: itemId, Qty: qty}, function(){
        return clearCart(function(){});
      });
    });
  });
}
```

YES, <b>CALLBACK HELL!!!</b> (and frisby does not behave with promises).

Some of the frisby custom matchers (`expectJSONTypes` and `expectJSON` for example) take an optional path parameter so that you can specify the path of the object you are testing.

The path also includes helpers for testing arrays allowing you to provide a `?` to ensure that the array contains at least one object that matches the expectations, or a `*` to ensure that all objects in the array match the expectations.

However, frisby falls short here as well, and only supports `?` or `*`  for arrays at the end of the path.

After a lot of initial frustration I did eventually complete the project using Frisby. However, it got me thinking that there must be a better way. Having no real reason for using javascript for testing other than the irrelevant fact that we were wrapping the API's with node, I decided to see what the Ruby community had to offer (Also because I love writing Ruby and am always looking for excuses to write some Ruby code). Anyway to my dismay, I was unable to find a Ruby library with similar functionality, which is surprising considering the fanaticism the Ruby community has with testing.

## Airborne Is Born ##

Determined not to let something as trivial as a absent library stop us from coding our API test suites in Ruby we ([Alex Friedman](https://twitter.com/brooklynDev) and myself) set out to create our own library called [Airborne](https://github.com/brooklynDev/airborne).

The synchronous nature of Ruby made for a much nicer API, and allows users to write tests in RSpec as they normally would, while adding in functionality to assist with testing API's.

We were also able to magically make the response available Just by making an API call, we provide you with the following properties:

* `response` - The HTTP response returned from the request
* `headers` - A symbolized hash of the response headers returned by the request
* `body` - The raw HTTP body returned from the request
* `json_body` - A symbolized hash representation of the JSON returned by the request

A basic airborne test looks something like this:
```ruby
require 'airborne'

describe 'sample spec' do
  it 'should validate types' do
    get 'http://example.com/api/v1/simple_get' #json api that returns { "name" : "John Doe" }
    expect_json_types(name: :string)
  end

  it 'should validate values' do
    get 'http://example.com/api/v1/simple_get' #json api that returns { "name" : "John Doe" }
    expect_json(:name => 'John Doe')
  end
end

```

And the above test cart code now looks something like this:
```ruby
 describe 'cart' do

  it 'should create new cart' do
    post '/carts'
    cart_id = json_body[:cartId]
    # some expectations
  end

  it 'should add an item to the cart' do
    post "/carts/#{cart_id}", {itemId: item_id, quantity: qty}
    # some expectations
  end

  it 'should edit an item in the cart' do
    put "/carts/#{cart_id}", {itemId: item_id, quantity: qty}
    # some expectations
  end

  it 'should clear the cart' do
    get "/carts/#{cart_id}"
    json_body[:items].each {|item| delete "/carts/#{cart_id}/items/#{item[:id]}"}
    # some expectations
  end

end
```

 Airborne also provides many of other added features and improvements including:

* The ability to test against any live API, or agains an existing Rack application like Sinatra, Grape, or rails without actually having a server running.
* Fully functioning path, allowing you to test against arrays, nested or otherwise.
* Global setup for a base url and header.
* Full set of custom matchers, for testing against.
* Helpers for regex, dates,  and custom callbacks.


[See the docs for more info on using airborne](https://github.com/brooklynDev/airborne)
