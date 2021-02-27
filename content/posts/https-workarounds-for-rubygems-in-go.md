---
title: "HTTPS Workarounds for Rubygems in Go"
date: 2021-02-27T20:24:51Z
draft: false
---

Sometimes you have a really old application which uses an old version of jRuby and can no longer talk to rubygems.org over HTTPS. In ~Dec 2020, Rubygems or Fastly removed TLSv1.1 support. I don't blame them but it caused a couple issues installing gems on old software.

You might see one of these sorts of errors:

```shell
Error fetching https://rubygems.org/:
	Received fatal alert: handshake_failure (https://api.rubygems.org/specs.4.8.gz)

ERROR:  While executing gem ... (Gem::RemoteFetcher::FetchError)
    Received fatal alert: handshake_failure (https://api.rubygems.org/api/v1/dependencies)
```

Below is Go program that is a proxy server which upgrades `http://` requests to `https://`. Simples right?

By default if you change the gem sources to `http://rubygems.org`, you might get the second error as they 301 redirect you to the secure version.

This is where this proxy comes in.


## How to use

1. Remove the https version of rubygems from sources: `gem sources --remove https://rubygems.org/`
2. Add the http version of rubygems: `gem sources --add http://rubygems.org/`
3. Build proxy server: `go build -o rubygems-proxy`
4. Run the proxy server: `./rubygems-proxy`. If you're lazy, you can build/run at the same time: `go run proxy.go`
5. Install your gems: `gem install --http-proxy  http://localhost:8989 vault`
6. All should work, fingers crossed. There's some logging in the proxy to see the requests going by.


## Notes

* I'm very new to Go, this was made using StackOverflow Driven Development 
* It's a workaround, this proxy shouldn't be used for anything more than getting out of a pinch
* Keep your software updated and you should't have to figure out these workarounds


## The source

```go
package main

import (
	"fmt"
	"log"
	"net/http"
	"net/http/httputil"
)

func main() {
	proxy := NewHTTPSUpgradeReverseProxy()
	fmt.Println("Awake and listening on :8989")

	http.HandleFunc("/", func(w http.ResponseWriter, req *http.Request) {
		log.Printf("%#v", req.URL)

		proxy.ServeHTTP(w, req)
	})

	err := http.ListenAndServe(":8989", nil)
	if err != nil {
		panic(err)
	}
}

// NewHTTPSUpgradeReverseProxy upgrades http requests to https
func NewHTTPSUpgradeReverseProxy() *httputil.ReverseProxy {
	director := func(req *http.Request) {
		req.URL.Scheme = "https" // Where the magic happens
	}
	return &httputil.ReverseProxy{Director: director}
}
```

