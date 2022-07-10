---
title: 'Websocket is not working behind AWS Load balancer?'
date: '2022-06-28'
---

Recently I set up a internal tool for engineers inside our company to run one-off jobs
without going to terminal. It is a simple FastAPI service, and use websocket to send 
live logs to the UI. 
The traffic path is below
```
cloudflare -> AWS classic LB -> Nginx ingress -> buzzfeed sso -> istio -> uvicorn -> fastAPI app
```
Here both `Nginx` and `buzzfeed sso` are reverse proxy and `buzzfeed sso` act as the authentication layer.
The problem is that I can see live log locally, but it does not work after I push to production.

### First, how I know there is a problem, and what problem it is?
When I ran a one-off job, the server log shows an entry 
```
INFO:     127.0.0.6:60295 - "GET /ws/abcdefghij HTTP/1.1" 404 Not Found
```
, so I know the request came in but somehow router 
does not recognize it. Also, I wonder why it is HTTP instead of websocket.
The ASGI standard `async def application(scope, receive, send):` says the first
argument `scope` has a `type` key that indicates the protocol name. I read
starlette source code and confirm it should be `websocket`. It makes no sense
to receive a `HTTP` record.

Meanwhile, I suspect this HTTP request is probably the websocket handshake 
message, but I did not see log `upgrade to websocket`.

It took me a while to find a way to see what was going on inside 
`uvicorn + fastAPI`: **monkey patch**.
```python
def new_data_received(self, data: bytes) -> None:
    self._unset_keepalive_if_required()

    try:
        logger.info(f"xxx data {data.decode('utf-8')}")
        self.parser.feed_data(data)
    except httptools.HttpParserError:
        msg = "Invalid HTTP request received."
        self.logger.warning(msg)
        self.send_400_response(msg)
        return
    except httptools.HttpParserUpgrade:
        self.handle_upgrade()


uvicorn.protocols.http.httptools_impl.HttpToolsProtocol.data_received = (
    new_data_received
)
```
What I see is below
```
GET /ws/b06df69c-2341-476f-b857-58c3f690fd57 HTTP/1.1
host: one-off-master.staging-oneoff.svc.cluster.local:8004
user-agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/102.0.0.0 Safari/537.36
accept-encoding: gzip
accept-language: en-US,en;q=0.9,zh-CN;q=0.8,zh;q=0.7,es;q=0.6,ru;q=0.5,fr;q=0.4
cache-control: no-cache
cdn-loop: cloudflare
cf-connecting-ip: 98.227.7.99
cf-ipcountry: US
cf-ray: 7212afd23c762bff-ORD
cf-visitor: {"scheme":"https"}
cookie: session=<...>
origin: https://staging-oneoff.tryevergreen.com
pragma: no-cache
sec-websocket-extensions: permessage-deflate; client_max_window_bits
sec-websocket-key: bIOl9pXidSou2h391fMsQg==
sec-websocket-version: 13
x-forwarded-email: xiong@ziphq.com
x-forwarded-for: 172.31.93.191, 127.0.0.6
x-forwarded-groups: 
x-forwarded-host: staging-oneoff.tryevergreen.com
x-forwarded-host: staging-oneoff.tryevergreen.com
x-forwarded-port: 80
x-forwarded-proto: http
x-forwarded-user: xiong
x-original-forwarded-for: 98.227.7.99, 172.70.130.172
x-real-ip: 172.31.93.191
x-request-id: c2b936fa4e66077ab0573418c9b86b2a
x-scheme: http
x-envoy-attempt-count: 1
x-forwarded-client-cert: <...>
x-b3-traceid: 40e4f85b4d4a7e6d345892d4ba4eb999
x-b3-spanid: 553262c905b63a40
x-b3-parentspanid: ebde97693103cd06
x-b3-sampled: 0

```
Ah!. The message is indeed the handshake request, but `Upgrade` and `Connection`
headers are missing. Where were they dropped?

### How do I know istio is not the problem?
I first suspect Istio dropped the websocket header because the sso services 
are running in different EKS cluster than our fastAPI server.
Also, I see a few posts online complaining websocket support in Istio.
Then I quickly tested the hypnosis.

