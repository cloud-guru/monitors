### Cấu hình zabbix để nhận cảnh báo về 1 interface của switch down:

- 1.Tại zabbix UI, thêm host : vào Configuration > host > create host <br/>
Thông tin cơ bản <br/>
![a](../image/host_switch.PNG)
Thiết lập SNMP_COMMUNITY tại tab Macros
![a](../image/host_macros.PNG)
- 2.Tạo item của host vửa tạo :
  - Type: SNMP2 agent 
  - Key:  net.if.status[ifOperStatus.*] thay * bằng tên oid port cần monitor , vd net.if.status[ifOperStatus.10121]
  - Type of information: Numeric (unsigned); Data type: Decimal
  - Show Value: IF-MIB::ifOperStatus
  - Application: Network Interfaces
![a](../image/item2.PNG)

- 4.Tạo trigger:
  - Name: interface down
  - Severity: High
  - Expression: 
  {Interface Gi1-0-21:net.if.status[ifOperStatus.10121].last()}=2 <br/>
với Interface Gi1-0-21:net.if.status[ifOperStatus.10121] là item thêm ở trên . bật trigger khi giá trị bằng 2
![a](../image/trigger14.PNG)

