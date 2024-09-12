 * After we queued a packet into sd->input_pkt_queue,
 * we need to make sure this queue is serviced soon.

 * - If this is another cpu queue, link it to our rps_ipi_list,
 *   and make sure we will process rps_ipi_list from net_rx_action().
 * 
 * - If this is our own queue, NAPI schedule our backlog.
 *   Note that this also raises NET_RX_SOFTIRQ.
 */
static void napi_schedule_rps(struct softnet_data *sd)

load balancer, irq balancer, cache miss 탐지
cpu가 얼마나 복잡한지 판별하는 과정
rps 효율 증대시키기, 놀고 있는 증거 찾기
input packet queue, poll list 차이

	ipi list 보낼 때, 깨울 코어를 지정할 수있는가? 아니면 모든 코어에 softirq를 보낸가



