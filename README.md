# GraphQL Decorate

`graphql-decorate` adds an easy-to-use interface for decorating types in `graphql-ruby`. It lets 
you move logic out of your type files and keep them declarative. 

## Installation

Add this line to your application's Gemfile:

```ruby
gem 'graphql-decorate'
```

And then execute:

    $ bundle

Or install it yourself as:

    $ gem install graphql-decorate

Once the gem is installed, you need to add the integrations to your base type and field classes. 
```ruby
class BaseType < GraphQL::Schema::Object
  extend GraphQL::Decorate::ObjectIntegration
end

class BaseField < GraphQL::Schema::Field
  include GraphQL::Decorate::FieldIntegration
end
```

## Usage

### Basic use case
```ruby
class Rectangle
  attr_reader :length

  def initialize(length)
    @length = length
  end
end

class RectangleDecorator < BaseDecorator
  def area
    length * 2
  end
end

class Rectangle < GraphQL::Schema::Object
  decorate_with RectangleDecorator
  
  field :area, Int, null: false
end
```
In this example, the `Rectangle` type is being decorated with a `RectangleDecorator`. Whenever a 
`Rectangle` gets resolved in the graph, the underlying object will be wrapped with a 
`RectangleDecorator`. All of the methods on the decorator are accessible on the type.

### Decorators
By default, `graphql-decorate` is set up to work with Draper-style decorators. These decorators 
provide a `decorate` method that wraps the original object and returns an instance of the 
decorator. They can also take in additional metadata.
```ruby
RectangleDecorator.decorate(rectangle, context: metadata)
```
If you are using a different decorator pattern then you can override this default behavior in 
the configuration.
```ruby
GraphQL::Decorate.configure do |config|
  config.decorate do |decorator_class, object, _metadata|
    decorator_class.decorate_differently(object)
  end
end
```

### Types
Three methods are made available on your type classes
#### decorate_with
`decorate_with` accepts a decorator class that will decorate every instance of your type.
```ruby
class Rectangle < GraphQL::Schema::Object
  decorate_with RectangleDecorator
end
```

#### decorate_when
`decorate_when` accepts a block which yields the underlying object. If you have multiple 
possible decorator classes you can return the one intended for the underling object.
```ruby
class Rectangle < GraphQL::Schema::Object
  decorate_when do |object|
    if object.length == object.width
      SquareDecorator
    else
      RectangleDecorator
    end
  end
end
```
`decorate_when` also optionally yields the current GraphQL `context`.
```ruby
class Rectangle < GraphQL::Schema::Object
  decorate_when do |object, context|
    # Do something with context
  end
end
```
#### decorator_metadata
`decorator_metadata` accepts a block which yields the underlying object. If your decorator pattern 
allows additional metadata being passed into the decorators, you can define it here.
```ruby
class Rectangle < GraphQL::Schema::Object
  decorator_metadata do |object|
    {
      name: object.name
    }
  end
end
```
`RectangleDecorator` will be initialized with a metadata of `{ name: <object_name> }`.
`decorator_metadata` also optionally yields the current GraphQL `context`.
```ruby
class Rectangle < GraphQL::Schema::Object
  decorator_metadata do |object, context|
    # Do something with context
  end
end
```

#### scoped_decorator_metadata
`scoped_decorator_metadata` functions similarly to `decorator_metadata`. Each decorated type with 
the `scoped_decorator_metadata` block defined will apply additional metadata to the decorator. The 
difference is that _every child field_ of the type will also receive the same metadata from 
further up in the tree. If a child field redefines the `scoped_decorator_metadata` then it will 
replace anything given from above and propagate the new metadata. This is useful for decorators 
that might depend on state from a field far above them.
```ruby
class Rectangle < GraphQL::Schema::Object
  scoped_decorator_metadata do |object|
    {
      name: object.name
    }
  end
  
  field :detailed_properties, DetailedProperties, null: false
end
```
`RectangleDecorator` will be initialized with a metadata of `{ name: <object_name> }`. 
The decorator of `DetailedProperties` will also be initialized with a metadata 
of `{ name: <object_name> }`. `scoped_decorator_metadata` also optionally yields the current GraphQL 
`context`.

```ruby
class Rectangle < GraphQL::Schema::Object
  scoped_decorator_metadata do |object, context|
    # Do something with context
  end
end
```

#### Combinations
You can mix and match these methods to suit your needs. Note that if `decorate_with` and 
`decorate_when` are both provided that `decorate_with` will take precedence.
```ruby
class Rectangle < GraphQL::Schema::Object
  decorate_with RectangleDecorator
  decorator_metadata do |object|
    {
      name: object.name
    }
  end
end
```

### Collections
By default `graphql-decorate` recognizes `Array` and `ActiveRecord::Relation` object types and 
decorates every element in the collection. If you have other collection types that should have 
their elements decorated, you can add them in the configuration. Custom collection classes must 
respond to `#map`.
```ruby
GraphQL::Decorate.configure do |config|
  config.custom_collection_classes = [Mongoid::Relations::Targets::Enumerable]
end
```

## Development

After checking out the repo, run `bin/setup` to install dependencies. Then, run `rake spec` to run the tests. You can also run `bin/console` for an interactive prompt that will allow you to experiment.

## License

The gem is available as open source under the terms of the [MIT License](https://opensource.org/licenses/MIT).
