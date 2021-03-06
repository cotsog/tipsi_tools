# About this package

[![Build Status](https://travis-ci.org/tipsi/tipsi_tools.svg?branch=master)](https://travis-ci.org/tipsi/tipsi_tools)

Here are set of internal tools that are shared between different projects internally. Originally most tools related to testing, so they provide some base classes for various cases in testing

**NOTE: all our tools are intentially support only 3.5+ python.**
Some might work with other versions, but we're going to be free from all these crutches to backport things like `async/await` to lower versions, so if it works - fine, if not - feel free to send PR, but it isn't going to be merged all times.


## PropsMeta

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


## AIOTestCase

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

## Commands

### tipsi_env_yaml

Convert template yaml with substituion of `%{ENV_NAME}` strings to appropriate environment variables.

Usage: `tipsi_env_yaml src_file dst_file`
