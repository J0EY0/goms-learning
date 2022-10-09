# 网络编程基础


- OSI七层网络协议
- 经典协议和数据包
- 三次握手和四次挥手
- TCP拥塞控制
- 粘包、拆包
- Go实现TCP、UDP、Http服务

TCP三次握手
- TCP的三次握手主要是保证连接是双工的，可靠主要通过重传机制来保证的  
- 因为连接是全双工的，双方都必须收到对方的FIN包及确认才可关闭

Time_wait 等待2MSL
- time_wait 等待2MSL(Maximum Segment Lifetime) 一般为30s-1min  
- 主要是为了保证TCP协议的全双工连接能够可靠关闭  
- 保证这次连接的重复数据段从网络中消失

为什么会出现大量的close_wait 
- close_wait 一般出现在被动关闭方
- 并发请求太多导致
- 被动关闭方未及时释放端口资源导致

TCP流量控制
- 通讯双方网速不同。通讯方任一方发送过快都会导致对方消息处理不过来，需要放进缓冲区
- 如果缓冲区满了，发送方还在发送，接收方只能丢弃数据包，因此需要控制发送速率

TCP拥塞控制
- 接收方网络资源繁忙，因未及时响应ACK导致发送方重传大量数据，导致网络更加拥堵
- 拥塞控制是动态调整win大小，不只是依赖缓冲区大小确定窗口大小
- 慢开始和拥塞避免
- 快速重传和快速恢复

为什么会出现粘包/拆包
- 应用程序写入的数据大于套接字缓冲区大小 --粘包
- 接受方法不及时读取套接字缓冲区数据 --粘包
- 应用程序写入的数据小于套接字缓冲区大小 --拆包
- 进行MSS(最大报文长度)大小的TCP分段，当TCP报文长度-TCP头部长度大于MSS的时候 --拆包

如何获取完整应用数据报文
- 使用带消息头的协议，头部写入包长度，然后再读包内容
- 设置定长消息，每次读取定长内容，长度不够时补充固定字符
- 设置消息边界，服务端从网络流中按消息边界分离出消息内容，一般使用'\n'
- 更复杂的协议如json, protobuf


定义一种数据格式: msg_header + content_len + content  
编解码  
tcp_client  
tcp_server

#### TCP协议  
服务端
```go
func main(){
    // 1、监听端口
    listener , err := net.Listen("tcp","127.0.0.1:80")
    if err!=nil{
        fmt.Printf("listen fail, err: %v\n",err)
        return
    }

    // 2.建立套接字连接
    for{
        conn, err:= listener.Accpet()
        if err!=nil{
            fmt.Printf("listen fail, err: %v\n",err)
            continue
        }

    //3. 协程处理
    go process(conn)
    }
}

func process(con net.Conn){
    defer conn.Close() //注意需声明conn的关闭，否则链接并不会马上释放

    for {
        var buf [128]byte
        n,err:=conn.Read(buf[:])
    
        if err!=nil{
            fmt.Printf("Read from connect failed, err: %v\n",err)
            break
        }
        str :=string(buf[:n])
        // ...... 处理逻辑
    }

}
```

客户端
```go
func main(){
    // 1.连接服务器
    conn, err :=net.Dial("tcp","127.0.0.1:80")
    defer conn.Close()
    if err!=nil{
        fmt.Printf("connect failed, err: %s\n",err.Error())
        return 
    }
    for{
        //2.逻辑
        //.......

        //3.发送

        _,err = conn.Write([]byte(trimmedInput))
        if err!=nil{
            fmt.Printf("write failed, err:%v\n",err)
            break
        }
    }
}
```

#### UDP协议
服务端
```go
    func main(){
        // 1.监听服务器
        listen, err:=net.ListenUDP("udp",&net.UDPAddr{
            IP: net.IPv4(0,0,0,0),
            Port: 9090,
        })
        if err!=nil{
            fmt.Printf("listen failed, err:%v\n",err)
            return
        }

        // 2.循环读取消息内容
        for {
            var data [1024]byte
            n,addr,err:=listen.ReadFromUDP(data[:])
            if err!=nil{
                fmt.Printf("Read fialed from addr: %v, err: %v\n",addr,err)
                break
            }

            go func(){
                _,err:=listen.WriteToUDP([]byte("received success!"),addr)
                if err!=nil{
                    fmt.Printf("write failed , err:%v\n",err)
                }
            }()
        }

    }
```
客户端 
```go
    func main(){
        // 1.连接服务器
        conn,err:=net.DialUDP("udp",nil,&net.UDPAddr{
            IP :net.IPv4(127,0,0,1)
            Port :9090,
        })

        if err!=nil{
            fmt.Printf("connect failed, err:%v\n",err)
            return 
        }

        for i :=0;i<100;i++{
            // 发送数据
            _,err = conn.Write([]byte("hello!"))

            if err!=nil{
                fmt.Printf("send data failed, err:%v\n",err)
                return
            }

            result : = make([]byte,1024)
            n ,remoteAddr, err:=conn.ReadFromUDP(result)
            if err!=nil{
                fmt.Printf("receive data failed, err:%v\n",err)
                return
            }
        }

    }
```

#### Http
服务端
```go
func main(){
    // 创建路由器
    mux := http.NewServeMux()
    // 创建路由
    mux.HandleFunc("/",hello)
    // 创建服务器

    server :=&http.Server{
        Addr:addr,
        WriteTimeout :time.Second *3,
        Handler :mux
    }
    // 运行
    go func(){
        if err:=server.ListenAndServe();err!=nil{
            log.Println("Failed for ",err.Error())
        }
    }()
}

func hello(w http.ResponseWriter, r *http.Request){
    time.Sleep(1 *time.Second)
    w.Write([]byte("Hi,there!"))
}
```
客户端
```go
func main(){
    transport :=&http.Transport{
        DialContext:(&net.Dialer{
            Timeout :30*time.Second, //连接超时
            KeepALive: 30*time.Second, //长连接超时时间
        }).DialContext,
        MaxIdleConns:100, //最大空闲连接
        IdleConnTimeout: 90*time.Second, //空闲超时时间
        TLSHandshakeTimeout: 10*time.Second, //tls握手超时时间
        ExpectContinueTimeout: 1*time.Second, // 100- continue状态码超时时间
    }
}

//创建客户端
client ：=&http.Client{
    Timeout:time.Second *30, //请求超时时间
    Transport ：transport,
}

//请求数据
resp , err:=client.Get("http://127.0.0.1:9090/hello");
if err!=nil{
    panic(err)
}

defer resp.Body.Close()

bbs,err:=ioutil.ReadAll(resp.Body)
if err!=nil{
    panic(err)
}
fmt.Println(json.Unmarshal(bds))
```
