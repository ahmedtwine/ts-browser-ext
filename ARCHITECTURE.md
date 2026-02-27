# ts-browser-ext — Architecture

A Tailscale client that runs entirely inside your browser as an extension + native host,
giving each browser profile its own independent tailnet connection without touching the
system VPN or network settings.

## High-Level Overview

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              BROWSER (Chrome / Firefox)                              │
│                                                                                     │
│  ┌──────────────────────┐       ┌──────────────────────────────────────────────┐    │
│  │      popup.html       │       │              background.js                   │    │
│  │    ┌──────────────┐   │       │  (Service Worker / Background Script)        │    │
│  │    │  Toggle UI   │   │       │                                              │    │
│  │    │  Status Text │   │       │  ┌─────────────┐   ┌─────────────────────┐  │    │
│  │    │  Settings Btn│   │       │  │ Native Msg   │   │ Proxy Config        │  │    │
│  │    └──────┬───────┘   │       │  │ Port (nmPort)│   │ (chrome.proxy /     │  │    │
│  │           │           │       │  │              │   │  browser.proxy)     │  │    │
│  │    popup.js           │       │  └──────┬───────┘   └────────┬────────────┘  │    │
│  └───────────┬───────────┘       └─────────┼────────────────────┼───────────────┘    │
│              │                             │                    │                    │
│    chrome.runtime.connect       chrome.runtime                  │                    │
│    (port: "popup")              .connectNative()                │                    │
│              │                             │                    │                    │
└──────────────┼─────────────────────────────┼────────────────────┼────────────────────┘
               │                             │                    │
               │        ┌───────────────────────────┐             │
               │        │  Native Messaging Protocol │             │
               │        │  (stdin/stdout, length-    │             │
               │        │   prefixed JSON, LE u32)   │             │
               │        └─────────────┬─────────────┘             │
               │                      │                           │
               │                      ▼                           ▼
               │  ┌───────────────────────────────────────────────────────────────┐
               │  │                  ts-browser-ext.go                            │
               │  │              (Native Host Process)                            │
               │  │                                                               │
               │  │  ┌──────────────────────────────────────────────────────────┐ │
               │  │  │                    host struct                           │ │
               │  │  │                                                          │ │
               │  │  │  ┌────────────┐  ┌───────────────┐  ┌────────────────┐  │ │
               │  │  │  │ Message    │  │  tsnet.Server  │  │ Proxy Listener │  │ │
               │  │  │  │ Reader     │  │  (embedded     │  │ (127.0.0.1:0) │  │ │
               │  │  │  │ (stdin)    │  │   Tailscale)   │  │               │  │ │
               │  │  │  └─────┬──────┘  └───────┬────────┘  └───────┬───────┘  │ │
               │  │  │        │                 │                   │          │ │
               │  │  │        │                 │          ┌────────┴────────┐ │ │
               │  │  │        │                 │          │  proxymux.Split │ │ │
               │  │  │        │                 │          │  SOCKSAndHTTP() │ │ │
               │  │  │        │                 │          ├────────┬────────┤ │ │
               │  │  │        │                 │          │        │        │ │ │
               │  │  │        │                 │     ┌────▼───┐ ┌──▼─────┐ │ │ │
               │  │  │        │                 │     │ HTTP   │ │ SOCKS5 │ │ │ │
               │  │  │        │                 │     │ Proxy  │ │ Server │ │ │ │
               │  │  │        │                 │     └────┬───┘ └──┬─────┘ │ │ │
               │  │  │        │                 │          │        │       │ │ │
               │  │  │        │                 │          └───┬────┘       │ │ │
               │  │  │        │                 │              │            │ │ │
               │  │  │        │                 │         userDial()        │ │ │
               │  │  │        │                 │              │            │ │ │
               │  │  │        │                 ▼              ▼            │ │ │
               │  │  │        │          ┌──────────────────────────┐       │ │ │
               │  │  │        │          │  tsnet Dialer            │       │ │ │
               │  │  │        │          │  (WireGuard tunnel)      │       │ │ │
               │  │  │        │          └────────────┬─────────────┘       │ │ │
               │  │  │        │                       │                    │ │ │
               │  │  └────────┼───────────────────────┼────────────────────┘ │ │
               │  └───────────┼───────────────────────┼──────────────────────┘ │
               │              │                       │                        │
               │              │                       ▼                        │
               │              │              ┌─────────────────┐               │
               │              │              │   Tailscale      │               │
               │              │              │   Coordination   │               │
               │              │              │   Server (DERP)  │               │
               │              │              └────────┬─────────┘               │
               │              │                       │                        │
               │              │                       ▼                        │
               │              │              ┌─────────────────┐               │
               │              │              │  Tailnet Peers   │               │
               │              │              │  (other nodes)   │               │
               │              │              └──────────────────┘               │
