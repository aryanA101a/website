# HTTP Service

In order to communicate with another REST/HTTP applications, gofr provides **HTTP service**.
It can be customised by adding options to :

- Call to a service protected by Auth
- Surge Protection
- Custom Retry

## **Create an HTTP service**

You can create a new HTTP service by using below function:

```go
// resourceAddr is the base URL for the service which you want to communicate with
// logger is required to log the service request made. gofr logger present in the context can be used.
// options is a struct which helps you enable features of HTTPService.
NewHTTPServiceWithOptions(resourceAddr string, logger log.Logger, options *Options)
```

HTTPService provides the options of authentication, surge protection and service level headers. The struct `Options` help enable these features.

```go
type Options struct {
    Headers map[string]string // this can be used to pass service level headers.
    NumOfRetries int
    *Auth
    *Cache
    *SurgeProtectorOption
}
```

All these options can be used together. HTTPService enables these services based on what is sent in options. If these fields are set in Options then HTTPService will enable it.

## **Call to a service protected by Auth**

If there is a need to call a service which is protected by basic Auth/CSPAuth/OAuth, gofr enables it by setting the Auth option.

Options for Auth can be set in the following function:

```javascript
NewHTTPServiceWithOptions(url string, logger log.Logger, options *Options)
```

`Auth` contains required fields and structs for basic Auth, CSPAuth and OAuth.

```javascript

type Auth struct {
    UserName string
    Password string
    *OAuthOption
}
```

`UserName` and `Password` can be set to enable basic auth. The token will be generated by the framework.

- `OAuthOption` if the http service uses OAuth then this can be set.
- **Note:** The framework does not allow the setting of basic auth and OAuth for the same HTTP servic

Here, `OAuthOption` and `CSPOption` are structs defined as below.

```go
type OAuthOption struct {
    ClientID       string
    ClientSecret   string
    KeyProviderURL string
    Scope          string
    Audience       string
    MaxSleep       int
    WaitForTokenGen bool
}
```

### Example

Provides a function where you can pass client credentials and gofr will generate the Bearer token and will add it to the request outbound for the OAuth protected service.

```go
package main

import (
    "gofr.dev/pkg/gofr"
    "gofr.dev/pkg/service"
)

func main() {
    app := gofr.New()

    oauthOption := service.OAuthOption{
    	ClientID:       "Alice",
    	ClientSecret:   "superSecret",
    	KeyProviderURL: "https://api-sb.zopsmart.com/v1/connect/oauth2/token",
    	Scope:          "",
        Audience:       "<intended audience>",
        MaxSleep:        300,
        WaitForTokenGen: true
    }

    httpSvc := service.NewHTTPServiceWithOptions("http://localhost:9000/", k.Logger, &service.Options{Auth: &service.Auth{OAuthOption: &oauthOption}})

    sH := SampleAPIHandler{httpSvc}

    app.GET("/hello-world", sH.Get)

    app.Start()
}

type SampleAPIHandler struct {
    sampleAPIService service.HTTP
}

func (h SampleAPIHandler) Get(ctx *gofr.Context) (interface{}, error) {
    resp,err := h.sampleAPIService.Get(ctx, "hello-world", nil)
    if err != nil {
        // handle error
    }
    .
    .
    .
}
```

---

| Parameter       | Description                                                                                                                                                                                                                                                       | Example Value                 |
| --------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------- |
| ClientID        | The client ID is a unique string representing the registration information provided by the client. It is a public identifier.                                                                                                                                     | some_id                       |
| ClientSecret    | The client secret is a secret known only to the application and the authorization server.                                                                                                                                                                         | some_secret                   |
| KeyProviderURL  | It is used by the application in order to get an access token or a refresh token.                                                                                                                                                                                 | https://dummy.com/oauth/token |
| Scope           | Scope represents the resource the client requests access to. It informs the client of the scope of the access token issued.                                                                                                                                       | dummy:data                    |
| Audience        | The audience of the token is the intended recipient of the token.                                                                                                                                                                                                 | https://data.dummy.com        |
| MaxSleep        | This field helps to set the maximum sleep duration to help the service generate the access token. _Default_ _value_ - **300 seconds**                                                                                                                             | 1600                          |
| WaitForTokenGen | This is a flag to make the first call to a service blocking basis token. If this flag is set to true then the call to service will not be made until the token is generated. If false, when there is no token an error is returned. _Default_ _value_ - **false** | true                          |

