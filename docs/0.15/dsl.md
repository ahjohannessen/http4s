---
layout: default
title: The http4s DSL
---

Recall from earlier that an `HttpService` is just a type alias for
`Kleisli[Task, Request, Response]`.  This provides a minimal
foundation for declaring services and executing them on blaze or a
servlet container.  While this foundation is composeable, it is not
highly productive.  Most service authors will seek a higher level DSL.

## Add the http4s-dsl to your build

One option is the http4s-dsl.  It is officially supported by the
http4s team, but kept separate from core in order to encourage
multiple approaches for different needs.

This tutorial assumes that http4s-dsl is on your classpath.  Add the
following to your build.sbt:

```scala
libraryDependencies ++= Seq(
  "org.http4s" %% "http4s-dsl" % http4sVersion,
)
```

All we need is a REPL to follow along at home:

```
$ sbt console
```

## The simplest service

We'll need the following imports to get started:

```scala
import org.http4s._, org.http4s.dsl._
// import org.http4s._
// import org.http4s.dsl._

import scalaz.concurrent.Task
// import scalaz.concurrent.Task
```

The central concept of http4s-dsl is pattern matching.  An
`HttpService` is declared as a simple series of case statements.  Each
case statement attempts to match and optionally extract from an
incoming `Request`.  The code associated with the first matching case
is used to generate a `Task[Response]`.

The simplest case statement matches all requests without extracting
anything.  The right hand side of the request must return a
`Task[Response]`.

```scala
val service = HttpService {
  case _ =>
    Task.delay(Response(Status.Ok))
}
// service: org.http4s.HttpService = Kleisli(<function1>)
```

## Testing the Service

One beautiful thing about the `HttpService` model is that we don't
need a server to test our route.  We can construct our own request
and experiment directly in the REPL.

```scala
scala> val getRoot = Request(Method.GET, uri("/"))
getRoot: org.http4s.Request = Request(method=GET, uri=/, headers=Headers()

scala> val task = service.run(getRoot)
task: scalaz.concurrent.Task[org.http4s.Response] = scalaz.concurrent.Task@93ba5c8
```

Where is our `Response`?  It hasn't been created yet.  We wrapped it
in a `Task`.  In a real service, generating a `Response` is likely to
be an asynchronous operation with side effects, such as invoking
another web service or querying a database, or maybe both.  Operating
in a `Task` gives us control over the sequencing of operations and
lets us reason about our code like good functional programmers.  It is
the `HttpService`'s job to describe the task, and the server's job to
run it.

But here in the REPL, it's up to us to run it:

```scala
scala> val response = task.run
<console>:22: warning: method run in class Task is deprecated: use unsafePerformSync
       val response = task.run
                           ^
response: org.http4s.Response = Response(status=200, headers=Headers())
```

Cool.

## Generating responses

We'll circle back to more sophisticated pattern matching of requests,
but it will be a tedious affair until we learn a more succinct way of
generating `Task[Response]`s.

### Status codes

http4s-dsl provides a shortcut to create a `Task[Response]` by
applying a status code:

```scala
scala> val okTask = Ok()
okTask: scalaz.concurrent.Task[org.http4s.Response] = scalaz.concurrent.Task@377b086

scala> val ok = okTask.run
<console>:20: warning: method run in class Task is deprecated: use unsafePerformSync
       val ok = okTask.run
                       ^
ok: org.http4s.Response = Response(status=200, headers=Headers())
```

This simple `Ok()` expression succinctly says what we mean in a
service:

```scala
HttpService {
  case _ => Ok()
}.run(getRoot).run
// <console>:23: warning: method run in class Task is deprecated: use unsafePerformSync
//        }.run(getRoot).run
//                       ^
// res0: org.http4s.Response = Response(status=200, headers=Headers())
```

This syntax works for other status codes as well.  In our example, we
don't return a body, so a `204 No Content` would be a more appropriate
response:

```scala
HttpService {
  case _ => NoContent()
}.run(getRoot).run
// <console>:23: warning: method run in class Task is deprecated: use unsafePerformSync
//        }.run(getRoot).run
//                       ^
// res1: org.http4s.Response = Response(status=204, headers=Headers())
```

### Responding with a body

#### Simple bodies

Most status codes take an argument as a body.  In http4s, `Request`
and `Response` bodies are represented as a
`scalaz.stream.Process[Task, ByteVector]`.  It's also considered good
HTTP manners to provide a `Content-Type` and, where known in advance,
`Content-Length` header in one's responses.

All of this hassle is neatly handled by http4s' [EntityEncoder]s.
We'll cover these in more depth in another tut.  The important point
for now is that a response body can be generated for any type with an
implicit `EntityEncoder` in scope.  http4s provides several out of the
box:

