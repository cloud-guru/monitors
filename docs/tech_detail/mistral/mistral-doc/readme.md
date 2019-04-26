host evacuate 
---
---
version: '2.0'
host-evacuate:
  type: direct
  input:
    - host_evacuate
  tasks:
    get_service_nova_compute_work:
      wait-before: 3 #time wait nova-compute down confirmed
      action: nova.services_list
      publish:
        nova_compute_work: <% task(get_service_nova_compute_work).result.where($.binary = "nova-compute").where($.state = "up").host %>
      on-complete: controll_flow

    controll_flow:
      action: std.noop
      on-complete:
        - send_error_email_1: <% $.host_evacuate in $.nova_compute_work %> # compute up
        - send_error_email_2: <% not bool($.nova_compute_work) %> # list empty
        - list_vms: <% (not $.host_evacuate in $.nova_compute_work) and bool($.nova_compute_work) %>

    send_error_email_1:
      action: std.echo
      input:
        output: "Fail when evacuate host <% $.host_evacuate %> caused by nova-compute on host not down"

    send_error_email_2:
      action: std.echo
      input:
        output: "Fail when evacuate host <% $.host_evacuate %> caused by there are no other host compute up"

    list_vms:
       action: nova.servers_list
       input:
         search_opts:
           {"host": <% $.host_evacuate %>}
       on-success: evacuate_instances

# instance in error state can't evacuate
    evacuate_instances:
       with-items: vm in <% task(list_vms).result.where($.status != "ERROR") %>
       workflow: instance_evacuate
       input :
         instance_id: <% $.vm.id %>

    # evacuate_instances:
    #    with-items: vm in <% $.vms %>
    #    workflow: instance_evacuate
    #    input :
    #      instance_id: <% $.vm.id %>
    #      instance_status_expect: <% $.vm.status %>

---
instance_evacuate

---
version: '2.0'
instance_evacuate:
  type: direct
  input:
    - instance_id
  tasks:
    get_instance_status_before:
      action: nova.servers_find id=<% $.instance_id %>
      publish:
        instance_name: <% task(get_instance_status_before).result.name %>
        status_before: <% task(get_instance_status_before).result.status %>
        host_before: <% task(get_instance_status_before).result["OS-EXT-SRV-ATTR:host"] %>
      on-success:
        - send_fail_email_0: <% $.status_before = "ERROR" %>
        - fail: <% $.status_before = "ERROR" %>
        - evacuate_instance: <% $.status_before != "ERROR" %>
      on-error: send_fail_email_1

    evacuate_instance:
       action: nova.servers_evacuate server=<% $.instance_id %>
       retry:
         delay: 10
         count: 10
       on-success: wait_for_instance_rebuild
       on-error: get_instance_status_after

    wait_for_instance_rebuild:
      action: nova.servers_find id=<% $.instance_id %> status="REBUILD"
      retry:
        delay: 2
        count: 30
      on-success: wait_instance_status_active
      on-error: get_instance_status_after

    wait_instance_status_active:
      action: nova.servers_find id=<% $.instance_id %> status="ACTIVE"
      retry:
        delay: 10
        count: 30
      on-complete: get_instance_status_after

    get_instance_status_after:
      action: nova.servers_find id=<% $.instance_id %>
      publish:
        status_after: <% task(get_instance_status_after).result.status %>
        host_after: <% task(get_instance_status_after).result["OS-EXT-SRV-ATTR:host"] %>
      on-complete: check_status_not_error

    check_status_not_error:
      action: std.noop
      on-complete: 
        - check_status_same_as_status_before : <% $.status_after != "ERROR" %>
        - send_fail_email_2: <% $.status_after = "ERROR" %>

    check_status_same_as_status_before:
      action: std.noop
      on-complete: 
        - check_diffrent_host : <% $.status_after = $.status_before %>
        - send_fail_email_3: <% $.status_after != $.status_before %>

    check_diffrent_host:
      action: std.noop
      on-complete:
        - test_vm: <% $.host_before != $.host_after %>
        - send_fail_email_4: <% $.host_before != $.host_after %>
        
    test_vm:
      workflow: verify_server server=<% $.instance_id %>
      publish:
        instance_pass_test: <% task(test_vm).result.instance_pass_test %>
      on-success: 
        - send_success_email: <% task(test_vm).result.instance_pass_test = true %>
        - send_fail_email_5: <% task(test_vm).result.instance_pass_test = false %>
    
    send_fail_email_0:
      action: std.echo
      input:
        output: |
          Fail when evacuate instance, instance status before workflow is ERROR
          Instance before workflow: [ name: <% $.instance_name %> ; status: <% $.status_before %> ; host: <% $.host_before %> ].

    send_fail_email_1:
      action: std.echo
      input:
        output: |
          Fail when evacuate instance, instance not found with id <% $.instance_id %>

    send_fail_email_2:
      action: std.echo
      input:
        output: |
          Fail when evacuate instance, instance status after workflow is ERROR
          Instance before workflow: [ name: <% $.instance_name %> ; status: <% $.status_before %> ; host: <% $.host_before %> ].
          Instance after workflow: [ name: <% $.instance_name %> ; status: <% $.status_after %> ; host: <% $.host_after %> ] 
          Workflow id: <% execution().id %>
 
    send_fail_email_3:
      action: std.echo
      input:
        output: |
          Fail when evacuate instance, instance status after workflow is not same as before workflow
          Instance before workflow: [ name: <% $.instance_name %> ; status: <% $.status_before %> ; host: <% $.host_before %> ].
          Instance after workflow: [ name: <% $.instance_name %> ; status: <% $.status_after %> ; host: <% $.host_after %> ] 
          Workflow id: <% execution().id %>   

    send_fail_email_4:
      action: std.echo
      input:
        output: |
          Fail when evacuate instance, instance after workflow not move to diffrent host
          Instance before workflow: [name: <% $.instance_name %> ; status_before: <% $.status_before %> ; host_before: <% $.host_before %> ].
          Instance after workflow: [ name: <% $.instance_name %> ; status_after: <% $.status_after %> ; host_after: <% $.host_after %> ]
          Instance pass test: false
          Workflow id: <% execution().id %> 

    send_fail_email_5:
      action: std.echo
      input:
        output: |
          Fail when evacuate instance, instance after workflow 
          Instance before workflow: [name: <% $.instance_name %> ; status_before: <% $.status_before %> ; host_before: <% $.host_before %> ].
          Instance after workflow: [ name: <% $.instance_name %> ; status_after: <% $.status_after %> ; host_after: <% $.host_after %> ]
          Instance pass test: false
          Workflow id: <% execution().id %> 
    

    send_success_email:
      action: std.echo
      input:
       output: |
          Success evacuate instance : 
          Instance before workflow: [ name: <% $.instance_name %> ; status_before: <% $.status_before %> ; host_before: <% $.host_before %> ].
          Instance after workflow: [ name: <% $.instance_name %> ; status_after: <% $.status_after %> ; host_after: <% $.host_after %> ]
          Instance pass test: strue
          Workflow id: <% execution().id %> 