```

## Detailed Data Flow

### 1. Installation & Registration

```
  User runs:
  $ ts-browser-ext --install=C<extension-id>      (Chrome)
  $ ts-browser-ext --install=F                     (Firefox)
         │
         ▼
  ┌──────────────────────────────────────────────────────────────────┐
  │  install() in ts-browser-ext.go                                  │
  │                                                                  │
  │  1. Copies binary to NativeMessagingHosts dir                    │
  │  2. Writes JSON manifest registering the native host             │
  │                                                                  │
  │  Target directories:                                             │
  │  ┌────────────────────────────────────────────────────────────┐  │
  │  │ macOS Chrome:                                              │  │
  │  │   ~/Library/Application Support/Google/Chrome/             │  │
  │  │     NativeMessagingHosts/com.tailscale.browserext.chrome   │  │
  │  │                                                            │  │
  │  │ macOS Firefox:                                             │  │
  │  │   ~/Library/Application Support/Mozilla/                   │  │
  │  │     NativeMessagingHosts/com.tailscale.browserext.firefox  │  │
  │  │                                                            │  │
  │  │ Linux Chrome:                                              │  │
  │  │   ~/.config/google-chrome/NativeMessagingHosts/            │  │
  │  │                                                            │  │
  │  │ Linux Firefox:                                             │  │
  │  │   ~/.mozilla/native-messaging-hosts/                       │  │
  │  └────────────────────────────────────────────────────────────┘  │
  │                                                                  │
  │  JSON manifest example (Chrome):                                 │
  │  {                                                               │
  │    "name": "com.tailscale.browserext.chrome",                    │
  │    "path": "/path/to/ts-browser-ext",                            │
  │    "type": "stdio",                                              │
  │    "allowed_origins": ["chrome-extension://<id>/"]               │
  │  }                                                               │
  └──────────────────────────────────────────────────────────────────┘
