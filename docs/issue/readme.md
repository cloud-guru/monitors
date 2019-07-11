Seft healing: bản chất là cơ chế phản ứng với sự cố trong openstack, với mục tiêu là khi có lỗi xảy ra trên máy chủ vật lý thì máy ảo và các dịch vụ cung cấp người dùng không- hoặc ít bị ảnh hưởng.<br/>

Ví dụ cho một cơ chế xử lý lỗi:

* Khi máy chủ vật lý gặp sự cố (tắt đột ngột, nic mất kết nối, disk low latency, cpu cao tải,vv…),hệ thống tự động di tản các máy ảo trên máy này sang một nốt khác để user tiếp tục sử dụng được máy ảo bình thường, minh họa như hình dưới

Mô tả ca sử dụng:

* User : cung cấp kịch bản cho việc tự động xử lý lỗi, kịch bản bao gồm:
  * Mô tả cảnh báo sẽ đến với hệ thống: là cảnh báo của tài nguyên vật lý hay ảo hóa. Đối tượng cụ thể cảnh báo chỉ ra là gì: nốt vật lý nào, card mạng nào, switch nào, hoặc máy ảo nào
  * Mô tả hệ thống cách ứng phó , ví dụ: gửi email cho quản trị, tự động thay thế máy ảo, di chuyển máy ảo về khu vực máy vật lý không có lỗi
* Hệ thống: Ghi lại các luật trên, khi có sự kiện cảnh báo xảy đến sẽ thực hiện đúng như mô tả của kịch bản. Chỉ ra được những kịch bản nào đã được áp dụng.
