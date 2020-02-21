+++
date = "2018-09-24T18:45:58+01:00"
draft = false
title = "Write go like a pro"
image = "Go-Logo_Black.png"
toc = true
tags        = [ "go", "makefile", "logging"]
+++


So you enjoy go and are ready to take your go-application to production standards. Here is a listing of some tricks on how to get your application production-ready. Not all these advices will apply to all kinds of applications, pick and choose whatever makes sense to your application.



# Use Make to build and test your app

Heavy-weight applications written in heavyweigh JVM languages has gradle/maven/ant/sbt/leiningen. Javascriptish languages has npm/yarn/hotoftheday. For go, you should lean on a rock-solid lightweight tool like `Make`, to build and test your app.

To have stable applications and enable reproducible builds the go-toolchain might fall short. If you are lucky your app still builds with the `go build` command, but you might want to run your tests with additional flags and clean up after building and testing. 

`Make` is the perfect tool for this task. A simple `Makefile` can be only a couple of lines and easy to read. `Make` has support for incremental builds, so when your build-process grows, make is there to support you. 

Start out with a simple makefile and let it grow to fit your needs. Your makefile should at least have the targets

* *build* - compile your application into a production-ready binary. Build should always depend on the test-target to make sure we ship a sane binary.
* *test* - run all your tests
* *clean* - clean up binaries produced by the build-target and output from your tests 
* some form of *lint*-check to check that your code conforms to your chosen standards. It is recommended to at least conform to golang formatting standards.

