# Django GRIP

Author: Justin Karneges <justin@fanout.io>

This library helps your Django application delegate long-lived HTTP/WebSocket connection management to a GRIP-compatible proxy such as [Fanout Cloud](https://fanout.io/) or [Pushpin](https://pushpin.org/).

## Setup

First, install this module:

```sh
pip install django-grip
```

In your `settings.py`, add the `GripMiddleware`:

```python
MIDDLEWARE = [
    'django_grip.GripMiddleware',
    ...
]
```

The middleware handles parsing/generating GRIP headers and WebSocket-Over-HTTP events. It should be placed early in the stack.

Additionally, set `GRIP_URL` with your proxy settings, e.g.:

```python
# fanout cloud
GRIP_URL = 'http://api.fanout.io/realm/your-realm?iss=your-realm&key=base64:your-realm-key'
```

```python
# pushpin
GRIP_URL = 'http://localhost:5561'
```

## Usage

### HTTP streaming

```python
from django.http import HttpResponse, HttpResponseNotAllowed
from gripcontrol import HttpStreamFormat
from django_grip import set_hold_stream, publish

def myendpoint(request):
    if request.method == 'GET':
        # if the request didn't come through a GRIP proxy, throw 501
        if not request.grip_proxied:
            return HttpResponse('Not Implemented\n', status=501)

        # subscribe every incoming request to a channel in stream mode
        resp = HttpResponse('[stream open]\n', content_type='text/plain')
        set_hold_stream(request, 'test')
        return resp
    elif request.method == 'POST':
        # publish data to subscribers
        data = request.POST['data']
        publish('test', HttpStreamFormat(data + '\n'))
        return HttpResponse('Ok\n')
    else:
        return HttpResponseNotAllowed(['GET', 'POST'])
```

### HTTP long-polling

```python
from django.http import HttpResponse, HttpResponseNotModified, HttpResponseNotAllowed
from gripcontrol import HttpResponseFormat
from django_grip import set_hold_response, publish

def myendpoint(request):
    if request.method == 'GET':
        # get object and its etag
        obj = ...
        etag = ...

        # if object is unchanged, long-poll, else return object
        inm = request.META.get('HTTP_IF_NONE_MATCH')
        if inm == etag:
            # subscribe request to channel, return status 304 after timeout
            resp = HttpResponseNotModified()
            set_hold_longpoll(request, 'test')
        else:
            resp = HttpResponse(obj.serialize())

        resp['ETag'] = etag
        return resp
    elif request.method == 'POST':
        data = request.POST['data']

        # update object based on request, and get resulting object and its etag
        obj = ...
        etag = ...

        # publish data to subscribers
        headers = {'ETag': etag}
        publish('test', HttpResponseFormat(obj.serialize(), headers=headers))
        return HttpResponse('Ok\n')
    else:
        return HttpResponseNotAllowed(['GET', 'POST'])
```

### WebSockets

Here's an echo service with a broadcast endpoint:

```python
from django.http import HttpResponse, HttpResponseNotAllowed
from gripcontrol import WebSocketMessageFormat
from django_grip import websocket_only, publish

# decorator means reject non-websocket-related requests. it also means we
#   don't need to return an HttpResponse object. the middleware will take care
#   of that for us.
@websocket_only
def echo(request):
    # since we used the decorator, this will always be a non-None value
    ws = request.wscontext

    # if this is a new connection, accept it and subscribe it to a channel
    if ws.is_opening():
        ws.accept()
        ws.subscribe('test')

    # here we loop over any messages
    while ws.can_recv():
        message = ws.recv()

        # if return value is None, then the connection is closed
        if message is None:
            ws.close()
            break

        # echo the message
        ws.send(message)

def broadcast(request):
    if request.method == 'POST':
        # publish data to all clients that are connected to the echo endpoint
        data = request.POST['data']
        publish('test', WebSocketMessageFormat(data))
        return HttpResponse('Ok\n')
    else:
        return HttpResponseNotAllowed(['POST'])
```

## Advanced settings

If you need to communicate with more than one GRIP proxy (e.g. multiple Pushpin instances, or Fanout Cloud + Pushpin), you can use `GRIP_PROXIES` instead of `GRIP_URL`. For example:

```python
GRIP_PROXIES = [
    # pushpin
    {
        'control_uri': 'http://localhost:5561'
    },
    # fanout cloud
    {
        'key': b64decode('your-realm-key'),
        'control_uri': 'http://api.fanout.io/realm/your-realm',
        'control_iss': 'your-realm'
    }
]
```

If it's possible for clients to access the Django app directly, without necessarily going through a GRIP proxy, then you may want to avoid sending GRIP instructions to those clients. An easy way to achieve this is with the `GRIP_PROXY_REQUIRED` setting. If set, then any direct requests that trigger a GRIP instruction response will be given a 501 Not Implemented error instead.

```python
GRIP_PROXY_REQUIRED = True
```

To prepend a fixed string to all channels used for publishing and subscribing, set `GRIP_PREFIX` in your configuration:

```python
GRIP_PREFIX = 'myapp-'
```
