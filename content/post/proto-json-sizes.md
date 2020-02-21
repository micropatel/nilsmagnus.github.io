+++
date = "2017-04-09T21:44:01+01:00"
draft = false
title = "Comparing sizes of protobuf vs json"
tags        = ["proto", "json"]
+++

__update__: now comparing gzipped json vs gzipped protobuf 

Google Protobuffer is a binary format claiming to much more compact than json and other text-formats, but just how much less space does it require? Does it hold for large arrays of data?

In this blogpost I will compare the sizes of the two formats. 


## Test-data with array of tickers

I will generate test-data with a home made tool, you can find it on github: https://github.com/nilsmagnus/protobuf-json-xml-size-comparison
     
The content of the data is defined in a proto-message:

    syntax = "proto3";
    
    package sample;
    
    message Test {
      string query = 1;
      int32 page_number = 2;
      int32 result_per_page = 3;
      repeated Ticker tickers = 4;
    }
    
    message Ticker {
      string name = 1;
      float value = 2;
    }
    
There is 1 string, 2 ints and an array of possibly unlimited array of tickers. 

The ticker-name is a random string of size 3 and the value is a random float value between 0.0 and 9.99. 
In json could looks like this with 2 tickers in the array:
    
    {
      "query": "myQuery",
      "page_number": 42,
      "result_per_page": 100,
      "tickers": [
        {
          "name": "rPs",
          "value": 9.768923
        },
        {
          "name": "WEo",
          "value": 6.067048
        }
      ]
    }
    

## The numbers

Comparing the raw-size only of json would be unfair, since it is usually gzipped before transfer between client/servers. 

However, take into account that when gzipping content you are using additional cpu to zip the content which is not needed when using protobuf.

*The size-numbers are in bytes.*

### Raw json

| no of tickers | size raw json | size protobuf | protobuf size(%)|
|---|---|---|---|
|0| 58  | 13 | 22.4 |
|1| 102  | 25 | 24.5 |
|2| 133  | 37 | 27.8 |
|10| 396  | 133 | 33.6 |
|20| 724  | 253 | 34.9 |
|200| 6578  | 2413 | 36.7 |
|2000| 65250  | 24013 | 36.8 |

Protobuf is clearly the winner for all sizes of the ticker-list, but is best when the 
ticker-list is smaller. 

### gzipped json and gzipped protobuf

The common case for transferring json-messages is to gzip them first, so lets see how that affects our numbers. 

| no of tickers | size gzipped json | gzipped protobuf | gzipped protobuf size(%)|
|---|---|---|---|
| 0 |  82 | 42 | 51.21 | 
| 1 |  125 | 54 | 43.20 | 
| 2 |  142 | 64 | 45.07 | 
| 10 |  235 | 137 | 58.29 | 
| 20 |  331 | 230 | 69.48 | 
| 200 |  1970 | 1629 | 82.69 | 
| 2000 |  17539 | 14808 | 84.42 | 
| 20000 |  171154 | 146378 | 85.52 | 

Protobuf clearly wins on the smaller map-sizes, but loses its clear advantage when the ticker-list grows in size.

Protobuf is still the winner on all sizes. 
 

# Conclusion

Protocol buffers is a clear winner for small messages where the protobuf size is as small as 16% of the gzipped json size.
 
However, when large arrays of data is transferred, gzipped protobuf is still smaller but seems to lose its clear advantage in message size to gzipped json.
 

# Read more 

Read more about protobuf in the documentation: https://developers.google.com/protocol-buffers/