```scala
scala> Ok("Received request.").run
<console>:20: warning: method run in class Task is deprecated: use unsafePerformSync
       Ok("Received request.").run
                               ^
res2: org.http4s.Response = Response(status=200, headers=Headers(Content-Type: text/plain; charset=UTF-8, Content-Length: 17))

scala> import java.nio.charset.StandardCharsets.UTF_8
import java.nio.charset.StandardCharsets.UTF_8

scala> Ok("binary".getBytes(UTF_8)).run
<console>:21: warning: method run in class Task is deprecated: use unsafePerformSync
       Ok("binary".getBytes(UTF_8)).run
                                    ^
res3: org.http4s.Response = Response(status=200, headers=Headers(Content-Type: application/octet-stream, Content-Length: 6))
```

Per the HTTP specification, some status codes don't support a body.
http4s prevents such nonsense at compile time:

```scala
scala> NoContent("does not compile")
<console>:21: error: too many arguments for method apply: ()scalaz.concurrent.Task[org.http4s.Response] in trait EmptyResponseGenerator
       NoContent("does not compile")
                ^
```

#### Asynchronous responses

While http4s prefers `Task`, you may be working with libraries that
use standard library [Future]s.  Some relevant imports:

```scala
import scala.concurrent.Future
// import scala.concurrent.Future

import scala.concurrent.ExecutionContext.Implicits.global
// import scala.concurrent.ExecutionContext.Implicits.global
```

You can seamlessly respond with a `Future` of any type that has an
`EntityEncoder`.

```scala
scala> val task = Ok(Future {
     |   println("I run when the future is constructed.")
     |   "Greetings from the future!"
     | })
task: scalaz.concurrent.Task[org.http4s.Response] = scalaz.concurrent.Task@744623cc

scala> task.run
<console>:24: warning: method run in class Task is deprecated: use unsafePerformSync
       task.run
            ^
res5: org.http4s.Response = Response(status=200, headers=Headers(Content-Type: text/plain; charset=UTF-8, Content-Length: 26))
```

As good functional programmers who like to delay our side effects, we
of course prefer to operate in [Task]s:

```scala
scala> val task = Ok(Task {
     |   println("I run when the Task is run.")
     |   "Mission accomplished!"
     | })
task: scalaz.concurrent.Task[org.http4s.Response] = scalaz.concurrent.Task@770b703e

scala> task.run
<console>:24: warning: method run in class Task is deprecated: use unsafePerformSync
       task.run
            ^
I run when the Task is run.
res6: org.http4s.Response = Response(status=200, headers=Headers(Content-Type: text/plain; charset=UTF-8, Content-Length: 21))
```

Note that in both cases, a `Content-Length` header is calculated.
http4s waits for the `Future` or `Task` to complete before wrapping it
in its HTTP envelope, and thus has what it needs to calculate a
`Content-Length`.

#### Streaming bodies

Streaming bodies are supported by returning a `scalaz.stream.Process`.
Like `Future`s and `Task`s, the stream may be of any type that has an
`EntityEncoder`.

An intro to scalaz-stream is out of scope, but we can glimpse the
power here.  This stream emits the elapsed time every 100 milliseconds
for one second:

```scala
val drip = {
  import scala.concurrent.duration._
  implicit def defaultScheduler = scalaz.concurrent.Strategy.DefaultTimeoutScheduler
  scalaz.stream.time.awakeEvery(100.millis).map(_.toString).take(10)
}
// drip: scalaz.stream.Process[scalaz.concurrent.Task,String] = Append(Halt(End),Vector(<function1>))
```

We can see it for ourselves in the REPL:

```scala
scala> drip.to(scalaz.stream.io.stdOutLines).run.run
<console>:24: warning: method run in class Task is deprecated: use unsafePerformSync
       drip.to(scalaz.stream.io.stdOutLines).run.run
                                                 ^
```

When wrapped in a `Response`, http4s will flush each chunk of a
`Process` as they are emitted.  Note that a stream's length can't
generally be anticipated before it runs, so this triggers chunked
transfer encoding:

```scala
scala> Ok(drip).run
<console>:24: warning: method run in class Task is deprecated: use unsafePerformSync
       Ok(drip).run
                ^
res8: org.http4s.Response = Response(status=200, headers=Headers(Content-Type: text/plain; charset=UTF-8, Transfer-Encoding: chunked))
```

## Matching and extracting requests

A `Request` is a regular `case class` - you can destructure it to extract its
values. By extension, you can also `match/case` it with different possible
destructurings. To build these different extractors, you can make use of the
DSL.

Most often, you extract the `Request` into a HTTP `Method` (verb) and the path,
via the `->` object. On the left side, you'll have the HTTP `Method`, on the
other side the path. Naturally, `_` is a valid matcher too, so any call to
`/api` can be blocked, regardless of `Method`:

```scala
scala> HttpService {
     |   case request @ _ -> Root / "api" => Forbidden()
     | }
res9: org.http4s.HttpService = Kleisli(<function1>)
```