---
instance_live_migrate:
---
version: '2.0'
instance_live_migrate:
  description: migrate VMs
  type: direct
  input:
    - instance_id
    - target_host:
    - block_migration: False
    - disk_over_commit: False
  output:
    old_host: <% $.old_host %>
    new_host: <% $.new_host %>
    instance_work: <% $.instance_work %>
  tasks:
    get_host_contain_instance_before:
      action: nova.servers_find id=<% $.instance_id %>
      publish:
        old_host: <% task(get_host_contain_instance_before).result["OS-EXT-SRV-ATTR:host"] %>
      on-success: live_migrate_instance

    live_migrate_instance:
       action: nova.servers_live_migrate
       input:
         server: <% $.instance_id %>
         host: <% $.target_host %>
         block_migration: <% $.block_migration %>
         disk_over_commit: <% $.disk_over_commit %>
       retry:
         delay: 10
         count: 10
       on-success: wait_for_instance_migrating

    wait_for_instance_migrating:
      action: nova.servers_find id=<% $.instance_id %> status="MIGRATING"
      retry:
        delay: 3
        count: 30
      on-success: wait_for_instance_status_active
      on-error: send_error_email

    wait_for_instance_status_active:
      action: nova.servers_find id=<% $.instance_id %> status="ACTIVE"
      retry:
        delay: 10
        count: 30
      on-success: get_host_contain_vm
      on-error: send_error_email

    get_host_contain_vm:
      description: get name vm's host
      action: nova.servers_find id=<% $.instance_id %>
      publish:
        new_host: <% task(get_host_contain_vm).result["OS-EXT-SRV-ATTR:host"] %>
      on-success: check_vm_work
      on-error: send_error_email

    check_vm_work:
      workflow: verify_server server=<% $.instance_id %>
      publish:
        instance_work: <% task(check_vm_work).result.instance_work %>
      on-success: 
        - send_success_email: <% task(check_vm_work).result.instance_work = true %>
        - send_error_email: <% task(check_vm_work).result.instance_work = false %>

    send_error_email:
      action: std.echo output="fail"

    send_success_email:
      action: std.echo output="success"

