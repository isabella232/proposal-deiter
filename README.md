# Double-Ended Iterator and Destructuring

This proposal has not yet been presented to TC39 plenary meetings.

## Motivation

Python and Ruby support `(first, *rest, last) = [1, 2, 3, 4]`, CoffeeScript supports `[first, rest..., last] = [1, 2, 3, 4]`, and Rust supports `[first, rest @ .., last] = [1, 2, 3, 4]`, all resulting in `first` be `1`, `last` be `4`, and `rest` be `[2, 3]`. But [surprisingly](https://stackoverflow.com/questions/33064377/destructuring-to-get-the-last-element-of-an-array-in-es6) `[first, ...rest, last] = [1, 2, 3, 4]` doesn't work in JavaScript.

And in some cases we really want to get the items from the end, for example getting `matchIndex` from [String.prototype.replace when using a function](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String/replace#Specifying_a_function_as_a_parameter):

```js
string.replace(pattern, (fullMatch, ...submatches, matchIndex, fullString) => {
  // `matchIndex` is always the second to last param (the full string is the last param).
  // There may be many submatch params, depending on the pattern.
})
```

A simple solution is making `let [first, ...rest, last] = iterable` to work as

```js
let [first, ...rest] = iterable
let last = rest.pop()
```

The concern is it requires saving all items in a `rest` array, although you may only need `last`. A possible mitigation is supporting `[..., last] = iterable` which saves the memory of `rest`, but you still need to consume the entire iterator. In the cases where `iterable` is a large array or something like `Number.range(1, 100000)`, it's very inefficient. And in case like `let [first, ..., last] = repeat(10)` (suppose `repeat` is a generator returns infinite sequence of a same value), theoretically both `first` and `last` could be `10`.

## Possible solution

Instead of the simple solution, we could introduce the double-ended iterator (like Rust std::iter::DoubleEndedIterator). A double-ended iterator could be consumed from both ends.

```js
let a = [1, 2, 3, 4, 5, 6]
let deiter = a.values() // suppose values() would be upgraded to return a double-ended iterator
deiter.next() // {value: 1}
deiter.next() // {value: 2}
deiter.next('back') // {value: 6}
deiter.next() // {value: 3}
deiter.next('back') // {value: 5}
deiter.next('back') // {value: 4}
deiter.next('back') // {done: true}
deiter.next() // {done: true}
```

With double-ended iterators, `let [a, b, ..., c, d] = iterable` would roughly work as

```js
let iter = iterable[Symbol.deIterator]()
let a = iter.next().value
let b = iter.next().value
let d = iter.next('back').value
let c = iter.next('back').value
iter.return()
```

## Generator

To implement double-ended iterator in userland, we could use a generator with the [`function.sent` feature](https://github.com/tc39/proposal-function.sent).

```js
Array.prototype.values = function *values() {
  // only for demo, for real upgrading of Array.prototype.values 
  // to double-ended iterator, it need to deal with the edge cases 
  // of mutating the array while iterating
  for (let start = 0, end = this.length; start < end;) {
    if (function.sent === 'back') yield this[--end]
    else yield this[start++]
  }
}
```

## Iterator helpers and reverse iterator

Double-ended iterator could have some extra [iterator helpers](https://github.com/tc39/proposal-iterator-helpers) like `reversed` and `reduceRight`.

```js
DoubleEndedIterator.prototype.reversed = function *reversed() {
  for (;;) {
    let result
    if (function.sent === 'back') result = this.next()
    else result = this.next('back')
    if (result.done) return result.value
    else yield result.value
  }
}
```

We could also easily have a default implementation for [reverse iterator](https://github.com/tc39/proposal-reverseIterator) if the object already supports double-ended iterator.

```js
Object.assign(X.prototype, {
  *[Symbol.reverseIterator]() {
    const iter = this[Symbol.deIterator]()
    for (;;) {
      let result
      if (function.sent === 'back') result = iter.next()
      else result = iter.next('back')
      if (result.done) return result.value
      else yield result.value
    }
  }
)
```

## Prior art
- Python [iterable unpacking](https://www.python.org/dev/peps/pep-3132/)
- Ruby [array decomposition](https://docs.ruby-lang.org/en/2.7.0/doc/syntax/assignment_rdoc.html#label-Array+Decomposition)
- CoffeeScript [destructuring assignment with splats](https://coffeescript.org/#destructuring)
- Rust [subslice pattern](https://rust-lang.github.io/rfcs/2359-subslice-pattern-syntax.html)
- Rust [std::iter::DoubleEndedIterator](https://doc.rust-lang.org/std/iter/trait.DoubleEndedIterator.html)
- Rust [Macro improved_slice_patterns::destructure_iter](https://docs.rs/improved_slice_patterns/2.0.1/improved_slice_patterns/macro.destructure_iter.html)

## Previous discussions
- https://github.com/tc39/proposal-array-last/issues/31
- https://github.com/tc39/proposal-reverseIterator/issues/1
- https://es.discourse.group/t/bidirectional-iterators/339

## Old discussions
- https://esdiscuss.org/topic/early-spread-operator
- https://esdiscuss.org/topic/parameter-lists-as-arguments-destructuring-sugar#content-3
- https://mail.mozilla.org/pipermail/es-discuss/2012-June/023353.html
- http://web.archive.org/web/20141214094119/https://bugs.ecmascript.org/show_bug.cgi?id=2034
- https://esdiscuss.org/topic/rest-parameter-anywhere
- https://esdiscuss.org/topic/rest-parameters
- https://esdiscuss.org/topic/strawman-complete-array-and-object-destructuring
- https://esdiscuss.org/topic/an-update-on-rest-operator
