[![Gem Version](https://badge.fury.io/rb/schema_plus_views.svg)](http://badge.fury.io/rb/schema_plus_views)
[![Build Status](https://secure.travis-ci.org/SchemaPlus/schema_plus_views.svg)](http://travis-ci.org/SchemaPlus/schema_plus_views)
[![Coverage Status](https://img.shields.io/coveralls/SchemaPlus/schema_plus_views.svg)](https://coveralls.io/r/SchemaPlus/schema_plus_views)
[![Dependency Status](https://gemnasium.com/lomba/schema_plus_views.svg)](https://gemnasium.com/SchemaPlus/schema_plus_views)

# SchemaPlus::Views

SchemaPlus::Views adds support for creating and dropping views in ActiveRecord migrations, as well as querying views.

SchemaPlus::Views is part of the [SchemaPlus](https://github.com/SchemaPlus/) family of Ruby on Rails extension gems.

## Installation

<!-- SCHEMA_DEV: TEMPLATE INSTALLATION - begin -->
<!-- These lines are auto-inserted from a schema_dev template -->
As usual:

```ruby
gem "schema_plus_views"                # in a Gemfile
gem.add_dependency "schema_plus_views" # in a .gemspec
```

<!-- SCHEMA_DEV: TEMPLATE INSTALLATION - end -->

## Compatibility

SchemaPlus::Views is tested on:

<!-- SCHEMA_DEV: MATRIX - begin -->
<!-- These lines are auto-generated by schema_dev based on schema_dev.yml -->
* ruby **2.1.5** with activerecord **4.2**, using **mysql2**, **sqlite3** or **postgresql**

<!-- SCHEMA_DEV: MATRIX - end -->

## Usage

### Creating views

In a migration, a view can be created using literal SQL:

```ruby
create_view :uncommented_posts, "SELECT * FROM posts LEFT OUTER JOIN comments ON comments.post_id = posts.id WHERE comments.id IS NULL"
```

or using an object that responds to `:to_sql`, such as a relation:

```ruby
create_view :posts_commented_by_staff,  Post.joins(comment: user).where(users: {role: 'staff'}).uniq
```

(It's of course a questionable idea for your migration files to depend on your model definitions.  But you *can* if you want.)

Additional options can be provided:

* `:force => true` if there's an existing view with the given name, deletes it first.  Note that this could fail if another view depends on it.

* `:allow_replace => true` will use the command "CREATE OR REPLACE" when creating the view, for seamlessly redefining the view even if other views depend on it.  It's only supported by MySQL and PostgreSQL, and each has some limitations on when a view can be replaced; see their docs for details.

SchemaPlus::Views also arranges to include the `create_view` statements (with literal SQL) in the schema dump.

### Dropping views

In a migration:

```ruby
drop_view :posts_commented_by_staff
drop_view :uncommented_posts, :if_exists => true
```

### Using views

ActiveRecord models can be based on views the same as ordinary tables.  That is, for the above views you can define

```ruby
class UncommentedPost < ActiveRecord::Base
end

class PostCommentedByStaff < ActiveRecord::Base
  table_name = "posts_commented_by_staff"
end
```

### Querying views

You can look up the defined views analogously to looking up tables:

```ruby
connection.tables # => array of table names [method provided by ActiveRecord]
connection.views  # => array of view names [method provided by SchemaPlus::Views]
```

Notes:

1. For Mysql and SQLite3, ActiveRecord's `connection.tables` method would return views as well as tables; SchemaPlus::Views normalizes them to return only tables.

2. For PostgreSQL, `connection.views` suppresses views prefixed with `pg_` as those are presumed to be internal.

### Querying view definitions

You can look up the definition of a view using

```ruby
connection.view_definition(view_name) # => returns SQL string
```

This returns just the body of the definition, i.e. the part after the `CREATE VIEW 'name' AS` command.

## Customization API: Middleware Stacks

All the methods defined by SchemaPlus::Views provide middleware stacks, in case you need to do any custom filtering, rewriting, triggering, or whatever.  For info on how to use middleware stacks, see the READMEs of [schema_monkey](https://github.com/SchemaPlus/schema_monkey) and [schema_plus_core](https://github.com/SchemaPlus/schema_plus_core).


### `Schema::Views` stack

Wraps the `connection.views` method.  Env contains:

Env Field    | Description | Initialized
--- | --- | ---
`:views`     | The result of the lookup | `[]`
`:connection` | The current ActiveRecord connection | *context*
`:query_name` | Optional label for ActiveRecord logging | *arg*

The base implementation appends its results to `env.views`

### `Schema::ViewDefinition` stack

Wraps the `connection.view_definition` method.  Env contains:

Env Field    | Description | Initialized
--- | --- | ---
`:connection` | The current ActiveRecord connection | *context*
`:view_name`  | The view to look up | *arg*
`:query_name` | Optional label for ActiveRecord logging | *arg*
`:definition` | The view definition SQL | `nil`

The base implementation looks up the definition of the view named
`env.view_name` and assigns the result to `env.definition`

### `Migration::CreateView` stack

Wraps the `migration.create_view` method.  Env contains:

Env Field    | Description | Initialized
--- | --- | ---
`:connection` | The current ActiveRecord connection | *context*
`:view_name`  | The view name | *arg*
`:definition` | The view definition SQL | *arg*
`:options` | Create view options | *arg*

The base implementation creates the view named `env.view_name` using the
definition in `env.definition` with options in `env.options`

### `Migration::DropView` stack

Wraps the `migration.drop_view` method.  Env contains:

Env Field    | Description | Initialized
--- | --- | ---
`:connection` | The current ActiveRecord connection | *context*
`:view_name`  | The view name | *arg*
`:options` | Drop view options | *arg*

The base implementation drops the view named `env.view_name` using the
options in `env.options`


## History

* 0.3.1 - Upgrade schema_plus_core and schema_dev dependencies
* 0.3.0
  - Added middleware stacks
  - Bug fix: view_definition: strip white space from result (postgresql)
* 0.2.3 - Remove unnecessary escaping in dump; use single-quote heredoc
* 0.2.2 - Prettier dumps: use heredoc for definition string
* 0.2.1 - Fix db:rollback
* 0.2.0 - Added :allow_replace option (thanks to [@hcarver](https://github.com/hcarver))
* 0.1.0 - Initial release, extracted from schema_plus 1.x

## Development & Testing

Are you interested in contributing to SchemaPlus::Views?  Thanks!  Please follow the standard protocol: fork, feature branch, develop, push, and issue pull request.

Some things to know about to help you develop and test:

<!-- SCHEMA_DEV: TEMPLATE USES SCHEMA_DEV - begin -->
<!-- These lines are auto-inserted from a schema_dev template -->
* **schema_dev**:  SchemaPlus::Views uses [schema_dev](https://github.com/SchemaPlus/schema_dev) to
  facilitate running rspec tests on the matrix of ruby, activerecord, and database
  versions that the gem supports, both locally and on
  [travis-ci](http://travis-ci.org/SchemaPlus/schema_plus_views)

  To to run rspec locally on the full matrix, do:

        $ schema_dev bundle install
        $ schema_dev rspec

  You can also run on just one configuration at a time;  For info, see `schema_dev --help` or the [schema_dev](https://github.com/SchemaPlus/schema_dev) README.

  The matrix of configurations is specified in `schema_dev.yml` in
  the project root.


<!-- SCHEMA_DEV: TEMPLATE USES SCHEMA_DEV - end -->

<!-- SCHEMA_DEV: TEMPLATE USES SCHEMA_PLUS_CORE - begin -->
<!-- These lines are auto-inserted from a schema_dev template -->
* **schema_plus_core**: SchemaPlus::Views uses the SchemaPlus::Core API that
  provides middleware callback stacks to make it easy to extend
  ActiveRecord's behavior.  If that API is missing something you need for
  your contribution, please head over to
  [schema_plus_core](https://github.com/SchemaPlus/schema_plus_core) and open
  an issue or pull request.

<!-- SCHEMA_DEV: TEMPLATE USES SCHEMA_PLUS_CORE - end -->

<!-- SCHEMA_DEV: TEMPLATE USES SCHEMA_MONKEY - begin -->
<!-- These lines are auto-inserted from a schema_dev template -->
* **schema_monkey**: SchemaPlus::Views is implemented as a
  [schema_monkey](https://github.com/SchemaPlus/schema_monkey) client,
  using [schema_monkey](https://github.com/SchemaPlus/schema_monkey)'s
  convention-based protocols for extending ActiveRecord and using middleware stacks.

<!-- SCHEMA_DEV: TEMPLATE USES SCHEMA_MONKEY - end -->
