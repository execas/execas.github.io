---
layout: post
title: "RegEx with type conversion in Python"
date: 2018-08-12
tags: [programming,python]
---

The below code illustrates how you can parse a string to multiple elements of data of specified type.

As an example, this is a string I get when I query a certain broker about a stock:

```
'45 "New York Stock Exchange" 20170522 true FOO 289.9 299.9 289.9 296.5 499874 148377740 (20170523 20170524 20172525) (236.4 275.6 228.3 275.6)'
```
The first number is the *message type number* while the rest of the string contains the message data.


## From `str` to `dict`

I use a dictionary with message type numbers as keys to store compiled RegEx patterns for different messages, e.g.:

~~~ python
msgDecode = {
    ...
    # example message
    '45' : re.compile(("(?P<ExchangeName>\".*?\")\s"
                       "(?P<Date>\S+)\s"
                       "(?P<ExchOpen>\S+)\s"
                       "(?P<Ticker>\S+)\s"
                       "(?P<Open>\S+)\s"
                       "(?P<High>\S+)\s"
                       "(?P<Low>\S+)\s"
                       "(?P<Close>\S+)\s"
                       "(?P<Volume>\S+)\s"
                       "(?P<Value>\S+)\s"
                       "(?P<NextTradingDays>\(.*?\))\s"
                       "(?P<Prev>\(.*?\))"))
      ...
      }
~~~

I use \S+ a lot in most message decode instructions since it handles symbols (e.g. AAPL), ints (e.g -42) floats (e.g. 1.3) and missing data (?), and some elements in my example string can vary between these. It also really simplfies the expressions.

Now the parsing of a message can be done using:

~~~python
msgDecode['45'].match('"New York Stock Exchange" 20170522 true FOO \
289.9 299.9 289.9 296.5 499874 148377740 \
(20170523 20170524 20172525) (236.4 275.6 228.3 275.6)').groupdict()
~~~

Which returns the following dictionary:

~~~ python
{'Close': '296.5',
 'Date': '20170522',
 'ExchOpen': 'true',
 'ExchangeName': '"New York Stock Exchange"',
 'High': '299.9',
 'Low': '289.9',
 'NextTradingDays': '(20170523 20170524 20172525)',
 'Open': '289.9',
 'Prev': '(236.4 275.6 228.3 275.6)',
 'Ticker': 'FOO',
 'Value': '148377740',
 'Volume': '499874'}
~~~

Notice that the even though the `dict` object gives us easy access to the data elements, all the data is still of type `str`.

## RegEx with type conversion

I use a small Python module called `regext` to go from a `str` to a dictionary containing properly typed data. You can find the code at the very bottom of this article.

### Compile

The RegEx pattern has to be slightly modified to support the type conversion (note the use of `regext` instead of `re`):

~~~ python
from regext import regext
msgDecode = {
    ...
    # example message
    '45' : regext.compile(("(?P<ExchangeName>\".*?\")\s"
                       "(?P<Date>\S+)\s"
                       "(?P_BOOL<ExchOpen>\S+)\s"
                       "(?P<Ticker>\S+)\s"
                       "(?P_FLOAT<Open>\S+)\s"
                       "(?P_FLOAT<High>\S+)\s"
                       "(?P_FLOAT<Low>\S+)\s"
                       "(?P_FLOAT<Close>\S+)\s"
                       "(?P_INT<Volume>\S+)\s"
                       "(?P_INT<Value>\S+)\s"
                       "(?P_LIST<NextTradingDays>\(.*?\))\s"
                       "(?P_FLOATLIST<Prev>\(.*?\))"))
      ...
      }
~~~

I call these added instructions *type labels*. These labels are peeled off and matched against supported types (see below). Note that the default type is `str`, so no type label is used for some elements (*ExchangeName*, *Date* and *Ticker*). Labels start with an underscore and are not case sensitive.


### Match

When `.match()` is run on the example string we get the following:

~~~ python
{'Close': 296.5,
 'Date': '20170522',
 'ExchOpen': True,
 'ExchangeName': '"New York Stock Exchange"',
 'High': 299.9,
 'Low': 289.9,
 'NextTradingDays': ['20170523', '20170524', '20172525'],
 'Open': 289.9,
 'Prev': [236.4, 275.6, 228.3, 275.6],
 'Ticker': 'FOO',
 'Value': 148377740,
 'Volume': 499874}
