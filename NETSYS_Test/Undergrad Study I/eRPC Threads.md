---
creator: 황준용
type: Hands-On
created: 2022-09-26
---
Erpc - Threads:

### _Worker Threads vs Dispatch Thread_

_**When request handlers are run directly in dispatch threads you can avoid expensive inter-thread communication** (adding up to 400ns of request latency). That’s fine when request handlers are short in duration, but **long handlers block other dispatch handling increasing tail latency, and prevent rapid congestion feedback.**_

_eRPC supports running handlers in dispatch threads for short duration request types (up to a few hundred nanoseconds), and worker threads for longer running requests. Which mode to use is specified when the request handler is registered. This is the only additional user input needed in eRPC._

> Striking a balance, eRPC allows running request handlers in both dispatch threads and worker threads: When registering a request handler, the programmer species whether the handler should run in a dispatch thread. This is the only additional user input required in eRPC. **In When registering a request handler, the programmer species whether the handler should run in a dispatch thread. This is the only additional user input required in eRPC. In typical use cases, handlers that require up to a few hundred nanoseconds use dispatch threads, and longer handlers use worker threads.**

⇒ Main Viewpoints

1. **Work distribution between Dispatch Thread & Worker Thread**
2. **How Thread functions actually Invoke / utilize thread properties.**

```cpp
// eRPC/src/rpc.h

/**
 * @brief An Rpc object is the main communication end point in eRPC.
 * Applications use it to create sessions with remote Rpc objects, send and
 * receive requests and responses, and run the event loop.
 *
 *** None of the functions are thread safe for user threads.
 *
 * eRPC's worker (background) threads have limited; concurrent access to Rpc
 * objects. The functions with the _st can be called from only the foreground
 * thread that owns the Rpc object.**
 *
 * @tparam TTr The unreliable transport
 */
```

- Thread safe
    
    In multi-threaded programming, thread safety generally means that a function, variable, or object is accessed from multiple threads at the same time, but there is no problem running a program. More precisely, when one function is called from one thread and is running, it is defined that the result of the function's performance in each thread comes out correctly, even if the other thread calls the function and runs together at the same time.
    

```cpp
template <class TTr>
void Rpc<TTr>::handle_connect_req**_st**(const SmPkt &sm_pkt) {
  **assert(in_dispatch());**
  assert(sm_pkt.pkt_type_ == SmPktType::kConnectReq &&
         sm_pkt.server_.rpc_id_ == rpc_id_);

  char issue_msg[kMaxIssueMsgLen];  // The basic issue message
  sprintf(issue_msg, "Rpc %u: Received connect request from %s. Issue", rpc_id_,
          sm_pkt.client_.name().c_str());

  // Handle duplicate session connect requests
  if (conn_req_token_map_.count(sm_pkt.uniq_token_) > 0) {
    uint16_t srv_session_num = conn_req_token_map_[sm_pkt.uniq_token_];
    assert(session_vec_.size() > srv_session_num);

    const Session *session = session_vec_[srv_session_num];
    if (session == nullptr || session->state_ != SessionState::kConnected) {
      ERPC_INFO("%s: Duplicate request, and response is unneeded.\\n",
                issue_msg);
      return;
    } else {
      SmPkt resp_sm_pkt = sm_construct_resp(sm_pkt, SmErrType::kNoError);
      resp_sm_pkt.server_ = session->server_;  // Re-send server endpoint info

      ERPC_INFO("%s: Duplicate request. Re-sending response.\\n", issue_msg);
      sm_pkt_udp_tx_st(resp_sm_pkt);
      return;
    }
  }

...
```

```cpp
/// Return true iff we're currently running in this Rpc's creator thread
  inline bool in_dispatch() const { return get_etid() == creator_etid_; }
```

Worker Thread ⇒ Background Thread

Dispatch Thread ⇒ Session Management Thread

---

Architecture

Background Thread

