
## Table of Contents

* [ActiveRecord](#activerecord)
	* [callbacks](###callbacks)
	* [save, save!, create, create!](#save-save-create-create)
* [i18n](#18n)
	* [Setup](#setup)
	* [Using I18n](#using-i18n)
	* [Advanced I18n](#advanced-i18n)
* [Role management](#role-management) 
* [Exercises](https://github.com/hackerschoolmty/Rails201/blob/master/01/exercises.md)

## ActiveRecord

### callbacks

Active Record controls the life cycle of model objects through callbacks. A normal life cycle of an object would consist on:

- create an object
- modified it
- updated it
- and finally destroyed it 

Using callbacks, Active Record lets our code participate in this monitoring process, by writing code that gets invoked at any significant event in the life of an object.  

Active Record defines sixteen [callbacks](http://guides.rubyonrails.org/active_record_callbacks.html#available-callbacks) and with these callbacks we can perform validations, map column values as they pass in and out of the database, and even prevent certain operations from completing, throughout the model life cycle.


```ruby
class Post < ActiveRecord::Base
  before_validation :downcase_title
  
  def downcase_title
    self.title = title.downcase
  end
end
```

```ruby 
class Article < ActiveRecord::Base
  after_destroy :log_destroy_action
 
  def log_destroy_action
    puts 'Article destroyed'
  end
end
```

### save, save!, create, create!

There are two versions of the save and create method, that differ in the way they report errors: 

* `save` returns true if the record was saved and `nil` otherwise.
* `save!` returns true if the save operation succeeded and it raises an exception otherwise.
* `create` returns the object regardless of whether it was successfully saved. If you want to determine wheter the data was actually written, you’ll need to check the object for validation errors
* `create!` returns the Active Record object on success and it raises an exception otherwise.

Active Record assumes `save` is called in the
context of a controller’s action method and that the view code will be presenting any errors back to the end user. And for many applications, that’s the normal behaviour! 
However, if we need to save a model object in a context where we want to make sure to handle all errors, we should use `save!`.


## I18n

I18n is a funny name, but it sure beats typing internationalization all the time. Internationalization, after all, starts with an *i*, ends with an *n*, and has eighteen letters in between.

### Setup Rails application for I18n

Rails will set up yor application with reasonable defaults, by adding all .rb and .yml files from the `config/locales/` directory to your translations load path, automatically. 

The I18n library will use *English* as a default locale. Go ahead and check out the `config/locales/en.yml` file, this one will contain a sample pair of translation strings: 


```ruby
en:
  hello: "Hello world"
```

If you use `t(:hello_world)` in some of your views:

```html
<h1><%= t(:hello_world) %></h1>
```

Rails will look inside the translations load path, pick the matching key and display the translation. 


But what about if I want my rails application to use english & spanish translations? If you need different settings, you can overwrite them easily, the first thing we must do is to create a configuration file that encapsulates our knowledge of what locales are available and which one is to be used as the default

```ruby
#encoding: utf-8I18n.default_locale = :enLANGUAGES = [	['English', 'en'],	["Espa&ntilde;ol".html_safe, 'es']]
```

The former initializer defines a list of associations between display names and locale names. To get Rails to pick up this configuration change, the server needs to be restarted. 


Since each page that is translated will have an "en" and "es" version, it makes sense to include this in the URL. Let's plan to put the locale up front, make it optional, and have it default to the current locale, which in turn will default to English. To accomplish this the first thing that we must do is to modify our `config/routes.rb` file

```ruby
Rails.application.routes.draw do 
  scope '(:locale)' do
  	resources :books
  end
end
```

What we have done is nested our resources and root declarations inside a scope declration for `:locale`. Furthermore, `:locale` is in parentheses, which is the way to say that it is optional. What this mean is that:

- `localhost:3000/books` will use the default locale (english)
- `localhost:3000/en/books` will also use the english default locale
- `localhost:3000/es/books` will also use the spanish default.

None of this modifications wil alter the controller nor the action, in each of the three cases the `index` action from the `Books` controller will be executed.

### Using I18n in our application

With the routing in place, we are ready to extract the locale from the parameters, and make it available to the application. To do this we just need to create a `before_action` callback and to set the `default_url_options`. The logical place to to both is in the common base class for all of our controllers, which is `ApplicationController`

```ruby
class ApplicationController < ActionController:Base
  before_action :set_i18n_locale_from_params
  
  #...
  
  protected
  
  def set_i18n_locale_from_params	if params[:locale]	  if I18n.available_locales.map(&:to_s).include?			(params[:locale])        I18n.locale = params[:locale] 	  else
 	    flash.now[:notice] ="#{params[:locale]} translation not available"
 	  end	end  end  def default_url_options
    { locale: I18n.locale }
  end
end

```
The `set_i18n_locale_from_params` does pretty much what it says: it sets the locale from the params, but only if there's a locale in the params, otherwise, it leaves the current locale alone. Care is taken to provide a message when there's a failure. 

And `default_url_options` also does pretty much what it says, in that it provides a hash of URL options that are to be considered as present whenever they aren't otherwise provided. In this case, we are providing a value for the `:locale`parameter. This is needed when a view on a page that does not have the locale specified attempts to construct a link to a page that does. 

With all of this settings with now can start to provide some translated text both in `config/locales/en.yml` and `config/locales/es.yml` files. 

So if we have the following locale files:

```
# en.yml
en:
  hello_world: "Hello world"
```

```
# es.yml
es:
  hello_world: "Hola mundo"
```

And the following view and routes

```html
# app/views/home/index.html.erb
<h1><%= t(:hello_world) %></h1>
```

```ruby
# routes.rb
root "home#index"
```
When we call `localhost:3000/en` a "Hello world" string between h3 tags will be display, otherwise if we call `localhost:3000/es` a "Hola mundo" string will show up

Please note:

- The translate key **must be unique**
- You can choose any name you like
- If you add new locale files **the server must be restarted** so Rails get to recognize the new configuraiton files.

### Advanced I18n

#### Passing variables to translations

You can use variables in the translations messages and pass their values from the view

```html
<h1><%= t(:hello_world, name: "John") %></h1>
```

```ruby
# config/locales/en.yml
en:
  hello_world: "Hello %{name}"
```

Given the previous configuration, when the server hits the corresponding view "Hello John" will be print.

#### Adding date/time formats

We can even format date & times through locales, because in rails the date/time localization exists. To localize the time format you can pass the Date/Time object to I18n.l or `l` helper for shorter options.

```html
<h1><%= t(:hello_world, name: "John") %></h1>
<p><%= l Time.now, format: :short %></p>
```


```ruby
# config/locales/en.yml
en:
  hello_world: "Hello %{name}"
  time:
    formats:
      short: "%d/%b"
  date:
    formats:
      short: "%b/%d"
```

That would give you a "Hello John" string between h3 tags and a date like "13/Feb". If you want to see the full list of the following flags you can use check the [documentation](http://apidock.com/ruby/DateTime/strftime) 

Also Rails contributors did a great job by translating Rails defaults for your locale, so go ahead and check [rails-18n](repository) for an archive of variety of various locale files. 

## Role Management

Rails has several gems that can help us with role management. [Cancancan](https://github.com/CanCanCommunity/cancancan) is a great example of this

