## Table of Contents

* [Polymorphic associations]()
* [Recursive associations]() 
* [Rspec & Rails]()
* [Exercises]()

## Polymorphic associations

Active Record's `has_many` and `belongs_to` associations work really well when the two sides of the relationship have fixed classes: 

- An `author`can have many `Books`. 
- A `Library` can have many `Books`,etc. 

But sometimes you may want to use one table and model to represent something that can be associated with many types of entities. For example what about if I have an `Address` model that belong to a `Person` and lets say that I also want to register addresses for companies, should I create two different models? like `AddressAuthor` and `AddressCompany`? Of course not! This is where Polymorphic associations come in handy. 

The name may be daunting, but there's nothing to fear. **Polymorphic assocations allow you to associate one type of object with objects of many types**. So for example, with polymorphics associations, an `Address` can belong to a `Person` or a `Company` or to any other model that wants to declare and use the association. 


Let's work through a basic example. 

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

I guess you have notice some things unusual about the `addresses` migration, why `addressable_id` and `addressable_type` show up? What are those? Well in order to declare a polymorphic association we need to departure from the usual ActiveRecord convention: 

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
As you can see the `has_many` calls in the two models are identical. But what about the `as: :addresable` part, this is the option that makes the new polymorphic association work, it tells ActiveRecord that the current model's roles in this association act as `addressable`, as opposed to say a `person` has many `addresses` or a `company` has many `addresses`.

Next we need to modify the `Address` model to say that it belongs_to addressable things:

```ruby
class Address
  belongs_to :addressable, polymorphic: true
end
```

If we had omitted the `:polymorphic` option to `belongs_to`, ActiveRecord would have assumed that `Address` belonged to object of class `Addressable`and would have managed the foreign keys and lookups in the usual way. However, since we've included the `:polymorphic` option in our `belongs_to` declaration, Active Record knows how to perform lookups based on both the foreign key and the type. 

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

Notice that in both examples, the addressable_id values have been set to 1. If the relationship wasn’t declared to be polymorphic, a call to `Company.find(1).addresses` would result in the same (incorrect) list that `Person.find(1).addresses` would return, because Active Record would have no way of distinguishing betweenperson 1 and company 1.Instead, a call to Company.find(1).addresses will execute the following SQL:


```sql
SELECT *FROM addressesWHERE (addresses.addressable_id = 1 ANDaddresses.addressable_type = 'Company')
```

## Recursive associations

When using rails relationships you will sometimes find a model that should have a relation to itself. For example, let's imagine you want to model an Employee - Subordinates relationship, and a Employee - Manager relationship. 
Should we do three different models? `Employee`, `Subordinate` and `Manager`? Well that'll be hard to maintain. Lets think about it for a minute, a `Manager` and a `Subordinate` are also `Employees`right? Why don't we use the same model? Luckily for us, with rails relationships we can store all employees in a single database model, and also be able to trace relationships such as between manager and subordinates:


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
      t.integer :manager, index: true
      t.timestamps null: false
    end
  end
end
```

## Rspec & rails
*Most this definitions and examples were taken from the great book of [Everyday Rails Testing with Rspec](http://everydayrails.com). by [Aaron Summer](https://twitter.com/ruralocity)*

Rails and automated testing go hand in hand, yet many people developing in Rails are either not testing their projects at all, or at best only adding a few token specs on model validations. Maybe is because writing tests is mostly perceived as time taked away from writing the features our clientes or bosses demand. Or maybe the habit of defining “test” as the practice of clicking links in the browser is just too hard to break. Rails ships with a built-in test framework, so yeah, testing’s pretty important in Rails.

### Why Rspec?

Nothing against the other test frameworks out there, but for whatever reason RSpec is the one that’s stuck with us. Maybe because RSpec’s capacity for specs that are readable, without being cumbersome, is a winner, even most non-technical people can read a spec written in RSpec and understand what's going on. 

### Testing philosophy

Approach on testing philosophy focuses on the following foundation:

- Tests should be reliable.
- Tests should be easy to write
- Tests should be easy to understand 
If you mind these three factors in your approach, you’ll go a long way toward becoming an honest-to-goodness practitioner of Test-Driven Development. In the end, though,  even if your tests are not quite as optimized as they could be, are a great way to start. This approach allows to take an advantage of a fully automated test suite and using tests to drive development and remove potential bugs and edge cases.

### Setting up Rspec


We need to configure our rails applications to recognize and use RSpec and to start generating the appropriate specs  whenever we employ a Rails generator to add code to the application. 

#### Gemfile

Since RSpec isn’t included in a default Rails application, we’ll need to install it by adding it to our Gemfile and running `bundle install`after that

```
group :development, :test do  gem "rspec-rails", "~> 2.14.0"
  gem "factory_girl_rails", "~> 4.2.1"endgroup :test do  gem "faker", "~> 1.1.2"
  gem "capybara", "~> 2.1.0"
  gem "database_cleaner", "~> 1.0.1"
  gem "launchy", "~> 2.3.0"end
