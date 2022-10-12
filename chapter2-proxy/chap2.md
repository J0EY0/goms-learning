# Chapter 2 网络代理
- http代理
- https代理
- websocket代理
- tcp代理


#### 网络代理
- 用户通过代理请求信息
- 请求通过网络代理完成转发到达目标服务器
- 目标服务器响应后再通过网络代理回传给用户
- 与网络转发的区别
  - 网络代理: 用户不直接连接服务器，网络代理去连接。获取数据后再返回给用户 
  - 网络转发: 路由器对报文的转发操作，中间可能对数据包修改
- 正向代理
   - 客户端代理技术，帮助客户端访问无法访问的服务资源也可以隐藏用户的真实ip比如VPN等
- 反向代理
- 是一种服务端的代理技术，帮助服务器做负载均衡、缓存、提供安全校验，可以隐藏服务器等真实IP。比如nginx proxy_pass

简单的实现一个Web浏览器正向代理
- 代理接收客户端请求，复制原请求对象，并根据数据配置新请求各种参数
- 把新请求发送到真实服务端，并接收到服务器端返回
- 代理服务器对相应做一些处理，然后返回给客户端
```go
type Proxy struct{}

func (p *Proxy) ServeHTTP(rw http.ResponseWriter, req *http.Request){
    fmt.Printf("Received request %s %s %s\n",req.Method,req.Host,req.RemoteAddr)
    transport := http.DefaultTransport
    // 1. 浅拷贝对象，然后新增属性数据
    outReq :=new(http.Request)

    *outReq = *req
    outReq.Header = DeepCopy(req.Header).(map[string][]string)

    if clientIP, _,err:=net.SplitHostPort(req.RemoteAddr);err ==nil{
        if prior, ok := outReq.Header["X-Forwarded-For"];ok{
            clientIP = strings.Join(prior,", ") + ", " +clientIP
        }
        outReq.Header.Set("X-Forwarded-For",clientIP)
    }
    // 2. 请求下游
    res, err:= transport.RoundTrip(outReq)
    if err!=nil{
        rw.WriteHeader(http.StatusBadGateway)
        return 
    }

    // 3. 把下游请求内容返回给上游
    for key,value :=range res.Header{
        for _,v :=range value{
            rw.Header().Add(key,v)
        }
    }
    rw.WriteHeader(res.StatusCode)
    io.Copy(rw,res.Body)
    res.Body.Close()

}
```

实现一个简版http反向代理
- 代理接收客户端请求，更改请求结构体信息
- 通过一定负载均衡算法获取下游服务地址
- 把请求发送到下游服务器，并获取返回内容
- 对返回内容做处理，然后返回给客户端
```go
var (
	proxy_addr = "http://127.0.0.1:2003"
	port       = "2002"
)

func handler(w http.ResponseWriter, r *http.Request) {
	// 1. 解析代理地址，并更改请求体的协议和主机
	proxy, err := url.Parse(proxy_addr)
	r.URL.Scheme = proxy.Scheme
	r.URL.Host = proxy.Host

	// 2. 请求下游
	transport := http.DefaultTransport
	resp, err := transport.RoundTrip(r)
	if err != nil {
		log.Print(err)
		return
	}

	// 3. 把下游请求内容返回给上游
	for k, vv := range resp.Header {
		for _, v := range vv {
			w.Header().Add(k, v)
		}
	}
	defer resp.Body.Close()
	bufio.NewReader(resp.Body).WriteTo(w)
}

func main() {
	http.HandleFunc("/", handler)
	log.Println("Start serving on port " + port)
	err := http.ListenAndServe(":"+port, nil)
	if err != nil {
		log.Fatal(err)
	}
}
```

#### ReverseProxy --go
- 更改内容支持
- 错误信息回调
- 自定义负载均衡
- url重写功能
- 连接池功能
- 支持websocket服务
- 支持https代理
```go
func main(){
    rs := "http://127.0.0.1:2003/base"
    url, err:=url.Parse(rs) //返回一个解析后的URL结构体
    if err!=nil{
        log.Println(err)
    }
    proxy:=httputil.NewSingleHostReverseProxy(url)
    log.Println("Starting httpserver at "+addr)
    log.Fatal(http.ListenAndServe(addr,proxy))
}
```
ReverseProxy 特殊Header Connection
标记请求发起方与第一代理的状态
决定当前事务完成后，是否关闭网络
- keep-alive 不关闭网络
- close      关闭网络
- Upgrade    协议升级

TE是 request_header，表示希望使用的传输编码类型
- 如：TE：trailers，deflate;q=0.5表示，期望在采用分块传输编码响应中接收挂载字段，zlib编码，0.5优先级排序
Trailer是response header， 允许发送方在消息后面添加额外的元信息
- 如：Trailer：Expires，表示Expires将出现在分块信息的结尾
  
功能

- 负载均衡
  - 随机负载 随机挑选目标服务器ip
  - 轮询负载 按顺序依次访问不同服务器
  - 加权负载 给目标设置访问权重，按照权重轮询
  - 一致性hash负载 请求固定URL访问指定ip
- 中间件支持
- 限流和熔断
- 权限认证
- 数据统计

实现一个随机的负载均衡
```go
type RandomBalance struct{
    curIndex int
    rss []string

    //观察者模式
    conf LoadBalanceConf
}

func (r *RandomBalance) Add(params ..string) error{
    if len(params) == 0{
        return errors.New("param len 1 at least")
    }
    for _,v:=range params{
        r.rss = append(r.rss,v)
   }
    return nil
}

func (r *RandomBalance) Next() string{
    if len(r.rss) ==0{
        return ""
    }
    r.curIndex = rand.Intn(len(r.rss))
    return r.rss[r.curIndex]
}

func (r *RandomBalance) Get(key string) (string,error){
    return r.Next(),nil
}

```

