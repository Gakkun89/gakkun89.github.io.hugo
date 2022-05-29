+++
title = "Presence in Rails"
date = "2022-05-29"
tags = ["presence", "Rails", "Ruby"]
showFullContent = true
readingTime = false
hideComments = false
+++

Some of the most common methods I use on a daily basis when writing `if/else`'s and [guard clauses](https://devblast.com/b/what-are-guard-clauses) are 'presence checkers'.
It can be really helpful to know if the `Array`, `String`, `object` etc. has is not just 'empty'/`[]`/`false`/`nil`as your existing logic may fail if passed such data. However Ruby (and Rails) have a number of methods to achieve this and as a junior dev I often found myself checking which to use in each specific scenario.

## `nil?` `blank?` `empty?` which one to use?
The methods I come back to the most are `blank?` and `present?` (which is actualy just an alias for `!blank?`) because they offer an extra layer of safety in their return:

|          | blank?  | present? |
|----------|---------|----------|
| nil      | true    | `false`  |
| false    | true    | `false`  |
| true     | `false` | true     |
| 0        | `false` | true     |
| 1        | `false` | true     |
| ""       | true    | `false`  |
| " "      | true    | `false`  |
| []       | true    | `false`  |
| [nil]    | `false` | true     |
| {}       | true    | `false`  |
| {a: nil} | `false` | true     |

Either way you are guarenteed a `true` or `false` response and can build your logic around that boolean.
Other methods are more restrictive:

- `nil?` will only return `true` when passed `nil` any other value results in `false` which restricts its usefulness to certain situations.

- `empty?` can only be used on `Enumerables` that is `Hash`'s (`{}`) and `Array`'s (`[]`) and `String`'s (`" "`), trying to use it on an `Integer` will result in:

```ruby
=> 0.empty?
NoMethodError: undefined method `empty?' for 0:Integer
```

- There is even `zero?` which is restricted to just `Integers` and responds `true` on 0 and `false` on any other number.

- Rails, more specifically `ActiveRecord` offers `exists?` which when used against an `ActiveModel::Model` can be used for checking the DB for the existance of a model that matches the parameters passed:
```ruby
# For a given model of 'Dog'
    Dog.exists?(5) # Checks for the ID of the model in the DB
    Dog.exists?('5') # See above
    Dog.exists?(['name LIKE ?', "%#{query}%"]) # Specify other attributes to query for
    Dog.exists?(name: 'Good Boy') # See above
    Dog.exists?(id: [1, 4, 8]) # Check multiple IDs at once
    Dog.exists?(false) # Return False
    Dog.exists? # Checks if there are any instances of Dog in the DB
    Dog.where(name: 'Skye', good_boy: true).exists? # More specific querying is possible
```
[Further reading on `exists?`](https://api.rubyonrails.org/classes/ActiveRecord/FinderMethods.html#method-i-exists-3F)

# Performance and presence checking

Unless you are Twitter scale it doesn't matter.

Of course there are differences between using the different methods against different data types ([as can be seen here in this comprehensive SO answer](https://stackoverflow.com/a/20814251/8459243)). However for most projects these presence checks will not be a performance bottle neck.

For example my favoured `empty?` is actually 2x slower than using `empty?`, this can easily be seen by looking at the [source code](https://github.com/rails/rails/blob/e2efc667dea886e71c33e3837048e34b7a1fe470/activesupport/lib/active_support/core_ext/object/blank.rb#L18):
```ruby

def blank?
  respond_to?(:empty?) ? !!empty? : !self
end
```

However in most projects this performance hit will not be noticable.

One caveat to the above statement is when dealing with `ActiveRecord` checks which hit the DB. The topic of performance in those queries is quite involved and is expertly handled in [this post](https://www.ombulabs.com/blog/benchmark/performance/rails/present-vs-any-vs-exists.html) which I would encourage anyone interested to read.

`Thats all folks`
```ruby
LinkedIn::Messenger.send(`Mike Warren`)  if questions.present?
```