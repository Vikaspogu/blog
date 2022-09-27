+++ 
date = 2019-12-16
title = "Basic Authentication in Go with Gin"
description = "Basic user authentication with gin web framework in Golang"
tags = ["golang", "basic-auth","gin"]
categories = []
externalLink = ""
series = []
socialShare=true
+++

This short post looks at adding basic authentication to GoLang applications. Below example application uses the gin web framework.

Let's start by creating a gin router with default middleware. By default, it serves on `:8080` unless we define a `PORT` environment variable.

```go
func main(){
    r := gin.Default()
    r.GET("/getAllUsers", basicAuth, handlers.UsersList)
    _ = r.Run()
}
```

Now that we have our primary route let us create a method to add authentication logic. First, get basic auth credentials from the context request and validate them. The browser will prompt an authentication window for username and password details if the user is not authenticated.

```go
func basicAuth(c *gin.Context) {
    // Get the Basic Authentication credentials
    user, password, hasAuth := c.Request.BasicAuth()
    if hasAuth && user == "testuser" && password == "testpass" {
        log.WithFields(log.Fields{
            "user": user,
        }).Info("User authenticated")
    } else {
        c.Abort()
        c.Writer.Header().Set("WWW-Authenticate", "Basic realm=Restricted")
        return
    }
}
```

Run application

```bash
go run main.go
```

Verify if authentication works

```bash
curl -X GET "http://testuser:testpass@localhost:8080/getAllUsers"
```
