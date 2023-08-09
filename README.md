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

# LoB Sponsor and Details

**TBD**

Define custom metadata for LoB Sponsor and Details and query it in the same method as above.

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
