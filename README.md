> [!IMPORTANT]
> Development on this package has moved to [Plain](https://plainframework.com?ref=django-oauth-login) and will continue there.
> Existing releases will remain published and I encourage you to fork this repo if you need to make changes.

# django-oauth-login

**Add OAuth login support to your Django project.**

[Watch on YouTube (3 mins) →](https://youtu.be/UxbxBa6AFsU)

This library is intentionally minimal.
It has no dependencies and a single database model.
If you simply want users to log in with GitHub, Google, Twitter, etc. (and maybe use that access token for API calls),
then this is the library for you.

There are three OAuth flows that it makes possible:

1. Signup via OAuth (new user, new OAuth connection)
2. Login via OAuth (existing user, existing OAuth connection)
3. Connect/disconnect OAuth accounts to a user (existing user, new OAuth connection)


## Usage

Install the package from PyPi:

```sh
pip install django-oauth-login
```

Add `oauthlogin` to your `INSTALLED_APPS` in `settings.py`:

```python
INSTALLED_APPS = [
    ...
    "oauthlogin",
]
```

In your `urls.py`, include `oauthlogin.urls`:

```python
urlpatterns = [
    path("oauth/", include("oauthlogin.urls")),
    ...
]
```

Then run migrations:

```sh
python manage.py migrate oauthlogin
```

Create a new OAuth provider ([or copy one from our examples](provider_examples)):

```python
# yourapp/oauth.py
import requests

from oauthlogin.providers import OAuthProvider, OAuthToken, OAuthUser


class ExampleOAuthProvider(OAuthProvider):
    authorization_url = "https://example.com/login/oauth/authorize"

    def get_oauth_token(self, *, code, request):
        response = requests.post(
            "https://example.com/login/oauth/token",
            headers={
                "Accept": "application/json",
            },
            data={
                "client_id": self.get_client_id(),
                "client_secret": self.get_client_secret(),
                "code": code,
            },
        )
        response.raise_for_status()
        data = response.json()
        return OAuthToken(
            access_token=data["access_token"],
        )

    def get_oauth_user(self, *, oauth_token):
        response = requests.get(
            "https://example.com/api/user",
            headers={
                "Accept": "application/json",
                "Authorization": f"token {oauth_token.access_token}",
            },
        )
        response.raise_for_status()
        data = response.json()
        return OAuthUser(
            id=data["id"],
            username=data["username"],
            email=data["email"],
        )
```

Create your OAuth app/consumer on the provider's site (GitHub, Google, etc.).
When setting it up, you'll likely need to give it a callback URL.
In development this can be `http://localhost:8000/oauth/github/callback/` (if you name it `"github"` like in the example below).
At the end you should get some sort of "client id" and "client secret" which you can then use in your `settings.py`:

```python
OAUTH_LOGIN_PROVIDERS = {
    "github": {
        "class": "yourapp.oauth.GitHubOAuthProvider",
        "kwargs": {
            "client_id": environ["GITHUB_CLIENT_ID"],
            "client_secret": environ["GITHUB_CLIENT_SECRET"],
            # "scope" is optional, defaults to ""

            # You can add other fields if you have additional kwargs in your class __init__
            # def __init__(self, *args, custom_arg="default", **kwargs):
            #     self.custom_arg = custom_arg
            #     super().__init__(*args, **kwargs)
        },
    },
}
```

Then add a login button (which is a form using POST rather than a basic link, for security purposes):

```html
<h1>Login</h1>
<form action="{% url 'oauthlogin:login' 'github' %}" method="post">
    {% csrf_token %}
    <button type="submit">Login with GitHub</button>
</form>
```

That's pretty much it!

## Advanced usage

### Email addresses should be unique

When you're integrating with an OAuth provider,
we think that the user's email address is the best "primary key" when linking to your `User` model in your app.
Unfortunately in Django, by default an email address is not required to be unique!
**We strongly recommend you require email addresses to be unique in your app.**

[As suggested by the Django docs](https://docs.djangoproject.com/en/4.0/topics/auth/customizing/#using-a-custom-user-model-when-starting-a-project),
one way to do this is to have your own `User` model:

```python
# In an app named "users", for example
from django.contrib.auth.models import AbstractUser

class User(AbstractUser):
    email = models.EmailField(unique=True)


# In settings.py
AUTH_USER_MODEL = 'users.User'
```

You'll also notice that there are no "email confirmation" or "email verification" flows in this library.
This is also intentional.
You can implement something like that yourself if you need to,
but the easier solution in our opinion is to use an OAuth provider you *trust to have done that already*.
If you look at our [provider examples](provider_examples) you'll notice how we often use provider APIs to get the email address which is "primary" and "verified" already.
If they've already done that work,
then we can just use that information.

### Handling OAuth errors

The most common error you'll run into is if an existing user clicks a login button,
but they haven't yet connected that provider to their account.
For security reasons,
the required flow here is that the user actually logs in with another method (however they signed up) and then *connects* the OAuth provider from a settings page.

For this error (and a couple others),
there is an error template that is rendered.
You can customize this by copying `oauthlogin/error.html` to one of your own template directories:

```html
{% extends "base.html" %}

{% block content %}
<h1>OAuth Error</h1>
<p>{{ oauth_error }}</p>
{% endblock %}
```

![Django OAuth duplicate email address error](https://user-images.githubusercontent.com/649496/159065848-b4ee6e63-9aa0-47b5-94e8-7bee9b509e60.png)

### Connecting and disconnecting OAuth accounts

To connect and disconnect OAuth accounts,
you can add a series of forms to a user/profile settings page.
Here's an very basic example:

```html
{% extends "base.html" %}

{% block content %}
Hello {{ request.user }}!

<h2>Existing connections</h2>
<ul>
    {% for connection in request.user.oauth_connections.all %}
    <li>
        {{ connection.provider_key }} [ID: {{ connection.provider_user_id }}]
        {% if connection.can_be_disconnected %}
        <form action="{% url 'oauthlogin:disconnect' connection.provider_key %}" method="post">
            {% csrf_token %}
            <input type="hidden" name="provider_user_id" value="{{ connection.provider_user_id }}">
            <button type="submit">Disconnect</button>
        </form>
        {% endif %}
    </li>
    {% endfor %}
</ul>

<h2>Add a connection</h2>
<ul>
    {% for provider_key in oauth_provider_keys %}
    <li>
        {{ provider_key}}
        <form action="{% url 'oauthlogin:connect' provider_key %}" method="post">
            {% csrf_token %}
            <button type="submit">Connect</button>
        </form>
    </li>
    {% endfor %}
</ul>

{% endblock %}
```

![Connecting and disconnecting Django OAuth accounts](https://user-images.githubusercontent.com/649496/159065096-30239a1f-62f6-4ee2-a944-45140f45af6f.png)

### Using a saved access token

```python
import requests

# Get the OAuth connection for a user
connection = user.oauth_connections.get(provider_key="github")

# If the token can expire, check and refresh it
if connection.access_token_expired():
    connection.refresh_access_token()

# Use the token in an API call
token = connection.oauth_token
response = requests.get(...)
```

### Using the Django system check

This library comes with a Django system check to ensure you don't *remove* a provider from `settings.py` that is still in use in your database.
You do need to specify the `--database` for this to run when using the check command by itself:

```sh
python manage.py check --database default
```

## FAQs

### How is this different from [other Django OAuth libraries](https://djangopackages.org/grids/g/oauth/)?

The short answer is that *it does less*.

In [django-allauth](https://github.com/pennersr/django-allauth)
(maybe the most popular alternative)
you get all kinds of other features like managing multiple email addresses,
email verification,
a long list of supported providers,
and a whole suite of forms/urls/views/templates/signals/tags.
And in my experience,
it's too much.
It often adds more complexity to your app than you actually need (or want) and honestly it can just be a lot to wrap your head around.
Personally, I don't like the way that your OAuth settings are stored in the database vs when you use `settings.py`,
and the implications for doing it one way or another.

The other popular OAuth libraries have similar issues,
and I think their *weight* outweighs their usefulness for 80% of the use cases.

### Why aren't providers included in the library itself?

One thing you'll notice is that we don't have a long list of pre-configured providers in this library.
Instead, we have some examples (which you can usually just copy, paste, and use) and otherwise encourage you to wire up the provider yourself.
Often times all this means is finding the two OAuth URLs ("oauth/authorize" and "oauth/token") in their docs,
and writing two class methods that do the actual work of getting the user's data (which is often customized anyway).

We've written examples for the following providers:

- [GitHub](provider_examples/github.py)
- [GitLab](provider_examples/gitlab.py)
- [Bitbucket](provider_examples/bitbucket.py)

Just copy that code and paste it in your project.
Tweak as necessary!

This might sound strange at first.
But in the long run we think it's actually *much* more maintainable for both us (as library authors) and you (as app author).
If something breaks with a provider, you can fix it immediately!
You don't need to try to run changes through us or wait for an upstream update.
You're welcome to contribute an example to this repo,
and there won't be an expectation that it "works perfectly for every use case until the end of time".
