---
creator: 김명수
type: Hands-On
created: 2022-08-23
---
<<[[eRPC Step By Step Further|next]]>>
## Code

sever.cc

```cpp
	1 #include "common.h"
  2 erpc::Rpc<erpc::CTransport> *rpc;
  3 
  4 void req_handler(erpc::ReqHandle *req_handle, void *) {
  5   auto &resp = req_handle->pre_resp_msgbuf_;
  6   rpc->resize_msg_buffer(&resp, kMsgSize);
  7   sprintf(reinterpret_cast<char *>(resp.buf_), "hello");
  8   rpc->enqueue_response(req_handle, &resp);
  9 }
 10 
 11 int main() {
 12   std::string server_uri = kServerHostname + ":" + std::to_string(kUDPPort);
 13   erpc::Nexus nexus(server_uri);
 14   nexus.register_req_func(kReqType, req_handler);
 15 
 16   rpc = new erpc::Rpc<erpc::CTransport>(&nexus, nullptr, 0, nullptr);
 17   rpc->run_event_loop(100000);
 18 }

```

1. Create Nexus object and register request handler(13-14)
2. Create RPC end point(16)
3. Run event loop(17)
4. Process request, make response and send to client(8)

client.cc

```cpp
	1 #include "common.h"
  2 erpc::Rpc<erpc::CTransport> *rpc;
  3 erpc::MsgBuffer req;
  4 erpc::MsgBuffer resp;
  5 
  6 void cont_func(void *, void *) { printf("%s\\n", resp.buf_); }
  7 
  8 void sm_handler(int, erpc::SmEventType, erpc::SmErrType, void *) {}
  9 
 10 int main() {
 11   std::string client_uri = kClientHostname + ":" + std::to_string(kUDPPort);
 12   erpc::Nexus nexus(client_uri);
 13 
 14   rpc = new erpc::Rpc<erpc::CTransport>(&nexus, nullptr, 0, sm_handler);
 15 
 16   std::string server_uri = kServerHostname + ":" + std::to_string(kUDPPort);
 17   int session_num = rpc->create_session(server_uri, 0);
 18 
 19   while (!rpc->is_connected(session_num)) 
 20		    rpc->run_event_loop_once();
 21 
 22   req = rpc->alloc_msg_buffer_or_die(kMsgSize);
 23   resp = rpc->alloc_msg_buffer_or_die(kMsgSize);
 24 
 25   rpc->enqueue_request(session_num, kReqType, &req, &resp, cont_func, nullptr);
 26   rpc->run_event_loop(100);
 27 
 28   delete rpc;
 29 }
```

1. Create RPC end point(12-14)
2. Create session to connect to remote sever(17-19)
3. Allocate buffer for request, response msg(21-22)
4. Send request(24)
5. Run event loop(25)

⇒Run “**cont_func”** by response(24→6)

## Create Session

```cpp
 17   int session_num = rpc->create_session(server_uri, 0); /* client */
```
![[Untitled (2).png]]


- “**remote_uri**” is formatted as “**hostname:udp_port**”
    
- “**remote_rpc_id**” is ID of the remote RPC obj
    
- In function “create_session_st”
    
![[Untitled (3).png]]

![[Untitled (4).png]]


-”session credit” is for packet-level flow control

-check session credits to know whether there is available ring entries

-limit the number of packets that client can send before receiving a reply

- Fill client, server endpoint metadata
![[Untitled(128).png]]

![[Untitled (6).png]]


- Session creation packet ⇒ SM(session Management)
- Make session management pkt & send to server
    - Set SmPktType to ‘kConnectReq’

```cpp
 17   rpc->run_event_loop(100000); /* server */
```

run_event_loop ⇒ run_event_loop_timeout_st
![[Untitled(129).png]]


- Call run_event_loop_do_one_st()
![[Untitled (8).png]]

![[Untitled (9).png]]


- SmPktType that Client send to Server is ‘kConnectReq’
    - ⇒ Call “handle_connect_req_st”

handle_connect_req_st()
![[Untitled (10).png]]


- Server create new Session to communicates with Client
- Make response and send to Client
    - Set session management pkt type **request** to **response**
![[Untitled(130).png]]


```cpp
 19   while (!rpc->is_connected(session_num)) 
 20       rpc->run_event_loop_once(); /* client */
```

- is_connected() returns TRUE if session state of ‘session_num’ is ‘kConnected’
    
    - if session state is not kConnected, call run_event_loop_once()
    - example of session types: kConnected, kConnectInProgress…
    ![[Untitled (7).png]]
    ![[Untitled (9)(1).png]]
    
    
- SmPktType is kConnectResp, so “**handle_connect_resp_st()”** will be called
    
![[Untitled(131).png]]


- Set session state to kConnected
- Can exit while loop

⇒Something like handshakes

## Allocate Msg Buffer

```cpp
 21   req = rpc->alloc_msg_buffer_or_die(kMsgSize); /* client */
 22   resp = rpc->alloc_msg_buffer_or_die(kMsgSize);
```
![[Untitled(132).png]]


- Allocated size is ‘requested allocate data size + (packet header size) * max #packets’

## Enqueue_request

- InProgress….

## Credit