# v2ray 的启动流程
v2ray 全局管理一个 server 实例，所有功能都会被注册到该实例上。其为 core.Instance 类型：

```
// Instance combines all functionalities in V2Ray.
type Instance struct {
	access             sync.Mutex
	features           []features.Feature
	featureResolutions []resolution
	running            bool
}
```

这里的 featureResolution 在正常运行的情况下，启动之后一定是空的，如果非空，说明在 feature 注册阶段发生了错误。

features 为 Feature 数组，Feature 也是 interface：

```
// Feature is the interface for V2Ray features. All features must implement this interface.
// All existing features have an implementation in app directory. These features can be replaced by third-party ones.
type Feature interface {
	// Start starts the runnable object. Upon the method returning nil, the object begins to function properly.
	Start() error
	Close() error
	// Type returns the type of the object.
	// Usually it returns (*Type)(nil) of the object.
	Type() interface{}
}
```

main 函数流程：

```
func main() {
    // 根据启动时传入的文件位置加载并解析配置
    // 并根据配置生成全局 server 实例
	server, err := startV2Ray()

    // 调用 server.features 中所有 Feature 的 Start 方法
	server.Start()
	defer server.Close()

	// Explicitly triggering GC to remove garbage from config loading.
	runtime.GC()

	{
		osSignals := make(chan os.Signal, 1)
		signal.Notify(osSignals, os.Interrupt, os.Kill, syscall.SIGTERM)
		<-osSignals
	}
}
```

可见主要分三步：

1. 根据用户配置生成相应的数据结构
2. 依次启动 server 上的各种 feature
3. 监听信号，等待退出


### 生成数据结构

#### server 生成
读取配置文件的过程没什么可说的，server 实例的生成过程如下：

```
// 根据用户配置，创建一个 Instance 对象
// 这里返回的 instance 还没有启动
// 配置中至少需要包含一个 dispatcher，一个 inboundhandler manager 和一个 outbound handler manager。其它 feature 都是可选的。
func New(config *Config) (*Instance, error) {
	var server = &Instance{}

	for _, appSettings := range config.App {
		settings, err := appSettings.GetInstance()

		obj, err := CreateObject(server, settings)

		if feature, ok := obj.(features.Feature); ok {
			server.AddFeature(feature)
		}
	}

	essentialFeatures := []struct {
		Type     interface{}
		Instance features.Feature
	}{
		{dns.ClientType(), localdns.New()},
		{policy.ManagerType(), policy.DefaultManager{}},
		{routing.RouterType(), routing.DefaultRouter{}},
		{stats.ManagerType(), stats.NoopManager{}},
	}

    // 用户没有提供的话，需要保证 essentialFeatuers 里面这四个模块有对应的默认实现
	for _, f := range essentialFeatures {
		if server.GetFeature(f.Type) == nil {
			server.AddFeature(f.Instance)
		}
	}

    // 注册 inbound handlers
	if err := addInboundHandlers(server, config.Inbound); err != nil {
		return nil, err
	}

    // 注册 outbound handlers
	if err := addOutboundHandlers(server, config.Outbound); err != nil {
		return nil, err
	}

	return server, nil
}
```

可见 feature 大部分是由用户代码提供的，在用户提供的 feature 不全时，v2ray 会把 dns、policy、routing、stats 这几个模块使用默认实现进行占位。server 初始化时会将 inbound 和 outbound 对应的 handler 注册好。

#### IOC，生成各种 Object

```
package common

import (
	"context"
	"reflect"
)

// ConfigCreator is a function to create an object by a config.
type ConfigCreator func(ctx context.Context, config interface{}) (interface{}, error)

var (
	typeCreatorRegistry = make(map[reflect.Type]ConfigCreator)
)

// RegisterConfig registers a global config creator. The config can be nil but must have a type.
func RegisterConfig(config interface{}, configCreator ConfigCreator) error {
	configType := reflect.TypeOf(config)
	if _, found := typeCreatorRegistry[configType]; found {
		return newError(configType.Name() + " is already registered").AtError()
	}
	typeCreatorRegistry[configType] = configCreator
	return nil
}

// CreateObject creates an object by its config. The config type must be registered through RegisterConfig().
func CreateObject(ctx context.Context, config interface{}) (interface{}, error) {
	configType := reflect.TypeOf(config)
	creator, found := typeCreatorRegistry[configType]
	if !found {
		return nil, newError(configType.String() + " is not registered").AtError()
	}
	return creator(ctx, config)
}

```

### feature 启动

启动后可通过 dlv 查看这些 feature 具体的类型：

