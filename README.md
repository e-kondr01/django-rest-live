# Django REST Live

[![CircleCI](https://circleci.com/gh/pennlabs/django-rest-live.svg?style=shield)](https://circleci.com/gh/pennlabs/django-rest-live)
[![Coverage Status](https://codecov.io/gh/pennlabs/django-rest-live/branch/master/graph/badge.svg)](https://codecov.io/gh/pennlabs/django-rest-live)
[![PyPi Package](https://img.shields.io/pypi/v/django-rest-live.svg)](https://pypi.org/project/django-rest-live/)

`django-rest-live` adds real-time subscriptions over websockets to [Django REST Framework](https://github.com/encode/django-rest-framework)
by leveraging websocket support provided by [Django Channels](https://github.com/django/channels).

## Contents
* [Inspiration and Goals](#inspiration-and-goals)
* [Dependencies](#dependencies)
* [Installation](#installation)
* [Usage](#usage)
    + [Basic Usage](#basic-usage)
      - [Server-Side](#server-side)
      - [Client-Side](#client-side)
    + [Advanced Usage](#advanced-usage)
      - [Subscribe to groups](#subscribe-to-groups)
      - [Permissions](#permissions)
      - [Conditional Serializer Pattern](#conditional-serializer-pattern)
* [Limitations](#limitations)

## Inspiration and Goals
The goal of this project is to enable realtime subscriptions without requiring any boilerplate or changing
any existing REST Framework views or serializers.
`django-rest-live` took initial inspiration from [this article by Kit La Touche](https://www.oddbird.net/2018/12/12/channels-and-drf/).

## Dependencies
- [Django](https://github.com/django/django/) (3.0 and up)
- [Django Channels](https://github.com/django/channels) (2.0 and up) 
- [Django REST Framework](https://github.com/encode/django-rest-framework/)
- [`channels_redis`](https://github.com/django/channels_redis) for
  [channel layer](https://channels.readthedocs.io/en/latest/topics/channel_layers.html) support in production.

## Installation

If your project already uses REST framework, but this is the first realtime component,
then make sure to install and properly configure Django Channels before continuing.

You can find details in [the Channels documentation](https://channels.readthedocs.io/en/latest/installation.html).

1. Add `rest_live` to your `INSTALLED_APPS`
```python
INSTALLED_APPS = [
    # Any other django apps
    "rest_framework",
    "channels",
    "rest_live",
]
```
    
2. Add `rest_live.consumers.SubscriptionConsumer` to your websocket routing. Feel'
free to choose any URL path, here we've chosen `/ws/subscribe/`. 
```python
from rest_live.consumers import SubscriptionConsumer

websockets = URLRouter(
    [path("ws/subscribe/", SubscriptionConsumer, name="subscriptions")]
)
application = ProtocolTypeRouter({
    "websocket": websockets
})
```

That's it! You're now ready to configure and use `django-rest-live`.

## Usage

These docs will use an example to-do app called `todolist` with the following models and serializers:
```python
# todolist/models.py
from django.db import models

class List(models.Model):
    name = models.CharField(max_length=64)

class Task(models.Model):
    text = models.CharField(max_length=140)
    done = models.BooleanField(default=False)
    list = models.ForeignKey("List", on_delete=models.CASCADE)

# todolist/serializers.py
from rest_framework import serializers

class TaskSerializer(serializers.ModelSerializer):
    class Meta:
        model = Task
        fields = ["id", "text", "done"]

class TodoListSerializer(serializers.ModelSerializer):
    tasks = TaskSerializer(many=True, read_only=True)
    class Meta:
        model = List
        fields = ["id", "name", "tasks"]
```

### Basic Usage

An important thing to remember about the `django-rest-live` package is that its sole purpose is sending updates to clients
over websocket connections. Clients should still use normal REST framework endpoints generated by ViewSets and views
to get initial data to populate a page, as well as any write-driven behavior (`POST`, `PATCH`, `PUT`, `DELETE`).
`django-rest-live` gets rid of the need for periodic GET requests for updated data.

#### Server-Side
In order to tell `django-rest-live` that you'd like to allow clients to subscribe to updates for a specific model, the package
provides the `@subscribable` class decorator. This decorator is meant to be applied to `ModelSerializer` subclasses,
so that the package can register both which models to allow subscriptions to as well as how those models should be
serialized when being sent to the client. To enable clients to subscribe to updates to individual to-dos, all you need
to do is apply the decorator to the `TodoSerializer`:

```python
# todolist/serializers.py
from rest_live.decorators import subscribable
...
@subscribable()
class TaskSerializer(serializers.ModelSerializer):
    ...
```

#### Client-Side
Subscribing to model updates from a client requires opening a [WebSocket](https://developer.mozilla.org/en-US/docs/Web/API/WebSocket)
connection to the URL you specified during setup. In our example case, that URL is `/ws/subscribe/`. After the connection
is established, send a JSON message (using `JSON.stringify()`) in this format:

```json5
{
  "model": "todolist.Task",
  "value": 1 
}
```

The model label should be in Django's standard `app.modelname` format. `value` field here is set to the value for
the [primary key](https://docs.djangoproject.com/en/3.1/topics/db/queries/#the-pk-lookup-shortcut) for the model instance
we're subscribing to. This is generally the value of the `id` field, but is equivalent to querying
for `Task.objects.filter(pk=<value>)`.

The example message above would subscribe to updates for the todo task with an primary key of 1.
As mentioned above, the client should make a GET request to get the entire list, with all its tasks and their
associated IDs, to figure out which IDs to subscribe to.

When the Task with primary key `1` updates, a message in this format will be sent over the websocket:

```json
{
    "model": "test_app.Todo",
    "instance": {"id": 1, "text": "test", "done": true},
    "action": "UPDATED",
    "group_key_value": 1
}
```

Valid `action` values are `UPDATED`, `CREATED`, and `DELETED`.
`group_key_value` might seem erroneous in this example, but is useful for group subscriptions, described in the next
section.

### Advanced Usage

#### Subscribe to groups
As mentioned above, subscriptions are grouped by the primary key by default: you send one message to the websocket to get updates for
a single Task with a given primary key. But in the todo list example, you'd generally be interested in an entire
list of tasks, including being notified of any tasks which have been created since the page was first loaded.

Rather than subscribe all tasks individually, you want to subscribe to a list: an entire group of tasks.
This is where group keys come in. Pass in the `group_key` you'd like to group tasks by
to the `@subscribable` decorator to register subscriptions for an entire list:


```python
# todolist/serializers.py
from rest_live.decorators import subscribable
...
@subscribable(group_key="list_id")
class TaskSerializer(serializers.ModelSerializer):
    ...
```

On the client side, we now have to specify the group key `property` we are subscribing to. In this case, the `list_id`
of the list you'd like to get updates from:

```json5
{
  "model": "todolist.Task",
  "property": "list_id",
  "value": 1
}
```

This will subscribe you to updates for all Tasks in the list which has ID 1.

What's important to remember here is that while the field is defined as a `ForeignKey` called `list` on the model,
the underlying integer field in the database that links together Tasks and Lists is called `list_id`. More generally,
`<fieldname>_id` for any related fieldname on the model.

The `subscribable` decorator can be stacked. If you want to enable subscriptions by both `list_id` for entire lists and
`pk` for individual tasks, add two decorators:

```python
# todolist/serializers.py
from rest_live.decorators import subscribable
...
@subscribable()
@subscribable(group_key="list_id")
class TaskSerializer(serializers.ModelSerializer):
    ...
```

Just note that clients which subscribe to list updates and individual pk updates will receive two messages when a task
updates.


#### Permissions
The `@subscribable` decorator also takes in a parameter called `check_permission`. This is a function which takes in 
a User and a model instance and determines whether or not the given user can access the given model. To make sure
users can only subscribe to lists when they are logged in, this code would suffice:

```python
# todolist/serializers.py
from rest_live.decorators import subscribable

def has_auth(user, instance):
    return user.is_authenticated

@subscribable(group_key="list_id", check_permission=has_auth)
class TaskSerializer(serializers.ModelSerializer):
    ...
```

#### Conditional Serializer Pattern
A common pattern in Django REST Framework is showing users different serializers based on their authentication status
by overloading `get_serializer_class()` in a `ViewSet`. This pattern can be mirrored in `django-rest-live`
using the `check_permission` callback. Let's say that for our to-do app, un-authenticated users can view tasks, but
cannot see if they're completed. Users can subscribe with the proper serializer with the following `serializers.py`:

```python
# todolist/serializers.py
from rest_framework import serializers
from rest_live.decorators import subscribable


def has_auth(user, instance):
    return user.is_authenticated

def has_no_auth(user, instance):
    return not has_auth(user, instance)

@subscribable(group_key="list_id", check_permission=has_auth)
class AuthedTaskSerializer(serializers.ModelSerializer):
    class Meta:
        model = Task
        fields = ["id", "text", "done"]

@subscribable(group_key="list_id", check_permission=has_no_auth)
class NoAuthTaskSerializer(serializers.ModelSerializer):
    class Meta:
        model = Task
        fields = ["id", "text"]
```
Clients in both situations would send the same subscribe request, but would receive different model instances depending
on their authentication status. 

## Limitations
This package works by listening in on model lifecycle events sent off by Django's [signal dispatcher](https://docs.djangoproject.com/en/3.1/topics/signals/).
Specifically, the [`post_save`](https://docs.djangoproject.com/en/3.1/ref/signals/#post-save)
and [`post_delete`](https://docs.djangoproject.com/en/3.1/ref/signals/#post-delete) signals. This means that `django-rest-live`
can only pick up changes that Django knows about. Bulk operations, like `filter().update()`, `bulk_create`
and `bulk_delete` do not trigger Django's lifecycle signals, so updates will not be sent.

## TODO

- [x] Permissions 
- [x] Conditional Serializers
- [ ] Permissions helpers for DRF `Permission` classes
- [ ] Error handling and reporting
- [ ] Expand related fields from `field` to `field_id`
