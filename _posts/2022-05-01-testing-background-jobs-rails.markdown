---
layout: post
title: Testing Background Jobs in Rails
date: 2022-05-01
description: How to test background jobs in a Rails application
img: test.gif
tags: [Ruby, Rails, Sidekiq, ActiveJob, TDD, RSpec]
---

In my last post I set up a background job implementation in a Rails application, you can read that [here](https://krislitman.github.io/background-jobs-rails/). For testing these changes, we will need to make adjustments to the User model, and add in a new test for the UserLogCreateJob class.

### User Model Test

Since the addition of a callback method in the user model (after_create), the test will need to account for enqueuing a job once a new user is created. First add the [rspec-rails gem](https://github.com/rspec/rspec-rails) to the Gemfile, afterwards be sure to run `bundle install`. To finish setting up RSpec, run the command `rails g rspec:install`. That will create a /spec folder with spec_helper & rails_helper ruby files. Create a basic test file for our user under the **spec** directory, in this file "spec/models/user_spec.rb".

```
require "rails_helper"

RSpec.describe User, type: :model do

    after(:all) do
        User.destroy_all
        UserLog.destroy_all
    end

    context "Relationships" do
        it "Has User Logs After Create" do
            user = User.create

            expect(user.user_logs.size).to eq(1)
            expect(user.user_logs.first).to be_an_instance_of(UserLog)
            expect(user.user_logs.first.user_id).to eq(user.id)
        end
    end
end
```

Running the test as-is will fail, due to the job not being processed yet so the UserLog has not been created. We can see 0 returning for the user_logs size in the stack trace:

```
Failure/Error: expect(user.user_logs.size).to eq(1)

       expected: 1
            got: 0
```

To fix this, we will need to add in test helper methods. Including `ActiveJob::TestHelper` within the test will give us access to certain methods that will make it possible to process the enqueued job. See more methods available [here](https://api.rubyonrails.org/v7.0.2.4/classes/ActiveJob/TestHelper.html). Adding in the method #perform_enqueued_jobs will process the job we need for this test to pass. With these changes the test will now pass:

```
require "rails_helper"

RSpec.describe User, type: :model do
    include ActiveJob::TestHelper

    after(:all) do
        User.destroy_all
        UserLog.destroy_all
    end

    context "Relationships" do
        it "Has User Logs After Create" do
            user = User.create
            perform_enqueued_jobs

            expect(user.user_logs.size).to eq(1)
            expect(user.user_logs.first).to be_an_instance_of(UserLog)
            expect(user.user_logs.first.user_id).to eq(user.id)
        end
    end
end
```

In addition to a model test, I'm going to create a specific test for our job in **spec/jobs/user_log_create_job_spec.rb**. After instantiating a new User, we can assert that a job to create a new user_log record has been enqueued. A basic test setup using User & UserLog models will look something like this:

```
require "rails_helper"

RSpec.describe UserLogCreateJob do
    include ActiveJob::TestHelper
    after(:each) do
        User.destroy_all
        UserLog.destroy_all
    end

    context "UserLogCreateJob" do
        it "Is enqueued successfully" do
            assert_enqueued_with(job: UserLogCreateJob) do
                User.create
            end
        end
    end
end
```
**In Progress**

### References:

[ActiveJob Testing](https://guides.rubyonrails.org/testing.html#testing-jobs)
