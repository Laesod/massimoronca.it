---
title: "Phoenix Framework: to the basics and beyond"
date: "2016-02-04"
description: "Phoenix is the exciting new kid on the block in the vast world of web frameworks.  
Its roots are in Rails, with the bonus of the performances of a compiled language.<br>  
This isn't exactly a getting started guide, but a list (albeit short) of things you'll have to know very soon in the process of writing a Phoenix application, that are just a bit beyond the <em><q>writing a blog engine in 15 minutes by using only the default generators</q></em>."
source_name: MIKAMAYHEM
source_url: http://dev.mikamai.com/post/138675652509/phoenix-to-the-basics-and-beyond
image: http://www.silpion.de/wp-content/uploads/phoenix-logo.png
tags:
- Elixir
- Phoenix Framework
- Rails alternatives
---


# Introduction

Phoenix is the exciting new kid on the block in the vast world of web frameworks.  
Its roots are in Rails, with the bonus of the performances of a compiled language.  
This isn't exactly a getting started guide, but a (albeit short) list of things you'll have to know very soon in the process of writing a Phoenix application, that are just a bit beyond the *<q>writing a blog engine in 15 minutes by using only the default generators</q>*.   
I assume previous knowledge of the Elixir language, the Phoenix framework and the command line tools

``` ruby
mix ecto.create         # Create the storage for the repo
mix ecto.drop           # Drop the storage for the repo
mix ecto.gen.migration  # Generate a new migration for the repo
mix ecto.gen.repo       # Generate a new repository
mix ecto.migrate        # Run migrations up on a repo
mix ecto.rollback       # Rollback migrations from a repo
mix phoenix.digest      # Digests and compress static files
mix phoenix.gen.channel # Generates a Phoenix channel
mix phoenix.gen.html    # Generates controller, model and views for an HTML based resource
mix phoenix.gen.json    # Generates a controller and model for a JSON based resource
mix phoenix.gen.model   # Generates an Ecto model
mix phoenix.gen.secret  # Generates a secret
mix phoenix.new         # Create a new Phoenix v1.1.2 application
mix phoenix.routes      # Prints all routes
mix phoenix.server      # Starts applications and their servers
mix test                # Runs a project's tests
iex -S mix              # Starts IEx and run the default task
```

> command line tools cheat sheet

# First, a digression: development environments

