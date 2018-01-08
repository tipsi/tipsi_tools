# About this package

[![Build Status](https://travis-ci.org/tipsi/tipsi_tools.svg?branch=master)](https://travis-ci.org/tipsi/tipsi_tools)
[![PyPi version](https://img.shields.io/pypi/v/tipsi_tools.svg)](https://pypi.python.org/pypi/tipsi_tools)


Here are set of internal tools that are shared between different projects internally. Originally most tools related to testing, so they provide some base classes for various cases in testing

**NOTE: all our tools are intentially support only 3.5+ python.**
Some might work with other versions, but we're going to be free from all these crutches to backport things like `async/await` to lower versions, so if it works - fine, if not - feel free to send PR, but it isn't going to be merged all times.

## Testing helpers

### ApiUrls

Defined in `tipsi_tools/testing/__init__.py`. Required for defining nested urls with formatting.

You can use it in fixtures, like:


```python
@pytest.fixture(scope='session')
def api(api_v_base):
    yield ApiUrls('{}/'.format(api_v_base), {
        'password_reset_request': 'password/request/code/',
        'password_reset': 'password/reset/',
        'user_review_list': 'user/{user_id}/review/',
        'user_review': 'user/{user_id}/review/{review_id}/',
        'wine_review': 'wine/{wine_id}/review/',
        'drink_review': 'drink/{drink_id}/review/',
    })


def test_review_list(user, api):
    resp = user.get_json(api.user_review_list(user_id=user1.id), {'page_size': 2})
```




### PropsMeta

You can find source in `tipsi_tools/testing/meta.py`.

For now it convert methods that are started with `prop__` into descriptors with cache.

```python
class A(metaclass=PropsMeta):
    def prop__conn(self):
        conn = SomeConnection()
        return conn
```

Became:

```python
class A:
    @property
    def conn(self):
        if not hasattr(self, '__conn'):
            setattr(self, '__conn', SomeConnection())
        return self.__conn
```

Thus it allows quite nice style of testing with lazy initialization. Like:

```python
class MyTest(TestCase, metaclass=PropsMeta):
    def prop__conn(self):
        return psycopg2.connect('')

    def prop__cursor(self):
        return self.conn.cursor()

    def test_simple_query(self):
        self.cursor.execute('select 1;')
        row = self.cursor.fetchone()
        assert row[0] == 1, 'Row: {}'.format(row)

```

Here you just get and use `self.cursor`, but automatically you get connection and cursor and cache they.

This is just simple example, complex tests can use more deep relations in tests. And this approach is way more easier and faster than complex `setUp` methods.


### AIOTestCase

**NOTE: we're highly suggest to use pytest alongside with existing async tests plugins**

Base for asyncronous test cases, you can use it as drop-in replacement for pre-existent tests to be able:

* write asyncronous test methods
* write asyncronous `setUp` and `tearDown` methods
* use asyncronous function in `assertRaises`

```python
class ExampleCase(AIOTestCase):
    async setUp(self):
        await async_setup()

    async tearDown(self):
        await async_teardown()

    async division(self):
        1/0

    async test_example(self):
        await self.assertRaises(ZeroDivisionError, self.async_division)
```

## tipsi_tools.unix helpers

Basic unix helpers

* run - run command in shell
* succ - wrapper around `run` with return code and stderr check
* wait_socket - wait for socket awailable (eg. you can wait for postgresql with `wait_socket('localhost', 5432)`
* asucc - asynchronous version of `succ` for use with `await`. supports realtime logging

#### interpolate_sysenv

Format string with system variables + defaults.

```python
PG_DEFAULTS = {
    'PGDATABASE': 'postgres',
    'PGPORT': 5432,
    'PGHOST': 'localhost',
    'PGUSER': 'postgres',
    'PGPASSWORD': '',
    }
DSN = interpolate_sysenv('postgresql://{PGUSER}:{PGPASSWORD}@{PGHOST}:{PGPORT}/{PGDATABASE}', PG_DEFAULTS)
```


## tipsi_tools.logging.JSFormatter

Enable json output with additional fields, suitable for structured logging into ELK or similar solutions.

Accepts `env_vars` key with environmental keys that should be included into log.

```python
# this example uses safe_logger as handler (pip install safe_logger)
import logging
import logging.config


LOGGING = {
    'version': 1,
    'disable_existing_loggers': True,
    'formatters': {
        'json': {
            '()': 'tipsi_tools.logging.JSFormatter',
            'env_vars': ['HOME'],
        },
        'standard': {
            'format': '%(asctime)s [%(levelname)s] %(name)s: %(message)s'
        },
    },
    'handlers': {
        'default': {
            'level': 'DEBUG',
            'class': 'safe_logger.TimedRotatingFileHandlerSafe',
            'filename': 'test_json.log',
            'when': 'midnight',
            'interval': 1,
            'backupCount': 30,
            'formatter': 'json',
            },
    },
    'loggers': {
        '': {
            'handlers': ['default'],
            'level': 'DEBUG',
        },
    },
}

logging.config.dictConfig(LOGGING)
log = logging.getLogger('TestLogger')

log.debug('test debug')
log.info('test info')
log.warn('test warn')
log.error('test error')
```

## tipsi_tools.drf.serializers.EnumSerializer

Allow you to deserealize incoming strings into `Enum` values.
You should add `EnumSerializer` into your serializers by hand.

```python
from enum import IntEnum

from django.db import models
from rest_framework import serializers

from tipsi_tools.drf.serializers import EnumSerializer


class MyEnum(IntEnum):
  one = 1
  two = 2

class ExampleModel(models.Model):
  value = models.IntegerField(choices=[(x.name, x.value) for x in MyEnum])

class ExampleSerializer(serializers.ModelSerializer):
  value = EnumSerializer(MyEnum)

# this allows you to post value as: {'value': 'one'}
```

Due to `Enum` and `IntegerField` realizations you may use `Enum.value` in querysets

```python
ExampleModel.objects.filter(value=MyEnum.two)
```

## tipsi_tools.django.log_requests.LoggerMiddleware

LoggerMiddleware will log request meta + raw post data into log.

For django<1.10 please use `tipsi_tools.django.log_requests.DeprecatedLoggerMiddleware`


## tipsi_tools.drf.use_form

Helps to use power of serializers for simple APIs checks.


```python
from rest_framework import serializers
from rest_framework.decorators import api_view
from tipsi_tools.drf import use_form


class SimpleForm(serializers.Serializer):
    test_int = serializers.IntegerField()
    test_str = serializers.CharField()


@api_view(['GET'])
@use_form(SimpleForm)
def my_api(data):
    print(f'Data: {data["test_int"]} and {data["test_str"]}')
```


## tipsi_tools.python.execfile

Backport of python's 2 `execfile` function.

Usage: execfile('path/to/file.py', globals(), locals())

Returns: True if file exists and executed, False if file doesn't exist


## tipsi_tools.doc_utils.tipsi_sphinx

Sphinx extensions to generate documentation for django restframework serializers and examples for http requests.

In order to use them specify dependency for package installation:
```
pip install tipsi_tools[doc_utils]
```

Usage:
```
# Add to Sphinx conf.py
extensions = [
    # ...
    'tipsi_tools.doc_utils.tipsi_sphinx.dyn_serializer',
    'tipsi_tools.doc_utils.tipsi_sphinx.http_log'
]
```

## Commands

### tipsi_env_yaml

Convert template yaml with substituion of `%{ENV_NAME}` strings to appropriate environment variables.

Usage: `tipsi_env_yaml src_file dst_file`


### tipsi_ci_script

Helper to run default CI pipeline. Defaults are set up for [giltab defaults](https://docs.gitlab.com/ee/ci/variables/#predefined-variables-environment-variables). Includes stages:

* build docker image with temporary name (commit sha by default)
* run tests (optional)
* push branch (by default only for master and staging branches)
* push tag, if there are tags
* cache image with common name
* delete image with temporary name

It's optimized for parallel launches, so you need to use unique temporary name (`--temp-name`). We want keep our system clean if possible, so we'll delete this tag in the end. But we don't want to repeat basic steps over and over, so we will cache image with common cache name (`--cache-name`), it will remove previous cached image.

### tipsi_wait

Wait for socket awailable/not-awailable with timeout.

```
# Wait until database port up for 180 seconds
tipsi_wait -t 180 postgres 5432

# Wait until nginx port down for 30 seconds
tipsi_wait -t 30 nginx 80
```

### run_filebeat

* checks environmental variables `-e KEY=VALUE -e KEY2=VALUE2`
* converts yaml template `tipsi_env_yaml {TEMPLATE} /tmp/filebeat.yml`
* run `/usr/bin/filebeat /tmp/filebeat.yml`

```
run_filebeat -e CHECKME=VALUE path_to_template
```


### doc_serializer

* output rst with list of serializers
* generates documentation artifacts for serializers

```
usage: doc_serializer [-h] [--rst] [--artifacts]

Parse serializers sources

optional arguments:
  -h, --help   show this help message and exit
  --rst        Output rst with serializers
  --artifacts  Write serializers artifacts
```
