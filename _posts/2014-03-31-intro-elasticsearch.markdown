---
layout: post
title:  "Intro to ElasticSearch"
date:   2014-03-31 08:00:00
author: "<a href='http://austincherry.me'>Austin Cherry</a>"
summary: "The long journey, sleepless nights, and overwhelming bouts to learn the beauty of full text searching in ElasticSearch."
tags: searching, elasticsearch, ruby, rails
---

Searching. Agurably one of the most important jobs of an API. I don't just mean hum drum, sort of kinda, I threw a SQL query together searching. I mean powerful, relevent, maybe even beautiful searching. A search that returns the searcher that very thing they were looking for. Today, I would like to describe my journey and first plunge into the world of full text searching, with what I would say is a decently complex data set using ElasticSearch.

I assume if you stumbled upon this article, you probably already know a little bit about what Elastic Search is, so I won't bore you with those details. I would like to add a little disclaimer before we dig into the code. I am far from an ElasticSearch expert. That being said, you probably want to take any opinions I have formed, worth a grain of salt. There is a good chance they could change or be flat out wrong. My hope in this article can save anyone else new to ElasticSearch some time, by documenting some of the things I found through my adventure. That said, let's get started!

So like a lot of first time users, I choose to use Tire. Tire is an awesome Ruby library that can be used in your Rails app for working with Elastic Search. It provides a nice DSL and has built in support for ActiveRecord models. Do note that it was retired September of last year (2013) and recommends you use the ElasticSearch sponsored gem instead. There is also a suite of gems to provide similar set of features for ActiveModel/Record and Rails as Tire does. Links to these gems will be at the end of this article. I have plans in the future to document some of the differences using these as opposed to Tire. Stay tune for that. Even though it says it is not considered compatible with ElasticSearch 1.x, I set it up with the current version of 1.x and it worked fine. I choose Tire as there was a whole lot more documentation and examples, which I knew would be helpful being a first time user.

First things first. Let's actually get ElasticSearch installed. If your a cool, hipster, developer, you should probably be using homebrew. You can use homebrew to install ElasticSearch. Just run `brew install elasticsearch`. Once that is installed You can start it with `elasticsearch: elasticsearch -f -D es.config=/usr/local/Cellar/elasticsearch/0.90.0/config/elasticsearch.yml`. I personally added this to my Procfile and use foreman to start things up. If you need more information on installation, either check out the ElasticSearch docs or Railscast 306, which will go over some of the basics. You can find more on foreman in links at the bottom of this article as well. Once installed and running, we need to add our tire gem to our bundle file with `gem tire`. Run `bundle install` and now we should be ready to go.

Alright, let's do an example model we want to add to our ElasticSearch index. Since pretty much every rails app has it, let's do a user model.

```ruby

class User < ActiveRecord::Base
  include Tire::Model::Search
  include Tire::Model::Callbacks

  has_many :followers
  has_many :locations
  has_many :interests

  after_touch() { tire.update_index }

  mapping do
    indexes :id, type: 'integer'
    indexes :name, type: 'string', analyzer: 'snowball'
    indexes :title, type: 'string', analyzer: 'snowball'
    indexes :coordinates, type: 'geo_point'
    indexes :user_interests, type: 'string'
  end
end

```

That's a good chuck of code, so let's break it down. First thing we need to do is add our ActiveRecord class extensions. You can find more about what each of theses do in the Tire documentation. For now, just know we need to include them on a ActiveRecord model and they will give us some cool methods to create an index in ElasticSearch. Next, we add our model's associations, which is pretty uninteresting. I will include a sample schema for our model in a gist at the end of this article, to help clear up any ambiguity. After that we have the `after_touch` method. We need this to update our index when an associated object is updated or added. This means in the follower, location, and interest models we need to add `touch: true` to our `belongs_to` like so:

```ruby

class Location < ActiveRecord::Base
  belongs_to :user, touch: true
end

```

