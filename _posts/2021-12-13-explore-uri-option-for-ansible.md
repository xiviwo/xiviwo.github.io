---
layout: post
title:  "Explore the uri library option for ansible"
date:   2021-12-14 13:03:50 +0800
categories: Ansible URI Module
---

As we know, to interact with webservices, there are multiple ways to do in Ansible realm. 

## Shell Module 
The most obvious and straight-foward way to request http resources, some like：

```sh
curl -k -u 'admin:admin' https://my.example.com/path/to/myresource.html 
```
Seem to be simple enough, but definitely, it's not elegant way I wish. 
Things could go to very messy soon enough if the url is getting longer or there are many variable in the curl command. You will find yourself at lost while reading the long string. 

One big problem of doing this way: it may dramatically increase the risk of exposing your password during the execution. 

## URI Module 

URI module is the official recommended module for almost every http request. 
And, it has been evolved sophisticatedly enough to handle most of the use case: details can be found on [here](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/uri_module.html)

One example from the offical

```yaml
- name: Login to a form based webpage using a list of tuples
  uri:
    url: https://your.form.based.auth.example.com/index.php
    method: POST
    body_format: form-urlencoded
    body:
    - [ name, your_username ]
    - [ password, your_password ]
    - [ enter, Sign in ]
    status_code: 302
  register: login
```

## GET_URI Module 
Get_uri module is similar to uri module, but its main purpose is to download file from http/https/ftp, etc. 

Example: 

```yaml
- name: Download file with custom HTTP headers
  get_url:
    url: http://example.com/path/file.conf
    dest: /etc/foo.conf
    headers:
      key1: one
      key2: two
```

## Custom Module 

It would be more interesting to take a look at the custom module approach.
As we know, ansible module is actually a executable file written in all kinds of program languages, python, shell, or even GO. 
But, most of the built-in modules come in with python, and ansible are basically python based. 
So, we would like to discuss python code module. 

When it comes to python code, there is no doubt the famouse `requests` library will come in the first place

