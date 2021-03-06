# New York, New York.
(very) small Sinatra clone.

[![Build Status](https://api.travis-ci.org/alisnic/nyny.png)](https://travis-ci.org/alisnic/nyny)
[![Coverage Status](https://coveralls.io/repos/alisnic/nyny/badge.png)](https://coveralls.io/r/alisnic/nyny)
[![Code Climate](https://codeclimate.com/repos/521b7ee513d637348712864a/badges/60e3637788bbac94f1cb/gpa.png)](https://codeclimate.com/repos/521b7ee513d637348712864a/feed)

    # myapp.rb

    require 'nyny'
    class App < NYNY::App
      get '/' do
        'Hello world!'
      end
    end

    App.run!
    
Install the gem:

    gem install nyny
    
Run the file:
    
    ruby myapp.rb
    
Open the browser at [http://localhost:9292](http://localhost:9292)

- [TOP](#new-york-new-york)
- [Motivation](#motivation)
- [Philosophy](#philosophy)
- [Why use NYNY instead of Sinatra](#why-use-nyny-instead-of-sinatra)
- [Usage](#usage)
    - [Running](#running)
    - [Defining routes](#defining-routes)
    - [Request scope](#request-scope)
    - [Filters](#filters)
    - [Middleware](#middleware)
    - [Helpers](#helpers)
- [FAQ](#f-a-q)
- [Contributing](#contributing)

# Motivation
My efforts to write __NYNY__ started when I wanted to understand how __Sinatra__
works, and stumbled upon the [base.rb][0]. The majority of the classes that
are used by sinatra are in one single file, which makes it nearly impossible
for a new person to grasp.

I wanted to understand how sinatra works, but the code was pretty challenging.
So I decided I should re-implement the basic things Sinatra has.
Thus, __NYNY__ was born.

# Philosophy
NYNY should have only the bare minimum to write basic web servers comfortably,
everything else should be in a extension. It is also
trivial to use NYNY to build large and complex apps, by writing multiple sub
apps and using Rack to mount them, or by using those sub apps in the "main" app
as middleware.

# Why use NYNY instead of Sinatra
- It's very small (<300 LOC), which is just a little overhead on top of Rack.
- Sinatra is a drop-in replacement for NYNY. Anytime you feel that you need more,
  you can just change your app to inherit from `Sinatra::Base`, your code will
  still work, and you will be able to use any of the Sinatra features.
- It's __~2 times faster__ than Sinatra (see [Performance][performance] for details)
- You want to dig into the source code and change to your needs (NYNY's source code is more welcoming)
- Each NYNY app is a Rack middleware, so it can be used inside of Sinatra, Rails, or any other Rack-based app.

# Usage

A NYNY app must _always_ be in a class which inherits from `NYNY::App`:

    class App < NYNY::App
      get '/' do
        'Hello, World'
      end
    end

## Running
There are two ways to run a NYNY app __directly__ [[?]](#middleware):

- by requiring it in a `config.ru` file, and then passing it as argument to the
Rack's `run` function:

        # config.ru
        
        require 'app'
        run App.new

- by using the `run!` method directly on the app class:

        # app.rb
        
        # ...app class definition...

        App.run!

`run!` takes the port number as optional argument (the default port is 9292).
Also the `run!` method will include 2 default middlewares to make the
development easier: Rack::CommonLogger and Rack::ShowExceptions.
This will show all requests in the log, and will provide useful details
in the case a error occurs during a request.


## Defining routes

NYNY supports the following verbs for defining a route: delete, get, head,
options, patch, post, put and trace.

    class App < NYNY::App
      post '/' do
        'You Posted, dude!'
      end
    end

NYNY also suports basic URL patterns:

    class App < NYNY::App
      get '/greet/:first_name/:last_name' do
        # the last expression in the block is _always_ considered the response body.
        "Hello #{params[:first_name]} #{params[:last_name]}!"
      end
    end

you can also tell NYNY to match a regex for a path:

    class App < NYNY::App
      get /html/ do
        'Your URL contains html!'
      end
    end

Each block that is passed to a route definition is evaluated in the context of
a request scope. See below what methods are available there.

## Request scope
As was said above, when you pass a block to a route definition,
that block is evaluated in the context of a [RequestScope][2].
This means that several methods/objects available inside that block:

- `request` - A `Rack::Request` object which encapsulates the request 
  to that route. (see [Rack::Request documentation][3] for more info)
- `params` - a hash which contains both POST body params and GET querystring params.
- `headers` - allows you to read/add headers to the response
  (ex: `headers 'Content-Type' => 'text/html'`)
- `status` - allows you to set the status of the response (ex: `status 403`)
- `redirect_to` - sets the response to redirect
  (ex: `redirect_to 'http://google.com'`)
- `cookies` - a hash which allows you to access/modify/remove cookies
  (ex: `cookies[:foo] = 'bar'`)
- `session` - a hash which allows you to access/modify/remove session variables
  (ex: `session[:foo] = 'bar'`)
- `halt` - allows you to instantly return a response, interrupting current
  handler execution (see [halt][halt-definition])

## Filters

Unlike Sinatra, NYNY supports only "generic" before and after filters.
This means that you can't declare a filter to execute depending on a URL pattern.
However, you can obtain the same effect by calling next in a before block
if the request.path matches a pattern.

    class App < NYNY::App
      before do
        next unless /html/ =~ request.path
        headers 'Content-Type' => 'text/html'
      end

      after do
        puts response.inspect
      end

      get '/' do
        'hello'
      end
    end

Before and after filters are also evaluated in a RequestScope context.
A little exception are the after filters, which can access
the __response__ object ([Rack::Response][4]).

## Middleware

A NYNY app is a Rack middleware, which means that it can be used inside 
Sinatra, Rails, or any other Rack-based app:

    class MyApp < Sinatra::Base
      use MyNYNYApp
    end

NYNY also supports middleware itself, and that means you can use Rack middleware
(or a Sinatra app) inside a NYNY app:

    class App < NYNY::App
      # this will serve all the files in the "public" folder
      use Rack::Static :url => ['public']
      use SinatraApp
    end

I recommend looking at [the list of Rack middlewares][rack-middleware]

## Helpers

NYNY supports helpers as Sinatra does:

    class App < NYNY::App
      helpers MyHelperModule
    end

Using a helper implies that the helper module is included in the [RequestScope][2],
and that all the methods in that module will be available inside a route
definition block.

# F. A. Q.
TBD.

# Contributing

1. Fork it
2. Create your feature branch (`git checkout -b my-new-feature`)
3. Commit your changes (`git commit -am 'Add some feature'`)
4. Push to the branch (`git push origin my-new-feature`)
5. Create new Pull Request

[0]: https://github.com/sinatra/sinatra/blob/master/lib/sinatra/base.rb
[1]: https://github.com/sinatra/sinatra/pull/716
[2]: https://github.com/alisnic/nyny/blob/master/lib/nyny/request_scope.rb
[3]: http://rack.rubyforge.org/doc/classes/Rack/Request.html
[4]: http://rack.rubyforge.org/doc/classes/Rack/Response.html
[performance]: https://github.com/alisnic/nyny/blob/master/Performance.md
[rack-middleware]: https://github.com/rack/rack/wiki/List-of-Middleware
[halt-definition]: https://github.com/alisnic/nyny/blob/master/lib/nyny/request_scope.rb#L36
