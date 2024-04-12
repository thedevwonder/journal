---
    title: Benchmarking SSL Terminator
    draft: true
---

SSL/TLS termination can act as a reverse proxy, and offload the SSL/TLS decryption process from the server. I wrote a simple code in go that terminates SSL and passes the request to the server and I wanted to benchmark it.
From the resources mentioned 
