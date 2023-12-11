# Django-openid Testproject for University of Helsinki Login Service
This project is an example of a Django OpenID Connect configuration for University of Helsinki login service. The app connects to a test service and does not deal with any personal information of students or personnel.
The project is available in github: https://github.com/ellaverak/django-openid

# The Basics

OpenID Connect is an identity layer built on top of the OAuth 2.0 framework. OpenID Connect is based on configuration data that is openly available and can be read by the OAuth-client. For example the configuration data for the University of Helsinki login (test) service can be found from https://login-test.it.helsinki.fi/.well-known/openid-configuration

The OAuth-client is set up using the configuration data. The Client communicates with the University of Helsinki sp-registry service responsible for distibuting user data. User data can be reached via an userinfo endpoint or it can be decoded from an id_token.

# Configuration

## [pyproject.toml](https://github.com/ellaverak/django-openid/blob/main/pyproject.toml)

```authlib = "^1.2.1"```
Configuration base for the OpenID Connect client (OAuth-client).

```django-environ = "^0.11.2"```
Add-on for environemnt variables.

```psycopg2 = "^2.9.9"```
This project uses a postgres database. The database connection is created with psycopg2.

```django-cryptography = "^1.1"```
Cryptography base for the postgres database.

## [settings.py](https://github.com/ellaverak/django-openid/blob/main/project/project/settings.py)

```
LOGOUT_REDIRECT_URL = 'https://login-test.it.helsinki.fi/idp/profile/Logout'
```

Defines the logout url as the University of Helsinki logout url. When the logout url is defined in the setting it doesn't need to have it's own function in views.py. However ```path("accounts/", include("django.contrib.auth.urls"))``` needs to be added to [urls.py](https://github.com/ellaverak/django-openid/blob/main/project/openid/urls.py)

```
AUTHENTICATION_BACKENDS = ['openid.authentication.LoginBackend']
```

Defines a custom backend for django authentication.

```
AUTHLIB_OAUTH_CLIENTS = {
    'helsinki': {
        'client_id': os.getenv('OIDC_CLIENT_ID'),
        'client_secret': os.getenv('OIDC_CLIENT_SECRET')
    }
}
```

Defines the name of the new OAuth-client (helsinki) and sets the client_id and client_secret parameters. The parameter values can be found from the University of Helsinki sp-registry.

## [views.py](https://github.com/ellaverak/django-openid/blob/main/project/openid/views.py)

```import urllib.request, json```
Keyset for decoding userdata from the id_token is in json format.

```from django.contrib.auth import authenticate as django_authenticate```

```from django.contrib.auth import login as django_login```
Django's inbuild functions are importent with custom names to avoid conflict with the view-function names.

```from authlib.integrations.django_client import OAuth```
OAuth-client.

```from authlib.oidc.core import CodeIDToken```
CodeIDToken includes the instructions for decoding the id_token.

```from authlib.jose import jwt```
Used for transferring claims between the OAuth-client and the corresponding service.

```
CONF_URL = 'https://login-test.it.helsinki.fi/.well-known/openid-configuration'
oauth = OAuth()
oauth.register(
    name='helsinki',
    server_metadata_url=CONF_URL,
    client_kwargs={
        'scope': 'openid profile'
    }
)
```

Registers the OAuth-client (helsinki) using the openid-configuration.

```
claims_data = {
        "id_token": {
            "hyPersonStudentId": None

        },
        "userinfo": {
            "email": None,
            "family_name": None,
            "given_name": None,
            "uid": None
        }
    }
```

Defines the claims that OAuth-client requests from the service.

```
with urllib.request.urlopen("https://login-test.it.helsinki.fi/idp/profile/oidc/keyset") as url:
    keys = json.load(url)
```

Gets the keyset for decoding the id_token. The url can be found from the openid configuration data: https://login-test.it.helsinki.fi/.well-known/openid-configuration

### login-function

```
redirect_uri = request.build_absolute_uri(reverse('auth'))
```

Builds the redirect url defined in the sp-registry. In this case the redirect url form is: [openshift-address]/auth/

```
return oauth.helsinki.authorize_redirect(request, redirect_uri, claims=claims)
```

Authorizes the redirect url and provides claims to the service.

### auth-function

```
token = oauth.helsinki.authorize_access_token(request)
```

Fetches the token from the service after a successfull login. The token is a dictionary that includes the access_token, id_token etc.

```
userinfo = oauth.helsinki.userinfo(token=token)
```

The token is used to access the userinfo-enpoint. The userinfo-enpoint returns the data defined in the userinfo claims. The data is returned as a dictionary.

```
userdata = jwt.decode(token['id_token'], keys, claims_cls=CodeIDToken)
userdata.validate()
```

Data from the id_token is decoded using the keys from the keyset and the instructions from the CodeIDToken. After that the data is validated and it is available as a dictionary.

```
user = django_authenticate(userinfo=userinfo, userdata=userdata)
    if user is not None:
        django_login(request, user)

    return redirect(home)
```

After a successful University of Helsinki login the user is logged in using a modified django-login. This ensures that the django framework works properly and the logged in user has access to django's default functionality.


## [authentication.py](https://github.com/ellaverak/django-openid/blob/main/project/openid/authentication.py)

Django's default authentication backend uses usernames for authentication. This custom authentication backend chances that to emails. The backend is called after a successfull university login.


## [models.py](https://github.com/ellaverak/django-openid/blob/main/project/openid/models.py)

```from django_cryptography.fields import encrypt```
Database encryption is implemented using django-cryptography.

```
class User(AbstractUser):
    student_id = encrypt((models.CharField(default="000000000")))
    first_name = encrypt((models.CharField(max_length = 100)))
    last_name = encrypt((models.CharField(max_length = 100)))
    email = (models.CharField(max_length = 100))
    username = encrypt((models.CharField(unique=True, max_length = 100)))
```

Defines the User-model corresponding to the user relation in the postgres database. Student_id, first-name, last_name and username are encrypted. Email is saved as plain text because django uses email for custom authentication and decrypting relation data is complicated during the authentication process.

