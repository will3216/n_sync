# NSync

Welcome to your new gem! In this directory, you'll find the files you need to be able to package up your Ruby library into a gem. Put your Ruby code in the file `lib/n_sync`. To experiment with that code, run `bin/console` for an interactive prompt.

TODO: Delete this and the text above, and describe your gem

## Installation

Add this line to your application's Gemfile:

```ruby
gem 'n_sync'
```

And then execute:

    $ bundle

Or install it yourself as:

    $ gem install n_sync

## Configuration

```ruby
NSync.configure do |config|

  config.storage_type               = :aws
  config.aws_access_key             = "your-aws-access-key"
  config.aws_secret_key             = "your-aws-secret-key"

  config.aws_bucket                 = "sync-bucket"
  config.aws_object_key_prefix      = "some/object/key/prefix" # Values will be stored at http://<aws-url>.sync-bucket/some/object/key/prefix/<model_name>/<environment>.json

  # The default setting for models that are 'syncable'
  config.autocommit                 = true

  config.cache_changes_with_redis   = true
  config.redis                      = Redis.current

  # Defaults to [:development, :production, :test]
  config.environments               = [:development, :production, :test, :my_env]
end
```

## Requirements
Each object to be tracked must have a unique value that must be the same across each environment. A value can be added if one does not already exist. Note, autoincrementing values should not be used.

Example:
```ruby
class Person < ActiveRecord::Base

  store     :preference_settings
  serialize :names_of_pets

  syncable :uid do

    # if this is not provided, defaults to the environment listing in config.environments
    link_group [:development, :production]
    link_group [:test, :my_env]

    # This will make it so that changes in production are picked up and applied to development
    #auto_link :development => :production

    sync_string     :name
    sync_integer    :zipcode
    sync_bool       :is_alive
    sync_text       :bio
    sync_float      :account_balance
    sync_hash       :preference_settings
    sync_list       :names_of_pets
    sync_timestamp  :created_at

    autocommit   false # Overrides global setting
  end
end
```

## With Autocommit Turned Off

On development
```ruby
Person.find_by_uid("12345")                      # => nil
# Make find by uid?
person = Person.sync.find("12345")               # => nil
person = Person.sync.find("12345", :development) # => nil




person = Person.new({uid: "12345", name: "Bob Ross"}, without_protection: true)
# => <Person id: nil, name: "Bob Ross", uid: "12345">

# DB   - name => nil
# dev  - name => nil
# prod - name => nil

person.new_record?                    # => true
person.sync.new_record?               # => true
person.sync.new_record?(:production)  # => true

person.persisted?                     # => false
person.sync.persisted?                # => false
person.sync.persisted?(:production)   # => false

person.synced?                        # => true
person.synced?(:production)           # => true

person.changes                        # => {"uid" => [nil, "12345"], "name" => [nil, "Bob Ross"]}
person.sync.changes                   # => {}
person.sync.changes(:production)      # => {}




person.save!                          # => true

# DB   - name => "Bob Ross"
# dev  - name => nil
# prod - name => nil

person.new_record?                    # => false
person.sync.new_record?               # => true
person.sync.new_record?(:production)  # => true

person.persisted?                     # => true
person.sync.persisted?                # => false
person.sync.persisted?(:production)   # => false

person.synced?                        # => false
person.synced?(:production)           # => false

person.changes                        # => {}
person.sync.changes                   # => {"uid" => ["12345", nil], "name" => ["Bob Ross", nil]}
person.sync.changes(:production)      # => {"uid" => ["12345", nil], "name" => ["Bob Ross", nil]}




person.sync.save!                     # => true

# DB   - name => "Bob Ross"
# dev  - name => "Bob Ross"
# prod - name => nil

person.new_record?                    # => false
person.sync.new_record?               # => false
person.sync.new_record?(:production)  # => true

person.persisted?                     # => true
person.sync.persisted?                # => true
person.sync.persisted?(:production)   # => false

person.synced?                        # => true
person.synced?(:production)           # => false

person.changes                        # => {}
person.sync.changes                   # => {}
person.sync.changes(:production)      # => {"uid" => ["12345", nil], "name" => ["Bob Ross", nil]}
```

