+++
date = "2018-02-23T11:09:58+01:00"
draft = false
title = "Using griblib for Meteorology"
tags        = [ "go", "grib2", "griblib", "weather"]
+++


Griblib is a library written in go to read grib2 files and do some filtering on the content. This post describes some details about how griblib is built and intended usage. 


## GRIB2: a binary format for meteorology

GRIB: __G__ eneral __R__ egularly-distributed __I__ nformation in __B__ inary form

Grib2 is a binary file-format used to store forecasts and historical data. The content can be meteorological, space-weather, sea-conditions etc. In this post I will only focus on meteorological data. It is the defacto standard for distributing meteoroligical data, so If you want to read the raw data you probably want to be able to read grib2-files. 

Grib2 files from http://www.noaa.gov/ are distributed freely from the 
[nooa file server](http://www.ftp.ncep.noaa.gov/data/nccf/com/gfs/prod/) 
where the most recent forecasts can be downloaded. Documentation for grib2 can 
be found several places, 
but [noaas docs](http://www.nco.ncep.noaa.gov/pmb/docs/grib2/grib2_doc.shtml) 
is a good place to start if you need to look up some specific documentation. 

## Data in grib2 files

Each grib2 file contains 1 or more _messages_ where each message contains some meteorology data in 2d for the whole planet at a specified time. The data can be temperature, wind, moist, pressure at different surfaces. For instance, a normal grib2 file from nooa contains 11 messages with temperature for 11 unique layers, meters over sea-level(isobaric surface).

The message is built of 8 unique sections, where `Section0-6` is meta-data, `Section7` contains the actual data and `Section8` is the EOM section. 

The data is represented as a 1-dimensional array in `Section7` of the message. The data format is raw, meaning that it does often not make sense to look at the raw numbers in `Section7` without scaling or adjusting based on the information from `Section0-6`.

# griblib

support for:

* reading grib files
* reducing files
   * by category and discipline
   * by layer-type and layer-value
   * by geographical area

## why go

Most of the libraries today are written in fortran or c and are not quite readable. You will have a hard time trying to extend
and modify those libraries unless you spend a significant amount of time trying to understand those libraries. 

One of the purposes of griblib is to have a simple library that is easy to extend and modify, written in a language that lets 
you focus on the data in the grib-files instead of memory-allocations, verbose error-handling etc. 

### Read binary content to structs from io.Reader

It turns out that dealing with binary file formats in go is quite pleasant and straight-forward when the 
specification of the file-format is as well documented as the grib2 format. For instance, Section0 is specified as follows:

| Octet number  | Content   |
|-------        |-----------| 
|  1-4          |  'GRIB' (Coded according to the International Alphabet Number 5) |  
|  5-6          |  Reserved |  
|  7            |  Discipline (From Table 0.0) |  
|  8            |  Edition number - 2 for GRIB2 |  
|  9-16         |  Total length of GRIB message in octets (All sections); |  


This means that

 * the total length of Section0 is 16 bytes.
 * byte 1-4 always contains a constant, representing 'GRIB' .
 * byte 7 indicates the discipline of the message(meteorological, hydrological, space, land, oceanic). 
 * byte 9-16 contains a number that specifies the total length in bytes of this message. 

To read Section0 with golang we need to make sure the `struct` for Section0 fits with this size and structure. 

    type Section0 struct {
        Indicator     uint32 // uint32 is 4 bytes
        Reserved      uint16 // 2 bytes
        Discipline    uint8  // 1 byte
        Edition       uint8  // 1 byte
        MessageLength uint64 // 8 bytes
    }

Note that we are not using `int`, but rather the types `uint32`,`uint8`,`uint16` and `uint64` as data-types since I need to consider the byte-length of each type to match the binary content. 
When we have a struct that matches the size and content of the section, we can 
use the `binary.Read` to read from an `io.Reader` by passing a pointer to a struct, the reader and specifying endianness:


    func ReadSection0(reader io.Reader) (section0 Section0, err error) {
        section0 = Section0{}
        err = binary.Read(reader, binary.BigEndian, &section0)
        
Plain and simple like that, binary read will read the bytes from the reader and create `Section0` from the bytes. To get this right, it is crucial 
that the data-fields in the `Section0` struct is aligned with the bytes returned from the reader. 

The `io.Reader` interface is implemented by several input sources like `os.File` and the `Body` field in `http.Response`. Using the 
interface `io.Reader` therefore allows us to use both files and http-responses (and whatever implements `io.Reader`) as a source to read grib-messages.

## focus, constraints

The library supports the common templates used for meteorology, and only complex packating and spatial differencing. Griblib has currently several constraints, some of them are listed here: 

* Only Category 0 (Meteorology) is supported
* Only Grid0 is supported(section3) when using filtering. The other grids are supported for reading.
* Only Product0 (analysis or forecast at a horizontal level or in a horizontal layer at a point in time) in Section4 is supported.
* Only DataTemplate3(complex packing and spatial differencing) is supported in section5. 

# using griblib

To use griblib you can either use it as an application without modifying it, simply `go get github.com/nilsmagnus/grib` and start using it:

    $ grib -h
    Usage of grib:
      -category int
        	Filters on Category within discipline. -1 means all categories (default -1)
      -dataExport
        	Export data values. (default true)
      -discipline int
        	Filters on Discipline. -1 means all disciplines (default -1)
      -export int
        	Export format. Valid types are 0 (none) 1(print discipline names) 2(print categories) 3(json) 
      -file string
        	Grib filepath
      -latMax int
        	Maximum latitude multiplied with 100000. (default 36000000)
      -latMin int
        	Minimum latitude multiplied with 100000.
      -longMax int
        	Maximum longitude multiplied with 100000. (default 9000000)
      -longMin int
        	Minimum longitude multiplied with 100000. (default -9000000)
      -maxmsg int
        	Maximum number of messages to parse. Does not work in combination with filters. (default 2147483647)
      -operation string
        	Operation. Valid values: 'parse', 'reduce'. (default "parse")
      -reducefile string
        	Destination for reduced file. (default "reduced.grib2")


For advanced usage, use it as a library. Add it to your GOPATH and start hacking:

     	file, _ := os.Open("some.grib.file.003")
     
     	messages, err := griblib.ReadMessages(file)
     	
     	filtered := griblib.Filter(messages, griblib.Options{ ... } )
     	
     	for _, message := range filtered {
     	    // your processing here
     	}


# future of griblib

I am currently using griblib to parse grib2-files from ncep, with focus on forecasts. I am aware of several bugs in 
the library, and am working on fixes when I have time. One of the key areas I will probably focus on in the future is to 
support more packaging types, especially complex packaging without spatial differencing. 

If you happen to have any ideas to improve it, please leave an issue on github with suggestions. Even better: send me a pull-request 
if with fixes and support for un-implemented features(packaging, categories etc. ).

https://github.com/nilsmagnus/grib