```

But hey why do we install it in two separate groups?

`rspec-rails` and `factory_girl_rails` are used in both the development and test environments. Specifically, they are used in development by generators we’ll be utilizing shortly. The remaining gems are only used when you actually run your specs, so they’re not necessary to load indevelopment. 

What we just install?

- [rspec-rails](https://github.com/rspec/rspec-rails) includes RSpec itself- [factory girl rails](https://github.com/thoughtbot/factory_girl_rails) replaces fixtures for feeding test data to the test suite with much more preferable factories.- [faker](https://github.com/stympy/faker) generates random data like names, email addresses, etc.- [capybara](https://github.com/jnicklas/capybara) makes it easy to programatically simulate your user's interactions with your application.- [database_cleaner](https://github.com/DatabaseCleaner/database_cleaner) helps make sure each spec run in RSpec begins with a clean slate.- [launchy](https://github.com/copiousfreetime/launchy) It opens your default web browser on demand toshow you what your application is rendering. Very useful for debugging tests.- [selenium-webdriver](https://github.com/seleniumhq/selenium) will let us test JavaScript-based browser interactions with Capybara.

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


Now you do have a test database. 

#### Rspec itself

Install RSpec with the following command line directive:

```bash$ rails g rspec:install
create  .rspec
create  spec
create  spec/spec_helper.rb
```
As the generator reports, we’ve now got:

- a configuration file for RSpec (`.rspec`)
- a directory for our spec files as we create them (spec)
- and a helper file where we’ll further customize how RSpec will interact with our code (`spec/spec_helper.rb`).Next and this is optional, but really recommended, change RSpec’s output from the default format to the easy to read documentation format, open .rspec and add the following line:

```
--format documentation
--color
```

This makes it easier to see which specs are passing and which are failing as your suite runs; it also provides an attractive outline of your specs for documentation purposes

### Model specs

It's easiest to learn testing at the model level because doing so allows you to examine and test the core building blocks of an application. Well-tested code at this level is solid foundation and the first step toward a reliable overall code base.

To get started, a model spec should include tests for the following:

- The model's create method, when passed valid attributes, should be valid
- Data thta faild validations should not be valid
- Class and instance methods perform as expected

#### Creating a model spec

Suppose we need a `Contact` model with the following requirements: 

- It should have a first name, last name and email
- It will be invalid without any of the former attributes
- It will be invalid with a duplicate email address
- We'll need to provide a way to display the full name

This give us quite a bit for startes, we can easly translate this requirements into our own model spec

First open up the `spec` directory and if necessary, create a subdirectory named `models`, finally inside this subdirectory create a file named `contact_spec`

```
# spec/models/contact_spec.rb
require spec_helper

describe Contact do  it "is valid with a firstname, lastname and email"  it "is invalid without a firstname"  it "is invalid without a lastname"  it "is invalid without an email address"  it "is invalid with a duplicate email address"  it "returns a contact's full name as a string"end
```

Notice: 

- The `require spec_helper`at the top, and get used to typing it across all of your specs
- It describes a set of expectations of what a `Contact` should look like
- Each example (a line beginning with `it`) only expects one thing
- Each example is explicit. The descriptive after `it`is techincally optiona in Rspec, however omitting it makes your specs more difficult to read
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
require 'spec_helper'describe Contact do
  it "is valid with a firstname, lastname and email" do
    contact = Contact.new(firstname: 'Cosme', lastname: 'Fulanito', email: 'cosme_fulanito@hackerschool.com')
    expect(contact).to be_valid  end  ...
end  
```

