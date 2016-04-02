---
title: "Writing custom commands for Drush: the Drupal swiss army knife"
date: "2014-05-14"
description: Recently I worked on a client project based on the Drupal platform. The most important part of the job was automating a data import from a remote source, but instead of writing a script to do the job, I created a command for Drush.
source_name: MIKAMAYHEM
source_url: http://dev.mikamai.com/post/85715381049/writing-custom-commands-for-drush-the-drupal-swiss
tags:
- Drupal
- Drush
- Custom Drush commands
---

Recently I worked on a client project based on the Drupal platform.  
The most important part of the job was automating a data import from a remote source, but instead of writing a script to do the job, I created a command for [Drush](https://github.com/drush-ops/drush).
Quoting from Drush repository site

> Drush is a command-line shell and scripting interface for Drupal, a veritable Swiss Army knife designed to make life easier for those who spend their working hours hacking away at the command prompt.

Drush can handle almost every aspect of a Drupal site, from the mundane [cache management](http://www.drushcommands.com/drush-7x/cache) to
[user management](http://www.drushcommands.com/drush-6x/user), from [packaging a Drupal install into a makefile](http://www.drushcommands.com/drush-6x/make) to
[project management](http://www.drushcommands.com/drush-6x/pm) and much more, including a [CLI for running sql queries](http://www.drushcommands.com/drush-6x/sql/sql-cli) an [http server for development](http://www.drushcommands.com/drush-6x/runserver/runserver) and an [rsync wrapper](http://www.drushcommands.com/drush-7x/core/core-rsync).  
Drush commands can also be executed on remote machines, provided Drush is installed, by specifing the server [alias](http://deeson-online.co.uk/labs/drupal-drush-aliases-and-how-use-them) (e.g. `drush clear-cache @staging`).  

There are different ways of creating Drush scripts:  

- prepending the script with the shebang `#!/usr/bin/env drush` or `#!/full/path/to/drush` and using
[Drush commands](http://www.drushcommands.com/)
- using Drush php interpreter `#!/full/path/to/drush php-script` and using the [Drush
PHP api](http://api.drush.org/)
- writing custom commands

This guide is about the last case.  
Drush commands are much like Rake or Grunt tasks, you give them a name (more like a namespace) and Drush figures out what function must be called.
To create a Drush command, follow these simple steps

- create a `namespace.drush.inc` in one of the standard import path
- implement the `namespace_drush_command` entry point function
- implement the command functions. By conventions the command functions are called `drush_namespace_commandname`

Drush search for commandfiles in the following locations:

- `/path/to/drush/commands` folder
- system-wide drush commands folder, e.g. `/usr/share/drush/commands`
- .drush folder in `$HOME` folder.
- `sites/all/drush` in the current Drupal installation
- all enabled modules folders in the current Drupal installation

## Implementing the command

To implement a Drush command, the script must implement the drush_command hook.
This function must return a data structure containing all the informations that define your custom command.  
As an example we will develop a command that rolls a dice and prints the result.  
We'll use `diceroller` as namespace and `roll-dice` as command name.  
This is the implementation of the main hook function

```php
<?php
function diceroller_drush_command() {
  $items = array();

  $items['roll-dice'] = array(
    'description' => "Roll a dice for your pleasure.",
    'arguments' => array(
      'faces' => 'How many faces the dice has? Default is 6, max is 100.',
    ),
    'options' => array(
      'rolls' => 'How many times the dice is rolled, default is 1 max is 100',
    ),
    'examples' => array(
      'drush drrd 6 --rolls=2' => 'Rolls a 6 faced dice 2 times',
    ),
    'aliases' => array('drrd'),
    'bootstrap' => DRUSH_BOOTSTRAP_DRUSH,
    // see http://drush.ws/docs/bootstrap.html for detailed informations
    // about the bootstrap values
  );

  return $items;
}
```

The command is easily implementd this way

```php
<?php

function drush_diceroller_roll_dice($faces=6) {
  $rolls = 1;

  if ($tmp = drush_get_option('rolls')) {
    $rolls = $tmp;
  }  

  drush_print(dt('Rolling a !faces faced dice !n time(s)', array(
    '!faces' => $faces,
    '!n' => $rolls
  )));
  // for n=0..$rolls
  // roll the nth dice
  // print the result
}

```

In this case we assume that the `--rolls` option contains a number, but we can guarantee that the function parameters are valid implementing the `validate` hook (there are others called just before and after the real command function).

```php
<?php

function drush_diceroller_roll_dice_validate($faces=6) {

  if($faces <= 0) {
    return drush_set_error('DICE_WITH_NO_FACES', dt('Cannot roll a dice with no faces!'));
  }
  if($faces > 100) {
    return drush_set_error('DICE_WITH_TOO_MANY_FACES', dt('Cannot roll a sphere!'));
  }

  $rolls = drush_get_option('rolls');
  if(isset($rolls)) {
    if(!is_numeric($rolls))
      return drush_set_error('ROLLS_MUST_BE_INT', dt('rolls value must be a number!'));

    if($rolls <= 0)
      return drush_set_error('NOT_ENOUGH_ROLLS', dt('What you\'re asking cannot be done!'));

    if($rolls > 100)
      return drush_set_error('TOO_MANY_ROLLS', dt('I\'m not your slave, roll it by yourself!'));
  }

}
```

If we did our job diligently, running `drush help roll-dice` should give us this ouput

```sh
Roll a dice for your pleasure.

Examples:
 drush drrd 6 --rolls=2                    Rolls a 6 faced dice 2 times

Arguments:
 faces                                     How many faces the dice has? Default is 6, max is 100.

Options:
 --rolls                                   How many times the dice is rolled, default is 1 max is 100

Aliases: drrd
```

Consult the [Drush api](http://api.drush.org/) for a complete list of hooks [functions](http://api.drush.org/api/drush/functions/6.x) and [constants](http://api.drush.org/api/drush/constants/6.x) or launch `drush topic docs-api` from the command line.  
For a complete implementation of a command example, see `drush topic docs-examplecommand`.  
