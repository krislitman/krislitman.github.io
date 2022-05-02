---
layout: post
title: Background Jobs in Rails
date: 2021-10-29
description: How to setup background jobs in a Rails application
img: sidekiq.png
tags: [Ruby, Rails, Sidekiq, ActiveJob, Redis]
---

After a few months now working in a production Ruby environment (not Rails) with a sophisticated background job setup, I wanted to step back and dive into how to implement a simple background job setup myself with Rails. Background jobs are typically used to run processes like sending an email, or processing a batch of data so that the user experience isn’t interrupted and the page can still load. The jobs are sent to a queue, where they are processed based on priority and whatever else may already be enqueued. I’m going to detail some steps to get started with Active Job, Sidekiq & Redis in a new Rails 6 application, creating users, user_logs and pushing a job on the queue when a user is created.

**rails new background_job_practice -d=’postgresql’ -T — skip-spring**

For this example, the Rails application will be setup using a Postgresql database, skipping the default test suite (will install RSpec for an example test with Background Jobs in the next part), and skipping spring auto-loader configurations. I’ll list some commands below to get started with some quick setup:

<ul>
    <li>rails g scaffold users - to create our user model, views, controller, migration</li>
    <li>rails db:{create,migrate} - to create & migrate the database</li>
</ul>

When we start up a Rails server and visit the users index, we can see that we are now able to do some basic crud actions with users.

**Run: rails s & visit: localhost:3000/users**

I’m going to create another model as well: UserLogs, that way whenever a User is created, the background job can create a user_log instance while we are redirected back to the users index. For now the user_log will just have created_at, updated_at, and user_id fields. Commands to run:

<ul>
    <li>rails g migration CreateUserLogs user:references</li>
    <li>rails db:migrate</li>
</ul>

Will also add a user_log.rb file under app/models. After running migrations the schema should now look like this:

```
ActiveRecord::Schema.define(version: 2021_10_26_023228) do

  # These are extensions that must be enabled in order to support this database
  enable_extension "plpgsql"

  create_table "user_logs", force: :cascade do |t|
    t.bigint "user_id", null: false
    t.datetime "created_at", precision: 6, null: false
    t.datetime "updated_at", precision: 6, null: false
    t.index ["user_id"], name: "index_user_logs_on_user_id"
  end

  create_table "users", force: :cascade do |t|
    t.datetime "created_at", precision: 6, null: false
    t.datetime "updated_at", precision: 6, null: false
  end

  add_foreign_key "user_logs", "users"
end
```

The app setup is in place, so I’m going to create the first job. You can use a Rails generator and run this command: rails g job user_log_create, which will give us a new file in app/jobs, user_log_create.rb. The queue_as option can be configured to different options such as :low_priority, :high_priority, depending on what you need for the job.

```
class UserLogCreateJob < ApplicationJob
  queue_as :default

  def perform(*args)
    # Do something later
  end
end
```

Instead of taking in *args, we can change the parameters we are accepting in the perform method to take in the user_id of the user that is created. Instead of placing the logic to create a new UserLog instance in say a UserLogController create method, we can essentially do the same thing within this perform method.

Under app/models in the user.rb file, we can add a callback after_create, that will run the background job after a user is created. Within the block, you will call the name of the job class we created UserLogCreate followed by perform_later(self.id). The perform_later method passes the user_id as an argument, and then pushes the job onto the queue. You could also use the method perform_now(*args) if you wanted to run the job right away, skipping the queue altogether.

```
class User < ApplicationRecord
  after_create do
    UserLogCreateJob.perform_later(self.id)
  end
end
```

Within the UserLogCreateJob class, you can use a callback to create logs for when jobs are completed as well.

```
class UserLogCreateJob < ApplicationJob
  queue_as :low

  after_perform do |job|
    Rails.logger.info "#{Time.now}: Job completed. #{job.inspect}"
  end

  def perform(user_id)
    UserLog.create(
      user_id: user_id,
    )
  end
end
```

Before running the server and testing the functionality, you’ll need to add the gem ‘sidekiq’ to the Gemfile and then bundle install. Inside of config/application.rb, the adapter will need to be updated to sidekiq as well with this line: config.active_job.queue_adapter = :sidekiq. To get everything running so that the background jobs are correctly sent to the queue, you will need to run these commands to get the Rails server, Redis Server, and Sidekiq running:

<ul>
    <li>rails s</li>
    <li>redis-server</li>
    <li>bundle exec sidekiq -q default</li>
</ul>

The q flag can be followed by different priorities such as high, low, default. Now when a new user is created in the browser, a new user_log instance is also made. You can see that everything ran successfully in the Sidekiq server.

#### Helpful Docs:

[Rails Guides](https://guides.rubyonrails.org/active_job_basics.html)
<br>
[Sidekiq Github](https://github.com/mperham/sidekiq/wiki/Active+Job)
<br>
[API Doc](https://apidock.com/rails/search?query=active%20job)