_Semi-advanced Makefile that considers vendor-folder, formatting, lint and strict checks_ :

     SHELL := /bin/bash
     
     # The name of the executable (default is current directory name)
     TARGET := $(shell echo $${PWD\#\#*/})
     .DEFAULT_GOAL: $(TARGET)
     
     # These will be provided to the target
     VERSION := 1.0.0
     BUILD := `git rev-parse HEAD`
     
     # Use linker flags to provide version/build settings to the target
     LDFLAGS=-ldflags "-X=main.Version=$(VERSION) -X=main.Build=$(BUILD)"
     
     # go source files, ignore vendor directory
     SRC = $(shell find . -type f -name '*.go' -not -path "./vendor/*")
     
     .PHONY: all build clean install uninstall fmt simplify check run
     
     all: test-all install
     
     $(TARGET): $(SRC)
     	go build $(LDFLAGS) -o $(TARGET)
     
     build: test $(TARGET)
     	@true
     
     clean:
     	rm -f $(TARGET)
     	rm *.png
     
     fmt:
     	gofmt -l -w $(SRC)
     
     test:
     	go test -short ./...
     
     lint:
     	go vet ./...
     
     test-all: lint test
     	go test -race ./...
     
     strict-check:
     	@test -z $(shell gofmt -l main.go | tee /dev/stderr) || echo "[WARN] Fix formatting issues with 'make fmt'"
     	@for d in $$(go list ./... | grep -v /vendor/); do golint $${d}; done
     	@go tool vet ${SRC}
     
     run: test-all install
     	@$(TARGET) 
     	
This Makefile supports several of the mentioned tasks. It is taken from one of my projects on github, [grib](https://github.com/nilsmagnus/grib).

# Control your dependencies

Using `go get` to install libraries is convenient, but that alone is not a good long-term solution. Versions of your dependencies change and in worst case will break your build on updates. To have stable build over time you do need a tool to manage your dependencies and versioning of your dependencies. Up until recently I would have recommended [dep](https://golang.github.io/dep/) to manage your dependencies, but after Brian Ketelsen introduced [vgo](https://github.com/golang/vgo), I am not sure which one of the two to recommend. Both supports semantic versioning and revision versioning of your dependencies. 

* `dep` is probably more mature and better tested at the moment.
* `vgo` is probably the future choice of the golang community.

When using `dep`, I would seriously consider adding your vendor-folder to your VCS to speed up builds and be independent of external source-control-providers(github/gitlab/bitbucket etc.) when building on new machines or in a CI-environment. 

# Build your projects on a CI-server

This is probably obvious, but I include it anyway: make sure you build your code on a ci-server. Building on your machine is not sufficient, since your machine might have some special environment variation. Building on a ci-server reveals any custom environment requirements and will allow new programmers adapt your code quickly.

If you dont have a ci-server yourself, check out the free ci-pipelines at [gitlab.com](https://gitlab.com/) or other providers. 

# Test continuously, write testable code

### Run tests on save 
One of the things we love with go is the blazingly fast compiler and fast execution. So why not make use of it and run the tests every time you save your code? Vscode, idea and other editors has support for save-actions. Add a save-action for running your tests with coverage and see how it fits you. Try to only run unit-tests this way, since integration-tests with databases and external services might take some time to run. 

### Testing libraries
The built in testing-framework in golang should cover your most needs for testing, so there is no need to include testing-libraries. But should you need something extra, other libraries can make your tests a bit more readable and specialized. Popular libraries include testify and httpexpect.

* [Built in testing package](https://golang.org/pkg/testing/)
* [testify](https://github.com/stretchr/testify)
* [httpexpect](https://github.com/gavv/httpexpect)

### Use interfaces to enable cleaner tests

If you use interfaces when programming, your testing-life becomes way lot easier. For instance, instead of passing a io.File as an argument, you might find that you can pass io.Reader or io.Writer in your implementation. This will allow writing a lot simpler tests. Nathan LeClaire wrote a nice post on testing with interfaces [here](https://nathanleclaire.com/blog/2015/10/10/interfaces-and-composition-for-effective-unit-testing-in-golang/) that you might find useful.


# Error handling - Don't Panic.

Don't panic. You should never use panic in your production code if you want to keep your application running. It will result in unpredictable behaviour and probably crash your application. Instead, handle your errors gracefully. You could of course [recover from a panic](https://blog.golang.org/defer-panic-and-recover), but it is messy and not readable. Instead, always check error-returns and handle them gracefully.

*Don't panic* is one of the [golang proverbs](https://go-proverbs.github.io/) for a reason.

# Logging 

The built in logging library is not sufficient for a production ready application. It has little support for log-levels and does not output logger-name and timestamp by default. There are some seriously good logging-libraries for golang, so you can pick and choose the best for you. 

I recommend using über's library [zap](https://github.com/uber-go/zap)  or [logrus](https://github.com/sirupsen/logrus). These are both good loggers that will serve you well. 

Zap is a high-performant logger "4-10x" faster than other loggers, and provides a structured logger. 

_Sample output from zap sugared logger_:
  
    2018-09-20T10:21:56.574+0200	INFO	ftplistener/main.go:93	hit ftpFolder 	{"foldername": "gfs.2018092000"}
    2018-09-20T10:22:00.383+0200	INFO	ftplistener/main.go:93	hit ftpFolder 	{"foldername": "gfs.2018091800"}
    2018-09-20T10:22:00.388+0200	INFO	ftplistener/main.go:148	Downloading 	{"file": "/media/larsgard/4TBWEATHER/gfs.2018/gfs.2018092000/gfs.t00z.pgrb2.1p00.f000"}
    2018-09-20T10:22:00.388+0200	INFO	ftplistener/main.go:148	Downloading 	{"file": "/media/larsgard/4TBWEATHER/gfs.2018/gfs.2018092000/gfs.t00z.pgrb2.1p00.f384"}
    2018-09-20T10:22:00.389+0200	INFO	ftplistener/main.go:148	Downloading 	{"file": "/media/larsgard/4TBWEATHER/gfs.2018/gfs.2018092000/gfs.t00z.pgrb2.1p00.f063"}


# Metrics

Exposing basic metrics from a golang-web-app is straight forward when using the built-in `expvar` package. This package allows you to add metrics of your choice. The metrics are kept in-memory so you probably want an scraper to read the metrics from a http.Handler. Have a look at the [expvar](https://golang.org/pkg/expvar/) package documentation.

* [Datadog](https://docs.datadoghq.com/integrations/go_expvar/) uses the expvar feature to monitor your app. 
* There are available libraries for [prometheus](https://github.com/prometheus/client_golang) if you are using prometheus for other applications already. 
 
_Using the expvar-package for custom metrics:_

    import "expvar" 
    
    var (
      requestConter = *expvar.Int
    )
    
    func init() {
      requestConter = expvar.NewInt("requests")
    }
    
    func handleRequest(){
    // Adds an integer value to expvar.Int counter
        requestConter.Add(1)
    }
    
# Read config from flags AND environment

You probably need to config your application in some way. Ports, urls, usernames and passwords are a good idea to read from configs. 

Config parameters should be possible to set with either env-variables, with the [flag](https://golang.org/pkg/flag/) package or , more advanced, config-files.

* Lars Wiegman's library, [namsral/flag](https://github.com/namsral/flag), reads config from both environment, cli-flags and config files.
* Kelsey Hightower's popular library [envconfig](https://github.com/kelseyhightower/envconfig) is a neat library to read config from environment to structs.  

