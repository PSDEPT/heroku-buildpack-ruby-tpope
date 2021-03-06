# Heroku buildpack: Ruby: tpope edition

This is my fork of the [official Heroku Ruby
buildpack](https://github.com/heroku/heroku-buildpack-ruby).  Use it when
creating a new app:

    heroku create myapp --buildpack \
      https://github.com/tpope/heroku-buildpack-ruby-tpope

## Usage

Or add it to an existing app:

    heroku config:add \
      BUILDPACK_URL=https://github.com/tpope/heroku-buildpack-ruby-tpope

Or just cherry-pick the parts you like into your own fork.

Contained within are a few tiny but significant differences from the official
version, distilled from project-specific buildpacks I've created in the past.
    $ heroku create --buildpack https://github.com/heroku/heroku-buildpack-ruby.git

## Custom compilation tasks

If the `COMPILE_TASKS` config variable is set, it will be passed verbatim to a
`rake` invocation.

You can use this for all sorts of things.  My favorite is `db:migrate`.

### Database migration during compilation

Let's take a look at the standard best practice for deploying Rails apps to
Heroku:

1.  `heroku maintenance:on`.
2.  `git push heroku master`.  This restarts the application when complete.  If
    you have any schema additions, your app is now broken (hence the need for
    maintenance mode).
3.  `heroku run rake db:migrate`.
4.  `heroku restart`.  This is necessary so the app picks up on the schema
    changes.
5.  `heroku maintenance:off`.

That's five different commands, none of them instantaneous, and two restarts.
The most common response to this mess is to wrap deployment up in a Rake task,
but now you have two problems: a suboptimal deployment procedure, and
application code concerned with deployment.

Now let's take a look at a typical deploy when `COMPILE_TASKS` includes
`db:migrate`:

1.  `git push heroku master`.
    * First the standard stuff happens.  Bundling, asset precompilation, that
      sort of thing.
    * `rake db:migrate` fires.  The app continues working unless the
      migrations drop something from the schema.
    * The app restarts.  Everything is wonderful.

We've reduced it to a single step, limiting our need for maintenance mode to
destructive migrations.  Even in that case, it's not always strictly
necessary, since the window for breakage is frequently only a few seconds.  Or
with a bit of planning, you can [avoid this situation entirely][no downtime].

[Twelve-factor][] snobs (of which I am one) would generally argue that
[admin processes][] belong in the run stage, not the build stage.  I agree in
theory, but it in practice, boy does this make things a whole lot simpler.
#### Bundler

[no downtime]: http://pedro.herokuapp.com/past/2011/7/13/rails_migrations_with_no_downtime/
[Twelve-factor]: http://www.12factor.net/
[Admin processes]: http://www.12factor.net/admin-processes

## Commit recording

**Broken and disabled pending further investigation.**

This takes the upcoming and previously deployed commit SHAs and makes them
available as `$REVISION` and `$ORIGINAL_REVISION` for the duration of the
compile.  They are also written to `HEAD` and `ORIG_HEAD` in the root of the
application for easy access after the deploy is complete.

These can be used from `COMPILE_TASKS` to make a poor man's post-deploy hook.
    $ ls
    app  config  db  doc  Gemfile  Gemfile.lock  lib  log  public  Rakefile  README  script  test  tmp  vendor

    $ ls config/environment.rb
    config/environment.rb

    $ heroku create --buildpack https://github.com/heroku/heroku-buildpack-ruby.git

    $ git push heroku master
    ...
    -----> Heroku receiving push
    -----> Ruby/Rails app detected
    -----> Installing dependencies using Bundler version 1.1.rc
    ...
    -----> Writing config/database.yml to read from DATABASE_URL
    -----> Rails plugin injection
           Injecting rails_log_stdout
    -----> Discovering process types
           Procfile declares types      -> (none)
           Default types for Ruby/Rails -> console, rake, web, worker

The buildpack will detect your app as a Rails 2 app if it has a `environment.rb` file in the `config`  directory.

#### Rails Log STDOUT
A [rails_log_stdout](http://github.com/ddollar/rails_log_stdout) is installed by default so Rails' logger will log to STDOUT and picked up by Heroku's [logplex](http://github.com/heroku/logplex).

#### Auto Injecting Plugins

Any vendored plugin can be stopped from being installed by creating the directory it's installed to in the slug. For instance, to prevent rails_log_stdout plugin from being injected, add `vendor/plugins/rails_log_stdout/.gitkeep` to your git repo.

### Rails 3

Example Usage:

    $ ls
    app  config  config.ru  db  doc  Gemfile  Gemfile.lock  lib  log  Procfile  public  Rakefile  README  script  tmp  vendor

    $ ls config/application.rb
    config/application.rb

    $ heroku create --buildpack https://github.com/heroku/heroku-buildpack-ruby.git

    $ git push heroku master
    -----> Heroku receiving push
    -----> Ruby/Rails app detected
    -----> Installing dependencies using Bundler version 1.1.rc
           Running: bundle install --without development:test --path vendor/bundle --deployment
           ...
    -----> Writing config/database.yml to read from DATABASE_URL
    -----> Preparing app for Rails asset pipeline
           Running: rake assets:precompile
    -----> Rails plugin injection
           Injecting rails_log_stdout
           Injecting rails3_serve_static_assets
    -----> Discovering process types
           Procfile declares types      -> web
           Default types for Ruby/Rails -> console, rake, worker

The buildpack will detect your apps as a Rails 3 app if it has an `application.rb` file in the `config` directory.

#### Assets

To enable static assets being served on the dyno, [rails3_serve_static_assets](http://github.com/pedro/rails3_serve_static_assets) is installed by default. If the [execjs gem](http://github.com/sstephenson/execjs) is detected then [node.js](http://github.com/joyent/node) will be vendored. The `assets:precompile` rake task will get run if no `public/manifest.yml` is detected.  See [this article](http://devcenter.heroku.com/articles/rails31_heroku_cedar) on how rails 3.1 works on cedar.

## Documentation

For more information about using Ruby and buildpacks on Heroku, see these Dev Center articles:

- [Heroku Ruby Support](https://devcenter.heroku.com/articles/ruby-support)
- [Getting Started with Ruby on Heroku](https://devcenter.heroku.com/articles/getting-started-with-ruby)
- [Getting Started with Rails 4 on Heroku](https://devcenter.heroku.com/articles/getting-started-with-rails4)
- [Buildpacks](https://devcenter.heroku.com/articles/buildpacks)
- [Buildpack API](https://devcenter.heroku.com/articles/buildpack-api)

## Hacking

To use this buildpack, fork it on Github.  Push up changes to your fork, then create a test app with `--buildpack <your-github-url>` and push to it.

To change the vendored binaries for Bundler, [Node.js](http://github.com/joyent/node), and rails plugins, use the rake tasks provided by the `Rakefile`. You'll need an S3-enabled AWS account and a bucket to store your binaries in as well as the [vulcan](http://github.com/heroku/vulcan) gem to build the binaries on heroku.

For example, you can change the vendored version of Bundler to 1.1.rc.

First you'll need to build a Heroku-compatible version of Node.js:

    $ export AWS_ID=xxx AWS_SECRET=yyy S3_BUCKET=zzz
    $ s3 create $S3_BUCKET
    $ rake gem:install[bundler,1.1.rc]

Open `lib/language_pack/ruby.rb` in your editor, and change the following line:

    BUNDLER_VERSION = "1.11.2"

Open `lib/language_pack/base.rb` in your editor, and change the following line:

    VENDOR_URL = "https://s3.amazonaws.com/zzz"

Commit and push the changes to your buildpack to your Github fork, then push your sample app to Heroku to test.  You should see:

    -----> Installing dependencies using Bundler version 1.1.rc

NOTE: You'll need to vendor the plugins, node, Bundler, and libyaml by running the rake tasks for the buildpack to work properly.

### Testing

The tests on this buildpack are written in Rspec to allow the use of
`focused: true`. Parallelization of testing is provided by
https://github.com/grosser/parallel_tests this lib spins up an arbitrary
number of processes and running a different test file in each process,
it does not parallelize tests within a test file. To run the tests: clone the repo, then `bundle install` then clone the test fixtures by running:

```sh
$ bundle exec hatchet install
```

then go to [hatchet](https://github.com/heroku/hatchet) repo and follow the
instructions to set it up.

Now run the tests:

```sh
$ bundle exec parallel_rspec -n 6 spec/
```

If you don't want to run them in parallel you can still:

```sh
$ bundle exec rake spec
```

Now go take a nap or do something for a really long time.