To also block all subcalls `/api/...`, you'll need `/:`, which is right
associative, and matches everything after, and not just the next element:

```scala
scala> HttpService {
     |   case request @ _ -> "api" /: _ => Forbidden()
     | }
res10: org.http4s.HttpService = Kleisli(<function1>)
```

For matching more than one `Method`, there's `|`:

```scala
scala> HttpService {
     |   case request @ (GET | POST) -> Root / "api"  => ???
     | }
res11: org.http4s.HttpService = Kleisli(<function1>)
```

Honorable mention: `~`, for matching file extensions.

```scala
scala> HttpService {
     |   case GET -> Root / file ~ "json" => Ok(s"""{"response": "You asked for $file"}""")
     | }
res12: org.http4s.HttpService = Kleisli(<function1>)
```

### Handling path parameters
Path params can be extracted and converted to a specific type but are
`String`s by default. There are numeric extractors provided in the form
of `IntVar` and `LongVar`.

```scala
import scalaz.concurrent.Task
// import scalaz.concurrent.Task

def getUserName(userId: Int): Task[String] = ???
// getUserName: (userId: Int)scalaz.concurrent.Task[String]

val usersService = HttpService {
  case request @ GET -> Root / "users" / IntVar(userId) =>
    Ok(getUserName(userId))
}
// usersService: org.http4s.HttpService = Kleisli(<function1>)
```

If you want to extract a variable of type `T`, you can provide a custom extractor
object which implements `def unapply(str: String): Option[T]`, similar to the way
in which `IntVar` does it.

```scala
import java.time.LocalDate
// import java.time.LocalDate

import scala.util.Try
// import scala.util.Try

import scalaz.concurrent.Task
// import scalaz.concurrent.Task

import org.http4s.client._
// import org.http4s.client._

object LocalDateVar {
  def unapply(str: String): Option[LocalDate] = {
    if (!str.isEmpty)
      Try(LocalDate.parse(str)).toOption
    else
      None
  }
}
// defined object LocalDateVar

def getTemperatureForecast(date: LocalDate): Task[Double] = Task(42.23)
// getTemperatureForecast: (date: java.time.LocalDate)scalaz.concurrent.Task[Double]

val dailyWeatherService = HttpService {
  case request @ GET -> Root / "weather" / "temperature" / LocalDateVar(localDate) =>
    Ok(getTemperatureForecast(localDate).map(s"The temperature on $localDate will be: " + _))
}
// dailyWeatherService: org.http4s.HttpService = Kleisli(<function1>)

println(GET(Uri.uri("/weather/temperature/2016-11-05")).flatMap(dailyWeatherService).run)
// <console>:33: warning: method run in class Task is deprecated: use unsafePerformSync
//        println(GET(Uri.uri("/weather/temperature/2016-11-05")).flatMap(dailyWeatherService).run)
//                                                                                             ^
// Response(status=200, headers=Headers(Content-Type: text/plain; charset=UTF-8, Content-Length: 44))
```

### Handling query parameters
A query parameter needs to have a `QueryParamDecoderMatcher` provided to
extract it. In order for the `QueryParamDecoderMatcher` to work there needs to
be an implicit `QueryParamDecoder[T]` in scope. `QueryParamDecoder`s for simple
types can be found in the `QueryParamDecoder` object. There are also
`QueryParamDecoderMatcher`s available which can be used to
return optional or validated parameter values.

In the example below we're finding query params named `country` and `year` and
then parsing them as a `String` and `java.time.Year`.

```scala
import java.time.Year
// import java.time.Year

import scalaz.ValidationNel
// import scalaz.ValidationNel

object CountryQueryParamMatcher extends QueryParamDecoderMatcher[String]("country")
// defined object CountryQueryParamMatcher

implicit val yearQueryParamDecoder = new QueryParamDecoder[Year] {
  def decode(queryParamValue: QueryParameterValue): ValidationNel[ParseFailure, Year] = {
    QueryParamDecoder.decodeBy[Year, Int](Year.of).decode(queryParamValue)
  }
}
// yearQueryParamDecoder: org.http4s.QueryParamDecoder[java.time.Year] = $anon$1@135ac1e9

object YearQueryParamMatcher extends QueryParamDecoderMatcher[Year]("year")
// defined object YearQueryParamMatcher

def getAverageTemperatureForCountryAndYear(country: String, year: Year): Task[Double] = ???
// getAverageTemperatureForCountryAndYear: (country: String, year: java.time.Year)scalaz.concurrent.Task[Double]

val averageTemperatureService = HttpService {
  case request @ GET -> Root / "weather" / "temperature" :? CountryQueryParamMatcher(country) +& YearQueryParamMatcher(year)  =>
    Ok(getAverageTemperatureForCountryAndYear(country, year).map(s"Average temperature for $country in $year was: " + _))
}
// averageTemperatureService: org.http4s.HttpService = Kleisli(<function1>)
```