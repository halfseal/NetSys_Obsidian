[[iperf_test 구조체]]

프로그램 하나가 옵션으로 서버와 클라이언트로 나누어짐. 서버는 단순하게 켜기만 하면 되고, 클라이언트는 서버의 아이피 주소를 입력함으로써 connection이 생성된다.

아무 옵션을 넣지 않았을 때, utm 내부에서는 6.3Gbps의 대역폭이 측정 됨. 10초간 측정하게 됨.

소스코드는 깃허브에서 확인하여 우분투에 다운로드 받음.

/src 폴더

main.c

→iperf_new_test() 에서 iperf_test라는 구조체 선언, 프로그램 실행 시 따라오는 인수들을 바탕으로 테스트의 환경 조건들을 구성함.

→iperf_defaults(test)에서 기본 값으로 설정하게 됨.

→run(struct iperf_test test) 함수를 실행하여 실제 테스트를 실행하게 됨.

run 에서는 역할이 서버인지 클라이언트인지 case문을 통해 결정되어 각각 해당하는 코드들을 실행하게 됨.

→ iperf_run_client() ==(src/iperf_client_api.c)==

1. 사전에 log file / json_output 과 같은 작업을 미리 함.
2. [[iperf_connet()를 통해 서버-클라이언트 간 연결]]
    
3. 해당하는 iperf_test 구조체의 state에서 현재 state를 저장하는데, IPERF_DONE state가 될 때까지 while문으로 아래 내용들을 반복하게 됨.
4. 타임아웃을 체크하고 event-driven 방식의 concurrent socket 방식을 사용함. → select함수를 이용하여 fd_set을 확인하는 과정을 거치게 됨.

만약 수신한게 있다면 → iperf_handle_message_client()

1. 패킷 수신이 정상적으로 이루어졌다면 넘어가게 됨.
    1. read() 함수를 통해 컨트롤 채널에서 컨트롤메시지가 잘 넘어왔는지 확인함.
    2. test→state에 따라 switch 문으로 분기하게 됨.
        1. PARAM_EXCHANGE의 경우 - iperf_exchange_parameters() 함수 호출, 서로 JSON을 통해 parameter를 주고 받고, CREATE_STREAM 상태로 변경. ==(src/iperf_api.c)==
        2. CREATE_STREAM의 경우 - iperf_create_streams() 함수 호출, → iperf_new_stream()호출로 새로운 스트림 생성. test→on_new_stream 함수를 콜백함. → connect_msg() → iperf_stream의 소켓 필드를 소켓프로그래밍 수준에서 셋팅하고 이를 알려줌.
            1. iperf_new_stream() → temp파일을 생성하고 buffer_fd로 접근하게 해줌 또한 unlink()로 파일 링크를 삭제하여 buffer_fd가 사라지면 파일이 삭제되도록 하고, ftruncate()으로 설정한 block size 만큼의 크기가 되도록 조절함. 마지막으로 buffer 포인터에 mmap() 함수를 통해 메모리 주소로 접근 가능하도록 하였음. 거기다가 fill_with_repeating_pattern()을 통해 128KB 짜리 임시 파일을 가득 채우게 됨.
                - 코드
                    
                    ```C
                    struct iperf_stream *
                    iperf_new_stream(struct iperf_test *test, int s, int sender)
                    {
                        struct iperf_stream *sp;
                        int ret = 0;
                    
                        char template[1024];
                        if (test->tmp_template) {
                            snprintf(template, sizeof(template) / sizeof(char), "%s", test->tmp_template);
                        } else {
                            //find the system temporary dir *unix, windows, cygwin support
                            char* tempdir = getenv("TMPDIR");
                            if (tempdir == 0){
                                tempdir = getenv("TEMP");
                            }
                            if (tempdir == 0){
                                tempdir = getenv("TMP");
                            }
                            if (tempdir == 0){
                    \#if defined(__ANDROID__)
                                tempdir = "/data/local/tmp";
                    \#else
                                tempdir = "/tmp";
                    \#endif
                            }
                            snprintf(template, sizeof(template) / sizeof(char), "%s/iperf3.XXXXXX", tempdir);
                        }
                    
                        sp = (struct iperf_stream *) malloc(sizeof(struct iperf_stream));
                        if (!sp) {
                            i_errno = IECREATESTREAM;
                            return NULL;
                        }
                    
                        memset(sp, 0, sizeof(struct iperf_stream));
                    
                        sp->sender = sender;
                        sp->test = test;
                        sp->settings = test->settings;
                        sp->result = (struct iperf_stream_result *) malloc(sizeof(struct iperf_stream_result));
                        if (!sp->result) {
                            free(sp);
                            i_errno = IECREATESTREAM;
                            return NULL;
                        }
                    
                        memset(sp->result, 0, sizeof(struct iperf_stream_result));
                        TAILQ_INIT(&sp->result->interval_results);
                    
                        /* Create and randomize the buffer */
                        sp->buffer_fd = mkstemp(template);
                        if (sp->buffer_fd == -1) {
                            i_errno = IECREATESTREAM;
                            free(sp->result);
                            free(sp);
                            return NULL;
                        }
                        if (unlink(template) < 0) {
                            i_errno = IECREATESTREAM;
                            free(sp->result);
                            free(sp);
                            return NULL;
                        }
                        if (ftruncate(sp->buffer_fd, test->settings->blksize) < 0) {
                            i_errno = IECREATESTREAM;
                            free(sp->result);
                            free(sp);
                            return NULL;
                        }
                        sp->buffer = (char *) mmap(NULL, test->settings->blksize, PROT_READ|PROT_WRITE, MAP_PRIVATE, sp->buffer_fd, 0);
                        if (sp->buffer == MAP_FAILED) {
                            i_errno = IECREATESTREAM;
                            free(sp->result);
                            free(sp);
                            return NULL;
                        }
                        sp->pending_size = 0;
                    
                        /* Set socket */
                        sp->socket = s;
                    
                        sp->snd = test->protocol->send;
                        sp->rcv = test->protocol->recv;
                    
                        if (test->diskfile_name != (char*) 0) {
                    	sp->diskfile_fd = open(test->diskfile_name, sender ? O_RDONLY : (O_WRONLY|O_CREAT|O_TRUNC), S_IRUSR|S_IWUSR);
                    	if (sp->diskfile_fd == -1) {
                    	    i_errno = IEFILE;
                                munmap(sp->buffer, sp->test->settings->blksize);
                                free(sp->result);
                                free(sp);
                    	    return NULL;
                    	}
                            sp->snd2 = sp->snd;
                    	sp->snd = diskfile_send;
                    	sp->rcv2 = sp->rcv;
                    	sp->rcv = diskfile_recv;
                        } else
                            sp->diskfile_fd = -1;
                    
                        /* Initialize stream */
                        if (test->repeating_payload)
                            fill_with_repeating_pattern(sp->buffer, test->settings->blksize);
                        else
                            ret = readentropy(sp->buffer, test->settings->blksize);
                    
                        if ((ret < 0) || (iperf_init_stream(sp, test) < 0)) {
                            close(sp->buffer_fd);
                            munmap(sp->buffer, sp->test->settings->blksize);
                            free(sp->result);
                            free(sp);
                            return NULL;
                        }
                        iperf_add_stream(test, sp);
                    
                        return sp;
                    }
                    ```
                    
            2. 여기서 iperf_stream에 대한 malloc이 이루어짐.
            3. diskfile_fd는 직접 해당 파일을 열어서 fd로 가지고 있음.
            4. buffer를 통해 해당 임시파일에 블럭 사이즈 만큼 0~9까지 전송할 바이트를 체우게 됨.
            5. iperf_init_stream() 함수 호출
            6. iperf_add_stream() 함수 호출 → multi stream 일 때 사용
        3. TEST_START인 경우 -