The first thing I tried is using a python library called `websockets`. I 
created a new pod running a python image in the sso cluster and then run below
command.
```bash
python -m websockets ws://one-off-master.staging-oneoff.svc.cluster.local:8004/ws/9adef6d1-0a43-4a9a-928c-7cc582f90051
```
It works.
Later on, I found a even simpler way: just enter the sso-proxy pod and run curl
below
```bash
curl -i -X GET \
-H "Connection: Upgrade" \
-H "Upgrade: websocket" \
-H "sec-websocket-extensions: permessage-deflate; client_max_window_bits" \
-H "sec-websocket-key: bIOl9pXidSou2h391fMsQg==" -H "sec-websocket-version: 13" \
-H "accept-encoding: gzip" \
-H "cdn-loop: cloudflare" \
one-off-master.staging-oneoff.svc.cluster.local:8004/ws/abcdefghijk
```
A 101 response was received and terminal hang there for input/output, so 
it means websocket handshake succeeded and a TCP connnection is `hijacked` 
for me. `Hijack` is a word I learned from golang. See `http.Hijacker`.


### How I know sso is not the problem
Now I suspect sso dropped the websocket headers.

First, I spent lots of time reading 
[sso source code](https://github.com/buzzfeed/sso). The code is not super 
complicated. The proxy part is basically a wrapper around golang standard 
module [reverseproxy.go](https://go.dev/src/net/http/httputil/reverseproxy.go).
Golang reverseproxy supports websocket natively. It clearly adds the headers 
back after dropping them.
```go
	// After stripping all the hop-by-hop connection headers above, add back any
	// necessary for protocol upgrades, such as for websockets.
	if reqUpType != "" {
		outreq.Header.Set("Connection", "Upgrade")
		outreq.Header.Set("Upgrade", reqUpType)
	}
```
So sso should not be the culprit. If you wonder why reverse proxy drops 
them first,
you can google `hop-by-hop` headers.

The second proof comes from below Nginx study.

### How do I know Nginx is not the problem?
I went to the Nginx pod and find a section in `/etc/nginx/nginx.conf`
```
# Allow websocket connections
proxy_set_header                        Upgrade           $http_upgrade;
proxy_set_header                        Connection        $connection_upgrade;
```
Therefore, Nginx should not be a problem. 
Meanwhile, running below request in the Nginx pod got a 101 response. 

```bash
curl -i -X GET \
-H "Connection: Upgrade" \
-H "Upgrade: websocket" \
-H "sec-websocket-extensions: permessage-deflate; client_max_window_bits" \
-H "sec-websocket-key: bIOl9pXidSou2h391fMsQg==" -H "sec-websocket-version: 13" \
-H "accept-encoding: gzip" \
-H "cdn-loop: cloudflare" \
-H 'Host: staging-oneoff.tryevergreen.com' \
localhost:80/ws/abcdefghij
```
It means Nginx correctly sent out the headers to sso and sso correctly 
forwarded them to Istio. There is no problem in this chain 
`Nginx -> sso -> Istio -> uvicorn -> fastAPI`.

A hack needs to be mentioned is I added `skip_auth_regex` to sso-proxy 
config to bypass the authentication step. Without it, above request will be 
redirected to `sso-oauth` service and asks me to login.

### Now I know LB or cloudflare is the problem. 

LB or cloudflare? Luckily, I found this post 
https://github.com/kubernetes/ingress-nginx/issues/3746

How do I verify it is LB problem, same as above, we can directly curl the 
load balancer. First, we need to find the LB DNS or A record. See below
```
% k get services
NAME                                                TYPE           CLUSTER-IP       EXTERNAL-IP                                                               PORT(S)                        AGE
nginx-internal-ingress-nginx-controller             LoadBalancer   10.100.37.123    ab4689a0641b74457a48a8575ddf7cf5-1999473190.us-east-2.elb.amazonaws.com   80:30931/TCP,443:32259/TCP     579d
```
Then we can replace `localhost` in above curl command with
`ab4689a0641b74457a48a8575ddf7cf5-1999473190.us-east-2.elb.amazonaws.com`.
OK. I reproduced the error message in the service log.
```
INFO:     127.0.0.6:60295 - "GET /ws/abcdefghij HTTP/1.1" 404 Not Found
```

After change the load balancer from classic to NLB, websocket starts working. 
