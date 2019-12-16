+++ 
date = 2019-12-16T15:29:34-05:00
title = "Basic Authentication in Go with Gin"
description = "Basic user authentication with gin web framework in Golang"
tags = ["golang", "basicauth","gin"]
categories = []
externalLink = ""
series = []
+++

This is short post on adding basic authentication to go applications. Our sample application uses gin web framework

Let's start by creating a gin router with default middleware, by default it serves on `:8080` unless a `PORT` environment variable was defined

```go
func main(){
	r := gin.Default()
	r.GET("/getAllUsers", basicAuth, handlers.UsersList)
	_ = r.Run()
}
```

Now that we have our basic route, lets create a method to add authentication logic. Get basic auth credentials from context request and validate them. If user isn't authenticated, authentication window is prompted with username and password.

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
$ go run main.go
```

Verify if authentication works

```bash
$ curl -X GET "http://testuser:testpass@localhost:8080/getAllUsers"
```
