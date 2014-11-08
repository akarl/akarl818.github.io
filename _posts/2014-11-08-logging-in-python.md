---
layout: post
title: Logging in Django
---

You know all the nice looking emails you get when your Django application crashes?
We want to get the same nice emails in a `try except` block while the program continues to run.

We are going to create a management command that are processing some users, we don't want this
command to exit when there is an error, we want it to continue running.

#### Creating the management command

Let's start by creating a simple management command to process some users from a cronjob. Simple enough right?

{% highlight python %}
from django.core.management.base import NoArgsCommand
import logging

logger = logging.getLogger(__name__)

class Command(NoArgsCommand):
    def handle(self):
        for user in users_who_needs_processing():
            try:
                self.process_user(user)
            except Exception, e:
                logger.exception('Error while processing user %i', user.pk)

    def process_user(self, user):
        logger.debug('Processing user %i', user.pk)
        # Do some work on user...
{% endhighlight %}

#### Let's explain.

{% highlight python %}
logger = logging.getLogger(__name__)
{% endhighlight %}

This gets a logger instance with the name of the current module, in this case it would be something like
`myapp.management.commands.process_users`.

{% highlight python %}
except Exception, e:
    logger.exception('Error while processing user %i', user.pk)
{% endhighlight %}

This is the most useful one, `.exception()` will log the stacktrace and
we can continue to process the other users while still logging the error.

#### Configuring Django

Django configures logging with a dict, but this dict is nothing special to Django, this is something built in
to Python's logging module.

The default setup only sets up logging for these modules:

- `django` Logs everything to the console.
- `django.request` Logs to console and mails administators.
- `django.security` Mails admins
- `py.warnings` Logs to console

So let's create our own logging configuration to mail administrators when
an error occurs while processing the users, stream the all warnings to syslog and
show the debug messages in the console when `DEBUG=False`.

{% highlight python %}
# settings.py

LOGGING = {
    'version': 1,
    'disable_existing_loggers': False,
    'filters': {
        'require_debug_false': {
            '()': 'django.utils.log.RequireDebugFalse',
        },
        'require_debug_true': {
            '()': 'django.utils.log.RequireDebugTrue',
        },
    },
    'handlers': {
        'console': {
            'level': 'DEBUG',
            'filters': ['require_debug_true'],
            'class': 'logging.StreamHandler',
        },
        'syslog': {
            'level': 'WARNING',
            'filters': ['require_debug_false'],
            'class': 'logging.SysLogHandler',
        },
        'mail_admins': {
            'level': 'ERROR',
            'filters': ['require_debug_false'],
            'class': 'django.utils.log.AdminEmailHandler'
        }
    },
    'loggers': {
        'myapp.management': {
            'handlers': ['mail_admins', 'syslog', 'console'],
            'level': 'DEBUG',
        },
        'django': {
            'handlers': ['mail_admins', 'syslog', 'console'],
            'level': 'INFO',
        },
    }
}
{% endhighlight %}

The `LOGGING` dict is later passed to Python's logging module using `logging.dictConf()` before your application starts.

You can read more about the configuration [here](https://docs.python.org/2/library/logging.config.html#dictionary-schema-details)

And more about the Pythons logging library [here](https://docs.python.org/2/library/logging.html)
