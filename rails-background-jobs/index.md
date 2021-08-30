# Background Jobs in Rails

## Problem

You want to run code periodically or outside of a request-response cycle.
You want the operation to be reliable, resilient, and performant.

## Solution

Define a [Sidekiq](https://sidekiq.org/) worker to perform an action asynchronously.

Below are some of the best practices we like to use when creating and executing Sidekiq workers. See the Sidekiq [documentation](https://github.com/mperham/sidekiq/wiki) for more detailed information.

### Defining a worker

Sidekiq worker classes require two components:

1. It **must** `include Sidekiq::Worker`
2. It **must** have a public `perform` method
3. It may have other options defined with `sidekiq_options`

A basic worker that will enqueue and execute jobs in our default queue looks like:

```rb
# app/workers/my_worker.rb
class MyWorker
  include Sidekiq::Worker
  sidekiq_options queue: :default

  def perform(params)
    # …
  end
end
```

### Running a worker

There are a few different ways to execute workers:

1. `MyWorker.perform_async`
   This will enqueue a job for this worker which will be executed as soon as possible. The amount of time it takes for the job to execute will depend on the number of jobs ahead of it in the queue and how quickly those jobs can be processed.

2. `MyWorker.perform_in`
   This will schedule a job to be enqueued in some number of seconds that you can define.

3. `MyWorker.perform_at`
   This will schedule a job to be enqueued at a specific time that you define.

4. `MyWorker.new.perform`
   This will execute the worker _synchronously_. This is generally not how you will want to execute the worker in production but is useful for executing your worker in unit tests.

Any parameters passed to `perform_*` will be passed into the `perform` instance method.

```rb
class MyWorker
  def perform(name, city)
    # …
  end
end

MyWorker.perform_async('King Zøg', 'Dreamland')

MyWorker.perform_in(420, 'Elfo', 'Dreamland')

MyWorker.perform_at(1.hour.from_now, 'Emperor Cloyd', 'Maru')

MyWorker.new.perform('Prince Derek Grunkwitz', 'Dreamland')
```

### Naming Convention

There are a few naming and organization conventions we like to use to help us easily locate workers and understand their function:

1. All worker classes are defined in the 'app/workers/' directory. This directory can also have subdirectories to namespace workers.
2. All worker classes end in 'Worker'
3. Worker class names should describe the action they perform. For example: 'UpdateFooWorker'

### Periodic jobs

We use [Sidekiq Enterprise](https://sidekiq.org/products/enterprise.html) which gives us access to all of the Sidekiq features. One of these features is scheduling [periodic jobs](https://github.com/mperham/sidekiq/wiki/Ent-Periodic-Jobs).

We use this feature whenever we need a job to execute at a regulary set interval. Periodic jobs are registered using [Cron](https://en.wikipedia.org/wiki/Cron). Whenever a worker class has a job enqueued via the periodic jobs register it will call `perform_async` with no parameters, so you'll need to make sure your `perform` method don't require params. This feature is best used in conjunction with our Enqueue-then-Process Pattern described below.

### Enqueue-then-Process Pattern

Often, we want to perform the same operation on a large number of records. This is particularly true for periodic batch jobs. For these jobs, we want to ensure:

1. Each job executes quickly (see **Time Limit** below)
2. A problem processing one record doesn't cause us to fail to process other records (see **Atomicity & Fragility** below)

We use the **enqueue-then-process** pattern to achieve this. If the job is called with no parameters, it is in **enqueue** mode. If it is called with parameters (often IDs), it is in **process** mode.

```rb
# app/workers/order_confirmation_worker.rb
class OrderConfirmationWorker
  include Sidekiq::Worker

  def process(order_id = nil)
    order_id.nil? ? enqueue_jobs : process_order(order_id)
  end

  private

  def enqueue_jobs
    order_ids = Order.where({ status: Order::CONFIRMED, notified_at: nil }).pluck(:id)
    order_ids.each do |id|
      self.class.perform_async(id)
    end
  end

  def process_order(order_id)
    o = Order.find(order_id)
    MyEmailService.deliver!(:order_confirmation, o)
    o.update!({ notified_at: Time.now })
  end
end
```

This pattern is also a great opportunity for us to use [Bulk Queueing](https://github.com/mperham/sidekiq/wiki/Bulk-Queueing).
By bulk queueing our jobs we can dramatically reduce the number of requests we need to make to Redis to enqueue jobs. This can have a significant impact on execution time when enqueueing hundreds or thousands of jobs.

Here's what the `#enqueue_jobs` method from above would look like if we used bulk queueing:

```rb
def enqueue_jobs
  order_ids = Order.where({ status: Order::CONFIRMED, notified_at: nil }).pluck(:id)

  # Limiting bulk queueing to 1000 jobs at a time
  order_ids.each_slice(1000) do |ids|
    Sidekiq::Client.push_bulk(
      'class' => self.class,
      'args' => ids.map { |id| [id] },
    )
  end
end
```

### Time Limit

Our workers run in Heroku dynos, which Heroku can shut down and cycle at will. Heroku will give the process 30 seconds to exit gracefully before forcefully quitting. If your job takes longer than 30 seconds to run, it may fail. We recommend limiting your jobs to 25 seconds. Use the other techniques in this recipe to ensure your job runs quickly and can recover gracefully.

These same principles apply to other hosting environments, though the details may differ.

### Idempotence

If a job fails or is terminated early, it will automatically be retried.
Make sure it is _idempotent_ — that is, it produces the same result if it
is run once or ten times.

This is idempotent:

```rb
class UpdateFooWorker
  def perform(id, new_params)
    Foo.find(id).update!(new_params)
  end
end

UpdateFooWorker.perform_async my_foo.id, { count: my_foo.count + 1 }
```

This is not:

```rb
class UpdateFooWorker
  def perform(id)
    foo = Foo.find(id)
    foo.update!({ count: foo.count + 1 })
  end
end

IncrementFooWorker.perform_async 420
```

### Batch Your Queries

When querying the database using Rails models, [`find_each`](https://apidock.com/rails/ActiveRecord/Batches/ClassMethods/find_each) is your friend. It automatically performs a query in batches, then yields the results one at a time.

```rb
Product.find_each(:conditions => "deleted_at is null") do |product|
  product.rebuild_product_detail_page!
end
```

### Atomicity & Fragility

We often use jobs to communicate with multiple systems. For example, the `OrderConfirmationWorker` might notify the Email service to send an email and then record the fact that the notification was sent locally.

One way to write this would be

```rb
def process(order_id)
  o = Order.find(order_id)
  o.update!({ notified_at: Time.now })
  MyEmailService.deliver!(:order_confirmation, o)
end
```

It's more likely that `MyEmailService.deliver!` will fail than `o.update!` will, so this implementation creates the risk that we will mark the order as notified, but not actually send the notification.

You can change it to

```rb
def process(order_id)
  o = Order.find(order_id)
  MyEmailService.deliver!(:order_confirmation, o)
  o.update!({ notified_at: Time.now })
end
```

Now the less reliable `MyEmailService.deliver!` is guaranteed to have happened by the time we run `o.update!`. Of course, "more reliable" doesn't mean "perfectly reliable," so now there's a risk that we send the notification but fail to mark the order has having been notified. If we re-run the job, the customer may get the same order confirmation email twice.

Another alternative would be

```rb
def process(order_id, activity = nil)
  return self.send(:"process_#{activity}", Order.find(order_id)) if activity.present?

  self.class.perform_async order_id, :notify
  self.class.perform_async order_id, :mark_notified
end

private

def process_notify(order)
  MyEmailService.deliver!(:order_confirmation, o)
end

def process_mark_notified(order)
  o.update!({ notified_at: Time.now })
end
```

The top-level job will enqueue two sub-jobs: one to send the notification and one to mark the order as having been notified. _Eventually_ both sub-jobs will succeed and the system will be consistent. But there may be a period of time when exactly one has succeeded.

Let's assume (or measure!) that this period of inconsistency won't ever last longer than 6 hours and that it's OK for us to delay sending the customer their order confirmation for those 6 hours. We can combine this sub-job approach with the **Enqueue-then-Process Pattern** above:

```rb
# app/workers/order_confirmation_worker.rb
class OrderConfirmationWorker
  include Sidekiq::Worker

  def process(order_id = nil, activity = nil)
    return enqueue_jobs if order_id.nil?

    order = Order.find(order_id)

    return self.send(:"process_#{activity}", order) if activity.present?

    self.class.perform_async order, :notify
    self.class.perform_async order, :mark_notified
  end

  private

  def enqueue_jobs
    Order.where({
      status: Order::CONFIRMED,  # confirmed orders
      notified_at: nil,          # where we haven't notified the user
      updated_at: ..6.hours.ago, # that haven't been updated in the last 6 hours
    }).find_in_batches do |batch|
      batch.ids.each do |id|
        self.class.perform_async(id, :notify)
        self.class.perform_async(id, :notify)
      end
    end
  end

  def process_notify(order)
    MyEmailService.deliver!(:order_confirmation, order)
  end

  def process_mark_notified(order)
    o.update!({ notified_at: Time.now })
  end
end
```