1. In Erpc, when making a nexus, background threads are launched.
2. Background threads are created with 1.bg_thread_func, and 2.bg_thread_ctx

```cpp
//eRPC/src/nexus/impl/nexus.cc

namespace erpc {

Nexus::Nexus(std::string local_uri, size_t numa_node, size_t num_bg_threads)
    : freq_ghz_(measure_rdtsc_freq()),
      hostname_(extract_hostname_from_uri(local_uri)),
      sm_udp_port_(extract_udp_port_from_uri(local_uri)),
      numa_node_(numa_node),
      num_bg_threads_(num_bg_threads),
      heartbeat_mgr_(hostname_, sm_udp_port_, freq_ghz_,
                     kMachineFailureTimeoutMs) {
  if (kTesting) {
    ERPC_WARN("eRPC Nexus: Testing enabled. Perf will be low.\\n");
  }

  rt_assert(sm_udp_port_ >= kBaseSmUdpPort &&
                sm_udp_port_ < (kBaseSmUdpPort + kMaxNumERpcProcesses),
            "Invalid management UDP port");
  rt_assert(num_bg_threads <= kMaxBgThreads, "Too many background threads");
  rt_assert(numa_node < kMaxNumaNodes, "Invalid NUMA node");

  kill_switch_ = false;

  **// Launch background threads
  ERPC_INFO("eRPC Nexus: Launching %zu background threads.\\n", num_bg_threads);
  for (size_t i = 0; i < num_bg_threads; i++) {
    assert(tls_registry_.cur_etid_ == i);

    BgThreadCtx bg_thread_ctx;
    bg_thread_ctx.kill_switch_ = &kill_switch_;
    bg_thread_ctx.req_func_arr_ = &req_func_arr_;
    bg_thread_ctx.tls_registry_ = &tls_registry_;
    bg_thread_ctx.bg_thread_index_ = i;
    bg_thread_ctx.bg_req_queue_ = &bg_req_queue_[i];

    bg_thread_arr_[i] = std::thread(bg_thread_func, bg_thread_ctx);

    // Wait for the launched thread to grab a eRPC thread ID, otherwise later
    // background threads or the foreground thread can grab ID = i.
    while (tls_registry_.cur_etid_ == i) {
      std::this_thread::sleep_for(std::chrono::microseconds(1));
    }
  }
	// Launch the session management thread
  SmThreadCtx sm_thread_ctx;
  sm_thread_ctx.hostname_ = hostname_;
  sm_thread_ctx.sm_udp_port_ = sm_udp_port_;
  sm_thread_ctx.freq_ghz_ = freq_ghz_;
  sm_thread_ctx.kill_switch_ = &kill_switch_;
  sm_thread_ctx.heartbeat_mgr_ = &heartbeat_mgr_;
  sm_thread_ctx.reg_hooks_arr_ = const_cast<volatile Hook **>(reg_hooks_arr_);
  sm_thread_ctx.reg_hooks_lock_ = &reg_hooks_lock_;

  // Bind the session management thread to the last lcore on numa_node
  size_t sm_thread_lcore_index = num_lcores_per_numa_node() - 1;
  ERPC_INFO("eRPC Nexus: Launching session management thread on core %zu.\\n",
            get_lcores_for_numa_node(numa_node).at(sm_thread_lcore_index));
  sm_thread_ = std::thread(sm_thread_func, sm_thread_ctx);
  bind_to_core(sm_thread_, numa_node, sm_thread_lcore_index);

  ERPC_INFO("eRPC Nexus: Created with management UDP port %u, hostname %s.\\n",
            sm_udp_port_, hostname_.c_str());
}**
```

- num_bg_threads
    
    No initialization nor incremental.
    
    ```
    public:
      Nexus(std::string local_uri, size_t numa_node = 0, size_t num_bg_threads = 0);
    ```
    
    Yet, we can see in the tests:
    
    ```cpp
    TEST(OneLargeRpc, Background) {
      config_num_iters = 1;
      config_num_rpcs = 1;
      config_num_bg_threads = 1;
      launch_helper();
    }
    ```
    
    Thus, initialized at start without incrementation.
    
