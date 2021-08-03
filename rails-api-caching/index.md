# API Caching in Rails

We mostly use [rails/jbuilder](https://github.com/rails/jbuilder) to render JSON. We find rendering JSON to be one of the slower parts of our API, so caching its results is critical to keeping our site fast.

### Implementation

We created a Rails Concern for caching API responses:

```rb
module Everlane::APICaching
  extend ActiveSupport::Concern

  # Fetch a rendered view from cache, or render it and cache the result.
  # This is a version of `ActiveSupport::Cache::RedisCacheStore#fetch`, enhanced with
  #   * New Relic transaction tagging
  #   * Protection against Redis outages
  def render_cached key, opts = {}, &block
    status = :hit

    view_to_digest =
      if opts.has_key? :view then
        opts.delete :view
      else
        # This is how Rails internally computes the default view inside
        # ActionController and ActionView.
        "#{controller_path}/#{action_name}"
      end

    digest = ActionView::Digestor.digest name: view_to_digest, finder: lookup_context
    key = [view_to_digest, key, digest].flatten.select(&:presence)

    # Expand to a string so we can also log it later
    key = ActiveSupport::Cache.expand_cache_key key

    # Attempt to read from the cache and set the body to nil if it fails
    body = Rails.cache.read(key) rescue nil

    if body.blank?
      status = :miss
      body = block.call

      # Attempt to write to the cache but fail silently
      Rails.cache.write(key, body, opts) rescue nil
    end

    tag_newrelic_transaction "cache-#{status}"

    self.response_body = body
    self.content_type  = 'application/json'
  end
end
```

## Usage: Index

To cache an index view, use the most recent `updated_at` timestamp as the cache key:

```rb
class Api::ProductsController
  include Everlane::APICaching

  def index
    cache_key = Product.maximum(:updated_at)

    render_cached(cache_key) do
      @products = Product.order(updated_at: :desc).limit(10)
      render_to_string
    end
  end
end
```

## Usage: Show

To cache a show view, use the record itself as the cache key:

```rb
class Api::ProductsController
  include Everlane::APICaching

  def show
    @product = Product.find(params[:id])
    render_cached(@product) do
      render_to_string
    end
  end
end
```

`ActiveSupport::Cache.expand_cache_key` will expand that `product` to something like `"products/420-20210420162042000000"`

## Best Practice: Less-Eager Loading

Without caching, a natural way to optimize your SQL queries for a view is to eagerly load associations that the view uses:

```rb
def show
  @product = Product.includes(:category, :inventory_counts).find(params[:id]).
end
```

When we add caching, we need to do the `Product.find()` outside the `render_cached` block because we need its `updated_at` timestamp. The simplest way to do that would be:

```rb
def show
  @product = Product.includes(:category, :inventory_counts).find(params[:id])

  render_cached(@product)
    render_to_string
  end
end
```

But now we're doing more work than necessary before checking the cache. We can move some of that work to the cache to optimize for the cache-hit branch:

```rb
def show
  # get only the fields necessary to calculate the cache key:
  cache_key = Product.select(:id, :updated_at).find(params[:id])

  render_cached(cache_key)
    # do the full query with joins:
    @product = Product.includes(:category, :inventory_counts).find(cache_key.id)
    render_to_string
  end
end
```

## New Relic Traces

This setup lets us compare the cache-hit vs cache-miss versions of an endpoint in New Relic:

![cache hit ~ 50ms](./new-relic-cache-hit.png?raw=true)

![cache miss ~ 125ms](./new-relic-cache-miss.png?raw=true)