And modified the model to meet our expectations

```
class Contact < ActiveRecord::Base
  validates :firstname, presence: true
  validates :lastname, presence: true
  validates :email, presence: true
end
```

This simple example uses RSpec's `be_valid` matcher to verify that our model knows what it has to look like to be valid. If we run rspec again, we'll see one passing example. Let's go ahead and complete the next spec 

```ruby
require 'spec_helper'describe Contact do
  it "is valid with a firstname, lastname and email" do
    contact = Contact.new(firstname: 'Cosme', lastname: 'Fulanito', email: 'cosme_fulanito@hackerschool.com')
    expect(contact).to be_valid  end
    it "is invalid without a firstname" do
    expect(Contact.new(firstname: nil)).to have(1).errors_on(:firstname)
  end  ...
end  
```

Notice that we're **expecting that the new contact (with a firstname explicitly set to nil) will not be valid, by returning an error on the contact's firstname attribute**. Given the validations inside our `Contact`model if we run rspec again we should be up two passing specs. Let's copy the same behaviour for spec 3 & 4

```ruby
require 'spec_helper'describe Contact do
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
  ...
end  
```

If we run rspec again we should see four of our specs passing! yeih! ... 

Testing email address uniqueness should be fairly simple as well:

```ruby
require 'spec_helper'describe Contact do
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
  ...
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
require 'spec_helper'describe Contact do
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

If we run rspec we should see all of our specs passing! 

```bash
$ rspec
Contact  returns a contact's full name as a string  is invalid without a firstname  is invalid with a duplicate email address  is invalid without a lastname  is valid with a firstname, lastname and email  is invalid without an email address

Finished in 0.44718 seconds6 examples, 0 failures
```

Awesome! 

#### Testing non ideal results

Lets complicate things a bit, we now got a new requirement: 

- A list of contacts whose names begin with a given letter should be return

Lets write the corresponding spec for this (*earlier specs will be omitted for reading purposes*) 

```ruby
require 'spec_helper'describe Contact do
  ... 
  
  it "returns a sorted array of results that match" do
  	cosme = Contact.create(firstname: 'Cosme', lastname: 'Fulanito', email: 'cosme@hackerschool.com')
  	jones = Contact.create(firstname: 'Tim', lastname: 'Jones', email: 'jones@hackerschool.com')
  	johnson = Contact.create(firstname: 'John', lastname: 'Johnson', email: 'johnson@hackerschool.com')   
   expect(Contact.by_letter("J")).to eq [johnson, jones] 
  end
end  
```

Let's incorpore a method into our `Contact` model that fulfills the spec:


```ruby
class Contact < ActiveRecord::Base
  validates :firstname, presence: true
  validates :lastname, presence: true
  validates :email, presence: true, uniqueness: true
  
  def name
    [firstname, lastname].join(' ')
  end
  
  def self.by_letter(letter)
    where("lastname LIKE ?", "#{letter}%").order(:lastname)
  end
end
```

Great! But there's a small problem we have only test for the *happy path*, we must also test for another not so happy endings

```ruby
require 'spec_helper'describe Contact do
  ... 
  
  it "returns a sorted array of results that match" do
  	cosme = Contact.create(firstname: 'Cosme', lastname: 'Fulanito', email: 'cosme@hackerschool.com')
  	jones = Contact.create(firstname: 'Tim', lastname: 'Jones', email: 'jones@hackerschool.com')
  	johnson = Contact.create(firstname: 'John', lastname: 'Johnson', email: 'johnson@hackerschool.com')   
   expect(Contact.by_letter("J")).to eq [johnson, jones] 
  end
  
  it "returns a sorted array of results that match only" do
  	cosme = Contact.create(firstname: 'Cosme', lastname: 'Fulanito', email: 'cosme@hackerschool.com')
  	jones = Contact.create(firstname: 'Tim', lastname: 'Jones', email: 'jones@hackerschool.com')
  	johnson = Contact.create(firstname: 'John', lastname: 'Johnson', email: 'johnson@hackerschool.com')   
   expect(Contact.by_letter("J")).to_not include cosme
  end
