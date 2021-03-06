---
title: HTTP-Proxy
date: 2020-04-14T10:09:14.246627+08:00
draft: false
---

## 背景

如果您向客户提供银行服务，则可能必须与第三方（例如其他银行和支付提供商）集成。为了保证高级别的安全性，参与方通常要求：

1. 对通信通道进行加密
2. 限制目的地和来源

对于Web服务：

1. 可通过使用具有传输层安全性（TLS）或虚拟专用网络（VPN）连接的HTTPS来解决加密问题。
2. 目的地限制是由防火墙和负载平衡器通过端口和数据包筛选来强制实施的。
3. 来源限制是由静态IP地址（例如，亚马逊网络服务（AWS）提供的弹性IP（EIP）或Google Cloud Platform（GCP）提供的静态外部IP地址）实施的。如果您的服务已经在这些云提供商的范围内运行，则只需将集成服务连接到这些静态IP。

> 但是，如果您的服务托管在平台即服务（PaaS）提供程序（例如Heroku）上，则必须考虑其他集成工作，以满足起源限制。这是HTTPS代理用来提供封装的。

## HTTP/S 代理是什么

代理是一种服务器或应用程序，可充当从客户端到目的地请求的中介。HTTP/S代理是支持HTTP请求转发（forwarding）和HTTP隧道（tunneling）的代理。它通常在开放系统互连（OSI）第7层上运行，这与通常在OSI第3层上运行的网络地址转换（NAT）应用程序区分开来。像HTTP隧道这种代理，有很多备用名称，例如网关（gateway）或透明代理（transparent proxy）。要代理HTTP/S请求，可以在两种类型之间进行广泛选择：转发代理（Forwarding Proxy）和反向代理（Reverse Proxy）。

### 转发代理

转发代理可以有两种变体。

#### 复制

一种变体称为复制（replication），其中代理端终止传入的客户端请求，创建自己的到目的地的传出请求，然后以目的地回复来响应客户端。这适用于HTTP，但不适用于HTTPS，因为它会附带一个重要的副作用，即能够检查（和修改）客户端的请求，这会破坏TLS完整性。

![image](/images/forwarding-proxy.png)

#### 隧道

另一个变种称为隧道（tunneling），其中代理端通过HTTP CONNECT方法使用HTTP隧道功能。这样，代理端就接受了一个初始CONNECT请求，该请求将整个URL当作HOST值，而不仅仅是主机地址。然后，代理端打开到目的地的TCP连接，并透明地将原始通信从客户端转发到目的地。这允许代理端仅检查和评估来自客户端的初始CONNECT请求，例如授权。但是，任何进一步的通信都不会被拦截，因此代理端无法读取TLS通信。这允许初始的连接代理端的CONNECT方法是HTTP或HTTPS，然后是通过代理端的HTTPS到目的地。

HTTP会话示例

```h
> CONNECT example.host.com:443 HTTP/1.1
> Host: example.host.com:443
> Proxy-Authorization: Basic base64-encoded-proxy-credentials
> Proxy-Connection: Keep-Alive
< HTTP/1.1 200 OK

> GET /foo/bar?baz#qux HTTP/1.1
> Host: example.host.com
> Authorization: Basic base64-encoded-destination-credentials
< HTTP/1.1 200 OK
< Connection: close
```

![image](/images/tunneling-proxy.png)

在这两种变体中，客户端都会直接影响目的地的选择，因为涉及两个不同的URL：代理端的URL和目的地的URL。

### 反向代理

在计算机网络中，反向代理是一种代理服务器，它代表一个客户端从一个或多个服务器检索资源。然后，将这些资源返回给客户端，就好像它们源自Web服务器本身一样。

反向代理接受来自客户端的传入请求，并根据该请求将它们路由到特定的目的地。目的地选择的区分因素可能是请求的主机，路径，查询参数，任何标头，甚至是请求体的有效负载。无论如何，反向代理都会拦截并终止HTTP和HTTPS连接，并向目的地创建新请求。因此，在使用反向代理时，客户端不会直接影响目的地选择。

## 小结

现在我们知道了不同的​​HTTP/S代理类型和变体，可以评估我们的要求并选择适当的代理实现：如果希望完全控制目的地的选择，那么选择转发代理，并且要保证高度的隐私和完整性，因此选择HTTP隧道。

## 示例

Golang的`net/http` 包中有客户端-服务器通信所需的大多数实现。您需要做的就是了解功能，并在需要时使用它。

### 转发代理示例

#### 生成证书

```bash
#!/usr/bin/env bash
case `uname -s` in
    Linux*)     sslConfig=/etc/ssl/openssl.cnf;;
    Darwin*)    sslConfig=/System/Library/OpenSSL/openssl.cnf;;
esac
openssl req \
    -newkey rsa:2048 \
    -x509 \
    -nodes \
    -keyout server.key \
    -new \
    -out server.pem \
    -subj /CN=localhost \
    -reqexts SAN \
    -extensions SAN \
    -config <(cat $sslConfig \
        <(printf '[SAN]\nsubjectAltName=DNS:localhost')) \
    -sha256 \
    -days 3650
```

