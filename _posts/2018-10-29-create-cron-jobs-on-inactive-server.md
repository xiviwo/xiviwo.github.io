---
layout: post
title:  "Create Cron Jobs on Inactive Server Dynamically"
date:   2018-10-29 15:24:50 +0800
categories: Shell
---

So, that comes a need to create cron job dynamically on inactive server in order to save load.
And, our active/inactive pair are swaping from node to node due to failover, so I hope to come up a automatic way to create cron job dymacially.

# How to decide which is active/inactive server
the basic idea is to resolve the two servers' ip address against the share domain name or service domain, say, `gitlab.example.com`, if it matches, that's active server

# how to resolve ip for server and share domain name
we use `dig` here, print the first line by default
```shell
resolve_ip(){
  host=$(to_domain $1)
  dig +short $host | awk '{ print ; exit }'
}
```

# how to get the domain name of server
Server name are coming in form of host name, need to update as domain, it's how our domain name looks like in my org. And, it won't change anything, if it's already a domain name

```shell
domain='gitlab.exmaple.com'

to_domain(){
  case "$1" in
  *example.com) echo "$1" ;;
  *)        echo "$1.example.com" ;;
  esac
}
```

# ok, let's find the active or inactive server


```shell
find_active(){
  for host in $*
  do
    if [ $(resolve_ip $host) == $(resolve_ip $domain) ]; then
      echo "$host"
    fi
  done
}

find_inactive(){
  for host in $*
  do
    if [ $(resolve_ip $host) != $(resolve_ip $domain) ]; then
      echo "$host"
    fi
  done
}

active=$(find_active server1 server2)
inactive=$(find_inactive server1 server2)

```

# Create cron jobs on inactive server

so, we need some remote ssh execution here, remember to use `BatchMode=yes` and `-i id_rsa` to avoid password prompt and passwordless login.

```shell
if [ ! -z "$inactive" ]; then

  ssh -o BatchMode=yes -i id_rsa -t root@$inactive <<'EOF'
  red=$'\e[1;31m'
  grn=$'\e[1;32m'
  end=$'\e[0m'
  printf "${grn}------------------$(hostname)-------------------${end}\n"
  cp /var/spool/cron/root /var/spool/cron/root.bak
  if ! grep '# Test Entry' /var/spool/cron/root >/dev/null 2>&1; then
    
    cat >> /var/spool/cron/root <<'EOU'

# Test Entry
10 6 * * * /bin/echo 'this is a test'

EOU
    printf "${red}%s${end}\n" "Crontab entry added on $(hostname)"
  else
    printf "${grn}%s${end}\n" "Crontab entry not changed on $(hostname)"
  fi
  diff /var/spool/cron/root /var/spool/cron/root.bak
EOF
```

# Delete cron jobs on active server
Similar stuff here

```shell

if [ ! -z "$active" ]; then

  ssh -o BatchMode=yes -i id_rsa -t root@$active <<'EOF'
  red=$'\e[1;31m'
  grn=$'\e[1;32m'
  end=$'\e[0m'
  printf "${grn}------------------$(hostname)-------------------${end}\n"

  sed -i.bak '/# Test Entry/d; /this is a test/d' /var/spool/cron/root
  if diff -q /var/spool/cron/root /var/spool/cron/root.bak >/dev/null; then
    printf "${grn}%s${end}\n" "Crontab entry not touched on $(hostname)"
  else
    printf "${red}%s${end}\n" "Crontab entry removed on $(hostname)"
  fi
  diff /var/spool/cron/root /var/spool/cron/root.bak
EOF

fi
```

So, we can create another cron job with this script, once the inactive/active roles are switched, your `# Test Entry` cron jobs should be created/deleted automatically on the corresponding servers. Cheers!