---

### Adding Custom Retry Logic For Service Call

The framework provides facility to add custom retry logic to make service call based on the status code, attempt, error occurred on the service call.

```javascript
RetryCheck func(logger log.Logger, err error, statusCode int, attemptCount int) (retry bool)
```

> Example:

```javascript
func main() {
     ...

     const numOfRetries = 3

     sampleSvc := svc.NewHTTPServiceWithOptions(app.Config.Get("SAMPLE_SERVICE"), app.Logger, &svc.Options{
		NumOfRetries: numOfRetries,
	})

     sampleSvc.RetryCheck = func(logger log.Logger, err error, statusCode, attemptCount int) bool {
		if statusCode == http.StatusOK {
			return false
		}

		if err != nil {
			logger.Logf("got error %v, on attempt %v", err, attemptCount)
		}

		switch attemptCount {
		case 1:
			time.Sleep(2 * time.Second)
		case 2:
			time.Sleep(4 * time.Second)
		case 3:
			time.Sleep(8 * time.Second)
		}

		return true
    }
    .
    .
    .
}

```

## Enabling Surge Protection

For any running service, gofr provides the capability to enable the surge protector, which checks the status of the service. If it is down, the protector allows subsequent requests to it after a certain time period so that the service does not get overloaded with requests during its downtime. The surge protector ensures that the HTTP client does not bombard a downstream service that is down, it returns a 500 right away, until the service is back up again. It figures out if the service is back up again by asynchronously making requests to the heartbeat API until its up again
| Parameter | Description |
|----|----|
| `isEnabled` | Specifies if surge protection is enabled or not. <br> The default value is **true** |
| `customHeartbeatURL` | The url to which an asynchronous status checker makes requests. The statuses received is used to update `serviceStatus`. <br> The default value is **/.well-known/heartbeat** |
| `retryFrequencySeconds` | The retry frequency of the asynchronous status checker in seconds. <br> The default value is **5** seconds |

The asynchronous status checker routinely makes requests to `customHeartbeatURL` to check whether the service is up or down.

> Example
>
> For a service, the surge protector has been enabled with a retry frequency of 10 seconds.

```javascript
package main

import (
    "gofr.dev/pkg/gofr"
    "gofr.dev/pkg/service"
)

func main() {
    app := gofr.New()

    httpSvc := service.NewHTTPServiceWithOptions("http://service", app.Logger,&service.Options{SurgeProtectorOption:&service.SurgeProtectorOption{
    	HeartbeatURL:   "/heartbeat",
    	RetryFrequency: 10,
    }})

    sH := SampleAPIHandler{httpSvc}

    app.GET("/hello-world", sH.Get)

    app.Start()
}

type SampleAPIHandler struct {
    sampleAPIService service.HTTP
}

func (h SampleAPIHandler) Get(c *gofr.Context) (interface{}, error) {
    return h.sampleAPIService.Get(c, "", nil), nil
}
```

The status of the service is then checked by checking the heartbeat specified by the URL. If a status code of 500 is returned indicating that the service was down the last time the asynchronous checker sent a request.

## Facts

- HTTP Service does not support exponential back-off for retry intervals yet.
- One can specify the retry interval.
  ```go
  svc:= svc.NewHTTPServiceWithOptions("http://dummy", log.NewLogger(), nil)
  svc.Client.Timeout = 10  // This will set retry duration to 10 seconds.
  ```
- HTTP Service does not support to set retry for specific HTTP status codes.
- If the `heartbeat` endpoint is not available then for request to any API endpoint, response will be service down. but one can set the heartbeat URL as specified above.