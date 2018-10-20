---
layout: post
title: "Composition and separation of concerns"
date: 2018-10-18 21:30:00 +0300
categories: design
tags: code quality, composition, decorator, functional composition, golang, go, design patterns, cross cutting concerns, clean code, unit testing
---

## Basics

Lets talk a bit about SOLID or rather SRP (Single responsibility principle). How do you think does this piece of code follow SRP principle?

```go
func (marshaller *binaryCommandHeaderDecoder) Decode(target interface{}) error {
    if marshaller.reader.ActualPosition() <= 0 {
        logger.Println("Error:", ErrReaderState)
        return ErrReaderState
    }

    if header, ok := target.(*Header); ok {

        header.Type = Type(marshaller.reader.ReadSingleByte())
        header.Tick = marshaller.reader.ReadSignedInt(32)
        header.PlayerSlot = marshaller.reader.ReadSingleByte()

        logger.Printf("Header has been decoded. Content %+v", header)

        if valid, err := header.Type.IsValid(); !valid {
            logger.Println("Error:", err)
            return err
        }
    } else {
        var err = fmt.Errorf(
            "incoming object has invalid type. Expected '*CommandHeader', actual %s",
            reflect.TypeOf(target))

        logger.Println("Error:", err)

        return err
    }

    return nil
}
```

The answer is no, it doesn't. There are two responsibilities for this piece of code: decoding some data and perform auditing (logging) for execution flow. In another words there are two reasons for modification: the first reason is changing the decode logic, and another can be a changes for content of log messages, for instance.

Is this a huge problem? In this particular case it is a not a big issue, but let's imagine that we have some kind of caching, authorization, load balancing, retry mechanism built on top of the main logic:

```go
func (c *Client) Do(r *http.Request) (res *http.Response, err error) {
    r.Header.Add("Authorization", c.Token)
    for i := 0; i <= c.Tolerance; i++ {
        r.URL.Host = c.Backends[atomic.AddUint64(&c.robin, 1)%unit64(len(c.Backends))]
        log.Printf("%s: %s %s", r.UserAgent, r.Method, r.URL)
        start := time.Now()
        res, err = http.DefaultClient.Do(r)
        c.latency.Observe(time.Since(start))
        c.requests.Add(1)
        if err != nil {
            time.Sleep(time.Duration(i) * c.Backoff)
            continue
        }
        break
    }
    return res, err
}
```

We just wanted to send a simple request, didn't we? :)

I see at least three cons of this approach:

- code duplication
- code tangling
- painful code testing

## Separation of concerns

Logging, authorization, load balancing, retries, caching are nothing more than [cross-cutting concerns](https://en.wikipedia.org/wiki/Cross-cutting_concern). And all this "concerns" need to be decoupled from main logic.

This is where composition (functional and object oriented) comes into place. Instead of placing all stuff into a single class or function we can decouple logic into a small composable pieces.

![Composition]({{ "/assets/composition/onion.png" | absolute_url }}){:style="display: block;margin: 0 auto;" }

Looks like an onion, right? :)

Here is how it can be implemented in object oriented manner:

```cs
    interface IVariantDao
    {
        Task<Variant> Get(Guid id);
    }

    internal class VariantDao : IVariantDao
    {
        private readonly IDocumentDbClient _client;

        public VariantDao(IDocumentDbClient client)
        {
            _client = client;
        }

        public Task<Variant> Get(Guid id)
        {
            return _client.GetAsync(id);
        }
    }

    internal class LoggingVariantDao : IVariantDao
    {
        private readonly ILogger _logger;
        private readonly IVariantDao _origin;

        public LoggingVariantDao(
            ILogger logger, IVariantDao origin)
        {
            _logger = logger;
            _origin = origin;
        }

        public async Task<Variant> Get(Guid id)
        {
            Variant result = await _origin.Get(id);

            _logger.LogDebug("The following variant has been retrieved: {variant}", result);

            return result;
        }
    }

    internal class CachingVariantDao : IVariantDao
    {
        private readonly IMemoryCache _cache;
        private readonly IVariantDao _origin;

        public CachingVariantDao(
            IMemoryCache cache, IVariantDao origin)
        {
            _cache = cache;
            _origin = origin;
        }

        public async Task<Variant> Get(Guid id)
        {
            if (!_cache.TryGetValue(id, out Variant entry))
            {
                entry = await _origin.Get(id);
            }

            return entry;
        }
    }


    static void Main(string[] args)
    {
        IVariantDao dao = new CachingVariantDao(
            new IMemoryCache(...),
            new LoggingVariantDao(
                new Logger(...),
                new VariantDao(new CosmosDbClient(...))));
    }
```

And the next example shows how it can be implemented in a functional manner using golang and functional types:

```go
type Decoder interface {
    Decode(target interface{}) error
}

// function type which implements `Decoder` interface
type DecodeFunc func(target interface{}) error
func (f DecodeFunc) Decode(target interface{}) error {
    return f(target)
}

// represents decorator for `Decoder` interface
type Decorator func(Decoder) Decoder

// the function which returns `Decorator` implementation
func Logging(logger *log.Logger) Decorator {
    return func(origin Decoder) Decoder {
        return DecodeFunc(func(target interface{}) error {
            err := origin.Decode(target)
            if err != nil {
                logger.Println("Error:", err)
            } else {
                logger.Printf("%s -> Result: %s\n", reflect.TypeOf(origin), string(target))
            }
            return err
        })
    }
}

// function which composes all decorators using variadic arguments
func Decorate(marshaller Decoder, decorators ...Decorator) Decoder {
    var decorated = marshaller
    for _, decorator := range decorators {
        decorated = decorator(decorated)
    }
    return decorated
}

// usage

var decoder = Decorate(
    command.NewDecoder(reader),
    Logging(
        log.New(
            io.MultiWriter(os.Stderr, logOut), prefix, log.LstdFlags,
        ),
    )
)

...

var decoder = Decorate(
    command.NewDecoder(reader),
    Logging(...),
    Caching(...),
    Authorization(...)
)
```

## Summing up

Composition is a more natural way for object oriented and functional paradigms, so you can easily compose things without original code changes.

Props:

- code simplicity
- simple testing
- following SOLID (SRP, OCP)
- code reusability

Limitations:

- composable parts have to satisfy contract (interface)

Enjoy :)