```
*v2ray.com/core/app/dispatcher.DefaultDispatcher {
*v2ray.com/core/app/proxyman/inbound.Manager {
*v2ray.com/core/app/proxyman/outbound.Manager {
*v2ray.com/core/app/log.Instance {
*v2ray.com/core/app/router.Router {
*v2ray.com/core/app/dns.Server {
v2ray.com/core/features/policy.DefaultManager {},
v2ray.com/core/features/stats.NoopManager {},
```

每一个 feature 启动在主流程中均调用 Start，我们只要依次查看其 Start 流程即可。

#### DefaultDispatcher.Start

```
// Start implements common.Runnable.
func (*DefaultDispatcher) Start() error {
	return nil
}
```

啥都没干

#### inbound.Manager.Start

```
// Start implements common.Runnable.
func (m *Manager) Start() error {
	m.access.Lock()
	defer m.access.Unlock()

	m.running = true

	for _, handler := range m.taggedHandlers {
		if err := handler.Start(); err != nil {
			return err
		}
	}

	for _, handler := range m.untaggedHandler {
		if err := handler.Start(); err != nil {
			return err
		}
	}
	return nil
}
```

```go
(dlv) p server.features[1].taggedHandlers["socksinbound"].workers
[]v2ray.com/core/app/proxyman/inbound.worker len: 2, cap: 2, [
	*v2ray.com/core/app/proxyman/inbound.tcpWorker {
		address: v2ray.com/core/common/net.Address(v2ray.com/core/common/net.ipv4Address) *(*"v2ray.com/core/common/net.Address")(0xc000210000),
		port: 1081,
		proxy: v2ray.com/core/proxy.Inbound(*v2ray.com/core/proxy/socks.Server) ...,
		stream: *(*v2ray.com/core/transport/internet.MemoryStreamConfig)(0xc000204460),
		recvOrigDest: false,
		tag: "socksinbound",
		dispatcher: v2ray.com/core/features/routing.Dispatcher(*v2ray.com/core/common/mux.Server) ...,
		sniffingConfig: *v2ray.com/core/app/proxyman.SniffingConfig nil,
		uplinkCounter: v2ray.com/core/features/stats.Counter nil,
		downlinkCounter: v2ray.com/core/features/stats.Counter nil,
		hub: v2ray.com/core/transport/internet.Listener(*v2ray.com/core/transport/internet/tcp.Listener) ...,},
	*v2ray.com/core/app/proxyman/inbound.udpWorker {
		RWMutex: (*sync.RWMutex)(0xc000206460),
		proxy: v2ray.com/core/proxy.Inbound(*v2ray.com/core/proxy/socks.Server) ...,
		hub: *(*v2ray.com/core/transport/internet/udp.Hub)(0xc0001fc9e0),
		address: v2ray.com/core/common/net.Address(v2ray.com/core/common/net.ipv4Address) *(*"v2ray.com/core/common/net.Address")(0xc000206490),
		port: 1081,
		tag: "socksinbound",
		stream: *(*v2ray.com/core/transport/internet.MemoryStreamConfig)(0xc000204460),
		dispatcher: v2ray.com/core/features/routing.Dispatcher(*v2ray.com/core/common/mux.Server) ...,
		uplinkCounter: v2ray.com/core/features/stats.Counter nil,
		downlinkCounter: v2ray.com/core/features/stats.Counter nil,
		checker: *(*v2ray.com/core/common/task.Periodic)(0xc0001c2750),
		activeConn: map[v2ray.com/core/app/proxyman/inbound.connID]*v2ray.com/core/app/proxyman/inbound.udpConn [],},
]

(dlv) p server.features[1].taggedHandlers["httpinbound"].workers
[]v2ray.com/core/app/proxyman/inbound.worker len: 1, cap: 1, [
	*v2ray.com/core/app/proxyman/inbound.tcpWorker {
		address: v2ray.com/core/common/net.Address(v2ray.com/core/common/net.ipv4Address) *(*"v2ray.com/core/common/net.Address")(0xc000210090),
		port: 8001,
		proxy: v2ray.com/core/proxy.Inbound(*v2ray.com/core/proxy/http.Server) ...,
		stream: *(*v2ray.com/core/transport/internet.MemoryStreamConfig)(0xc0002046e0),
		recvOrigDest: false,
		tag: "httpinbound",
		dispatcher: v2ray.com/core/features/routing.Dispatcher(*v2ray.com/core/common/mux.Server) ...,
		sniffingConfig: *v2ray.com/core/app/proxyman.SniffingConfig nil,
		uplinkCounter: v2ray.com/core/features/stats.Counter nil,
		downlinkCounter: v2ray.com/core/features/stats.Counter nil,
		hub: v2ray.com/core/transport/internet.Listener(*v2ray.com/core/transport/internet/tcp.Listener) ...,},
]
(dlv)
```

