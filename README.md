### Bài triển khai thuật toán distance-vector

## Mục tiêu của thuật toán DV:
Mỗi router sẽ biết chi phí đến hàng xóm trực tiếp. 
Nhận bảng định tuyến (distance vector) từ các hàng xóm.
-> Dùng thông tin này để tính chi phí ngắn nhất đến tất cả các đích.

## Ý tưởng chính triển khai:
Chúng ta cần đảm bảo router:
- Biết được hàng xóm nào kết nối với mình.
- Ghi nhớ bảng định tuyến của từng hàng xóm.
- Dùng thông tin đó để tính bảng định tuyến (forwarding table) của mình.
- Gửi thông tin này cho các hàng xóm khác (broadcast distance vector).

## Các thuộc tính:
- distance_vector: Bảng định tuyến hiện tại của router (destination → cost).
- forwarding_table: Lưu "next hop" (router kế tiếp) cho từng đích.
- neighbor_vectors: Ghi lại bảng định tuyến (DV) của từng hàng xóm.
- neighbor_costs: Ghi lại chi phí kết nối trực tiếp đến từng hàng xóm.
- port_to_neighbor: Ánh xạ từ cổng port đến địa chỉ hàng xóm.
- heartbeat_time: Thời gian giữa các lần gửi DV định kỳ.
- last_time: Thời điểm gần nhất gửi DV.

## Các method:
# __init__(self, addr, heartbeat_time)
- Mục đích: 
Khởi tạo router với địa chỉ và thời gian gửi DV định kỳ.
- Công việc chính:
+ Thiết lập bảng định tuyến ban đầu (chỉ biết về chính mình với cost = 0).

Router.__init__(self, addr)  
        self.heartbeat_time = heartbeat_time
        self.last_time = 0
        
+ Khởi tạo các bảng hỗ trợ như: forwarding table, bảng hàng xóm, ...

self.distance_vector = defaultdict(lambda: float('inf'))  # Distance to each router
        self.distance_vector[addr] = 0  # Distance to self is 0
        self.forwarding_table = {}
        self.neighbor_vectors = {}  # addr -> vector
        self.neighbor_costs = {}    # port -> (neighbor, cost)
        self.port_to_neighbor = {}  # port -> neighbor

# handle_new_link(self, port, endpoint, cost)
- Mục đích: 
Xử lý khi phát hiện kết nối mới (link mới đến router hàng xóm).
- Công việc chính:
Cập nhật thông tin hàng xóm mới.
Ghi nhận chi phí kết nối trực tiếp đến hàng xóm đó.
Cập nhật bảng định tuyến hiện tại (DV).
Gửi (broadcast) lại DV của mình cho hàng xóm.

# handle_remove_link(self, port)
- Mục đích:
Xử lý khi một link bị ngắt.
- Công việc chính:
Xóa thông tin liên quan đến hàng xóm bị mất khỏi tất cả bảng liên quan.
Tính lại bảng định tuyến của mình.
Gửi lại DV nếu có thay đổi.

# handle_packet(self, port, packet)
- Mục đích: 
Xử lý khi nhận một gói tin trên một cổng.
- Gồm 2 trường hợp:
+ Gói Traceroute (data packet):
Dùng forwarding_table để gửi gói đi qua "next hop".
+ Gói Routing (gói mang distance vector):
Giải mã packet.content để lấy DV của hàng xóm gửi đến.
Lưu lại DV hàng xóm.
Gọi recompute_routing_table() để tính lại DV của mình.
Nếu có thay đổi, gửi (broadcast) lại DV.

# handle_time(self, time_ms)
- Mục đích: 
Hàm xử lý định kỳ theo thời gian (tính bằng millisecond).
- Công việc chính:
Nếu đã đủ heartbeat_time kể từ lần cuối gửi:
Gửi lại bảng định tuyến hiện tại của mình đến các hàng xóm (broadcast DV).

# broadcast_distance_vector(self)
- Mục đích: 
Gửi bảng định tuyến (distance vector) của mình tới tất cả hàng xóm.

- Công việc chính:
Chuyển distance_vector sang dạng JSON.
Tạo Packet.ROUTING cho mỗi hàng xóm và gửi qua cổng tương ứng.

# recompute_routing_table(self)
- Mục đích:
Tính lại bảng định tuyến dựa trên DV của hàng xóm và link trực tiếp.

- Nguyên lý: Dựa trên thuật toán Bellman-Ford:
Với mỗi đích D, xét tất cả các hàng xóm N.
Tính tổng chi phí cost = cost_to_N + N[D].
Nếu có route tốt hơn, cập nhật DV và forwarding table.
Trả về True nếu có thay đổi, ngược lại False.

# __repr__(self)
- Mục đích: 
Hiển thị thông tin tóm tắt router khi debug (không ảnh hưởng grading).

## Kiểm chứng thành công:
Khi chạy test từ file JSON (định nghĩa correct_routes), hệ thống đã học được đầy đủ các tuyến đường từ mọi client (b, c, d) đến các đích còn lại.
Ví dụ:

b -> b: ['b', 'A', 'b']
b -> c: ['b', 'A', 'c']
b -> d: ['b', 'A', 'E', 'd']
c -> b: ['c', 'A', 'b']
c -> c: ['c', 'A', 'c']
c -> d: ['c', 'A', 'E', 'd']
d -> b: ['d', 'E', 'A', 'b']
d -> c: ['d', 'E', 'A', 'c']
d -> d: ['d', 'E', 'd']

SUCCESS: All Routes correct!
=> Tất cả đều đúng, phản ánh tính đúng đắn của triển khai.
