# elastic-recurring-plugin

Allow to work in ES with some features of recurrent dates defined in [rfc2445](https://www.ietf.org/rfc/rfc2445.txt). 
This plugin adds a new type named *recurring* and the native scripts: *nextOccurrence*, *hasOccurrencesAt*, *occurBetween* and *notHasExpired*.

It was tested in ES 6.6.2

[![Build Status](https://travis-ci.org/betorcs/elastic-recurring-plugin.svg?branch=5.0)](https://travis-ci.org/betorcs/elastic-recurring-plugin)

## Getting start

### Compiling and installing plugin
r
Generating zip file, execute command bellow, the file will be created in folder `build\distributions`.

```./gradlew clean build```

To installing plugin in elasticsearch, run this command in your elasticsearch server.

```$bin/elasticsearch-plugin install recurring-plugin-1.0.zip```

## Recurring Type
Mapper type called _recurring_ to support recurrents dates. The declaration looks as follows:
```
{
    "event" : {
        "properties" : {
            "recurrent_date" : {
                "type" : "recurring"
            }
        }
    }
}
```
The above mapping defines a _recurring_, which accepts the follow format:
```
{
    "event" : {
        "recurrent_date" : {
            "start_date" : "2016-12-25",
            "end_date" : null,
            "rrule": "RRULE:FREQ=YEARLY;BYMONTH=12;BYMONTHDAY=25"
        }
    }
}
```

## Native scripts

### nextOccurrence

Script field returns date of next occurrence of event in yyyy-MM-dd when it's recurrent, 
and the `start_date` when start/end is valid yet, to both situation today'll be considered if `from` parameter is omitted.

*Parameters:*
- *field* - Name of property, type must be _recurring_.
- *from* - Optional, date to be considered _from_.

### hasOccurrencesAt

Script field returns `true` if event occurrs in determinated date.

*Parameters:*
- *field* - Name of property, type must be _recurring_.
- *date* - Date 

### occurBetween

Script field returns `true` if event occurrs in determinated range of date.

*Parameters:*  
- *field* - Name of property, type must be _recurring_.
- *start* - Starting date inclusive.
- *end* - Ending date inclusive.

### notHasExpired

Script field returns `true` if event is not expired considering server date.

*Parameters:*  
- *field* - Name of property, type must be _recurring_.

### Samples

## Adding a mapping

PUT `sample/event/_mapping`
```
{
  "settings": {
    "index": {
      "number_of_shards": 1,
      "number_of_replicas": 0
    }
  },
  "mappings": {
    "event": {
      "properties": {
        "name": {
          "type": "text"
        },
        "recurrent_date": {
          "type": "recurring"
        }
      }
    }
  }
}
```

## Adding data

PUT `sample/event/1`
```
{
  "name": "Christmas",
  "recurrent_date": {
    "start_date": "2015-12-25",
    "rrule": "RRULE:FREQ=YEARLY;BYMONTH=12;BYMONTHDAY=25;WKST=SU"
  }
}
```

PUT `sample/event/2`
```
{
  "name": "Mother's day",
  "recurrent_date": {
    "start_date": "2016-05-08",
    "rrule": "RRULE:FREQ=YEARLY;BYMONTH=5;BYDAY=2SU;WKST=SU"
  }
}
```

PUT `sample/event/3`
```
{
  "name": "Halloween",
  "recurrent_date": {
    "start_date": "2012-10-31",
    "rrule": "RRULE:FREQ=YEARLY;BYMONTH=10;BYMONTHDAY=31;WKST=SU"
  }
}
```

PUT `sample/event/4`
```
{
  "name": "5 maintenance monthly of cruze ",
  "recurrent_date": {
    "start_date": "2016-03-10",
    "rrule": "RRULE:FREQ=MONTHLY;BYMONTHDAY=10;COUNT=5;WKST=SU"
  }
}
```

## Using scripts

### nextOccurrence

POST `sample/event/_search`
```
{
  "query" : {
    "match_all": {}
  },
  "script_fields": {
    "nextOccur": {
      "script": {
        "source": "nextOccurrence",
        "lang": "native",
        "params": {
          "field": "recurrent_date"
        }
      }
    }
  }
}
```
RESPONSE
```
{
  "took": 9,
  "timed_out": false,
  "_shards": {
    "total": 1,
    "successful": 1,
    "failed": 0
  },
  "hits": {
    "total": 4,
    "max_score": 1,
    "hits": [
      {
        "_index": "sample",
        "_type": "event",
        "_id": "3",
        "_score": 1,
        "fields": {
          "nextOccur": [
            "2017-10-31"
          ]
        }
      },
      {
        "_index": "sample",
        "_type": "event",
        "_id": "2",
        "_score": 1,
        "fields": {
          "nextOccur": [
            "2017-05-14"
          ]
        }
      },
      ...
    ]
  }
}
```

### hasOccurrencesAt

POST `sample/event/_search`
```
{
  "query": {
    "bool": {
      "filter": {
        "script": {
          "script": {
            "source": "hasOccurrencesAt",
            "lang": "native",
            "params": {
              "field": "recurrent_date",
              "date": "2015-01-31"
            }
          }
        }
      }
    }
  }
}
``` 
RESPONSE
```
{
  "took": 8,
  "timed_out": false,
  "_shards": {
    "total": 1,
    "successful": 1,
    "failed": 0
  },
  "hits": {
    "total": 1,
    "max_score": 0,
    "hits": [
      {
        "_index": "sample",
        "_type": "event",
        "_id": "2",
        "_score": 0,
        "_source": {
          "name": "Mother's day",
          "recurrent_date": {
            "start_date": "2016-05-08",
            "rrule": "RRULE:FREQ=YEARLY;BYMONTH=5;BYDAY=2SU;WKST=SU"
          }
        }
      }
    ]
  }
}
```


### occurBetween

POST `sample/event/_search`
```
{
  "query": {
    "bool": {
      "filter": {
        "script": {
          "script": {
            "source": "occurBetween",
            "lang": "native",
            "params": {
              "field": "recurrent_date",
              "start": "2016-01-31",
			  "end": "2016-07-26"
            }
          }
        }
      }
    }
  }
}
``` 
RESPONSE
```
{
  "took": 8,
  "timed_out": false,
  "_shards": {
    "total": 1,
    "successful": 1,
    "failed": 0
  },
  "hits": {
    "total": 2,
    "max_score": 0,
    "hits": [
      {
        "_index": "sample",
        "_type": "event",
        "_id": "2",
        "_score": 0,
        "_source": {
          "name": "Mother's day",
          "recurrent_date": {
            "start_date": "2016-05-08",
            "rrule": "RRULE:FREQ=YEARLY;BYMONTH=5;BYDAY=2SU;WKST=SU"
          }
        }
      },
      {
        "_index": "sample",
        "_type": "event",
        "_id": "4",
        "_score": 0,
        "_source": {
          "name": "5 maintenance monthly of cruze ",
          "recurrent_date": {
            "start_date": "2016-03-10",
            "rrule": "RRULE:FREQ=MONTHLY;BYMONTHDAY=10;COUNT=5;WKST=SU"
          }
        }
      }
    ]
  }
}
```

### notHasExpired

POST `sample/event/_search`
```
{
  "query": {
    "bool": {
      "filter": {
        "script": {
          "script": {
            "source": "notHasExpired",
            "lang": "native",
            "params": {
              "field": "recurrent_date"
            }
          }
        }
      }
    }
  }
}
``` 
RESPONSE
```
{
  "took": 9,
  "timed_out": false,
  "_shards": {
    "total": 1,
    "successful": 1,
    "failed": 0
  },
  "hits": {
    "total": 3,
    "max_score": 0,
    "hits": [
      {
        "_index": "sample",
        "_type": "event",
        "_id": "3",
        "_score": 0,
        "_source": {
          "name": "Halloween",
          "recurrent_date": {
            "start_date": "2012-10-31",
            "rrule": "RRULE:FREQ=YEARLY;BYMONTH=10;BYMONTHDAY=31;WKST=SU"
          }
        }
      },
      {
        "_index": "sample",
        "_type": "event",
        "_id": "2",
        "_score": 0,
        "_source": {
          "name": "Mother's day",
          "recurrent_date": {
            "start_date": "2016-05-08",
            "rrule": "RRULE:FREQ=YEARLY;BYMONTH=5;BYDAY=2SU;WKST=SU"
          }
        }
      },
      {
        "_index": "sample",
        "_type": "event",
        "_id": "1",
        "_score": 0,
        "_source": {
          "name": "Christmas",
          "recurrent_date": {
            "start_date": "2015-12-25",
            "rrule": "RRULE:FREQ=YEARLY;BYMONTH=12;BYMONTHDAY=25;WKST=SU"
          }
        }
      }
    ]
  }
}
```