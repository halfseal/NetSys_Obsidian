```c
/* Find the correct entry in the "incomplete datagrams" queue for
 * this IP datagram, and create new one, if nothing is found.
 */
static struct ipq *ip_find(struct net *net, struct iphdr *iph,
			   u32 user, int vif)
{
	struct frag_v4_compare_key key = {
		.saddr = iph->saddr,
		.daddr = iph->daddr,
		.user = user,
		.vif = vif,
		.id = iph->id,
		.protocol = iph->protocol,
	};
	struct inet_frag_queue *q;

	q = inet_frag_find(net->ipv4.fqdir, &key);
	if (!q)
		return NULL;

	return container_of(q, struct ipq, q);
}
```