On production
```ruby

# DB   - name => nil
# dev  - name => "Bob Ross"
# prod - name => nil

Person.find_by_uid("12345")                      # => nil

# Make find by uid?
person = Person.sync.find("12345")               # => nil
person = Person.sync.find("12345", :development) # => <Person id: nil, name: "Bob Ross", uid: "12345">

person.new_record?                    # => true
person.sync.new_record?               # => true
person.sync.new_record?(:development) # => false

person.persisted?                     # => false
person.sync.persisted?                # => false
person.sync.persisted?(:development)  # => true

person.synced?                        # => true
person.synced?(:development)          # => false

person.changes                        # => {"uid" => [nil, "12345"], "name" => [nil, "Bob Ross"]}
person.sync.changes                   # => {}
person.sync.changes(:development)     # => {"uid" => [nil, "12345"], "name" => [nil, "Bob Ross"]}




person.save!                          # => true

# DB   - name => "Bob Ross"
# dev  - name => "Bob Ross"
# prod - name => nil

person.new_record?                    # => false
person.sync.new_record?               # => true
person.sync.new_record?(:development) # => false

person.persisted?                     # => true
person.sync.persisted?                # => false
person.sync.persisted?(:development)  # => true

person.synced?                        # => false
person.synced?(:development)          # => true

person.changes                        # => {}
person.sync.changes                   # => {"uid" => [nil, "12345"], "name" => [nil, "Bob Ross"]}
person.sync.changes(:development)     # => {}




person.sync.save!                     # => true

# DB   - name => "Bob Ross"
# dev  - name => "Bob Ross"
# prod - name => "Bob Ross"

person.new_record?                    # => false
person.sync.new_record?               # => false
person.sync.new_record?(:development) # => false

person.persisted?                     # => true
person.sync.persisted?                # => true
person.sync.persisted?(:development)  # => true

person.synced?                        # => true
person.synced?(:development)          # => true

person.changes                        # => {}
person.sync.changes                   # => {}
person.sync.changes(:development)     # => {}




person.name                           # => "Bob Ross"
person.name = "Robert Ross"           # => "Robert Ross"

# DB   - name => "Bob Ross"
# dev  - name => "Bob Ross"
# prod - name => "Bob Ross"

person.synced?                        # => true
person.synced?(:development)          # => true

person.changed?                       # => true
person.sync.changed?                  # => false
person.sync.changed?(:development)    # => false

person.changes                        # => {"name" => ["Bob Ross", "Robert Ross"]}
person.sync.changes                   # => {}
person.sync.changes(:development)     # => {}




person.save!                          # => true

# DB   - name => "Robert Ross"
# dev  - name => "Bob Ross"
# prod - name => "Bob Ross"

person.synced?                        # => false
person.synced?(:development)          # => false

person.changed?                       # => false
person.sync.changed?                  # => true
person.sync.changed?(:development)    # => true

person.changes                        # => {}
person.sync.changes                   # => {"name" => ["Robert Ross", "Bob Ross"]}
person.sync.changes(:development)     # => {"name" => ["Robert Ross", "Bob Ross"]}




person.sync.save!                     # => true

# DB   - name => "Robert Ross"
# dev  - name => "Bob Ross"
# prod - name => "Robert Ross"

person.synced?                        # => true
person.synced?(:development)          # => false

person.changed?                       # => false
person.sync.changed?                  # => false
person.sync.changed?(:development)    # => true

person.changes                        # => {}
person.sync.changes                   # => {}
person.sync.changes(:development)     # => {"name" => ["Bob Ross", "Robert Ross"]}
```