end  
```

Now we are also testing for letters with no results.

#### DRYer specs with describe and context

Just as in our rails application, **the DRY principles applies to our tests**, and if we check our two latest specs we can see that there's a lot of duplicated code, let's use another rspec feacture to clean this up

```ruby
require 'spec_helper'describe Contact do
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
require 'spec_helper'describe Contact do
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

While `describe` and `context` are technically interchangeable, its considered a good practice to use `describe` to outline general functionality and `context` to outline specific state. When we run the specs we'll see a nice outline like this

```bash
Contact  returns a contact's full name as a string  is invalid without a firstname  is invalid with a duplicate email address  is invalid without a lastname  is valid with a firstname, lastname and email  is invalid without an email address  filter last name by letter    matching letters      returns a sorted array of results that match    non-matching letters      returns a sorted array of results that match

Finished in 0.44718 seconds8 examples, 0 failures
```

#### Factories

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
FactoryGirl.define do  factory :contact do    firstname "Cosme"    lastname "Fulanito"    sequence(:email) { |n| "cosme_fulanito#{n}@hackerschool.com"}end
```

Notice that:

- Whenever we called `FactoryGirl.create(:contact)`, the contact name will be called `Cosme`
- We're using sequences for our uniqueness email validation. Sequences are a really handy feature provided by Factory Girl, as you might have guessed they will automatically increment n inside the block, yielding stuff like `cosme_fulanito1@hackerschool.com`, `cosme_fulanito2@hackerschool.com`, and so on
- If we want we can have all our factories in a single file. However the good practice is to have a plural file that mirrors the model name, so for example `spec/factories/contacts.rb` corresponds to `Contact` model


In the previous example we're using strings for all of our attributes but you might also use whatever attribute's data type you want, and that includes integers, boolean, dates and you can even pass Ruby code to dynamically  assign values! Cool huh?


With our contact factory set up, let's return now to our `contact_spec`

```ruby
require 'spec_helper'

describe Contact do
  it "has a valid factory" do
    expect(FactoryGirl.build(:contact)).to be_valid
  end
  ...
end  
```

The former example instantiates (but does not save) a new contact with attributes as assigned by the factory, it then tests that new contact's validity. Compare to the old one, before de factories:

```
it "is valid with a firstname, lastname and email" do
  contact = Contact.new(firstname: 'Cosme', lastname: 'Fulanito', email: 'cosme_fulanito@hackerschool.com')
  expect(contact).to be_validend
```

You can notice that the spec with the factories is a lot more concise. Lets change the following spec so it uses FactoryGirl

```
it "is invalid without a firstname" do
  contact = FactoryGirl.build(:contact, firstname: nil)
  expect(contact).to have(1).errors_on(:firstname)
end
```

The same changes will apply for the following two specs, what about the *"is invalid with a duplicate email address"*, well here is the result:

```ruby
it "is invalid with a duplicate email address" do
  FactoryGirl.create(:contact, email: "cosme_fulanito@hackerschoo.com")
  contact = FactoryGirl.build(:contact, email: "cosme_fulanito@hackerschoo.com")
  expect(contact).to have(1).errors_on(:email)end
```

Our complete `contact_spec` would look like this

```ruby
require 'spec_helper'describe Contact do
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


Our contact spec is looking great! There's just a little something, us as a programmers we hate extra typing, so maybe everytime we type `FactoryGirl.build(:contact)` a kitty might died, and we don't want that do we? Let's use rspec short syntax! To enable it we need to add the following line into our  `rails_helper.rb`

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
require 'spec_helper'

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

Given the former models, the factory for Phone would look like this like this.

