### Cấu hình để khi triger zabbix bật lên sẽ gửi cảnh báo về vitrage ( gửi vào rabbitmq)
- Thêm media type: media mà sẽ thông báo vào messeage queue của openstack , topic vitrage_notification <br/>
Đây là dạng media sẽ thực hiện chạy script , ta dowload script về , đưa vào thư mục /usr/lib/zabbix/alertscripts/

```
$ wget https://raw.githubusercontent.com/openstack/vitrage/master/vitrage/datasources/zabbix/auxiliary/zabbix_vitrage.py
$ cp zabbix_vitrage.py /usr/lib/zabbix/alertscripts/
$ chmod 755 /usr/lib/zabbix/alertscripts/zabbix_vitrage.py
```

- Từ zabbix ui : [Administration > Media Types > Create Media Type]

Name: Vitrage Notifications

Type: Script

Script name: zabbix_vitrage.py

Script parameters:

1st line: {ALERT.SENDTO}

2nd line: {ALERT.SUBJECT}

3rd line: {ALERT.MESSAGE}

![a](../image/mediatype1.PNG)

- Add media-type này cho admin user [Administration > Users]

Type: Vitrage Notifications

Send to: rabbit://rabbit_user:rabbit_pass@10.60.117.159:5672/ <— Vitrage message bus url

When active: 1-7,00:00-24:00

Use if severity: tick all options

Status: Enabled


![a](../image/mediatype2.PNG)
- Thêm action cho các trigger: [Configuration > Actions > Create Action > Action]

Name: Forward to Vitrage

Default Subject: {TRIGGER.STATUS}

Add an operation:

Send to Users: Admin

Send only to: Vitrage Notifications

Default Message:

host={HOST.NAME1}

hostid={HOST.ID1}

hostip={HOST.IP1}

triggerid={TRIGGER.ID}

description={TRIGGER.NAME}

rawtext={TRIGGER.NAME.ORIG}

expression={TRIGGER.EXPRESSION}

value={TRIGGER.VALUE}

priority={TRIGGER.NSEVERITY}

lastchange={EVENT.DATE} {EVENT.TIME}

To send events add under the Conditions tab:

(A) Maintenance status not in maintenance
![a](../image/action1.PNG)
![a](../image/action2.PNG)

### Ref:
https://github.com/openstack/vitrage/blob/master/doc/source/contributor/zabbix_vitrage.rst
