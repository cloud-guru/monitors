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
[[inputs.exec]]
  ## Commands array
  commands = [
    "/usr/share/telegraf_check_nic.sh",
  ]
  timeout = "5s"
  data_format = "influx"
```

Như vậy , telegraf sẽ liên tục gửi về cho influxdb metric định dạng:

  - nic,host=compute01,interface=ens8 operstate= 0 nếu nic ens8 down
  - nic,host=compute01,interface=ens8 operstate= 1 nếu nic ens8 up

- Tiếp theo cần thiết lập cảnh báo kapacitor:
Tạo 1 tick script:

!!! note "/etc/kapacitor/tick_script_nicdown.txt"
```
dbrp "telegraf"."autogen"
var db = 'telegraf'
var rp = 'autogen'
var measurement = 'nic'
var groupBy = ['host', 'interface']
var whereFilter = lambda: TRUE
var name = 'nic down'
var idVar = name + '-{{.Group}}'
var message = ' {{ index .Tags "interface" }} down'
var idTag = 'alertID'
var levelTag = 'level'
var messageField = 'message'
var durationField = 'duration'
var outputMeasurement = 'alerts'
var triggerType = 'threshold'
var crit = 1
var data = stream
    |from()
        .database(db)
        .retentionPolicy(rp)
        .measurement(measurement)
        .groupBy(groupBy)
        .where(whereFilter)
    |eval(lambda: "operstate")
        .as('value')
var trigger = data
    |alert()
        .crit(lambda: "value" < crit)
        .message(message)
        .id(idVar)
        .idTag(idTag)
        .levelTag(levelTag)
        .messageField(messageField)
        .durationField(durationField)
        .stateChangesOnly()
        .exec('/usr/bin/python', '/etc/kapacitor/kapacitor_vitrage.py', 'rabbit://openstack:Welcome123@192.168.77.51')
trigger
    |httpOut('output')
```

Import tick_script trên cho kapacitor:

```
kapacitor define check_nic_down -tick /etc/kapacitor/tick_script_nicdown.txt
kapacitor endnable check_nic_down
```

- Thực hiện cấu hình cho kapacitor trả về alarm được cho vitrage: https://docs.openstack.org/vitrage/latest/contributor/kapacitor-datasource.html
- Kết quả sau khi đặt monitor nic xong: thử ip link down ens8:
     - ![b12](image/usecase2_2.png)
     

## Cấu hình vitrage
- Thêm kịch bản để khi có alarm nic của host down sẽ xác định cho host nào bị ảnh hưởng.
 
!!! note "scenario2-1.yaml"
```
metadata:
    version: 2
    name: kapacitor host nic alarm to nic alarm
    type: standard
    description: kapacitor host nic alarm to nic alarm
definitions:
    entities:
        - entity:
            category: ALARM
            name.regex: (.ens8) down
            template_id: host_alarm
        - entity:
            category: RESOURCE
            type: host.nic
            template_id: nic
        - entity:
            category: RESOURCE
            type: nova.host
            template_id: host
    relationships:
        - relationship:
            source: host_alarm
            target: host
            relationship_type: on
            template_id : alarm_on_host

        - relationship:
            source: nic
            target: host
            relationship_type: attach
            template_id : nic_attach_host

scenarios:
    - scenario:
        condition: alarm_on_host and nic_attach_host
        actions:
            - action:
               action_type : raise_alarm
               properties:
                  alarm_name: down
                  severity: critical
               action_target:
                  target: nic
```

!!! note "scenario2-2.yaml"
```
metadata:
    version: 2
    name: nic down scenario 
    type: standard
    description: nic down scenario 
definitions:
    entities:
        - entity:
            category: ALARM
            name: down
            template_id: nic_alarm
        - entity:
            category: RESOURCE
            type: neutron.network
            template_id: net
        - entity:
            category: RESOURCE
            type: host.nic
            template_id: nic
        - entity:
            category: RESOURCE
            type: neutron.port
            template_id: port

        - entity:
            category: RESOURCE
            type: nova.instance
            template_id: instance

        - entity:
            category: RESOURCE
            type: nova.host
            template_id: host

        - entity:
            category: ALARM
            name: disconnect
            template_id: instance_alarm


    relationships:
        - relationship:
            source: nic_alarm
            target: nic
            relationship_type: on
            template_id : alarm_on_nic

        - relationship:
            source: nic
            target: net
            relationship_type: provide
            template_id : nic_provide_net

        - relationship:
            source: net
            target: port
            relationship_type: contains 
            template_id : net_contains_port

        - relationship:
            source: port
            target: instance
            relationship_type: attached 
            template_id : port_attached_instance

        - relationship:
            source: host
            target: instance
            relationship_type: contains
            template_id : host_contains_instance

        - relationship:
            source: nic
            target: host
            relationship_type: attach
            template_id : nic_attach_host

        - relationship:
            source: instance_alarm
            target: instance
            relationship_type: on
            template_id : alarm_on_instance