#### 代理程序

```go
package main
import (
    "crypto/tls"
    "flag"
    "io"
    "log"
    "net"
    "net/http"
    "time"
)

func handleTunneling(w http.ResponseWriter, r *http.Request) {
    dest_conn, err := net.DialTimeout("tcp", r.Host, 10*time.Second)
    if err != nil {
        http.Error(w, err.Error(), http.StatusServiceUnavailable)
        return
    }
    w.WriteHeader(http.StatusOK)
    hijacker, ok := w.(http.Hijacker)
    if !ok {
        http.Error(w, "Hijacking not supported", http.StatusInternalServerError)
        return
    }
    client_conn, _, err := hijacker.Hijack()
    if err != nil {
        http.Error(w, err.Error(), http.StatusServiceUnavailable)
    }

    go transfer(dest_conn, client_conn)
    go transfer(client_conn, dest_conn)
}

func transfer(destination io.WriteCloser, source io.ReadCloser) {
    defer destination.Close()
    defer source.Close()
    io.Copy(destination, source)
}

func handleHTTP(w http.ResponseWriter, req *http.Request) {
    resp, err := http.DefaultTransport.RoundTrip(req)
    if err != nil {
        http.Error(w, err.Error(), http.StatusServiceUnavailable)
        return
    }
    defer resp.Body.Close()
    copyHeader(w.Header(), resp.Header)
    w.WriteHeader(resp.StatusCode)
    io.Copy(w, resp.Body)
}

func copyHeader(dst, src http.Header) {
    for k, vv := range src {
        for _, v := range vv {
            dst.Add(k, v)
        }
    }
}

func main() {
    var pemPath string
    flag.StringVar(&pemPath, "pem", "server.pem", "path to pem file")
    var keyPath string
    flag.StringVar(&keyPath, "key", "server.key", "path to key file")
    var proto string
    flag.StringVar(&proto, "proto", "https", "Proxy protocol (http or https)")
    flag.Parse()
    if proto != "http" && proto != "https" {
        log.Fatal("Protocol must be either http or https")
    }

    server := &http.Server{
        Addr: ":8888",
        Handler: http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            if r.Method == http.MethodConnect {
                handleTunneling(w, r)
            } else {
                handleHTTP(w, r)
            }
        }),
        // Disable HTTP/2.
        TLSNextProto: make(map[string]func(*http.Server, *tls.Conn, http.Handler)),
    }

    if proto == "http" {
        log.Fatal(server.ListenAndServe())
    } else {
        log.Fatal(server.ListenAndServeTLS(pemPath, keyPath))
    }
}

//  升级版

package main
import (
    "crypto/tls"
    "fmt"
    "net/http"
    "net/http/httputil"
    "net/url"
)
func main() {
    u, err := url.Parse("https://localhost:8888")
    if err != nil {
        panic(err)
    }
    tr := &http.Transport{
        Proxy: http.ProxyURL(u),
        // Disable HTTP/2.
        TLSNextProto: make(map[string]func(authority string, c *tls.Conn) http.RoundTripper),
    }
    client := &http.Client{Transport: tr}
    resp, err := client.Get("https://google.com")
    if err != nil {
        panic(err)
    }
    defer resp.Body.Close()
    dump, err := httputil.DumpResponse(resp, true)
    if err != nil {
        panic(err)
    }
    fmt.Printf("%q", dump)
}
```

#### 测试

```sh
Chrome --proxy-server=https://localhost:8888

curl -Lv --proxy https://localhost:8888 --proxy-cacert server.pem https://google.com
```

### 反向代理示例一

已经有singleHostReverseProxy函数，在httputil包中。

```go
func NewSingleHostReverseProxy(target *url.URL) *ReverseProxy
```

我们只需要发送一个URL到返回reverseProxy的函数即可。

```go
func (p *ReverseProxy) ServeHTTP(rw http.ResponseWriter, req *http.Request)
```

然后，如果我们调用ServeHTTP函数，它将http请求发送到适当的服务。

#### 用例

假设有一个可通过某个端口的http访问的服务，但是不希望公开该服务，或者想要添加一些自定义规则，或者想要在界面上进行一些性能分析，则可以使用反向代理。

在此示例中，我们将通过将所有Web请求从一台服务器（运行在任意的9090端口上）转发到某处的另一台服务器来演示反向代理的作用，例如 `<http://127.0.0.1:8080>` ：

- 反向代理golang服务器将在9090端口上运行。
- 对服务器的所有请求将透明地转发到目标Web服务器，并将响应发送到第一台服务器。
- 在转发请求之前，将捕获请求正文。借助`ioutil`包的ReadAll和NopCloser函数，该函数有助于在不修改请求缓冲区的情况下复制请求正文。
- 还捕获了原始Web服务器为每个路径提供请求所花费的时间。

#### 路由流量

创建一个称为Prox的简单结构，它将处理反向代理的业务逻辑。

