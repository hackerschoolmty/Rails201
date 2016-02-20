## Table of contents

* [Recursive associations](#recursive-associations) 
* [Active Job](#active-job)
	* [Sidekiq](#sidekiq) 


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


## Active Job

Almost any modern application has the need for a variety of queueing services: email handling, scheduling newsletters or even database housekeeping tasks, [Active Job](http://edgeguides.rubyonrails.org/active_job_basics.html) can help you with all those tasks and more. The main point of this library is to ensure a job infraestructure for all rails applications, and for that infraestructure to be a place where you can put other gems on top, withouth having to worry about API differences.

Active Job was introduced in Rails 4.2 so you need this or an ealier version of rails. 
m without having to rewrite your jobs


As with everything in rails, there's a generator:

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

```
$ ln -sfv /usr/local/opt/redis/*.plist ~/Library/LaunchAgents
```

Or can turn it on manually with:

```
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