```

### 2. Startup Sequence

```
 Browser loads extension
         │
         ▼
 ┌─ background.js ────────────────────────────────────────────────────┐
 │                                                                    │
 │  1. connectToNativeHost()                                          │
 │     chrome.runtime.connectNative("com.tailscale.browserext.chrome")│
 │         │                                                          │
 │         │  Browser launches the registered native binary           │
 │         │  as a child process, wiring stdin/stdout                 │
 │         ▼                                                          │
 │  ┌─ ts-browser-ext.go (new process) ────────────────────────────┐  │
 │  │                                                              │  │
 │  │  main()                                                      │  │
 │  │    │                                                         │  │
 │  │    ├─ newHost(os.Stdin, os.Stdout)                           │  │
 │  │    │    └─ creates tsnet.Server (NOT started yet)            │  │
 │  │    │                                                         │  │
 │  │    ├─ getProxyListener()                                     │  │
 │  │    │    ├─ net.Listen("tcp", "127.0.0.1:0")  (random port)  │  │
 │  │    │    ├─ proxymux.SplitSOCKSAndHTTP(ln)                    │  │
 │  │    │    ├─ Start HTTP proxy  ──► httpProxyHandler()          │  │
 │  │    │    └─ Start SOCKS5 server ──► socks5.Server{Dialer:    │  │
 │  │    │                                             userDial}   │  │
 │  │    │                                                         │  │
 │  │    ├─ send({procRunning: {port: N, pid: P}})                 │  │
 │  │    │    │                                                    │  │
 │  │    │    │  ┌──────────────────────────────────────────┐      │  │
 │  │    │    │  │ Native Messaging Wire Format             │      │  │
 │  │    │    │  │                                          │      │  │
 │  │    │    │  │  ┌───────────┬───────────────────────┐   │      │  │
 │  │    │    │  │  │ 4 bytes   │  N bytes              │   │      │  │
 │  │    │    │  │  │ LE uint32 │  JSON payload         │   │      │  │
 │  │    │    │  │  │ (length)  │                       │   │      │  │
 │  │    │    │  │  └───────────┴───────────────────────┘   │      │  │
 │  │    │    │  │  Max message size: 1 MiB                 │      │  │
 │  │    │    │  └──────────────────────────────────────────┘      │  │
 │  │    │    │                                                    │  │
 │  │    │    ▼  (message arrives in background.js)                │  │
 │  │    │                                                         │  │
 │  │    └─ readMessages()  ◄─── blocks, waiting for commands      │  │
 │  │                                                              │  │
 │  └──────────────────────────────────────────────────────────────┘  │
 │                                                                    │
 │  2. On receiving {procRunning: {port: N}}:                         │
 │     └─ setProxy(N)  ──► configures browser proxy to 127.0.0.1:N   │
 │                                                                    │
 │  3. Load profileID from chrome.storage.local                       │
 │     └─ If none exists, generate UUID and store it                  │
 │                                                                    │
 │  4. maybeSendInit()                                                │
 │     └─ nmPort.postMessage({cmd: "init", initID: "<uuid>"})        │
 │                                                                    │
 └────────────────────────────────────────────────────────────────────┘
         │
         ▼
 ┌─ ts-browser-ext.go handles "init" ────────────────────────────────┐
 │                                                                    │
 │  handleInit(msg)                                                   │
 │    │                                                               │
 │    ├─ Validate initID (hex + hyphens, max 60 chars)                │
 │    │                                                               │
 │    ├─ Set hostname: "<username>-browser-ext"                       │
 │    │                                                               │
 │    ├─ Set state dir: $XDG_CONFIG/tailscale-browser-ext/<initID>    │
 │    │  (each browser profile gets its own Tailscale state)          │
 │    │                                                               │
 │    ├─ tsnet.Server.Start()                                         │
 │    │   └─ Starts embedded WireGuard, connects to coordination      │
 │    │      server, joins tailnet as a new node                      │
 │    │                                                               │
 │    ├─ lc.WatchIPNBus()  ──► spawns goroutine: watchIPNBus()       │
 │    │   └─ Monitors state changes (Running, NeedsLogin, Stopped)    │
 │    │   └─ Monitors netmap changes (tailnet domain, peer list)      │
 │    │   └─ Sends status updates to extension on every change        │
 │    │                                                               │
 │    ├─ web.NewServer()  ──► Tailscale web admin UI                  │
 │    │   └─ Serves at http://100.100.100.100 via the proxy           │
 │    │                                                               │
 │    └─ send({init: {error: ""}})  ──► success                      │
 │                                                                    │
 └────────────────────────────────────────────────────────────────────┘
