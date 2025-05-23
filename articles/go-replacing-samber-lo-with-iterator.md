---
title: "[Go]github.com/samber/loをiteratorで置き換える"
emoji: "📦"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["go"]
published: false
---

## [Go]github.com/samber/loをiteratorで置き換える

<!-- other languages referenced -->

[Java]: https://www.java.com/
[TypeScript]: https://www.typescriptlang.org/
[python]: https://www.python.org/
[C]: https://www.c-language.org/
[C++]: https://isocpp.org/
[Rust]: https://www.rust-lang.org
[The Rust Programming Language 日本語]: https://doc.rust-jp.rs/book-ja/
[Lua]: https://www.lua.org/

<!-- other lib/SDKs referenced -->

[Node.js]: https://nodejs.org/en
[deno]: https://deno.com/
[tokio]: https://tokio.rs/

<!-- editors -->

[Visual Studio Code]: https://code.visualstudio.com/
[vscode]: https://code.visualstudio.com/
[neovim]: https://neovim.io/

<!-- tools -->

[git]: https://git-scm.com/
[Git Credential Manager]: https://github.com/git-ecosystem/git-credential-manager?tab=readme-ov-file
[Docker]: https://www.docker.com/
[Dockerfile]: https://docs.docker.com/build/concepts/dockerfile/
[Elasticsearch]: https://www.elastic.co/docs/solutions/search

<!-- RFCs -->

[RFC 6901]: https://datatracker.ietf.org/doc/html/rfc6901

<!-- Go versions -->

[Go]: https://go.dev/
[Go 1.11]: https://go.dev/doc/go1.11
[Go 1.14]: https://go.dev/doc/go1.14
[Go 1.18]: https://go.dev/doc/go1.18
[Go 1.21]: https://go.dev/doc/go1.21
[Go 1.22]: https://go.dev/doc/go1.22
[Go 1.23]: https://go.dev/doc/go1.23
[Go 1.24]: https://go.dev/doc/go1.24

<!-- Go doc links -->

[A Tour of Go]: https://go.dev/tour/welcome/
[GOAUTH]: https://pkg.go.dev/cmd/go#hdr-GOAUTH_environment_variable

<!-- Go tools -->

[gotip]: https://pkg.go.dev/golang.org/dl/gotip
[gopls]: https://github.com/golang/tools/tree/master/gopls
[github.com/go-task/task]: https://github.com/go-task/task

<!-- references to spec -->

[type assertion]: https://go.dev/ref/spec#Type_assertions
[type switch]: https://go.dev/ref/spec#Type_switches

<!-- references to sdk library -->

[panic]: https://pkg.go.dev/builtin@go1.24.3#panic
[*json.Decoder]: https://pkg.go.dev/encoding/json@go1.24.3#Decoder
[*json.Decoder.More]: https://pkg.go.dev/encoding/json@go1.24.3#Decoder.More
[*json.Encoder]: https://pkg.go.dev/encoding/json@go1.24.3#Encoder
[json.Marshal]: https://pkg.go.dev/encoding/json@go1.24.3#Marshal
[json.Marshaler]: https://pkg.go.dev/encoding/json@go1.24.3#Marshaler
[json.MarshalIndent]: https://pkg.go.dev/encoding/json@go1.24.3#MarshalIndent
[json.Unmarshaler]: https://pkg.go.dev/encoding/json@go1.24.3#Unmarshaler
[json.Unmarshal]: https://pkg.go.dev/encoding/json@go1.24.3#Unmarshal
[xml.NewTokenDecoder]: https://pkg.go.dev/encoding/xml@go1.24.3#NewTokenDecoder
[errors.New]: https://pkg.go.dev/errors@go1.24.3#New
[errors.Is]: https://pkg.go.dev/errors@go1.24.3#Is
[errors.As]: https://pkg.go.dev/errors@go1.24.3#As
[errors.Join]: https://pkg.go.dev/errors@go1.24.3#Join
[fmt.Errorf]: https://pkg.go.dev/fmt@go1.24.3#Errorf
[fs.ErrNotExist]: https://pkg.go.dev/io/fs@go1.24.3#ErrNotExist
[http.Server]: https://pkg.go.dev/net/http@go1.24.3#Server
[*http.Server]: https://pkg.go.dev/net/http@go1.24.3#Server
[io.EOF]: https://pkg.go.dev/io@go1.24.3#EOF
[io.MultiWriter]: https://pkg.go.dev/io@go1.24.3#MultiWriter
[io.Pipe]: https://pkg.go.dev/io@go1.24.3#Pipe
[io.Reader]: https://pkg.go.dev/io@go1.24.3#Reader
[io.Writer]: https://pkg.go.dev/io@go1.24.3#Writer
[syscall.Errno]: https://pkg.go.dev/syscall@go1.24.3#Errno
[text/template]: https://pkg.go.dev/text/template@go1.24.3
