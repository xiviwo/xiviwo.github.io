---
layout: post
title:  "Nested Heredoc Variable Expansion"
date:   2018-11-03 10:12:50 +0800
categories: shell
---

As well known, the format of here-documents is:

      <<[-]word
              here-document
      delimiter

>No parameter expansion, command substitution, arithmetic expansion, or pathname expansion is performed on word. If any characters in word are quoted, the delimiter is the result of quote removal on word, and the lines in the here-document are not expanded. If word is unquoted, all lines of the here-document are subjected to parameter expansion, command substitution, and arithmetic expansion. [...]

So, in a word, if you need variable expansion/command substitution, use `EOF`, otherwise `'EOF'`.

But, if you come across some crazy case like nested heredoc, you will have headache..

Say, we need variable expansion inside first `EOF`, so we should unquoted it.
And, we don't need variable expansion inside second `'EOT'`, so we need quote, well, look messy.

But, it won't work like below, as variable expansion/command substitution is happening inside `EOT` block on current machine instead of remote.

Say, `VBoxManage list runningvms|egrep -o '{.+}'|sed "s/[{}]//g"` will run on current machine, and list vm on your current machine, or throw `'VBoxManage' command not found` error if no Virtualbox installed on current machine.

```shell

ssh -o BatchMode=yes  -t root@$h << EOF
  echo do serious something on $h

  su - jenkins  <<'EOT'
    uuids=`VBoxManage list runningvms|egrep -o '{.+}'|sed "s/[{}]//g"`
    for uuid in ${uuids}; do
      VBoxManage controlvm $uuid poweroff 
      VBoxManage unregistervm $uuid --delete
    done

EOT
EOF
```

So, it look very messy, you need variable expansion in part of the heredoc, and not in other part.

Well, there is one options, use `\` to escape where variable expansion.

The following would work, hope someone need this.

```shell
ssh -o BatchMode=yes -t root@$h << EOF
  echo do serious something on $h

  su - jenkins  <<'EOT'
    uuids="\$(VBoxManage list runningvms|egrep -o '{.+}'|sed 's/[{}]//g')"
    for uuid in \${uuids}; do
      VBoxManage controlvm \$uuid poweroff 
      VBoxManage unregistervm \$uuid --delete
    done
EOT
EOF
```