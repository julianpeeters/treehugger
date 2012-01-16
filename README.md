## treehugger

treehugger is a library to write Scala source code programmatically. It's also an implementation of Scala AST based on Reflection API.

### story

It's fun to write a code that generates code, but it's not so fun writing code that strings Strings together. Then I started seeing some references to Reflection API lifting live code into AST (abstract syntax tree) as follows:

    scala> scala.reflect.Code.lift((x: Int) => x + 1).tree
    res0: scala.reflect.Tree = Function(List(LocalValue(NoSymbol,x,PrefixedType(ThisType(Class(scala)),Class(scala.Int)))),Apply(Select(Ident(LocalValue(NoSymbol,x,PrefixedType(ThisType(Class(scala)),Class(scala.Int)))),Method(scala.Int.$plus,MethodType(List(LocalValue(NoSymbol,x,PrefixedType(ThisType(Class(scala)),Class(scala.Int)))),PrefixedType(ThisType(Class(scala)),Class(scala.Int))))),List(Literal(1))))

Who better knows the Scala syntax than the compiler, right? It seemed like a good place to get the implementation of the AST.
The first issue however, is that Reflection API isn't done. It's a 2.10 thing. Besides, I want something that worked across the Scala versions.

The second issue is that being the reflection API, its view of Scala code is a bit skewed towards runtime knowledge of scalac. For example, it doesn't have the concept of `for` expression. The compiler expands it into one of `map`/`flatMap`/`foreach` and never looks back.

Set aside some of the issues, Reflection API has many appealing points as well. First, much of the hard work has been done at least for the use of code generation. There's a cake pattern module called `TreePrinters`, which prints AST back to source code. There's also `TreeDSL` that helps building AST from code. Second, using similar code to the scalac implementation, it should require less learning for those who are already familiar with scalac code.

Thus, treehugger. Code equivalent of Frankenstein's creature patched up by borrowing scalac source.

## DSL

treehugger DSL is an expansion of `TreeDSL` in scalac. It already did many things, but some details were missing. Let's see the actual code. Assume at the beginning of the code there's the following set up:

### Hello world

```
lazy val universe = new treehugger.Universe
import universe._
import definitions._
import CODE._

object sym {
  val println = ScalaPackageClass.newMethod("println")
}

val tree = sym.println APPLY LIT("Hello, world!")
val s = treeToString(tree)
println(s)
```

The above prints:

```scala
println("Hello, world!")
```

If we remove all the setups, the actual AST comes down to:

```scala
sym.println APPLY LIT("Hello, world!")
```

The setup code will be abbreviated from here.

### method declaration

```scala
DEF("hello", UnitClass.toType) := BLOCK(
  sym.println APPLY LIT("Hello, world!"))
```

This prints out:

```scala
def hello() {
  println("Hello, world!")
}
```

### for expression and infix application

for expression and infix application are something that is completely missing in scalac's tree:

```scala
FOR(VALFROM("i") := LIT(0) INFIX (sym.to, LIT(2))) DO
  (sym.print APPLY (greetStrings APPLY REF("i")))
```

This prints out:

```scala
for (i <- 0 to 2)
  print(greetStrings(i))
```

### class, object, and package

class, object, and package declarations are something new to treehugger DSL:

```
val IntQueue: ClassSymbol = RootClass.newClass("IntQueue".toTypeName)

CLASSDEF(IntQueue) withFlags(ABSTRACT) := BLOCK(
  DEF("get", IntClass.toType).empty,
  DEF("put", UnitClass.toType) withParams(VAL("x", IntClass.toType).empty) empty
)
```

The above is an example of an abstract class declaration, which prints out:

```scala
abstract class IntQueue {
  def get(): Int
  def put(x: Int): Unit
}
```

### pattern matching

pattern matching was mostly in the original DSL (except `UNAPPLY` and `INFIXUNAPPLY`):

```scala
val maxListUpBound = RootClass.newMethod("maxListUpBound")
val T = maxListUpBound.newTypeParameter("T".toTypeName)
val upperboundT = TypeBounds.upper(orderedType(T.toType))

DEF(maxListUpBound.name, T.toType)
    withTypeParams(TypeDef(T, TypeTree(upperboundT))) withParams(VAL("elements", listType(T.toType)).empty) :=
  REF("elements") MATCH(
    CASE(ListClass UNAPPLY()) ==> THROW(IllegalArgumentExceptionClass, "empty list!"),
    CASE(ListClass UNAPPLY(ID("x"))) ==> REF("x"),
    CASE(ID("x") INFIXUNAPPLY("::", ID("rest"))) ==> BLOCK(
      VAL("maxRest") := maxListUpBound APPLY(REF("rest")),
      IF(REF("x") INFIX (">", REF("maxRest"))) THEN REF("x")
      ELSE REF("maxRest") 
    )
  )
```

This prints out:

```scala
def maxListUpBound[T <: Ordered[T]](elements: List[T]): T =
  elements match {
    case List() => throw new IllegalArgumentException("empty list!")
    case List(x) => x
    case x :: rest => {
      val maxRest = maxListUpBound(rest)
      if (x > maxRest) x
      else maxRest
    }
  }
```

## Licensing

Any code I took from scalac is covered by SCALA LICENSE.
Any other code I wrote is covered by MIT License.
