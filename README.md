# Objective

List of items that are typically required to be exported to external systems for compliance.

# List of Event Attendees

For this we will be using the [report.getTable](https://developer.kaltura.com/api-docs/service/report/action/getTable) API call, with the `TOP_USERS_WEBCAST` as the report type.

The following code retrieves the list of attendees for the live entry ID 1_nqhohx5m.

```python
report_type = KalturaReportType.TOP_USERS_WEBCAST
report_input_filter = KalturaReportInputFilter()
report_input_filter.fromDate = 1406910480
report_input_filter.toDate = 1691425740
report_input_filter.entryIdIn = "1_nqhohx5m"
pager = KalturaFilterPager()
pager.pageIndex = 1
pager.pageSize = 99

result = client.report.getTable(report_type, report_input_filter, pager, "", "", KalturaReportResponseOptions())
print(result.header)
for row in result.data.split(';'):
  print(row)
```

The result in this case is:

```
user_id,user_name,registered,count_loads,count_plays,sum_view_period,sum_live_view_period,avg_live_buffer_time,total_completion_rate,live_engaged_users_play_time_ratio
ron.raz.inc@gmail.com,Foo Bar,0,5,5,45.833333333333,45.833333333333,2.1454545584592E-5,0,0
ron.raz@kaltura.com,ron.raz@kaltura.com,0,3,2,8.6666666666667,8.6666666666667,0,0,0.019230769230769
```

# List of Hosts and Moderators

For this we will be using the [media.get](https://developer.kaltura.com/api-docs/service/media/action/get) API call, with the live entry ID as the parameter.

The following code retrieves the list of hosts (co-editors and co-publishers) and moderators.
Note that for live entries, moderators are listed in the co-viewing collaboration field.

```python
result = client.media.get('1_nqhohx5m', -1)
print('Co-Editors:',result.entitledUsersEdit)
print('Co-Publishers:',result.entitledUsersPublish)
print('Moderators:',result.entitledUsersView)
```

The result in this case is a list of groups (Producers, and WebcastingAdmin) and users:

```
Co-Editors: Producers,ron.raz@kaltura.com
Co-Publishers: Producers
Moderators: Producers,WebcastingAdmin,hari.seldon@foundation.org
```

# List of Presenters

Note that the list of presenters is only for display purposes.
The users lists do not have any functional role in the production of the event.

## Step 1. Get metadataProfile.id

The presenters are stored as custom metadata of the profile "EntryAdditionalInfo".
To get the corresponding metadataProfile.id, we will use [metadataProfile.list](https://developer.kaltura.com/api-docs/service/metadataProfile/action/list) API call, with "EntryAdditionalInfo" as the filter parameter:

```python
filter = KalturaMetadataProfileFilter()
filter.systemNameEqual = "EntryAdditionalInfo"
result = client.metadata.metadataProfile.list(filter, KalturaFilterPager())
print(result.getObjects()[0].id)
```

The result in this case (your ID will probably be different) is:

```
7490081
```

## Step 2. Get Presenters XML

We will use the [metadata.list](https://developer.kaltura.com/api-docs/service/metadata/action/list) API call, to retrieve the Presenters XML using the metadataProfile.id from Step 1 and the live entry ID:

```python
filter = KalturaMetadataFilter()
filter.metadataObjectTypeEqual = KalturaMetadataObjectType.ENTRY
filter.objectIdEqual = "1_nqhohx5m"
filter.metadataProfileIdEqual = 7490081

result = client.metadata.metadata.list(filter, KalturaFilterPager())
print(result.getObjects()[0].xml)
```

In this case, the result is:

```xml
<?xml version="1.0"?>
<metadata>
...
	<Detail>
		<Key>presenters</Key>
		<Value>["presenter_ron.raz@kaltura.com","presenter_ron.raz.inc@gmail.com"]</Value>
	</Detail>
</metadata>
```

# Live Start and End Times

To get the real date and time in which the live event started, we will use the [media.get](https://developer.kaltura.com/api-docs/service/media/action/get) API call with the live entry ID as the parameter.

Assuming that the encoder streams the live broadcast in one session, we will use `firstBroadcast` for the live start time, and `lastBroadcastEndTime` for the live end time:

```python
result = client.media.get('1_5ri75c3b', -1)
print('firstBroadcast:',datetime.fromtimestamp(result.firstBroadcast))
print('lastBroadcastEndTime:',datetime.fromtimestamp(result.lastBroadcastEndTime))
```

In this case the result is:

```
firstBroadcast: 2023-07-06 02:04:49
lastBroadcastEndTime: 2023-07-06 02:09:54
```

# Get Captions from Events

Captions from a live event are available when the recording is ready. To get the captions from events, we will use the [media.list](https://developer.kaltura.com/api-docs/service/media/action/list) API call, with the filter set for `mediaTypeEqual = KalturaMediaType.LIVE_STREAM_FLASH` to select only live event entries, as well as `isRecordedEntryIdEmpty = KalturaNullableBoolean.FALSE_VALUE` to make sure we're getting only live entries with recordings.

When we iterate over the live entries, we will use the [captionAsset.list](https://developer.kaltura.com/api-docs/service/captionAsset/action/list) to get a list of all the captions for the recording, and [captionAsset.getUrl](https://developer.kaltura.com/api-docs/service/captionAsset/action/getUrl) to get the contents for each captions file.

```python
entryFilter = KalturaLiveEntryFilter()
entryFilter.mediaTypeEqual = KalturaMediaType.LIVE_STREAM_FLASH
entryFilter.isRecordedEntryIdEmpty = KalturaNullableBoolean.FALSE_VALUE
entryFilter.orderBy = KalturaLiveEntryOrderBy.CREATED_AT_DESC

captionFilter = KalturaAssetFilter()

result = client.media.list(entryFilter, KalturaFilterPager())
for liveEntry in result.getObjects()[:5]:
  print('\nLive Entry ID=',liveEntry.id,'Recording ID=',liveEntry.recordedEntryId)
  captionFilter.entryIdEqual = liveEntry.recordedEntryId
  captions = client.caption.captionAsset.list(captionFilter, KalturaFilterPager())
  print('There are',captions.totalCount,'captions for this event')
  for caption in captions.getObjects():
    print('Language=',caption.label,',',caption.fileExt)
    link = client.caption.captionAsset.getUrl(caption.id, 0)
    print(link)
```

In my case, the result is:

```
Live Entry ID= 1_al1jskn2 Recording ID= 1_5clqx976
There are 2 captions for this event
Language= English , srt
https://cfvod.kaltura.com/api_v3/index.php/service/caption_captionAsset/action/serve/captionAssetId/1_634tgh60/v/11/ks/djJ8NTI1MDc5Mnw8Pb0MNu6h4TKxbMV4QSsgYfVun0_OUolWQz_e9x3Zr5fWNL9wPlWmGrmqI8qpb8SvmRfzz-qojWknhhgQfxRVwKJEtjGJ-RNNO0rFo-zICNIXTdl-aEzDxJfAOtUWnJNwzSE4AJ4WexIU9NdDjtGEc_HP9uCBdFXLSKTniuYbWAAA4PdQXq_AfUwscOluECixqflrtpuBhyDGc_TbH5T3
Language= French , srt
https://cfvod.kaltura.com/api_v3/index.php/service/caption_captionAsset/action/serve/captionAssetId/1_hexraxk5/v/11/ks/djJ8NTI1MDc5Mnx-AkUJZq2o53RTY3Hi-mvmAdMNf6pn69GFn4po_T21QlbGElgxe6re6iNZtzm8VxlMM8hKcdEGSIHMG5F-1P2DFQsROCj5PQLLmvAHoYv4qqX-DCf_KpemxRj5j48qpPj6AxbHPy1u9FyIBRp2rwPP32-28MofYIszgdnoCcKIhK-PRueO4Hg2oyrmhgHKUO4ycgiOS0NeavbugS00IVfd

Live Entry ID= 1_cppqjsge Recording ID= 1_34keid6e
There are 3 captions for this event
Language= English , srt
https://cfvod.kaltura.com/api_v3/index.php/service/caption_captionAsset/action/serve/captionAssetId/1_jez7vpvj/v/11/ks/djJ8NTI1MDc5MnwMbKb6M7dtPYrE4uYu90eISnA4HXm9JXiUfHzer9X67Bz3ptdUmUf4aY6og-0uVewfjo_2bxGylD_Y2NhJNnJJ9zKQbxpThEe6rW9T4ieRyJGmDuZVX3CYVuPzGBlqwQmbet69pYkaZe2GPDXQshUlG_9iiZ86g44BUj_aDCgzplQ8reTUhxPvd9o1tUZe5LbitU99OqdrAHKG_LmkGNYB
Language= Spanish , srt
https://cfvod.kaltura.com/api_v3/index.php/service/caption_captionAsset/action/serve/captionAssetId/1_yzypf36x/v/11/ks/djJ8NTI1MDc5Mnw8oP-YIt0fIpUdMcDuLiYQofx6GdG-e0-VBVDE2tUpzfIa9dDTrZ8r04NY5KYGjFvqmNaApr2P1GBhRni4S376E7tXES43iDfWp8xF9MgNpKU2kGGjUrSWzLa8t8MwjhVIJ0pT8nd7Zk5Ta4fNRGubhGx0Mkxqgk07VlxWVxSOdWDCrZgVOF7i0OCqEV9cN8Nb0wkz3M7n-evm6idBpu8F
Language= English , srt
https://cfvod.kaltura.com/api_v3/index.php/service/caption_captionAsset/action/serve/captionAssetId/1_yi20fvtw/ks/djJ8NTI1MDc5MnxRMliNG3Bio9mHLBZt4-YrBAEAhRG9fAn3eMZ9nVYqMZZIkA2obnnwrr3Afxbgx29yMYh-UjDqEk7UAhlEviYxMwyvKzXdvrwBTqdKTfHR9ckhEYJ4InomrlSicCstS2JDx2h5AAWjiAxXQvHUGD5sSaxIaNXKbhtw8J47B2Puc1Y6nY9R-KHDaL1nn-WGoasGjSll1PWlbknAVsR_NUrs
```

# Get Events List

Webcasting events are media entries, with specific metadata and custom metadata that sets them apart. First, we will get the custom metadata profile ID for the Start and End times for the event using [metadataProfile.list](https://developer.kaltura.com/api-docs/service/metadataProfile/action/list), and then we will use the [media.list](https://developer.kaltura.com/api-docs/service/media/action/list) API call to get a list of the events, using a filter to indicate that the media entries should be of source type `LIVE_STREAM` and with `adminTagsLike = "kms-webcast-event"` to list only events that were created using Kaltura MediaSpace.

```python
# find metadataProfile ID which holds the StartTime and EndTime for the event 

filter = KalturaMetadataProfileFilter()
filter.systemNameEqual = "KMS_EVENTS3"
pager = KalturaFilterPager()

result = client.metadata.metadataProfile.list(filter, pager)
metadataProfileId= result.getObjects()[0].id

# set the metadata filter

metadataFilter = KalturaMetadataFilter()
metadataFilter.metadataProfileIdEqual = metadataProfileId

# loop on events and print the details including the metadata

eventColsToPrint=['id','name','description','userId','creatorId','tags','createdAt','updatedAt']

filter = KalturaLiveEntryFilter()
filter.orderBy = KalturaLiveEntryOrderBy.CREATED_AT_DESC
filter.sourceTypeEqual = KalturaSourceType.LIVE_STREAM
filter.adminTagsLike = "kms-webcast-event"

result = client.media.list(filter, KalturaFilterPager())
for event in result.getObjects()[:2]:
  for col in eventColsToPrint:
    print(col,':',getattr(event,col))

  metadataFilter.objectIdEqual = event.id
  metadataResults = client.metadata.metadata.list(metadataFilter, KalturaFilterPager())
  print('startEndTimes :',metadataResults.getObjects()[0].xml.split('\n')[1],'\n')
```

In my case, the result is:

```
id : 1_k4delxma
name : CEO Townhall
description : Quarterly updates by the executive team.
userId : ron.raz@kaltura.com
creatorId : ron.raz@kaltura.com
tags : internal, ceo, executive
createdAt : 1704409105
updatedAt : 1704420802
startEndTimes : <metadata><StartTime>1704466800</StartTime><EndTime>1704470400</EndTime><Timezone>US/Eastern</Timezone></metadata> 

id : 1_004xlzin
name : HR updates
description : Updates from Jane Smith. Chief People Officer
userId : ron.raz@kaltura.com
creatorId : ron.raz@kaltura.com
tags : internal, hr dept
createdAt : 1704408940
updatedAt : 1704420672
startEndTimes : <metadata><StartTime>1704405600</StartTime><EndTime>1704409200</EndTime><Timezone>US/Eastern</Timezone></metadata> 
```

Note, that I listed the two most recent events for demo purposes.
You can use filters such as the following to filter by creation dates:
```python
filter.createdAtGreaterThanOrEqual = 1704408940
filter.createdAtLessThanOrEqual = 1704420672
```

