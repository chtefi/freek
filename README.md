# Freek

## The simple Free enhance for effects combining

### Motivations

At [ProjectSeptember](www.projectseptember.com), we love typesafe & functional programming.

We also like the concept of `Free Monad` which decouples completely the description of your program from its execution.

Free has a cost in terms of performance & short-term structures allocation but in our domain, IO is much more the bottleneck than these aspects so we can use those concepts without remorse.

We also believe current implementations can be improved progressively using theoretical and even brutal compiler optimizations. So we want to push those concepts further and further.

In Cats & Scalaz, `Free[F, A]` representation has already been optimized into a right associated structure and now embeds Coyoneda trick removing the dependency on Functor. Now we can use any effect/container `F[_]` in a `Free[F, A]` which is great.

Free is often associated to effects too and there are also interesting newer approaches in Scala world:

- [Emm](https://github.com/djspiewak/emm) : A very clever idea to represent stack of effects and a nice implementation (from which a few ideas have been stolen for freek). It's interesting because it can use existing structures but it has many implicits meaning it has a cost at compile-time...

- [Scala Eff](http://atnos-org.github.io/eff-cats/): a more theoretical and deeply interesting implementation based on [Freer Monads, more extensible effects](http://okmij.org/ftp/Haskell/extensible/more.pdf)'s paper by Oleg Kiselyov & friends. It's the next-gen of effects management but it also requires more _aware developers_ certainly... next step

- [Idris Eff port](https://github.com/mandubian/scalaeff): this is my personal toy... In Idris, it's (almost) nice, in Scala, it's more an experiment for now showing it could work. But it's the _next-gen + 1/2/3/4_ IMHO so let's be patient and make it grow (or not :))

> But for now, `Free` starts to enter in mind of people so let's try to use it as is with a few enhancements.

So in general, you want to use simple DSL like:

```scala

//////////////////////////////////////////////////////////////////////////
// LOG DSL
object Log {
  sealed trait LogLevel
  case object ErrorLevel extends LogLevel
  case object WarnLevel extends LogLevel
  case object InfoLevel extends LogLevel
  case object DebugLevel extends LogLevel

  trait DSL[A]
  case class LogMsg(level: LogLevel, msg: String) extends DSL[Unit]

  /** just helpers */
  def debug(msg: String) = LogMsg(DebugLevel, msg)
  def info(msg: String) = LogMsg(InfoLevel, msg)
}

//////////////////////////////////////////////////////////////////////////
// DB DSL
object DB {

  // DB DSL
  type Entity = Map[String, String]

  sealed trait DBError 
  case object NotFound extends DBError

  sealed trait DSL[A]
  case class FindById(id: String) extends DSL[Xor[DBError, Entity]]

}

```

... and you want to build simple programs like:

```scala
object DBService {

  /** Pseudo code */
  def findById(id: String): Free[F, A] = 
    for {
      _    <- Log.debug("Searching for entity id:"+id)
      res  <- FindById(id)
      _    <- Log.debug("Search result:"+res)
    } yield (res)
}
```

#### What is this `F` here?

Logically, it's combining `Log.DSL[_]` and `DB.DSL[_]`, right?
But how?
We could represent that using Shapeless Coproduct

```
// Log.DSL or DB.DSL
t => DB.DSL[A] :+: Log.DSL[t] :+: CNil
```

`A` is the type returned by `FindById` ie `Xor[DBError, Entity]`

So we have:

```scala
// Pseudo scala code
type PRG[A] = DB.DSL[A] :+: Log.DSL[t] :+: CNil

def findById(id: String): Free[PRG, Xor[DBError, Entity]] =
  for {
    _    <- Log.debug("Searching for entity id:"+id)
    res  <- FindById(id)
    _    <- Log.debug("Search result:"+res)
  } yield (res)
```

Now we want to execute this program, right?

First, we need interpreters in the shape of natural transformations

```scala
//////////////////////////////////////////////////////////////////////////
// Interpreters as simple TransNat
object Logger extends (Log.DSL ~> Id) {
  def apply[A](a: Log.DSL[A]) = a match {
    case Log.LogMsg(lvl, msg) =>
      println(s"$lvl $msg")
  }
}

object DBManager extends (DB.DSL ~> Id) {
  def apply[A](a: DB.DSL[A]) = a match {
    case DB.FindById(id) =>
      println(s"DB Finding $id")
      Xor.right(Map("id" -> id, "name" -> "toto"))
  }
}
```

Finally we want to execute our `findById` program using those interpreters.
That is done using `foldMap`

```scala
// Pseudo scala code
val interpreter: F ~> Id = DBManager combine Logger
findById(...).foldMap(interpreter).run
```

but what is `F[_]` here?
Our `type PRG[A] = DB.DSL[A] :+: Log.DSL[t] :+: CNil` naturally!

and `combine` is the operation:

```scala
(DB.DSL ~> Id combine Log.DSL) ~> Id => (DB.DSL :+: Log.DSL :+: CNil) ~> Id
```

#### Nice isn't it? But no, it doesn't work ouf of the box :(

- Shapeless Coproduct isn't very good for Coproduct of higher-kinded structures (or any other Coproducts I know).

`t => F[t] :+: (t => G[t] :+: CNil)` isn't `t => F[t] :+: G[t] :+: CNil`

- Combining interpreters like that doesn't work

- As you manipulate higher-kinded structures, you quickly hit the sadly famous `SI2712` issue.


## Freek, freer yet simple Free to combine your DSL seamlessly

Freek is not much, just a few helpers to make previous use-case straightforward.

Here are the ingredients of it:

### CoproductK

This is a specialized implementation of Shapeless Coproduct for higher-kinded structures allowing:

```
t => F[t] :@: (t => G[t] :@: CNilK[t])` is `t => F[t] :@: G[t] :@: CNilK[t]
```

Naturally, the type is a bit more complicated than simpler version so there are a few helpers to allow building things with a nice syntax.

In Freek, you would write it like that:

```
type PRG[A] = (Log.DSL :@: DB.DSL :@: FXNil)#Cop[A]
```

### `.freek[PRG]`

You are manipulating `DSL` but you want them to become Free to be able to bind them in a monadic flow.
But not any Free, you want a Free of your `PRG` which combines all your DSL.

Freek makes it straighforward using `.freek[PRG]` which lifts your nice little `DSL[A]` into a super-powerful `Free[PRG, A]`.

```scala

def findById(id: String): Free[PRG, Xor[DBError, Entity]] = 
  for {
    _    <- Log.debug("Searching for entity id:"+id).freek[PRG]
    res  <- FindById(id).freek[PRG]
    _    <- Log.debug("Search result:"+res).freek[PRG]
  } yield (res)
```


### Interpreters combining

Using `CoproductK`, it is also able to combine several natural transformations of a DSL into natural transformation of `CoproductK` of those DSL.

```
val interpreter: PRG ~> Id = DBManager :@: Logger
```

### SI2712 patch

SI2712 recent patch released by @milessabin has changed a lot the way we can build type-intensive libraries because we aren't limited by this terrible issue.

> We need that [PR](https://github.com/scala/scala/pull/5102#issuecomment-219868111) merged in Scala 2.12, it's really important ;)

In Freek, it is used by default as it allows writing code without ugly `.asInstanceOf` to help poor Scalac.

Thanks @milessabin again!

## Freek, finally all together

```

//////////////////////////////////////////////////////////////////////////
// LOG DSL
object Log {
  sealed trait LogLevel
  case object ErrorLevel extends LogLevel
  case object WarnLevel extends LogLevel
  case object InfoLevel extends LogLevel
  case object DebugLevel extends LogLevel

  trait DSL[A]
  case class LogMsg(level: LogLevel, msg: String) extends DSL[Unit]

  /** just helpers */
  def debug(msg: String) = LogMsg(DebugLevel, msg)
  def info(msg: String) = LogMsg(InfoLevel, msg)
}

//////////////////////////////////////////////////////////////////////////
// DB DSL
object DB {

  // DB DSL
  type Entity = Map[String, String]

  sealed trait DBError 
  case object NotFound extends DBError

  sealed trait DSL[A]
  case class FindById(id: String) extends DSL[Xor[DBError, Entity]]

}

//////////////////////////////////////////////////////////////////////////
// Program
object DBService {
  import DB._

  type PRG[A] = (Log.DSL :@: DB.DSL :@: FXNil)#Cop[A]

  /** the program */
  def findById(id: String): Free[PRG, Xor[DBError, Entity]] = 
    for {
      _    <- Log.debug("Searching for entity id:"+id).freek[PRG]
      res  <- FindById(id).freek[PRG]
      _    <- Log.debug("Search result:"+res).freek[PRG]
    } yield (res)
}

//////////////////////////////////////////////////////////////////////////
// Interpreters
object Logger extends (Log.DSL ~> Id) {
  def apply[A](a: Log.DSL[A]) = a match {
    case Log.LogMsg(lvl, msg) =>
      println(s"$lvl $msg")
  }
}

object DBManager extends (DB.DSL ~> Id) {
  def apply[A](a: DB.DSL[A]) = a match {
    case DB.FindById(id) =>
      println(s"DB Finding $id")
      Xor.right(Map("id" -> id, "name" -> "toto"))
  }
}

//////////////////////////////////////////////////////////////////////////
// Execution
val interpreter: Interpreter[PRG, Id] = Logger :@: DBManager

DBService.findById(XXX).foldMap(interpreter.nat)

```

THE END...