```ruby
FactoryGirl.define do
  factory :phone do
    association :contact
    phone '123-555-1234'	 phone_type 'home'  end
end
```

Did you spot the `association` keyword? Well it tells to FactoryGirl to create a new `Contact` on the fly for this phone, but only if we didn't passed specifically a value into the `build` or `create` method.


But hey what about different types of phones? The one for the office, for the home and a mobile one? Well the long way is to do something like this

```ruby
it "allows two contacts to share a phone number" do  create(:phone, phone_type: 'home', phone: "785-555-1234")  expect(build(:phone, phone_type: 'home', phone: "785-555-1234")).to be_validend
```

But FactoryGirl provides us the ability to create *inherited factories, overriding attributes as necessary* 

```
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

And with that the `phone_spec` can be simplified to: 

```ruby
it "allows two contacts to share a phone number" do  create(:home_phone, phone: "785-555-1234")  expect(build(:home_phone, phone: "785-555-1234")).to be_validend
```

#### Generating more realistic fake data

We can improve our test data to make it look more realistic through [faker]() gem, which is a ruby por library for generating fake names, addresses, sentences and more!

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


### Controller specs

There are a few good reasons to explicitly test your controllers methods:

- Controllers are classes with methods too, and in Rails applications, they’re pretty important classes so it’s a good idea to put them on equal footing, as your Rails models.- Controller specs can often be written more quickly than their integration spec counterparts. 
- Controller specs usually run more quickly than integration specs, making them very valuable during bug fixing and checking the bad paths your users can take (in addition to the good ones, of course).


Lets write our first controller specs, dont worry you'll spot a lot of similarities to earlier specs we've written

#### Testing GET requests

As you know in a typical rails controller we'll have four GET-based methods: *index, show, new and edit*. These methods are generally the easis to test, so let's start with them.

```ruby
require 'spec_helper'

describe ContactsController do
  describe 'GET #index' do
    it "populates an array of all contacts" do
      cosme = create(:contact, lastname: 'Fulanito')
      smithers = create(:contact, lastname: 'Smithers')
      get :index
      expect(assigns(:contacts)).to match_array([cosme, smithers])
    end

    it "renders the :index view" do
      get :index
      expect(response).to render_template :index
    end
  end
end   
```

Let's break this down. We’re checking for two things here: 

1. That all the created contacts are retrieved and properly assigned to the specific instance variable. To accomplish this, we're taking advange of the `assigns` method, that verifies that the value assigned in `contacts`is what we expect to see

2. The second expectation may be self-explanatory, thanks to RSpec’s clean, readable syntax: The response sent from the controller back up the chain toward the browserwill be rendered using the `index.html.erb` template.


Also notice that this two simple expectations demonstrate the following key concepts of controller testing:

- The basic DSL for interacting with controler methods. EAch HTTP verb has its own method (in these cases, `get`), which expects the controller method as a symbol (here `:index`), followed by any params (`id: contact`)
- Variables instantiated by the controller method can be evaluated using `assigns(:variable)`
- The finish product return from the controller can be evaluated through response

Let's continue with our spec for the show action:

```ruby
describe 'GET #show' do
  it "assigns the requested contact to @contact" do
    contact = create(:contact)
    get :show, id: contact
    expect(assigns(:contact)).to eq contact
  end

  it "renders the :show template" do
    contact = create(:contact)
    get :show, id: contact
    expect(response).to render_template :show
  end
end
```

As you can see if follows the same basic constructs as the `index` method, let's continue with `new` & `edit` methods

```ruby
describe 'GET #new' do
  it "assigns a new Contact to @contact" do
    get :new
    expect(assigns(:contact)).to be_a_new(Contact)
  end

  it "renders the :new template" do
    get :new
    expect(response).to render_template :new
  end
end
 
describe 'GET #edit' do
  it "assigns the requested contact to @contact" do
    contact = create(:contact)
    get :edit, id: contact
    expect(assigns(:contact)).to eq contact
  end

  it "renders the :edit template" do
    contact = create(:contact)
    get :edit, id: contact
    expect(response).to render_template :edit
  end
