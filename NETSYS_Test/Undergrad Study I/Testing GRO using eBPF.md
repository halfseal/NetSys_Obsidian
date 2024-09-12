---
creator: Maike
type: Hands-On
created: 2022-07-19
---
# Work in Progress! Might change down the line!

The idea:

attaching eBPF to functions before and after GRO is performed, counting packets and comparing packet sizes.

Reason?: Getting acquainted with eBPF while also observing a mechanism we discussed during the study in action.

<aside> ❗ For now, I am just jotting down my progress and hopefully, organize it later on_

</aside>

Challenge 1

How do I trace a kernel function that is

a) not a system call?

b) not in a binary known to me?

The easy answer to this: With its name. To start off, I tried to trace the __napi_schedule() function, which worked perfectly with just

```python
b.attach_kprobe(event="__napi_schedule", fn_name="gro_trace")
```

Challenge 2

Which function should I trace to follow

---

_Presentation Material from here_

Program: [https://github.com/hema2601/eBPF-AssortedProjects/blob/master/gro_test.py](https://github.com/hema2601/eBPF-AssortedProjects/blob/master/gro_test.py)

Output:
![[Untitled(75).png]]


How did I find the functions?

**ftrace**

[https://jvns.ca/blog/2017/03/19/getting-started-with-ftrace/](https://jvns.ca/blog/2017/03/19/getting-started-with-ftrace/)

[https://opensource.com/article/21/7/linux-kernel-trace-cmd](https://opensource.com/article/21/7/linux-kernel-trace-cmd)

Command

```python
sudo trace-cmd record -p function_graph -P 0
```

<aside> ‼️ After running ftrace, eBPF might stop working until you reboot

</aside>
![[Untitled(76).png]]


```c
//taken from /net/core/dev.c in linux kernel version 4.18.15
[...]
gro_result_t napi_gro_receive(struct napi_struct *napi,  struct sk_buff *skb)
{
	skb_mark_napi_id(skb, napi);
	trace_napi_gro_receive_entry(skb);

	skb_gro_reset_offset(skb);

	return napi_skb_finish(dev_gro_receive(napi, skb), skb);
}
[...]

gro_result_t napi_skb_finish(gro_result_t ret, struct sk_buff *skb)
{
	switch(ret){
	case GRO_NORMAL:
		if (netif_receive_skb_internal(skb))
			ret = GRO_DROP;
		break;

	case GRO_DROP:
		kfree_skb(skb);
		break;

	case GRO_MERGED_FREE:
		if(NAPI_GRO_CB(skb)->free == NAPI_GRO_FREE_STOLEN_HEAD)
			napi_skb_free_stolen_head(skb);
		else
			__kfree_skb(skb);
		break;

	case GRO_HELD:
	case GRO_MERGED:
	case GRO_CONSUMED:
		break;			
	}

	return ret;
}

```

Why not trace napi_skb_finish()?
![[Untitled(77).png]]


How can I determine whether a function is traceable?

```c
vim /proc/kallsyms
```

A collection of all kernel symbols that can be connected to a debugger.
![[Untitled(78).png]]

![[Untitled(79).png]]

![[Untitled(80).png]]


Things to elaborate on:

1. Are the eBPF functions called before or after the actual function call?
2. is the napi_gro_receive() function actually the function I need?
3. Will I see results once I generate desirable traffic with actual data?