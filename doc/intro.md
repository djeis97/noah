# noah
> A Clojure library atop Kafka Streams

> * This documentation assumes that the reader is familiar with the Kafka Streams Java API.

> * This is alpha software which has not seen a lot of real-world use, and no warranty is provided.

### Introduction

This is an experimental library which aims to make Kafka Streams more Clojure-friendly by handling mundane details for you. It also aims not to obstruct the underlying APIs at all (it should mix with Java code, for example), and to introduce Clojure abstractions where it pays to do so.

#### High-level API
A very simple, no-frills Clojure interface to the higher-level API is provided in `noah.core`. It looks like this:

```clojure
(require '[noah.core :as n])

(defn topology []
  (let [b (n/streams-builder)
        text (-> b (n/stream "text"))]
    (-> text
        (n/flat-map-values #(str/split % #"\s+"))
        (n/group-by #(-> %2))
        n/count
        n/to-stream
        (n/to "word-counts" {::n/value-serde :long}))
    (n/build b)))
```

It's intended to be free of surprises and mostly just makes things look better. The functions in `noah.core` wrap every public method of these classes (including static methods, except constructors):

`[ KStream KGroupedStream KTable KGroupedTable StreamsBuilder ]`

In order to minimize surprises, almost all of these functions are generated by a macro using Java reflection from the classes in the Java library, and are named by kebab-casing the Java method. They come with docstrings which list the signatures.

These generated wrappers convert certain types as they cross the boundary with Java land, leveraging the reflected information about the Kafka Streams method signatures. That means it can be smart about turning your Clojure function into an instance of the proper one of the Java classes which wrap function application in a new and exciting name. For these calls:

```clojure
(n/flat-map-values stream (fn [v]   ... ))
(n/flat-map-values stream (fn [k v] ... ))
(n/flat-map        stream (fn [k v] ... ))
```

the first Clojure function will be converted to a `ValueMapper`, the second to a `ValueMapperWithKey`, and the third to a `KeyValueMapper`. You can probably forget all about this and just use it. If you pass it an invalid arity, it will yell at you immediately. Otherwise it will probably do what you intended.

The conversions of maps (such as the `{::n/value-serde :long}` above) is also a context-sensitive. You can use them anywhere you need a `Consumed`, `Produced`, `Serialized`, or `Materialized` instance (everything is exposed, though some of it lacks sugar).

#### The Processor API
A single macro is provided to allow concise definitions of `Transformer`s. It has supports punctuations and `StateStore`s with a minimum of fuss. You can also define `:init` if you need the `ProcessorContext`.

```clojure
(require '[noah.core :as n])
(require '[noah.transformer :refer [deftransformer]])

(deftransformer expiring-transformer
  "Expires old records"
  [mystore]
  :schedule (java.time.Duration/ofMinutes 20) ::n/stream-time
  (fn expiration-scan [ts]
    (let [cutoff (- ts (* 1000 60 20))]
      (doseq [r (iterator-seq (.all mystore))]
        (when (and (.value r) (< (.value r) cutoff))
          (.forward context (.key r) nil)))))
  ;;:init  (fn my-init [ctx] )
  ;;:close (fn my-close [ctx stores] )
  [k v]
  (if (nil? v)
    (.delete mystore k)
    (do (.put mystore k (.timestamp context))
        (.forward context k v))))
```

#### Transducers

You can transduce a KStream, using `noah.core/transduce`, and that seems like a good plan:

```clojure
(-> nums
    (n/transduce (comp (filter even?)
                       (interpose 42)))
    (n/to "transduced-nums"))
```

And this does what it should do... right up until the next rebalance, and then it will do  **unspeakable things!** :broken_heart:

It will do really frightening and awful things, on accounta `interpose` is a stateful transducer, after a rebalance its state would be reinitialized as `false`, which is incorrect. Potential consequences:

1. the business could fail
1. two or more numbers could be produced without a `42` in between them! :fire: 
1. people could die

Never mind! This won't do.

> IFF you're cool with states going in the bit bucket, you can skip this section and use your favorite stateful transducers with `noah`.

Stateful transducers can be reimplemented (trivially), for use with `noah`. This seems unavoidable because transducers are not "fully decoupled" from their own state, which they construct themselves in a lexical scope. However, if we're willing to live with a whole heap of fairly painless and straightforward code duplication, then we can have fault-tolerant stateful transduction of a KStream, and it might be worth it. Maybe we could do something dangerous and crazy like instrumenting the bytecode where they use `volatile!`. I'm not sure. Anyway, rewriting a transducer is most copy & paste which feels kinda bad but is not totally crazy:

```clojure
(defn interpose
  "Returns a noah stateful transducer which separates elements by sep."
  [sep]
  (fn [rf]
    (let [started (*state* false)]
      ([] (rf))
      ([result] (rf result))
      ([result input]
       (if @started
         (let [sepr (rf result sep)]
           (if (reduced? sepr)
             sepr
             (rf sepr input)))
         (do
           (vreset! started true)
           (rf result input)))))))
```

In this case the code is *identical* to the transducer arity of `clojure.core/interpose`, except that `volatile!` has been replaced with `noah.transduce/*state*`. This is `noah.transduce/interpose`. Note it still uses `vreset!` because it is still using a `volatile!`, as many transducers do. It's just not in charge of constructing it anymore.

Instead, here's what `noah` will do:

* For each input record key, check the `StateStore`
* Provide the transducer with either the stored value for that key, or its initial value
* Run the transduced step function on the input record, potentially forwarding records, and updating the `volatile!`
* `.put()` the value of the `volatile!` into the `StateStore`

If we then enable exactly-once, we should *always* see a `42` where there ought to be a `42`, and never one where it doesn't belong. Everybody is safe and happy. :heart: :moneybag:

#### Stateful transducers compose

Their changelog-backed states also compose.

```clojure
(require '[noah.transduce :refer [partition-by partition-all interpose]])
(-> nums
    (n/transduce (comp (map inc)
                       (partition-by #(or (zero? (mod % 11))
                                          (zero? (mod % 7))))
                       (filter #(not= 1 (count %)))
                       (map #(reduce + %))
                       (partition-all 4)
                       (interpose :avocado)))
    (n/to "critical-analysis"))
```

`noah` puts vectors of values into a single `StateStore` for the states of the whole stack.
The values in this case will be 4-element vectors.
Two states for `partition-by` and one for each of the others.
Since `noah` is stuffing them all into a vector, each update to the states of the transducer stack makes one entry in the changelog topic, and there will be one new entry in the changelog topic for each new input record regardless of how many output records it produces.

Some transducers out there use mutable Java collections. That isn't going to work, unless you provide serdes for them, but they can be reimplemented using Clojure data.

I've taken the liberty of adding all the ones from `clojure.core`, even the ones that are probably ill-advised, like `distinct`, and those that don't seem to make a lick of sense, like `take`. There is a madness to my method though, stay tuned for windowed transducers...