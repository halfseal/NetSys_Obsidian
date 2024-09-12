---
creator: 김명수
type: Hands-On
created: 2022-09-05
---
<<[[eRPC Step by Step|previous]]>>

![[Untitled(141).png]]


## Nexus

```cpp
12   erpc::Nexus nexus(client_uri);
```

- Nexus is identified by port number(same hostname)
- Request handler function is registerd in nexus
- RPC endpoint is identified by rpc_id
![[Untitled (1)(1).png]]


- There is valid range of eRpc udp port number
    
    - **kBaseSmUdpPort ≤ udp_port < kBaseSmUdpPort + kMaxNumERpcProcesses**
        
        **kBaseSmUdpPort = 31850, kMaxNumERPCProcesses = 32**
        
- Hook will be explained later…..
    
- ⇒Kind of connection between of Nexus and RPC endpoint
    
- ⇒RPC is registered to Nexus through hook
    
![[Untitled (2)(1).png]]


- Select target hook using **target_rpc_id**

## Request handler

- Register request handler must be done **before** any RPC registers a **hook** with the Nexus

```cpp
 14   nexus.register_req_func(kReqType, req_handler);
```
![[Untitled (3)(1).png]]


- Create **ReqFunc** object which is request handler function and register it in **req_func_arr_** indexed by **req_type**
- Server must has **request handler** corresponding to **kReqType**
- Multi request handlers can be registered in same Nexus
![[Untitled (4)(1).png]]


- ReqFunc object has **request handler function**

### RPC end-point

```cpp
 /* client */
 14   rpc = new erpc::Rpc<erpc::CTransport>(&nexus, nullptr, 0, sm_handler);

 /* server */
 14   nexus.register_req_func(kReqType, req_handler);
 15 
 16   rpc = new erpc::Rpc<erpc::CTransport>(&nexus, nullptr, 0, nullptr);

 17   int session_num = rpc->create_session(server_uri, 0);
```
![[Untitled (5).png]]


- RPC endpoint is connected to Nexus through **hook**
- RPC register hook using its **rpc_id** for ****Nexus
![[Untitled (6)(1).png]]


- In RPC constructor, RPC register hook for Nexus
![[Untitled (7)(1).png]]


- After RPC register hook for Nexus, **disallow register request handler function** at that Nexus
- Nexus can has multiple RPC endpoint
![[Untitled (8)(1).png]]

![[Untitled(142).png]]

![[Untitled(143).png]]


## Request(Client)

```jsx
24   rpc->enqueue_request(session_num, kReqType, &req, &resp, cont_func, nullptr); 
/*client */
```
![[Untitled(144).png]]


- Set dest_session_num known by session handshakes
- Get session object in session vector idexed by session_num
- If there is no avail credit ⇒ push sslot in stallq
- Then process stall queue in event loop’s process_credit_stall_queue_st
- Why compare to 0?
- Push in stallq only when there is no credit
![[Untitled(145).png]]


- Session Slot class
![[Untitled(146).png]]


- Client side info
- cont_func_ for client side response function
![[Untitled(147).png]]


- Server side info
    
- req_type for request handler function
    
- Session class
    
![[Untitled(148).png]]

![[Untitled (1)(2).png]]


- Session has **credits** which included in **client_info_**
    
- default = 32
    
- pkthdr.h
    
    ```cpp
    struct pkthdr_t {
      static_assert(kHeadroom == 0 || kHeadroom == 40, "");
      uint8_t headroom_[kHeadroom + 2];  /// Ethernet L2/L3/L3 headers
    
      // Fit one set of fields in six bytes, left over after (kHeadroom + 2) bytes.
      // On MSVC, these fields cannot use uint64_t. In the future, we can increase
      // the bits for msg_size_ by shrinking req_type_.
    
      uint32_t req_type_ : 8;             /// RPC request type
      uint32_t msg_size_ : kMsgSizeBits;  /// Req/resp msg size, excluding headers
      uint16_t dest_session_num_;  /// Session number of the destination endpoint
    
      // The next set of fields goes in eight bytes total
    
      uint64_t pkt_type_ : 2;           /// The eRPC packet type
      uint64_t pkt_num_ : kPktNumBits;  /// Monotonically increasing packet number
    
      /// Request number, carried by all data and control packets for a request.
      uint64_t req_num_ : kReqNumBits;
      uint64_t magic_ : k_pkt_hdr_magic_bits;  ///< Magic from alloc_msg_buffer()
    
      /// Fill in packet header fields
      void format(uint64_t _req_type, uint64_t _msg_size,
                  uint64_t _dest_session_num, uint64_t _pkt_type, uint64_t _pkt_num,
                  uint64_t _req_num) {
        req_type_ = _req_type;
        msg_size_ = _msg_size;
        dest_session_num_ = _dest_session_num;
        pkt_type_ = _pkt_type;
        pkt_num_ = _pkt_num;
        req_num_ = _req_num;
        magic_ = kPktHdrMagic;
      }
    ......
    }
    ```
    
![[Untitled (2)(3).png]]


- select number of sending packets to min(credits, num_pkts_ - num_tx)
    - num_pkts_ : number of packets in msgbuf
    - num_tx : number of packets that client sent
- Decrease client credits by 1
- Remained pkts are sent at event loop…?
![[Untitled (3)(2).png]]


- collect and send
![[Untitled (4)(2).png]]


- while running event loop in client
![[Untitled(149).png]]

![[Untitled (6)(2).png]]

![[Untitled(150).png]]

![[Untitled (5) (1).png]]


- wire_pkts returns total number of packets sent by one RPC endpoint
- req_pkts_pending returns true if there is more request packets to send

## Response(server)

### -RX

- Receive packets while running **run_event_loop_do_one_st()**
![[Untitled(151).png]]

![[Untitled(152).png]]


- Received packets’(client’s requests) PktType is **kReq**
- In common, allocate msgbuffer and copy packet’s data to msgbuffer
- Run request handler function is indicated by req_type

## Processing small request(one packet)
![[Untitled(153).png]]


## Processing large request(more than one packet)
![[Untitled(154).png]]


- Send credit return mesg and run request_handler_function

## Credit return(server send)

- Credit return mesg is sent to client
![[Untitled (1)(4).png]]


- PktType of credit return mesg is **“kExplCR”**
![[Untitled (2)(4).png]]


## Credit return(client receive)

- While running event_loop
- RX
![[Untitled(155).png]]

![[Untitled(156).png]]

![[Untitled (3)(3).png]]

![[Untitled (4)(3).png]]


## Receive RFR(Server)
![[Untitled(157).png]]

![[Untitled(158).png]]

![[Untitled(159).png]]


### Enqueue response(called by reuest handler)

```cpp
  4 void req_handler(erpc::ReqHandle *req_handle, void *) {
  5   auto &resp = req_handle->pre_resp_msgbuf_;
  6   rpc->resize_msg_buffer(&resp, kMsgSize);
  7   sprintf(reinterpret_cast<char *>(resp.buf_), "hello");
  8   rpc->enqueue_response(req_handle, &resp);
  9 }
```
![[Untitled(160).png]]

![[Untitled(161).png]]

![[Untitled(162).png]]

![[Untitled(163).png]]


batch만큼 안 쌓일 경우 event_loop의 뒷 부분에서 처리