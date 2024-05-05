---
    title: Benchmarking SSL Terminator
---

The benchmark of SSL Terminator will depend on TLS overhead, and so in this article we will perform tests on TLS performance and try to run benchmarks. SSL/TLS termination can act as a reverse proxy, and offload the SSL/TLS decryption process from the server. I wrote a simple code in go that terminates SSL and passes the request to the main server.

SSL Performance is benchmarked on Handshakes/sec and tps [[1](https://www.haproxy.com/blog/benchmarking-ssl-performance)]. As per [[2](https://www.imperialviolet.org/2010/06/25/overclocking-ssl.html)], SSL/TLS accounts for less than 1% of the CPU load, less than 10KB of memory per connection and less than 2% of network overhead. It seems pretty more impressive that there are multiple performance optimization techniques today which further reduces tls overhead. 

As per the openssl benchmarks for ecdsa,

```
    version: 3.1.3
    built on: Tue Sep 19 13:01:49 2023 UTC
    options: bn(64,64)
    compiler: clang -fPIC -arch arm64 -O3 -Wall -DL_ENDIAN -DOPENSSL_PIC -D_REENTRANT -DOPENSSL_BUILDING_OPENSSL -DNDEBUG
    CPUINFO: OPENSSL_armcap=0x187d
                              sign    verify    sign/s verify/s
    256 bits ecdsa (brainpoolP256r1)   0.0002s   0.0002s   6020.5   5880.1
    256 bits ecdsa (brainpoolP256t1)   0.0002s   0.0002s   6026.2   6208.9
```

The benchmarks I got from the ssl performance of go are,

```
    go test -bench . -benchmem
    goos: darwin
    goarch: arm64
    pkg: main/sslterminator
    BenchmarkUsingECDSAKeys-8             82          14295354 ns/op           95724 B/op            696 allocs/op
    testing: BenchmarkUsingECDSAKeys-8 left GOMAXPROCS set to 1
    BenchmarkUsingRSAKeys-8               81          14185131 ns/op           95733 B/op            696 allocs/op
    testing: BenchmarkUsingRSAKeys-8 left GOMAXPROCS set to 1
    PASS
    ok      main/sslterminator      3.142s
```

Our results show that, number of requests completed per sec is ~81. This is surprisingly low compared to what we got from OpenSSL benchmarks. This prompts us to look closely at our own benchmarks. 
The results are obtained for a single thread execution ( limited by GOMAXPROCS(1) function ). The following is the Go version used

```
    go version
    go version go1.21.3 darwin/arm64
```

The benchmark test `BenchmarkUsingECDSAKeys` handles the client execution. A client sends a request to the server which listens of `PORT :443`. The listener accepts the request and returns a connection var. In the next step, we initiate a handshake.

If we calculate the time for Handshake func to execute, we get the following results.

```
go test -bench . -benchmem
    873µs time taken for handshake
    971.833µs time taken for handshake
    954.292µs time taken for handshake
    915µs time taken for handshake
    848.916µs time taken for handshake
        76          15383503 ns/op           95747 B/op        698 allocs/op
    testing: BenchmarkUsingECDSAKeys-8 left GOMAXPROCS set to 1
```

The results obtained for the execution of Handshake are ~0.9ms which is close to the openSSL benchmarks of 0.2ms.

Remember, any operation here does not affect the load on the downstream servers. SSL/TLS termination has been completely offloaded to a proxy server exposed directly to the client. Any further optimiZations we do, improve the overall latency of the request and do not affect the downstream servers.

Let's look at the benchmarks from the official go benchmarks [[https://go-review.googlesource.com/c/go/+/44730/4](3)]. 

```
    crypto/tls: add BenchmarkHandshakeServer

    name                                       time/op
    HandshakeServer/RSA-4                      1.10ms ± 0%
    HandshakeServer/ECDHE-P256-RSA-4           1.23ms ± 1%
    HandshakeServer/ECDHE-P256-ECDSA-P256-4     178µs ± 1%
    HandshakeServer/ECDHE-X25519-ECDSA-P256-4   180µs ± 2%
    HandshakeServer/ECDHE-P521-ECDSA-P521-4    19.8ms ± 1%
```
This suggests something is wrong, our time calculated for Handshake is 5-6x slower. What I want to do next is isolate the Handshake function and let Go do the benchmarking instead of me using time.Sub to know what is wrong.
What I've found is `listener.Aceept()` function takes much of the time mentioned, however, `Handshake()` function does not take that much time

For this I have been studying the [https://go.dev/src/crypto/tls/handshake_server_test.go](https://go.dev/src/crypto/tls/handshake_server_test.go) file. Will post a complete analysis of this and add some of my own testing on a later blog.

