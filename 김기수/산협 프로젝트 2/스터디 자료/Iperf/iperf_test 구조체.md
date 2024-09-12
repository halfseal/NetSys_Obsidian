```C
struct iperf_test
{
    pthread_mutex_t print_mutex;

    char      role;                             /* 'c' lient or 's' erver */
    enum iperf_mode mode;
    int       sender_has_retransmits;
    int       other_side_has_retransmits;       /* used if mode == BIDIRECTIONAL */
    struct protocol *protocol;
    signed char state;
    char     *server_hostname;                  /* -c option */
    char     *tmp_template;
    char     *bind_address;                     /* first -B option */
    char     *bind_dev;                         /* bind to network device */
    TAILQ_HEAD(xbind_addrhead, xbind_entry) xbind_addrs; /* all -X opts */
    int       bind_port;                        /* --cport option */
    int       server_port;
    int       omit;                             /* duration of omit period (-O flag) */
    int       duration;                         /* total duration of test (-t flag) */
    char     *diskfile_name;			/* -F option */
    int       affinity, server_affinity;	/* -A option */
\#if defined(HAVE_CPUSET_SETAFFINITY)
    cpuset_t cpumask;
\#endif /* HAVE_CPUSET_SETAFFINITY */
    char     *title;				/* -T option */
    char     *extra_data;			/* --extra-data */
    char     *congestion;			/* -C option */
    char     *congestion_used;			/* what was actually used */
    char     *remote_congestion_used;		/* what the other side used */
    char     *pidfile;				/* -P option */

    char     *logfile;				/* --logfile option */
    FILE     *outfile;

    int       ctrl_sck;
    int       mapped_v4;
    int       listener;
    int       prot_listener;

    int	      ctrl_sck_mss;			/* MSS for the control channel */

\#if defined(HAVE_SSL)
    char      *server_authorized_users;
    EVP_PKEY  *server_rsa_private_key;
    int       server_skew_threshold;
    int       use_pkcs1_padding;
\#endif // HAVE_SSL

    /* boolean variables for Options */
    int       daemon;                           /* -D option */
    int       one_off;                          /* -1 option */
    int       no_delay;                         /* -N option */
    int       reverse;                          /* -R option */
    int       bidirectional;                    /* --bidirectional */
    int	      verbose;                          /* -V option - verbose mode */
    int	      json_output;                      /* -J option - JSON output */
    int	      json_stream;                      /* --json-stream */
    int	      zerocopy;                         /* -Z option - use sendfile */
    int       debug;				/* -d option - enable debug */
    enum      debug_level debug_level;          /* -d option option - level of debug messages to show */
    int	      get_server_output;		/* --get-server-output */
    int	      udp_counters_64bit;		/* --use-64-bit-udp-counters */
    int       forceflush; /* --forceflush - flushing output at every interval */
    int	      multisend;
    int	      repeating_payload;                /* --repeating-payload */
    int       timestamps;			/* --timestamps */
    char     *timestamp_format;

    char     *json_output_string; /* rendered JSON output if json_output is set */
    /* Select related parameters */
    int       max_fd;
    fd_set    read_set;                         /* set of read sockets */
    fd_set    write_set;                        /* set of write sockets */

    /* Interval related members */
    int       omitting;
    double    stats_interval;
    double    reporter_interval;
    void      (*stats_callback) (struct iperf_test *);
    void      (*reporter_callback) (struct iperf_test *);
    Timer     *omit_timer;
    Timer     *timer;
    int        done;
    Timer     *stats_timer;
    Timer     *reporter_timer;

    double cpu_util[3];                            /* cpu utilization of the test - total, user, system */
    double remote_cpu_util[3];                     /* cpu utilization for the remote host/client - total, user, system */

    int       num_streams;                      /* total streams in the test (-P) */

    atomic_iperf_size_t bytes_sent;
    atomic_iperf_size_t blocks_sent;

    atomic_iperf_size_t bytes_received;
    atomic_iperf_size_t blocks_received;

    iperf_size_t bitrate_limit_stats_count;               /* Number of stats periods accumulated for server's total bitrate average */
    iperf_size_t *bitrate_limit_intervals_traffic_bytes;  /* Pointer to a cyclic array that includes the last interval's bytes transferred */
    iperf_size_t bitrate_limit_last_interval_index;       /* Index of the last interval traffic inserted into the cyclic array */
    int          bitrate_limit_exceeded;                  /* Set by callback routine when average data rate exceeded the server's bitrate limit */

    int server_last_run_rc;                      /* Save last server run rc for next test */
    uint server_forced_idle_restarts_count;      /* count number of forced server restarts to make sure it is not stack */
    uint server_forced_no_msg_restarts_count;    /* count number of forced server restarts to make sure it is not stack */
    uint server_test_number;                     /* count number of tests performed by a server */

    char      cookie[COOKIE_SIZE];
//    struct iperf_stream *streams;               /* pointer to list of struct stream */
    SLIST_HEAD(slisthead, iperf_stream) streams;
    struct iperf_settings *settings;

    SLIST_HEAD(plisthead, protocol) protocols;

    /* callback functions */
    void      (*on_new_stream)(struct iperf_stream *);
    void      (*on_test_start)(struct iperf_test *);
    void      (*on_connect)(struct iperf_test *);
    void      (*on_test_finish)(struct iperf_test *);

    /* cJSON handles for use when in -J mode */\
    cJSON *json_top;
    cJSON *json_start;
    cJSON *json_connected;
    cJSON *json_intervals;
    cJSON *json_end;

    /* Server output (use on client side only) */
    char *server_output_text;
    cJSON *json_server_output;

    /* Server output (use on server side only) */
    TAILQ_HEAD(iperf_textlisthead, iperf_textline) server_output_list;

};
```

  

protocol 구조체 아래에 send, recv를 가리키는 함수 포인터가 있음. 각각 iperf_stream 구조체 포인터를 인수로 가짐.