If you want to skip this section, [go directly where the action is](#models)

### Emacs

Emacs and [Alchemist](http://www.alchemist-elixir.org/) are the most advanced programming environment for Elixir and Phoenix development.  
The features range from the simple syntax highlighting, to autocomplete, mix and iex integration, testing, compiling, running and evaluating code on the fly, everything you would expect from a modern IDE.  
Unfortunately (for me) I can't really Emacs, the fact that my favourite Emacs distribution is [Spacemacs](http://spacemacs.org/) says a lot about which one of the two kind of people I am.  
Just for reference, if you are like me, `SPC m t a` launch `mix test` in a new buffer and rerun it every time you save a file.  
As simple as it might look, Alchemist is the only environment that offers this feature out of the box (everytime in Spacemacs you see `, somekey someotherkey`, that comma is translated to `m` in Spacemacs bindings).

![, somekey someotherkey](http://i.imgur.com/zkhBOii.png)

> , somekey someotherkey

**TL;DR**: for hardcore Emacs users [Alchemist](http://www.alchemist-elixir.org/) is a must have.

### Sublime/Textmate

Sublime is my editor of choice on OS X and is the one that I use to write Elixir too.  
There is a single bundle to be installed, it's called Elixir and it's the same for Sublime and Textmate.  
It provides syntax highlighting, autocomplete, snippets and a couple of commands (`Build with: elixir-mix-test` and `Build with: elixir`) to run tests and projects directly from the editor.  
I've also installed [SublimeREPL](https://sublimerepl.readthedocs.org/en/latest/) which currently supports Elixir and iex, so you can split the screen between and iex shell and a code editor pane.

**TL;DR**: All the usual goodies of Sublime applies.

### Atom

Atom is great, but actually is not.  
This sentiment strucks me everytime I use it, something is awesome, something else is awful, there is no middle ground when it comes to the Github editor.  
For example I find unacceptable that in 2016 an editor cannot soft wrap on column 78 by default and I've found no package that does it.  
Coming from node, Atom inherits its problems, specifically the need to install many small packages to just get what you need.  
This time I installed `language elixir`, `autocomplete-elixir`, `elixir-cmd` (I'm not very convinced it is useful at all) and `iex`. The last one is the best of them all, it is actually pretty great honestly, letting you run an iex console and tests inside it (by pressing `ALT+CMD+a`).

**TL;DR**: Sorry Atom, I've tried, but you did not convince me.

### Beauty contest

I use Solarized Dark theme everywhere, from my editor, to the shell, I even used the Solarized palette in my personal website.  
Based on that, I will share my opinions on the different syntax highlighting engines.

IMHO the winner is SublimeText (and Textmate, the bundle is the same).  
The different elements are clearly highlighted and less prominent ones (like comments) are low contrast, you can almost <q>not see them</q> after a little bit of training.

![SublimeText 3](http://i.imgur.com/kgCAyls.png)

> SublimeText 3


Emacs comes second, we have a clear distinction here too, but the different colours are too soft/pastel and comments look too bright.

![Emacs](http://i.imgur.com/CSripmC.png)

> Emacs

  
Atom is the loser here.  
I like a lot the polished interface and the look of the theme (the best of the lot), but the syntax highlighting is actually pretty cheap.

![Atom](http://i.imgur.com/aL2Jmot.png)

> Atom


<a name="models"></a>

# Models

Phoenix models are a very thin layer around [`Ecto models`](https://hexdocs.pm/ecto/Ecto.Model.html).  
The [schema dsl](https://hexdocs.pm/ecto/Ecto.Schema.html) is very lightweight making it super easy to model the data layer.

> What a minimal model looks like

```elixir
defmodule App.Email do
  use App.Web, :model

  schema "emails" do
    field :address, :string
    timestamps # the ususal timestamps: created_at, updated_at
  end
end
```

With `use App.Web, :model` you just import some packages defined in `web/web.ex` that you can customize.  
Specifically, by default Phoenix `use`s `Ecto.model` and `import`s from `Ecto.Changeset` and `Ecto.Query`.  
The difference between `use` and `import` is that `use` calls the `__using__` macro and inject code in the module that uses it, while `import` make functions in the imported module available to the caller and let you chose to import everything or just some of them.

### Custom primary keys

The first “beyond the basics” we can do is to personalize the primary key of our model.  
By default they are autogenerated, autoincrementing `integers` and are called `id`.

Suppose we want to use a [GUID](https://en.wikipedia.org/wiki/Globally_unique_identifier), how can we do that?

It's pretty simple: first of all we have to tell the model that we don't want Phoenix to autogenerate the primary key.  
We do that at the `migration` level, where we also tell Phoenix to generate a field of type `:uuid`, [it's a predefined type that maps to `Ecto.UUID`](https://github.com/phoenixframework/phoenix/blob/5f230cf52a5f601e4a1cd35dbb6a2d8ba8082728/lib/mix/tasks/phoenix.gen.model.ex#L209)

```elixir
defmodule App.Repo.Migrations.CreateEmail do
  use Ecto.Migration

  def change do
    create table(:emails, primary_key: false) do
      add :key, :uuid, primary_key: true

      # ... rest of the table definition
```

Then we instruct the model to use our new primary key

```elixir
defmodule App.Email do
  use App.Web, :model

  # autogenerate GUIDs through the included GUID generator
  # the type is binary_id
  # by passing --binary-id to 'mix phoenix.new' 
  # binary_id is assumed as the default format for ids
  @primary_key {:key, :binary_id, autogenerate: true}

  # instruct Phoenix on how to extract the primary key for this model
  @derive {Phoenix.Param, key: :key}

```

That's all.  
We can now safely browse urls like 
`http://localhost/emails/1b72fe0b-5977-48bd-a19c-38140d1d1fed`.

### Custom validators

Validators in Phoenix are just functions that accept a changeset and optionally other parameters and return the same changeset or a new one filled with errors.  
[Changeset](https://hexdocs.pm/ecto/Ecto.Changeset.html) are the data structure Phoenix uses to keep track of the changes in the model and provide constraints and validations facilities.

Changesets provide the following default validators

```elixir
validate_change(changeset, field, validator)
# Validates the given field change
validate_confirmation(changeset, field, opts \\ [])
# Validates that the given field matches the confirmation parameter of that field
validate_exclusion(changeset, field, data, opts \\ [])
# Validates a change is not included in the given enumerable
validate_format(changeset, field, format, opts \\ [])
# Validates a change has the given format
validate_inclusion(changeset, field, data, opts \\ [])
# Validates a change is included in the given enumerable
validate_length(changeset, field, opts)
# Validates a change is a string or list of the given length
validate_number(changeset, field, opts)
# Validates the properties of a number
validate_subset(changeset, field, data, opts \\ [])
# Validates a change, of type enum, is a subset of the given enumerable. 
# Like validate_inclusion/4 for lists
```

> default validators

On a first look, it seems like Ecto developers have been lazy and provided only a handful of validators.  
In reality it's incredibly easy to build custom validators, thank to the `|>` operator we actually build validation pipelines.

For example this code validates an Email model by ensuring the presence of the fields name and address and enforcing the format for the address field.

```elixir
defmodule Wejustsend.Email do
  # all the usual imports and schema omitted  

  @doc """
  Creates a changeset based on the `model` and `params`.

  If no params are provided, an invalid changeset is returned
  with no validation performed.
  """
  def changeset(model, params \\ :empty) do
    model
    # build a changeset from params
    # cast enforce the type of the fields and the presence of the required 
    |> cast(params, @required_fields, @optional_fields) 
    |> validate_name
    |> validate_address
  end

  defp validate_name(changeset) do
    # name cannot be shorter than 4 characters
    changeset |> validate_length(:name, min: 4)
  end

  defp validate_address(changeset) do
    changeset
    |> validate_length(:address, min: 5) # a@a.a is 5 characters long
    |> validate_address_list
  end

  defp validate_address_list(changeset) do
    # validate a list of one or more email addresses separated by ','
    # an email address is considered valid if it matches the regex
    # ^\S+@\S+\.\S+$
    changeset |> to_address_list |> Enum.reduce(changeset, fn(address, acc) ->
      if address =~ address_format do
        # if matches we return the changeset as it is
        acc
      else
        # if it doesn't, we add the error to the changeset and return the new invalid changeset
        add_error(acc, :address, "#{address} is not a valid email address")
      end
    end)
  end

  defp to_address_list(changeset) do
    # if get_change returns the default value, it means that the change is empty
    # and we return the empty list
    # otherwise we split the string on ',' and strip the spaces around them
    case get_change(changeset, :address, []) do
      []      -> []
      address -> address |> String.split(",") |> Enum.map(&String.strip/1)
    end
  end

  defp address_format do
    ~r/^\S+@\S+\.\S+$/
  end
end
```

> validations for an Email model

It is a little verbose, honestly, but it is also very clear and required a basic knowledge of the framework to build it.  
Of course validation code, being just pure functions, can be generalized and moved somewhere else (a different Module for example) to be used in different models or in different applications.

# The Repository Pattern and the macros

Phoenix does not supply any ORM or ORM like library for accesing data, instead it relies on the Repository Pattern (and the awesome [`Ecto.Query`](http://hexdocs.pm/ecto/Ecto.Query.html)), which, IMHO, is superior to Active Record any day of the week.  
Phoenix implemented it in a very generic way, using the facilities provided by `Ecto`:

```elixir
defmodule App.Repo do
  use Ecto.Repo, otp_app: :app, adapter: Sqlite.Ecto # default adapter is PostgreSQL
end

# then you can query the repository

Repo.all(Email)                                    # -> all emails
Repo.all(User)                                     # -> all users
Repo.get(User, 2)                                  # -> user with id 2
Repo.get_by(User, {name: 'Rambo')                  # -> first user whose name is Rambo
Repo.insert(%Email{address: 'marsrover@nasa.gov'}) # -> insert a new email
```

However, Active Record is a very well established and recognized pattern in web applications, it is perfectly possible that its absence could let some developer down.  
Let's add something that resembles it to our models:

```elixir
defmodule App.Email do
  # all the usual imports and schema omitted

  alias App.Repo

  # returns all Emails
  def all(opts \\ []) do
    Repo.all(Email, opts)
  end

  # find Emails by filtering out all emails not respecting clauses  
  # for example: {domain: 'email.com'}
  def find(clauses, opts \\ []) do
    Enum.filter all(opts), fn map ->
      Enum.all?(clauses, fn {key, val} -> Map.get(map, key) == val end)
    end
  end

  # get the Email with id equal to id
  def get(id, opts \\ []) do
    Repo.get(Email, id, opts)
  end

  # more functions
  # I suppose you get the pattern…
end

# now you can do this
Email.all
Email.find({domain: 'email.com'})
Email.get(1)
# etc. etc.
```

Elixir has a powerful mechanism to write generic code, [macros](http://elixir-lang.org/getting-started/meta/macros.html).  
Phoenix is an Elixir app after all, so we can use macros to write a generic `ActiveRecord` like wrapper for the `Repo` and inject it into the `Model` code at compile time (yes, at compile time, it will be like we actually wrote that code with no loss of performance whatsoever).

```elixir
defmodule App.Activerecord do
  # when we "use" a Module, Elixir calls the relative __using__ macro inside it  
  defmacro __using__(_) do
    quote do
      alias App.Repo

      # __MODULE__ is resolved at compile time with the name
      # of the caller, inside "quote do" we are in the caller context
      def all(opts \\ []) do
        Repo.all __MODULE__, opts
      end

      def find(clauses, opts \\ []) do
        Enum.filter all(opts), fn map ->
          Enum.all?(clauses, fn {key, val} -> Map.get(map, key) == val end)
        end
      end

      def get(id, opts \\ []) do
        Repo.get __MODULE__, id, opts
      end

      def get_by(clauses, opts \\ []) do
        Repo.get_by __MODULE__, clauses, opts
      end

    end

  end

end

# and then do
defmodule App.Email do
  # all the usual imports and schema omitted
  use App.Activerecord # <- this injects the code into the Email module
end

# now you can do this
Email.all
Email.find({domain: 'email.com'})
Email.get(1)

# but also this, if you use the ActiveRecord module in the User model
User.all
User.find({role: 'admin'})
User.get(1)
```

I don't know if this is a good idea, but it is very easy to implement and the mechanism is so powerful that opens up the way for writing well organized and understandable code.

Next time we'll talk about the assets pipeline and long running processes.