scenarios:
    - scenario:
        condition: alarm_on_nic and nic_provide_net and net_contains_port and port_attached_instance and host_contains_instance and nic_attach_host
        actions:
            - action:
               action_type : raise_alarm
               properties:
                  alarm_name: disconnect
                  severity: critical
               action_target:
                  target: instance
            - action:
               action_type : execute_mistral
               properties:
                  workflow: instance_live_migrate
                  input:
                    instance_id: get_attr(instance, id)
    - scenario:
        condition: alarm_on_nic and host_contains_instance and nic_attach_host and alarm_on_instance
        actions:
            - action:
               action_type : add_causal_relationship
               action_target:
                  source: nic_alarm
                  target: instance_alarm
               
```

- Chạy lệnh:

```bash
$ vitrage template validate --type standard --path /etc/vitrage/templates/scenario2-1.yaml
$ vitrage template add --type standard --path /etc/vitrage/templates/scenario2-1.yaml
$ vitrage template validate --type standard --path /etc/vitrage/templates/scenario2-2.yaml
$ vitrage template add --type standard --path /etc/vitrage/templates/scenario2-2.yaml
```
## Cấu hình mistral

Thêm workflow instance_live_migrate sau:

!!! note "workflow_live_migrate.yaml"
```
version: '2.0'
instance_live_migrate:
 type: direct
 input:
   - instance_id
   - notifi_to_email: admin-openstack@yopmail.com
 tasks:
   get_instance_status_before:
     action: nova.servers_get server=<% $.instance_id %>
     publish:
       instance_name: <% task(get_instance_status_before).result.name %>
       status_before: <% task(get_instance_status_before).result.status %>
       host_before: <% task(get_instance_status_before).result["OS-EXT-SRV-ATTR:host"] %>
     on-success: live_migrate_instance
     on-error: send_error_email

   live_migrate_instance:
      action: nova.servers_live_migrate
      input:
        server: <% $.instance_id %>
        block_migration: False
        disk_over_commit: False
        host:
      retry:
        count: 10
        delay: 30
      on-success: wait_for_instance_migrate
      on-error: send_error_email

   wait_for_instance_migrate:
     action: nova.servers_get server=<% $.instance_id %>
     retry:
       count: 30
       delay: 2
       continue-on: <% task(wait_for_instance_migrate).result.status !="MIGRATING" %>
     on-success: wait_instance_status_active

   wait_instance_status_active:
     action: nova.servers_get server=<% $.instance_id %>
     retry:
       delay: 10
       count: 30
       continue-on: <% task(wait_instance_status_active).result.status !="ACTIVE" %>
     on-success: get_instance_status_after

   get_instance_status_after:
     action: nova.servers_get server=<% $.instance_id %>
     publish:
       status_after: <% task(get_instance_status_after).result.status %>
       host_after: <% task(get_instance_status_after).result["OS-EXT-SRV-ATTR:host"] %>
     on-complete: check_diffrent_host
   check_diffrent_host:
     action: std.noop
     on-complete:
       - send_error_email: <% $.status_before != $.status_after or $.host_before = $.host_after %>

   send_error_email:
     action: std.email
     input:
       to_addrs: [<% $.notifi_to_email %>]
       subject: ERROR live migrate virtual machine
       body: |
        We try to migrate vm <% $.instance_id %> when host have problem
         Please look at mistral workflow <% execution().id %> for more detail
       from_addr: ngohoa211@gmail.com
       smtp_server: smtp.gmail.com
       smtp_password: secret
```

## Kết quả 
- Khi tắt card mạng, đầu tiên, sẽ có vitrage deduce alarm trên máy báo hiệu máy ảo mất kết nối
![us1](image/usecase2_4.png)
- Sau đó, hệ thống tự động chuyển máy ảo sang host khác
- ![us17](image/usecase2_5.png)




