## Table of contents

* [Rspec & Rails](#rspec--rails)
	* [Controller specs](#controller-specs)
		* [GET requests](#get-requests)
		* [POST requests](#post-requests)
		* [PUT requests](#put-requests)
		* [DELETE requests](#delete-requests)
	* [Authorization and Roles](#authorization-and-roles)
	* [Integration tests](#integration-tests)
* [Active Job](#active-job)
	* [Sidekiq](#sidekiq) 

## Rspec & rails 

### Controller specs

There are a few good reasons to explicitly test your controllers methods:

- Controllers are classes with methods too, and in Rails applications, they’re pretty important classes so it’s a good idea to put them on equal footing, as your Rails models.- Controller specs can often be written more quickly than their integration spec counterparts. 
- Controller specs usually run more quickly than integration specs, making them very valuable during bug fixing and checking the bad paths your users can take (in addition to the good ones, of course).


Lets write our first controller specs, dont worry you'll spot a lot of similarities to earlier specs we've written

#### GET requests

As you know in a typical rails controller we'll have four GET-based methods: *index, show, new and edit*. These methods are generally the easist to test, so let's start with them.

```ruby
require 'rails_helper'

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

Let's break this down, we’re checking for two things here: 

1. That all the created contacts are retrieved and properly assigned to the `@contacts` variable. To accomplish this, we're taking advange of the `assigns` method, that verifies that the value assigned in `contacts`is what we expect to see

2. The second expectation may be self-explanatory, thanks to RSpec’s clean, readable syntax: The response sent from the controller back up the chain toward the browserwill be rendered using the `index.html.erb` template.


Also notice that this two simple expectations demonstrate the following key concepts of controller testing:

- Each HTTP verb has its own method (in these cases, `get`), which expects the controller action as a symbol (here `:index`), followed by any params (`id: contact`)
- Variables instantiated by the controller method can be evaluated using `assigns(:variable)`
- The finish result return from the controller can be evaluated through response

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

Reading these examples you will notice that once you learn how to test one typical GET-based method, you can test most of them with a standard set of conventions

#### POST requests

One key difference from the GET methods is that instead of the `:id` we passed to the GET methods, for POST requests we need to pass the equivalent of `params[:contact]`, which is the content of the form in which a user would enter a new contact.

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
  ...
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

Really important things to notice here:

- Checkout the use of context blocks. Again just like in the `Contact` model spec, we're using `context`to explain a state, and `describe` for the controller actions

- Also take a look of how we're passing the full HTTP request inside the `expect` block, in this case the results are evaluated before and after the block, making it simple to determine whether the anticipated change happened or did not happen


#### PUT requests

In this case we need to check the next couple of things:

- It locales the requested `@contact`
- If the attributes are correctly updated:
	-  we redirect to the show action,
	-  Otherwise we re-render the edit form

Let's make the context of valid attributes first:

```ruby
describe 'PUT #update' do
  before :each do
    @contact = create(:contact, firstname: 'Cosme', lastname: 'Fulanito')
  end

  context "valid attributes" do
    it "finds the requested @contact" do
     put :update, id: @contact, contact: attributes_for(:contact)
     expect(assigns(:contact)).to eq(@contact)
    end

    it "changes @contact's attributes" do
      put :update, id: @contact, contact: attributes_for(:contact,
      firstname: "Apu", lastname: "Nahasapeemapetilon")
      @contact.reload
      expect(@contact.firstname).to eq("Apu")
      expect(@contact.lastname).to eq("Nahasapeemapetilon")
    end

    it "redirects to the updated contact" do
      put :update, id: @contact, contact: attributes_for(:contact)
      expect(response).to redirect_to @contact
    end
  end
  ...
  # Context with invalid attributes continues right here
end  
```   

Interesting things here:

- Since we're updating an existing Contact, we need to persist something first, as you can see in the `before` block, we're making sure to assign the persisted Contact to `@contact`to access it later

- We need to call `@reload` on `@contact` to check that our updates are actually persisted

And as we did in the previous POST examples, we need to test that those "unhappy" endings (where things go wrong)

```ruby
context "with invalid attributes" do
  it "does not change the contact's attributes" do
    put :update, id: @contact, contact: attributes_for(:contact, firstname: "Apu", lastname: nil)
    @contact.reload
    expect(@contact.firstname).to_not eq("Cosme")
    expect(@contact.lastname).to eq("Fulanito")
  end

  it "re-renders the edit template" do
    put :update, id: @contact, contact: attributes_for(:invalid_contact)
    expect(response).to render_template :edit
  end
end
```

#### DELETE requests

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

The first expectation checks to see if the destroy method in the controller actually deletes the object, and the second expectation confirm that the user is redirected back to the index

## Authorization and roles

*This section assumes you're using [devise](https://github.com/plataformatec/devise) gem in your application* 

Normally most of our application will include authentication, and RSpec could also help us to make sure our controllers do what we expect them to do.

First things first, we'll need to tell rspec that we're using `Devise` as an authentication gem, for that we need to include this line into our `rails_helper` file

```ruby
config.include Devise::TestHelpers, type: :controller
```

And after that add a method to sign_in a user:

```ruby
def sign_in_as_user
  Rails.cache.clear
  @user = create(:user)
  sign_in @user
end
```

Notice that our user factory must have a `password` and a `password_confirmation` associated 

```ruby
# spec/factories/contacts.rb
FactoryGirl.define do
  factory :user, aliases: [:author] do
    sequence(:email){ |n| "awesome_email_#{n}@pm.com"} 
    password "imawesome"
    password_confirmation "imawesome"
  end
end  
```

After that we just need to wrap of all our specs into another describe block 

```ruby
describe ContactsController do
  describe "administrator access" do    before :each do      sign_in_as_user    end
  
    describe 'GET #index' do
     # ...
    end
  
    describe 'GET #new' do
     # ...
    end
  
    describe 'GET #edit' do
     # ...
    end
  
    describe 'GET #show' do
     # ...
    end
  
    describe 'POST #create' do
     # ...
    end
  
    describe 'PUT #update' do
     # ...
    end
  
    describe 'PUT #destroy' do
     # ...
    end
  end
end
```

Notice that we're calling the method `sign_in_as_user` in a before block at the beginning of the describe method. That way a user is sign in in each of our spec.

And what about not authenticated users? Well that's pretty straightforward! Since guest users only have access to `index` action, we'll need to add another block with only this action:

```ruby
describe ContactsController do
  describe "administrator access" do
    # ...
  end
  
  describe "guest access" do
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

Hey but what about happy endings? We shouldn't verify that a guest user doesn't have any access to the other actions? And yes! You're absolutely right!

```ruby
describe ContactsController do
  describe "administrator access" do
    # ...
  end
  
  describe "guest access" do
    describe 'GET #index' do
    end
    
    describe 'GET #new' do
      it "requires login" do
        get :new
        expect(response).to redirect_to login_url
      end
    end
    
    describe 'GET #show' do
      it "requires login" do
        get :show, id: @contact
        expect(response).to redirect_to login_url
      end
    end
    
    describe 'GET #edit' do
      it "requires login" do
        get :edit, id: @contact
        expect(response).to redirect_to login_url
      end
    end
    
    describe 'POST #create' do
      it "requires login" do
        post :create, contact: attributes_for(:contact)
        expect(response).to redirect_to login_url
      end
    end
    
    describe 'PUT #update' do
      it "requires login" do
        put :update, id: @contact, contact: attributes_for(:contact)
        expect(response).to redirect_to login_url
      end
    end
    
    describe 'DELETE #destroy' do
      it "requires login" do
        delete :destroy, id: @contact
        expect(response).to redirect_to login_url
      end
    end
  end
end
```

But hey did you notice that we are repeating the same code in `describe "GET 'index'"` for "AdministratorAccess" and "GuestAccess", we can clean that up with `shared_example`

```ruby
describe ContactsController do
  shared_examples("public access to contacts") do
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
  
  describe "administrator access" do
    it_behaves_like "public access to contacts"
    describe 'GET #new' do
     # ...
    end
  
    describe 'GET #edit' do
     # ...
    end
  
    describe 'GET #show' do
     # ...
    end
  
    describe 'POST #create' do
     # ...
    end
  
    describe 'PUT #update' do
     # ...
    end
  
    describe 'PUT #destroy' do
     # ...
    end
  end
  
  describe "guest access" do
    it_behaves_like "public access to contacts"
    describe 'GET #new' do
     # ...
    end
  
    describe 'GET #edit' do
     # ...
    end
  
    describe 'GET #show' do
     # ...
    end
  
    describe 'POST #create' do
     # ...
    end
  
    describe 'PUT #update' do
     # ...
    end
  
    describe 'PUT #destroy' do
     # ...
    end
  end
end
```

### Integration tests

So far we’ve added a good amount of test coverage to our contacts test suite. 

We got:
- RSpec installed and configured
- set up some unit tests on models and controllers,- used factories to generate test data. 

Now it’s time to put everything together for integration testing–in other words, **making sure those models and controllers all play nicely with other models and controllers in the application**. These tests are calledfeature specs in RSpec. You may also hear them called acceptance tests

For that we'll use [capybara](https://github.com/jnicklas/capybara) & [launchy](https://github.com/copiousfreetime/launchy)


Capybara lets you simulate how a user would interact with your application through a web browser, using a series of easy-to-understand methods like `click_link`, `fill_in`, and `visit`. 

For this section we'll see a live demo on class, so don't miss it :)


## Active Job

Almost any modern application has the need for a variety of queueing services: email handling, scheduling newsletters or even database housekeeping tasks, [Active Job](http://edgeguides.rubyonrails.org/active_job_basics.html) can help you with all those tasks and more. The main point of this library is to ensure a job infraestructure for all rails applications, and for that infraestructure to be a place where you can put other gems on top, withouth having to worry about API differences.

Active Job was introduced in Rails 4.2 so you need this or an ealier version of rails. As with everything in rails, there's a generator:

```bash
$ rails g job Mailer
$ create  app/jobs/mailer_job.rb
```

Here's what the file would look like

```ruby
class MailerJob < ActiveJob::Base
  queue_as :default

  def perform(*args)
  end
end
```


So if you want to send an email in a background job, you'll have to do something like this:

```ruby
class MailerJob < ActiveJob::Base
  queue_as :default

  def perform user_id
  	user = User.find(user_id)
  	UserMailer.send_activation_mailer(user).deliver
  end
end
```

Note: 

- You can define `perform` with as many arguments as you want
- It's considered a good practice to use `id`s for your jobs instead of ruby objects, since ruby object information could change between the moment the job is scheduled and the actual moment when the code is executed

And the you can set the job inside your controller/model:



```ruby
MailerJob.perform_later(@user.id)
```


You can also set a different time for the jobs like:

```ruby
MailerJob.set(wait: 1.week.from_now).perform_later(@user.id)
```


### Sidekiq

Like we said ActiveJob can use whichever job runners, so let's use the amazing [sidekiq](http://sidekiq.org). 

First things first, Sidekiq depends on [redis](http://redis.io), so we must install this dependency first. If you are using Mac OS X and Homebrew simply type the following command in your terminal 

```bash
brew install redis
```

You can configure to launch redis when your computer starts with

```bash
$ ln -sfv /usr/local/opt/redis/*.plist ~/Library/LaunchAgents
```

Or can turn it on manually with:

```bash
$ redis-server /usr/local/etc/redis.conf
```

After that we just need to tell our  application that Sidekiq is going to be our multiple queuing backend, and for that lets add the following line inside `config/application.rb` file

```ruby
# config/application.rb
module YourApp
  class Application < Rails::Application
  
    config.active_job.queue_adapter = :sidekiq
  end
end
```

Now as usual we need to add it to our `Gemfile` and run `bundle install`:

```ruby
gem 'sidekiq' 
gem 'sinatra', :require => nil
```

Finally add the following in `config/routes.rb` file:

```ruby
require 'sidekiq/web'
mount Sidekiq::Web => '/sidekiq'
```

And restart your app.

To start sidekiq you need to open another tab in your terminal and run the following command:

```bash
$ bundle exec sidekiq
```

You'll see an output like this:

```bash


         m,
         `$b
    .ss,  $$:         .,d$
    `$$P,d$P'    .,md$P"'
     ,$$$$$bmmd$$$P^'
   .d$$$$$$$$$$P'
   $$^' `"^$$$'       ____  _     _      _    _
   $:     ,$$:       / ___|(_) __| | ___| | _(_) __ _
   `b     :$$        \___ \| |/ _` |/ _ \ |/ / |/ _` |
          $$:         ___) | | (_| |  __/   <| | (_| |
          $$         |____/|_|\__,_|\___|_|\_\_|\__, |
        .d$$                                       |_|

2016-02-20T03:50:25.656Z 48153 TID-oxo1mlzpk INFO: Running in ruby 2.2.1p85 (2015-02-26 revision 49769) [x86_64-darwin14]
2016-02-20T03:50:25.656Z 48153 TID-oxo1mlzpk INFO: See LICENSE and the LGPL-3.0 for licensing details.
2016-02-20T03:50:25.656Z 48153 TID-oxo1mlzpk INFO: Upgrade to Sidekiq Pro for more features and support: http://sidekiq.org
2016-02-20T03:50:25.656Z 48153 TID-oxo1mlzpk INFO: Booting Sidekiq 4.0.2 with redis options {:url=>nil}
```

Now go to `localhost:3000/sidekiq` and watch the awesomeness

![sidekiq](sidekiq.png)

To leave sidekiq simply type `ctrl` + `c`