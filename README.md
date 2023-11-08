# django-mini-7-Throttling
منبع: https://www.django-rest-framework.org/api-guide/throttling/

in view: ([authentication](https://github.com/mr-fact/django-mini-5-authentication) -> [permission](https://github.com/mr-fact/django-mini-6-permission) -> **throttling** -> other codes)

## How throttling is determined
As with permissions and authentication, throttling in REST framework is always defined as a `list of classes`. For example, you might want to limit a user to a maximum of 60 requests per minute, and 1000 requests per day.

raise -> `exceptions.Throttled`

## Setting the throttling policy
set in `settings`
``` python
REST_FRAMEWORK = {
    'DEFAULT_THROTTLE_CLASSES': [
        'rest_framework.throttling.AnonRateThrottle',
        'rest_framework.throttling.UserRateThrottle'
    ],
    'DEFAULT_THROTTLE_RATES': {
        'anon': '100/day',
        'user': '1000/day'
    }
}
```
set in `APIView`
``` python
from rest_framework.response import Response
from rest_framework.throttling import UserRateThrottle
from rest_framework.views import APIView

class ExampleView(APIView):
    throttle_classes = [UserRateThrottle]

    def get(self, request, format=None):
        content = {
            'status': 'request was permitted'
        }
        return Response(content)
```
set in `@api_view`
``` python
@api_view(['GET'])
@throttle_classes([UserRateThrottle])
def example_view(request, format=None):
    content = {
        'status': 'request was permitted'
    }
    return Response(content)
```
set in `method of class`
``` python
@action(detail=True, methods=["post"], throttle_classes=[UserRateThrottle])
def example_adhoc_method(request, pk=None):
    content = {
        'status': 'request was permitted'
    }
    return Response(content)
```
The rate descriptions used in `DEFAULT_THROTTLE_RATES` may include `second`, `minute`, `hour` or `day` as the throttle period.

## How clients are identified
- If the `X-Forwarded-For` header is present then it will be used
- otherwise the value of the `REMOTE_ADDR` variable from the `WSGI` environment will be used.

If you need to strictly identify unique client IP addresses, you'll need to first configure the number of application proxies that the API runs behind by setting the `NUM_PROXIES` setting [0 -> n]
- If set to `non-zero` then the client IP will be identified as being the last IP address in the `X-Forwarded-For` header, once any application proxy IP addresses have first been excluded
- If set to `zero`, then the `REMOTE_ADDR` value will always be used as the identifying IP address

It is important to understand that if you configure the `NUM_PROXIES` setting, then all clients behind a unique [NAT'd](https://en.wikipedia.org/wiki/Network_address_translation) gateway will be treated as a single client.

## Setting up the cache
- The throttle classes provided by REST framework use Django's cache backend
- You should make sure that you've set appropriate [cache settings](https://docs.djangoproject.com/en/stable/ref/settings/#caches)
- The default value of `LocMemCache` backend should be okay for simple setups
- See Django's [cache documentation](https://docs.djangoproject.com/en/stable/topics/cache/#setting-up-the-cache) for more details

``` python
from django.core.cache import caches

class CustomAnonRateThrottle(AnonRateThrottle):
    cache = caches['alternate']
```

## A note on concurrency
The built-in throttle implementations are open to [race conditions](https://en.wikipedia.org/wiki/Race_condition#Data_race), so under high concurrency they may allow a few extra requests through.

If your project relies on guaranteeing the number of requests during concurrent requests, you will need to implement your own throttle class. See [issue #5181](https://github.com/encode/django-rest-framework/issues/5181) for more details.
