---
layout: post
title: Introducing glue
date: 2015-05-18
summary: My attempt on streamlining data sharing in go middlewares.
categories: programming golang
---

Now I could try to explain the basic gist of this package in a couple of words and be done with it, but where's the fun in that?

## Scenario

Let's say we're developing a web application in Go. We use the common `middleware(http.handler) http.handler` signature to create chains of middlewares.
This allows us to have an authentication-layer, which is placed in front of all private routes.

For the sake of simplicity, our authentication method is a static token in the HTTP-Header.

We implement this layer as an aforementioned middleware. Its tasks consist of:

- Checking if the token is associated to a user in our database.
- And if so, save the user-object in a request-specific value.

These two simple tasks highlight two problems, which arise when developing web applications with Go.

### Accessing global objects

Our database is a global singleton object. Before you say anything bad about singletons (and rightfully so!): I don't mean the infamous pattern, its just that there is one single instance of it at all times.

So our middleware has our database as a dependency. And what is the first solution that come to mind?

A global object.

This is the point where the argument starts whether globals are cool or a sign of something inherently wrong with the codebase. But we choose not to use globals, since we want to make our code testable.

Alright, we need an alternative. [This great article](http://www.jerf.org/iri/post/2929) suggests using an environment which we can pass to all dependent functions. Nice, that sounds a lot like dependency injection and we love buzzwords.

```go
type Environment struct {
  DB *SomeDBType
}
```

Now we simply pass our environment as an argument to our middleware:

```go
env := Environment{DB: NewDB()}
http.Handle("/", authenticationMiddleware(&env, someHandler))
```

Well, it works and we're able to replace all dependencies with mock-objects in our unit tests. But do you see what we just did?

```go
func authenticationMiddleware(env *Environment, next http.Handler) http.Handler { ...
```

We broke our `middleware(http.handler) http.handler` signature. Its not the end of the world, but now our middleware isn't compatible with most of the popular middleware chaining packages, such as [Alice](https://github.com/justinas/alice).

So our middleware now does look something like this:

```go
func authenticationMiddleware(env *Environment, next http.Handler) http.Handler {
  return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    token := extractToken(r) //extracts the token from the request
    if user := env.DB.FindUser(token); user != nil {
      next.ServeHTTP(w, r) //+ user?
    } else {
      http.Error(w, "Unauthorized", 401)
    }
  })
}
```

But how do we pass the user to our next handler?

### Request-specific values

There are a lot of packages in the wild which aim to make this possible. [gorilla/context](http://www.gorillatoolkit.org/pkg/context) is probably the best known.

Its basically a global map which uses the request as the key and another map as its value. The downside in this approach lies within its memory management. The global map has no implicit way of knowing when a request is considered completed and thus cannot free the associated map.

To prevent memory leaks, you have to call `context.Clear()` at the end of every request. Since doing that quickly becomes tedious, the package provides a `ClearHandler(http.Handler) http.Handler` middleware. And if you're using the great [gorilla/mux](http://www.gorillatoolkit.org/pkg/mux), the context is even cleared automatically.

Thats good enough for our middleware:

```go
func authenticationMiddleware(env *Environment, next http.Handler) http.Handler {
  return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    if user := env.DB.FindUser(token); user != nil {
      context.Set(r, "user", user)
      next.ServeHTTP(w, r)
    } else {
      http.Error(w, "Unauthorized", 401)
    }
  })
}
```

So what role does glue play in all of this?

## What is glue?

[Glue](https://github.com/fdborn/glue) is, apart from the sticky stuff, my attempt to solve both these problems (and most of their shortcomings).

Its embarrassingly simple and has below 30 SLOC, but hear me out.

First of all, you create your environment. We already did that:

```go
type Environment struct {
  DB *SomeDBType
}
```

Now we create a new glue object with a pointer to our environment:

```go
env := Environment{DB: NewDB()}
g := glue.New(&env)
```

Lastly, we use the `Glue.Apply()` method as root of all our routes:

```go
http.Handle("/", g.Apply(authenticationMiddleware(someHandler)))
```

Since we restored the common middleware signature to `middleware(http.Handler) http.Handler`, we can use [Alice](https://github.com/justinas/alice) and friends:

```go
chain := alice.New(g.Apply, authenticationMiddleware)
http.Handle("/", chain.Then(someHandler))
```

And how does our middleware look?

```go
func authenticationMiddleware(next http.Handler) http.Handler {
  return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    context := glue.Extract(w)        //new
    env := context.Env.(*Environment) //new

    token := extractToken(r) //extracts the token from the request
    if user := env.DB.FindUser(token); user != nil {
      context.Vars["user"] = user     //changed
      next.ServeHTTP(w, r)
    } else {
      http.Error(w, "Unauthorized", 401)
    }
  })
}
```

I admit, its more code than before and the type assertion is ugly, but

- we injected the environment without changing the signature and
- saved request-specific values without using a global map.

### How does it work?

If you found yourself wondering why, in a `http.Handler`, the `http.ResponseWriter` is passed by value and the `http.Request` by pointer, you will be pleased to hear that both are passed as pointers. Its just that `http.ResponseWriter` is an interface and `http.Request` a concrete type.

That means, that every object which satisfies said interface can be used interchangeably as the first parameter in `ServeHTTP`. And herein lies all its magic.

```go
type Context struct {
	http.ResponseWriter

	Vars map[string]interface{}
	Env  interface{}
}
```

On every call of `glue.Apply()`, a new Context is created, which embeds the previous `http.ResponseWriter`. This object holds a `map[string]interface{}` and a pointer to the previously passed environment.

```go
func (g *Glue) Apply(handler http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		context := Context{
			ResponseWriter: w,
			Vars:           make(map[string]interface{}),
			Env:            g.env,
		}
		handler.ServeHTTP(&context, r)
	})
}
```

The extraction inside our middleware is a simple type assertion:

```go
func Extract(w http.ResponseWriter) *Context {
	if c, ok := w.(*Context); ok {
		return c
	}
	return nil
}
```

*I know, Extract is a bad name for upcasting the interface but whatever.*

## Final words

This idea is nothing new and originally it wasn't meant to end up as a package. Since its so small, you should probably use it as inspiration and build your own method of embedding the `http.ResponseWriter` interface.

Actually, its even better if you roll your own Context-like object. You evade the ugly type assertion by using a concrete type.

```go
type Context struct {
	http.ResponseWriter

	Vars map[string]interface{}
	DB   *SomeDBType
}
```

Yeah, you should do that.