위의 과정이 다 끝났다면 TEST_RUNNING인지 확인하고, 매 스트림마다 pthread_create을 통해 실제 전송을 할 iperf_client_worker_run 함수를 호출하게 됨.

→iperf_client_worker_run() - pthread 관련 설정을 해주고 sender면 iperf_send_mt(), 아니면 ,iperf_recv_mt() 함수 호출

while(! test→done() && ! sp→ done()) 이므로, 여기서 iperf_send_mt가 반복된다고 보면 된다.

→iperf_send_mt() → sp→snd를 호출함.

snd는 iperf_tcp_send()함수임. 맨 위에서 test default setting 에서 그렇게 설정함.

[[iperf_tcp_send()]]

  

→ iperf_run_server() ==(src/iperf_server_api.c)==

1. 사전에 log file / json_output 과 같은 작업을 미리 함.
2. iperf_server_listen()함수 실행
    1. netannounce() 실행 → 내부적으로 socket()으로 소켓을 만들고 listen() 함수를 호출함.
3. 타임아웃을 확인함.
4. select로 새로운 connection을 확인하고, 타임아웃이 되었는지 한번 더 확인
5. connection이 확인 된 경우
    1. iperf_accept() → socket 함수들 중 accept() 함수를 wrapping 하는 함수임. 해당 connection에 해당하는 file descriptor는 iperf_test 구조체의 ctrl_sck에 저장하여 꺼내다 쓸 수 있게 하였음.
    2. 보낼 내용들을 설정하여 iperf_handle_message_server() 함수를 호출함.
        1. Nread, Nwrite 함수 ⇒ 해당 하는 fd에 읽고 쓰는 wrapping function
        2. iperf_test→state를 보고 현재 어떤 상태인지 파악, 다음 수행 결정.
        3. DISPLAY_RESULTS로 상태를 설정하고, reporter_callback(iperf_test를 가리킴)을 통해 결과 표시.
        4. pthread_create을 통해 실제 전송을 수행하는 iperf_server_worker_run()함수를 실행하게 됨.
    3. iperf_server_worker_run()
        1. iperf_send_mt에서 실제로

  

  

  

---

Nwrite(int fd, const char *buf, size_t count, int prot) →해당 file descriptor에 write 함수를 사용. prot는 사용하지 않는 인수임. 한번에 count만큼 write 하도록 하고, 실제 전송된 byte 수를 r로 받아서 count에서 r을 빼서 전송해야 할 남은 바이트 수를 계속 업데이트 함. 최종적으로 count를 return 하여 전송한 byte 수를 알림. ==(src/net.c)==

  

  

계획 - wireshark로 어떤 커넥션들이 얼마나 생기는지 확인해 보고자 함.

추가적으로 연결을 관리하기 위해 또다른 연결을 사용 중인 것으로 보임.

---

문제 인식 - 커널 단 앞까지 100Gbps를 지원함에도 불구하고 본 Iperf application은 최고 속도가 나오지 않음. 이를 해결하기 위한 내부 소스코드 중에서 병목이 생기는 부분을 파악하고 이를 해결하고자 함.

![[Untitled 4.png|Untitled 4.png]]

TCP window size -n 14 로 했다가 컴퓨터 터질 뻔함.

![[Untitled 1 3.png|Untitled 1 3.png]]

양쪽 전부 100Gbps로 설정하고 실행하였으나 달라진 건 없었음

1. 100Gbps가 제대로 설정이 안됨
2. 노트북 성능이 안 돼서 되지 않음
3. iperf 자체 오버헤드임.
4. client → server로 데이터 전송이 이루어짐

![[Untitled 2 3.png|Untitled 2 3.png]]

---