- bg_req_queue
    
    ```cpp
    MtQueue<BgWorkItem> bg_req_queue_[kMaxBgThreads];  ///< Background req queues
    ```
    
    ```cpp
    void Nexus::register_hook(Hook *hook) {
      uint8_t rpc_id = hook->rpc_id_;
      assert(rpc_id <= kMaxRpcId);
      assert(reg_hooks_arr_[rpc_id] == nullptr);
    
      reg_hooks_lock_.lock();
    
      req_func_registration_allowed_ = false;  // Disable future Ops registration
      reg_hooks_arr_[rpc_id] = hook;           // Save the hook
    
      **// Install background request submission lists
      for (size_t i = 0; i < num_bg_threads_; i++) {
        hook->bg_req_queue_arr_[i] = &bg_req_queue_[i];
      }**
    
      reg_hooks_lock_.unlock();
    }
    ```
    
- bg_thread_arr_
    
    ```cpp
    std::thread bg_thread_arr_[kMaxBgThreads];  ///< Background thread context
    ```
    
- BgWorkItem
    
    ```cpp
    class BgWorkItem {
       public:
        BgWorkItem() {}
    
        static inline BgWorkItem make_req_item(void *context, SSlot *sslot) {
          BgWorkItem ret;
          ret.wi_type_ = BgWorkItemType::kReq;
          ret.context_ = context;
          ret.sslot_ = sslot;
          return ret;
        }
    
        static inline BgWorkItem make_resp_item(void *context,
                                                erpc_cont_func_t cont_func,
                                                void *tag) {
          BgWorkItem ret;
          ret.wi_type_ = BgWorkItemType::kResp;
          ret.context_ = context;
          ret.cont_func_ = cont_func;
          ret.tag_ = tag;
          return ret;
        }
    
        BgWorkItemType wi_type_;
        void *context_;  ///< The Rpc's context
    
        // Fields for request handlers. For request handlers, we still have
        // ownership of the request slot, so we can hold it until enqueue_response.
        SSlot *sslot_;
    
        // Fields for continuations. For continuations, we have lost ownership of
        // the request slot, so the work item contains all needed info by value.
        erpc_cont_func_t cont_func_;
        void *tag_;
    
        bool is_req() const { return wi_type_ == BgWorkItemType::kReq; }
      };
    ```
    
- sm_thread_
    
    ```cpp
    void Nexus::sm_thread_ = std::thread(sm_thread_func, sm_thread_ctx);(SmThreadCtx ctx) {
      UDPServer<SmPkt> udp_server(ctx.sm_udp_port_, kSmThreadRxBlockMs);
      UDPClient<SmPkt> udp_client;
    
      // This is not a busy loop because of recv_blocking()
      while (*ctx.kill_switch_ == false) {
        SmPkt sm_pkt;
        const size_t ret = udp_server.recv_blocking(sm_pkt);
        if (ret == 0) continue;
    
        rt_assert(static_cast<size_t>(ret) == sizeof(sm_pkt),
                  "eRPC SM thread: Invalid SM packet RX size.");
    
        if (sm_pkt.pkt_type_ == SmPktType::kUnblock) {
          ERPC_INFO("eRPC SM thread: Received unblock packet.\\n");
          continue;
        }
    
        ERPC_INFO("eRPC SM thread: Received SM packet %s\\n",
                  sm_pkt.to_string().c_str());
    
        const uint8_t target_rpc_id =
            sm_pkt.is_req() ? sm_pkt.server_.rpc_id_ : sm_pkt.client_.rpc_id_;
    
        // Lock the Nexus to prevent Rpc registration while we lookup the hook
        ctx.reg_hooks_lock_->lock();
        Hook *target_hook = const_cast<Hook *>(ctx.reg_hooks_arr_[target_rpc_id]);
    
        if (target_hook != nullptr) {
          target_hook->sm_rx_queue_.unlocked_push(
              SmWorkItem(target_rpc_id, sm_pkt));
        } else {
          // We don't have an Rpc object for the target Rpc. Send an error
          // response iff it's a request packet.
          if (sm_pkt.is_req()) {
            ERPC_INFO(
                "eRPC SM thread: Received session management request for invalid "
                "Rpc %u from %s. Sending response.\\n",
                target_rpc_id, sm_pkt.client_.name().c_str());
    
            const SmPkt resp_sm_pkt =
                sm_construct_resp(sm_pkt, SmErrType::kInvalidRemoteRpcId);
    
            udp_client.send(resp_sm_pkt.client_.hostname_,
                            resp_sm_pkt.client_.sm_udp_port_, resp_sm_pkt);
          } else {
            ERPC_INFO(
                "eRPC SM thread: Received session management response for invalid "
                "Rpc %u from %s. Dropping.\\n",
                target_rpc_id, sm_pkt.client_.name().c_str());
          }
        }
    
        ctx.reg_hooks_lock_->unlock();
      }
    
      ERPC_INFO("eRPC SM thread: Session management thread exiting.\\n");
      return;
    }
    ```
    