实现一个轮询负载均衡
```go
type RandomBalance struct{
    curIndex int
    rss []string

    //观察者模式
    conf LoadBalanceConf
}

func (r *RandomBalance) Add(params ..string) error{
    if len(params) == 0{
        return errors.New("param len 1 at least")
    }
    for _,v:=range params{
        r.rss = append(r.rss,v)
   }
    return nil
}

func (r *RandomBalance) Next() string{
    if len(r.rss) ==0{
        return ""
    }
    len :=len(r.rss)
    
    curAddr :=r.rss[r.curIndex]
    r.curIndex = (r.curIndex + 1) % lens
    return curAdr
}
```
实现一个加权负载均衡
- Weight 初始化时对节点约定的权重
- currentWeight 节点临时权重，每轮都会变化
- effectiveWeight 节点有效权重，默认与Weight相同
- totalWeight 所有节点有效权重之和 sum(effectiveWeight)
```go
type WeightRoundRobinBalance struct {
    curIndex int
    rss []*WeightNode
    rsw []int
    //观察主体
    conf LoadBalanceConf
}

type WeightNode struct{
    addr string
    weight int
    currentWeight int 
    effectiveWeight int
}

func (r *WegihtRoundRobinBalance) Add(params ...string) error{
    if len(params) !=2 {
        return errors.New("param len need 2")
    }
    parInt, err:= strconv.ParseInt(params[1], 10, 64)

    if err!=nil{
        return err
    }

    node :=&WeightNode{addr :params[0], weight: int(parInt)}
    node.effectiveWeight = node.weight
    r.rss = append(r.rss,node)
    return nil
}

func (r *WeightRoundRobinBalance) Next() string{
    total :=0 
    var best *WeightNode
    for i:=0;i<len(r.rss); i++{
        w:=r.rss[i]
        // step1 统计所有有效权重之和
        total +=w.effectiveWeight

        // step2 变更节点临时权重为节点临时权重与节点有效权重之和
        w.currentWeight +=w.effectiveWeight
        
        // step3 有效权重默认与权重相同，通讯异常时为-1，通讯成功+1，直到恢复weight大小
        if w.effectiveWeight < w.weight{
            w.effectWeight++
        }

        // step4 选择最大临时权重点节点
        if best ==nil || w.currentWeight > best.currentWeight{
            best = w
        }
        if best ==nil{
            return ""
        }
        // step5 变更临时权重为 临时权重-有效权重之和
        best.currentWeight -=total
        return best.addr
    }
}
```

一致性hash指标
- 单调性
- 平衡性
- 分散性

```go
type Hash func(data []byte) uint32

type UInt32Slice []uint32

func (s UInt32Slice) Len()int{
    return len(s)
}

func (s UInt32Slice) Less(i, j int) bool{
    return s[i]<s[j]
}

func (s UInt32Slice) Swap(i,j int) {
    s[i],s[j] =s[j], s[i]
}

type ConsistentHashBalance struct{
    mux sync.RWMutex
    hash Hash
    replicas int //复制因子
    keys UInt32Slice //已排序的节点hash切片
    hashMap map[uint32]string //节点哈希和Key的map，键是hash值，值是节点的key

    //观察主体
    conf LoadBalanceConf 
}

func (c *ConsistentHashBalance) Add(params ...string) error{
    if len(params) ==0{
        return errors.New("param len 1 at least")
    }
    addr := params[0]
    c.mux.Lock()
    defer c.mux.Unlock()
    //结合复制因子计算所有虚拟节点的hash值，并存入m.keys中,同时在m.hashMap中保存哈希值和key的映射
    for i :=0;i<c.replicas;i++{
        hash:=c.hash([]byte(strconv.Itoa(i)+addr))
        c.keys = append(c.keys,hash)
        c.hashMap[hash] =addr
    }
    //对所有虚拟节点的哈希值进行排序，方便二分查找
    sort.Sort(c.keys)
}

func (c *ConsistentHashBalance) Get(key string) (string,error){
    if c.IsEmpty(){
        return "", errors.New("node is empty")
    }
    hash := c.hash([]byte(key))

    // 通过二分查找获取最优节点，第一个“服务器hash”值大于"数据hash"值就是最优“服务器节点”
    idx :=sort.Search(len(c.keys),func(i int) bool {return c.keys[i]>=hash})
    
    // 如果查找结果大于服务器节点哈希数组到最大索引，表示此时该对象哈希值位于最后一个节点之后，那么放入第一个节点中
    if idx ==len(c.keys){
        idx =0
    }
    c.mux.RLock()
    defer c.mux.RUnlock()
    return c.hashMapp[c.keys[idx]],nil
```

为反向代理增加负载均衡的功能
```go
type LbType int
const (
    LbRandom LbType =iota
    LbRoundRobin
    LbWeightRoundRobin
    LbConsistentHash
)
type LoadBalance interface{
    Add(...string) error
    Get(string) (string,error)

    //后期服务发现补充
    Update()
} 
func LoadBalanceFactory(lbType LBType) LoadBalance{
    swithc lbType{
    case LbRandom:
        return &RandomBalance{}
    case LbConsistentHash:
        return NewConsistentHashBalance(10,nil)
    case LbRooundRobin:
        return &RoundRobinBalance{}
    case LbWeightRoundRobin:
        return &WeightRoundRobinBalance{}
    default:
        return &RandomBalance{} 
    }
}
```