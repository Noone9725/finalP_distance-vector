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
+ Gọi hàm khởi tạo của lớp cha Router:
        Router.__init__(self, addr)
+ Lưu thời gian gửi định kỳ DV:
        self.heartbeat_time = heartbeat_time
+ Khởi tạo distance_vector với mặc định vô cực:
        self.distance_vector = defaultdict(lambda: float('inf'))
+ Đặt khoảng cách tới chính mình là 0:
        self.distance_vector[addr] = 0
+ Khởi tạo bảng định tuyến:
        self.forwarding_table = {}
+ Các bảng phụ trợ khác để lưu thông tin hàng xóm:
        self.neighbor_vectors = {}
        self.neighbor_costs = {}
        self.port_to_neighbor = {}

# handle_new_link(self, port, endpoint, cost)
- Mục đích: 
Xử lý khi phát hiện kết nối mới (link mới đến router hàng xóm).
- Công việc chính:
+ Ghi nhận thông tin hàng xóm mới:
        self.port_to_neighbor[port] = endpoint
+ Lưu chi phí đến hàng xóm:
        self.neighbor_costs[endpoint] = cost
+ Thêm hàng xóm vào DV (vì là link trực tiếp):
        self.distance_vector[endpoint] = cost
+ Thiết lập next hop là chính hàng xóm đó:
        self.forwarding_table[endpoint] = endpoint
+ Gửi DV mới của mình cho hàng xóm:
        self.broadcast_distance_vector()

# handle_remove_link(self, port)
- Mục đích:
Xử lý khi một link bị ngắt.
- Công việc chính:
+ Tìm địa chỉ hàng xóm tương ứng với port:
        neighbor = self.port_to_neighbor.get(port)
+ Xóa thông tin liên quan đến hàng xóm bị mất khỏi tất cả bảng liên quan:
        del self.port_to_neighbor[port]  
        del self.neighbor_costs[neighbor]  
        del self.neighbor_vectors[neighbor]
+ Tính lại bảng định tuyến (sau khi mất một hàng xóm):
        updated = self.recompute_routing_table()
+ Gửi DV nếu có thay đổi:
        if updated: self.broadcast_distance_vector()

# handle_packet(self, port, packet)
- Mục đích: 
Xử lý khi nhận một gói tin trên một cổng.
Gồm 2 trường hợp:
- Gói Traceroute (data packet):
Dùng forwarding_table để gửi gói đi qua "next hop":
        dst = packet.dst_addr
            if dst in self.forwarding_table:
                next_hop = self.forwarding_table[dst]
                for p, n in self.port_to_neighbor.items():
                    if n == next_hop:
                        self.send(p, packet)
                        break
- Gói Routing (gói mang distance vector):
+ Giải mã packet.content để lấy DV của hàng xóm gửi đến:
        vector = json.loads(packet.content)
+ Lưu DV của hàng xóm vào bảng phụ:
        self.neighbor_vectors[sender] = vector
+ Gọi recompute_routing_table() để tính lại DV của mình:
        updated = self.recompute_routing_table()
+ Nếu có thay đổi, gửi (broadcast) lại DV:
        if updated: self.broadcast_distance_vector()

# handle_time(self, time_ms)
- Mục đích: 
Hàm xử lý định kỳ theo thời gian (tính bằng millisecond).
- Công việc chính:
+ Kiểm tra đã đến thời điểm gửi DV chưa:
        if time_ms - self.last_time >= self.heartbeat_time:
+ Cập nhật lại thời gian:
        self.last_time = time_ms
+ Gửi DV hiện tại cho hàng xóm (Nếu đã đủ heartbeat_time kể từ lần cuối gửi):
        self.broadcast_distance_vector()

# broadcast_distance_vector(self)
- Mục đích: 
Gửi bảng định tuyến (distance vector) của mình tới tất cả hàng xóm.
- Công việc chính:
+ Chuyển DV sang JSON:
        content = json.dumps(dict(self.distance_vector))
+ Tạo Packet.ROUTING cho mỗi hàng xóm và gửi qua cổng tương ứng:
        for port in self.links:
            neighbor = self.links[port].e2
            packet = Packet(Packet.ROUTING, self.addr, neighbor, content)
            self.send(port, packet)

# recompute_routing_table(self)
- Mục đích:
Tính lại bảng định tuyến dựa trên DV của hàng xóm và link trực tiếp.
- Nguyên lý: Dựa trên thuật toán Bellman-Ford:
- Công việc chính:

Khởi tạo bảng tạm thời:
+ Khởi tạo biến updated để kiểm tra có thay đổi gì không:
        updated = False
+ Tạo bảng distance_vector mới với giá trị mặc định là vô cực:
        new_dv = defaultdict(lambda: float('inf'))
+ Đặt khoảng cách tới chính mình là 0:
        new_dv[self.addr] = 0
+ Khởi tạo bảng định tuyến mới (forwarding table):
        new_ft = {}
  
Thu thập tất cả các đích có thể:
+ Tập hợp tất cả địa chỉ đích từ DV hiện tại và DV của hàng xóm:
        all_dests = set([self.addr]) | set(self.distance_vector.keys())  
        for vec in self.neighbor_vectors.values():  
            all_dests.update(vec.keys())

Tính chi phí tối thiểu đến từng đích:
+ Bỏ qua chính router hiện tại (không cần tính đến chính mình):
        if dest == self.addr: continue
+ Khởi tạo giá trị chi phí tối thiểu ban đầu:
        min_cost = float('inf')
        best_next_hop = None
+ Duyệt qua từng hàng xóm để tính chi phí đến dest thông qua họ:
        for neighbor, cost_to_neighbor in self.neighbor_costs.items():
                neighbor_vector = self.neighbor_vectors.get(neighbor, {})
                cost_from_neighbor = neighbor_vector.get(dest, float('inf'))
                total_cost = cost_to_neighbor + cost_from_neighbor
                if total_cost < min_cost:
                    min_cost = total_cost
                    best_next_hop = neighbor

So sánh với link trực tiếp (nếu có):
        if dest in self.neighbor_costs and self.neighbor_costs[dest] < min_cost:
                min_cost = self.neighbor_costs[dest]
                best_next_hop = dest

Cập nhật bảng DV và bảng định tuyến tạm:
+ Nếu tìm được đường đi đến dest, lưu kết quả vào bảng mới:
        if min_cost < float('inf'):
                new_dv[dest] = min_cost
                new_ft[dest] = best_next_hop

So sánh với bảng cũ và cập nhật nếu có thay đổi:
+ Nếu bảng DV mới khác với bảng hiện tại thì cập nhật lại:
        if dict(new_dv) != dict(self.distance_vector):
            self.distance_vector = new_dv
            self.forwarding_table = new_ft
            updated = True

Trả về updated = True nếu có thay đổi, ngược lại False.

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