end
```  

Reading these examples you will notice that once you learn how to test one typical GET-based method, you can test most of them with a standard set of conventiosn

#### Testing POST requests

One key difference from the GET methods is that instead of the `:id` we passed to the GEt methods, we now need to pass the equivalent of `params[:contact]`, which is the content of the form in which a user would enter a new contact.

Lets build a spec for the `post` method first with valid attributes

```ruby
describe "POST #create" do
  context "with valid attributes" do
    it "saves the new contact in the database" do
      expect{
        post :create, contact: attributes_for(:contact)
      }.to change(Contact, :count).by(1)
    end

    it "redirects to contacts#show" do
      post :create, contact: attributes_for(:contact)
      expect(response).to redirect_to contact_path(assigns(:contact))
    end
  end
  ... 
  # Context with invalid attributes continues right here
end
```

And then let's add the block with invalid attributes

```ruby
context "with invalid attributes" do
  it "does not save the new contact in the database" do
    expect{ 
      post :create, contact: attributes_for(:invalid_contact) 
    }.to_not change(Contact, :count)
  end
  
  it "re-renders the :new template" do
    post :create, contact: attributes_for(:invalid_contact)
    expect(response).to render_template :new
  end
end
```

Important things to notice here:

- Checkout the use of context blocks. Although `describe` and `context` may be used interchangeably, **it's considered best practice to use `context` when describing different states**

- Take a look of how we're passing the full HTTP request inside the `expect` block, in this case the results are evaluated before and after the block, making it simple to determine whether the anticipated change happened or did not happen


#### Testing PUT requests

In this case we need to check the next couple of things:

- It locales the requested @contact
- If the attributes are correctly updated:
	-  we redirect to the show action otherwise
	-  we re-render the edit form

Let's make the context of valid attributes first:

```ruby
describe 'PATCH #update' do
  before :each do
    @contact = create(:contact, firstname: 'Lawrence', lastname: 'Smith')
  end

  context "valid attributes" do
    it "located the requested @contact" do
     patch :update, id: @contact, contact: attributes_for(:contact)
     expect(assigns(:contact)).to eq(@contact)
    end

    it "changes @contact's attributes" do
      patch :update, id: @contact, contact: attributes_for(:contact,
      firstname: "Larry", lastname: "Smith")
      @contact.reload
      expect(@contact.firstname).to eq("Larry")
      expect(@contact.lastname).to eq("Smith")
    end

    it "redirects to the updated contact" do
      patch :update, id: @contact, contact: attributes_for(:contact)
      expect(response).to redirect_to @contact
    end
  end
  ...
  # Context with invalid attributes continues right here
end  
```   

And as we did in the previous POST examples, we need to test that those things don't happen if invalid attributes are passed through the params

```
context "with invalid attributes" do
  it "does not change the contact's attributes" do
    patch :update, id: @contact, contact: attributes_for(:contact, firstname: "Larry", lastname: nil)
    @contact.reload
    expect(@contact.firstname).to_not eq("Larry")
    expect(@contact.lastname).to eq("Smith")
  end

  it "re-renders the edit template" do
    patch :update, id: @contact, contact: attributes_for(:invalid_contact)
    expect(response).to render_template :edit
  end
end
```

Interesting things here:

- Since we're updating an existing Contact, we need to persist something first, as you can see in the `before` hook, we're making sure to assign the persisted Contact to `@contact`to access it later
- We need to call `@reload` on `@contact`to check that our updates are actually persisted


#### Testing DELETE requests

After all that, testing the `destroy` method is relatively straightforward 

```
describe 'DELETE destroy' do
  before :each do
    @contact = create(:contact)
  end

  it "deletes the contact" do
    expect{
      delete :destroy, id: @contact
    }.to change(Contact,:count).by(-1)
  end

  it "redirects to contacts#index" do
    delete :destroy, id: @contact
    expect(response).to redirect_to contacts_url
  end
end
```

The first expectation checks to see if the destroy method in the controller actually deletes the object, and the second expectation confirm that the user is redirected back to the index upon success

#### Testing roles

### Integration tests
