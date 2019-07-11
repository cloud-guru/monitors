# Mistral
Mistral -  OpenStack workflow service, là dịch vụ giúp người dùng có khả năng định nghĩa, thực hiện và quản lý một nhiệm vụ hoặc một chuỗi hành động. Nó cũng định nghĩa chuỗi hành động này dưới dạng template như heat và khi thực hiện sẽ gọi đến các API của openstack, một loại dịch vụ giúp người dùng quản lý tự động các tài nguyên của mình bằng code , hay “infrastructure as code”. Nhưng khác với heat, một nó tập trung vào chuỗi hành động: bao gồm các nhiệm vụ nhỏ nhỏ hoạt động theo một cấu trúc điều khiển dạng nếu-thì IFTTT (if this, then that) như một ngồn ngữ lập trình

So sánh với Heat: heat giúp người dùng chạy kịch bản mô tả về một nhóm các tài nguyên ảo hóa, còn mistral giúp người dùng chạy kịch bản về một chuỗi các hành động . Ví dụ cùng muốn tạo máy ảo nhưng nhưng mẫu mô tả mà người dùng đưa vào của 2 bên sẽ khác biệt:

* Người sử dụng heat sẽ cung cấp mô tả: loại tài nguyên cần là gì, số lượng và mối quan hệ giữa chúng như thế nào
* Người của mistral sẽ cung cấp mô tả:
  * hành động 1: nova.create-instance  
  * hành động 2: cinder.create volume
  * hành động 3: cinder.attach-volume
  * if: nếu thành công thì gửi mail thông báo
  * if: nếu thất bại, thử lại chuỗi hành động trên 5 lần

Vai trò của mistral trong hệ thống:
* Mistral giúp ích cho nhà cung cấp dịch vụ đám mây đặt các kịch bản để tự động các thao tác với hệ thống.
* Mistral kết hợp với vitrage giúp thực hiện quá trình vượt qua lỗi cho hệ thống , cụ thể:

Ý tưởng ở đây là: khi cảnh báo về máy vật lý  được gửi đến -> Vitrage suy luận về tác động của cảnh báo, gọi mistral -> Mistral tác động lại với các tài nguyên ảo hóa bị ảnh hưởng.




