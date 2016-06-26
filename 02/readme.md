## Table of Contents

* [Polymorphic associations](#polymorphic-associations)
* [Recursive associations](#recursive-associations) 
* [i18n](#18n)
	* [Setup](#setup)
	* [Using I18n](#using-i18n)
	* [Advanced I18n](#advanced-i18n)
* [Rspec & Rails](#rspec--rails)
	* [Why Rspec?](#why-rspec) 
	* [Testing Philosophy](#testing-philosophy)
	* [Setting up RSpec](#setting-up-rspec)
	* [Model Specs](#model-specs)
		* [Creating a model spec](#creating-a-model-spec)
		* [Testing non ideal results](#testing-non-ideal-results)
		* [DRYer specs with describe and context](#dryer-specs-with-describe-and-context) 
	* [Factories](#factories)
	* [Generating more realistic fake data](#generating-more-realistic-fake-data)
* [Exercises](https://github.com/hackerschoolmty/Rails201/blob/master/02/exercises.md)

## Polymorphic associations

Active Record's `has_many` and `belongs_to` associations work really well when the two sides of the relationship have fixed classes: 

- An `Author`can have many `Books`. 
- A `Library` can have many `Books`, etc. 

**But sometimes you may want to use one table and model to represent something that can be associated with many types of entities.**

For example let's say that I want to register addresses for People & Companies, how should I model that? The first idea that came into my mind is create two different models one for People (`AddressPerson`) and another one for Companies (`AddressCompany`), but that's way to repetitive! This is kind of scenario when Polymorphic associations come in handy.

The name may be daunting (Polymorphic uhhhhhh :ghost:), but there's really nothing to fear. Let's work through the previous example: 


First we'll need to create migrations for Person, Company & Address model: 

```ruby
class CreatePeople < ActiveRecord::Migration  def change
    create_table :people do |t|
      t.string :name
      t.timestamps    end
  endend

class CreateCompanies < ActiveRecord::Migration  def change
    create_table :companies do |t|
      t.string :name
      t.timestamps    end
  endend

class CreateAddresses < ActiveRecord::Migration  def change
    create_table :addresses do |t|
      t.string :street
      t.string :city
      t.string :state
      t.integer :addressable_id
      t.integer :addessable_type
      t.timestamps    end
  endend
```

I guess you have notice some things unusual about the `CreateAddresses` migration, why `addressable_id` and `addressable_type` show up? What are those? Well in order to declare a polymorphic association we need to departure from the usual ActiveRecord convention: 

- First the name of the foreign key is neither `people_id` nor `company_id`, it's called `addressable_id` instead. 
- Second, we need to add a column called `addressable_type`, you'll see in a moment how we're going to use these columns.

Now let's generate models for Person, Company and Address. 

```ruby
class Person < ActiveRecord::Base
  has_many :addresses, as: :addressable
end

class Company < ActiveRecord::Base
  has_many :addresses, as: :addressable
end
```

As you can see the `has_many` calls in the two models are identical, but what about the `as: :addresable` part? This is the option that makes the new polymorphic association work, it tells ActiveRecord that the current model's roles in this association act as `addressable`, as opposed to say a person has many addresses or a company has many addresses.

Next we need to modify the `Address` model to say that it belongs_to addressable things:

```ruby
class Address
  belongs_to :addressable, polymorphic: true
end
```

If we had omitted the `:polymorphic` option to `belongs_to`, ActiveRecord would have assumed that `Address` belonged to object of class `Addressable` and would have managed the foreign keys and lookups in the usual way. However, since we've included the `:polymorphic` option, Active Record knows how to perform lookups based on both the foreign key and the type. 

The best way to understand what's going on here is to see it in action, let's load the rails console and give our news model a spin

```ruby
> person = Person.create(name: "Cosme")
> => #<Person id: 1, name: "Cosme", ...>
> address = Address.create(street: "Av. Siempre viva",...)
> address.addressable = person
> address.addressable_id 
> => 1
> address.addresable_type
> => "Person"
``` 

Aha! Associating a Person with an Address populates both the  `addressable_id` and `addressable_type` field. Naturally associating a `Company` with an `Address` will have the same effect 

```ruby
> company = Company.create(name: "Planta Nuclear")
> => #<Company id: 1, name: "Planta Nuclear",...>
> address = Address.create(street: "Av. Siempre viva 2",...)
> address.addressable = company
> address.addressable_id
> => 1
> address.addressable_type
> => "Company"
```

A call to Company.find(1).addresses will execute the following SQL:


```sql
SELECT *FROM addressesWHERE (addresses.addressable_id = 1 ANDaddresses.addressable_type = 'Company')
```

## Recursive associations

When using rails relationships you'll sometimes find a model that should have a relation to itself. For example, let's imagine you want to model an Employee - Subordinates relationship, and a Employee - Manager relationship. 
Should we do three different models? `Employee`, `Subordinate` and `Manager`? Well we could, but that'll be hard to maintain. 

Lets think about it for a minute, a `Manager` and a `Subordinate` are also `Employees`right? Why don't we use the same model? Luckily for us, with rails relationships we can store all employees in a single database model, and also be able to trace relationships:


```ruby
class Employee < ActiveRecord::Base
  has_many :subordinates, class_name: "Employee",
                          foreign_key: "manager_id"
 
  belongs_to :manager, class_name: "Employee"
end
```

With this setup we can retrieve subordinates from an employee with `@employee.subordinates` and a employee manager with: `@employee.manager`.

Notice that in order for this to work, we'll need to add a column that references to the model itself when creating the employees table

```ruby
class CreateEmployees < ActiveRecord::Migration
  def change
    create_table :employees do |t|
      t.integer :manager
      t.timestamps null: false
    end
  end
end
```


## I18n

I18n is a funny name, but it sure beats typing internationalization all the time. Internationalization after all, starts with an *i*, ends with an *n*, and has eighteen letters in between.

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


But what about if I want my rails application to use english & spanish translations? If you need different settings you can overwrite them easily, the first thing we must do is to create a configuration file that encapsulates our knowledge of what locales are available and which one is to be used as the default

```ruby
#encoding: utf-8
I18n.default_locale = :enLANGUAGES = [	['English', 'en'],	["Espa&ntilde;ol".html_safe, 'es']]
```

The former initializer defines a list of associations between display names and locale names. To get Rails to pick up this configuration change, the server needs to be restarted. 


Since each page that is translated will have an "en" and "es" version, it makes sense to include this in the URL. 

Let's plan to put the locale up front, make it optional, and have it default to the current locale, which in turn will default to English. To accomplish this the first thing that we must do is to modify our `config/routes.rb` file

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
  
  def set_i18n_locale_from_params	if params[:locale]	  if I18n.available_locales.map(&:to_s).include? (params[:locale])        I18n.locale = params[:locale] 	  else
 	    flash.now[:notice] ="#{params[:locale]} translation not available"
 	  end	end  end  def default_url_options
    { locale: I18n.locale }
  end
end

```
The `set_i18n_locale_from_params` does pretty much what it says: it sets the locale from the params, but only if there's a locale in the params, otherwise, it leaves the current locale alone.  

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
- If you add new locale files **the server must be restarted** so Rails get to recognize the new configuration files.

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


## Rspec & Rails

*Most of this definitions and examples were taken from the great book of [Everyday Rails Testing with Rspec](http://everydayrails.com). by [Aaron Summer](https://twitter.com/ruralocity)*

Rails and automated testing go hand in hand, yet many people developing in Rails are either not testing their projects at all, or at best only adding a few token specs on model validations. Maybe is because writing tests is mostly perceived as time taked away from writing the features our clientes or bosses demand. Or maybe the habit of defining “test” as the practice of clicking links in the browser is just too hard to break. Rails ships with a built-in test framework, so yeah, testing’s pretty important in Rails.

### Why Rspec?

Nothing against the other test frameworks out there, but for whatever reason RSpec is the one that’s stuck with us. Maybe because RSpec’s capacity for specs that are readable, without being cumbersome, is a winner, even most non-technical people can read a spec written in RSpec and understand what's going on. 

### Testing philosophy

Approach on testing philosophy focuses on the following foundation:

- Tests should be reliable.
- Tests should be easy to write
- Tests should be easy to understand 
In the end, though, even if your tests are not quite as optimized as they could be, are a great way to start. This approach allows to take an advantage of a fully automated test suite and using tests to drive development and remove potential bugs and edge cases.

### Setting up Rspec


We need to configure our rails applications to recognize and use RSpec and to start generating the appropriate specs  whenever we employ a Rails generator to add code to the application. 

#### Gemfile

Since RSpec isn’t included in a default Rails application, we’ll need to install it by adding it to our Gemfile and running `bundle install` after that

```ruby
group :development, :test do  gem "rspec-rails" 
  gem "factory_girl_rails" endgroup :test do  gem "faker"
  gem "capybara" 
  gem "database_cleaner" 
  gem 'rspec-collection_matchers'
  gem "capybara-selenium"end
```

But hey why do we install it in two separate groups?

`rspec-rails` and `factory_girl_rails` are used in both the development and test environments. Specifically, they are used in development by generators we’ll be utilizing shortly. The remaining gems are only used when you actually run your specs, so they’re not necessary to load indevelopment. 

What we just install?

- [rspec-rails](https://github.com/rspec/rspec-rails) includes RSpec itself
- [rspec-collection-matchers](https://github.com/rspec/rspec-collection_matchers) lets you expressed expected outcomes on collection objects or an object itself- [factory girl rails](https://github.com/thoughtbot/factory_girl_rails) replaces fixtures for feeding test data to the test suite with much more preferable factories.- [faker](https://github.com/stympy/faker) generates random data like names, email addresses, etc.- [capybara](https://github.com/jnicklas/capybara) makes it easy to programatically simulate your user's interactions with your application.- [database_cleaner](https://github.com/DatabaseCleaner/database_cleaner) helps make sure each spec run in RSpec begins with a clean slate.
- [rspec-collection_matcher](https://github.com/rspec/rspec-collection_matchers) lets you express expected outcomes on collections of an object in an example.- [capybara-selenium](https://github.com/seleniumhq/selenium) will let us test JavaScript-based browser interactions with Capybara.

#### Database configuration

Wait, what? Yup tests have its own database. Open `config/database.yml` file to see which databases your application is ready to talk to. If you haven’t made any changes to the file, you should see something like the following:

```ruby
default: &default
  adapter: sqlite3
  pool: 5
  timeout: 5000

development:
  <<: *default
  database: db/development.sqlite3

test:
  <<: *default
  database: db/test.sqlite3

production:
  <<: *default
  database: db/production.sqlite3

```

You see the `test` section? That is where we configure our test database. To ensure there’s a database to talk to, run the following rake task:

```bash$ rake db:create:all
```

Or simply create a database for our test env

```bash
$ RAILS_ENV=test rake db:create
```

After that, just as development,  you'll need to run migrations

```bash
$ RAILS_ENV=test rake db:migrate
```

Now you do have a test database!

#### Rspec itself

Install RSpec into the application with the following command line directive:

```bash$ rails g rspec:install
create  .rspec
create  spec
create  spec/rails_helper.rb
create  spec/spec_helper.rb
```
As the generator reports, we’ve now got:

- a configuration file for RSpec (`.rspec`)
- a directory for our spec files as we create them (spec)
- and two helper files where we’ll further customize how RSpec will interact with our rails app(`spec/spec_helper.rb` & `spec/rails_helper.rb`).

In prior versions to Rspec 3, only a single `spec_helper.rb` file was generated. This file has been moved to `rails_helper.rb`. This change was made to accomplish two general goals:

- Keep the installation process in sync with regular RSpec changes

- Provide an out-of-the-box way to avoid loading Rails for those specs that do not require itNext and this is optional, but really recommended, change RSpec’s output from the default format to the easy to read documentation format, open .rspec and add the following line:

```
--format documentation
--color
```

This makes it easier to see which specs are passing and which are failing as your suite runs; it also provides an attractive outline of your specs for documentation purposes

### Model specs

It's easiest to learn testing at the model level because doing so allows you to examine and test the core building blocks of an application. Well-tested code at this level is solid foundation and the first step toward a reliable overall code base.

To get started, a model spec should include tests for the following:

- The model's create method, when passed valid attributes, should be valid
- Data that failed validations should not be valid
- Class and instance methods perform as expected

#### Creating a model spec

Suppose we need a `Contact` model with the following requirements: 

- It should have a first name, last name and email
- It will be invalid without any of the former attributes
- It will be invalid with a duplicate email address
- We'll need to provide a way to display the full name

This give us quite a bit for start, we can easly translate this requirements into our own model spec

First open up the `spec` directory and if necessary, create a subdirectory named `models`, finally inside this subdirectory create a file named `contact_spec`

```ruby
# spec/models/contact_spec.rb
require 'rails_helper'

describe Contact do  it "is valid with a firstname, lastname and email"  it "is invalid without a firstname"  it "is invalid without a lastname"  it "is invalid without an email address"  it "is invalid with a duplicate email address"  it "returns a contact's full name as a string"end
```

Did you see the `require 'rails_helper'` at the top of the file? Well get ready to typing it across all of your specs. 

In the former file take notice that: 

- Each example (a line beginning with `it`) only expects one thing
- Also each "it" describe a set of expectations of what a `Contact` should look like. It is really explicit!
- The descriptive after `it`is technically optional in Rspec, however omitting it makes your specs more difficult to read
- **Each example descriptions begins with a verb**. Try to read the expectations out loud: *Contact is valid with a firstname, last name and email, Contact is invalid without a firstname* and so on. **Readibility as you see is really important**

Now simply create a model `Contact` inside app/models like:

```ruby
class Contact < ActiveRecord::Base
end
```

If we ran the specs right now from the command line, we'll obtain an output like the following:

```bash
$ rspec
Contact  is valid with a firstname, lastname and email (PENDING:Not yet implemented)  is invalid without an email address (PENDING: Not yet implemented)  returns a contact's full name as a string (PENDING:Not yet implemented)  is invalid with a duplicate email address (PENDING:Not yet implemented)  is invalid without a firstname (PENDING: Not yet implemented)  is invalid without a lastname (PENDING: Not yet implemented)Pending:Contact is valid with a firstname, lastname and email# Not yet implemented# ./spec/models/contact_spec.rb:4Contact is invalid without an email address# Not yet implemented# ./spec/models/contact_spec.rb:7Contact returns a contact's full name as a string# Not yet implemented# ./spec/models/contact_spec.rb:9Contact is invalid with a duplicate email address# Not yet implemented# ./spec/models/contact_spec.rb:8Contact is invalid without a firstname# Not yet implemented# ./spec/models/contact_spec.rb:5Contact is invalid without a lastname# Not yet implemented# ./spec/models/contact_spec.rb:6

Finished in 0.00098 seconds
6 examples, 0 failures, 6 pending
```

Which basically states that we have six pending specs, let's start with the first one

```ruby
it "is valid with a firstname, lastname and email" do
  contact = Contact.new(firstname: 'Cosme', lastname: 'Fulanito', email: 'cosme_fulanito@hackerschool.com')
  expect(contact).to be_validend
```

And modified the model to meet our expectations

```ruby
class Contact < ActiveRecord::Base
  validates :firstname, presence: true
  validates :lastname, presence: true
  validates :email, presence: true
end
```

This simple example uses RSpec's `be_valid` matcher to verify that our model knows what it has to look like to be valid. If we run rspec again, we'll see one passing example. Let's go ahead and complete the next spec 

```ruby
it "is invalid without a firstname" do
  expect(Contact.new(firstname: nil)).to have(1).errors_on(:firstname)
end
```

Notice that we're **expecting that the new contact (with a firstname explicitly set to nil) will not be valid, by returning an error on the contact's firstname attribute**. Given the validations inside our `Contact` model if we run rspec again we should be up two passing specs. Let's copy the same behaviour for spec 3 & 4

```ruby
it "is invalid without a lastname" do
  expect(Contact.new(lastname: nil)).to have(1).errors_on(:lastname)
end
  
it "is invalid without an email" do
  expect(Contact.new(email: nil)).to have(1).errors_on(:email)
end 
```

If we run rspec again we should see four of our specs passing! yeih! ... Testing email address uniqueness should be fairly simple as well:

```ruby  
  it "is invalid with a duplicate email address" do
    Contact.create(firstname: 'Cosme', lastname: 'Fulanito', email: 'cosme_fulanito@hackerschool.com')
    contact = Contact.new( firstname: 'Abraham', lastname: 'Cosme', email: 'cosme_fulanito@hackerschool.com')
    expect(contact).to have(1).errors_on(:email)  end
end  
```

There's a subtle difference in the last example: We use `create` instead of `new`, so we persisted a contact, why? Becase the spec requires an email contact to compare to, so by saving a contact with an email, we can ensure there's information to run against it. If we modify our model accordingly:

```ruby
class Contact < ActiveRecord::Base
  validates :firstname, presence: true
  validates :lastname, presence: true
  validates :email, presence: true, uniqueness: true
end
```

And then run rspec again, we'll have 5 of our 6 specs passing! Nice! 

Let's finish this up by completing the last spec:

```ruby
it "returns a contact's full name as a string" do  contact = Contact.new(firstname: 'Cosme', lastname:'Fulanito', email: 'cosme_fulanito@email.com')
  expect(contact.name).to eq 'Cosme Fulanito'end
```

And adding the regarding code into our `Contact` model

```ruby
class Contact < ActiveRecord::Base
  validates :firstname, presence: true
  validates :lastname, presence: true
  validates :email, presence: true, uniqueness: true
  
  def name
    [firstname, lastname].join(' ')
  end
end
```

This is the complete `contact_spec` content:

```ruby
require 'rails_helper'describe Contact do
  it "is valid with a firstname, lastname and email" do
    contact = Contact.new(firstname: 'Cosme', lastname: 'Fulanito', email: 'cosme_fulanito@hackerschool.com')
    expect(contact).to be_valid  end
    it "is invalid without a firstname" do
    expect(Contact.new(firstname: nil)).to have(1).errors_on(:firstname)
  end  
  it "is invalid without a lastname" do
    expect(Contact.new(lastname: nil)).to have(1).errors_on(:lastname)
  end
  
  it "is invalid without an email" do
    expect(Contact.new(email: nil)).to have(1).errors_on(:email)
  end
  
  it "is invalid with a duplicate email address" do
    Contact.create(firstname: 'Cosme', lastname: 'Fulanito', email: 'cosme_fulanito@hackerschool.com')
    contact = Contact.new( firstname: 'Abraham', lastname: 'Cosme', email: 'cosme_fulanito@hackerschool.com')
    expect(contact).to have(1).errors_on(:email)  end
  
  it "returns a contact's full name as a string" do    contact = Contact.new(firstname: 'Cosme', lastname:'Fulanito', email: 'cosme_fulanito@email.com')
    expect(contact.name).to eq 'Cosme Fulanito'  end
end 
```

If we run rspec we should see all of our specs passing! 

```bash
$ rspec
Contact  returns a contact's full name as a string  is invalid without a firstname  is invalid with a duplicate email address  is invalid without a lastname  is valid with a firstname, lastname and email  is invalid without an email address

Finished in 0.44718 seconds6 examples, 0 failures
```

Awesome! 

#### Testing non ideal results

Lets complicate things a bit, we now got a new requirement (**damn you client!**): 

- The user must have the ability to make a search for a contact based on a given letter.

This action will also need the controller, but for know lets just keep focus on the model, so lets write the corresponding spec for this (*earlier specs will be omitted for reading purposes*) 

```ruby
require 'rails_helper'describe Contact do
  ... 
  # Earlier specs
  ...
  it "returns a sorted array of results that match" do
  	cosme = Contact.create(firstname: 'Cosme', lastname: 'Fulanito', email: 'cosme@hackerschool.com')
  	jones = Contact.create(firstname: 'Tim', lastname: 'Jones', email: 'jones@hackerschool.com')
  	johnson = Contact.create(firstname: 'John', lastname: 'Johnson', email: 'johnson@hackerschool.com')   
   expect(Contact.by_letter("J")).to eq [johnson, jones] 
  end
end  
```

Let's incorpore the method `by_letter` into our `Contact` model that fulfills the spec:

```ruby
class Contact < ActiveRecord::Base
  validates :firstname, presence: true
  validates :lastname, presence: true
  validates :email, presence: true, uniqueness: true
  
  def name
    [firstname, lastname].join(' ')
  end
  
  def self.by_letter(letter)
    where("lastname LIKE ?", "%#{letter}%").order(:lastname)
  end
end
```

Great! But there's a small problem we have only test for the *happy path*, we must also test for another not so happy endings, lets do that:

```ruby
require 'rails_helper'describe Contact do
  ... 
  
  it "returns a sorted array of results that match" do
  	cosme = Contact.create(firstname: 'Cosme', lastname: 'Fulanito', email: 'cosme@hackerschool.com')
  	jones = Contact.create(firstname: 'Tim', lastname: 'Jones', email: 'jones@hackerschool.com')
  	johnson = Contact.create(firstname: 'John', lastname: 'Johnson', email: 'johnson@hackerschool.com')   
   expect(Contact.by_letter("J")).to eq [johnson, jones] 
  end
  
  it "returns only results that match" do
  	cosme = Contact.create(firstname: 'Cosme', lastname: 'Fulanito', email: 'cosme@hackerschool.com')
  	jones = Contact.create(firstname: 'Tim', lastname: 'Jones', email: 'jones@hackerschool.com')
  	johnson = Contact.create(firstname: 'John', lastname: 'Johnson', email: 'johnson@hackerschool.com')   
   expect(Contact.by_letter("J")).to_not include cosme
  end
end  
```

Now we are also testing the other side of the coin: We're verifying that `by_letter`method returns the expecting results and to not return those contacts whose name doesn't match with the letter

#### DRYer specs with describe and context

Just as in our rails application, **the DRY principles also applies to our tests**, and if we check our two latest specs we can see that there's a lot of duplicated code. Let's use another rspec feacture to clean this up

```ruby
require 'rails_helper'describe Contact do
  ... 
  describe "filter last name by letter" do
    before(:each) do
      @cosme = Contact.create(firstname: 'Cosme', lastname: 'Fulanito', email: 'cosme@hackerschool.com')
  	  @jones = Contact.create(firstname: 'Tim', lastname: 'Jones', email: 'jones@hackerschool.com')
  	  @johnson = Contact.create(firstname: 'John', lastname: 'Johnson', email: 'johnson@hackerschool.com')
    end
    
    it "returns a sorted array of results that match" do
      expect(Contact.by_letter("J")).to eq [@johnson, @jones]
    end
    
    it "returns a sorted array of results that match" do
      expect(Contact.by_letter("J")).to_not include @cosme
    end
  end
end
```
Notice that:

- We're using a `describe` block within the `describe Contact` block to help us sort similar examples together.
- We're also using a `before` hook, which is vital to clean up our nasty redundancy. As you might have guess the code inside our `before` block is run before `each` example 

Let's break things down even further by including a couple of `context` blocks one for matching letters and one for non-matching


```ruby
require 'rails_helper'describe Contact do
  ... 
  describe "filter last name by letter" do
    before(:each) do
      @cosme = Contact.create(firstname: 'Cosme', lastname: 'Fulanito', email: 'cosme@hackerschool.com')
  	  @jones = Contact.create(firstname: 'Tim', lastname: 'Jones', email: 'jones@hackerschool.com')
  	  @johnson = Contact.create(firstname: 'John', lastname: 'Johnson', email: 'johnson@hackerschool.com')
    end
    
    context "matching letters" do
     it "returns a sorted array of results that match" do
       expect(Contact.by_letter("J")).to eq [@johnson, @jones]
     end
    end
    
    context "non-matching letters" do
      it "returns a sorted array of results that match" do
        expect(Contact.by_letter("J")).to_not include @cosme
      end
    end
  end
end
```

While `describe` and `context` are technically interchangeable, **its considered a good practice to use `describe` to outline general functionality and `context` to outline specific state.** When we run the specs we'll see a nice outline like this

```bash
Contact  returns a contact's full name as a string  is invalid without a firstname  is invalid with a duplicate email address  is invalid without a lastname  is valid with a firstname, lastname and email  is invalid without an email address  filter last name by letter    matching letters      returns a sorted array of results that match    non-matching letters      returns a sorted array of results that match

Finished in 0.44718 seconds8 examples, 0 failures
```

### Factories

So far we've been using plain old Ruby objects to create temporary data for our tests, the thing is that as we test for more complex scenarios, we would need to simplify that aspect of the process and **focus more on the test instead of the data**. Luckily, a handful of Ruby libraries exist to make test data generation easy.

Out of the box, Rails provides a way of quickly generating sample data called "fixtures". A fixture is essentially a YAML-formatted file which helps create sample data, for our `Contact` model a simple fixture file would look like this:

```
cosme:  firstname: "Cosme"  lastname: "Fulanito"  email: "cosme_fulanito@hackerschool.com"
```

The things with fixtures is that **Rails bypasses ActiveRecord when it loads fixture data into your models, so basically all validations are ignored!**, and of course this is bad, really really bad.

That's is why factories exist: they are simple, flexible, and allow to build blocks for test data. And for building factories there's a wonderful gem called [Factory Girl](https://github.com/thoughtbot/factory_girl)

##### Adding factories to our application

Back in our spec directory, we'll need to add another subdirectory named `factories`, and then create a file called `contacts.rb` with the following content:

```
FactoryGirl.define do  factory :contact do    firstname "Cosme"    lastname "Fulanito"    sequence(:email) { |n| "cosme_fulanito#{n}@hackerschool.com"}
  endend
```

Let's break the called to this factory, So whenever we called `FactoryGirl.create(:contact)`:

- The contact name will be called `Cosme`
- With a lastname of "Fulanito"
- Since we have an uniqueness email validation in our `Contact` model, we'll take an advantage of a cool FactoryGirl feature `sequences`. As you might have guessed sequences will automatically increment n inside the block, yielding stuff like `cosme_fulanito1@hackerschool.com`, `cosme_fulanito2@hackerschool.com`, and so on

If we want we can have all our factories in a single file. However the good practice is to have a plural file that mirrors the model name, so for example `spec/factories/contacts.rb` corresponds to `Contact` model


In the previous example we're using strings for all of our attributes but you might also use whatever attribute's data type you want, and that includes integers, boolean, dates and you can even pass Ruby code to dynamically  assign values! Cool huh?


With our contact factory set up, let's return now to our `contact_spec`

```ruby
require 'rails_helper'

describe Contact do
  it "has a valid factory" do
    expect(FactoryGirl.build(:contact)).to be_valid
  end
  ...
end  
```

The former example instantiates (**but does not save**) a new contact with attributes as assigned by the factory, it then tests that new contact's validity. 

Compared to the old one, before de factories, you can notice that the spec with the factories is a lot more concise: 

```ruby

# OLd one without factories
it "is valid with a firstname, lastname and email" do
  contact = Contact.new(firstname: 'Cosme', lastname: 'Fulanito', email: 'cosme_fulanito@hackerschool.com')
  expect(contact).to be_validend
```

Lets change the following specs:

```ruby
it "is invalid without a firstname" do
  contact = FactoryGirl.build(:contact, firstname: nil)
  expect(contact).to have(1).errors_on(:firstname)
end

it "is invalid without a lastname" do
  contact = FactoryGirl.build(:contact, lastname: nil)
  expect(contact).to have(1).errors_on(:lastname)
  end

it "is invalid without an email address" do
  contact = FactoryGirl.build(:contact, email: nil)
  expect(contact).to have(1).errors_on(:email)
end
```

What about the *"is invalid with a duplicate email address"*, well here is the result:

```ruby
it "is invalid with a duplicate email address" do
  FactoryGirl.create(:contact, email: "cosme_fulanito@hackerschoo.com")
  contact = FactoryGirl.build(:contact, email: "cosme_fulanito@hackerschoo.com")
  expect(contact).to have(1).errors_on(:email)end
```

Our complete `contact_spec` would look like this

```ruby
require 'rails_helper'describe Contact do
  it "has a valid factory" do
    expect(FactoryGirl.build(:contact)).to be_valid
  end
    it "is invalid without a firstname" do
    contact = FactoryGirl.build(:contact, firstname: nil)
    expect(contact).to have(1).errors_on(:firstname)
  end

  it "is invalid without a lastname" do
    contact = FactoryGirl.build(:contact, lastname: nil)
    expect(contact).to have(1).errors_on(:lastname)
  end

  it "is invalid without an email address" do
    contact = FactoryGirl.build(:contact, email: nil)
    expect(contact).to have(1).errors_on(:email)
  end
  
  it "is invalid with a duplicate email address" do
    FactoryGirl.create(:contact, email: "cosme_fulanito@hackerschoo.com")
    contact = FactoryGirl.build(:contact, email: "cosme_fulanito@hackerschoo.com")
    expect(contact).to have(1).errors_on(:email)  end
  
  it "returns a contact's full name as a string" do   contact = FactoryGirl.build(:contact, firstname: "Cosme", lastname: "Fulanito")
    expect(contact.name).to eq "Cosme Fulanito"  end
end 
```
##### Shorter FactoryGirl syntax


Our contact spec is looking great! There's just a little something, us as a programmers we hate extra typing, so maybe everytime we type `FactoryGirl.build(:contact)` a kitty might died, and we don't want that do we? 

Let's use rspec short syntax! To enable it we need to add the following line into our  `rails_helper.rb`

```ruby
# spec/rails_helper.rb
RSpec.configure do |config|  #Include Factory Girl syntax to simplify calls to factories  config.include FactoryGirl::Syntax::Methods
end
```

Now our specs can use the shorter version of FactoryGirl methods: 

- `build(:contact)` instead of `FactoryGirl.build(:contact)`
- `create(:contact)` instead of `FactoryGirl.create(:contact)` and so on. 

Look of how clean looks our `contact_spec`

```ruby
require 'rails_helper'

describe Contact do
  it "has a valid factory" do
    expect(build(:contact)).to be_valid
  end

  it "is invalid without a firstname" do
    contact = build(:contact, firstname: nil)
    expect(contact).to have(1).errors_on(:firstname)
  end

  it "is invalid without a lastname" do
    contact = build(:contact, lastname: nil)
    expect(contact).to have(1).errors_on(:lastname)
  end

  it "is invalid without an email address" do
    contact = build(:contact, email: nil)
    expect(contact).to have(1).errors_on(:email)
  end

  it "is invalid with a duplicate email address" do
    create(:contact, email: "cosme_fulanito@hackerschool.com")
    contact = build(:contact, email: "cosme_fulanito@hackerschool.com")
    expect(contact).to have(1).errors_on(:email)
  end

  it "returns a contact's full name as a string" do
    contact = build(:contact,
      firstname: "Cosme", lastname: "Fulanito")
    expect(contact.name).to eq "Cosme Fulanito"
  end

  describe "filter last name by letter" do
    before :each do
      @cosme = create(firstname: 'Cosme', lastname: 'Fulanito', email: 'cosme@hackerschool.com')
  	  @jones = create(firstname: 'Tim', lastname: 'Jones', email: 'jones@hackerschool.com')
  	  @johnson = create(firstname: 'John', lastname: 'Johnson', email: 'johnson@hackerschool.com')
    end

    context "matching letters" do
      it "returns a sorted array of results that match" do
        expect(Contact.by_letter("J")).to eq [@johnson, @jones]
      end
    end

    context "non-matching letters" do
      it "returns a sorted array of results that match" do
        expect(Contact.by_letter("J")).to_not include @smith
      end
    end
  end
end
```

##### Associations and inheritance

What about associations? How can I associate factories to represent data just like models do?. Well lets see, imagine our `Contact` model has many `Phone`s

```ruby
# contact.rb
class Contact < ActiveRecord::Base
  has_many :phones
end  

# phone.rb
class Phone < ActiveRecord::Base
  belongs_to :contact
end  
```

Given the former models, the factory for `Phone` would look like this.

```ruby
FactoryGirl.define do
  factory :phone do
    association :contact
    phone '123-555-1234'    phone_type 'home'  end
end
```

Did you spot the `association` keyword? Well it tells FactoryGirl to create a new `Contact` on the fly for this phone, but if we passed another `:contact`association whether in `build` or `create` method this value will be overriden. 


So what'd happen if in the application we handle different types of phones? You know one for the office, another one for home and a mobile phone? Well the long way is to do something like this

```ruby
it "allows two contacts to share a phone number" do  create(:phone, phone_type: 'home', phone: "785-555-1234")  expect(build(:phone, phone_type: 'home', phone: "785-555-1234")).to be_validend
```

But FactoryGirl provides us the ability to **inherited factories, overriding attributes as necessary**

```
FactoryGirl.define do
  factory :phone do
    association :contact
    phone "123-456-789"

    factory :home_phone do
      phone_type 'home'
    end

    factory :work_phone do
      phone_type 'work'
    end

    factory :mobile_phone do
      phone_type 'mobile'
    end
  end
end
```

And with that the `phone_spec` can be simplified to: 

```ruby
it "allows two contacts to share a phone number" do  create(:home_phone, phone: "785-555-1234")  expect(build(:home_phone, phone: "785-555-1234")).to be_validend
```

#### Generating more realistic fake data

We can improve our test data to make it look more realistic through [faker](https://github.com/stympy/faker) gem, which is a ruby por library for generating fake names, addresses, sentences and more!

If we incorpore some fake data into our contact factory:

```
require 'faker'

FactoryGirl.define do
  factory :contact do
    firstname { Faker::Name.first_name }
    lastname { Faker::Name.last_name }
    email { Faker::Internet.email }
  end
end
```

Now our specs will use a random email address each time the phone factory is used! 

There are some important things to notice here

- We've required the faker library to load in the first line
- We are calling `Faker` methods inside blocks, this is because Factory Girl considers this a "lazy attribute" as opposed to the statically-added string

Let's modify also our `phone`'s factory

```ruby
require 'faker'

FactoryGirl.define do
  factory :phone do
    association :contact
    phone { Faker::PhoneNumber.phone_number }

    factory :home_phone do
      phone_type 'home'
    end

    factory :work_phone do
      phone_type 'work'
    end

    factory :mobile_phone do
      phone_type 'mobile'
    end
  end
end
```

Faker isn't strictly necessary, we could keep using sequences and everything would be fine, but faker does give us a bit more realistic data with which to test (and also some generated data is pretty fun :-))


