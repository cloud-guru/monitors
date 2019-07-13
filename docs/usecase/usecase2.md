# Nội dung kịch bản

Đầu vào monitor:

- Nic của host (qua TICK, prometheus, zabbix,..)
- Kết nối đến máy ảo. (vd ping đến máy đó)

Kịch bản: xuất hiện các alarm

- Alarm 1: nic host mất kết nối : nic_operstate {host= compute01, interface = ens8}= 0

- Alarm 2: Máy ảo mất kết nối (có thể do vitrage tự deduce ra, hoặc monitor tool phát hiện)

Yêu cầu:

- Collect được alarm khi xảy ra

- Vitrage nhân định Alarm 1 => Alarm 2

- Vitrage gọi đến mistral thực hiện seft healing

# Thực hiện
## Chuẩn bị monitor các thành phần:

Ở đây cần nic của host. Lựa chọn TICK  lấy alarm.

- Đầu tiên cần thiết lập cho telegraf lấy được metric host nic down: ta tạo file

!!! note "/usr/share/telegraf_check_nic.sh"
```
#!/bin/sh
#nics=`find /sys/class/net ! -type d | xargs --max-args=2 realpath  | awk -F\/ '/pci/{print $NF}'`
nics=`ip link  | grep "up" | awk -F": " {'print $2'}  | grep ^e`
hostname=`hostname`
for nic in $nics
do
  operstate=`cat /sys/class/net/$nic/operstate`
  if [ "$operstate" = "up" ]
  then
     field=1
  else
     field=0
  fi
  echo "nic,host=${hostname},interface=${nic} operstate=${field}"
done
```

sửa trong file config của telegraf:

!!! note "/etc/telegraf/telegraf.conf"
```

```

- Trên UI của zabbix server, ta cấu hình để monitor memory và application như sau:
- (*) Monitor instance memory: 
  - Ứng với mỗi instance tạo 1 zabbix host tương ứng
    - Hostname: tên instance.
    - Agent interface: interface của instance, kết nối zabbix server và agent
    - ![us1](image/use-case1-1.png)
  - Tạo item để thu thập thông tin memory của instance
    - Vào tab Configuration > host > [instance09] > item > create item 
      - Key: vm.memory.size(pushed) <br/>
       (Key này mang ý nghĩa:   % (active + wired) / total memory) <br/>
       Tham khảo tại: https://www.zabbix.com/documentation/3.2/manual/appendix/items/vm.memory.size_params
      - ![us1](image/use-case1-2.png)
  - Tạo trigger cảnh báo : khi mem dùng của instance vượt quá 90% thì bắn alarm
    - Vào tab: Configuration > host > [instance09] > Triggers > create trigger
    - ![us1](image/use-case1-3.png)
    - Ở đây avg(50s) tức nó sẽ xem xét giá trị trung bình trong khoảng 50s. Nếu cần trong 5 phút giá trị này cần đặt 1500
- (*)Monitor instance application:
  - Ta nhận biết application chết bằng cách kiểm tra có sự thay đổi pid không.
  - Tạo host: ứng với mỗi application tạo 1 zabbix host: <br/>
       Giả sử app ta muốn monitor là netcat <br/>
       ![us1](image/use-case1-4.png)
  - Lấy thông tin pid: thêm user parameter: <br/>
      Tạo file
        
    !!! note "/etc/zabbix/zabbix_agentd.d/userparameter_application.conf"
    
        UserParameter=application.pid[*],if [ "$(pidof $1)" = "" ]; then echo "-1"; else echo $(pidof $1); fi;
        

      - Vào tab Configuration > host > [instance09] > item > create item <br>
        với key: appication.pid.[*] thay * bằng tên process muốn monitor
        ![us1](image/use-case1-5.png)
  - Tạo trigger cảnh báo : khi app không chạy hoặc application pid bị đổi thì bắn alarm <br/>
     - application.pid [*] = -1 
     - OR application.pid [*].max(7d) <> application.pid [*].min(7d)
     - ![us1](image/use-case1-6.png)

## Cấu hình vitrage
- Map các alarm vào đồ thị:
  - Thêm entity app vào đồ thị, vd ta muốn thêm monitor vào 1 app “netcat”
  - Tạo file /etc/vitrage/static_datasources/app-netcat.yaml nội dung:
 
!!! note "app-netcat.yaml"

    ```---
    metadata:
      name: list of application run on instance
      description: list of application run on instance
    definitions:
      entities:
       - static_id: app-netcat
         type: application
         id: app-netcat
         state: available
       - static_id: instance9
         type: nova.instance
         id: eff1daed-0d97-4975-abbd-3d3e907aeedf
      relationships:
       - source: instance9
         target: app-netcat
         relationship_type: run
    ```
   
  - mapping cho alarm của zabbix vào đồ thị: <br/>
    Thêm vào file /etc/vitrage/zabbix_conf.yaml

!!! note "zabbix_conf.yaml"
  
    ```---
    - zabbix_host: instance09
      type: nova.instance
      name: eff1daed-0d97-4975-abbd-3d3e907aeedf
    - zabbix_host: app-netcat
      type: application
      name: app-netcat
    ```

  - Vậy ta đã chuẩn bị xong mô hình input, kết quả: <br/>
  - ![us1](image/use-case1-7.png)
  - Cấu hình root-cause-analys:
  - Thêm template <br/>
    Tạo file template /etc/vitrage/templates/usecase-1.yaml

!!! note "usecase-1.yaml"
    
    ```---
    metadata:
      name: rca application died caused by high mem on instance
      description: rca application died caused by high mem on instance
    definitions:
      entities:
          - entity:
              template_id: alarm_high_memory_used
              category: ALARM
              name: instance high memory usage
          - entity:
              template_id: alarm_application_died
              category: ALARM
              name: application died
          - entity:
              template_id: instance
              category: RESOURCE
              type: nova.instance
          - entity:
              template_id: application
              category: RESOURCE
              type: application
      relationships:
          - relationship:
              template_id : alarm_high_memory_on_instance
              source: alarm_high_memory_used
              target: instance
              relationship_type: on
          - relationship:
              template_id : alarm_application_died_on_applicaion
              source: alarm_application_died
              target: application
              relationship_type: on
          - relationship:
              template_id : instance_run_application
              source: instance
              target: application
              relationship_type: run
    scenarios:
        - scenario:
            condition: alarm_high_memory_on_instance and alarm_application_died_on_applicaion and instance_run_application
            actions:
                - action:
                    action_type : add_causal_relationship
                    action_target:
                      source: alarm_high_memory_used
                      target: alarm_application_died
    ```

  - Chạy lệnh:

```bat
$ vitrage template validate --type standard --path /etc/vitrage/templates/usecase1.yaml
$ vitrage template add --type standard --path /etc/vitrage/templates/usecase1.yaml
```

  - Kết quả sau khi add template:
  - ![us1](image/use-case1-8.png)

       