```go
type Prox struct {
    target *url.URL
    proxy  *httputil.ReverseProxy
}

func NewProxy(target string) *Prox {
    url, _ := url.Parse(target)
    return &Prox{
        target: url,
        proxy: httputil.NewSingleHostReverseProxy(url)
        }
}
```

如您所见，没有太多代码。我们只需要发送目标url，NewProxy函数将返回Prox结构对象。

```go
func (p *Prox) handle(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("X-GoProxy", "GoProxy")
    p.proxy.Transport = &myTransport{}
    p.proxy.ServeHTTP(w, r)
}
```

由于我们要捕获请求/响应主体或为每个请求添加一些自定义规则，因此我们不得不更改传输接口。因此，我们为Transport创建了自己的RoundTrip函数并将其分配给我们之前创建的`proxy.Transport`。

```go
type myTransport struct {
}


func (t *myTransport) RoundTrip(request *http.Request) (*http.Response, error) {
    buf, _ := ioutil.ReadAll(request.Body)
    rdr1 := ioutil.NopCloser(bytes.NewBuffer(buf))
    rdr2 := ioutil.NopCloser(bytes.NewBuffer(buf))

    fmt.Println("Request body : ", rdr1)
    request.Body = rdr2

    response, err := http.DefaultTransport.RoundTrip(request)
    if err != nil {
        print("\n\ncame in error resp here", err)
        return nil, err //Server is not reachable. Server not working
    }

    body, err := httputil.DumpResponse(response, true)
    if err != nil {
        print("\n\nerror in dumb response")
        // copying the response body did not work
        return nil, err
    }

    log.Println("Response Body : ", string(body))
    return response, err
}
```

借助Go的time包，我们可以测量函数所花费的时间，在这种情况下，我们可以测量原始服务器的响应时间。我们修改了RoundTrip函数，以测量每个路径所花费的时间。我们添加了代码，以便对于特定路径，我们可以得到进行http调用的次数，所有调用花费的时间，平均时间等。

```go
var globalMap = make(map[string]Montioringpath)

func (t *myTransport) RoundTrip(request *http.Request) (*http.Response, error) {
    start := time.Now()
    response, err := http.DefaultTransport.RoundTrip(request)
    if err != nil {
        print("\n\ncame in error resp here", err)
        return nil, err //Server is not reachable. Server not working
    }
    elapsed := time.Since(start)

    key := request.Method + "-" + request.URL.Path
    // for example for POST Method with /path1 as url path key=POST-/path1

    if val, ok := globalMap[key]; ok {
        val.Count = val.Count + 1
        val.Duration += elapsed.Nanoseconds()
        val.AverageTime = val.Duration / val.Count
        globalMap[key] = val
        //do something here
    } else {
        var m Montioringpath
        m.Path = request.URL.Path
        m.Count = 1
        m.Duration = elapsed.Nanoseconds()
        m.AverageTime = m.Duration / m.Count
        globalMap[key] = m
    }

    b, err := json.MarshalIndent(globalMap, "", "  ")
    if err != nil {
        fmt.Println("error:", err)
    }

    body, err := httputil.DumpResponse(response, true)
    if err != nil {
        print("\n\nerror in dumb response")
        // copying the response body did not work
        return nil, err
    }

    log.Println("Response Body : ", string(body))
    log.Println("Response Time:", elapsed.Nanoseconds())
}
```

Go的Flag 包使得在运行程序时接受命令行参数。我们可以通过命令行参数设置http端口（将在其中运行代理服务器的位置）和重定向url（需要将http请求路由到的位置）。

```go
func main() {
    const (
        defaultPort        = ":9090"
        defaultPortUsage   = "default server port, ':9090'"
        defaultTarget      = "http://127.0.0.1:8080"
        defaultTargetUsage = "default redirect url, 'http://127.0.0.1:8080'"
    )

    // flags
    port = flag.String("port", defaultPort, defaultPortUsage)
    redirecturl = flag.String("url", defaultTarget, defaultTargetUsage)

    flag.Parse()

    fmt.Println("server will run on :", *port)
    fmt.Println("redirecting to :", *redirecturl)

    // proxy
    proxy := NewProxy(*redirecturl)

    http.HandleFunc("/proxyServer", ProxyServer)

    // server redirection
    http.HandleFunc("/", proxy.handle)
    log.Fatal(http.ListenAndServe(":"+*port, nil))
}
```

果在运行程序时未设置参数，则默认端口设置为9090，并且请求将路由到http://127.0.0.1:8080。

### 反向代理示例二

```go
func main() {
//New functionality written in Go
http.HandleFunc("/new", func(w http.ResponseWriter, r   *http.Request){
      fmt.Fprint(w, "New function")
})

//gospot.mchampaneri.in ===> localhost:8085
u1, _ := url.Parse("http://localhost:8085/")
http.Handle("gospot.mchampaneri.in/", httputil.NewSingleHostReverseProxy(u1))

// www.mchampaneri.in ===> localhost:8081  
u2, _ := url.Parse("http://localhost:8081/")
http.Handle("www.mchampaneri.in/", httputil.NewSingleHostReverseProxy(u2))

 // Start the server
http.ListenAndServe(":80", nil)
}
```