1. bg_thread_ctx
    
    ** **bg_req_queue_ ≠ bg_req_queue_**
    

```cpp
/// Background thread context
  class BgThreadCtx {
   public:
    volatile bool *kill_switch_;  ///< The Nexus's kill switch

    **/// The Nexus's request functions array. Unlike Rpc threads that create a
    /// copy of the Nexus's request functions, background threads have a
    /// pointer. This is because background threads are launched before request
    /// functions are registered.**
    std::array<ReqFunc, kReqTypeArraySize> *req_func_arr_;

    TlsRegistry *tls_registry_;          ///< The Nexus's thread-local registry
    size_t bg_thread_index_;             ///< Index of this background thread
    MtQueue<BgWorkItem> *bg_req_queue_;  ///< Background thread request queue
  };
```

1. bg_thread_func is created by the work item.

```cpp
void Nexus::bg_thread_func(BgThreadCtx ctx) {
  ctx.tls_registry_->init();  // Initialize thread-local variables

  // The BgWorkItem request list can be indexed using the background thread's
  // index in the Nexus, or its eRPC TID.
  assert(ctx.bg_thread_index_ == ctx.tls_registry_->get_etid());
  ERPC_INFO("eRPC Nexus: Background thread %zu running. Tiny TID = %zu.\\n",
            ctx.bg_thread_index_, ctx.tls_registry_->get_etid());

  while (*ctx.kill_switch_ == false) {
    if (ctx.bg_req_queue_->size_ == 0) {
      // TODO: Put bg thread to sleep if it's idle for a long time
      continue;
    }

    while (ctx.bg_req_queue_->size_ > 0) {
      BgWorkItem wi = ctx.bg_req_queue_->unlocked_pop();

      if (wi.is_req()) {
        SSlot *s = wi.sslot_;  // For requests, we have a valid sslot
        uint8_t req_type = s->server_info_.req_type_;
        const ReqFunc &req_func = ctx.req_func_arr_->at(req_type);
        req_func.req_func_(static_cast<ReqHandle *>(s), wi.context_);
      } else {
        // For responses, we don't have a valid sslot
        wi.cont_func_(wi.context_, wi.tag_);
      }
    }
  }

  ERPC_INFO("eRPC Nexus: Background thread %zu exiting.\\n",
            ctx.bg_thread_index_);
  return;
}
```

