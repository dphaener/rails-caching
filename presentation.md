class: center, middle, inverse

# Low-Level Caching in Rails

## Darin Haener

---

class: center, middle, inverse

# What is low-level caching?

Essentially, this is caching at the very lowest level imaginable, so that
you can maintain very fine grained control over your caching mechanisms.

When implementing caching in Drip, we realized that the standard caching that
Rails provides would not be sufficient for our use case.

---

class: middle, inverse

# High level caching

Rails provides some very nice, easy to use high level caching mechanisms that
makes it easy to get started right away:

```erb
<% Order.find_recent.each do |o| %>
  <%= o.buyer.name %> bought <%= o.product.name %>
<% end %>

All available products:
<% Product.all.each do |p| %>
  <% cache(p) do %>
    <%= link_to p.name, product_url(p) %>
  <% end %>
<% end %>
```

--

This is nice, and only caches the portion of the page within the cache block.
You can see that we are using each instance of the Product class `p` as a
cache key, which provides very simple cache expiration.

---

class: middle, inverse

# Model based cache expiration

In the previous scenario, we used the model instance as a cache key.
Behind the scenes, a method called cache_key will be invoked on the model and
it returns a string like products/23-20130109142513. This is comprised of the
model name `products`, the id of the record `23`, and the `updated_at`
timestamp.

--

For most every day use cases this is perfectly acceptable, and very easy to
implement. But what if you have a presentation layer that relies on data that
spans multiple tables, requiring complex joins and data aggregation?

--

This is the scenario we faced when implementing caching in Drip. This is where
low level caching comes into play.

---

class: inverse

# Implementing a Cacheable module

```ruby
module Cacheable
  def get_or_set_cache(cache_key, expires_in: 1.hour)
    Rails.cache.fetch(cache_key, expires_in: expires_in) do
      yield
    end
  end
end
```
--

In this simple module, all we are doing is creating a
method that will yield a block. The `Rails.cache.fetch` method is surprisingly
simple. It checks the cache store for the key provided, and if it exists,
returns the value at the key, otherwise it executes the code block and stores
the return value for that cache key.

---

class: inverse

# Using this module

Now that we've implemented a reusable module that we can use to extend any class
let's put it to use.

--

```ruby
class ProductAnalytics
  include Cacheable

  def base_cache_key
    "#{self.class.name}/#{timezone}/#{start_at}/#{end_at}"
  end

  def product_sales_analysis
    get_or_set_cache([base_cache_key, "product_sales"], expires_in: 1.hour) do
      sql = <<-SQL
        SELECT sum(price) from products p
        INNER JOIN product_sales ps
          ON ps.product_id = p.id
        WHERE ps.price > 20
        AND ps.created_at BETWEEN '#{start_at}' AND '#{end_at}'
      SQL

      ActiveRecord::Base.connection.select_all(sql)
    end
  end
end
```

---

class: inverse

# Taking the module farther

This was just a simple implementation, but now that we have an extendable
module, we can use it to create a more complex implementation.

--

```ruby
module Cacheable
  def get_or_set_cache(cache_key, expires_in: expiration_time)
    Rails.cache.fetch(cache_key, expires_in: expires_in) do
      yield
    end
  end

  def expiration_time
    1.hour
  end
end
```

--

We can add methods and classes to extend this module and make complex
decisions on how and when to expire our cache.

---

class: inverse

# Extending the analytics class

Now that we have this default `expiration_time` method setup, we can easily
override it if we want in the `ProductAnalytics` class

--

```ruby
class ProductAnalytics
  include Cacheable

  def base_cache_key
    "#{self.class.name}/#{timezone}/#{start_at}/#{end_at}"
  end

  def expiration_time
    [Product.count / 100, 1].max.hours
  end

  def product_sales_analysis
    get_or_set_cache([base_cache_key, "product_sales"]) do
      sql = <<-SQL
        ...
      SQL

      ActiveRecord::Base.connection.select_all(sql)
    end
  end
end
```

---

class: center, middle, inverse

# Wrap up

Just implementing a simple algorithm to dynamically set the cache expiration
like this can become extremely powerful, and give you full control over all
of your caching mechanisms.

---

class: middle, center, inverse

# Thanks!!!
