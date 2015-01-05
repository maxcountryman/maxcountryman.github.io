---
layout: post
title: "Simple Routing With Ring"
date: 2015-01-04 08:27:25
---

## Handling Requests
This is a short post about routing requests in a Clojure Ring webserver. While
there are many well-known approaches, in particular [Compojure][1], you may
find these are not appropriate for some uses cases. For instance, when writing
a RESTy API server, it may be desirable to respond with a 405 Method Not
Allowed as opposed to a 404 Not Found, when the request method does not match
the route. Furthermore you may want a route handler to be capable of responding
to multiple, but not all, request methods. Say you have a `/user` endpoint
which can both be accessed with GET and POST--this is difficult to do with the
Compojure DSL. We will talk about some ways of achieving these goals without a
DSL.

When I started with Clojure and Ring I was coming from a background in Python,
using primarily [Flask][2], a microframework. With Flask, the default response
for a request to a route of the wrong method is a 405--this is mostly cosmetic
however. More important is the ease with which a route can handle multiple,
specific request methods:

```python
import flask

@flask.route('/user/<uid>', methods=['GET', 'POST'])
def user(uid):
    if flask.request.method == 'GET':
        return db.get_user_by_id(uid)
    return db.create_user(flask.request, uid)
```

Instead of using Compojure, my simple API servers used the underlying route
matcher utility, [clout][3]. This was partly because my application needed one
or two simple routes and so the additional power and complexity of Compojure
was unnecessary. By using clout directly, the surface area of the route
handling is reduced and explicit:

```clojure
(require '[clout.core :refer [route-matches]])

(defn wrap-user-routes [handler]
  (fn [{:keys [request-method] :as request}]
    (condp route-matches request
      "/user/:uid" :>> (fn [uid]
                         (condp = request-method
                           :get  (db/get-user-by-id uid)
                           :post (db/create-user request uid)
                           {:status 405
                            :body   "Method Not Allowed"}))
      (handler request))))
```

The Python and Clojure examples are more or less equivalent. We will have to
pretend that the database methods make sense and that the proper data
validation has been done. However for the sake of illustration, those pieces
are omitted.

This Clojure handler works relatively well, to a point. Certainly for smaller
applications, especially those that do not need to switch on request method,
using clout directly can be quite nice. Nevertheless, there is an improvement
that can be made here. Using Clojure's multimethods we can isolate the logic
responsible for handling each request method in a fairly natural way:

```clojure
(defmulti user-response :request-method)

(defmethod user-response :get [request uid]
  (db/get-user-by-id uid))

(defmethod user-response :post [request uid]
  (db/create-user request uid))

(defmethod user-response :default [_ _]
  {:status 405 :body "Method Not Allowed"})

(defn wrap-user-routes [handler]
  (fn [{:keys [request-method] :as request}]
    (condp route-matches request
      "/user/:uid" :>> #(user-response request %)
      (handler request)))
```

### Data Validation
Personally, this is starting to look fairly robust for my particular use-cases:
this covers declarative routing based on request method (and actually any
arbitrary piece of the request map) and there is no need to dip into the depths
of the Compojure DSL. That said, there are still important facets of this
approach that deserve to be addressed.

You should be asking about something we skipped over earlier: How to validate
incoming data? We need to ensure that our data the route receives is in the
correct shape before we do anything with it. This is important so that when a
caller sends us a malformed or malicious request we can respond with an error
and without our server melting down.

On my first attempt at using clout directly, the solutions I came up with felt
ad hoc and fragile. They also quickly became difficult to read and write. I
asked about this on the Clojure mailing list and someone (apologies, I've lost
the mail now so forgive me for not giving credit where it's due) responded with
an excellent idea: Using a combination of `some-fn` and simple boolean logic,
we can pass a request map through a series of validators. These validators
must return `nil` so long as the request is valid. When the request is invalid,
they return an errored response map. Here's what that might look like:

```clojure
(defn json?
  [{:keys [content-type]}]
  (boolean
    (when content-type
      (re-find #"^application/(.+\+)?json" content-type))))

(defn ensure-json
  [request]
  (when-not (json? request)
    {:status 400 :body "Non-JSON Content-Type"}))

(defn ensure-body
  [{:keys [body]}]
  (or (when-not (map? body)
        {:status 400 :body "Malformed request body"})
      (when-not (:password body)
        {:status 400 :body "Missing key: password"})))
```

We can use these validators with our existing routing logic. For instance, we
should update the user POST handler like so:

```clojure
(defmethod user-response :post [request uid]
  (let [some-errors (some-fn ensure-json ensure-body)]
    (or (some-errors request)
        (db.create-user request uid))))
```

When a request is received, it's first dispatched by the route clout matches
it to. We then use a multimethod to filter the request by request method. At
this point, our route handler can validate the request map. As we did above,
the request map is passed to `some-errors` which is the composition of our
validators--this is kind of a special composite because any non-nil value
terminates the composition and immediately returns. This enables us to return
an error response or return a normal response.

Note that in a real application, we would need to add stricter validation. In
the interest of keeping examples short, we are using simplified examples.

### Putting It All Together
Finally we have omitted how this all interacts with the construction of a Ring
server. For example, here's how we can use our user routes:

```clojure
(require '[ring.adapter.jetty :refer [run-jetty]])

(def handler
  (-> (constantly {:status 404 :body "Not Found"})
      wrap-user-routes))

(run-jetty handler {:host "localhost" :port 3000})
```

Because routes are just middleware, we could attach them wherever we like--
which is an incredible boon for reusability! It's important to point out that
a route should have some kind of prefix, if the intent is to reuse it in other
applications--this prevents routes from overwriting one another.

For instance, when writing a server component for my [flake ID generation
library][4], I decided to write it as [a middleware][5]. That means that
applications could require the middleware and add it to their Ring handler.
Whether or not this has any practical value is highly dependent on the kind
of application you are building.

So far I've found the following benefits to this approach:

1. Small reusable components,
2. isolation and grouping of related logic,
3. simple, declarative style which is easy to reason about--look Ma, no macros!

In particular, the fact that routes in Compojure are constructed with a macro
means that it can be tricky to do things like dynamically bind a var over a
route. This bit me while writing a small API server which used `binding` to
abstract away some runtime logic--because the routes were evaluated at
macro-expansion time, the binding was never percolated correctly.

Another interesting aspect of using clout directly for routing is that a web
server can grow from a very small set of routes which don't do much to
a much more robust and comprehensive scale, progressively.

With all that in mind, you may still prefer Compojure and some of the other
nice tools it provides, but hopefully this has shown that it isn't entirely
necessary for certain contexts.

[1]: https://github.com/weavejester/compojure "A Concise Routing Library for Ring/Clojure"
[2]: http://flask.pocoo.org "Flask: Web Development One Drop at a Time"
[3]: https://github.com/weavejester/clout "HTTP Route-Matching Library for Clojure"
[4]: https://github.com/maxcountryman/flake "Decentralized, K-Ordered Unique IDs in Clojure"
[5]: https://github.com/maxcountryman/blizzard "HTTP Unique ID Generation Service"