### Python Requests Library 
[requests](https://docs.python-requests.org/en/latest/) library is one of the best python library I ever worked with, with its simple elegant syntax, powerful feature. 
I think that is why it gain such wide-spread popularity in python's world. 

The only one problem with `requests` is that it's needed to be installed on that specific host before it can be refered to by ansible module running on that host, or you will be seeing ugly and familiar message, `No module named 'requests'`. 

```sh
# Need to install on host in order for ansible module to import 
pip install requests 
```

It sound like quite simple to pip install this library, but for some rare case, with some highly secure dmz zone, which have no access to any external or internal pip source, although it is not totally impossible, but definitely a great headache to set it up.


### Ansible Built-in fetch_url Methods

Another possible option is to use ansible built-in methods, `fetch_url`. 

One of great thing about `fetch_url` is that it is available everywhere as long as ansible is installed, no need to install anything on target host except python. 

But, I believe there is no Linux host coming without python at all. ^^. 

However, it's not well documented too much on the internet. 

The best way to understand it is to download the source code from [here](https://github.com/ansible/ansible/blob/devel/lib/ansible/module_utils/urls.py)

From `lib/ansible/module_utils/urls.py`, we can see some simple document on how to use that. 
Basically, it's quite similar to its counterpart `requests` lib. 

```py

def fetch_url(module, url, data=None, headers=None, method=None,
              use_proxy=True, force=False, last_mod_time=None, timeout=10,
              use_gssapi=False, unix_socket=None, ca_path=None, cookies=None, unredirected_headers=None):
    """Sends a request via HTTP(S) or FTP (needs the module as parameter)

    :arg module: The AnsibleModule (used to get username, password etc. (s.b.).
    :arg url:             The url to use.

    :kwarg data:          The data to be sent (in case of POST/PUT).
    :kwarg headers:       A dict with the request headers.
    :kwarg method:        "POST", "PUT", etc.
    :kwarg boolean use_proxy:     Default: True
    :kwarg boolean force: If True: Do not get a cached copy (Default: False)
    :kwarg last_mod_time: Default: None
    :kwarg int timeout:   Default: 10
    :kwarg boolean use_gssapi:   Default: False
    :kwarg unix_socket: (optional) String of file system path to unix socket file to use when establishing
        connection to the provided url
    :kwarg ca_path: (optional) String of file system path to CA cert bundle to use
    :kwarg cookies: (optional) CookieJar object to send with the request
    :kwarg unredirected_headers: (optional) A list of headers to not attach on a redirected request

    :returns: A tuple of (**response**, **info**). Use ``response.read()`` to read the data.
        The **info** contains the 'status' and other meta data. When a HttpError (status >= 400)
        occurred then ``info['body']`` contains the error response data::

    Example::

        data={...}
        resp, info = fetch_url(module,
                               "http://example.com",
                               data=module.jsonify(data),
                               headers={'Content-type': 'application/json'},
                               method="POST")
        status_code = info["status"]
        body = resp.read()
        if status_code >= 400 :
            body = info['body']
    """
```

But, as matter of fact, this simple document doesn't serve much help to write working code. 
So, I will try to show some example below. 

#### How to import `fetch_url`

```py
from ansible.module_utils.urls import fetch_url
```

#### How to supply username/password to fetch_url

For example, for your custom module

```yaml
- name: My Demo Custom Module
  my_memo_module:
    username: admin
    password: admin
    state: present
    validate_certs: false
    basic_auth: true
```

password/username **MUST** be passed on to `url_username`/`url_password` to module params. 
And, then `fetch_url` will get this info from module object by passing in module as parameter.

```py
username = module.params['username']
password = module.params['password']
module.params['url_username'] = username
module.params['url_password'] = password
```

And, if you hope to enforce basic auth, pass on that as well 

```py
force_basic_auth = module.params['basic_auth']
module.params['force_basic_auth'] = force_basic_auth
```
Same case with `validate_certs`/`follow_redirects`/`http_agent`. 


#### How to do simple GET with `fetch_url`

```py
(resp, info) = fetch_url(
            module=module,
            url=url,
            headers=headers,
            method="GET")

if info['status'] == 200:
    module.exit_json(changed=True)
else:
    module.fail_json(msg="unable to call url: %s"% url)
```

#### How to do simple POST with `fetch_url`

```py
data = {}
(resp, info) = fetch_url(
            module=module,
            url=url,
            data=urllib.parse.urlencode(data),
            headers=headers,
            method="POST")

if info['status'] == 200:
    module.exit_json(changed=True)
else:
    module.fail_json(msg="unable to post url: %s"% url)
```

In my case, `urllib.parse.urlencode(data)` work better than `module.jsonify(data)`

#### How to POST binary file with mutipart protocal 

There is no much magic about `fetch_url`, except that there is no built-in support for uploading binary file along with `mutipart` protocal according to [this](https://github.com/ansible/ansible/issues/73621)

I do some deep research about that, finally be able to do that with some hack work. 

##### Create local module_utils dir 

Create `module_utils` to host shared lib file to `fetch_url`

```sh
cd <your playbook dir>
mkdir -pv library/module_utils
```

Add two lines to your `ansible.cfg` , `[default]` section

```ini
[defaults]
library = ./library 
module_utils = ./library/module_utils
```

##### Download requests toolbelt lib 

download `toolbelt` lib  with pip in your local machine 

```sh
pip3 install requests-toolbelt -t ./
cp -Rv requests_toolbelt/multipart/ <playbook_dir>/library/module_utils
```

Copy over necessary file， `fields.py` 

```sh
cp -Rv urllib3/fields.py <playbook_dir>/library/module_utils/multipart/
```

Now, it should look like this 

```sh
library/
├── module_utils
│   └── multipart
│       ├── decoder.py
│       ├── encoder.py
│       ├── fields.py
│       ├── __init__.py
```

##### Update `library/module_utils/multipart/encoder.py`

Update this part 
```py
import requests

from .._compat import fields

```
to 

```py
# import requests

from .fields import RequestField

```

```py
#self.session = session or requests.Session() 
#This line become 
self.session = session

#....
#field = fields.RequestField(name=k, data=file_pointer,
#This line become 
field = RequestField(name=k, data=file_pointer,
```
##### Import the `multipart` module utils you bake

Import the utils
```py
try:
  from ansible.module_utils.multipart import MultipartEncoder
except ImportError:
  from module_utils.multipart import MultipartEncoder
```

##### Use the same way as normal toolbelt lib 

```py
m = MultipartEncoder(
    fields={'field0': 'value', 'field1': 'value',
            'field2': ('filename', open('file.zip', 'rb'), 'application/octet-stream')}
    )
headers = {}
headers['Content-Type'] = m.content_type
(resp, info) = fetch_url(
            module=module,
            url=url,
            data=m,
            headers=headers,
            method="POST")

```

Cheers !

