Testing weppy application
=========================
*New in version 0.6*

> Untested code is broken code

The origin of this quote is quite uncharted and even if it's not entirely correct, it's also not so far from the truth. Untested applications make it hard to improve existing code and developers of untested applications tend to become pretty paranoid. If an application has automated tests, on the contrary, you can make changes and instantly know if anything breaks.

weppy provides an integrated test client that lets you send requests to your application and test your routes with your favorite testing suite. In this documentation we will use the *pytest* package, but you can obviously use unittest or whatever package you prefer.


Your first test
---------------

In order to learn how to use the test client, let's get back to the *bloggy* example we seen in the [tutorial chapter](./tutorial) and write down a test to verify that the home page shows "No posts here so far" when we access the index of our application. We can use a *fixture* of the pytest package to share the client instance between the tests, and write down something like this in a *tests.py* file:

```python
import pytest
from bloggy import app

@pytest.fixture()
def client():
    return app.test_client()

def test_empty_db(client):
    rv = client.get('/')
    assert 'No posts here so far' in rv.data
```

Notice that our test function begin with the word test: this allows pytest to automatically identify the method as a test to run.

By using `client.get` we can send an HTTP GET request to the application with the given path. The return value will be a `ClientResponse` object. We can now use the `data` attribute to inspect the return value (as string) from the application. In this case, we ensure that 'No posts here so far' is part of the output.

If we now run the test suite, we should see the following output:

```
> py.test tests.py
================================== test session starts ==================================
platform darwin -- Python 2.7.8 -- py-1.4.26 -- pytest-2.6.4
collected 1 item

tests.py .

=============================== 1 passed in 0.28 seconds ================================
```

Test client methods and options
-------------------------------

The client object gives you all the methods that will make requests using the matching HTTP methods:

- get
- patch
- put
- post
- delete
- head
- options

Under the default behavior, the test client will store cookies between requests, so if you make two or more consequent requests with the same client instance, you will be capable of use cookies related features. If you wan't to avoid this behavior, you can initialize an instance of the client with the apposite parameter:

```python
client = app.test_client(use_cookies=False)
```

Also, all the requests won't follow redirects unless you specify it in the *specific* request method:

```python
client.get('/', follow_redirects=True)
```

Using the application context
-----------------------------

Quite often you will need to perform tests that depends on the client status, in particular you may need to access the client session or perform operations depending on the last request. One typical example is the one where you need the client to login before performing other operations.

You have several ways to access the weppy context objects when running tests. In fact, when you make a request with the test client, you can access the `request`, `response`, `session` and `T` objects referred to the client in the same way you use them in your application:

```python
from weppy import request, response, session, T
```

All these objects will be, indeed, the real objects of the weppy context, and will change their values every time you make a request with the client.

When you need to preserve those objects between multiple requests, you can use the `context` property of the `ClientResponse` object. This property will return an object with `request`, `response`, `session` and `T` attributes from the request context:

```python
r = client.get('/')
last_session = r.context.session
```

The test client and its return value support both the `with` notation. weppy won't do any action on entering and exiting the code block, but you may like to use this notation:

```python
with client.get('/').context as ctx:
    client.post('/', data={'someval': ctx.session.otherval})
```

### Login with the test client

When you're using the builtin [auth](./auth) module, you can login the client in order to have a logged session. The only caveat is that you have to inject the form token due to the CSRF protection. You can write a function like this:

```python
def logged_client():
    c = app.test_client()
    c.get('/account/login')
    c.post('/account/login', data={
        'email': 'walter@massivedynamics.com',
        'password': 'pocketuniverse',
        '_csrf_token': list(session._csrf)[-1]
    }, follow_redirects=True)
    return c
```

or using the with notation:

```python
def logged_client():
    c = app.test_client()
    with c.get('/account/login').context as ctx:
        c.post('/account/login', data={
            'email': 'walter@massivedynamics.com',
            'password': 'pocketuniverse',
            '_csrf_token': list(ctx.session._csrf)[-1]
        }, follow_redirects=True)
        return c
```

### Database connection

The default database handler in weppy will reconnect the database when the request starts and closes the connection when the request ends.

Due to this, if you need to run database operations, you should manually re-establish the connection of the adapter:

```python
db._adapter.reconnect()
```

and you also need to remember to commit or rollback changes:

```python
db.commit()
```