On development:
```ruby

person = Person.sync.find("12345")    # => <Person id: 1, name: "Bob Ross", uid: "12345">

# DB   - name => "Bob Ross"
# dev  - name => "Bob Ross"
# prod - name => "Robert Ross"

person.synced?                        # => true
person.synced?(:production)           # => false

person.changed?                       # => false
person.sync.changed?                  # => false
person.sync.changed?(:production)     # => true

person.changes                        # => {}
person.sync.changes                   # => {}
person.sync.changes(:production)      # => {"name" => ["Bob Ross", "Robert Ross"]}



person = Person.sync.find("12345", :production)
# => <Person id: 1, name: "Robert Ross", uid: "12345">

# DB   - name => "Bob Ross"
# dev  - name => "Bob Ross"
# prod - name => "Robert Ross"

person.synced?                        # => true
person.synced?(:production)           # => false

person.changed?                       # => true
person.sync.changed?                  # => false
person.sync.changed?(:production)     # => true

person.changes                        # => {"name" => ["Bob Ross", "Robert Ross"}
person.sync.changes                   # => {}
person.sync.changes(:production)      # => {"name" => ["Bob Ross", "Robert Ross"]}



person = Person.find_by_uid("12345")  # => <Person id: 1, name: "Bob Ross", uid: "12345">

# DB   - name => "Bob Ross"
# dev  - name => "Bob Ross"
# prod - name => "Robert Ross"

person.synced?                        # => true
person.synced?(:production)           # => false

person.changed?                       # => false
person.sync.changed?                  # => false
person.sync.changed?(:production)     # => true

person.changes                        # => {}
person.sync.changes                   # => {}
person.sync.changes(:production)      # => {"name" => ["Bob Ross", "Robert Ross"]}




# DB   - name => "Bob Ross"
# dev  - name => "Bob Ross"
# prod - name => "Robert Ross"

person.sync.update_attributes(:production)  # => true

# DB   - name => "Robert Ross"
# dev  - name => "Bob Ross"
# prod - name => "Robert Ross"

person.synced?                              # => false
person.synced?(:production)                 # => true

person.changed?                             # => false
person.sync.changed?                        # => true
person.sync.changed?(:production)           # => false

person.changes                              # => {}
person.sync.changes                         # => {"name" => ["Robert Ross", "Bob Ross"]}
person.sync.changes(:production)            # => {}





# DB   - name => "Robert Ross"
# dev  - name => "Bob Ross"
# prod - name => "Robert Ross"

person.sync.update_attributes               # => true

# DB   - name => "Bob Ross"
# dev  - name => "Bob Ross"
# prod - name => "Robert Ross"

person.synced?                              # => true
person.synced?(:production)                 # => false

person.changed?                             # => false
person.sync.changed?                        # => false
person.sync.changed?(:production)           # => true

person.changes                              # => {}
person.sync.changes                         # => {}
person.sync.changes(:production)            # => {"name" => ["Bob Ross", "Robert Ross"]}




# DB   - name => "Bob Ross"
# dev  - name => "Bob Ross"
# prod - name => "Robert Ross"

person.sync.assign_attributes(:production)  # => true

# DB   - name => "Bob Ross"
# dev  - name => "Bob Ross"
# prod - name => "Robert Ross"

person.synced?                              # => true
person.synced?(:production)                 # => false

person.changed?                             # => true
person.sync.changed?                        # => false
person.sync.changed?(:production)           # => true

person.changes                              # => {"name" => ["Bob Ross", "Robert Ross"]}
person.sync.changes                         # => {}
person.sync.changes(:production)            # => {"name" => ["Bob Ross", "Robert Ross"]}




# DB   - name => "Bob Ross"
# dev  - name => "Bob Ross"
# prod - name => "Robert Ross"

person.save!                                # => true

# DB   - name => "Robert Ross"
# dev  - name => "Bob Ross"
# prod - name => "Robert Ross"

person.synced?                              # => false
person.synced?(:production)                 # => true

person.changed?                             # => false
person.sync.changed?                        # => true
person.sync.changed?(:production)           # => false

person.changes                              # => {}
person.sync.changes                         # => {"name" => ["Robert Ross", "Bob Ross"]}
person.sync.changes(:production)            # => {}




# DB   - name => "Robert Ross"
# dev  - name => "Bob Ross"
# prod - name => "Robert Ross"

person.sync.save!                           # => true

# DB   - name => "Robert Ross"
# dev  - name => "Robert Ross"
# prod - name => "Robert Ross"

person.synced?                              # => true
person.synced?(:production)                 # => true

person.changed?                             # => false
person.sync.changed?                        # => false
person.sync.changed?(:production)           # => false

person.changes                              # => {}
person.sync.changes                         # => {}
person.sync.changes(:production)            # => {}




# DB   - name => "Robert Ross"
# dev  - name => "Robert Ross"
# prod - name => "Robert Ross"

person.destroy                              # => <Person id: 1, name: "Bob Ross", uid: "12345">

# DB   - name => nil
# dev  - name => "Robert Ross"
# prod - name => "Robert Ross"

person.synced?                              # => false
person.synced?(:production)                 # => false

person.changed?                             # => false
person.sync.changed?                        # => true
person.sync.changed?(:production)           # => true

person.persisted?                           # => false
person.sync.persisted?                      # => true
person.sync.persisted?(:production)         # => true

person.destroyed?                           # => true
person.sync.destroyed?                      # => false
person.sync.destroyed?(:production)         # => false

person.changes                              # => {}
person.sync.changes                         # => {"uid" => [nil, "12345"], "name" => [nil, "Robert Ross"]}
person.sync.changes(:production)            # => {"uid" => [nil, "12345"], "name" => [nil, "Robert Ross"]}




# DB   - name => nil
# dev  - name => "Robert Ross"
# prod - name => "Robert Ross"

person.sync.destroy                         # => <Person id: 1, name: "Bob Ross", uid: "12345">

# DB   - name => nil
# dev  - name => nil
# prod - name => "Robert Ross"

person.synced?                              # => true
person.synced?(:production)                 # => false

person.changed?                             # => false
person.sync.changed?                        # => false
person.sync.changed?(:production)           # => true

person.persisted?                           # => false
person.sync.persisted?                      # => false
person.sync.persisted?(:production)         # => true

person.destroyed?                           # => true
person.sync.destroyed?                      # => true
person.sync.destroyed?(:production)         # => false

person.changes                              # => {}
person.sync.changes                         # => {}
person.sync.changes(:production)            # => {"uid" => [nil, "12345"], "name" => [nil, "Robert Ross"]}
```