~~~

### Supported types
The code supports the following types:

~~~ python
_TYPES = {'int'  : int,
      'float': float,
      'bool' : _bool_convert,
      'list' : _to_list,
      'intlist' : _to_list_of_ints,
      'floatlist' : _to_list_of_floats,
}
~~~

Note that some of these are builtin Python type converters, and some are custom converter methods.
The supported types can be easily extended.

### No type labels

Patterns can be kept clean (no type labels) by instead supplying the `regext.compile()` method with conversion keywords.

~~~python
msgDecode = {
    # example message
    '45' : regext.compile(("(?P<ExchangeName>\".*?\")\s"
                       "(?P<Date>\S+)\s"
                       "(?P<ExchOpen>\S+)\s"
                       "(?P<Ticker>\S+)\s"
                       "(?P<Open>\S+)\s"
                       "(?P<High>\S+)\s"
                       "(?P<Low>\S+)\s"
                       "(?P<Close>\S+)\s"
                       "(?P<Volume>\S+)\s"
                       "(?P<Value>\S+)\s"
                       "(?P<NextTradingDays>\(.*?\))\s"
                       "(?P<Prev>\(.*?\))"),
                       bools = ['ExchOpen'],
                       floats = ['Open', 'High', 'Low', 'Close'],
                       ints = ['Volume', 'Value'],
                       lists = ['NextTradingDays'],
                       floatlists = ['Prev'])
      }
~~~

Keywords are named the same as labels, but have to be lowercase with an appended 's'.

## `regext` source code

```python
import re
import time

# type converters
_TRUE = ['y', 'yes', 't', 'true']
_FALSE = ['n' 'no' 'f', 'false']
def _bool_convert(truthValue):
        tv = truthValue.lower()
        if tv in _TRUE:
            b = True
        elif tv in _FALSE:
            b = False
        else:
            errmsg = "boolean value of '{}' not known".format(truthValue)
            raise ValueError, errmsg
        return b

def _to_list(listr, itemConvert = None):
    l = listr.strip("[]()")
    m = re.findall("[^,|\s]+", l)
    if itemConvert:
        m = [_TYPES[itemConvert](item) for item in m]
    return m

def _to_list_of_ints(listr):
    return _to_list(listr, itemConvert = 'int')

def _to_list_of_floats(listr):
    return _to_list(listr, itemConvert = 'float')


# type mappings
_TYPES = {'int'  : int,
          'float': float,
          'bool' : _bool_convert,
          'list' : _to_list,
          'intlist' : _to_list_of_ints,
          'floatlist' : _to_list_of_floats,
          }

class _CompiledRegextDict:
    def __init__(self, pattern, typeMapping):
        self.mappings = {}
        if typeMapping: # user has supplied keywords to map types to keys
            for typeCategory, keys in typeMapping.iteritems():
                typeName = typeCategory[:-1]
                if typeName not in _TYPES:
                    errmsg = "'{}' is not a supported type".format(typeName)
                    raise ValueError, errmsg
                else:
                    self.mappings[_TYPES[typeName]] = keys
        else: # find type-key mappings in pattern
            matchTypes = ('|'.join(_TYPES.keys()))
            parsePattern = "\?P\_({})<(\S+?)>".format(matchTypes)
            typeMapping = re.findall(parsePattern , pattern, re.IGNORECASE)
            for mapping in typeMapping:
                typ = _TYPES[mapping[0].lower()]
                key = mapping[1]
                self.mappings.setdefault(typ, []).append(key)
            # clean pattern
            subPattern = "\_({})<".format(matchTypes)
            pattern = re.sub(subPattern, '<', pattern, flags=re.IGNORECASE)
        self.pattern = re.compile(pattern)

    def match(self, s, convertBypass = False):
        m = re.match(self.pattern, s).groupdict()
        if not convertBypass:
            for typeConverter, keys in self.mappings.iteritems():
                for k in keys:
                    try:
                        m[k] = typeConverter(m[k])
                    except ValueError:
                        pass # leave unconverted
        return m

def compile(pattern, **typeMapping):
    return _CompiledRegextDict(pattern, typeMapping)
```
