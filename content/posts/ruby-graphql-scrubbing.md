---
title: "Scrubbing GraphQL Params in Ruby & Rails"
tags: ["graphql", "ruby"]
date: 2019-04-27T17:09:42-05:00
draft: false
summary: How to filter out sensitive params from logs with graphql-ruby.
---
## Keep your logs clean
Filtering or scrubbing sensitive information from logs is an important security measure. It's not hard and can quickly be done in rails _and_ GraphQL. Don't find yourself in [twitter's shoes](https://blog.twitter.com/official/en_us/topics/company/2018/keeping-your-account-secure.html).

## Normal filtering in Rails

Rails presents params in logs as such:

```bash
Started POST "/users" for ::1 at 2019-04-27 12:00:00 -0800
Processing by UsersController#create as HTML
  Parameters: {"utf8"=>"✓", "authenticity_token"=>"lbASdvai82avl", "user"=>{"first_name" => "Thor", "password"=>"AsgardRulz"}, "commit"=>"Create user"}
```

Filtering params in rails is as easy as adding this to an initializer:

```ruby
Rails.application.config.filter_parameters += [:password]
```

Which will result in this log output:
```bash
Started POST "/users" for ::1 at 2019-04-27 12:00:00 -0800
Processing by UsersController#create as HTML
  Parameters: {"utf8"=>"✓", "authenticity_token"=>"lbASdvai82avl", "user"=>{"first_name" => "Thor", "password"=>"[FILTERED]"}, "commit"=>"Create user"}
```
## The GraphQL side of things

This isn't going to work for GraphQL queries because everything will be serialized into a single `query` param. To srub the output, we'd need to parse the query. Luckily, [graphql-ruby](https://github.com/rmosolgo/graphql-ruby) has a class to help us out!

Before GraphQL executes a query, it must first parse the query string to a document. All we have to do is specify a special printer for the document like this:

```ruby
document = GraphQL.parse(query_string)
document.to_query_string(printer: GraphQLScrubber.new)
```
 To hook this up to rails, we need to add it to our initializer:

 ```ruby
 Rails.application.config.filter_parameters +=
  [
    lambda do |key, value|
      value.replace(filtered_gql_query(value)) if key.to_s == 'query'
    rescue GraphQL::ParseError
    end
  ]

def filtered_gql_query(value)
  document = GraphQL.parse(value)
  document.to_query_string(printer: GraphQLScrubber.new)
end
 ```

 Now we can handle all the magic inside our custom printer:
```ruby
class GraphQLScrubber < GraphQL::Language::Printer
  FILTERED_VALUE = '[FILTERED]'.freeze
  FILTERED_ARGS = %w[password].freeze

  def print_argument(arg)
    if FILTERED_ARGS.include?(arg.name)
      "#{arg.name}: #{FILTERED_VALUE}"
    else
      super
    end
  end
end
```
And finally, our query will look like this:

```bash
Started POST "/graphql" for ::1 at 2019-04-29 14:14:31 -0500
Processing by GraphqlController#execute as */*
  Parameters: {"query"=>"mutation {\n  createUser(password: [FILTERED]) {...}"}
```

There's a small leap here from a normal REST endpoint to GraphQL, but the gist is the same. HMU on [twitter](https://twitter.com/chevinbrown) if you have questions!
