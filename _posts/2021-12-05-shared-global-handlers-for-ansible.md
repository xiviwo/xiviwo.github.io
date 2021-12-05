---
layout: post
title:  "Shared or Global Handlers for Ansible"
date:   2021-12-05 13:03:50 +0800
categories: Ansible handlers global shared
---

I am not sure is there anything like this for ansible: I need a global shared handlers, which I can call anywhere I want, ie, from playbook, from any roles inside. I can't find any official document or tutorial anywhere regarding my use case, so I have to cook up myself, and share here. Hopefully it can help someone who need it. 

## The dilemma of normal ansible handler 

#### Normal handler is too simple to handle complicate situation

For example, for apache handler, it could be as simple as this
```yaml
handlers:
  - name: Restart apache
    ansible.builtin.service:
      name: httpd
      state: restarted
```
In reality, for some kind of service, it fast more complicated than this:

```yaml
## Fake codes for demo purpose
handlers:
  - name: Restart complicated service
    block:
      - name: stop complicated service
        command: "complicate_service stop" 

      - name: Check until process is stop 
        command: "ps -elf | grep java -c" 
        retry: 100 
        register: cmd_out
        until: cmd_out.stdout == "0"

      - name: Start complicated service
        command: "complicate_service start"

      - name: wait for until it fully available
        wait_for:
          port: "{{ complicate_service_port }}
      #.... something more like this
```

So, in this case, the better way is to extract the code into a separate task and include that:

```yaml
- name: restart  
  include_tasks: restart_demo.yml 
```

At first place, I believe the separate task `restart_demo.yml` should reside the same folder as `handlers`, something like this: 

```sh
demo_service/
├── handlers
│   ├── main.yml
│   ├── restart_demo.yml
│   ├── start_demo.yml
│   └── stop_demo.yml
└── tasks
    └── main.yml
```

But, as matter of fact, all those tasks can be placed side by side with `tasks` folder, which gave way to a more reusable approach to be discussed later. 

```sh
demo_service
├── handlers
│   └── main.yml
└── tasks
    ├── main.yml
    ├── restart_demo.yml
    ├── start_demo.yml
    └── stop_demo.yml
```

### Sharing handler is a pain, shared handler role come to rescue 

What if we need to call same handler in another role ? 
Simple way is to copy the handlers folder and tasks files alongside into the role's folder.
No doubt, it's the dumbest thing ever to do. 
It will cause duplicate code here and there, very hard to manage. 

The best way I am aware of is to create a role to share common handler among other roles.
There are multiple ways to reference shared handlers role in other roles or tasks:
* reference by include_role
  ```yaml
  - include_role:
      name: demo_service
    tags: always 
  ```
* reference by role 
  ```yaml
  roles:
    - { role: demo_service, tags: always } 
  ```
* reference by dependencies 
  `meta/main.yml`
  ```yaml
  dependencies:
    - role: demo_service
      tags: always
  ```


> :Warning: **Always TAGS Your shared handler as `always` to make sure it always run **

But, it leads to one problem: the main task `main.yml` will always run as well when the role is included or referred. 

```sh
   └── tasks
        ├── main.yml
```

One possible way is to add one placeholder block to only execute when specific `vars` is provided.
For example, nothing will be executed unless we provide `service_name` variable, but the handlers inside is always included.

```yaml
- name: Service Placeholder
  debug:
    msg: "{{ service_name }}"
  changed_when: true 
  when: service_name is defined
  notify: "{{service_name}}"
```

### Need to reuse handler code as `task` not as `handler`

In some rare case, we need to call the handler as `task` to reuse code as much as possible. 

For example, we need to stop service ***immediately*** before some task or some depended service must be restarted **immediately** to make the change to take effect, so that the depending step can proceed. 

**immediately stop demo service**
```yaml
include_role:
  name: demo_service
  tasks_from: stop_demo.yml
```

**immediately restart demo service**
```yaml
include_role:
  name: demo_service
  tasks_from: restart_demo.yml
```


As mentioned before, we place those task files along side the `main.yml`, so it can be easily resued by `include_role` and `tasks_from` as above. 
```sh
demo_service
├── handlers
│   └── main.yml
└── tasks
    ├── main.yml
    ├── restart_demo.yml
    ├── start_demo.yml
    └── stop_demo.yml
```

## A Full Demo Code for Scenarios Discussed Above
```yaml
---
- name: Shared handler Scenario 1
  hosts: testhost
  become: true
  gather_facts: no
  tasks:
    # Must include demo_service role, so that shared handler can be found
    - include_role:
        name: demo_service
      tags: always 

    - name: "Something change, need to notify to restart demo"
      shell: "true"
      notify: restart demo


- name: Shared handler Scenario 2
  hosts: testhost
  become: true
  gather_facts: no
  tasks:
    - name: Restart demo service immediately without waiting for handler(case 1)
      include_role:
        name: demo_service
        tasks_from: restart_demo.yml

    - name: Restart demo service directly with handler (case 2)
      include_role:
        name: demo_service
      vars:
        service_name: restart demo


- name: Shared handler Scenario 3
  hosts: testhost
  become: true
  gather_facts: no
  roles:
    # Must include demo_service role, so that shared handler can be found
    - { role: demo_service, tags: always } 
    - { role: call_shared_handler_role, tags: test }
```
## Share Demo Service as Handler and Task Playbook Run

