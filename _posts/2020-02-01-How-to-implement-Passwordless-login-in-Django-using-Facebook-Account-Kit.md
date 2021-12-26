---
layout: post
title: How to implement Passwordless login in Django using Facebook Account Kit
description: This post implements passwordless less in Django using Facebook’s Account Kit.
date: '2020-02-01T11:48:01.866Z'
tag:
- django
category: blog
author: taranjeet
---

![django-facebook-account-kit-cover-pic](/assets/img/django_facebook_account_kit_cover_pic.png "Django + Facebook Account Kit")

This post implements [Account Kit](https://www.accountkit.com/) using Django. It can be considered as a python translation of [Passwordless Login with Facebook Account Kit](https://auth0.com/blog/facebook-account-kit-passwordless-authentication/). The code for this blog post can be found [here](https://github.com/all-about-django/django-account-kit-demo).

#### Introduction

Passwordless login allows a user to log in to the application without having to enter and remember the password. It uses One Time Password called OTP which is delivered to the user via SMS or email.

Facebook provides a service called Account Kit which is used to implement passwordless authentication. Users only need to enter their phone number or email and Account Kit takes care of sending OTP and verifying them.

To implement Account Kit in an application, a developer should have [Facebook Developer Account](https://developers.facebook.com/) and a Facebook application with “_Account Kit_” enabled. Detailed instructions for how to do this can be found in [this blog](https://auth0.com/blog/facebook-account-kit-passwordless-authentication/) under section “_Integrating Passwordless Authentication with Facebook Account Kit_” and subsection “_Getting Started_”. After creating an application, remember to change the following values:

*   `Server Domains` should be `localhost:8000`
*   `Redirect URLs` should be `localhost:8000/sendcode`

![](/assets/img/1__OIzO2LZrC1iDuNbjV9tWTw.png)

**Note:** Here we have assumed that the application is running on port `8000`. If not, please change it accordingly.

Now let's write the code which will integrate with Account Kit and help a user login to the application.

This post assumes that you are familiar with the following Django concepts:

*   Starting a project
*   Creating an app using `startapp`
*   Using Django server command
*   Setting up and using templates

Assuming that you have set up a barebones Django application and `runserver` is running successfully.

```shell
python manage.py runserver
```

Now let’s create a new application called `logindemo` by running `startapp` command.

```shell
python manage.py startapp logindemo
```

Now, let’s add the newly created app `logindemo` to `INSTALLED_APPS` in `settings.py`

```python
# settings.py

INSTALLED_APPS = (
  # ...
  'logindemo',
)
```

Now let’s write our first view which will render the login page and pass some variables which are necessary to initialize account kit.

```python
# logindemo/views.py

from django.conf import settings
from django.shortcuts import render


def render_login_page(request):
    template_name = 'logindemo/index.html'
    context = {}
    context['app_id'] = settings.ACCOUNT_KIT_APP_ID
    context['api_version'] = settings.ACCOUNT_KIT_API_VERSION
    return render(request, template_name, context)
```

Here we have defined certain variables which are provided at the time of creating an application in Facebook Developer Console. We will be using the following variables, which should be defined in `settings.py` file.

```python
ACCOUNT_KIT_API_VERSION = 'v1.0'
ACCOUNT_KIT_ME_ENDPOINT_BASE_URL = 'https://graph.accountkit.com/v1.0/me'
ACCOUNT_KIT_TOKEN_EXCHANGE_BASE_URL = 'https://graph.accountkit.com/v1.0/access_token'
ACCOUNT_KIT_APP_ID = 'your-app-id'
ACCOUNT_KIT_APP_SECRET = 'your-app-secret'
```

Notice here we need to update the value of `ACCOUNT_KIT_APP_ID` and `ACCOUNT_KIT_APP_SECRET` obtained from Facebook Developer Console.

Let’s create the template file which will render a login form. The template is placed under `logindemo` folder and will only work if `TEMPLATES` variable is properly set up in `settings.py` file.

**Note:** This template file assumes that a `__base.html` is created and it contains a `container` and `js` block. For reference, the file can be [found here](https://github.com/all-about-django/django-account-kit-demo/blob/master/src/accountkitdemo/templates/__base.html)


```html{% raw %}
<!-- logindemo/index.html -->

{% extends "__base.html" %}

{% block container %}

<div class="card" style="height:400px; width:400px;">
    <div class="card-body">
      <h5 class="card-title">
          Account Kit Login Demo
      </h5>
      <div class="row mt-5">
          <div class="col-12">
              <button type="button" class="btn btn-primary btn-block" onClick="loginWithSMS()">Login With SMS</button>
          </div>
      </div>
      <form id="loginForm" name="loginForm" action="/sendcode" method="POST" style="display: none;">
        <input type="text" id="code" name="code">
        {% csrf_token %}
        <input type="submit" value="Submit">
      </form>
    </div>
</div>

{% endblock %}

{% block js %}
<script src="https://sdk.accountkit.com/en_US/sdk.js"></script>
<script type="text/javascript">
  AccountKit_OnInteractive = function(){
    AccountKit.init(
      {
        appId:'{{ app_id }}',
        state:"{{csrf_token}}",
        version:"{{ api_version }}",
        debug: true,
        redirect:"/sendcode"
      }
    );
  };
  function loginCallback(response) {
    if (response.status === "PARTIALLY_AUTHENTICATED") {
      document.getElementById("code").value = response.code;
      document.getElementsByName("csrfmiddlewaretoken")[0].value = response.state;
      document.getElementById("loginForm").submit();

      // here send a ajax call.
    }
    else if (response.status === "NOT_AUTHENTICATED") {
      // handle authentication failure
    }
    else if (response.status === "BAD_PARAMS") {
      // handle bad parameters
    }
  }

  function loginWithSMS(){
    AccountKit.login("PHONE",{}, loginCallback);
  }
</script>
{% endblock %}
{% endraw %}
```

Now let’s update our `urls.py` to include the `render_login_page` view

```python
# urls.py

from django.urls import path
from django.contrib import admin

from logindemo.views import (
    render_login_page,
)

urlpatterns = [

    path('', render_login_page, name='login-page'),
    path('admin/', admin.site.urls),
]
```

Now if we open [localhost:8000](http://localhost:8000), we will see the following page:

![](/assets/img/1__JWuxLdTjBvoa8IN1rrRGVg.png)

Now let’s implement another view which will be responsible for handling the callback received from Facebook’s Server and finally authenticating the users.

```python
# logindemo/views.py

import requests

from django.conf import settings
from django.shortcuts import render
from django.views.decorators.csrf import csrf_protect

from .utils import genAppSecretProof


@csrf_protect
def process_login(request):
    template_name = 'logindemo/success.html'
    context = {}
    if request.method == 'POST':
        app_access_token = '|'.join([
            'AA', settings.ACCOUNT_KIT_APP_ID,
            settings.ACCOUNT_KIT_APP_SECRET])
        params = {
            'grant_type': 'authorization_code',
            'code': request.POST.get('code'),
            'access_token': app_access_token,
        }
        try:
            response = requests.get(settings.ACCOUNT_KIT_TOKEN_EXCHANGE_BASE_URL,
                                    params=params)
        except Exception as e:
            print ('Failed to request token exchange api', e)
            response = None

        if response and response.status_code == 200:
            data = response.json()
            appsecret_proof = genAppSecretProof(
                settings.ACCOUNT_KIT_APP_SECRET,
                data['access_token']
            )
            context['user_access_token'] = data['access_token']
            context['user_id'] = data['id']
            params = {
                'access_token': data['access_token'],
                'appsecret_proof': appsecret_proof,
            }
            try:
                me_response = requests.get(
                    settings.ACCOUNT_KIT_ME_ENDPOINT_BASE_URL,
                    params=params)
            except Exception as e:
                me_response = None
                print ('Failed to request user access api', e)

            if me_response and me_response.status_code == 200:
                me_data = me_response.json()
                if me_data.get('phone'):
                    context['method'] = 'SMS'
                    context['identity'] = me_data['phone']['number']
    return render(request, template_name, context)
```

This view will render a template called `success.html` which a user can only see after being authenticated.

```html{% raw %}
{% extends "__base.html" %}
{% block container %}

<div class="card" style="height:400px; width:400px;">
    <div class="card-body">
        <div class="row">
            <div class="col">
                You have successfully logged in.
            </div>
        </div>
        <div class="row mt-4">
            <div class="col">Fingerprint</div>
            <div class="col">{{method}}</div>
        </div>
        <div class="row mt-2">
            <div class="col">Face</div>
            <div class="col">{{identity}}</div>
        </div>
        <div class="row mt-2">
            <div class="col">Person</div>
            <div class="col">{{user_id}}</div>
        </div>
        <div class="row mt-2">
            <div class="col">Person</div>
            <div class="col">{{request.user}}</div>
        </div>

    </div>
</div>
{% endblock %}
{% endraw %}
```

Now let’s finally mount this view in the `urls.py` file.

```python
# urls.py
from django.urls import path
from django.contrib import admin

from logindemo.views import (
    process_login,
)

urlpatterns = [

    # ...
    path('sendcode', process_login, name='login-process'),

]
```

This completes the entire implementation of Authentication using Account Kit in Django. On successful login, the page would look like:

![](/assets/img/1__9C6rIqEBfzxVRYie__MqpDg.png)

#### Conclusion

This post walks through implementing passwordless login using Account Kit in Django. Passwordless login is a convenient way of authenticating users without the need of them remembering their passwords.

#### Acknowledgements:

Much credits to Auth0’s blog post on [Passwordless Login with Facebook Account Kit](https://auth0.com/blog/facebook-account-kit-passwordless-authentication/).