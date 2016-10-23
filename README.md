# BTrDB REST API examples

BTrDB includes a REST api that can be used from any language supporting
HTTP. This guide will show you how to do some common operations using
`curl`. Bear in mind that the binary API is faster, and has golang
and python support.


## Checking status

To verify that the server is up, and the firewall is correctly configured,
you can use the `status` endpoint.

```bash
# edit this for your server
SERVER=http://127.0.0.1:9000
curl $SERVER/status
```

you should see `OK`. If you see an interaction like the following,
your server is either on a different port or your firewall is incorrectly configured.

```bash
$ SERVER=http://127.0.0.1:9000
$ curl $SERVER/status
curl: (7) Failed to connect to 127.0.0.1 port 9000: Connection refused
```

You can use this endpoint in health monitoring services to verify
that the server is up. The response is always `OK`.

## Querying data

BTrDB stores data in streams identified by UUID. You can find the UUID
of a stream you wish to query by looking at its metadata in the plotter.

Once you have the UUID you can either do a "raw" query which gets the
data at its native resolution, or a "statistical" query which obtains
coarser resolution data but is substantially faster.

Lets begin by getting a rough overview of what data exists in a stream,
using a 'bracket' query

```bash
curl -X POST $SERVER/q/bracket -d '{
  "UUIDS": [ ... ]
}'
```

The bracket query accepts a list of UUIDs and returns the interval of
time that contains data for each of them, as well as the union of all
the different ranges (this is used by the plotter to work out how to
autoscale the X axis). I use the `jq` command to pretty-print the returned
JSON:
```bash
$ SERVER=http://127.0.0.1:9000
$ curl -X POST $SERVER/q/bracket -d '{
  "UUIDS": ["f0896731-0e43-4ed5-a97f-19266d8e732c"]
}' | jq
```
And we get:
```json
{
  "Brackets": [
    [
      1477100329040881700,
      1477186729034743600
    ]
  ],
  "Merged": [
    1477100329040881700,
    1477186729034743600
  ]
}
```

To get an idea of how much data is present, you can use iterative statistical
queriesm starting at a coarse level and going finer. Before going into statistical
queries, it is best to explain *pointwidth*. Many queries take a point width or `pw`
parameter. This is a way of describing a resolution. It is equal to log base 2 of
a time interval in nanoseconds. For example pw=23 is 2^23 nanoseconds, or
8.388 ms. pw=34 is 2^34 nanoseconds or 17.17 seconds. Here are some useful
pointwidths for your reference:


| PW   | Rough equivalent  | Actual value      |
| ---- |------------------ | -----------------:|
| 23   | 8.388 ms          |    16,777,216     |
| 30   | 1 second          | 1,073,741,824     |
| 36   | 1 minute          | 68,719,476,736    |
| 42   | 1 hour (73 min)   | 4,398,046,511,104 |
| 46   | 1 day (19.5 hour) | 70,368,744,177,664 |
| 49   | 1 week (6.5 days) | 562,949,953,421,312 |
| 51   | 1 month (26 days) | 2,251,799,813,685,248 |
| 54   | 1 year (208 days) | 18,014,398,509,481,984 |

BTrDB does support exact windowing, although not through the REST API. The
reason you would want to use pointwidths for your windows is that they are
extremely fast, querying for pw=42 windows is significantly faster than
querying for exactly 60.000 minute windows using the WINDOW command through
the binary interface.

Now we can do a sequency of statistical queries over our example stream
to get an idea of how the data is distributed:

```bash
STIME=1477100329040881700 #unix time in nanoseconds
ETIME=1477186729034743600
PW=46 #days
PARAMS="starttime=$STIME&endtime=$ETIME&unitoftime=ns&pw=$PW"
UUID=f0896731-0e43-4ed5-a97f-19266d8e732c
curl -X GET "$SERVER/data/uuid/$UUID?$PARAMS" | jq
```

and we get a response like:

```json
[
  {
    "uuid": "f0896731-0e43-4ed5-a97f-19266d8e732c",
    "XReadings": [
      [
        1477039940289,
        167360,
        95915605781875420,
        95915929807604350,
        95916253833333300,
        1197648
      ],
      [
        1477110309033,
        345024,
        95916253833874380,
        95918538533067200,
        95920823232259970,
        8444587
      ]
    ],
    "version": 2
  }
]
```

Here we have two records, so the data in this stream only covers roughly two days.
Each record contains five fields:

 - timestamp in milliseconds since the epoch
 - the remainder of the timestamp in nanoseconds
 - the minimum value
 - the mean value
 - the maximum value
 - the number of values

 `jq` can be used to extract parts of this data, so for example to get only
 the density of the data you can append:

 ```bash
 curl ...(as above) | jq .[0].XReadings[][4]
 ```

 Which will return only the 5th field for every reading (the count).

 You can also perform statistical queries and obtain the result as a CSV
 file. This endpoint is a little easier to use as it accepts JSON parameters
 rather than URL parameters:

 ```bash
 SERVER=http://127.0.0.1:9000
 curl -X POST $SERVER/directcsv -d '
{
   "UUIDS": ["f0896731-0e43-4ed5-a97f-19266d8e732c"],
   "Labels": ["C1MAG"],
   "StartTime": 1477100329040881646,
   "EndTime": 1477116729040881997,
   "UnitofTime": "ns",
   "PointWidth": 42
}' > file.csv
```

Here, the fields are named for you:

```bash
$ head file.csv
Time[ns],Time,C1MAG(cnt),C1MAG(min),C1MAG(mean),C1MAG(max)
1477097114893811712,2016-10-21 17:45:14.893,142074,95915605781875424.000000,95915644220002320.000000,95915682658129248.000000
1477101512940322816,2016-10-21 18:58:32.940,527787,95915682658670368.000000,95915825452200800.000000,95915968245731280.000000
1477105910986833920,2016-10-21 20:11:50.986,527787,95915968246272384.000000,95916111039802848.000000,95916253833333296.000000
1477110309033345024,2016-10-21 21:25:09.033,527786,95916253833874384.000000,95916396627134288.000000,95916539420394192.000000
...
```

Finally, over small regions of time, you can query the raw data. This looks the
same as the statistical JSON query above, but does not include the pw parameter:
```bash
STIME=1477100329040881700 #unix time in nanoseconds
ETIME=1477186729034743600
PARAMS="starttime=$STIME&endtime=$ETIME&unitoftime=ns"
UUID=f0896731-0e43-4ed5-a97f-19266d8e732c
curl -X GET "$SERVER/data/uuid/$UUID?$PARAMS" > data.json
```

You will notice that even for this short range of time (~2 days) the file
is quite large (430MB). Take care not to perform  raw queries over too much data:

```bash
$ ls -hal data.json
$ -rw-rw-r-- 1 michael michael 434M Oct 23 13:05 data.json
```
