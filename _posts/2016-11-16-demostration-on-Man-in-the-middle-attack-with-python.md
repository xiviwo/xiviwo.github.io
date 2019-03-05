---
layout: post
title:  "Demostration on Man-in-the-middle attack with python"
date:   2016-11-16 08:33:50 +0800
categories: security 
---

#### Interesting Fact 

There is a application in my network always trying to upload some data to somewhere, which I want to figure out what it's doing.

It's a java? or C++? desktop app, which I would not care. What I do care how it is doing with my data. 

#### Redirect to localhost
I found out it's server url in the configuration file, let's call it `http://data.example.com`, it looks like it use `http` protocal, sound good. So let's redirect its url to `localhost` by adding  `127.0.0.1	data.example.com` to `/etc/hosts` host files.

#### Set up simple mock server

Then, we should set up a minimal dummy http server with basic `Get/Post` ability in whatever language you like, python demo as below:

```python
from http.server import HTTPServer, BaseHTTPRequestHandler

from io import BytesIO

class SimpleHTTPRequestHandler(BaseHTTPRequestHandler):

    def do_GET(self):
        self.send_response(200)
        self.end_headers()
        self.wfile.write(b'Hello, world!')

    def do_POST(self):
        content_length = int(self.headers['Content-Length'])
        body = self.rfile.read(content_length)
        self.send_response(200)
        self.end_headers()
        response = BytesIO()
        response.write(b'This is POST request. ')
        response.write(b'Received: ')
        response.write(body)
        self.wfile.write(response.getvalue())


httpd = HTTPServer(('localhost', 80), SimpleHTTPRequestHandler)
httpd.serve_forever()
```

#### Get Actual Request Info

When running the dummy http server and trigger data upload with this application, we get what it's actually trying to do.

```console
POST /tools/dataupload HTTP/1.1
Host: data.example.com
Accept: */*
Content-Type: multipart/form-data; boundary:@@##$$thisisboundary##$$@@
Connection: close
Content-Length: 4447
Expect: 100-continue

--@@##$$thisisboundary##$$@@
Content-Disposition: form-data; name:"uuid_SL"; filename:"uuid_SL"
Content-Type: application/octet-stream
Content-Transfer-Encoding: binary

"uuid",70002,"4.08-20.x86_64"

--@@##$$thisisboundary##$$@@
Content-Disposition: form-data; name:"uuid_WS"; filename:"uuid_WS"
Content-Type: application/octet-stream
Content-Transfer-Encoding: binary

"uuid", 0, 0, 0, 0, 0, 0, 0, 0, , , , , " 2.3.1.0", , ,

--@@##$$thisisboundary##$$@@
Content-Disposition: form-data; name:"uuid_SW"; filename:"uuid_SW"
Content-Type: application/octet-stream
Content-Transfer-Encoding: binary

"uuid",70004,"33.12.10-1.i586"
.......

--@@##$$thisisboundary##$$@@
Content-Disposition: form-data; name:"uuid_HW"; filename:"uuid_HW"
Content-Type: application/octet-stream
Content-Transfer-Encoding: binary

"Intel(R) Corporation Intel(R) Core(TM) iX-233232M CPU @ 10GHz", 3223323, 3223, 3232, "Red Hat Enterprise Linux Workstation release 7.4"

--@@##$$thisisboundary##$$@@--
``` 

So, at first glance, it's trying to gather software/hardware information and upload as a few attachments seperated by boundary called `@@##$$thisisboundary##$$@@`.

Let's try if I can simulate its action and accepted by the server.


#### requests-toolbelt

Regarding to data uploading, let's try `MultipartEncoder` from `requests-toolbelt`.

Probablly need to:
```console
pip install requests-toolbelt
```

#### Field of MultipartEncoder
Firstly, define the files to be upload with `field` , each field takes the form like `'fieldname': ('filename to be attached', open('file.py', 'rb'), 'Content-Type like text/plain')`, so we got.

```python
fields=[('uuid_SL', ('uuid_SL', open('uuid_SL', 'rb'), 'application/octet-stream',{'Content-Transfer-Encoding': 'binary'}))]
```
Why we use `'application/octet-stream',{'Content-Transfer-Encoding': 'binary'}` ? 
Because we are seeing such http headers in the request the application sent.

```text
Content-Type: application/octet-stream
Content-Transfer-Encoding: binary
```

#### MultipartEncoder

Important to note, `boundary='@@##$$thisisboundary##$$@@'` should be added to tell `MultipartEncoder` to separate these attachment with boundary
```python
m = MultipartEncoder(fields, boundary='@@##$$thisisboundary##$$@@')
```

#### Headers of http request

Based on the request the app sent, similar header should be added, like
Headers from request:

```http
Accept: */*
Content-Type: multipart/form-data; boundary:@@##$$thisisboundary##$$@@
Connection: close
Expect: 100-continue
```

Corresponding codes:
```python
prepped.headers['Content-Type'] = 'multipart/form-data; boundary=@@##$$thisisboundary##$$@@'
prepped.headers['Accept'] = '*/*'
prepped.headers['Connection'] = 'close'
prepped.headers['Expect'] = '100-continue'
```

#### Execute and Compare

Run the script against dummy server and compare the one requested from real application and make sure they match mostly


#### Run against Real Server

If everything looks good, remove the entry `127.0.0.1	data.example.com` from `/etc/hosts` and run the script again. It looks working ! Server returns correct response as real application ! Cheers !

```http
HTTP/1.1 200 OK
Date: Thu, 16 May 2016 08:17:32 GMT
Content-Type: text/plain;charset:ISO-8859-1
Content-Language: en-US
Connection: close
Set-Cookie: ..........
Content-Length: 182

[results]
rc1:0
filename1:uuid_SL
rc2:0
filename2:uuid_WS
rc3:0
filename3:uuid_SW
rc4:0
filename4:uuid_HW
```

#### Conclusion
With this script, I can easily go without the application and upload whatever data I wish, no matter how unreal it is, to the server.

So, it's simple demostration on how to manipulate data over insecure `http` protocal, we had better utilize `https` in all circumstance possible.


#### The script

```python

from requests import Request, Session

from requests_toolbelt import MultipartEncoder
url = "http://data.example.com/tools/dataupload"

fields=[('uuid_SL', ('uuid_SL', open('uuid_SL', 'rb'), 'application/octet-stream',{'Content-Transfer-Encoding': 'binary'})),
 		('uuid_WS', ('uuid_WS', open('uuid_WS', 'rb'), 'application/octet-stream',{'Content-Transfer-Encoding': 'binary'})),
 		('uuid_SW', ('uuid_SW', open('uuid_SW', 'rb'), 'application/octet-stream',{'Content-Transfer-Encoding': 'binary'})),
 		('uuid_HW', ('uuid_HW', open('uuid_HW', 'rb'), 'application/octet-stream',{'Content-Transfer-Encoding': 'binary'}))
 ]

m = MultipartEncoder(fields, boundary='@@##$$thisisboundary##$$@@')


s = Session()
req = Request('POST', url, data=m.to_string())
prepped = req.prepare()

prepped.headers['Content-Type'] = 'multipart/form-data; boundary=@@##$$thisisboundary##$$@@'
prepped.headers['Accept'] = '*/*'
prepped.headers['Connection'] = 'close'
prepped.headers['Expect'] = '100-continue'
print prepped.headers

r = s.send(prepped)

print r.text

```
