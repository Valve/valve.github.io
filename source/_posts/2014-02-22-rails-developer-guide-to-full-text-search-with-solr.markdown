---
layout: post
title: "Introduction to full text search for Rails developers"
date: 2014-02-22 15:05
comments: true
categories: [ruby, rails, full-text, solr, sunspot]
---

Every developer heard about full-text search. 

Most developers search with SQL and relational databases. 

Almost every developer knows somewhere deep that full-text search is better suited for searching text, but keeps using old `LIKE '%?%'` queries.

I've been one of those developers who never looked at full-text search, 
but I have changed and I invite other people to join me in this change 
and discover the other side of search with Solr.

<!--more--> 

This article assumes you're comfortable with Ruby, Rails and PostgreSQL. I'll build a simple _people near me_ application using Solr in small incremental steps and hopefully help readers to 
overcome the feeling of uncomfortable uneasiness when thinking about full text search technology. 

**A word of disclaimer:** _my goal here is to familiarize a reader with full-text searching, 
not create an ideal rails application structure.
I'll be using long views, JavaScript inside ERB templates and what not. 
The point is to make a small but complete application in a single article
and it is possible to do so only by keeping it really simple._

OK, engough talk, let's build the app!

Let's call this app `Neibo`: 

{%codeblock %}
➜  personal  ruby -v
ruby 2.0.0p353 (2013-11-22 revision 43784) [x86_64-darwin13.0.0]
➜  personal  rails -v
Rails 4.0.2
➜  personal  rails new neibo
      create  
      create  README.rdoc
      create  Rakefile
      create  config.ru
      create  .gitignore
      create  Gemfile
      create  app
      ...
{% endcodeblock %}

Let's remove the `sqlite3`, `turbolinks`, `coffee-rails`, `jbuilder` and `jquery-rails` gems, 
we will not need them. We should also add a `pg` gem to talk to Postgres DB.

My Gemfile now is: 

{% codeblock %}
source 'https://rubygems.org'

gem 'rails', '4.0.2'
gem 'pg'
gem 'sass-rails', '~> 4.0.0'
gem 'uglifier', '>= 1.3.0'

{% endcodeblock %}

Now you need to set the pg connection in `config/database.yml` file: 

{%codeblock lang:yaml %}
# config/database.yml
common: &common
  adapter: postgresql
  encoding: unicode
  pool: 5
  timeout: 5000
  min_messages: warning

development:
  <<: *common
  database: neibo

test:
  <<: *common
  database: neibo_test

production:
  <<: *common
  database: neibo_production
{% endcodeblock %}

Then let's create a `Person` model and generate a migration for it: 

{%codeblock lang:ruby %}

# app/models/person.rb
class Person < ActiveRecord::Base
end

{% endcodeblock %}

{%codeblock %}
➜  neibo  rails g migration create_people
      invoke  active_record
      create    db/migrate/20140222113048_create_people.rb
{% endcodeblock %}

{%codeblock lang:ruby %}
# db/migrate/20140222113048_create_people.rb 
class CreatePeople < ActiveRecord::Migration
  def change
    create_table :people do |t|
      t.string :name, null: false
      t.text :about, null: false
      t.string :likes
      t.string :dislikes
      t.float :lat, null: true
      t.float :lon, null: true

      t.timestamps
    end
  end
end
{% endcodeblock %}

So for every person we store a `name`, 
an `about` - this is where a person can tell the world about himself,
`likes` - stuff person likes and `dislikes`. 
We also want to store a person's location, so that other people could find him in certain radius.

