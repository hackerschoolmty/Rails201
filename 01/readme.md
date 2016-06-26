
## Table of Contents

* [ActiveRecord](#activerecord)
	* [Callbacks](###callbacks)
	* [save, save!, create, create!](#save-save-create-create)

* [Role management](#role-management) 
* [Exercises](https://github.com/hackerschoolmty/Rails201/blob/master/01/exercises.md)

## ActiveRecord

### Callbacks

Active Record controls the life cycle of model objects through callbacks. A normal life cycle of an object would consist on:

1. Create an object
2. Modified it
3. Updated it
4. And finally destroyed it 

Using callbacks, Active Record lets our code participate in this monitoring process by writing code that gets invoked **at any significant event in the life of an object**.  

Active Record defines sixteen [callbacks](http://guides.rubyonrails.org/active_record_callbacks.html#available-callbacks) and with these callbacks we can perform validations, map column values as they pass in and out of the database, and even prevent certain operations from completing, throughout the model life cycle.

In this example we're ensuring a downcase version of `title` attribute no matter how was set initially:

```ruby
class Post < ActiveRecord::Base
  before_validation :downcase_title
  
  def downcase_title
    self.title = title.downcase
  end
end
```

In this example we're printing a message in the console after an article is destroyed: 

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



## Role Management

Rails has several gems that can help us with role management. [Cancancan](https://github.com/CanCanCommunity/cancancan) is a great example of this

To install this gem you just need to add it to your application Gemfile

```ruby
gem 'cancancan', '~> 1.10'
```

And then run bundle install in your terminal:

```bash
bundle install
```

After that you need to create the `ability` class which CanCanCan uses to define user permissions, you can generate this class through the following command:

```ruby
rails g cancan:ability
```

An example of an `Ability` class looks like this:


```ruby
class Ability
  include CanCan::Ability

  def initialize(user)
    user ||= User.new # guest user (not logged in)
    if user.admin?
      can :manage, Product
    else
      can :read, Product
    end
  end
end
```

As you can see current user model is passed into the initialized method, so the permissions can be modified based on any user attributes. In the former example a user that is an admin can manage (*read, create, update, edit & delete*) a Product, and an user that is not an admin can only read (*have access to only index & show actions*) a Product



