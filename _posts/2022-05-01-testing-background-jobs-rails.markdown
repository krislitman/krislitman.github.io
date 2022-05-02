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

Since the addition of a callback method in the user model (after_create), the test will need to account for enqueuing a job once a new user is created. First add the [rspec-rails gem](https://github.com/rspec/rspec-rails) to the Gemfile, afterwards be sure to run `bundle install`. To finish setting up RSpec, run the command `rails g rspec:install`. That will create a /spec folder with spec_helper & rails_helper ruby files. Create a basic test file for our user in this file "spec/models/user_spec.rb".

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

Running the test as-is will fail, due to the job being created but not fully processed. We can see this in the stack trace:

```
Failure/Error: expect(user.user_logs.size).to eq(1)

       expected: 1
            got: 0
```

To fix this, we will need to add in some test helper methods. To get access to these methods we will need to include `ActiveJob::TestHelper`. That will provide the method #perform_enqueued_jobs that will process the job we need for this test to pass. With these changes the test will now pass:

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

**In Progress**

### References:

[ActiveJob Testing](https://guides.rubyonrails.org/testing.html#testing-jobs)
