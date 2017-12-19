# httpclient - functional options experiment

I've written quite a few golang api clients and libraries. I always end up finding myself wrapping the http client work up in somewhat similar ways.
In working on rewriting my rundeck client library, I ran into an interesting issue.

The rundeck api can use xml OR json. The original library ONLY supported xml but the json stuff is a bit easier to work with for contributors.
I started rewriting the http client bits to handle remove the `napping` library it was using and make it a bit easier to swap between xml and json for requests without needing to swap all the structs and unit tests around (it's a lot of work).

The resulting http client wrapper was ugly as shit - crazy function signatures and such.

## Functional Options

[Brian Akins](https://github.com/bakins) is a big fan of the functional options approach and I figured this would be a chance for me to REALLY grok it by doing it.
You can read a bit about the concept [here](https://dave.cheney.net/2014/10/17/functional-options-for-friendly-apis)

The gist is you use have a function signature that has a variadic argument (removing the need for a bunch of `nil` placeholders in the function signature).
The variadic argument is a function that returns a function operating on an instance of a newly created 'whatever'.

At least that's the ugly definition. Code is a bit easier to grok in this case I think.

The general idea is I went from a potential function signature like so:

```go
func foo(x string, y string, z *something, t int) (bar, error)
```

where in some cases anything after the last option could be `nil` or `""` thus having potential invocations like:

``` go
foo("bar", "ugg", nil, 10) // I didn't need the third argument but I did the fourth
foo("baz", "qux", &snarf{}, 0) // oh I needed all but the last
foo("fml", "", nil, 0) // I only needed the first argument
```

to something like:

```go
func foo(x string, opts ...Option) (bar, error)
```

with invocations more sane like:

```go
foo("bar", UggOpt("ugg"), IntOpt(10) // replaces foo("bar", "ugg", nil, 10)
foo("baz", QuxOpt("qux"), SnarfOpt(&snarf{})) // replaces foo("baz", "qux", &snarf{}, 0)
foo("happy life") // replaces foo("fml", "", nil, 0)
```

In practice, I've found this much easier to work with.

## httpclient examples

So how did I put this into practice? That's what this code is.
Take the following usage from one of the unit tests:

```go
    qp := make(map[string]string)
    qp["foo"] = "bar"
    response, err := Get("https://httpbin.org/get", QueryParams(qp), JSON())
```

In this case I want to make a `get` request to the provided url. Note that I could have stopped there i.e. `Get("https://httpbin.org/get")` but I wanted to customize it a bit:

- Pass in some custom query params
- Set the `Accept` and `Content-Type` headers to `application/json`

So I just turned on those options.

The function signature for `Get` looks like this:

```go
func Get(url string, opts ...RequestOption) (*Response, error) {
    opts = append(opts, get())
    opts = append(opts, setURL(url))
    return doRequest(opts...)
}
```

I can pass in any number of options. I want and don't need to do any special handling before passing off to `doRequest` after setting the defaults for a get request.

And inside `doRequest` there's no super magic either other than setting some sane defaults.

So what does one of those "options" look like?

```go
type RequestOption func(*Request) error // RequestOption type for reference

func JSON() RequestOption {
    return func(r *Request) error {
        r.accept = ContentTypeJSON
        r.contentType = ContentTypeJSON
        return nil
    }
}
```

The neat thing is that this also allows the user to create their OWN options if they want. I do this in the unit tests:

```go
func testCustomOption() RequestOption {
    return func(r *Request) error {
        return errors.New("i blew up")
    }
}
```

as a way to test the error handling.

The part I had the hardest time groking before doing my own version was where that instance of `Request` came from in those functions since
when you USE the option, it **appeared** you were CALLING the function inside the option but obviously that's not the case.

I couldn't get it around my head at first that you were returning a function (which has yet to be invoked). This is why there's no mutability outside the return. It's all pending mutability (which sounds scary but really isn't in this case)

You can see this in the constructor:

```go
func newHTTPRequest(opts ...RequestOption) (*Request, *http.Request, error) {
    // create a new instance of a Request
    r := &Request{}
    if r.httpClient == nil {
        r.setHTTPClient(&http.Client{})
    }
    codes := make([]int, 0)
    r.allowedStatusCodes = codes
    // iterate the opts
    for _, opt := range opts {
        r.Lock()
        // oh look THAT'S where the instance is passed in
        // mutability incoming
        if err := opt(r); err != nil {
            r.Unlock()
            return nil, nil, err
        }
        r.Unlock()
    }
    req, err := r.httpRequest()
    return r, req, err
}
```

So I hope this has helped some folks. It helped me. I don't know that this library REALLY makes sense but I *DO* find the API to be kind of pleasant to work with even if it's just an internal one.