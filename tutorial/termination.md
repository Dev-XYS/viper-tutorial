# Termination (Conditional Decreases Clauses)

Viper provides the possibility to proof termination of recursive functions and methods as well as termination of while loops. A user has therefore to define a termination measure which is then by Viper checked to decrease in each iteration (with respect to a well-founded order).
If no termination measure is defined Viper, by default, wont check for termination.

Termination measures can be defined in *decreases clauses*.

```silver
decreases [exp]
```
Here, `[exp]` is a tuple of comma-separated expressions which is the termination measure. Henceforth, we refer to such a decreases clause as *decreases tuple*.

For a function or a method the decreases tuple can be defined at the position of the preconditions, for a loop at the position of the invariants.

A simple example is the standard factorial function, which is terminating because the parameter n decreases with respect to the usual well-founded order over positive numbers.

```silver {.runnable}
import <decreases/int.vpr>

function factorial(n:Int) : Int
  requires 0 <= n
  decreases n
{ n == 0 ? 1 : n * factorial(n-1) }
```

Viper verifies successfully that the termination measure `n` decreases for each recursive invocation of `factorial` and always is positive. The well-founded order over positive numbers is defined in the file `decreases/int.vpr`, which is provided by Viper and can be imported with `import <decreases/int.vpr>`.
Viper further provides the following definitions of well-founded orders for all the build-in types (`<_` represents the fictive well-founded decreasing operator):

| Build-In Type | Provided Well-Founded Order |
| ---- | ---- |
|`Ref`<br>(`ref.vpr`)| `r1 <_ r2 <==> r1 == null && r2 != null`
|`Bool`<br>(`bool.vpr`)| `b1 <_ b2 <==> b1 == false && b2 == true`
|`Int`<br>(`int.vpr`)| `i1 <_ i2 <==> i1 < i2 && 0 <= i2`
|`Rational`<br>(`rational.vpr`):| `r1 <_ r2 <==> r1 <= r2 - 1/1 && 0/1 <= r2`
|`Perm`<br>(`perm.vpr`)| `p1 <_ p2 <==> p1 <= p2 - write && none <= p2`
|`Seq[T]`<br>(`seq.vpr`)| `s1 <_ s2 <==> |s1| < |s2|`
|`Set[T]`<br>(`set.vpr`)| `s1 <_ s2 <==> |s1| < |s2|`
|`Multiset[T]`<br>(`multiset.vpr`)| `s1 <_ s2 <==> |s1| < |s2|`

Another well-founded order can be defined on predicate instances. Due to the least fixpoint interpretation of predicates, any predicate instance has a finite depth of predicates (the number of nested folded predicate instances). This implies that a predicate instance `p1`, which is nested inside a predicate instance `p2`, has a smaller but still a non-negative depth of predicates than `p2`. Viper provides therefore also the following definition of a well-founded order over predicate instances:

| Type | Provided Well-Founded Order |
| ---- | ---- |
|`PredicateInstance`<br>(`predicate_instance.vpr`)| `p1 <_ p2 <==> nested(p1, p2)`

It is important to note, that `PredicateInstance` is not a build-in type and is currently only supported in decreases tuples as termination measure.
Usually for recursive function, a predicate instance can be used as a termination measure if the recursive call is inside an unfolding expression. The `listLength` function, which recursively calls itself inside an unfolding expression, is such a function.

```silver {.runnable}
import <decreases/predicate_instance.vpr>

field elem: Int
field next: Ref

predicate list(this: Ref) {
  acc(this.elem) && acc(this.next) &&
  (this.next != null ==> list(this.next))
}

function listLength(l:Ref) : Int
  decreases list(l)
  requires list(l)
  ensures  result > 0
{ 
  unfolding list(l) in l.next == null ? 1 : 1 + listLength(l.next) 
}
```

If all provided well-founded orders are required, the file `decreases/all.vpr` can be imported.

