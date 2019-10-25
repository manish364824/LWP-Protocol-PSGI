# NAME

LWP::Protocol::PSGI - Override LWP's HTTP/HTTPS backend with your own PSGI application

# SYNOPSIS

    use LWP::UserAgent;
    use LWP::Protocol::PSGI;

    # $app can be any PSGI application: Mojolicious, Catalyst or your own
    my $app = do {
        use Dancer;
        set apphandler => 'PSGI';
        get '/search' => sub {
            return 'searching for ' . params->{q};
        };
        dance;
    };

    # Register the $app to handle all LWP requests
    LWP::Protocol::PSGI->register($app);

    # can hijack any code or module that uses LWP::UserAgent underneath, with no changes
    my $ua  = LWP::UserAgent->new;
    my $res = $ua->get("http://www.google.com/search?q=bar");
    print $res->content; # "searching for bar"

    # Only hijacks specific host (and port)
    LWP::Protocol::PSGI->register($psgi_app, host => 'localhost:3000');

    my $ua = LWP::UserAgent->new;
    $ua->get("http://localhost:3000/app"); # this routes $app
    $ua->get("http://google.com/api");     # this doesn't - handled with actual HTTP requests

# DESCRIPTION

LWP::Protocol::PSGI is a module to hijack **any** code that uses
[LWP::UserAgent](https://metacpan.org/pod/LWP::UserAgent) underneath such that any HTTP or HTTPS requests can
be routed to your own PSGI application.

Because it works with any code that uses LWP, you can override various
WWW::\*, Net::\* or WebService::\* modules such as [WWW::Mechanize](https://metacpan.org/pod/WWW::Mechanize),
without modifying the calling code or its internals.

    use WWW::Mechanize;
    use LWP::Protocol::PSGI;

    LWP::Protocol::PSGI->register($my_psgi_app);

    my $mech = WWW::Mechanize->new;
    $mech->get("http://amazon.com/"); # $my_psgi_app runs

# TESTING

This module is extremely handy if you have tests that run HTTP
requests against your application and want them to work with both
internal and external instances.

    # in your .t file
    use Test::More;
    use LWP::UserAgent;

    unless ($ENV{TEST_LIVE}) {
        require LWP::Protocol::PSGI;
        my $app = Plack::Util::load_psgi("app.psgi");
        LWP::Protocol::PSGI->register($app);
    }

    my $ua = LWP::UserAgent->new;
    my $res = $ua->get("http://myapp.example.com/");
    is $res->code, 200;
    like $res->content, qr/Hello/;

This test script will by default route all HTTP requests to your own
PSGI app defined in `$app`, but with the environment variable
`TEST_LIVE` set, runs the requests against the live server.

You can also combine [Plack::App::Proxy](https://metacpan.org/pod/Plack::App::Proxy) with [LWP::Protocol::PSGI](https://metacpan.org/pod/LWP::Protocol::PSGI)
to route all requests made in your test against a specific server.

    use LWP::Protocol::PSGI;
    use Plack::App::Proxy;

    my $app = Plack::App::Proxy->new(remote => "http://testapp.local:3000")->to_app;
    LWP::Protocol::PSGI->register($app);

    my $ua = LWP::UserAgent->new;
    my $res = $ua->request("http://testapp.com"); # this hits testapp.local:3000

# METHODS

- register

        LWP::Protocol::PSGI->register($app, %options);
        my $guard = LWP::Protocol::PSGI->register($app, %options);

    Registers an override hook to hijack HTTP requests. If called in a
    non-void context, returns a guard object that automatically resets
    the override when it goes out of context.

        {
            my $guard = LWP::Protocol::PSGI->register($app);
            # hijack the code using LWP with $app
        }

        # now LWP uses the original HTTP implementations

    When `%options` is specified, the option limits which URL and hosts
    this handler overrides. You can either pass `host` or `uri` to match
    requests, and if it doesn't match, the handler falls back to the
    original LWP HTTP protocol implementor.

        LWP::Protocol::PSGI->register($app, host => 'www.google.com');
        LWP::Protocol::PSGI->register($app, host => qr/\.google\.com$/);
        LWP::Protocol::PSGI->register($app, uri => sub { my $uri = shift; ... });

    The options can take either a string, where it does a complete match, a
    regular expression or a subroutine reference that returns boolean
    given the value of `host` (only the hostname) or `uri` (the whole
    URI, including query parameters).

- unregister

        LWP::Protocol::PSGI->unregister;

    Resets all the overrides for LWP. If you use the guard interface
    described above, it will be automatically called for you.

# DIFFERENCES WITH OTHER MODULES

## Mock vs Protocol handlers

There are similar modules on CPAN that allows you to emulate LWP
requests and responses. Most of them are implemented as a mock
library, which means it doesn't go through the LWP guts and just gives
you a wrapper for receiving HTTP::Request and returning HTTP::Response
back.

LWP::Protocol::PSGI is implemented as an LWP protocol handler and it
allows you to use most of the LWP extensions to add capabilities such
as manipulating headers and parsing cookies.

## Test::LWP::UserAgent

[Test::LWP::UserAgent](https://metacpan.org/pod/Test::LWP::UserAgent) has the similar concept of overriding LWP
request method with particular PSGI applications. It has more features
and options such as passing through the requests to the native LWP
handler, while LWP::Protocol::PSGI only allows one to map certain hosts
and ports.

Test::LWP::UserAgent requires you to change the instantiation of
UserAgent from `LWP::UserAgent->new` to `Test::LWP::UserAgent->new` somehow and it's your responsibility to
do so. This mechanism gives you more control which requests should go
through the PSGI app, and it might not be difficult if the creation is
done in one place in your code base. However it might be hard or even
impossible when you are dealing with third party modules that calls
LWP::UserAgent inside.

LWP::Protocol::PSGI affects the LWP calling code more globally, while
having an option to enable it only in a specific block, thus there's
no need to change the UserAgent object manually, whether it is in your
code or CPAN modules.

# AUTHOR

Tatsuhiko Miyagawa <miyagawa@bulknews.net>

# COPYRIGHT

Copyright 2011- Tatsuhiko Miyagawa

# LICENSE

This library is free software; you can redistribute it and/or modify
it under the same terms as Perl itself.

# SEE ALSO

[Plack::Client](https://metacpan.org/pod/Plack::Client) [LWP::UserAgent](https://metacpan.org/pod/LWP::UserAgent)