#### outbound.Manager.Start

```
// Start implements core.Feature
func (m *Manager) Start() error {
	m.access.Lock()
	defer m.access.Unlock()

	m.running = true

	for _, h := range m.taggedHandler {
		if err := h.Start(); err != nil {
			return err
		}
	}

	for _, h := range m.untaggedHandlers {
		if err := h.Start(); err != nil {
			return err
		}
	}

	return nil
}
```

```go
(dlv) p server.features[2].taggedHandler
map[string]v2ray.com/core/features/outbound.Handler [
	"v2.com_1.1.1.2": *v2ray.com/core/app/proxyman/outbound.Handler {
		tag: "v2.com_1.1.1.2",
		senderSettings: *(*v2ray.com/core/app/proxyman.SenderConfig)(0xc000200200),
		streamSettings: *(*v2ray.com/core/transport/internet.MemoryStreamConfig)(0xc000204d20),
		proxy: v2ray.com/core/proxy.Outbound(*v2ray.com/core/proxy/vmess/outbound.Handler) ...,
		outboundManager: v2ray.com/core/features/outbound.Manager(*v2ray.com/core/app/proxyman/outbound.Manager) ...,
		mux: *v2ray.com/core/common/mux.ClientManager nil,},
	"v2.com_1.0.7.2": *v2ray.com/core/app/proxyman/outbound.Handler {
		tag: "v2.com_1.0.7.2",
		senderSettings: *(*v2ray.com/core/app/proxyman.SenderConfig)(0xc000200480),
		streamSettings: *(*v2ray.com/core/transport/internet.MemoryStreamConfig)(0xc000205720),
		proxy: v2ray.com/core/proxy.Outbound(*v2ray.com/core/proxy/vmess/outbound.Handler) ...,
		outboundManager: v2ray.com/core/features/outbound.Manager(*v2ray.com/core/app/proxyman/outbound.Manager) ...,
		mux: *v2ray.com/core/common/mux.ClientManager nil,},
	"decline": *v2ray.com/core/app/proxyman/outbound.Handler {
		tag: "decline",
		senderSettings: *(*v2ray.com/core/app/proxyman.SenderConfig)(0xc000200700),
		streamSettings: *(*v2ray.com/core/transport/internet.MemoryStreamConfig)(0xc000205860),
		proxy: v2ray.com/core/proxy.Outbound(*v2ray.com/core/proxy/blackhole.Handler) ...,
		outboundManager: v2ray.com/core/features/outbound.Manager(*v2ray.com/core/app/proxyman/outbound.Manager) ...,
		mux: *v2ray.com/core/common/mux.ClientManager nil,},
	"direct": *v2ray.com/core/app/proxyman/outbound.Handler {
		tag: "direct",
		senderSettings: *(*v2ray.com/core/app/proxyman.SenderConfig)(0xc000200740),
		streamSettings: *(*v2ray.com/core/transport/internet.MemoryStreamConfig)(0xc0002059f0),
		proxy: v2ray.com/core/proxy.Outbound(*v2ray.com/core/proxy/freedom.Handler) ...,
		outboundManager: v2ray.com/core/features/outbound.Manager(*v2ray.com/core/app/proxyman/outbound.Manager) ...,
		mux: *v2ray.com/core/common/mux.ClientManager nil,},
]
(dlv)
```

#### log.Instance.Start

```
// Start implements common.Runnable.Start().
func (g *Instance) Start() error {
	if err := g.startInternal(); err != nil {
		return err
	}

	newError("Logger started").AtDebug().WriteToLog()

	return nil
}
```

#### router.Router.Start

```
// Start implements common.Runnable.
func (*Router) Start() error {
	return nil
}
```

#### dns.Server.Start

```
// Start implements common.Runnable.
func (s *Server) Start() error {
	return nil
}
```

#### policy.DefaultManager.Start

```
// Start implements common.Runnable.
func (DefaultManager) Start() error {
	return nil
}
```
#### stats.NoopManager.Start

```
func (NoopManager) Start() error { return nil }
```

### 信号监听

这部分比较古典，没什么可说的：

```
osSignals := make(chan os.Signal, 1)
signal.Notify(osSignals, os.Interrupt, os.Kill, syscall.SIGTERM)
<-osSignals
```