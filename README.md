# learning_akka_http
akka http learning roadmap


## gradle setting
```

group 'test'
version '1.0-SNAPSHOT'

apply plugin: 'java'
apply plugin: 'scala'

sourceCompatibility = 1.5

repositories {
    mavenCentral()
}

dependencies {
    testCompile group: 'junit', name: 'junit', version: '4.11'
    compile 'com.typesafe.akka:akka-actor_2.11:2.4.14'
    compile 'com.typesafe.akka:akka-agent_2.11:2.4.14'
    compile 'com.typesafe.akka:akka-camel_2.11:2.4.14'
    compile 'com.typesafe.akka:akka-cluster_2.11:2.4.14'
    compile 'com.typesafe.akka:akka-cluster-metrics_2.11:2.4.14'
    compile 'com.typesafe.akka:akka-cluster-sharding_2.11:2.4.14'
    compile 'com.typesafe.akka:akka-cluster-tools_2.11:2.4.14'
    compile 'com.typesafe.akka:akka-contrib_2.11:2.4.14'
    compile 'com.typesafe.akka:akka-multi-node-testkit_2.11:2.4.14'
    compile 'com.typesafe.akka:akka-osgi_2.11:2.4.14'
    compile 'com.typesafe.akka:akka-persistence_2.11:2.4.14'
    compile 'com.typesafe.akka:akka-persistence-tck_2.11:2.4.14'
    compile 'com.typesafe.akka:akka-remote_2.11:2.4.14'
    compile 'com.typesafe.akka:akka-slf4j_2.11:2.4.14'
    compile 'com.typesafe.akka:akka-stream_2.11:2.4.14'
    compile 'com.typesafe.akka:akka-stream-testkit_2.11:2.4.14'
    compile 'com.typesafe.akka:akka-testkit_2.11:2.4.14'
    compile 'com.typesafe.akka:akka-distributed-data-experimental_2.11:2.4.14'
    compile 'com.typesafe.akka:akka-typed-experimental_2.11:2.4.14'
    compile 'com.typesafe.akka:akka-persistence-query-experimental_2.11:2.4.14'

    compile 'com.typesafe.akka:akka-http-core_2.11:10.0.0'
    compile 'com.typesafe.akka:akka-http_2.11:10.0.0'
    compile 'com.typesafe.akka:akka-http-testkit_2.11:10.0.0'
    compile 'com.typesafe.akka:akka-http-spray-json_2.11:10.0.0'
    compile 'com.typesafe.akka:akka-http-jackson_2.11:10.0.0'
    compile 'com.typesafe.akka:akka-http-xml_2.11:10.0.0'
}
```


## low level
>  - low level中有handlewithasynchandler和handlewithsynchandler两种
>  - 前者异步调用反悔Future,后者同步
>  - 异步能接受更大并发，同步更快，酌情。
```
import akka.actor.ActorSystem
import akka.http.scaladsl.Http
import akka.http.scaladsl.model.HttpMethods._
import akka.http.scaladsl.model._
import akka.stream.ActorMaterializer
import akka.stream.scaladsl.Sink
import scala.concurrent.Future


object webServer extends App {
  implicit val system = ActorSystem("fefew")
  implicit val materializer = ActorMaterializer()
  implicit val executionContext = system.dispatcher
  val serverSource = Http().bind(interface = "localhost", port = 8080)

  val requestHandler: HttpRequest => HttpResponse = {
    case HttpRequest(GET, Uri.Path("/"), _, _, _) =>
      HttpResponse(entity = HttpEntity(
        ContentTypes.`text/html(UTF-8)`,
        "<html><body>Hello world!</body></html>"))

    case HttpRequest(GET, Uri.Path("/ping"), _, _, _) =>
      println("pong()")
      HttpResponse(entity = "PONG!")

    case HttpRequest(GET, Uri.Path("/crash"), _, _, _) =>
      sys.error("BOOM!")

    case r: HttpRequest =>
      r.discardEntityBytes() // important to drain incoming HTTP Entity stream
      HttpResponse(404, entity = "Unknown resource!")
  }

  val requestHandler2: HttpRequest => Future[HttpResponse] = {
    case r: HttpRequest => {
      println("other2")
      Future(HttpResponse(200, entity = HttpEntity(ContentTypes.`text/html(UTF-8)`,
        "<html><body>Hello world222222!</body></html>")))
    }
  }

  val bindingFuture: Future[Http.ServerBinding] =
    serverSource.to(Sink.foreach { connection =>
      println("Accepted new connection from " + connection.remoteAddress)
//      connection handleWithAsyncHandler requestHandler2
//      println("out-==================")
      connection handleWithSyncHandler requestHandler

      // this is equivalent to
//       connection handleWith { Flow[HttpRequest] map requestHandler }
    }).run()
}
``` 