```

### 3. Proxy Traffic Flow (Steady State)

```
 ┌─────────────────────────────────────────────────────────────────────────┐
 │                        Browser makes a request                          │
 │                     e.g. http://my-server.ts.net                        │
 └──────────────────────────────────┬──────────────────────────────────────┘
                                    │
                     ┌──────────────┴──────────────┐
                     │   Browser Proxy Settings     │
                     │                              │
                     │   Chrome: chrome.proxy.      │
                     │     settings.set({           │
                     │       mode: "fixed_servers", │
                     │       rules: {               │
                     │         singleProxy: {       │
                     │           scheme: "http",    │
                     │           host: "127.0.0.1", │
                     │           port: <port>       │
                     │         },                   │
                     │         bypassList:          │
                     │           ["localhost",      │
                     │            "127.*"]           │
                     │       }                      │
                     │     })                       │
                     │                              │
                     │   Firefox: browser.proxy.    │
                     │     onRequest listener       │
                     │     (per-request routing)    │
                     └──────────────┬───────────────┘
                                    │
            ┌───────────────────────┼───────────────────────┐
            │                       │                       │
            ▼                       ▼                       ▼
   ┌─────────────────┐   ┌──────────────────┐   ┌──────────────────┐
   │  HTTP request   │   │  HTTPS (CONNECT) │   │  SOCKS5 request  │
   │  (plain HTTP)   │   │  request         │   │  (Firefox-only   │
   │                 │   │                  │   │   for DNS)       │
   └────────┬────────┘   └────────┬─────────┘   └────────┬─────────┘
            │                     │                       │
            ▼                     ▼                       ▼
   ┌────────────────────────────────────────────────────────────────┐
   │              127.0.0.1:<port>  (single listener)               │
   │                                                                │
   │              proxymux.SplitSOCKSAndHTTP()                      │
   │              ┌──────────────────┬──────────────────┐           │
   │              │  Detect protocol │  by first byte:  │           │
   │              │  0x05 = SOCKS5   │  other = HTTP    │           │
   │              └────────┬─────────┴────────┬─────────┘           │
   │                       │                  │                     │
   │              ┌────────▼────────┐ ┌───────▼────────────┐        │
   │              │  socks5.Server  │ │  httpProxyHandler() │        │
   │              │                 │ │                     │        │
   │              │  Dialer:        │ │  if host ==         │        │
   │              │   userDial()    │ │  100.100.100.100:   │        │
   │              │                 │ │    ──► web.Server   │        │
   │              │                 │ │    (Tailscale admin)│        │
   │              │                 │ │                     │        │
   │              │                 │ │  if CONNECT:        │        │
   │              │                 │ │    ──► TCP tunnel   │        │
   │              │                 │ │    via userDial()   │        │
   │              │                 │ │                     │        │
   │              │                 │ │  else (plain HTTP): │        │
   │              │                 │ │    ──► ReverseProxy │        │
   │              │                 │ │    via userDial()   │        │
   │              └────────┬────────┘ └───────┬────────────┘        │
   │                       │                  │                     │
   │                       └──────┬───────────┘                     │
   │                              │                                 │
   │                       userDial(ctx, net, addr)                 │
   │                              │                                 │
   │                     tsnet.Sys().Dialer                         │
   │                       .UserDial()                              │
   │                              │                                 │
   └──────────────────────────────┼─────────────────────────────────┘
                                  │
                                  ▼
                   ┌──────────────────────────┐
                   │    WireGuard Tunnel       │
                   │    (embedded in tsnet)    │
                   │                           │
                   │    Encrypted UDP packets  │
                   │    to/from tailnet peers  │
                   └──────────────────────────┘
```

### 4. Firefox vs Chrome Proxy Differences

```
 ┌─ Chrome ─────────────────────────────────┐  ┌─ Firefox ──────────────────────────────────┐
 │                                          │  │                                            │
 │  chrome.proxy.settings.set({             │  │  browser.proxy.onRequest.addListener(       │
 │    value: {                              │  │    function(requestInfo) {                  │
 │      mode: "fixed_servers",              │  │      url = new URL(requestInfo.url)         │
 │      rules: {                            │  │                                            │
 │        singleProxy: {                    │  │      if (url.hostname == '100.100.100.100') │
 │          scheme: "http",                 │  │        return {type: "http", ...port}       │
 │          host: "127.0.0.1",              │  │                                            │
 │          port: <port>                    │  │      // SOCKS5 for everything else          │
 │        },                                │  │      // (proxyDNS: true for DNS resolution) │
 │        bypassList: [                     │  │      return {type: "socks", ...port,        │
 │          "localhost", "127.*"            │  │              proxyDNS: true}                │
 │        ]                                 │  │    }                                       │
 │      }                                   │  │  )                                         │
 │    }                                     │  │                                            │
 │  })                                      │  │  Why? Firefox's HTTP proxy mode doesn't    │
 │                                          │  │  forward DNS queries through the proxy.    │
 │  All traffic goes through HTTP proxy.    │  │  SOCKS5 with proxyDNS: true does, which    │
 │  DNS resolution happens normally.        │  │  is needed to resolve Tailscale hostnames  │
 │                                          │  │  (like *.ts.net) via MagicDNS.             │
 │                                          │  │                                            │
 │                                          │  │  Exception: 100.100.100.100 uses HTTP      │
 │                                          │  │  proxy since the web admin client expects   │
 │                                          │  │  HTTP, not SOCKS.                          │
 └──────────────────────────────────────────┘  └────────────────────────────────────────────┘
