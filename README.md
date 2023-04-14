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
		buf := bytes.NewBuffer([]byte("hello world"))
		buf.Write(tran.EndIdentity)
		buf.Write(tran.LFBytes)
		client.Write(buf.Bytes())
	})

	// start client
	client := tran.NewClient(ip, port, true, certFile, false)
	err = client.Connect()
	if err != nil {
		log.Error(err, "tcp client connect to tcp server error")
		return
	}
	defer client.Close()

	// communication
	client.Write([]byte("bye~"))
	data, err := client.ReadAll()
	if err != nil {
		log.Error(err, "tcp client read server data error")
	} else {
		log.Info("receive message from server => %s", string(data))
	}
}
```
