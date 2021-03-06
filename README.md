[![Coverage Status](https://coveralls.io/repos/github/nbs-system/nxtool-ng/badge.svg?branch=master)](https://coveralls.io/github/nbs-system/nxtool-ng?branch=master)
[![Code Health](https://landscape.io/github/nbs-system/nxtool-ng/master/landscape.svg?style=flat)](https://landscape.io/github/nbs-system/nxtool-ng/master)
[![Code Climate](https://codeclimate.com/github/nbs-system/nxtool-ng/badges/gpa.svg)](https://codeclimate.com/github/nbs-system/nxtool-ng)
[![Build Status](https://travis-ci.org/nbs-system/nxtool-ng.svg?branch=master)](https://travis-ci.org/nbs-system/nxtool-ng)

```
              __                __                  
.-----.--.--.|  |_.-----.-----.|  |____.-----.-----.
|     |_   _||   _|  _  |  _  ||  |____|     |  _  |
|__|__|__.__||____|_____|_____||__|    |__|__|___  |
                                             |_____|

 -- Because life is too short to transform naxsi logs into rules by hand.
```
          
nxtool-ng is a tool to magically transform your [naxsi]( http://naxsi.org ) logs into useful rules.
It can get its data from your elastic instance, or you can feed it flat files,
and it will magically show you some statistics, generate relevant whitelists,
provide type-based rules, …

It works with *modules*, that are generating whitelists, without overlapping each other.

Proudly powered by [Python]( https://python.org ) (2 and 3 by the way),
using (optionally) [elasticsearch-dsl]( https://elasticsearch-dsl.readthedocs.org/en/latest/ ),
written with love and tears by the great people of [NBS-System]( https://nbs-system.com ),
nxtool-ng is released under the [GPL]( https://gnu.org/licenses/gpl.html ).

# Installation

Nxtool-ng depends on [nxapi](https://github.com/nbs-system/nxapi) for naxsi-related magic,
and optionally on [elasticsearch-dsl]( https://github.com/elastic/elasticsearch-dsl-py )
if you want to generate rules from an Elastic instance. You can install them with

### Elasticsearch 5.x
`pip install -r ./requirements-v5.txt`

### Elasticsearch 2.x
`pip install -r ./requirements-v2.txt`

### Elasticsearch 1.x

`pip install -r /requirements-v1.txt`

# Usage

```bash
$ python nxtool.py -h
usage: nxtool.py [-h] [-v] [--elastic] [--flat-file] [--stdin] [--archive]
                 [--typing] [--whitelist] [--filter FILTER] [--stats]
                 [hostname]

Sweet tool to help you managing your naxsi logs.

positional arguments:
  hostname

optional arguments:
  -h, --help       show this help message and exit
  -v, --verbose

Log sources:
  --elastic-source
  --flat-file
  --stdin

Actions:
  --typing
  --whitelist
  --elastic-dest
  --filter FILTER
  --stats
  --slack
```

First you can populate an elasticsearch instance by:
```bash
$ python nxtool.py --elastic-dest --flat-file example.com.log
```

For example, if you want some stats about `example.com` using your elasticsearch instance:

```bash
$ python nxtool.py --elastic-source --stats example.com
2.39.218.24: 14
14.76.8.132: 18
13.24.13.122: 8
157.5.39.176: 13
19.187.104.23: 8
80.24.150.43: 21
50.2.176.10: 198
79.14.72.145: 44
14.26.23.213: 80
86.242.8.36: 58

# URI #
/cache.php: 12
/11.php: 12
/call-for-paper-contact/: 82
/: 22
/xmlrpc.php: 22
/en/production/type.asp: 41
/contact/: 21
/wp-json/oembed/1.0/embed: 38
/en/production/formation.asp: 68
/totallylegit/: 14

# ZONE #
BODY: 276
ARGS|NAME: 24
URL: 22
ARGS: 146
HEADERS: 54
BODY|NAME: 10
FILE_EXT: 4

# SERVER #
example.com: 536
```

To generate some whitelists for `example.com`, using your elasticsearch instance:

```bash
$ python nxtool.py --elastic-source --whitelist example.com
[+] Generating Google analytics rules
[+] Generating Image 1002 rules
[+] Generating cookies rules
[+] Generating var + zone rules
[+] Generating site rules
[+] Generating zone rules
[+] Generating url rules

Generated whitelists:
	BasicRule wl:1310,1311 "mz:$HEADERS_VAR:cookie" "msg:Cookies";
```

You can add the `--verbose` flag if you want more information about what's going on.
If you're using *flat files*, you can either pass, well flat files, but also *archives*,
like `.zip` or `.tar.gz`.

You can add the `--slack` flag if you want loosen constraints on whitelist generation.
It can be useful with only little amount of logs.

You can also use nxtool-ng to query your elasticsearch instance, for example
to search for access to `/admin`, that triggered the rule `1010` in the `HEADERS`:

```bash
$ python nxtool.py --elastic-source --filter 'uri=/admin,zone=HEADERS,id=1010'

zone: HEADERS
ip: 133.144.211.172
whitelisted: false
uri: /admin
comments: import:2016-08-30 09:44:17.938620
server: example.com
content: 
var_name: cookie
country: 
date: 2016-08-30T09:45:13+0200
id: 1010

zone: HEADERS
ip: 15.125.251.122
whitelisted: false
uri: /admin
comments: import:2016-08-30 11:00:03.523580
server: example.com
content: 
var_name: cookie
country: 
date: 2016-08-30T11:06:36+0200
id: 1010

```

It's also possible to *type* your parameters, to tighten a bit the security of
your application:

```
$ python nxtool.py --elastic-source --typing --verbose example.com

Generated types:

BasicRule negative "rx:^$" "msg:empty" "mz:FILE_EXT:user_avatar" "s:BLOCK";
BasicRule negative "rx:^$" "msg:empty" "mz:FILE_EXT:society_logo" "s:BLOCK";
BasicRule negative "rx:^https?://([0-9a-z-.]+\.)+[\w?+-=&/ ]+$" "msg:url" "mz:ARGS:url" "s:BLOCK";
```

# Note on the structure of ElasticSearch entries

Each core rule violation is logged in a NAXSI_FMT entry. Each violation is reported once in the ElasticSearch instance.
Types in the used elasticsearch entries are enforced:

```
    ip = Ip
    coords = GeoPoint
    learning = Boolean
    total_processed = Integer
    total_blocked = Integer
    blocked = Boolean
    cscore0 = Keyword
    score0 = Integer
    zone = Keyword
    id = Integer
    var_name = Keyword
    date = Date
    whitelisted = Boolean
    uri = Text
    server = Text
    comments = Text
    vers = Text
```
First term is the key used in NAXSI_FMT and second term is the defined ElasticSearch type. Text is used as a backward
compatible version of Keyword. We may drop the support of old elasticsearch version in the near future and replace
Text with Keyword.

It is noteworthy that one request might violate multiple core rule and lead to multiple entries in ElasticSearch.
