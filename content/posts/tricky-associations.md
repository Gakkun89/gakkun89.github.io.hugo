+++
title = "Dude where's my association?"
date = "2022-06-09"
tags = ["ActiveRecord", "Associations"]
showFullContent = true
readingTime = false
hideComments = false
+++

I ran into an issue I had never faced recently when validating changes made to associated models.

### Context:  
Let's suppose we have the following models (non-related info removed):
```ruby
class User < ApplicationRecord
  has_many :rules
  has_many :activities
end
```
```ruby
class Rule < ApplicationRecord
  belongs_to :user
  def check!(activity)
    #PSEUDO CODE
    unless allowed_actions.include?(activity.action) && allowed_tags.include?(activity.tags)
      activity.add_error("Not allowed to do this activity")
    end
  end
end
```
```ruby
class Activity < ApplicationRecord
   belongs_to :user
   has_and_belongs_to_many :tags
   validates_with ActivityValidator
end
```
```ruby
class Tag < ApplicationRecord
   has_and_belongs_to_many :activities
   # Tags contain information regarding the activity and rules can 'invalidate' an activity based on certain tags
   # E.g Rule does not allow User X to do any activities tagged with 'dangerous'
end
```
```ruby
class ActivityValidator < ActiveModel::Validator
  IGNORED_ATTRIBUTES = ["status"]
  def validate(activity)
    # Allow certain changes without validating rules again
    no_changes_present   = (activity.changed_attributes.keys - IGNORED_ATTRIBUTES).empty? 
    return if no_changes_present
    activity.user.rules.each do |rule|
      rule.check!(activity)
    end
  end
end
```

### Issue
Maybe the more seasoned Rails devs have already noticed the problem.

The issue I faced was that when changing any of the attributes on the activity (excluding the ignored `status`) the `Rule` was correctly evaluated. However, if there was a `Rule` which was based on checking `Tag`s when making any changes to just the associated tags the validator passed even with 'invalid' `Tag`s.

What was going on? Why aren't my associations being validated?

First port of call was to slap in a `binding.pry` to see what `no_changes_present` and to my suprise it was returning `true` even when I was either deleting or adding `Tag`s.

It was at this point I found out that `activity.changed_attributes` (and by extension `#changes`) ONLY covers changes to that model's direct attributes and does not include changes to any associated models. [Docs for `changed_attributes`](https://api.rubyonrails.org/classes/ActiveModel/Dirty.html#method-i-changed_attributes)

Yea I know a real 'duh' moment (the clue is in the name of the method) but due to less than perfect tests (which I will write more on in another post) this went unnoticed for too long.

### Solution

The solution I chose is pretty simple, add the following to the `no_changes_present` check.
```ruby
&& !activity.tags.any? { |tag| tag.marked_for_destruction? || tag.new_record? }
```
Now if a `Tag` is added (`new_record?`) or removed (`marked_for_destruction?`) `no_changes_present` will now be `false` and the validation will be carried out as normal.

One gotcha to remember is that we need to do a similar thing for the `check!` method on the `Rule`. However this time we only want to validate against `Tag`s that are already associated or newly associated so:

```ruby
def check!(activity)
  #PSEUDO CODE
  active_tags = activity.tags.reject { | tag | tag.marked_for_destruction? }
  unless allowed_actions.include?(activity.action) && allowed_tags.include?(active_tags)
    activity.add_error("Not allowed to do this activity")
  end
end
```
This way we can remove the due to be deleted tags to give us a correct validation before saving the edited `Activity`. 

Now I am aware there are probably a million ways I could've achieved the same result so I would love to hear from anyone out there who has any suggestions on how else this could be implemented.

Til next time..