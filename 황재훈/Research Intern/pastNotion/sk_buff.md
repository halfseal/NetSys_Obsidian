1. The kernel maintains all sk_buff structures in a doubly linked list.
2. each sk_buff structure must be able to  
    find the head of the whole list quickly. To implement this requirement, an extra  
    structure of type sk_buff_head is inserted at the beginning of the list, as a kind of  
    dummy element  
    
3. unsigned int len  
    This is the size of the block of data in the buffer. This length includes both the  
    data in the main buffer (i.e., the one pointed to by head) and the data in the fragments.  
    
4. struct timeval stamp  
    This is usually meaningful only for a received packet. It is a timestamp that represents  
    when a packet was received or (occasionally) when one is scheduled for  
    transmission.  
    

char cb[40]  
This is a “control buffer,” or storage for private information, maintained by each  
layer for internal use. It is statically allocated within the sk_buff structure (currently  
with a size of 40 bytes) and is large enough to hold whatever private data  
is needed by each layer. In the code for each layer, access is done through macros  
to make the code more readable. TCP, for example, uses that  

  

struct sock * sk

This is a pointer to a sock data structure of the socket that owns this buffer. This  
pointer is needed when data is either locally generated or being received by a  
local process, because the data and socket-related information is used by L4  
(TCPor UDP) and by the user application. When a buffer is merely being forwarded  
(that is, neither the source nor the destination is on the local machine),  
this pointer is NULL.