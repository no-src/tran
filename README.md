# tran

[![Build](https://img.shields.io/github/actions/workflow/status/no-src/tran/go.yml?branch=main)](https://github.com/no-src/tran/actions)
[![License](https://img.shields.io/github/license/no-src/tran)](https://github.com/no-src/tran/blob/main/LICENSE)
[![Go Reference](https://pkg.go.dev/badge/github.com/no-src/tran.svg)](https://pkg.go.dev/github.com/no-src/tran)
[![Go Report Card](https://goreportcard.com/badge/github.com/no-src/tran)](https://goreportcard.com/report/github.com/no-src/tran)
[![codecov](https://codecov.io/gh/no-src/tran/branch/main/graph/badge.svg?token=cp1R37aqHt)](https://codecov.io/gh/no-src/tran)

A simple tcp transport server and client component.

## Installation

```bash
go get -u github.com/no-src/tran
```

## Quick Start

```go
package main

import (
	"bytes"
	"time"

	"github.com/no-src/gofs/auth"
	"github.com/no-src/gofs/report"
	"github.com/no-src/log"
	"github.com/no-src/tran"
)

func main() {
	ip := "127.0.0.1"
	port := 6060
	certFile := "./testdata/cert.pem"
	keyFile := "./testdata/key.pem"
	users, err := auth.RandomUser(1, 8, 8, auth.FullPerm)
	if err != nil {
		log.Error(err, "generate random user error")
		return
	}

	// start server
	server := tran.NewServer("127.0.0.1", port, true, certFile, keyFile, users, report.NewReporter())
	err = server.Listen()
	if err != nil {
		log.Error(err, "Listen: the tcp server listen error")
		return
	}
	defer server.Close()

	go server.Accept(func(client *tran.Conn, data []byte) {
		// parse client auth package and authorize
		authPkg := bytes.TrimRight(data, string(tran.EndIdentity))
		log.Info("receive message from client => %s", string(authPkg))
		if bytes.Equal(authPkg, []byte("auth package mock")) {
			user := users[0].ToHashUser()
			ok, _ := server.Auth(user)
			if ok {
				client.MarkAuthorized(user)
			}
		}
	})

	go func() {
		time.Sleep(time.Second * 3)
		err = server.Send([]byte("hello world"))
		if err != nil {
			log.Error(err, "send message to clients error")
		}
	}()

	// start client
	client := tran.NewClient(ip, port, true, certFile, false)
	err = client.Connect()
	if err != nil {
		log.Error(err, "tcp client connect to tcp server error")
		return
	}
	defer client.Close()

	// send custom auth package
	client.Write([]byte("auth package mock"))
	data, err := client.ReadAll()
	if err != nil {
		log.Error(err, "tcp client read server data error")
	} else {
		log.Info("receive message from server => %s", string(data))
	}
}
```