We store a location using two [floating point](http://www.postgresql.org/docs/9.3/static/datatype-numeric.html#DATATYPE-FLOAT) numbers, `lat` - for latitude, and `lon` - for longitude. 
It's quite possible to use a
specialized [Point](http://www.postgresql.org/docs/current/static/datatype-geometric.html#AEN6547) data type, but I want to keep it simple here.

I make `lat` & `lon` attributes nullable in case a user
denies the browser geolocation permission and his profile is saved without those values.

Let's create the databases and run the migration.

{%codeblock%}
➜  neibo  rake db:create
➜  neibo  rake db:migrate
==  CreatePeople: migrating ===================================================
-- create_table(:people)
   -> 0.0080s
==  CreatePeople: migrated (0.0080s) ==========================================
{%endcodeblock %}

We now need to create a controller, a route and a view:

{%codeblock lang:ruby %}
# app/controllers/people_controller.rb
class PeopleController < ApplicationController
  def index
  end
end

{% endcodeblock %}

{% codeblock lang:ruby %}
# config/routes.rb
Neibo::Application.routes.draw do
  root to: 'people#index'
  resources :people
end
{% endcodeblock %}

`app/views/people/index.html.erb`
{% codeblock lang:erb%}
<% if flash[:alert].present? %>
  <h1 style="color:red"><%= flash[:alert] %>
<% end %>
<% if current_user %>
  <h2>Hello, <%= current_user.name %></h2>

  <h3>Search people near you: </h3>
  <%= form_tag people_path, method: :get do %>
    <div>
      <%= text_field_tag :search, params[:search], 
      placeholder: 'Search nearby people', type: :search, style: 'width: 400px' %>
      <label>Search within mile radius:
        <%= select_tag :radius, options_for_select([10, 50, 100]) %>
      </label>
    </div>
    <div><%= submit_tag 'Search' %></div>
  <% end %>
<% else %>
  <h2>Hello, guest</h2>
  <h3>
    Fill your profile so that people could find you. 
    Allow browser to access your location if you want to be found by people near you.
  </h3>

  <%= form_for @new_person do |f| %>
    <div> <%= f.text_field :name, placeholder: 'Enter your name' %> </div>
    <div> <%= f.text_area :about, placeholder: 'Tell about yourself' %> </div>
    <div> <%= f.text_field :likes, placeholder: 'Stuff you like' %> </div>
    <div> <%= f.text_field :dislikes, placeholder: 'Stuff you don\'t ' %> </div>
    <div> <%= f.submit 'Save' %> </div>
  <% end %>
<% end %>
{% endcodeblock %}

This UI is a two part thing: if a user has already filled his details, 
he can use the search form and search for people nearby. 
If this is a new user, he fills his details, optionally allows a browser to get his location and 
saves the profile in the database.

We now need to modify the `app/assets/javascripts/application.js` and remove the files we're not using. In my case I remove them all and leave the `application.js` empty.

The view code checks the `current_user` method to see if current user profile has been filled. 
Let's create this method:

{% codeblock lang:ruby %}
# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  protect_from_forgery with: :exception
  helper_method :current_user

  private

  def current_user
    @current_user ||= Person.find(session[:current_user_id]) if session[:current_user_id]
  end
end
{% endcodeblock%}

I'll be storing current user ID in session and get the information about 
the user from the database. 

OK, let's concentrate on the `new user` scenario. In order for the application to know about user's location, we need to grab it from browser and save. 

Adding the code to the view:

{% codeblock lang:html %}
  <h2>Hello, guest</h2>
  <h3>
    Fill your profile so that people could find you. 
    Allow browser to access your location if you want to be found by people near you.
  </h3>

  <%= form_for @new_person do |f| %>
    <div> <%= f.text_field :name, placeholder: 'Enter your name' %> </div>
    <div> <%= f.text_area :about, placeholder: 'Tell about yourself' %> </div>
    <div> <%= f.text_field :likes, placeholder: 'Stuff you like' %> </div>
    <div> <%= f.text_field :dislikes, placeholder: 'Stuff you don\'t ' %> </div>
    <%= f.hidden_field :lat %>
    <%= f.hidden_field :lon %>
    <div> <%= f.submit 'Save' %> </div>
  <% end %>
  <script>
    navigator.geolocation.getCurrentPosition(function(position) {   
      document.querySelector('#person_lat').value = position.coords.latitude;
      document.querySelector('#person_lon').value = position.coords.longitude;
    });
  </script>
{% endcodeblock %}

The lines we're interested in are:

{% codeblock lang:html %}
    <%= f.hidden_field :lat %>
    <%= f.hidden_field :lon %>
{% endcodeblock %}

And the JavaScript:
{% codeblock lang:javascript %}
  <script>
    navigator.geolocation.getCurrentPosition(function(position) {   
      document.querySelector('#person_lat').value = position.coords.latitude;
      document.querySelector('#person_lon').value = position.coords.longitude;
    });
  </script>
{% endcodeblock %}

When a view loads, JavaScripts asks a user for permission to get his location. If users agrees, the callback is called and the location is saved in hidden fields so that the form can
submit them back to the server.

Now the controller part: 

{% codeblock lang:ruby %}
# app/controllers/people_controller.rb
def index
  @new_person = Person.new
end

def create
  @new_person = Person.new(person_params)
  if @new_person.save
    session[:current_user_id] = @new_person.id
    redirect_to people_path
  else
    flash.now.alert = 'Please fill your profile'
    render :index
  end
end

private

def person_params
  params.require(:person).permit(:name, :about, :likes, :dislikes, :lat, :lon)
end
{% endcodeblock %}

Here we have a boilerplate ruby code, we're using strong parameterss to only allow a known
set of attributes. 
We then try to create a user and save the new user ID in the session.
This way `current_user` helper method will be able to get the current user back from the 
database. 
If the validation fails, we just display the message and render the view again.

Let's add those validations:

{% codeblock lang:ruby %}
# app/models/person.rb
class Person < ActiveRecord::Base
  validates :name, :about, :likes, :dislikes, presence: true
end
{% endcodeblock %}

Now when we go back to the browser and reload the page we can enter the profile data,
allow browser to get our geolocation and click save. 

So far so good. At this point we introduced ourselves to 
the system and `current_user.id` is stored in the encrypted cookie. 

Next part is where the fun starts: we need to be able to search for other users nearby.
We should be able to limit the search radius, specify the search term and see the results.

What's important is that we should not see people who have our search term in `dislikes` attribute. For example if a person dislikes Chinese quisine, and we're searching for people
who like it,... you get the idea.

Let's take a little detour and speak about the Solr and the gems that enable it on Rails.
We'll be using [sunspot](https://github.com/sunspot/sunspot) - an excellent gem that
adds a nice DSL (really, it's nice) on top of [rsolr](https://github.com/rsolr/rsolr). 

At this point you might be asking: _"Wait! What's RSolr? I'm now totally confused with Solr, RSolr and Sunspot and how they relate to each other"_. 
I totally understand your confusion. Let's break this mess into pieces:

1. Solr - a Java server that runs as a separate service and talks to the outside world 
via XML over HTTP API. It is generally considered a robust and full-featured, yet hard to learn
full-text search solution. The only way you can communicate to Solr from Rails application
directly is to send rather cryptic XML requests. 
2. Nobody wants to mess with raw XML over HTTP, so here enters RSolr - a wrapper around Solr HTTP API that allows interacting with Solr from Ruby code.
3. However RSolr is still rather low-level and does not provide any DSL or convenience methods
to define which Rails models should be searchable and how the indexes will be updated. 
The need for a new library was apparent, so the Sunspot was born. A really nice DSL that 
integrates directly into ActiveRecord models and allows to specify which attributes we need 
to index, how to transform and query the data. 

Now you're saying: "_I still don't understand, if the Solr is a Java service it means
I need to install and configure it on my system? That's a horrible perspective, get me out of this!_". Absolutely not. Sunspot gem is bundled with a development version of Solr and has a nice set of rake tasks to manage it. You can start, stop, reindex the data, all using rake tasks. There is no need to install Solr manually, all you need is to add two gems:

`sunspot_solr` and `sunspot_rails`. 

`sunspot_solr` is the pre-packaged development version of Solr and `sunspot_rails` 
is the Sunspot gem itself. So you need to make sure you place the `sunspot_solr` into `:development` group in your Gemfile.

OK, now that confusion is hopefully out of the way, let's continue with our people search 
scenario.

Let us define the searchable attributes on our `Person` model:

{% codeblock lang:ruby %}
class Person < ActiveRecord::Base
  validates :name, :about, :likes, :dislikes, presence: true

  searchable do 
    text :name, boost: 5.0
    text :about, :likes
    latlon(:location) { Sunspot::Util::Coordinates.new(lat, lon) }
  end
end
{% endcodeblock %}

Let's break it down piece by piece:

1. `searchable` block is a place where you define the full-text indexing behavior. 
Inside this block you can specify various rules describing which attributes should
be indexed, their pre-index transformations, facets, filters and so on.
2. `text :name` - person should be searchable by its name. By searchable I mean full-text searchable.
3. `boost: 5.0` - boost option tells Solr to prioritize the results found by this particular attribute. If you're searching for `John Doe`, all the people with such name will come first, and only after them those, who dislike Johns Doe (or John Does, I don't know which is correct).
4. `text :about, :likes` - person should be searchable by these attributes. 
5. `latlon(:location) { Sunspot::Util::Coordinates.new(lat, lon) }` - create a geo-spatial 
index on person's location using `lat` and `lon` attributes. This will allow to search for 
people within a certain mile radius.


Great, wasn't that simple? We've defined a set of searchable attributes on a `Person` model.
Now we're ready to actually search for people.

Let us add a `_person` partial where search result item will be displayed: 

{% codeblock lang:erb %}
<%# app/views/people/_person.html.erb %>
<div class="person">
 <h4><%= person.name %> </h4>
 <h5>About:</h5>
 <italic><%= person.about %></italic>
 <h5>Likes:</h5>
 <italic><%= person.likes %></italic>
 <h5>Dislikes:</h5>
 <italic><%= person.dislikes %></italic>
</div>
<hr/>
{% endcodeblock %}

We also need to add the iteration to the `index` view:

{% codeblock lang:erb %}
<%# app/views/people/index.html.erb %>
<% @people.each do |person| %>
  <%= render partial: 'person', locals: {person: person} %>
<% end %>
{% endcodeblock %}

So the view is ready, let's modify the controller code:

{% codeblock lang:ruby %}
# app/controllers/people_controller.rb
def index
  if current_user
    if params[:search].present? || params[:radius].present?
      search = Person.search do 
        fulltext params[:search]
        if current_user.has_location? 
          with(:location).in_radius(current_user.lat, current_user.lon, params[:radius])
        end
      end
      @people = search.results
    else
      @people = []
    end
  else
    @new_person = Person.new
  end
end
{% endcodeblock %}

On line `3` we check if current user is saved, on line `4` we verify we have something to
search by, either a search term or a radius. Then on lines `5 - 10` is where the actual 
full-text search happens. We use a `Model.search` method and pass it a block. 
Inside this block we need to specify the logic of the search.
In our case we call `fulltext` method and pass it our search term.

Let me be clear, we have two phases: **indexing** and **searching**. Indexing is defined
inside a model in a `searchable` block. You use `text` method to specify which attributes
should be full-text searchable.

Searching is done by calling `Model.search` method and passing it a block too. But this time
we call `fulltext` method to actually do full-text search on indexed attributes.

OK, we know know how to do full-text search on text attributes, we're already doing it on 
`name`, `about` and `likes` attributes. What we also need is a way to restrict the results
to a certain radius on a map. This is what lines `7 - 9` are for.

In out application it's possible that user denies a geolocation permission and his 
profile is saved without coordinates. So we need a convenience method to see
if current user has a location or not:

{% codeblock lang:ruby %}
# app/models/person.rb
def has_location?
  lat && lon
end
{% endcodeblock %}

This method is useful in `Person.search` block where we specify the search radius: 

{% codeblock lang:ruby %}
# app/controllers/people_controller.rb
if current_user.has_location? 
  with(:location).in_radius(current_user.lat, current_user.lon, params[:radius])
end
{% endcodeblock %}

We're using current user's `lat` & `lon` attributes and the radius from params to perform the 
filtering. You should remember to convert miles to kilimeters, because Sunspot operates on
kilometers.

OK, first version of the people search is ready to try, let's run it.

Works fine, but when I search for someone within 10 mile radius, I find myself too. 
There should be a way to search for _other_ people, excluding myself. Let's fix it.

Sunspot allows using attributes as filters. For this we should call methods like `integer`,
`string`, `datetime` etc. In this case we need to search for all people except those
with `:id` equal to the `:id` of current user. We also need to filter out the people with 
`dislikes` equal to the search term:

{% codeblock lang:ruby %}
# app/models/person.rb
searchable do 
  text :name, boost: 5.0
  text :about, :likes
  integer (:id)
  string(:dislikes)
  latlon(:location) { Sunspot::Util::Coordinates.new(lat, lon) }
end
{% endcodeblock %}

On line `5` we're creating an indexed filter on `:id` column, and on the next line a filter on
`:dislikes` column.

Now the filtering itself:

{% codeblock lang:ruby %}
# app/controllers/people_controller.rb
def index
  if current_user
    if params[:search].present? || params[:radius].present?
      search = Person.search do 
        without(:id, current_user.id)
        without(:dislikes, params[:search]) if params[:search].present?
        fulltext params[:search]
        if current_user.has_location? 
          with(:location).in_radius(current_user.lat, current_user.lon, params[:radius])
        end
      end
      @people = search.results
    else
      @people = []
    end
  else
    @new_person = Person.new
  end
end
{% endcodeblock %}

On line `6` we're filtering out people with `:id` equal to current user's id.
On line `7` we're filtering out people who dislike stuff I'm searching for.

What does `Person.search` return? It's a special Sunspot object that has a `results`
method. So to grab actual active record items, we use `@people = search.results` code.

Finally we have all pieces of the puzzle. If we run the app now we should be able to save 
current user's profile and then go search for other people.

In this article I've barely scratched the surface of the Solr & Sunspot capabilities. You should definitely look for more in the documentation if you want to create a full-featured
application.

### By why should I use fulltext search if I can do everything in SQL?

You're right, except you can't. 
Full text search is a huge topic with a huge set of capabilities. 
It can do synonyms search, wildcard search, stemming
and a [lot, lot more](https://wiki.apache.org/solr/AnalyzersTokenizersTokenFilters).

Solr can be as intelligent as to perform a word decomposition during search, operate on
word parts and generally behave as a human (almost).

Full-text search is faster too. How much faster? This is a tricky question, because it all
depends on the indexed data, but one can safely assume it can be at least several times 
faster than equivalent SQL searching. For complex searches Solr can be orders of magnitude
faster than SQL.

### How is new data indexed?

Sunspot handles it for you. It registers a set of hooks that trigger the automatic indexing
of updated and new records. If you look into rails log, you'll see something like:

{%codeblock %}
  SOLR Request (455.4ms)  [ path=update parameters={} ]
   (1.1ms)  COMMIT
Redirected to http://localhost:3000/people
  SOLR Request (60.9ms)  [ path=update parameters={} ]
{%endcodeblock %}

### How do I test it?

You should generally avoid touching Solr in unit tests. Either design your tests to avoid
talking to Solr in unit tests, or just stub Solr to return pre-canned results.

As for integration tests, indexing data before running them worked best for me. 
I first prepare some test data, then I reindex it with: 
`rake sunspot:reindex` 
and then run the integration tests.

If you find the topic of testing interesting, drop me a line, I'll cover it in the next article.


### Code

https://github.com/Valve/neibo

Well, I hope the explanation wasn't too packed, share your ideas in the comments :)
