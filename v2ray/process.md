# 网络交互

```go
func (w *tcpWorker) Start() error {
	ctx := context.Background()
	hub, err := internet.ListenTCP(ctx, w.address, w.port, w.stream, func(conn internet.Connection) {
		go w.callback(conn)
	})
	if err != nil {
		return newError("failed to listen TCP on ", w.port).AtWarning().Base(err)
	}
	w.hub = hub
	return nil
}
```

internet/tcp_hub.go:
```go
func ListenTCP(ctx context.Context, address net.Address, port net.Port, settings *MemoryStreamConfig, handler ConnHandler) (Listener, error) {
	if settings == nil {
		s, err := ToMemoryStreamConfig(nil)
		if err != nil {
			return nil, newError("failed to create default stream settings").Base(err)
		}
		settings = s
	}

	if address.Family().IsDomain() && address.Domain() == "localhost" {
		address = net.LocalHostIP
	}

	if address.Family().IsDomain() {
		return nil, newError("domain address is not allowed for listening: ", address.Domain())
	}

	protocol := settings.ProtocolName
	listenFunc := transportListenerCache[protocol]
	if listenFunc == nil {
		return nil, newError(protocol, " listener not registered.").AtError()
	}
	listener, err := listenFunc(ctx, address, port, settings, handler)
	if err != nil {
		return nil, newError("failed to listen on address: ", address, ":", port).Base(err)
	}
	return listener, nil
}
```

```go
func RegisterTransportListener(protocol string, listener ListenFunc) error {
	if _, found := transportListenerCache[protocol]; found {
		return newError(protocol, " listener already registered.").AtError()
	}
	transportListenerCache[protocol] = listener
	return nil
}
```

### http listen

```go
internet.RegisterTransportListener("http", Listen)
```

```go
func Listen(ctx context.Context, address net.Address, port net.Port, streamSettings *internet.MemoryStreamConfig, handler internet.ConnHandler) (internet.Listener, error) {
	httpSettings := streamSettings.ProtocolSettings.(*Config)
	listener := &Listener{
		handler: handler,
		local: &net.TCPAddr{
			IP:   address.IP(),
			Port: int(port),
		},
		config: *httpSettings,
	}

	config := tls.ConfigFromStreamSettings(streamSettings)
	if config == nil {
		return nil, newError("TLS must be enabled for http transport.").AtWarning()
	}

	server := &http.Server{
		Addr:              serial.Concat(address, ":", port),
		TLSConfig:         config.GetTLSConfig(tls.WithNextProto("h2")),
		Handler:           listener,
		ReadHeaderTimeout: time.Second * 4,
	}

	listener.server = server
	go func() {
		tcpListener, err := internet.ListenSystem(ctx, &net.TCPAddr{
			IP:   address.IP(),
			Port: int(port),
		}, streamSettings.SocketSettings)
		if err != nil {
			newError("failed to listen on", address, ":", port).Base(err).WriteToLog(session.ExportIDToError(ctx))
			return
		}

		err = server.ServeTLS(tcpListener, "", "")
		if err != nil {
			newError("stoping serving TLS").Base(err).WriteToLog(session.ExportIDToError(ctx))
		}
	}()

	return listener, nil
}
```

### quic listen

internet/quic/hub.go:
```
internet.RegisterTransportListener("quic", Listen)
```

listen 实现:
```go
// Listen creates a new Listener based on configurations.
func Listen(ctx context.Context, address net.Address, port net.Port, streamSettings *internet.MemoryStreamConfig, handler internet.ConnHandler) (internet.Listener, error) {
	if address.Family().IsDomain() {
		return nil, newError("domain address is not allows for listening quic")
	}

	tlsConfig := tls.ConfigFromStreamSettings(streamSettings)
	if tlsConfig == nil {
		tlsConfig = &tls.Config{
			Certificate: []*tls.Certificate{tls.ParseCertificate(cert.MustGenerate(nil, cert.DNSNames(internalDomain), cert.CommonName(internalDomain)))},
		}
	}

	config := streamSettings.ProtocolSettings.(*Config)
	rawConn, err := internet.ListenSystemPacket(context.Background(), &net.UDPAddr{
		IP:   address.IP(),
		Port: int(port),
	}, streamSettings.SocketSettings)

	if err != nil {
		return nil, err
	}

	quicConfig := &quic.Config{
		ConnectionIDLength:    12,
		HandshakeTimeout:      time.Second * 8,
		IdleTimeout:           time.Second * 45,
		MaxIncomingStreams:    32,
		MaxIncomingUniStreams: -1,
	}

	conn, err := wrapSysConn(rawConn, config)
	if err != nil {
		conn.Close()
		return nil, err
	}

	qListener, err := quic.Listen(conn, tlsConfig.GetTLSConfig(), quicConfig)
	if err != nil {
		conn.Close()
		return nil, err
	}

	listener := &Listener{
		done:     done.New(),
		rawConn:  conn,
		listener: qListener,
		addConn:  handler,
	}

	go listener.keepAccepting()

	return listener, nil
}
```

### tcp listen

