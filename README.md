# DwollaV2

![Build Status](https://travis-ci.org/Dwolla/dwolla-v2-python.svg)

Dwolla V2 Python client. For the V1 Python client see [Dwolla/dwolla-python](https://github.com/Dwolla/dwolla-python).

[API Documentation](https://docsv2.dwolla.com)

## Installation

```
pip install dwollav2
```

## `dwollav2.Client`

### Basic usage

Create a client using your application's consumer key and secret found on the applications page
([UAT][apuat], [Production][approd]).

[apuat]: https://uat.dwolla.com/applications
[approd]: https://www.dwolla.com/applications

```python
client = dwollav2.Client(id = os.environ['DWOLLA_ID'], secret = os.environ['DWOLLA_SECRET'])
```

### Using the sandbox environment (optional)

```python
client = dwollav2.Client(
  id = os.environ['DWOLLA_ID'],
  secret = os.environ['DWOLLA_SECRET'],
  environment = 'sandbox'
)
```

`environment` defaults to `'production'`.

### Configure an `on_grant` callback (optional)

An `on_grant` callback is useful for storing new tokens when they are granted. The `on_grant`
callback is called with the `Token` that was just granted by the server.

```python
# config/initializers/dwolla.rb
client = dwollav2.Client(
  id = os.environ['DWOLLA_ID'],
  secret = os.environ['DWOLLA_SECRET'],
  on_grant = lambda t: save(t)
)
```

It is highly recommended that you encrypt any token data you store.

## `Token`

Tokens can be used to make requests to the Dwolla V2 API. There are two types of tokens:

### Application tokens

Application tokens are used to access the API on behalf of a consumer application. API resources that
belong to an application include: `webhook-subscriptions`, `events`, and `webhooks`. Application
tokens can be created using the [`client_credentials`][client_credentials] OAuth grant type:

[client_credentials]: https://tools.ietf.org/html/rfc6749#section-4.4

```python
application_token = client.Auth.client()
```

*Application tokens do not include a `refresh_token`. When an application token expires, generate
a new one using `client.Auth.client()`.*

### Account tokens

Account tokens are used to access the API on behalf of a Dwolla account. API resources that belong
to an account include `customers`, `funding-sources`, `documents`, `mass-payments`, `mass-payment-items`,
`transfers`, and `on-demand-authorizations`.

There are two ways to get an account token. One is by generating a token at
https://uat.dwolla.com/applications (sandbox) or https://www.dwolla.com/applications (production).

You can instantiate a generated token by doing the following:

```python
account_token = client.Token(access_token = '...', refresh_token = '...')
```

The other way to get an account token is using the [`authorization_code`][authorization_code]
OAuth grant type. This flow works by redirecting a user to dwolla.com in order to get authorization
and sending them back to your website with an authorization code which can be exchanged for a token.
For example:

[authorization_code]: https://tools.ietf.org/html/rfc6749#section-4.1

```python
# http://www.twobotechnologies.com/blog/2014/02/importance-of-state-in-oauth2.html
state = binascii.b2a_hex(os.urandom(15))
auth = client.Auth(redirect_uri = 'https://yoursite.com/callback',
                   scope = 'ManageCustomers|Funding',
                   state = state)

# redirect the user to dwolla.com for authorization
redirect_to(auth.url)

# exchange the code for a token
token = auth.callback({'code': '...', 'state': state})
```

### Refreshing tokens

Tokens with a `refresh_token` can be refreshed using `client.Auth.refresh`, which takes a
`Token` as its first argument and returns a new token.

```python
new_token = client.Auth.refresh(expired_token)
```

### Initializing tokens:

`Token`s can be initialized with the following attributes:

```python
client.Token(access_token = '...',
             refresh_token = '...',
             expires_in = 123,
             scope = '...',
             account_id = '...')
```

## Requests

`Token`s can make requests using the `#get`, `#post`, and `#delete` methods.

```python
# GET api.dwolla.com/resource?foo=bar
token.get('resource', foo = 'bar')

# POST api.dwolla.com/resource {"foo":"bar"}
token.post('resource', foo = 'bar')

# POST api.dwolla.com/resource multipart/form-data foo=...
token.post('resource', foo = ('mclovin.jpg', open('mclovin.jpg', 'rb'), 'image/jpeg'))

# PUT api.dwolla.com/resource {"foo":"bar"}
token.put('resource', foo = 'bar')

# DELETE api.dwolla.com/resource
token.delete('resource')
```

## Responses

Requests return a `Response`.

```python
res = token.get('/')

res.status
# => 200

res.headers
# => {'server'=>'cloudflare-nginx', 'date'=>'Mon, 28 Mar 2016 15:30:23 GMT', 'content-type'=>'application/vnd.dwolla.v1.hal+json; charset=UTF-8', 'content-length'=>'150', 'connection'=>'close', 'set-cookie'=>'__cfduid=d9dcd0f586c166d36cbd45b992bdaa11b1459179023; expires=Tue, 28-Mar-17 15:30:23 GMT; path=/; domain=.dwolla.com; HttpOnly', 'x-request-id'=>'69a4e612-5dae-4c52-a6a0-2f921e34a88a', 'cf-ray'=>'28ac1f81875941e3-MSP'}

res.body['_links']['events']['href']
# => 'https://api-uat.dwolla.com/events'
```

## Errors

If the server returns an error, a `dwollav2.Error` (or one of its subclasses) will be raised.
`dwollav2.Error`s are similar to `Response`s.

```python
try:
  token.get "/not-found"
except dwollav2.NotFoundError:
  e.status
  # => 404

  e.headers
  # => {"server"=>"cloudflare-nginx", "date"=>"Mon, 28 Mar 2016 15:35:32 GMT", "content-type"=>"application/vnd.dwolla.v1.hal+json; profile=\"http://nocarrier.co.uk/profiles/vnd.error/\"; charset=UTF-8", "content-length"=>"69", "connection"=>"close", "set-cookie"=>"__cfduid=da1478bfdf3e56275cd8a6a741866ccce1459179332; expires=Tue, 28-Mar-17 15:35:32 GMT; path=/; domain=.dwolla.com; HttpOnly", "access-control-allow-origin"=>"*", "x-request-id"=>"667fca74-b53d-43db-bddd-50426a011881", "cf-ray"=>"28ac270abca64207-MSP"}

  e.body.code
  # => "NotFound"
except dwollav2.Error:
  # ...
```

### `dwollav2.Error` subclasses:

*See https://docsv2.dwolla.com/#errors for more info.*

- `dwollav2.AccessDeniedError`
- `dwollav2.InvalidCredentialsError`
- `dwollav2.NotFoundError`
- `dwollav2.BadRequestError`
- `dwollav2.InvalidGrantError`
- `dwollav2.RequestTimeoutError`
- `dwollav2.ExpiredAccessTokenError`
- `dwollav2.InvalidRequestError`
- `dwollav2.ServerError`
- `dwollav2.ForbiddenError`
- `dwollav2.InvalidResourceStateError`
- `dwollav2.TemporarilyUnavailableError`
- `dwollav2.InvalidAccessTokenError`
- `dwollav2.InvalidScopeError`
- `dwollav2.UnauthorizedClientError`
- `dwollav2.InvalidAccountStatusError`
- `dwollav2.InvalidScopesError`
- `dwollav2.UnsupportedGrantTypeError`
- `dwollav2.InvalidApplicationStatusError`
- `dwollav2.InvalidVersionError`
- `dwollav2.UnsupportedResponseTypeError`
- `dwollav2.InvalidClientError`
- `dwollav2.MethodNotAllowedError`
- `dwollav2.ValidationError`

## Development

After checking out the repo, run `pip install -r requirements.txt` to install dependencies.
Then, run `python setup.py test` to run the tests.

To install this gem onto your local machine, run `pip install -e .`.

## Contributing

Bug reports and pull requests are welcome on GitHub at https://github.com/Dwolla/dwolla-v2-python.

## License

The package is available as open source under the terms of the [MIT License](https://github.com/Dwolla/dwolla-v2-python).
