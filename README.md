Author

![Sadham Hussian M](https://miro.medium.com/fit/c/96/96/1*brrZFIIEnEvjw1yTvqRZ4Q.jpeg)
Sadham Hussian M

Application Monitoring in Golang application with Elastic APM
=============================================================

[Elastic APM](https://www.elastic.co/solutions/apm) (application performance monitoring) provides rich insights into application performance and visibility for distributed workloads, with official support for a number of languages including [Go](https://www.elastic.co/guide/en/apm/agent/go/1.x/index.html), [Java](https://www.elastic.co/guide/en/apm/agent/java/1.x/index.html), [Ruby](https://www.elastic.co/guide/en/apm/agent/ruby/2.x/index.html), [Python](https://www.elastic.co/guide/en/apm/agent/python/4.x/index.html), and JavaScript ([Node.js](https://www.elastic.co/guide/en/apm/agent/nodejs/2.x/index.html) and [real user monitoring](https://www.elastic.co/guide/en/apm/agent/js-base/4.x/index.html) (RUM) for the browser). We’re going to see how to use Elastic APM to trace our Go APIs.

**Elastic APM**
===============

Elastic APM consists of four components:

1.  APM agents
2.  Elastic APM Integration
3.  Elasticsearch
4.  Kibana

There are two ways that these four components can be integrated, we are going to discuss one way in which APM agents on edge machines send data to a centrally hosted APM integration. APM agents are basically a software package that will be installed in all the services which you want to monitor. APM agent will collect the log and send it to the APM integration, and finally, we can visualize the data in Kibana.

Set up the APM server and Fleet by following the steps in the official [documentation](https://www.elastic.co/guide/en/apm/guide/current/apm-quick-start.html).

Once you have set up the stack and configured your application to send data to the APM Server. You will need to know the APM Server’s URL and secret token. The APM Go agent is configured via environment variables.

To configure the APM Server URL, secret token, and service name used to identify the microservice, export the following environment variables, so they will get picked up by your application:

```
export ELASTIC\_APM\_SERVER\_URL=https://-------:443
export ELASTIC\_APM\_SECRET\_TOKEN=-----------------
export ELASTIC\_APM\_SERVICE\_NAME=any\_name
```

After implementing all the features below in the blog, your APM dashboard should look like this

**Tracing Web requests**
========================

The Elastic APM Go agent supplies a number of modules for instrumenting various web frameworks, RPC frameworks, and database drivers; and for integrating with several logging frameworks. Check out the [full list of the supported technologies](https://www.elastic.co/guide/en/apm/agent/go/1.x/supported-tech.html).

```
package main
import (
 "fmt"
 "net/http"
 "time"
 "github.com/gin-contrib/cors"
 "github.com/gin-gonic/gin"
 "go.elastic.co/apm/module/apmgin"
)
func SetGINMode() bool {
 gin.SetMode(gin.ReleaseMode)
 return true
}
var (
 \_      = SetGINMode()
 router = gin.New()
)
func main() {
 router.Use(cors.Default())
 router.Use(apmgin.Middleware(router))
 router.GET("/ping", func(c \*gin.Context) {
  c.String(http.StatusOK, "pong")
 })
 s := &http.Server{
  Addr:         fmt.Sprintf(":%v", 8082),
  Handler:      router,
  ReadTimeout:  1 \* time.Minute,
  WriteTimeout: 1 \* time.Minute,
  IdleTimeout:  1 \* time.Minute,
 }
 s.ListenAndServe()
}
```

**SQL Queries**
===============

```
package main

import (
 "fmt"
 "log"
 "net/http"
 "time"
 "elasti-apm-poc/callbacks"
 "github.com/gin-contrib/cors"
 "github.com/gin-gonic/gin"
 "gorm.io/gorm"
 "go.elastic.co/apm"
 "go.elastic.co/apm/module/apmgorm"
 \_ "go.elastic.co/apm/module/apmgorm/dialects/mysql"
)
func SetGINMode() bool {
 gin.SetMode(gin.ReleaseMode)
 return true
}
var (
 \_      = SetGINMode()
 router = gin.New()
 db     \*gorm.DB
)
func main() {
  db, err := apmgorm.Open("mysql", fmt.Sprintf("%s:%s@(%s:%s)/%s?charset=utf8&parseTime=True&loc=Local&charset=utf8mb4&collation=utf8mb4\_unicode\_ci",
  "root",
  "root",
  "localhost",
  "3306",
  "orders",
 ))
 if err != nil {
  panic(err)
 }
 router.Use(cors.Default())
 router.Use(router)
 router.GET("/query", func(c \*gin.Context) {
  type Result struct {
   count int
  }
  ctx := c.Request.Context()
  var result Result
  db.Set("elasticapm:context", ctx).Raw("SELECT COUNT(\*) AS count FROM orders").Scan(&result)
  var count int
  db.Set("elasticapm:context", ctx).Table("orders").Count(&count)
  c.JSON(200, result)
 })
}
```

**Tracing outgoing HTTP**
=========================

```
package main

import (
 "fmt"
 "log"
 "net/http"
 "time"
 "github.com/gin-contrib/cors"
 "github.com/gin-gonic/gin"
 "go.elastic.co/apm"
 "go.elastic.co/apm/module/apmgin"
 "go.elastic.co/apm/module/apmhttp"
)
func SetGINMode() bool {
 gin.SetMode(gin.ReleaseMode)
 return true
}
var (
 \_      = SetGINMode()
 router = gin.New()
 db     \*gorm.DB
)
func main() {
 router.Use(cors.Default())
 router.GET("/tracehttp", func(c \*gin.Context) {
  ctx := c.Request.Context()
  req, err := http.NewRequest("GET", "http://google.com", nil)
  if err != nil {
   log.Println(err)
   c.JSON(http.StatusInternalServerError, err)
  }
  client := apmhttp.WrapClient(http.DefaultClient)
  resp, err := client.Do(req.WithContext(ctx))
  if err != nil {
   log.Println(err)
   c.JSON(http.StatusInternalServerError, err)
  }
  c.JSON(http.StatusOK, resp.Body)
 })
 s := &http.Server{
  Addr:         fmt.Sprintf(":%v", 8082),
  Handler:      router,
  ReadTimeout:  1 \* time.Minute,
  WriteTimeout: 1 \* time.Minute,
  IdleTimeout:  1 \* time.Minute,
 }
 s.ListenAndServe()
}
```

**Tracing custom spans**
========================

```
package main

import (
 "fmt"
 "log"
 "net/http"
 "time"
 "github.com/gin-contrib/cors"
 "github.com/gin-gonic/gin"
 "go.elastic.co/apm"
 "go.elastic.co/apm/module/apmgin"
 "go.elastic.co/apm/module/apmhttp"
)
func SetGINMode() bool {
 gin.SetMode(gin.ReleaseMode)
 return true
}
var (
 \_      = SetGINMode()
 router = gin.New()
 db     \*gorm.DB
)
func main() {
 router.Use(cors.Default())
 router.GET("/tracehttp", func(c \*gin.Context) {
ctx := c.Request.Context()
  span, ctx := apm.StartSpan(ctx, "getNameStats", "custom")
  defer span.End()


  req, err := http.NewRequest("GET", "http://google.com", nil)
  if err != nil {
   log.Println(err)
   c.JSON(http.StatusInternalServerError, err)
  }
  client := apmhttp.WrapClient(http.DefaultClient)
  resp, err := client.Do(req.WithContext(ctx))

  if err != nil {
   log.Println(err)
   c.JSON(http.StatusInternalServerError, err)
  }
  c.JSON(http.StatusOK, resp.Body)
 })
 s := &http.Server{
  Addr:         fmt.Sprintf(":%v", 8082),
  Handler:      router,
  ReadTimeout:  1 \* time.Minute,
  WriteTimeout: 1 \* time.Minute,
  IdleTimeout:  1 \* time.Minute,
 }
 s.ListenAndServe()
}
```

**Adding custom parameters to trace**
=====================================

```
package main

import (
 "fmt"
 "log"
 "net/http"
 "time"
 "github.com/gin-contrib/cors"
 "github.com/gin-gonic/gin"
 "go.elastic.co/apm"
 "go.elastic.co/apm/module/apmgin"
)
func SetGINMode() bool {
 gin.SetMode(gin.ReleaseMode)
 return true
}
var (
 \_      = SetGINMode()
 router = gin.New()
 db     \*gorm.DB
)
func main() {
 router.Use(cors.Default())
 router.GET("/ping", func(c \*gin.Context) {
  c.String(http.StatusOK, "pong")
 })
 router.GET("transaction", func(c \*gin.Context) {
  transaction := apm.TransactionFromContext(c.Request.Context())
  transaction.Context.SetLabel("request", c.Request.Body)
  transaction.Context.SetLabel("env", "DEV")
  c.String(http.StatusOK, "success")
 })
 s := &http.Server{
  Addr:         fmt.Sprintf(":%v", 8082),
  Handler:      router,
  ReadTimeout:  1 \* time.Minute,
  WriteTimeout: 1 \* time.Minute,
  IdleTimeout:  1 \* time.Minute,
 }
 s.ListenAndServe()
}
```

**Conclusion**
==============

We can also use this data from APM and create our own dashboards using Kibana. Create data view for the traces-apm\*,apm-\*,logs-apm\*,apm-\*,metrics-apm\*,apm-\* index, and Kibana and make use of all the data in the index.
