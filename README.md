# unifn

A Clojure library designed to provide "universal"
functions interface, which is composable by "meta"-data
into pipelines and workflows. 

Inspired by pedestal, prismatic libraries & onyx.


## Motivation

Some pieces of code could be expressed 
as a pipeline, for example processing a HTTP request and producing HTTP response: 


```
 request -> [fn] -event-> [fn] -event-> [fn] -> response

```

Ring uses functions decoration to build request processing stack,
but it has some drawbacks

* async interfaces
* introspectoin
* stacktraces

Pedestal is based on interceptor's stack, 
which is dynamic and executed by extrnal executor. 
This is better and gives us control point 
between interceptors. Pedestal also introduce 
*context map* as context for all interceptors.


Onyx gives us an idea separating functions from workflow 
definitions by data dsl. 

Let's combine all together:

We build processing stack from functions with uniform
interface - functions of one argument (context map), which 
returins hash-map, interpreted as patch to original context map.

```clj
(defn pipeline-element 
   [ctx-map]
   {:some-context value})
```

Such functions could be composed into pipeline:

```clj
(def pipeline
   [pileline-fn-1
    pileline-fn-2
    pileline-fn-3])
```

And external executor could reduce this pipeline aka:

```clj

(reduce 
  (fn [ctx-map item]
    (let [result (item ctx-map)]
       (deep-merge ctx-map result))) 
  {:request http-request-map} 
  pipeline)

```

We could introduce configuration data into pipeline definition:

```clj
(def pipeline
  [[fn-1 {:some "config"}]
   [fn-2 {:another "config"}]
   ....])

;; and interpretation

(reduce 
  (fn [ctx-map [f config]]
    (let [result (f (merge ctx-map config))] ;; merge config into ctx-map 
       (deep-merge ctx-map result))) 
  {:request http-request-map} 
  pipeline)

```

By merging config map into only argument, 
we allows upstream pipeline functions to inject
config into context map for downstream funcftions,
making configuration more dynamic.

For example some dynamic routes could 
be fetched from database and injected into 
context map before dispatching function.

```clj
(def pipeline
   [[routes-from-db]
    [routing]])
```

Sometimes you want to stop
normal pipeline processing (for example security check returns unauthorized
and we want to respond with 403 status).

There are different ways to implement it:

* return some magic key {:status :stop}, which will be interpreted by 
  pipeline manager as alternative path
* (pedestal way) allow function to modify
  rest of pipline

Let's consider first, more static approach
and allow pipline functions subscribe to different
context map states.


```clj
(def pipline
  [[verify-jwt]
   [security-check {cfg}]
   [handler]
   [format-response {:status [:access-denied :succes]}]])
   
(fn security-check [{req :request}]
  (when-not (authorized? req ...)
    {:status :access-denied
     :response {:message "..."}}))

(fn format-response [{req :request resp :response}]
  ...
  {:response {:body  (json-or-other-format (:body resp))}})
 
;; pipeline interpretation
  
(reduce 
  (fn [ctx-map [f config]]
    (if (or (nil? (:status ctx-map)) 
            (subscribed? config (:status ctx-map)))
       (f ctx-map)
       ctx-map ;; skip))
  {:request http-request-map} 
  pipeline)
  
   
````

## License

Copyright © 2017 niquola

Distributed under the Eclipse Public License either version 1.0 or (at
your option) any later version.
