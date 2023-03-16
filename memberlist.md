# cortex ingester 基于 hash ring 进行 token 管理
# 整体流程
![p1](https://user-images.githubusercontent.com/123643548/225557238-14745767-a0ff-4e45-8742-98f09cd9c536.png)

比如有3个ingester（下文简称in)，每个 ingester 产生如下512 个不同的 tokens
```
4222218899 4244974091 4247636007 4260587700 4270043296 4278792571
```
distributor 对ingester 产生的所有 token 进行排序
```
0(in1),5(in2),9(in3),20(in2),30(in3),100(in1)
```
有一个series 经过hash得到12，找到第一个大于12的token在9-20 的区间，所以这个series被写到in2。
假设现在新增一个ingester 节点，in4,他也产生512个token,重新加入上述排序队列，上述token队列可能变成
```
0(in1),5(in2),7(in4),9(in3),14(in4),20(in2),30(in3),50(in4),100(in1)
```
上述的series发现落在9-14区间，所以后续的series数据，都被写到in4。
![image](https://user-images.githubusercontent.com/123643548/225601331-d682948f-f22d-473c-9eb1-b505ffdd34f3.png)

# ring 工作流程
Ingester 加入 Memberlist 的流程
![image](https://user-images.githubusercontent.com/123643548/225558163-02139983-ef6c-4121-9ea0-22eaa84a55df.png)

## 初始化 KV store
```
type KV struct {
	// Protects access to memberlist and broadcasts fields.
	initWG     sync.WaitGroup
	memberlist *memberlist.Memberlist
	broadcasts *memberlist.TransmitLimitedQueue

    // KV Store.
	storeMu sync.Mutex
	store   map[string]valueDesc

	// Codec registry
	codecs map[string]codec.Codec

	// Key watchers
	watchersMu     sync.Mutex
	watchers       map[string][]chan string
	prefixWatchers map[string][]chan string
    }
```
```
// Client implements kv.Client interface, by using memberlist.KV
type Client struct {
	kv    *KV // reference to singleton memberlist-based KV
	codec codec.Codec
}
```
初始化一个基于 memberlist 的KV 存储,其他类型比如etcd/consul也有相应Client 的 interface。
1）store 表示实际传递的消息体，如果需要流转多种类型消息体，可以用map进行映射。
2）codecs 定义好 memberlist 中转发数据的解码工具codec。
```
// NewDesc returns an empty ring.Desc
func NewDesc() *Desc {
	return &Desc{
		Ingesters: map[string]InstanceDesc{},
	}
}
```
这里表示，ingester的ring,进行消息传递时，对应的pb结构体是 InstanceDesc。
3）watchers 动态监听key的数据集。
如下代码实例一个 memberlist 子节点：
```
case "memberlist":
	kv, err := cfg.MemberlistKV()
	if err != nil {
		return nil, err
	}
	client, err = memberlist.NewClient(kv, codec)
	if err != nil {
		return nil, err
	}
```
## 定义 ring 结构体
```
type Lifecycler struct {
	KVStore         kv.Client

	// These values are initialised at startup, and never change
	ID       string
	Addr     string
	RingName string
	RingKey  string
	Zone     string

	state        InstanceState
	tokens       Tokens
```
初始化一个基于memberlist的 Lifecycer Ring 结构体，
1) KVStore 即是前文的memberlist.NewClient，
2) tokens 记录当前 Ring 中的内容，由KVStore经过codecs解析后获取。
```
ringDesc = in.(*Desc)
myTokens, takenTokens := ringDesc.TokensFor(i.ID)
```
## ingester 节点加入 ring
1）ingester注册时，初始化KV store和Ring子节点，
2）首次加入memlister时，携带空的token，为了加快ring子节点间快速探测及加入
```
ringDesc.AddIngester(i.ID, i.Addr, i.Zone, []uint32{}, i.GetState(), registeredAt)
```
3）当tsdb文件加载完成后，再生成token，重新刷新ring的信息
```
myTokens, takenTokens := ringDesc.TokensFor(i.ID)
ringDesc.AddIngester(i.ID, i.Addr, i.Zone, i.getTokens(), i.GetState(), i.getRegisteredAt())
```
```
func (d *Desc) AddIngester(id, addr, zone string, tokens []uint32, state InstanceState, registeredAt time.Time) InstanceDesc {
...
ingester := InstanceDesc{
	Addr:                addr,
	Timestamp:           time.Now().Unix(),
	RegisteredTimestamp: registeredTimestamp,
	State:               state,
	Tokens:              tokens,
	Zone:                zone,
  }
	d.Ingesters[id] = ingester
```
## disrtibutor 获取 ingester的 token 信息
disrtibutor启动时，也会初始化一个memberlist的client，使用WatchKey来探测Ingester的token变化。
```
r.KVClient.WatchKey(ctx, r.key, func(value interface{}) bool {
	if value == nil {
		level.Info(r.logger).Log("msg", "ring doesn't exist in KV store yet")
		return true
	}
	r.updateRingState(value.(*Desc))
	return true
})
```
当有series进入Distributor时，会使用最新的token集合，来判断将此token推到哪一个Ingester进行后续处理。
```
func (d *Distributor) Push(ctx context.Context, req *cortexpb.WriteRequest)
  ...
  ring.DoBatch
    ...
	r.Get(key, op, bufDescs[:0], bufHosts[:0], bufZones[:0])
```
```
func (r *Ring) Get(key uint32, op Operation, bufDescs []InstanceDesc, bufHosts, bufZones []string) (ReplicationSet, error) {
	start      = searchToken(r.ringTokens, key)
```