How well-founded orders can be defined is explained later in this chapter.

## Tuple

In the examples given above, the termination measures were always single expressions. This approach works easily for many cases, but sometimes it can be difficult to find a single expression which decreases well-founded in each iteration. Viper, therefore, allows a tuple of expressions to be defined as termination measure. The expressions of the tuple are then considered in lexicographical order.

A function for which normally a tuple as termination measure is used to proof termination is the Ackermann function.

```silver {.runnable}
import <decreases/int.vpr>

function ack(m:Int, n:Int):Int
decreases m,n
requires m >= 0
requires n >= 0
ensures result >= 0
{
  m==0 ? n+1 :
    n==0 ?
      ack(m-1,1) :
      ack(m-1, ack(m, n-1))
}
```

For the first recursive call `ack(m-1,1)` and the second recursive call `ack(m-1, ack(m, n-1))` the first expression of the tuple (i.e. `m`) decreases, hence, the other part of the tuple is not required to decrease. In the third and nested recursive call `ack(m, n-1)` the first expression of the tuple stays unchanged while the second expression (i.e. `n`) decreases.

## Decreases Wildcard

In a function for which Viper checks termination, i.e. for which a termination measure is defined, any function call must also terminate. Since functions for which no termination measure is defined are considered not to terminate, this would require that any function directly or indirectly invoked by another function for which a termination measure is defined also a termination measure must be provided. This can become cumbersome if termination of some functions can be assumed or really difficult in other cases.

Therefore, Viper allows to define functions for which termination can be assumed but is not checked by Viper. This is accomplished by providing a *decreases wildcard* instead of a decreases tuple.

```silver
decreases _
```

## Define Well-Founded Order

So far in the presented examples, only build-in types (and predicate instances) were used in the termination measures. For each of these types a file provided by Viper had to be imported, which defined a well-founded order for it.
To be able to use some additionally defined type as a termination measure either a function mapping to a build-in type has to be used or well-founded order on the additional type has to be defined. Using one example we will show the first approach and then present the second approach.

As 

## Conditions

_Note: this section introduces an extension to the decreases clauses presented above, which users who are just starting-out with Viper may wish to skip over for the moment._

In many cases termination should be proven (or assumed) in any possible execution. However, this might not always be the case and termination should only be proven (or assumed) under certain conditions.
Such condition can be provided after decreases clauses.

```silver
decreases [exp] if [condition]
```

```silver
decreases _ if [condition]
```

`[condition]` is a boolean expression under which the decreases clause is considered. The condition `true` is equivalent to the decreases clause without any condition (as shown in the examples above). The condition `false` is equivalent to not providing the decreases clause.

The following example shows a method which clearly terminates if it is invoked with a non-negative `x`, which should be checked by using `x` as termination measure. However, if invoked with a negative `x`, termination should be assumed.

```silver {.runnable}
import <decreases/int.vpr>

method m(x: Int)
  decreases x if x >= 0
  decreases _ if x < 0
{
  if (x >= 0){
    if (x != 0){
      m(x-1)
    }
  }else{
    var y: Int
    m(y)
  }
}
```

We introduce now the concept of the *termination specification*. For functions, methods or loops, a termination specifications encapsulate the provided decreases clauses. For each termination specification (i.e. for each function, method or loop) at most one decreases tuple and one decreases wildcard can be defined. The condition of the decreases tuple defines the *tuple condition*. The disjunction of the conditions of the decreases tuple and the decreases wildcard defines the *termination condition*.

For the method `m` from the example above, the tuple condition is `x >= 0` and the termination condition is `x >= 0 || x < 0`, which is equivalent to `true`.
Assuming the tuple condition, it is checked that the termination measure (here `x`) will decreases as well as that the tuple condition will still be satisfied.
After a successful verification the tuple condition defines the states in which the termination measure definitely decreases.
The termination condition is checked to be satisfied if termination is required.

## Mutually Recursive Functions and Effective Termination Measure