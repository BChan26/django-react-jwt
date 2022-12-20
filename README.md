

[![General Assembly Logo](https://camo.githubusercontent.com/1a91b05b8f4d44b5bbfb83abac2b0996d8e26c92/687474703a2f2f692e696d6775722e636f6d2f6b6538555354712e706e67)](https://generalassemb.ly/education/web-development-immersive)

# Django React JWT

Have Django provide a JWT token to the React client in order to make athenticated API calls.

## Prerequisites

- tunr
- simple react app

## Objectives

By the end of this, developers should be able to:

- Athenticate a user using Django Rest Framework
- Provide a JWT token to the React client app
- Use the token subsquent authorized API requests.

## Introduction

We are going to use our tunr django app that we built up in our previous lessons.

## Checklist:

<details>
  <summary><strong>Tunr App running with data</strong></summary>

You should be able to run your tunr django app and go to `http://localhost:8000/artists` and get a list of artists rendered through the django rest framework UI.

</details>



<summary><strong>Simple react app working in heroku?</strong></summary>
Create a React App that is white listed via CORS to make requests to the tunr django app. Once you are able to make a fetch request to `http://localhost:8000/artists` push it to heroku and replace config vars in heroku to point to the django address instead.

> if in your `.env` file you have a `REACT_APP_API_URL=http://localhost:8000/` then add the config var `REACT_APP_API_URL` in heroku with the value being the address of your django server in heroku. i.e. `mytunrdjangoapp.heruko.app` **\*\*Do not** copy this example url, make sure to look for your own.

</details>

<details>
  <summary><strong>React app can fetch data from django both hosted in heroku?</strong></summary>

> Your react app hosted in Heroku should be fetching data from your django instance in Heroku.

</details>

## Install and Configure Django Rest Framework Simple JWT (DRF SJWT)

Let's begin by installing the simple JWT package in our django tunr app folder. Make sure your virtual enviroment is activated `pipenv shell`

`pip install djangorestframework-simplejwt`

Modify the `REST_FRAMEWORK` in your django project settings to add the authentication class from DRF SJWT

```python
REST_FRAMEWORK = {
    ...
    'DEFAULT_AUTHENTICATION_CLASSES': (
        ...
        'rest_framework_simplejwt.authentication.JWTAuthentication',
    )
    ...
}
```

Modify the `tunr_django` urls to wire the views that allow you to obtain and refresh JWT tokens. This functionality is included with `rest_framework_simplejwt`:
library:

```python
from rest_framework_simplejwt.views import (
    TokenObtainPairView,
    TokenRefreshView,
)

urlpatterns = [
    ...
    path('api/token/', TokenObtainPairView.as_view(), name='token_obtain_pair'),
    path('api/token/refresh/', TokenRefreshView.as_view(), name='token_refresh'),
    ...
]
```

Now let's test getting a token from the command line using cURL. Make sure you have credentials of the superuser you created in `tunr` or create a new one.

For testing this will be the example credentials:
> username: admin
> password: admin

```bash
curl \
  -X POST \
  -H "Content-Type: application/json" \
  -d '{"username":"admin", "password":"admin"}' \
  http://localhost:8000/api/token/
```

You should see something like this output:
```bash
{"refresh":"eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJ0b2tlbl90eXBlIjoicmVmcmVzaCIsImV4cCI6MTY0MjU2MTQwOCwiaWF0IjoxNjQyNDc1MDA4LCJqdGkiOiJmZGRiZjhlMDAzNjQ0MTllYTBlMmNhZDk5NzU5ZTAzOCIsInVzZXJfaWQiOjF9.zzdUyuxAnRvR4gwxc9FE3xBcyGFGqBJSzZpbbPwAlLI","access":"eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJ0b2tlbl90eXBlIjoiYWNjZXNzIiwiZXhwIjoxNjQyNDc1MzA4LCJpYXQiOjE2NDI0NzUwMDgsImp0aSI6IjU5YTFiZjM1YWNhZjQ5ZjliMDM0MjJjYTNlMjBkNmNiIiwidXNlcl9pZCI6MX0.kBZOd-c6NcXjnXzC-Ov4GtH2Q6llRQ9D1GxOts8EoIA"}
```

#### And voila, simple right?!
Now you should be able to get JWT tokens that can authorize your API calls. **Fortunately, we're far from over!!**



## User serializer, view and urls

To begin with user serialization lets start by creating an `accounts` app, if you already don't have one.

To allow user creation all we need to do is serialized Django's user model inside `accounts.models`

#### Serializer
```python
from rest_framework import serializers
from django.contrib.auth import get_user_model


# Use get_user_model instead of importing the User model directly
# from django.contrib.auth.models import User
# This allows getting a custom user models instead if there was one
User = get_user_model()


class UserSerializer(serializers.ModelSerializer):

    # The write_only option ensures the field may be used when updating or creating an instance,
    # but it is not included when serializing the representation.
    password = serializers.CharField(write_only=True)

    # The create method is provided by the ModelSerializer
    # But if needed method is essentially just: return ExampleModel.objects.create(**validated_data)
    def create(self, validated_data):
        """
        Create and return a new User instance, given the validated data.
        """
        # make sure to user create_user method and not create
        # the later will not know how to hash the password properly
        user = User.objects.create_user(
            username=validated_data["username"],
            password=validated_data["password"],
            email=validated_data["email"],
        )
        return user

    class Meta:
        model = User
        fields = ("id", "username", "password", "email")

```
#### View

We'll user the CreateAPIView that comes with `rest_framework` inside `accounts.views`

```python
from rest_framework import permissions
from rest_framework.generics import CreateAPIView
from django.contrib.auth import get_user_model

from .serializers import UserSerializer

# Create your views here.
class CreateUserView(CreateAPIView):

    model = get_user_model()
    permission_classes = [
        permissions.AllowAny,  # Unauthenticated users must be able to sign up
    ]
    serializer_class = UserSerializer
```

#### Url
Finally lets wire the view we just added in `accounts.urls`

```python
urlpatterns = [
  ...
  path("api/signup/", CreateUserView.as_view()),
  ...
]
```

## React Client

### React app setup and simple fetch
- Create a react app
- Create simple fetch logic that `console.log` artists

Navigate to a your sandbox and create the `tunr-react`

`npx create-react-app tunr-react`

Create and `.env.local` file and set the api url in an envarionment variable, in this case our server would be at `localhost:8000`.

`REACT_APP_API_URL=http://localhost:8000/`

Write the artist fetch logic to confirm your client can talk to your backend server. If it is, you should see an array of the artist objects in the database printed in the console.

```jsx
  useEffect(() => {
    const res = axios.get(process.env.REACT_APP_API_URL + 'artists/')
      .then(res => {
        console.log('The app is responding fine', res);
      });
  }, []);
```

Nothing shows up? Make sure that:
- You're getting a `200` response code from your fetch
- You have artist objects in your database

### Authenticating a user a.k.a getting a token with `fetch`

Let's create a component with a controlled form that we'll use to submit the username and password to our `token.api` route in order to get a refresh and an access token.

- Add a state object inside your component
- Create an HTML form with a text input and a password input field
- Write the handleChange logic
- Write the onSubmit `fetch` logic
- Wire the handleChange and onSubmit to your form and its inputs
- Make sure to have an options object to pass as a second arguments to your fetch, in order to add the `Content-Type` header
- Append the `body` 

### Signing up a user

Same as before let's just add two more fields,`email` and `confirmpwd`

Use the latter to create on page logic to make sure the form cannot be submitted if the password doesn't match the confirm password field.

### Create an artist object

Same idea as before but now we need to account for the "Bearer" token. In order to be able to POST to our app, we need to have in the headers of our request an `Authorization` key along with the access token we got from the server prefixed by the word "Bearer". So in the case of the first access token we received, that token would be:
`Bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJ0b2tlbl90eXBlIjoiYWNjZXNzIiwiZXhwIjoxNjQyNDc1MzA4LCJpYXQiOjE2NDI0NzUwMDgsImp0aSI6IjU5YTFiZjM1YWNhZjQ5ZjliMDM0MjJjYTNlMjBkNmNiIiwidXNlcl9pZCI6MX0.kBZOd-c6NcXjnXzC-Ov4GtH2Q6llRQ9D1GxOts8EoIA`





## Additional Resources

- [Simple JWT](https://django-rest-framework-simplejwt.readthedocs.io/en/latest/getting_started.html)
- [DRF Extensions](https://chibisov.github.io/drf-extensions/docs/)

## [License](LICENSE)

1. All content is licensed under a CC­BY­NC­SA 4.0 license.
2. All software code is licensed under GNU GPLv3. For commercial use or alternative licensing, please contact legal@ga.co.
