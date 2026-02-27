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

---

## Appendix: Architecture Comparison — Native Messaging vs WebAssembly

This section compares the approach used by this project (Tailscale, native messaging)
with an alternative approach used by [Iroh](https://github.com/n0-computer/iroh-examples/tree/main/browser-echo)
(compiling networking code to WebAssembly and running it entirely inside the browser).

### Side-by-Side: Where Does the Networking Code Run?

```
╔═══════════════════════════════════════════════════════════════════════════════╗
║              APPROACH A: NATIVE MESSAGING (this project)                     ║
║                                                                              ║
║  ┌─ Browser Sandbox ─────────────────────────┐                               ║
║  │                                           │                               ║
║  │  background.js          popup.js          │                               ║
║  │  ┌───────────────┐     ┌─────────────┐    │                               ║
║  │  │ Proxy config  │     │ Toggle UI   │    │                               ║
║  │  │ Status relay  │     │ Status text │    │                               ║
║  │  └───────┬───────┘     └─────────────┘    │                               ║
║  │          │                                │                               ║
║  │          │  No networking logic here.      │                               ║
║  │          │  JS is just a remote control.   │                               ║
║  │          │                                │                               ║
║  │          │ chrome.runtime.connectNative() │                               ║
║  └──────────┼────────────────────────────────┘                               ║
║             │  stdin/stdout pipe                                              ║
║             │  (length-prefixed JSON)                                         ║
║  ┌──────────▼────────────────────────────────┐                               ║
║  │                                           │                               ║
║  │  ts-browser-ext (Go binary)               │                               ║
║  │  RUNS ON THE OS, OUTSIDE THE SANDBOX      │                               ║
║  │                                           │                               ║
║  │  • tsnet.Server (full Tailscale node)     │                               ║
║  │  • WireGuard tunnel (raw UDP)             │                               ║
║  │  • TCP listener on 127.0.0.1              │                               ║
║  │  • HTTP + SOCKS5 proxy server             │                               ║
║  │  • Filesystem access (state storage)      │                               ║
║  │  • Full OS networking privileges          │                               ║
║  │                                           │                               ║
║  └───────────────────────────────────────────┘                               ║
║                                                                              ║
╚══════════════════════════════════════════════════════════════════════════════╝


╔═══════════════════════════════════════════════════════════════════════════════╗
║              APPROACH B: WEBASSEMBLY (Iroh browser-echo)                     ║
║                                                                              ║
║  ┌─ Browser Sandbox ─────────────────────────────────────────────────────┐   ║
║  │                                                                       │   ║
║  │  main.js (UI + glue)                                                  │   ║
║  │  ┌──────────────────┐                                                 │   ║
║  │  │ Form handling    │                                                 │   ║
║  │  │ Event display    │                                                 │   ║
║  │  │ DOM updates      │                                                 │   ║
║  │  └────────┬─────────┘                                                 │   ║
║  │           │  JS calls into WASM                                       │   ║
║  │           ▼                                                           │   ║
║  │  ┌─────────────────────────────────────────────────────────────────┐  │   ║
║  │  │                                                                 │  │   ║
║  │  │  browser-echo.wasm (Rust compiled to WebAssembly)               │  │   ║
║  │  │  ALL NETWORKING LOGIC RUNS HERE, INSIDE THE BROWSER             │  │   ║
║  │  │                                                                 │  │   ║
║  │  │  ┌────────────┐  ┌────────────────┐  ┌──────────────────────┐  │  │   ║
║  │  │  │ EchoNode   │  │ iroh::Endpoint │  │ Protocol handler    │  │  │   ║
║  │  │  │ (app logic)│─►│ (QUIC over     │─►│ (echo: read→write)  │  │  │   ║
║  │  │  │            │  │  WebTransport) │  │                     │  │  │   ║
║  │  │  └────────────┘  └────────────────┘  └──────────────────────┘  │  │   ║
║  │  │                                                                 │  │   ║
║  │  │  Returns Web Streams API ReadableStream to JS                   │  │   ║
║  │  └─────────────────────────────┬───────────────────────────────────┘  │   ║
║  │                                │                                      │   ║
║  │                                │  Browser-native networking only      │   ║
║  │                                │  (WebTransport / WebSocket / fetch)  │   ║
║  └────────────────────────────────┼──────────────────────────────────────┘   ║
║                                   │                                          ║
║               NO NATIVE PROCESS. NOTHING OUTSIDE THE SANDBOX.                ║
║                                                                              ║
╚══════════════════════════════════════════════════════════════════════════════╝
```

### How Each Approach Handles the Browser Sandbox

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                    THE BROWSER SANDBOX PROBLEM                               │
│                                                                              │
│  Browser JS/WASM CANNOT:              Both projects need to:                 │
│  ├─ Open raw UDP sockets              ├─ Establish encrypted P2P tunnels     │
│  ├─ Open raw TCP listeners            ├─ Route traffic through those tunnels │
│  ├─ Intercept DNS queries             ├─ Resolve custom hostnames            │
│  ├─ Access the filesystem             └─ Persist cryptographic identity      │
│  └─ Spawn OS processes                                                       │
│                                                                              │
│                     How each project escapes these constraints:               │
│                                                                              │
│  ┌─ Tailscale ─────────────────────┐  ┌─ Iroh ──────────────────────────┐   │
│  │                                 │  │                                  │   │
│  │  ESCAPE THE SANDBOX             │  │  STAY INSIDE THE SANDBOX         │   │
│  │                                 │  │                                  │   │
│  │  Use Native Messaging to run    │  │  Redesign the protocol to only   │   │
│  │  a full OS process that has     │  │  use APIs the browser provides:  │   │
│  │  no restrictions.               │  │                                  │   │
│  │                                 │  │  Raw UDP ──► WebTransport (QUIC) │   │
│  │  Raw UDP? ──► Go binary can.    │  │  TCP listen ──► not needed       │   │
│  │  TCP listen? ──► Go binary can. │  │  DNS ──► connect by node ID      │   │
│  │  DNS? ──► Go binary can.        │  │  Filesystem ──► IndexedDB        │   │
│  │  Filesystem? ──► Go binary can. │  │                                  │   │
│  │                                 │  │  Trade-off: can only tunnel       │   │
│  │  Trade-off: requires native     │  │  connections the WASM code       │   │
│  │  binary install on the machine. │  │  explicitly opens. Cannot proxy  │   │
│  │  Desktop-only.                  │  │  all browser traffic.            │   │
│  │                                 │  │  Works everywhere, even mobile.  │   │
│  └─────────────────────────────────┘  └──────────────────────────────────┘   │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

### Traffic Flow Comparison

```
 TAILSCALE — every browser request gets tunneled:
 ════════════════════════════════════════════════

  Browser tab                                              Tailnet peer
  types URL ──► chrome.proxy routes ──► 127.0.0.1:<port> ──► WireGuard ──► peer
                ALL traffic to proxy     (Go process)        (raw UDP)

  ┌────────┐    ┌──────────────┐    ┌───────────────────┐    ┌──────────────┐
  │ Browser│───►│ Proxy Config │───►│ Go: HTTP/SOCKS5   │───►│  Peer node   │
  │  tab   │    │ (extension   │    │   proxy server    │    │  on tailnet  │
  │        │◄───│  configures) │◄───│   + WireGuard     │◄───│              │
  └────────┘    └──────────────┘    └───────────────────┘    └──────────────┘
                  in browser            on the OS               on the internet
                  sandbox               (unsandboxed)



 IROH — only explicit app connections get tunneled:
 ══════════════════════════════════════════════════

  WASM module                                              Remote peer
  calls connect() ──► iroh::Endpoint ──► WebTransport ──► relay/direct ──► peer

  ┌────────────────────────────────────────────────┐    ┌──────────────┐
  │ Browser                                        │    │              │
  │                                                │    │  Remote      │
  │  ┌─────────┐    ┌───────────────────────────┐  │    │  peer        │
  │  │ main.js │───►│ WASM: iroh::Endpoint      │──┼───►│  (also       │
  │  │  (UI)   │◄───│  connect(node_id, payload)│◄─┼────│   running    │
  │  └─────────┘    └───────────────────────────┘  │    │   iroh)      │
  │                                                │    │              │
  │  Normal browser requests (google.com etc)      │    └──────────────┘
  │  go directly to the internet, NOT tunneled.    │
  │                                                │
  └────────────────────────────────────────────────┘
     everything inside the browser sandbox
```

### Capability Comparison

```
┌──────────────────────┬─────────────────────────────┬────────────────────────────────┐
│                      │  TAILSCALE                   │  IROH                          │
│                      │  (Native Messaging + Go)     │  (Rust → WebAssembly)          │
├──────────────────────┼─────────────────────────────┼────────────────────────────────┤
│ Code runs            │ OS process (outside sandbox)│ Inside browser (WASM sandbox) │
├──────────────────────┼─────────────────────────────┼────────────────────────────────┤
│ Transport            │ WireGuard (raw UDP)          │ QUIC over WebTransport         │
├──────────────────────┼─────────────────────────────┼────────────────────────────────┤
│ Proxies ALL browser  │ YES — chrome.proxy routes   │ NO — only app-level            │
│ traffic?             │ every request through it     │ connections opened by WASM     │
├──────────────────────┼─────────────────────────────┼────────────────────────────────┤
│ Custom DNS?          │ YES — MagicDNS via tsnet    │ NO — browser-native DNS only   │
├──────────────────────┼─────────────────────────────┼────────────────────────────────┤
│ Installation         │ TWO steps: extension +       │ ONE step: open a webpage       │
│                      │ native binary on machine     │ (WASM loads automatically)     │
├──────────────────────┼─────────────────────────────┼────────────────────────────────┤
│ Sandbox escape?      │ YES — native process has     │ NO — fully sandboxed           │
│                      │ full OS privileges           │                                │
├──────────────────────┼─────────────────────────────┼────────────────────────────────┤
│ Raw sockets?         │ YES (Go binary)              │ NO (WebTransport/WS/fetch)     │
├──────────────────────┼─────────────────────────────┼────────────────────────────────┤
│ State persistence    │ Filesystem                   │ Browser storage (IndexedDB)    │
├──────────────────────┼─────────────────────────────┼────────────────────────────────┤
│ Mobile browsers?     │ NO — native messaging        │ YES — any browser with         │
│                      │ unavailable on mobile         │ WASM + WebTransport            │
├──────────────────────┼─────────────────────────────┼────────────────────────────────┤
│ Security surface     │ Larger — native binary has   │ Smaller — constrained to       │
│                      │ full OS access               │ browser sandbox                │
└──────────────────────┴─────────────────────────────┴────────────────────────────────┘
```

### Why Tailscale Cannot Use the WASM Approach

```
  Tailscale requires:                       Why WASM can't provide it:
  ───────────────────                       ─────────────────────────

  WireGuard (raw UDP sockets)         ───►  Browser has no raw UDP API.
                                            WebTransport uses QUIC, which is a
                                            different protocol. You can't run
                                            WireGuard over WebTransport.

  Route ALL browser traffic           ───►  WASM can only open connections that
  through the VPN tunnel                    it initiates itself. It cannot
                                            intercept requests from the URL bar,
                                            other tabs, or other extensions.
                                            The chrome.proxy API requires a real
                                            TCP server to proxy to.

  MagicDNS (*.ts.net resolution)      ───►  WASM cannot intercept or override
                                            the browser's DNS resolution.
                                            Iroh sidesteps this entirely by
                                            connecting via node IDs, not hostnames.

  Persistent node identity stored     ───►  WASM has no filesystem access.
  on the filesystem                         Could use IndexedDB, but tsnet
                                            expects a real directory path.

  Iroh was designed around browser constraints from day one (QUIC/WebTransport).
  Tailscale's protocol predates browser support and is built on WireGuard,
  which fundamentally requires raw UDP — impossible inside a browser sandbox.
```

### Can Extensions Use WASM? (Yes, But It Doesn't Help)

A natural question: could you skip the native binary and put the WASM networking
code inside a browser extension instead of a webpage? Extensions **can** load WASM,
but WASM inside an extension has the same sandbox restrictions as WASM in a webpage.
It does not grant any new networking capabilities.

```
┌───────────────────────────────────────────────────────────────────────────┐
│              WASM Inside an Extension — What It Can and Cannot Do          │
│                                                                           │
│  ┌─ Extension with WASM ──────────────────────────────────────────────┐   │
│  │                                                                    │   │
│  │  background.js (service worker)                                    │   │
│  │  ┌─────────────────────────────────────────────────────────────┐   │   │
│  │  │  const wasm = await WebAssembly.instantiateStreaming(       │   │   │
│  │  │    fetch(chrome.runtime.getURL("iroh.wasm"))               │   │   │
│  │  │  );                                                        │   │   │
│  │  │                                                            │   │   │
│  │  │  iroh.wasm runs here ──► STILL INSIDE THE SANDBOX          │   │   │
│  │  │                                                            │   │   │
│  │  │  CAN:                          CANNOT:                     │   │   │
│  │  │  • Run Rust/C/Go code fast     • Open raw UDP/TCP sockets  │   │   │
│  │  │  • Do crypto, computation      • Listen on ports           │   │   │
│  │  │  • Call browser APIs via JS    • Access the filesystem     │   │   │
│  │  │  • Use WebTransport            • Bypass browser networking │   │   │
│  │  │  • Use fetch() / WebSocket     • Intercept DNS             │   │   │
│  │  │                                • Do anything JS can't do   │   │   │
│  │  │                                                            │   │   │
│  │  │  WASM is a COMPUTE sandbox. It lets you run native code    │   │   │
│  │  │  fast, but it does NOT escalate network privileges.        │   │   │
│  │  │  It has EXACTLY the same network access as JavaScript.     │   │   │
│  │  └─────────────────────────────────────────────────────────────┘   │   │
│  │                                                                    │   │
│  │  The extension platform adds:                                      │   │
│  │  • chrome.proxy API (route traffic through a proxy server)         │   │
│  │  • chrome.webRequest (observe/modify HTTP requests)                │   │
│  │  • Persistent background execution (service worker)                │   │
│  │  • chrome.storage (persistent key-value storage)                   │   │
│  │  • But still NO raw sockets, NO TCP port listeners                │   │
│  │                                                                    │   │
│  └────────────────────────────────────────────────────────────────────┘   │
│                                                                           │
└───────────────────────────────────────────────────────────────────────────┘
```

#### Extension + WASM vs Webpage + WASM vs Extension + Native

```
┌──────────────────────┬─────────────────────┬─────────────────────┬─────────────────────┐
│                      │ Iroh in a           │ Iroh in an          │ Tailscale            │
│                      │ WEBPAGE (WASM)      │ EXTENSION (WASM)    │ EXTENSION (native)   │
├──────────────────────┼─────────────────────┼─────────────────────┼──────────────────────┤
│ Connect to specific  │ YES                 │ YES                 │ YES                  │
│ peers                │                     │                     │                      │
├──────────────────────┼─────────────────────┼─────────────────────┼──────────────────────┤
│ Tunnel app-level     │ YES                 │ YES                 │ YES                  │
│ traffic              │                     │                     │                      │
├──────────────────────┼─────────────────────┼─────────────────────┼──────────────────────┤
│ Proxy ALL browser    │ NO                  │ NO                  │ YES                  │
│ traffic              │                     │                     │                      │
├──────────────────────┼─────────────────────┼─────────────────────┼──────────────────────┤
│ Persistent           │ NO — dies when      │ YES — service       │ YES — native process │
│ background           │ tab closes          │ worker persists     │ persists             │
├──────────────────────┼─────────────────────┼─────────────────────┼──────────────────────┤
│ Native binary        │ NO                  │ NO                  │ YES — must install   │
│ required?            │                     │                     │                      │
├──────────────────────┼─────────────────────┼─────────────────────┼──────────────────────┤
│ Raw sockets?         │ NO                  │ NO                  │ YES                  │
├──────────────────────┼─────────────────────┼─────────────────────┼──────────────────────┤
│ Custom DNS?          │ NO                  │ NO                  │ YES                  │
├──────────────────────┼─────────────────────┼─────────────────────┼──────────────────────┤
│ Mobile browsers?     │ YES                 │ limited (Firefox    │ NO                   │
│                      │                     │ Android only)       │                      │
└──────────────────────┴─────────────────────┴─────────────────────┴──────────────────────┘

  The middle column (extension + WASM) gains persistent background execution
  over the webpage approach, but gains NOTHING in terms of network capabilities.
  It still cannot proxy all browser traffic.
```

#### When Extension + WASM DOES Make Sense

```
  If you DON'T need to proxy all browser traffic and just want a
  persistent P2P node running in the background:

  ┌─ Hypothetical Iroh Extension (WASM, no native binary) ────────────┐
  │                                                                    │
  │  background.js (service worker)                                    │
  │  ┌──────────────────────────────────────────────────────────────┐  │
  │  │  iroh.wasm                                                   │  │
  │  │                                                              │  │
  │  │  • Persistent Iroh node running in the background            │  │
  │  │  • Connects to peers via WebTransport (QUIC)                 │  │
  │  │  • Syncs files, messages, or data between peers              │  │
  │  │  • Receives incoming connections from other Iroh nodes       │  │
  │  │  • Stores state in IndexedDB                                 │  │
  │  └──────────────────────────────────────────────────────────────┘  │
  │                                                                    │
  │  popup.js                                                          │
  │  ┌──────────────────────────────────────────────────────────────┐  │
  │  │  • Shows sync status, connected peers, transfer progress     │  │
  │  │  • No proxy configuration, no traffic interception           │  │
  │  │  • Just a P2P application that lives in the extension bar    │  │
  │  └──────────────────────────────────────────────────────────────┘  │
  │                                                                    │
  │  This works perfectly. Zero native install.                        │
  │  But it's a P2P app, not a VPN for all browser traffic.           │
  │                                                                    │
  └────────────────────────────────────────────────────────────────────┘

  Use cases where this is the right choice:
  • P2P file sharing extension
  • Decentralized messaging
  • Collaborative editing (CRDT sync)
  • Browser-to-browser data transfer

  Use cases where you MUST use native messaging instead:
  • VPN / tunnel all browser traffic        (needs TCP listener + chrome.proxy)
  • Custom DNS resolution                   (needs raw socket access)
  • WireGuard-based networking              (needs raw UDP)
  • System-level proxy                      (needs OS networking)
```


---

## Appendix B: Browser-Only Agent Architecture (OpenClaw-in-Extension)

What if you wanted to build an autonomous agent system (like OpenClaw) that
lives **entirely inside a browser extension** — no native host, no server?

### Diagram 1: The Best Possible Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         BROWSER EXTENSION (MV3)                             │
│                                                                             │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                     SERVICE WORKER (background.js)                    │  │
│  │                                                                       │  │
│  │  Lifecycle: event-driven, killed after ~5 min idle                    │  │
│  │  Woken by: chrome.alarms, Web Push, chrome.runtime messages           │  │
│  │                                                                       │  │
│  │  ┌─────────────────┐  ┌──────────────────┐  ┌─────────────────────┐  │  │
│  │  │  Agent Engine    │  │  Tool Executor    │  │  Task Scheduler     │  │  │
│  │  │                 │  │                  │  │                     │  │  │
│  │  │  • LLM calls    │  │  • fetch() to    │  │  • chrome.alarms    │  │  │
│  │  │    via fetch()  │  │    external APIs  │  │    (min 1 min)     │  │  │
│  │  │  • Plan/decide  │  │  • DOM injection  │  │  • Queues tasks in  │  │  │
│  │  │  • Orchestrate  │  │    via content    │  │    IndexedDB        │  │  │
│  │  │    multi-step   │  │    scripts        │  │  • Resumes after    │  │  │
│  │  │    workflows    │  │  • Tab control    │  │    worker restart   │  │  │
│  │  └────────┬────────┘  └────────┬─────────┘  └──────────┬──────────┘  │  │
│  │           │                    │                        │             │  │
│  │  ┌────────┴────────────────────┴────────────────────────┴──────────┐  │  │
│  │  │                    PERSISTENT STORAGE LAYER                      │  │  │
│  │  │                                                                  │  │  │
│  │  │  IndexedDB          OPFS                   chrome.storage.local  │  │  │
│  │  │  ┌──────────────┐   ┌──────────────────┐   ┌────────────────┐   │  │  │
│  │  │  │ Structured   │   │ File-like storage │   │ Key-value      │   │  │  │
│  │  │  │ data, queues,│   │ for large blobs,  │   │ for settings,  │   │  │  │
│  │  │  │ agent memory,│   │ conversation logs,│   │ state flags,   │   │  │  │
│  │  │  │ tool results │   │ WASM modules      │   │ sync across    │   │  │  │
│  │  │  │              │   │                   │   │ devices         │   │  │  │
│  │  │  │ (unlimited)  │   │ (unlimited)       │   │ (10 MB cap)    │   │  │  │
│  │  │  └──────────────┘   └──────────────────┘   └────────────────┘   │  │  │
│  │  │  ↑ All survive worker kills. State rehydrated on every wake.    │  │  │
│  │  └─────────────────────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                       CONTENT SCRIPTS                                 │  │
│  │                                                                       │  │
│  │  Injected into web pages on demand                                    │  │
│  │  • Read/modify DOM  • Extract page data  • Fill forms  • Click       │  │
│  │  • Communicate with service worker via chrome.runtime.sendMessage     │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                       POPUP / SIDE PANEL UI                           │  │
│  │                                                                       │  │
│  │  • Dashboard showing agent status, task queue, logs                    │  │
│  │  • Manual trigger for tasks                                           │  │
│  │  • Settings (API keys, schedules)                                     │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
└──────────────────────────────┬──────────────────────────────────────────────┘
                               │
              ┌────────────────┼────────────────────┐
              │                │                    │
              ▼                ▼                    ▼
   ┌──────────────────┐ ┌──────────────┐  ┌────────────────────┐
   │   LLM API        │ │ External APIs│  │  Push Relay        │
   │   (Claude, etc.) │ │ (GitHub,     │  │  (thin server)     │
   │                  │ │  Slack, etc.)│  │                    │
   │   fetch() ────►  │ │  fetch() ──► │  │  Webhook ──► Push  │
   │   ◄──── JSON     │ │  ◄── JSON    │  │  (translates       │
   └──────────────────┘ └──────────────┘  │  webhooks into      │
                                          │  Web Push messages  │
                                          │  that wake the      │
                                          │  service worker)    │
                                          └────────────────────┘

  INBOUND EVENT FLOW (how the outside world reaches the extension):

  External Event (e.g. GitHub webhook, new email)
       │
       ▼
  Push Relay Server (Cloudflare Worker / tiny VPS)
       │  Receives webhook, encrypts with VAPID,
       │  forwards to FCM (Chrome) or Mozilla Push (Firefox)
       ▼
  Browser Push Service (FCM / Mozilla)
       │
       ▼
  Service Worker wakes via 'push' event
       │  Reads payload, stores in IndexedDB,
       │  triggers agent logic
       ▼
  Agent processes event
```

### Diagram 2: Hard Limits — What This Architecture CANNOT Do

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      IMPOSSIBLE IN A BROWSER EXTENSION                      │
│                                                                             │
│  ╔═══════════════════════════════════════════════════════════════════════╗   │
│  ║  NO INBOUND CONNECTIONS                                             ║   │
│  ║                                                                     ║   │
│  ║  Cannot listen on a TCP/UDP port. No ServerSocket API exists.       ║   │
│  ║  Cannot accept incoming HTTP requests.                              ║   │
│  ║  Cannot run a web server, API server, or tunnel endpoint.           ║   │
│  ║                                                                     ║   │
│  ║  Workaround: Push Relay (see above) or polling via chrome.alarms    ║   │
│  ╚═══════════════════════════════════════════════════════════════════════╝   │
│                                                                             │
│  ╔═══════════════════════════════════════════════════════════════════════╗   │
│  ║  NO RAW SOCKETS                                                     ║   │
│  ║                                                                     ║   │
│  ║  No TCP, UDP, or ICMP sockets. Only fetch() and WebSocket.          ║   │
│  ║  Cannot implement WireGuard, custom DNS, or any wire protocol.      ║   │
│  ║  Cannot proxy traffic at the network level (no chrome.proxy target).║   │
│  ║                                                                     ║   │
│  ║  Workaround: None — requires native host for raw networking         ║   │
│  ╚═══════════════════════════════════════════════════════════════════════╝   │
│                                                                             │
│  ╔═══════════════════════════════════════════════════════════════════════╗   │
│  ║  NO PERSISTENT EXECUTION                                            ║   │
│  ║                                                                     ║   │
│  ║  Service worker killed after ~5 min idle (30 sec hard limit per     ║   │
│  ║  event in some browsers). Cannot run continuous background loops.   ║   │
│  ║  All in-memory state lost on kill. No cron-like sub-second timers.  ║   │
│  ║                                                                     ║   │
│  ║  Workaround: chrome.alarms (1 min minimum), Web Push to wake,      ║   │
│  ║  persist all state to IndexedDB/OPFS between events                 ║   │
│  ╚═══════════════════════════════════════════════════════════════════════╝   │
│                                                                             │
│  ╔═══════════════════════════════════════════════════════════════════════╗   │
│  ║  NO FILESYSTEM / OS ACCESS                                          ║   │
│  ║                                                                     ║   │
│  ║  Cannot read/write real files. No child processes. No OS APIs.      ║   │
│  ║  Cannot run shell commands, access clipboard persistently, or       ║   │
│  ║  interact with other desktop applications.                          ║   │
│  ║                                                                     ║   │
│  ║  Workaround: OPFS for virtual files; native messaging for OS       ║   │
│  ║  access (but then it's no longer "browser-only")                    ║   │
│  ╚═══════════════════════════════════════════════════════════════════════╝   │
│                                                                             │
│  ╔═══════════════════════════════════════════════════════════════════════╗   │
│  ║  NO CROSS-ORIGIN DOM ACCESS (without permission)                    ║   │
│  ║                                                                     ║   │
│  ║  Content scripts can only touch pages matching manifest patterns.   ║   │
│  ║  Cannot read iframes from other origins. Cannot bypass CORS.        ║   │
│  ║                                                                     ║   │
│  ║  Workaround: Declare broad host_permissions in manifest; use        ║   │
│  ║  fetch() from service worker (which bypasses CORS)                  ║   │
│  ╚═══════════════════════════════════════════════════════════════════════╝   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘

  SUMMARY: A browser extension can be a capable outbound agent (call APIs,
  scrape pages, automate tabs) with durable state (IndexedDB, OPFS) and
  wake-on-push (Web Push API). It CANNOT be a server, a tunnel, or a
  continuously running daemon.
```
