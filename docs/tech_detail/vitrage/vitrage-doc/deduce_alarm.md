- Tổng quan:
Tem
Khi bắt đầu chạy các service của vitrage, thành phần Scenario Evaluator được gọi đến ngay sau khi tất cả datasoucrce gọi get-all() lấy về hết các element.
Evaluator gọi ngay khi có events đến với graph : (create/update/delete) trong các element (vertex/edge)

```
metadata:
 name: basic_template
 description: basic template for general tests
definitions:
 entities:
  - entity:
     category: ALARM
     template_id: alarm
     type: zabbix
     name: HOST_HIGH_CPU_LOAD
  - entity:
     category: RESOURCE
     template_id: resource
     type: nova.host
 relationships:
  - relationship:
     source: alarm
     target: resource
     relationship_type: on
     template_id : alarm_on_host
scenarios:
 - scenario:
    condition: alarm_on_host
    actions:
     - action:
        action_type: set_state
        properties:
         state: SUBOPTIMAL
        action_target:
         target: resource
```


- set_state: Two scenarios setting the state of the same resource
- raise_alarm: Two scenarios raising the same deduced alarm (with possibly different severity)
- add_causal_relationship: Two scenarios adding the same causal relationship

CRITICAL > SEVERE > WARNING > N/A > OK
DELETED > ERROR > SUBOPTIMAL > TRANSIENT > OK
```
- entity:
   category: RESOURCE
   type: nova.instance
   template_id: instance1
```
category ,type, template-id:

condition convert [DNF Disjunctive Normal Form](https://en.wikipedia.org/wiki/Disjunctive_normal_form) dạng [[and_var1, and_var2, …], or_list_2, …]. Tức ta có thể sử dụng các phép and, or , not (không thể viết 1 condition toàn phép not)
Use case 1:  
- Deduce-alarm: Nếu có alarm trên host (cpu cao vượt ngưỡng) => có alarm về cao CPU trên các instance của host đó => chuyển state của host và instance về WARNING
- RCA: alarm về cao CPU của host => alarm của instance và kiểm tra trong các trường hợp:
  - Alarm của instance là alarm deduce từ vitrage
  - Alarm của instance được báo về từ aodh

Use case 2: 
- Deduce-alarm Nếu có alarm trên port switch nối với host bị DOWN => có alarm mất kết nối trên instrance ảnh hưởng => chuyển state của host và instance về WARNING, ERROR
- RCA: alarm của port switch => alarm mất kết nối của instance
  - Alarm của instance là alarm deduce từ vitrage
  - Alarm của instance được báo về từ aodh

vitrage template validate --type standard --path /etc/vitrage/templates/port_switch_down_scenarios.yaml


?[--type {standard,definition,equivalence}]