## With Autocommit Turned On

On development
```ruby
Person.find_by_uid("12345")                      # => nil
# Make find by uid?
person = Person.sync.find("12345")               # => nil
person = Person.sync.find("12345", :development) # => nil




person = Person.new({uid: "12345", name: "Bob Ross"}, without_protection: true)
# => <Person id: nil, name: "Bob Ross", uid: "12345">

# DB   - name => nil
# dev  - name => nil
# prod - name => nil

person.new_record?                    # => true
person.sync.new_record?               # => true
person.sync.new_record?(:production)  # => true

person.persisted?                     # => false
person.sync.persisted?                # => false
person.sync.persisted?(:production)   # => false

person.synced?                        # => true
person.synced?(:production)           # => true

person.changes                        # => {"uid" => [nil, "12345"], "name" => [nil, "Bob Ross"]}
person.sync.changes                   # => {"uid" => [nil, "12345"], "name" => [nil, "Bob Ross"]}
person.sync.changes(:production)      # => {}




# DB   - name => nil
# dev  - name => nil
# prod - name => nil

person.save!                          # => true

# DB   - name => "Bob Ross"
# dev  - name => "Bob Ross"
# prod - name => nil

person.new_record?                    # => false
person.sync.new_record?               # => false
person.sync.new_record?(:production)  # => true

person.persisted?                     # => true
person.sync.persisted?                # => true
person.sync.persisted?(:production)   # => false

person.synced?                        # => true
person.synced?(:production)           # => false

person.changes                        # => {}
person.sync.changes                   # => {}
person.sync.changes(:production)      # => {"uid" => ["12345", nil], "name" => ["Bob Ross", nil]}




person.sync.save!                     # => true

# DB   - name => "Bob Ross"
# dev  - name => "Bob Ross"
# prod - name => nil

person.new_record?                    # => false
person.sync.new_record?               # => false
person.sync.new_record?(:production)  # => true

person.persisted?                     # => true
person.sync.persisted?                # => true
person.sync.persisted?(:production)   # => false

person.synced?                        # => true
person.synced?(:production)           # => false

person.changes                        # => {}
person.sync.changes                   # => {}
person.sync.changes(:production)      # => {"uid" => ["12345", nil], "name" => ["Bob Ross", nil]}
```

