---
creator: Maike
type: Hands-On
created: 2022-08-09
---
Took the professor's advice and downloaded a file.

PDF file of size ~2.8MB

Results:
![[Untitled(109).png]]


### Let's look at the numbers

4 packets of size 1420B merge into one packet of size 5524B. 1420B x 4 = 5680B 5680B - 5524B = 156B 156B / 3 = 52B

→ Our headers are 52B each! Weird, because Ethernet header(14B), IP header (20B) and TCP header (20B) are 54B.

### What is the header?

The problem: I cannot print the packet!

Why?

- I cannot use a loop due to safety restrictions
- I cannot use some helper functions such as bpf_skb_load_bytes_relative() due to (probably) my old kernel version

So, based on header knowledge, print protocol values.

1st Step: Get a feeling for endianness. Print well-known values such as MAC address!
![[Untitled(110).png]]


The result:
![[Untitled(111).png]]


Trying from skb→head[6]~[11] same inconsistency

The issue?
![[Untitled(112).png]]


[Source](http://vger.kernel.org/~davem/skb_data.html)

→ skb->head is probably pointing to garbage

Retrying with skb→data[6]~[11]
![[Untitled(113).png]]


Retrying with skb→data[0]~[5]
![[Untitled(114).png]]


AHA! All packets start with 0x45 0x00, which is the beginning of an IP header!

0x1594, would be the length then. 0x1594 → 5524, which is the length!
![[Untitled(115).png]]


New Info:

- IP header is indeed 20B.
- Transport Protocol is TCP

Trying for skb→data[-14]~[-9]
![[Untitled(116).png]]

![[Untitled(117).png]]


Which is actually the right MAC address!!

New Info:

- The Ethernet header seems to be 14B

What does the end of the TCP header look like?
![[Untitled(118).png]]


Seems normal, consistent 0s for the urgent pointer, changing checksum and consistent window size.

So, if the IP header has length 20B (indicated by the length field), the correct MAC address can be found 14B from the beginning of the two bytes placed at skb→data+38 and skb→data+39 are reserved for the urgent pointer… **Why is the header size 52B?**

### Back to the Beginning. What do the Numbers mean?
![[Untitled(119).png]]


We have:

1. Protocol number
2. skb→len
3. skb→data_len

What is the difference between skb→ len and skb→data_len?

According to [this source](https://people.cs.clemson.edu/~westall/853/notes/skbuff.pdf), the data_len is usually 0 and describes the length of the data in the **fragment list** and unmapped page buffers. len includes the length of the data in the actual memory region for the skb plus the data_len part.

This highlights a few points:

1. len - data_len in last packet ALWAYS 1420. → Everything is attached to one complete packet.
2. data_len = (num_packets - 1) * 1420 - ((num_packets - 1) * 52) → somehow the header is still 52B
3. GRO seems to actually use the old IP fragmentation API

Final Idea: The len value is incremented and decremented as headers are attached/detached.

printing skb→data[32]~[37]
![[Untitled(120).png]]


TCP header length is 8!! So 8*4B = 32B.

Mystery Solved.