```

### 5. Message Protocol (Native Messaging)

```
 ┌──────────────────────────────────────────────────────────────────────────┐
 │                      Message Types & Flow                                │
 │                                                                          │
 │  background.js ──────────────────────────► ts-browser-ext.go             │
 │  (Requests)                                (Commands)                    │
 │                                                                          │
 │  ┌─────────────────────┐                                                 │
 │  │ {cmd: "init",       │ ──► handleInit()                               │
 │  │  initID: "<uuid>"}  │     Start tsnet, join tailnet, watch IPN bus    │
 │  └─────────────────────┘                                                 │
 │                                                                          │
 │  ┌─────────────────────┐                                                 │
 │  │ {cmd: "get-status"} │ ──► sendStatus()                               │
 │  └─────────────────────┘     Return current tailnet state                │
 │                                                                          │
 │  ┌─────────────────────┐                                                 │
 │  │ {cmd: "up"}         │ ──► handleUp() ──► setWantRunning(true)        │
 │  └─────────────────────┘     EditPrefs: WantRunning=true                 │
 │                                                                          │
 │  ┌─────────────────────┐                                                 │
 │  │ {cmd: "down"}       │ ──► handleDown() ──► setWantRunning(false)     │
 │  └─────────────────────┘     EditPrefs: WantRunning=false                │
 │                                                                          │
 │                                                                          │
 │  ts-browser-ext.go ──────────────────────► background.js                 │
 │  (Replies)                                 (Handled in onMessage)        │
 │                                                                          │
 │  ┌───────────────────────────────┐                                       │
 │  │ {procRunning:                 │  First message on startup.            │
 │  │   {port: 54321, pid: 1234}}  │  Extension configures proxy to port.  │
 │  └───────────────────────────────┘                                       │
 │                                                                          │
 │  ┌───────────────────────────────┐                                       │
 │  │ {init: {error: ""}}           │  Response to "init" command.          │
 │  └───────────────────────────────┘  Empty error = success.               │
 │                                                                          │
 │  ┌───────────────────────────────┐                                       │
 │  │ {status: {                    │  Sent on state changes                │
 │  │   running: true/false,        │  (from IPN bus watcher)               │
 │  │   tailnet: "example.com",     │  and in response to "get-status".    │
 │  │   needsLogin: true/false,     │                                       │
 │  │   browseToURL: "https://...", │                                       │
 │  │   error: "..."                │                                       │
 │  │ }}                            │                                       │
 │  └───────────────────────────────┘                                       │
 │                                                                          │
 └──────────────────────────────────────────────────────────────────────────┘
```

### 6. UI Communication (Popup ↔ Background)

```
 ┌─ popup.html / popup.js ──────────────────────────────────────────────┐
 │                                                                      │
 │  ┌──────────────────┐                                                │
 │  │   Tailscale Logo │  ┌──────────┐                                  │
 │  │                  │  │ Toggle   │◄── slider input                  │
 │  └──────────────────┘  └────┬─────┘                                  │
 │                              │                                       │
 │  ┌──────────────────────────┐│  ┌──────────────┐                     │
 │  │ Status: "Connected as   ││  │  [Settings]  │                     │
 │  │  example.com"            ││  │   button     │                     │
 │  └──────────────────────────┘│  └──────┬───────┘                     │
 │                              │         │                             │
 │     On toggle change:        │     On click:                         │
 │     sendMessage({command:    │     tabs.create({                     │
 │       "toggleProxy"})        │       url: "http://100.100.100.100"}) │
 │              │               │         │                             │
 └──────────────┼───────────────┘─────────┼─────────────────────────────┘
                │                         │
                ▼                         ▼
 ┌─ background.js ──────────┐   ┌──────────────────────────────────────┐
 │                          │   │  Tailscale Web Admin UI               │
 │  onMessage listener      │   │  served via HTTP proxy at             │
 │    toggleProxy:          │   │  100.100.100.100 ──► web.Server       │
 │      ├─ enable ──► up    │   └──────────────────────────────────────┘
 │      └─ disable ──► down │
 │                          │
 │  ◄── port "popup" ──►    │
 │  Persistent connection   │
 │  for push status updates │
 │    sendPopupStatus()     │
 │      ├─ {installCmd:...} │  (native host not found)
 │      └─ {status: {...}}  │  (normal status update)
 │                          │
 └──────────────────────────┘