```
// ListenTCP creates a new Listener based on configurations.
func ListenTCP(ctx context.Context, address net.Address, port net.Port, streamSettings *internet.MemoryStreamConfig, handler internet.ConnHandler) (internet.Listener, error) {
	listener, err := internet.ListenSystem(ctx, &net.TCPAddr{
		IP:   address.IP(),
		Port: int(port),
	}, streamSettings.SocketSettings)
	if err != nil {
		return nil, err
	}
	newError("listening TCP on ", address, ":", port).WriteToLog(session.ExportIDToError(ctx))

	tcpSettings := streamSettings.ProtocolSettings.(*Config)
	l := &Listener{
		listener: listener,
		config:   tcpSettings,
		addConn:  handler,
	}

	if config := tls.ConfigFromStreamSettings(streamSettings); config != nil {
		l.tlsConfig = config.GetTLSConfig(tls.WithNextProto("h2"))
	}

	if tcpSettings.HeaderSettings != nil {
		headerConfig, err := tcpSettings.HeaderSettings.GetInstance()
		if err != nil {
			return nil, newError("invalid header settings").Base(err).AtError()
		}
		auth, err := internet.CreateConnectionAuthenticator(headerConfig)
		if err != nil {
			return nil, newError("invalid header settings.").Base(err).AtError()
		}
		l.authConfig = auth
	}
	go l.keepAccepting()
	return l, nil
}
```

### websocket listen

```
func ListenWS(ctx context.Context, address net.Address, port net.Port, streamSettings *internet.MemoryStreamConfig, addConn internet.ConnHandler) (internet.Listener, error) {
	wsSettings := streamSettings.ProtocolSettings.(*Config)

	var tlsConfig *tls.Config
	if config := v2tls.ConfigFromStreamSettings(streamSettings); config != nil {
		tlsConfig = config.GetTLSConfig()
	}

	listener, err := listenTCP(ctx, address, port, tlsConfig, streamSettings.SocketSettings)
	if err != nil {
		return nil, err
	}

	l := &Listener{
		config:   wsSettings,
		addConn:  addConn,
		listener: listener,
	}

	l.server = http.Server{
		Handler: &requestHandler{
			path: wsSettings.GetNormalizedPath(),
			ln:   l,
		},
		ReadHeaderTimeout: time.Second * 4,
		MaxHeaderBytes:    2048,
	}

	go func() {
		if err := l.server.Serve(l.listener); err != nil {
			newError("failed to serve http for WebSocket").Base(err).AtWarning().WriteToLog(session.ExportIDToError(ctx))
		}
	}()

	return l, err
}
```

### domainsocket listen

```
func Listen(ctx context.Context, address net.Address, port net.Port, streamSettings *internet.MemoryStreamConfig, handler internet.ConnHandler) (internet.Listener, error) {
	settings := streamSettings.ProtocolSettings.(*Config)
	addr, err := settings.GetUnixAddr()
	if err != nil {
		return nil, err
	}

	unixListener, err := net.ListenUnix("unix", addr)
	if err != nil {
		return nil, newError("failed to listen domain socket").Base(err).AtWarning()
	}

	ln := &Listener{
		addr:    addr,
		ln:      unixListener,
		config:  settings,
		addConn: handler,
	}

	if !settings.Abstract {
		ln.locker = &fileLocker{
			path: settings.Path + ".lock",
		}
		if err := ln.locker.Acquire(); err != nil {
			unixListener.Close()
			return nil, err
		}
	}

	if config := tls.ConfigFromStreamSettings(streamSettings); config != nil {
		ln.tlsConfig = config.GetTLSConfig()
	}

	go ln.run()

	return ln, nil
}
```

### mkcp listen

```
func ListenKCP(ctx context.Context, address net.Address, port net.Port, streamSettings *internet.MemoryStreamConfig, addConn internet.ConnHandler) (internet.Listener, error) {
	return NewListener(ctx, address, port, streamSettings, addConn)
}

func NewListener(ctx context.Context, address net.Address, port net.Port, streamSettings *internet.MemoryStreamConfig, addConn internet.ConnHandler) (*Listener, error) {
	kcpSettings := streamSettings.ProtocolSettings.(*Config)
	header, err := kcpSettings.GetPackerHeader()
	if err != nil {
		return nil, newError("failed to create packet header").Base(err).AtError()
	}
	security, err := kcpSettings.GetSecurity()
	if err != nil {
		return nil, newError("failed to create security").Base(err).AtError()
	}
	l := &Listener{
		header:   header,
		security: security,
		reader: &KCPPacketReader{
			Header:   header,
			Security: security,
		},
		sessions: make(map[ConnectionID]*Connection),
		config:   kcpSettings,
		addConn:  addConn,
	}

	if config := v2tls.ConfigFromStreamSettings(streamSettings); config != nil {
		l.tlsConfig = config.GetTLSConfig()
	}

	hub, err := udp.ListenUDP(ctx, address, port, streamSettings, udp.HubCapacity(1024))
	if err != nil {
		return nil, err
	}
	l.Lock()
	l.hub = hub
	l.Unlock()
	newError("listening on ", address, ":", port).WriteToLog()

	go l.handlePackets()

	return l, nil
}
```