Now that our associations are kosher, let's look at this big block of mapping. Here is where we are actually telling ElasticSearch what kind of index we will look to build based on our model's table data. Keep in mind that ElasticSearch is a document based, schema free, full text engine. All that jargon means that when it searches it is in it's own data structure, which we refer to as an index. Only the things we add to the index will be able to search on. This probably makes you think why not just add the whole table to the index and call it a day? We normally only want to add the data we need indexed to our index, as the index data structure is designed for speed of accessing and finding documents, not space efficiency like SQL. In our code above we are adding the user's `id`, `name`, `title`, `coordinates` and `user_interests`. We set what the types are for each field. Notice that `name` and `title` are regular strings and have snowball analyzer. Analyzer tell ElasticSearch how to index the string. Simply put, they can build the string based on all sorts of different tokens and filters. Our snowball analyzer is a good example. It is built using a standard tokenizer, meaning it builds tokens (words) off most European languages. It also has a lowercase token filter to normalize all our tokens text into lowercase. You can even create custom filters, but we wouldn't get into that here. Check out ElasticSearch documentation for more information. Next up is our `coordinates` and `user_interests`. As you probably guessed these are not actual fields in our database, but actually methods in our User class.

```ruby

def coordinates
  array = []
  locations.each do |l|
    array << {lat: l.latitude, lon: l.longitude}
  end
  array
end

def user_interests
  array = []
  interests.map { |i| array << i[:category_id] }
  array
end

def to_indexed_json
  to_json(methods: ['coordinates', 'user_interests'])
end

```

This methods are pretty simple and take a user associated data and build an array we want to add to our index. Now that we have a way to build our index up, we can focus on the actual searching part.

```ruby

def self.search(params, user)
  array = [user.id]
  pending = []
  u_interests = []
  user.followers.map { |f| f[:pending] ? pending << f[:follower] : array << f[:follower] }
  user.interests.map { |i| u_interests << i[:category_id] }
  tire.search(page: params[:page], per_page: params[:per_page]) do
    query do
      boolean({ :minimum_number_should_match => 1 }) do
        should { match :user_interests, u_interests, {boost: 1}} if !u_interests.empty?
        should { match :id, pending, { boost: 100 } } if !pending.empty?
      end
    end
    filter :geo_distance, distance: "#{params[:distance]}mi", coordinates: "#{params[:latitude]},#{params[:longitude]}"
    filter :not, { :terms => { id: array } }
  end
end

```

Again, let's break this search method down. First notice it is a class method that takes params from our controller and a user object. First we have some processing to setup the data from the client the we want for searching the index. Once everything is setup the way we like it, we then call the tire.search method, passing in the paging information. Inside our tire search block, we create a query block to query the index with. In that query we setup a boolean search that expects at least a minimum match from our user_interests and pending to be successful. For both `user_interests` and pending we use the ElasticSearch multi-match to match our arrays against the arrays in the actual data set to see if things match up. If they do, we add a boost to move them up in the actual data set. We are just using the builtin best matcher, but there are some other builtin ones you can use as well as create custom ones. After the actual query, we then have filters. These basically allow us to filter down our data returned from the query. In our case we are filtering based on our geo location and filtering out our users who are already our user's followers. Already now our search is ready to go. On to the controller!

```ruby

class UsersController < ApplicationController
  def index
    @users = User.search(params, @current_user)
    render json: User.search_json(@users)
  end
end

```

The controller is pretty simple. Really just passing the params and current\_user on to the search method and letting it do it's thing. Something to note on the second line. The render is calling a `search_json` which is another class method I created on the user model. I did that to build some custom json response as I did not like the way the way ElasticSearch had it formatted. Let me also note a few snags I ran into with this. First of all, Active\_Model\_Serializers are a no go here. This is because the actual returned results from the Tire search are not an Active Relation object, but a special tire collection object. Also you can't override the `to_json` since we are using that for tire. There is actually a gem that extends Active\_Model\_Serializers to support that tire collection object, but it was not working with the latest version of tire when I was testing, so I opted for my own method.

With that, we should have a nice little intro to ElasticSearch. Again I am not an expert and if you have any feedback, feel free to get in touch and let me know how I can improve this article.

[ElasticSearch Overview](http://www.elasticsearch.org/overview/elasticsearch)

[Tire](https://github.com/karmi/retire)

[Elasticsearch-ruby](https://github.com/elasticsearch/elasticsearch-ruby)

[Elasticsearch-rails](https://github.com/elasticsearch/elasticsearch-rails)

[Railscast](http://railscasts.com/episodes/306-elasticsearch-part-1)

[ElasticSearch](http://www.elasticsearch.org/overview/elkdownloads/)

[Analyzers](http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/analysis-analyzers.html)

[Gist](https://gist.github.com/austiniam/9897280)

[Twitter](https://twitter.com/AC_Macalister)