On production
```ruby

# DB   - name => nil
# dev  - name => "Bob Ross"
# prod - name => nil

Person.find_by_uid("12345")                      # => nil

# Make find by uid?
person = Person.sync.find("12345")               # => nil
person = Person.sync.find("12345", :development) # => <Person id: nil, name: "Bob Ross", uid: "12345">

person.new_record?                    # => true
person.sync.new_record?               # => true
person.sync.new_record?(:development) # => false

person.persisted?                     # => false
person.sync.persisted?                # => false
person.sync.persisted?(:development)  # => true

person.synced?                        # => true
person.synced?(:development)          # => false

person.changes                        # => {"uid" => [nil, "12345"], "name" => [nil, "Bob Ross"]}
person.sync.changes                   # => {"uid" => [nil, "12345"], "name" => [nil, "Bob Ross"]}
person.sync.changes(:development)     # => {"uid" => [nil, "12345"], "name" => [nil, "Bob Ross"]}




person.save!                          # => true

# DB   - name => "Bob Ross"
# dev  - name => "Bob Ross"
# prod - name => "Bob Ross"

person.new_record?                    # => false
person.sync.new_record?               # => false
person.sync.new_record?(:development) # => false

person.persisted?                     # => true
person.sync.persisted?                # => true
person.sync.persisted?(:development)  # => true

person.synced?                        # => true
person.synced?(:development)          # => true

person.changes                        # => {}
person.sync.changes                   # => {}
person.sync.changes(:development)     # => {}




person.sync.save!                     # => true

# DB   - name => "Bob Ross"
# dev  - name => "Bob Ross"
# prod - name => "Bob Ross"

person.new_record?                    # => false
person.sync.new_record?               # => false
person.sync.new_record?(:development) # => false

person.persisted?                     # => true
person.sync.persisted?                # => true
person.sync.persisted?(:development)  # => true

person.synced?                        # => true
person.synced?(:development)          # => true

person.changes                        # => {}
person.sync.changes                   # => {}
person.sync.changes(:development)     # => {}




person.name                           # => "Bob Ross"
person.name = "Robert Ross"           # => "Robert Ross"

# DB   - name => "Bob Ross"
# dev  - name => "Bob Ross"
# prod - name => "Bob Ross"

person.synced?                        # => true
person.synced?(:development)          # => true

person.changed?                       # => true
person.sync.changed?                  # => true
person.sync.changed?(:development)    # => false

person.changes                        # => {"name" => ["Bob Ross", "Robert Ross"]}
person.sync.changes                   # => {"name" => ["Bob Ross", "Robert Ross"]}
person.sync.changes(:development)     # => {}




person.save!                          # => true

# DB   - name => "Robert Ross"
# dev  - name => "Bob Ross"
# prod - name => "Robert Ross"

person.synced?                        # => truew
person.synced?(:development)          # => false

person.changed?                       # => false
person.sync.changed?                  # => false
person.sync.changed?(:development)    # => true

person.changes                        # => {}
person.sync.changes                   # => {}
person.sync.changes(:development)     # => {"name" => ["Robert Ross", "Bob Ross"]}




person.sync.save!                     # => true

# DB   - name => "Robert Ross"
# dev  - name => "Bob Ross"
# prod - name => "Robert Ross"

person.synced?                        # => true
person.synced?(:development)          # => false

person.changed?                       # => false
person.sync.changed?                  # => false
person.sync.changed?(:development)    # => true

person.changes                        # => {}
person.sync.changes                   # => {}
person.sync.changes(:development)     # => {"name" => ["Bob Ross", "Robert Ross"]}
```