```sh
ansible-playbook test_handlers.yml -i inventory/inventory 

PLAY [Shared handler Scenario 1] *******************************************************************

TASK [include_role : demo_service] *****************************************************************

TASK [demo_service : Service Placeholder] **********************************************************
skipping: [testhost]

TASK [Something change, need to notify to restart demo] ********************************************
changed: [testhost]

RUNNING HANDLER [demo_service : restart demo] ******************************************************
included: /home/tom/ansible-shared-handlers/roles/demo_service/tasks/restart_demo.yml for testhost

RUNNING HANDLER [demo_service : include_tasks] *****************************************************
included: /home/tom/ansible-shared-handlers/roles/demo_service/tasks/stop_demo.yml for testhost

RUNNING HANDLER [demo_service : Stop very complicate service] **************************************
ok: [testhost] => {
    "msg": "Stop complicate demo service successfully "
}

RUNNING HANDLER [demo_service : include_tasks] *****************************************************
included: /home/tom/ansible-shared-handlers/roles/demo_service/tasks/start_demo.yml for testhost

RUNNING HANDLER [demo_service : Start very complicate service] *************************************
ok: [testhost] => {
    "msg": "start complicate demo service successfully"
}

PLAY [Shared handler Scenario 2] *******************************************************************

TASK [Restart demo service immediately without waiting for handler(case 1)] ************************

TASK [demo_service : include_tasks] ****************************************************************
included: /home/tom/ansible-shared-handlers/roles/demo_service/tasks/stop_demo.yml for testhost

TASK [demo_service : Stop very complicate service] *************************************************
ok: [testhost] => {
    "msg": "Stop complicate demo service successfully "
}

TASK [demo_service : include_tasks] ****************************************************************
included: /home/tom/ansible-shared-handlers/roles/demo_service/tasks/start_demo.yml for testhost

TASK [demo_service : Start very complicate service] ************************************************
ok: [testhost] => {
    "msg": "start complicate demo service successfully"
}

TASK [Restart demo service directly with handler (case 2)] *****************************************

TASK [demo_service : Service Placeholder] **********************************************************
changed: [testhost] => {
    "msg": "restart demo"
}

RUNNING HANDLER [demo_service : restart demo] ******************************************************
included: /home/tom/ansible-shared-handlers/roles/demo_service/tasks/restart_demo.yml for testhost

RUNNING HANDLER [demo_service : include_tasks] *****************************************************
included: /home/tom/ansible-shared-handlers/roles/demo_service/tasks/stop_demo.yml for testhost

RUNNING HANDLER [demo_service : Stop very complicate service] **************************************
ok: [testhost] => {
    "msg": "Stop complicate demo service successfully "
}

RUNNING HANDLER [demo_service : include_tasks] *****************************************************
included: /home/tom/ansible-shared-handlers/roles/demo_service/tasks/start_demo.yml for testhost

RUNNING HANDLER [demo_service : Start very complicate service] *************************************
ok: [testhost] => {
    "msg": "start complicate demo service successfully"
}

PLAY [Shared handler Scenario 3] *******************************************************************

TASK [demo_service : Service Placeholder] **********************************************************
skipping: [testhost]

TASK [call_shared_handler_role : Something change inside role, need to notify to restart demo service] ***
changed: [testhost]

RUNNING HANDLER [demo_service : restart demo] ******************************************************
included: /home/tom/ansible-shared-handlers/roles/demo_service/tasks/restart_demo.yml for testhost

RUNNING HANDLER [demo_service : include_tasks] *****************************************************
included: /home/tom/ansible-shared-handlers/roles/demo_service/tasks/stop_demo.yml for testhost

RUNNING HANDLER [demo_service : Stop very complicate service] **************************************
ok: [testhost] => {
    "msg": "Stop complicate demo service successfully "
}

RUNNING HANDLER [demo_service : include_tasks] *****************************************************
included: /home/tom/ansible-shared-handlers/roles/demo_service/tasks/start_demo.yml for testhost

RUNNING HANDLER [demo_service : Start very complicate service] *************************************
ok: [testhost] => {
    "msg": "start complicate demo service successfully"
}

PLAY RECAP *****************************************************************************************
testhost                   : ok=22   changed=3    unreachable=0    failed=0    skipped=2    rescued=0    ignored=0   
```

Full code avaible [here][code]

[code]:https://github.com/xiviwo/ansible-shared-handlers-demo.git