## high-level

```
import akka.actor.{ActorRef, ActorSystem}
import akka.http.scaladsl.coding.Deflate
import akka.http.scaladsl.marshalling.ToResponseMarshaller
import akka.http.scaladsl.model.StatusCodes.MovedPermanently
import akka.http.scaladsl.server.Directives._
import akka.http.scaladsl.unmarshalling.FromRequestUnmarshaller
import akka.pattern.ask
import akka.stream.ActorMaterializer
import akka.util.Timeout

// types used by the API routes
type Money = Double // only for demo purposes, don't try this at home!
type TransactionResult = String
case class User(name: String)
case class Order(email: String, amount: Money)
case class Update(order: Order)
case class OrderItem(i: Int, os: Option[String], s: String)

// marshalling would usually be derived automatically using libraries
implicit val orderUM: FromRequestUnmarshaller[Order] = ???
implicit val orderM: ToResponseMarshaller[Order] = ???
implicit val orderSeqM: ToResponseMarshaller[Seq[Order]] = ???
implicit val timeout: Timeout = ??? // for actor asks
implicit val ec: ExecutionContext = ???
implicit val mat: ActorMaterializer = ???
implicit val sys: ActorSystem = ???

// backend entry points
def myAuthenticator: Authenticator[User] = ???
def retrieveOrdersFromDB: Seq[Order] = ???
def myDbActor: ActorRef = ???
def processOrderRequest(id: Int, complete: Order => Unit): Unit = ???

val route = {
  path("orders") {
    authenticateBasic(realm = "admin area", myAuthenticator) { user =>
      get {
        encodeResponseWith(Deflate) {
          complete {
            // marshal custom object with in-scope marshaller
            retrieveOrdersFromDB
          }
        }
      } ~
      post {
        // decompress gzipped or deflated requests if required
        decodeRequest {
          // unmarshal with in-scope unmarshaller
          entity(as[Order]) { order =>
            complete {
              // ... write order to DB
              "Order received"
            }
          }
        }
      }
    }
  } ~
  // extract URI path element as Int
  pathPrefix("order" / IntNumber) { orderId =>
    pathEnd {
      (put | parameter('method ! "put")) {
        // form extraction from multipart or www-url-encoded forms
        formFields(('email, 'total.as[Money])).as(Order) { order =>
          complete {
            // complete with serialized Future result
            (myDbActor ? Update(order)).mapTo[TransactionResult]
          }
        }
      } ~
      get {
        // debugging helper
        logRequest("GET-ORDER") {
          // use in-scope marshaller to create completer function
          completeWith(instanceOf[Order]) { completer =>
            // custom
            processOrderRequest(orderId, completer)
          }
        }
      }
    } ~
    path("items") {
      get {
        // parameters to case class extraction
        parameters(('size.as[Int], 'color ?, 'dangerous ? "no"))
          .as(OrderItem) { orderItem =>
            // ... route using case class instance created from
            // required and optional query parameters
            complete("") // hide
          }
      }
    }
  } ~
  pathPrefix("documentation") {
    // optionally compresses the response with Gzip or Deflate
    // if the client accepts compressed responses
    encodeResponse {
      // serve up static content from a JAR resource
      getFromResourceDirectory("docs")
    }
  } ~
  path("oldApi" / Remaining) { pathRest =>
    redirect("http://oldapi.example.com/" + pathRest, MovedPermanently)
  }
}
```