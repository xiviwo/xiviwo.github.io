---
layout: post
title:  "How to skip tags by default based on environment for ansible"
date:   2021-12-02 21:25:50 +0800
categories: Ansible
---

It has been quite a while since last post, as I move to a new company where I can't easily sync anything to github due to strict security policy, of course, there is nothing relative any policy here. So I have to type something all from my memory. Oh, God. 

### My Use Case 
I have one playbook which needs to skip `dev_only` tag for `prod` environment and likewise, skip `prod_only` for `dev` environment. So I wish there were some built-in ansible magic variable(s) that I can set skip-tags by default. Unfortunately, as far as I know, there is no such thing for ansible. Yes, there is one magic variable: `ansible_skip_tags`, but it's read-only variable, you can see what tags are being skipped, but you can't set it. 

After some digging, there are two possible workaround, may not be elegant, but it works. 

### Skip Tasks by default with variable(tags)

Suppose there are two groups for your playbook, namely, `prod` and `dev`, which stands for your group vars for `prod env` and `dev env`, obviously.
```sh
group_vars/
├── dev.yml
└── prod.yml

0 directories, 2 files
```
We need one variable with same name to be set for two env. 
For example, for `dev.yml`.

```yaml
default_tag_name: dev
```
On the contrary, for `prod.yml`: 

```yaml
default_tag_name: prod
```

In your playbook, then, you can simply skips those tasks by env with: 

```yaml
- name: Add the user 'johnd' only for dev env
  ansible.builtin.user:
    name: johnd
    comment: John Doe
    uid: 1040
    group: admin
  when: default_tag_name == 'dev'

- name: Add the user 'dave' only for prod env
  ansible.builtin.user:
    name: dave 
    comment: dave Jone
    uid: 1045
    group: admin
  when: default_tag_name == 'prod'
```

### Create a role to ensure correct tags to skip 
Similarly, we need to define one vars as `default_skip_tags`.

For `dev.yml`.

```yaml
default_skip_tags: prod_only
```
For `prod.yml`: 

```yaml
default_skip_tags: dev_only
```

Create a role with name `ensure_tag`
```sh
roles
├── ensure_tag
│   └── tasks
│       └── main.yml

```
We can come up something like this.
`main.yml` 
```yaml
---
- name: Fail if default_skip_tags is not defined
  fail:
    msg: Variable default_skip_tags is not defined
  when: default_skip_tags is not defined 

- name: Fail if no skip tags is specified
  fail:
    msg: You don't specify skip tags with "--skip-tags"
  when: 
    - ansible_skip_tags is not defined
    - ansible_skip_tags | length == 0

- name: Fail if skip tags is not correct 
  fail:
    msg: Skip tags switch is incorrect, please specify "--skip-tags '{{ default_skip_tags }}'"
  when: default_skip_tags not in ansible_skip_tags
```

Then include this role as `always` for any playbook need to ensure correct skip tags.

```yaml
- name: test ensure_tag role
  hosts: dev_host
  roles:
    - { role: ensure_tag, tags: always }
```

So, if you specify the wrong tags or forget to provide the tags, it will let your know, in case you will shoot your toe by mistake.

```sh
ansible-playbook test_role.yml -i inventory/inventory -v

PLAY [test ensure_tag role] ************************************************************************************************************************************

TASK [Gathering Facts] *****************************************************************************************************************************************
ok: [dev_host]

TASK [ensure_tag : Fail if default_skip_tags is not defined] ***************************************************************************************************
skipping: [dev_host] => changed=false 
  skip_reason: Conditional result was False

TASK [ensure_tag : Fail if no skip tags is specified] **********************************************************************************************************
skipping: [dev_host] => changed=false 
  skip_reason: Conditional result was False

TASK [ensure_tag : Fail if skip tags is not correct] ***********************************************************************************************************
fatal: [dev_host]: FAILED! => changed=false 
  msg: Skip tags switch is incorrect, please specify "--skip-tags 'prod_only'"

PLAY RECAP *****************************************************************************************************************************************************
dev_host                    : ok=1    changed=0    unreachable=0    failed=1    skipped=2    rescued=0    ignored=0  
```

Give you all green if correct switch and tags are provides,(^:^)

```sh
ansible-playbook test_role.yml -i inventory/inventory -v --skip-tags 'prod_only'

PLAY [test ensure_tag role] ************************************************************************************************************************************

TASK [Gathering Facts] *****************************************************************************************************************************************
ok: [dev_host]

TASK [ensure_tag : Fail if default_skip_tags is not defined] ***************************************************************************************************
skipping: [dev_host] => changed=false 
  skip_reason: Conditional result was False

TASK [ensure_tag : Fail if no skip tags is specified] **********************************************************************************************************
skipping: [dev_host] => changed=false 
  skip_reason: Conditional result was False

TASK [ensure_tag : Fail if skip tags is not correct] ***********************************************************************************************************
skipping: [dev_host] => changed=false 
  skip_reason: Conditional result was False

PLAY RECAP *****************************************************************************************************************************************************
dev_host                    : ok=1    changed=0    unreachable=0    failed=0    skipped=3    rescued=0    ignored=0   
```

Hope it will help someone in need! Cheers !