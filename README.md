# Laravel Skeema

[![phpunit](https://github.com/smakecloud/skeema/actions/workflows/phpunit.yml/badge.svg)](https://github.com/smakecloud/skeema/actions/workflows/phpunit.yml)
[![Latest Version on Packagist](https://img.shields.io/packagist/v/smakecloud/skeema.svg?style=flat-square)](https://packagist.org/packages/smakecloud/skeema)
[![Total Downloads](https://img.shields.io/packagist/dt/smakecloud/skeema.svg?style=flat-square)](https://packagist.org/packages/smakecloud/skeema)

This package provides a Laravel wrapper around the [Skeema](https://www.skeema.io/) tool.

Skeema is a tool for managing MySQL database schemas.
It allows you to define your database schema in simple SQL files,
and then use Skeema to keep your database schema in sync.

We also included a helper command to make it easier to transition from the default Laravel migrations to Skeema.
See [Laravel Migrations to Skeema "converting"](#laravel-migrations-to-skeema-converting) for more information.

If you want to run migrations from CI / CD pipelines checkout [Deployment Checking](#deployment-checking) for a way to check if the current laravel app includes any classic migrations or dumps that could break your deployment.

## Table of Contents

- [Requirements](#requirements)
- [Installation](#installation)
- [Configuration](#configuration)
- [Usage](#usage)
    - [Dumping the schema](#dumping-the-schema)
    - [Linting the schema](#linting-the-schema)
    - [Diffing the schema](#diffing-the-schema)
    - [Pushing the schema](#pushing-the-schema)
    - [Pulling the schema](#pulling-the-schema)
    - [Deployment Checking](#deployment-checking)
    - [Laravel Migrations to Skeema "converting"](#laravel-migrations-to-skeema-converting)
- [Testing](#testing)
- [Disclaimer](#disclaimer)
- [License](#license)
- [Credits](#credits)

## Requirements

- PHP 8.1 or higher
- [Laravel](https://laravel.com/) ( tested with laravel version 9 )
- [Skeema](https://www.skeema.io/) ( tested with skeema version 1.9.0-community )

**Optional**

- [gh-ost](https://github.com/github/gh-ost) ( tested with gh-ost version 1.1.5 )
- [percona-toolkit (pt-online-schema-change)](https://www.percona.com/software/database-tools/percona-toolkit) ( untested ! )
## Installation

If you haven't already, install [Skeema](https://www.skeema.io/download/).

We also recommend installing [gh-ost](https://github.com/github/gh-ost/releases). This is a tool that allows you to perform online schema changes without locking the table. ( Percona's pt-online-schema-change is also supported by Skeema but untested by us. )

You can install the package via composer:

```bash
composer require smakecloud/skeema

php artisan vendor:publish --provider="SmakeCloud\Skeema\SkeemaServiceProvider"
```

## Configuration

``` php
<?php

return [
    /*
     * The path to the skeema binary.
     */
    'bin' => env('SKEEMA_BIN', 'skeema'),

    /*
     * The directory where the schema files will be stored.
     */
    'dir' => 'database/skeema',

    /*
     * The connection to use when dumping the schema.
     */
    'connection' => env('DB_CONNECTION', 'mysql'),

    /**
     * Alter Wrapper
     */
    'alter_wrapper' => [
        /*
         * Enable the alter wrapper.
         */
        'enabled' => env('SKEEMA_WRAPPER_ENABLED', false),

        /*
         * The path to the wrapper binary.
         */
        'bin' => env('SKEEMA_WRAPPER_BIN', 'gh-ost'),

        /**
         * Any table smaller than this size (in bytes) will ignore the alter-wrapper option. This permits skipping the overhead of external OSC tools when altering small tables.
         */
        'min_size' => '0',

        /**
         * This is how we do it at Smake.
         * We highly recommend you to read documentation of
         * gh-ost or pt-online-schema-change.
         * https://github.com/github/gh-ost/blob/master/doc/command-line-flags.md
         * https://docs.percona.com/percona-toolkit/pt-online-schema-change.html
         */
        'params' => [
            '--max-load=Threads_running=25',
            '--critical-load=Threads_running=1000',
            '--chunk-size=1000',
            '--throttle-control-replicas='.env('DB_REPLICAS'),
            '--max-lag-millis=1500',
            '--verbose',
            '--assume-rbr',
            '--allow-on-master',
            '--cut-over=default',
            '--exact-rowcount',
            '--concurrent-rowcount',
            '--default-retries=120',
            '--timestamp-old-table',
            // https://github.com/github/gh-ost/blob/master/doc/command-line-flags.md#postpone-cut-over-flag-file
            '--postpone-cut-over-flag-file=/tmp/ghost.postpone.flag',
        ],
    ],

    /**
     * Linter specific config
     * lint, diff, push
     */
    'lint' => [
        /**
         * Linting rules for all supported cmds
         */
        'rules' => [
            \Smakecloud\Skeema\Lint\AutoIncRule::class => 'warning',
            \Smakecloud\Skeema\Lint\CharsetRule::class => 'warning',
            \Smakecloud\Skeema\Lint\CompressionRule::class => 'warning',
            \Smakecloud\Skeema\Lint\DefinerRule::class => 'error',
            \Smakecloud\Skeema\Lint\DisplayWidthRule::class => 'warning',
            \Smakecloud\Skeema\Lint\DupeIndexRule::class => 'error',
            \Smakecloud\Skeema\Lint\EngineRule::class => 'warning',
            \Smakecloud\Skeema\Lint\HasEnumRule::class => 'ignore',
            \Smakecloud\Skeema\Lint\HasFkRule::class => 'ignore',
            \Smakecloud\Skeema\Lint\HasFloatRule::class => 'ignore',
            \Smakecloud\Skeema\Lint\HasRoutineRule::class => 'ignore',
            \Smakecloud\Skeema\Lint\HasTimeRule::class => 'ignore',
            \Smakecloud\Skeema\Lint\NameCaseRule::class => 'ignore',
            \Smakecloud\Skeema\Lint\PkRule::class => 'warning',
            \Smakecloud\Skeema\Lint\ZeroDateRule::class => 'warning',

        /**
         * These rules are disabled by default
         * because they are not available in the Community edition of Skeema
         *
         * https://www.skeema.io/download/
         */

            // \Smakecloud\Skeema\Lint\HasTriggerRule::class => 'error',
            // \Smakecloud\Skeema\Lint\HasViewRule::class => 'error',
        ],

        /**
         * Linting rules for diff
         * Set to false to disable linting for diff
         * See https://www.skeema.io/docs/commands/diff
         */
        'diff' => [
            // \Smakecloud\Skeema\Lint\ZeroDateRule::class => 'error',
        ],

        /**
         * Linting rules for push
         * Set to false to disable linting for push
         * See https://www.skeema.io/docs/commands/push
         */
        'push' => [
            // \Smakecloud\Skeema\Lint\ZeroDateRule::class => 'error',
        ],
    ],
];
```

## Usage

### Dumping the schema

Run this once against your production database to generate the initial schema files.
This will also create a `.skeema` configuratiob file in configured skeema base dir.


We use env var interpolation to inject the database credentials from laravel into the config file.

```shell
$ php artisan skeema:init
```

```
Description:
  Inits the database schema

Usage:
  skeema:init [options]

Options:
      --force
      --connection[=CONNECTION]
  -h, --help                     Display help for the given command. When no command is given display help for the list command
  -q, --quiet                    Do not output any message
  -V, --version                  Display this application version
      --ansi|--no-ansi           Force (or disable --no-ansi) ANSI output
  -n, --no-interaction           Do not ask any interactive question
      --env[=ENV]                The environment the command should run under
  -v|vv|vvv, --verbose           Increase the verbosity of messages: 1 for normal output, 2 for more verbose output and 3 for debug
```

---

Example .skeema file:

```ini
generator=skeema:1.9.0-community

[laravel]
flavor=mysql:5.7
host=$LARAVEL_SKEEMA_DB_HOST
password=$LARAVEL_SKEEMA_DB_PASSWORD
port=$LARAVEL_SKEEMA_DB_PORT
schema=$LARAVEL_SKEEMA_DB_SCHEMA
user=$LARAVEL_SKEEMA_DB_USER
```

### Linting the schema

Lint the schema files with your configured rules.

Take a look at skeema [linting documentation](https://www.skeema.io/docs/commands/lint/) for more information.

```shell
$ php artisan skeema:lint
```

```
Description:
  Lint the database schema

Usage:
  skeema:lint [options]

Options:
      --skip-format                            Skip formatting the schema files
      --strip-definer[=STRIP-DEFINER]          Remove DEFINER clauses from *.sql files
      --strip-partitioning                     Remove PARTITION BY clauses from *.sql files
      --allow-auto-inc[=ALLOW-AUTO-INC]        List of allowed auto_increment column data types for lint-auto-inc
      --allow-charset[=ALLOW-CHARSET]          List of allowed character sets for lint-charset
      --allow-compression[=ALLOW-COMPRESSION]  List of allowed compression settings for lint-compression
      --allow-definer[=ALLOW-DEFINER]          List of allowed routine definers for lint-definer
      --allow-engine[=ALLOW-ENGINE]            List of allowed storage engines for lint-engine
      --update-views                           Reformat views in canonical single-line form
      --ignore-warnings                        Exit with status 0 even if warnings are found
      --output-format[=OUTPUT-FORMAT]          Output format for lint results. Valid values: default, github, quiet
      --connection[=CONNECTION]
  -h, --help                                   Display help for the given command. When no command is given display help for the list command
  -q, --quiet                                  Do not output any message
  -V, --version                                Display this application version
      --ansi|--no-ansi                         Force (or disable --no-ansi) ANSI output
  -n, --no-interaction                         Do not ask any interactive question
      --env[=ENV]                              The environment the command should run under
  -v|vv|vvv, --verbose                         Increase the verbosity of messages: 1 for normal output, 2 for more verbose output and 3 for debug
```

### Diffing the schema

Diff the schema files against the database.

Take a look at skeema [diffing documentation](https://www.skeema.io/docs/commands/diff/) for more information.

```shell
$ php artisan skeema:diff
```

```
Description:
  Diff the database schema

Usage:
  skeema:diff [options]

Options:
      --ignore-warnings                        No error will be thrown if there are warnings
      --alter-algorithm[=ALTER-ALGORITHM]      The algorithm to use for ALTER TABLE statements
      --alter-lock[=ALTER-LOCK]                The lock to use for ALTER TABLE statements
      --alter-validate-virtual                 Apply a WITH VALIDATION clause to ALTER TABLEs affecting virtual columns
      --compare-metadata                       For stored programs, detect changes to creation-time sql_mode or DB collation
      --exact-match                            Follow *.sql table definitions exactly, even for differences with no functional impact
      --partitioning[=PARTITIONING]            Specify handling of partitioning status on the database side
      --strip-definer[=STRIP-DEFINER]          Ignore DEFINER clauses when comparing procs, funcs, views, or triggers
      --allow-auto-inc[=ALLOW-AUTO-INC]        List of allowed auto_increment column data types for lint-auto-inc
      --allow-charset[=ALLOW-CHARSET]          List of allowed character sets for lint-charset
      --allow-compression[=ALLOW-COMPRESSION]  List of allowed compression settings for lint-compression
      --allow-definer[=ALLOW-DEFINER]          List of allowed routine definers for lint-definer
      --allow-engine[=ALLOW-ENGINE]            List of allowed storage engines for lint-engine
      --allow-unsafe                           Permit generating ALTER or DROP operations that are potentially destructive
      --safe-below-size[=SAFE-BELOW-SIZE]      Always permit generating destructive operations for tables below this size in bytes
      --skip-verify                            Skip Test all generated ALTER statements on temp schema to verify correctness
      --connection[=CONNECTION]
  -h, --help                                   Display help for the given command. When no command is given display help for the list command
  -q, --quiet                                  Do not output any message
  -V, --version                                Display this application version
      --ansi|--no-ansi                         Force (or disable --no-ansi) ANSI output
  -n, --no-interaction                         Do not ask any interactive question
      --env[=ENV]                              The environment the command should run under
  -v|vv|vvv, --verbose                         Increase the verbosity of messages: 1 for normal output, 2 for more verbose output and 3 for debug
```

### Pushing the schema

Push the schema files to the database.

Take a look at skeema [pushing documentation](https://www.skeema.io/docs/commands/push/) for more information.

```shell
$ php artisan skeema:push
```

```
Description:
  Push the database schema

Usage:
  skeema:push [options]

Options:
      --alter-algorithm[=ALTER-ALGORITHM]      Apply an ALGORITHM clause to all ALTER TABLEs
      --alter-lock[=ALTER-LOCK]                Apply a LOCK clause to all ALTER TABLEs
      --alter-validate-virtual                 Apply a WITH VALIDATION clause to ALTER TABLEs affecting virtual columns
      --compare-metadata                       For stored programs, detect changes to creation-time sql_mode or DB collation
      --exact-match                            Follow *.sql table definitions exactly, even for differences with no functional impact
      --partitioning[=PARTITIONING]            Specify handling of partitioning status on the database side
      --strip-definer[=STRIP-DEFINER]          Ignore DEFINER clauses when comparing procs, funcs, views, or triggers
      --allow-auto-inc[=ALLOW-AUTO-INC]        List of allowed auto_increment column data types for lint-auto-inc
      --allow-charset[=ALLOW-CHARSET]          List of allowed character sets for lint-charset
      --allow-compression[=ALLOW-COMPRESSION]  List of allowed compression settings for lint-compression
      --allow-definer[=ALLOW-DEFINER]          List of allowed routine definers for lint-definer
      --allow-engine[=ALLOW-ENGINE]            List of allowed storage engines for lint-engine
      --allow-unsafe                           Permit generating ALTER or DROP operations that are potentially destructive
      --safe-below-size[=SAFE-BELOW-SIZE]      Always permit generating destructive operations for tables below this size in bytes
      --skip-verify                            Skip Test all generated ALTER statements on temp schema to verify correctness
      --dry-run                                Output DDL but don’t run it; equivalent to skeema diff
      --foreign-key-checks                     Force the server to check referential integrity of any new foreign key
      --force
      --connection[=CONNECTION]
  -h, --help                                   Display help for the given command. When no command is given display help for the list command
  -q, --quiet                                  Do not output any message
  -V, --version                                Display this application version
      --ansi|--no-ansi                         Force (or disable --no-ansi) ANSI output
  -n, --no-interaction                         Do not ask any interactive question
      --env[=ENV]                              The environment the command should run under
  -v|vv|vvv, --verbose                         Increase the verbosity of messages: 1 for normal output, 2 for more verbose output and 3 for debug
```

### Pulling the schema

Pull the schema files from the database.

Take a look at skeema [pulling documentation](https://www.skeema.io/docs/commands/pull/) for more information.

```shell
$ php artisan skeema:pull
```

```
Description:
  Pull the database schema

Usage:
  skeema:pull [options]

Options:
      --skip-format                    Skip Reformat SQL statements to match canonical SHOW CREATE
      --include-auto-inc               Include starting auto-inc values in new table files, and update in existing files
      --new-schemas                    Detect any new schemas and populate new dirs for them (enabled by default; disable with skip-new-schemas)
      --strip-definer[=STRIP-DEFINER]  Omit DEFINER clauses when writing procs, funcs, views, and triggers to filesystem
      --strip-partitioning             Omit PARTITION BY clause when writing partitioned tables to filesystem
      --update-views                   Update definitions of existing views, using canonical form
      --update-partitioning            Update PARTITION BY clauses in existing table files
      --connection[=CONNECTION]
  -h, --help                           Display help for the given command. When no command is given display help for the list command
  -q, --quiet                          Do not output any message
  -V, --version                        Display this application version
      --ansi|--no-ansi                 Force (or disable --no-ansi) ANSI output
  -n, --no-interaction                 Do not ask any interactive question
      --env[=ENV]                      The environment the command should run under
  -v|vv|vvv, --verbose                 Increase the verbosity of messages: 1 for normal output, 2 for more verbose output and 3 for debug
```

### Deployment Checking

**This should not be used in production environments, run it in a dedicated CI environment !**

This custom command checks for existing laravel migrations,
mysql-dump files, or running gh-ost migrations.

This can be usefull for pre-deployment checks ( in CI/CD pipelines ).

```shell
$ php artisan skeema:check-deployment
```

### Laravel Migrations to Skeema "converting"

**This should not be used in production environments, run it in development environments only !**

This custom command "converts" existing laravel migrations to skeema schema files.
This is achieved by executing the following steps:

1. Force pushing the current skeema files to the database ( Skippable with `--no-push` )
2. Looping through existing laravel migrations
    - If the have been executed already, they will be deleted
    - If they haven't been executed yet, they will be executed and then deleted
3. Pulling the new skeema files from the database

```shell
$ php artisan skeema:convert-migrations
```


## Testing

```shell
$ composer test
```

**With coverage**

```shell
$ composer test:coverage
```

## Disclaimer

This package is not affiliated with Skeema in any way.

**Read the documentation of Skeema before using this package !**

**We don't take any responsibility for any damage caused by this package.**

## License

The MIT License (MIT). Please see [License File](LICENSE) for more information.

## Credits

- [Daursu](https://github.com/Daursu) - for the initial idea
- [Skeema](https://www.skeema.io/) - making all of this possible
- [GitHub](https://github.com) - gh-ost, skeema
- [Percona](https://www.percona.com/) - pt-online-schema-change
- [Smake® IT GmbH](https://github.com/smakecloud)