1. Thus, bg_req_queue will be filled, when background thread proccess is needed.
    
    → done from register_hook through nexus_hook_
    
    ```cpp
    void Nexus::register_hook(Hook *hook) {
      uint8_t rpc_id = hook->rpc_id_;
      assert(rpc_id <= kMaxRpcId);
      assert(reg_hooks_arr_[rpc_id] == nullptr);
    
      reg_hooks_lock_.lock();
    
      req_func_registration_allowed_ = false;  // Disable future Ops registration
      reg_hooks_arr_[rpc_id] = hook;           // Save the hook
    
      // Install background request submission lists
      for (size_t i = 0; i < num_bg_threads_; i++) {
        hook->bg_req_queue_arr_[i] = &bg_req_queue_[i];
      }
    
      reg_hooks_lock_.unlock();
    }
    ```
    
    ```
    // Register the hook with the Nexus. This installs SM and bg command queues.
      nexus_hook_.rpc_id_ = rpc_id;
      nexus->register_hook(&nexus_hook_);
    ```
    
    →done in submit_bg_req_st / submit_bg_req_st
    
    ```
    template <class TTr>
    void Rpc<TTr>::submit_bg_req_st(SSlot *sslot) {
      assert(in_dispatch());
      assert(nexus_->num_bg_threads_ > 0);
    
      const size_t bg_etid = fast_rand_.next_u32() % nexus_->num_bg_threads_;
      auto *req_queue = nexus_hook_.bg_req_queue_arr_[bg_etid];
    
      req_queue->unlocked_push(Nexus::BgWorkItem::make_req_item(context_, sslot));
    }
    ```
    
    ```cpp
    template <class TTr>
    void Rpc<TTr>::submit_bg_resp_st(erpc_cont_func_t cont_func, void *tag,
                                     size_t bg_etid) {
      assert(in_dispatch());
      assert(nexus_->num_bg_threads_ > 0);
      assert(bg_etid < nexus_->num_bg_threads_);
    
      auto *req_queue = nexus_hook_.bg_req_queue_arr_[bg_etid];
      req_queue->unlocked_push(
          Nexus::BgWorkItem::make_resp_item(context_, cont_func, tag));
    }
    ```
    
    Thus see where those 2 functions are used:
    
    submit_bg_resp_st in process_resp_one_st (Process a single-packet response)
    
    ```cpp
    if (likely(cont_etid == kInvalidBgETid)) {
        cont_func(context_, tag);
      } else {
        submit_bg_resp_st(cont_func, tag, cont_etid);
      }
    ```
    
    - kInvalidBgETid
        
        ```cpp
        static constexpr size_t kInvalidBgETid = kMaxBgThreads;
        ```
        
    - cont_func ???
        
        ```cpp
        void cont_func(void *, void *) { printf("%s\\n", resp.buf_); }
        ```
        
    
    submit_bg_req_st in process_small_req_st / process_large_req_one_st /
    
    ```cpp
    // Remember request metadata for enqueue_response(). req_type was invalidated
      // on previous enqueue_response(). Setting it implies that an enqueue_resp()
      // is now pending; this invariant is used to safely reset sessions.
      assert(sslot->server_info_.req_type_ == kInvalidReqType);
      sslot->server_info_.req_type_ = pkthdr->req_type_;
      sslot->server_info_.req_func_type_ = req_func.req_func_type_;
    
    if (likely(!req_func.is_background())) {
        if (kZeroCopyRX) {
          // For foreground request handlers, a "fake" static request msgbuf
          // suffices. This improves performance, but it restricts ownership of the
          // request msgbuf to the duration of req_func.
          req_msgbuf = MsgBuffer(pkthdr, pkthdr->msg_size_);
        } else {
          req_msgbuf = alloc_msg_buffer(pkthdr->msg_size_);
          memcpy(req_msgbuf.buf_, pkthdr + 1, pkthdr->msg_size_);  // Omit header
        }
        req_func.req_func_(static_cast<ReqHandle *>(sslot), context_);
        return;
      } else {
        // Background request handlers need an RX ring--independent request copy
        req_msgbuf = alloc_msg_buffer(pkthdr->msg_size_);
        memcpy(req_msgbuf.buf_, pkthdr + 1, pkthdr->msg_size_);  // Omit header
        submit_bg_req_st(sslot);
        return;
      }
    ```
    
    ```cpp
    // req_msgbuf here is independent of the RX ring, so don't make another copy
      if (likely(!req_func.is_background())) {
        req_func.req_func_(static_cast<ReqHandle *>(sslot), context_);
      } else {
        submit_bg_req_st(sslot);
      }
    ```
    