On development:
```ruby

person = Person.sync.find("12345")    # => <Person id: 1, name: "Bob Ross", uid: "12345">

# DB   - name => "Bob Ross"
# dev  - name => "Bob Ross"
# prod - name => "Robert Ross"

person.synced?                        # => true
person.synced?(:production)           # => false

person.changed?                       # => false
person.sync.changed?                  # => false
person.sync.changed?(:production)     # => true

person.changes                        # => {}
person.sync.changes                   # => {}
person.sync.changes(:production)      # => {"name" => ["Bob Ross", "Robert Ross"]}



person = Person.sync.find("12345", :production)
# => <Person id: 1, name: "Robert Ross", uid: "12345">

# DB   - name => "Bob Ross"
# dev  - name => "Bob Ross"
# prod - name => "Robert Ross"

person.synced?                        # => true
person.synced?(:production)           # => false

person.changed?                       # => true
person.sync.changed?                  # => true
person.sync.changed?(:production)     # => true

person.changes                        # => {"name" => ["Bob Ross", "Robert Ross"]}
person.sync.changes                   # => {"name" => ["Bob Ross", "Robert Ross"]}
person.sync.changes(:production)      # => {"name" => ["Bob Ross", "Robert Ross"]}



person = Person.find_by_uid("12345")  # => <Person id: 1, name: "Bob Ross", uid: "12345">

# DB   - name => "Bob Ross"
# dev  - name => "Bob Ross"
# prod - name => "Robert Ross"

person.synced?                        # => true
person.synced?(:production)           # => false

person.changed?                       # => false
person.sync.changed?                  # => false
person.sync.changed?(:production)     # => true

person.changes                        # => {}
person.sync.changes                   # => {}
person.sync.changes(:production)      # => {"name" => ["Bob Ross", "Robert Ross"]}




# DB   - name => "Bob Ross"
# dev  - name => "Bob Ross"
# prod - name => "Robert Ross"

person.sync.update_attributes(:production)  # => true

# DB   - name => "Robert Ross"
# dev  - name => "Robert Ross"
# prod - name => "Robert Ross"

person.synced?                              # => true
person.synced?(:production)                 # => true

person.changed?                             # => false
person.sync.changed?                        # => false
person.sync.changed?(:production)           # => false

person.changes                              # => {}
person.sync.changes                         # => {}
person.sync.changes(:production)            # => {}





# DB   - name => "Robert Ross"
# dev  - name => "Robert Ross"
# prod - name => "Robert Ross"

person.sync.update_attributes               # => true

# DB   - name => "Bob Ross"
# dev  - name => "Bob Ross"
# prod - name => "Robert Ross"

person.synced?                              # => true
person.synced?(:production)                 # => false

person.changed?                             # => false
person.sync.changed?                        # => false
person.sync.changed?(:production)           # => true

person.changes                              # => {}
person.sync.changes                         # => {}
person.sync.changes(:production)            # => {"name" => ["Bob Ross", "Robert Ross"]}




# DB   - name => "Bob Ross"
# dev  - name => "Bob Ross"
# prod - name => "Robert Ross"

person.sync.assign_attributes(:production)  # => true

# DB   - name => "Bob Ross"
# dev  - name => "Bob Ross"
# prod - name => "Robert Ross"

person.synced?                              # => true
person.synced?(:production)                 # => false

person.changed?                             # => true
person.sync.changed?                        # => true
person.sync.changed?(:production)           # => true

person.changes                              # => {"name" => ["Bob Ross", "Robert Ross"]}
person.sync.changes                         # => {"name" => ["Bob Ross", "Robert Ross"]}
person.sync.changes(:production)            # => {"name" => ["Bob Ross", "Robert Ross"]}




# DB   - name => "Bob Ross"
# dev  - name => "Bob Ross"
# prod - name => "Robert Ross"

person.save!                                # => true

# DB   - name => "Robert Ross"
# dev  - name => "Robert Ross"
# prod - name => "Robert Ross"

person.synced?                              # => true
person.synced?(:production)                 # => true

person.changed?                             # => false
person.sync.changed?                        # => false
person.sync.changed?(:production)           # => false

person.changes                              # => {}
person.sync.changes                         # => {}
person.sync.changes(:production)            # => {}




# DB   - name => "Robert Ross"
# dev  - name => "Robert Ross"
# prod - name => "Robert Ross"

person.sync.save!                           # => true

# DB   - name => "Robert Ross"
# dev  - name => "Robert Ross"
# prod - name => "Robert Ross"

person.synced?                              # => true
person.synced?(:production)                 # => true

person.changed?                             # => false
person.sync.changed?                        # => false
person.sync.changed?(:production)           # => false

person.changes                              # => {}
person.sync.changes                         # => {}
person.sync.changes(:production)            # => {}




# DB   - name => "Robert Ross"
# dev  - name => "Robert Ross"
# prod - name => "Robert Ross"

person.destroy                              # => <Person id: 1, name: "Bob Ross", uid: "12345">

# DB   - name => nil
# dev  - name => nil
# prod - name => "Robert Ross"

person.synced?                              # => true
person.synced?(:production)                 # => false

person.changed?                             # => false
person.sync.changed?                        # => false
person.sync.changed?(:production)           # => true

person.persisted?                           # => false
person.sync.persisted?                      # => false
person.sync.persisted?(:production)         # => true

person.destroyed?                           # => true
person.sync.destroyed?                      # => true
person.sync.destroyed?(:production)         # => false

person.changes                              # => {}
person.sync.changes                         # => {}
person.sync.changes(:production)            # => {"uid" => [nil, "12345"], "name" => [nil, "Robert Ross"]}




# DB   - name => nil
# dev  - name => nil
# prod - name => "Robert Ross"

person.sync.destroy                         # => <Person id: 1, name: "Bob Ross", uid: "12345">

# DB   - name => nil
# dev  - name => nil
# prod - name => "Robert Ross"

person.synced?                              # => true
person.synced?(:production)                 # => false

person.changed?                             # => false
person.sync.changed?                        # => false
person.sync.changed?(:production)           # => true

person.persisted?                           # => false
person.sync.persisted?                      # => false
person.sync.persisted?(:production)         # => true

person.destroyed?                           # => true
person.sync.destroyed?                      # => true
person.sync.destroyed?(:production)         # => false

person.changes                              # => {}
person.sync.changes                         # => {}
person.sync.changes(:production)            # => {"uid" => [nil, "12345"], "name" => [nil, "Robert Ross"]}
```

## Development Notes

* When synced? is called on an object with pending unsaved changes: the changes must be stashed, the sync status must be checked, and then the changes must be re-staged before returning the result.
* Autocommit from another environment by using file-system watching

After checking out the repo, run `bin/setup` to install dependencies. Then, run `rake rspec` to run the tests. You can also run `bin/console` for an interactive prompt that will allow you to experiment.

To install this gem onto your local machine, run `bundle exec rake install`. To release a new version, update the version number in `version.rb`, and then run `bundle exec rake release`, which will create a git tag for the version, push git commits and tags, and push the `.gem` file to [rubygems.org](https://rubygems.org).

## Contributing

Bug reports and pull requests are welcome on GitHub at https://github.com/[USERNAME]/n_sync.