```

### 7. State Management (Per-Profile Isolation)

```
 ┌─ Browser ────────────────────────────────────────────────────────────┐
 │                                                                      │
 │  Profile A (Personal)              Profile B (Work)                  │
 │  ┌─────────────────────┐          ┌─────────────────────┐            │
 │  │ chrome.storage.local│          │ chrome.storage.local│            │
 │  │ profileId: "abc-123"│          │ profileId: "def-456"│            │
 │  └─────────┬───────────┘          └─────────┬───────────┘            │
 │            │                                │                        │
 └────────────┼────────────────────────────────┼────────────────────────┘
              │                                │
              ▼                                ▼
  ┌───────────────────────┐      ┌───────────────────────┐
  │ ts-browser-ext (pid A)│      │ ts-browser-ext (pid B)│
  │                       │      │                       │
  │ State dir:            │      │ State dir:            │
  │ $CONFIG/tailscale-    │      │ $CONFIG/tailscale-    │
  │  browser-ext/abc-123/ │      │  browser-ext/def-456/ │
  │                       │      │                       │
  │ Hostname:             │      │ Hostname:             │
  │ "alice-browser-ext"   │      │ "alice-browser-ext"   │
  │                       │      │                       │
  │ Tailnet: personal.net │      │ Tailnet: work.corp    │
  │ Proxy: 127.0.0.1:9001│      │ Proxy: 127.0.0.1:9002│
  └───────────────────────┘      └───────────────────────┘
              │                                │
              ▼                                ▼
     ┌────────────────┐              ┌────────────────┐
     │ WireGuard      │              │ WireGuard      │
     │ Tunnel A       │              │ Tunnel B       │
     │ (personal.net) │              │ (work.corp)    │
     └────────────────┘              └────────────────┘
```

### 8. Complete Request Lifecycle

```
 User types "http://my-server.ts.net" in browser
         │
         ▼
 ┌───────────────────────────────────────────────────────────────────────┐
 │ 1. Browser checks proxy settings                                      │
 │    "127.0.0.1" not in bypassList ──► route through proxy              │
 └───────────────────────────────┬───────────────────────────────────────┘
                                 │
                                 ▼
 ┌───────────────────────────────────────────────────────────────────────┐
 │ 2. Browser connects to 127.0.0.1:<port>                               │
 │                                                                       │
 │    Chrome: HTTP CONNECT my-server.ts.net:443                          │
 │    Firefox: SOCKS5 connect to my-server.ts.net:443 (proxyDNS=true)   │
 └───────────────────────────────┬───────────────────────────────────────┘
                                 │
                                 ▼
 ┌───────────────────────────────────────────────────────────────────────┐
 │ 3. proxymux detects protocol (SOCKS5 byte 0x05 vs HTTP)               │
 │    Routes to appropriate handler                                      │
 └───────────────────────────────┬───────────────────────────────────────┘
                                 │
                                 ▼
 ┌───────────────────────────────────────────────────────────────────────┐
 │ 4. Handler calls userDial(ctx, "tcp", "my-server.ts.net:443")         │
 │    └─► tsnet.Sys().Dialer.UserDial()                                  │
 │        └─► Resolves "my-server.ts.net" via MagicDNS (100.100.100.100)│
 │        └─► Finds peer in netmap                                       │
 │        └─► Establishes WireGuard-encrypted connection to peer         │
 └───────────────────────────────┬───────────────────────────────────────┘
                                 │
                                 ▼
 ┌───────────────────────────────────────────────────────────────────────┐
 │ 5. Data flows bidirectionally:                                        │
 │                                                                       │
 │    Browser ◄──► Proxy (127.0.0.1) ◄──► WireGuard ◄──► Peer node     │
 │                                                                       │
 │    The proxy handler uses io.Copy in both directions with goroutines  │
 │    for concurrent bidirectional streaming (CONNECT tunneling).         │
 └───────────────────────────────────────────────────────────────────────┘
```

## File Structure

```
ts-browser-ext/
├── ts-browser-ext.go        # Go native host: tsnet, proxy, native messaging
├── manifest.json            # Chrome extension manifest (MV3)
├── background.js            # Chrome service worker: proxy config, native msg
├── popup.html               # Shared popup UI (Tailscale-branded)
├── popup.js                 # Chrome popup logic: toggle, status display
├── icon.png                 # Default extension icon
├── online.png               # Icon: connected state
├── offline.png              # Icon: disconnected state
├── go.mod / go.sum          # Go module (depends on tailscale.com)
├── firefox/                 # Firefox-specific overrides
│   ├── manifest.json        #   Gecko-specific manifest (browser_specific_settings)
│   ├── background.js        #   Uses browser.* API, SOCKS5 proxy for DNS
│   ├── popup.js             #   Uses browser.* API, incognito permission check
│   ├── popup.html           #   (same as root)
│   ├── icon.png / *.png     #   (same icons)
└── chrome.txt               # Dev notes: example native host registration
```