So, when is req_func.is_background()?

```cpp
inline bool is_background() const {
    return req_func_type_ == ReqFuncType::kBackground;
  }
```

When used?

```cpp
auto range_handler_type = FLAGS_num_server_bg_threads > 0
                                  ? erpc::ReqFuncType::kBackground
                                  : erpc::ReqFuncType::kForeground;
    nexus.register_req_func(kAppRangeReqType, range_req_handler,
                            range_handler_type);
```

- register_req_func
    
    ```cpp
    int register_req_func(uint8_t req_type, erpc_req_func_t req_func,
                            ReqFuncType req_func_type = ReqFuncType::kForeground);
    ```
    

Should be explicitly stated when requesting.

```cpp
/// 1 primary, 1 backup, both in background
TEST(Base, BothInBackground) {
  primary_bg = true;
  backup_bg = true;

  auto reg_info_vec = {
      ReqFuncRegInfo(kTestReqTypeCP, req_handler_cp, ReqFuncType::kBackground),
      ReqFuncRegInfo(kTestReqTypePB, req_handler_pb, ReqFuncType::kBackground)};

  // 2 client sessions (=> 2 server threads), 3 background threads
  launch_server_client_threads(2, 3, client_thread, reg_info_vec,
                               ConnectServers::kTrue, 0.0);
}
```

Or, is there something I don’t get?

++additional

```cpp
nexus.register_req_func(kAppReqTypeIncast, req_handler_incast);
nexus.register_req_func(kAppReqTypeRegular, req_handler_regular);
```

```cpp
void TEST(Base, SimpleDisconnect) {
  launch_server_client_threads(1, 0, simple_disconnect, reg_info_vec,
                               ConnectServers::kFalse, 0.0);
}(
    size_t num_sessions, size_t num_bg_threads,
    void (*client_thread_func)(Nexus *, size_t),
    std::vector<ReqFuncRegInfo> req_func_reg_info_vec,
    ConnectServers connect_servers, double srv_pkt_drop_prob) {
  Nexus nexus("127.0.0.1:31850", kTestNumaNode, num_bg_threads);

  // Register the request handler functions
  for (ReqFuncRegInfo &info : req_func_reg_info_vec) {
    nexus.register_req_func(info.req_type_, info.req_func_,
                            info.req_func_type_);
  }

  num_servers_up = 0;
  all_servers_ready = false;
  client_done = false;

  test_printf("test: Using %zu sessions\\n", num_sessions);

  std::vector<std::thread> server_threads(num_sessions);

  // Launch one server Rpc thread for each client session
  for (size_t i = 0; i < num_sessions; i++) {
    // Server threads need an SM handler iff we're connecting servers together
    sm_handler_t sm_handler = connect_servers == ConnectServers::kFalse
                                  ? basic_empty_sm_handler
                                  : basic_sm_handler;

    server_threads[i] = std::thread(
        basic_server_thread_func, &nexus, kTestServerRpcId + i, sm_handler,
        num_sessions, connect_servers, srv_pkt_drop_prob);
  }

  // Wait for all servers to be ready before launching client thread
  while (!all_servers_ready) {
    std::this_thread::sleep_for(std::chrono::microseconds(1));
  }

  std::thread client_thread(client_thread_func, &nexus, num_sessions);

  for (auto &thread : server_threads) thread.join();
  client_thread.join();
}
```

```cpp
TEST(Base, SimpleDisconnect) {
  launch_server_client_threads(1, 0, simple_disconnect, reg_info_vec,
                               ConnectServers::kFalse, 0.0);
}
```