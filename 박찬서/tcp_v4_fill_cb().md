---
Parameter:
  - iphdr
  - tcphdr
  - sk_buff
Return: void
Location: /net/ipv4/tcp_ipv4.c
---

```c title=tcp_v4_fill_cb()
static void tcp_v4_fill_cb(struct sk_buff *skb, const struct iphdr *iph,

const struct tcphdr *th)

{

/* This is tricky : We move IPCB at its correct location into TCP_SKB_CB()

* barrier() makes sure compiler wont play fool^Waliasing games.

*/

memmove(&TCP_SKB_CB(skb)->header.h4, IPCB(skb),

sizeof(struct inet_skb_parm));

barrier();

  

TCP_SKB_CB(skb)->seq = ntohl(th->seq);

TCP_SKB_CB(skb)->end_seq = (TCP_SKB_CB(skb)->seq + th->syn + th->fin +

skb->len - th->doff * 4);

TCP_SKB_CB(skb)->ack_seq = ntohl(th->ack_seq);

TCP_SKB_CB(skb)->tcp_flags = tcp_flag_byte(th);

TCP_SKB_CB(skb)->tcp_tw_isn = 0;

TCP_SKB_CB(skb)->ip_dsfield = ipv4_get_dsfield(iph);

TCP_SKB_CB(skb)->sacked = 0;

TCP_SKB_CB(skb)->has_rxtstamp =

skb->tstamp || skb_hwtstamps(skb)->hwtstamp;

}
```

`s