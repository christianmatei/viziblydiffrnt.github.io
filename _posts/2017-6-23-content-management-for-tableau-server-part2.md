---
layout: post
title: "Taking Tableau Further: Create Your own Content Management Tool, Part 2"
date: 2017-06-23
tags: [Tableau Server, APIs, Content Management]
header:
  teaser: /assets/images/lego_question.jpg
---

<p align="center">
<img src="https://viziblydiffrnt.github.io/assets/images/lego_question.jpg">
</p>

# Collecting the Building Blocks

In the [first part](https://viziblydiffrnt.github.io/blog/2017/06/13/content-management-for-tableau-server) of this series I talked about how we can build our own Content Management solution using Tableau's APIs to add some functionality that will add life to your server. In this entry I'll go into detail about which data points we need to retrieve from Tableau's Postgres DB so we can build out our automation. I'll also present some methods for constructing the code but won't provide a full solution as you may have goals that differ from mine. What I want to leave you with are the concepts that can be applied more broadly.

## Setting up the Config

I chose to design my tool using Python so we'll use that for the examples here but you can use any language so long as there is a library that can connect to Postgres. 

To save time and not be redundant, I'm going to use a central configuration file to manage all of the settings for our tool. I like using `yaml` for this because it's really clean and easy to modify and works well with Python.

```yaml

tab_server_settings:
	days_since_viewed: 180 #Setting the default to 180 days (6 months)
	days_since_accessed: 180
	tab_server_url: 'http://YourTableauServerURL'
	run_mode: 'live' #Could also be demo
	tableau_username: AdminUserName
	tableau_user_pw: AdminPassword
	remove_disabled_content: False
	
postgres_settings:
	tab_server_pg_db: 'tab_server_postgres_hostname'
	pg_user: pg_username #usually this is 'readonly'
	pg_pw: password #you specify this
	pg_database: 'workgroup' #this is the default for Tableau Server 
	pg_port_num: 8060 #default port
	
```

Save the file with `.yml` extension and make sure you have [PyYAML](http://pyyaml.org) installed. You can add more configs as you go but for now let's keep things simple.


## Postgres

*The following assumes that you have access to Tableau Server's Postgres Database.*

Not every data point that we need (id values of extracts and content usage statistics) is available from Tableau's APIs but because every Tableau Server ships with its own Postgres DB, we can go right to the source to get our data. It's best to connect to a copy of the server database and not the live one to avoid impacting your server's performance. I'm using the [psycopg2](http://initd.org/psycopg/) library to connect to the Workgroup database.

### Connect to Postgres

```python
import time
import os, sys
from os import path
import more_itertools
from  more_itertools import unique_everseen
from operator import itemgetter
import psycopg2
import sys
import pprint
import requests
import xml.etree.ElementTree as ET
import urllib
import zipfile
import shutil
import yaml
sys.path.append("..")

# import settings from yaml config

with open("config.yml", 'r') as ymlfile:
	cfg = yaml.load(ymlfile)

tab_server_settings = cfg["tab_server_settings"]
postgres_settings = cfg["postgres_settings"]

# Set variables based on Config File
run_mode = tab_server_settings['run_mode']

days_since_viewed = tab_server_settings['days_since_viewed']
days_since_accessed = tab_server_settings['days_since_accessed']

tab_server_pg_db = postgres_settings['tab_server_pg_db']
pg_user = postgres_settings['pg_user']
pg_pw = postgres_settings['pg_pw']
pg_database = postgres_settings['pg_database']
pg_port_num = postgres_settings['pg_port_num']

def pg_connect():
    """
    Signs in to the Postgres database specified in the global tab_server_pg_db variable.
    Returns a cursor with an open connection for querying tables in the workgroup database.
    """
    # Define our connection string
    global conn_string
    conn_string = "host='%s' dbname='%s' user='%s' password='%s' port='%s'" % (tab_server_pg_db,pg_database,pg_user,pg_pw,pg_port_num)

    # print the connection string we will use to connect
    print "Connecting to database\n	->host:%s, dbname:%s, port:%s" % (tab_server_pg_db, pg_database, pg_port_num)

    # get a connection, if a connect cannot be made an exception will be raised here
    global conn
    try:
        conn = psycopg2.connect(conn_string)
    except Exception as err:
        print "Unexpected error while establishing connection:{}".format(err)
        sys.exit(1)

    # conn.cursor will return a cursor object, you can use this cursor to perform queries
    global cursor
    cursor = conn.cursor()
    print "Connected!\n"

    return cursor

```

Once you open the connection, the `cursor` can be used to issue other commands to the database like this:

```python
cursor.execute("SELECT [fields] from [table]")
data = cursor.fetchall()
```

When you're done getting data from Postgres, close the connection like this:

```python
conn.close()
```

### Get Sites

Because we want to review all of the content on our server, regardless of Site, we'll need to get the details for all Sites. Using the cursor you opened, write a function to query the `sites` table for the following data:

* id: the numerical id value of the Site
* name: the friendly name of the Site
* url_namespace: also known as the Site ID or contentUrl. It is the value that appears after "/site" in the URL on Tableau Server
* luid: the unique identifier of the Site (it is used as the "site_id" for API calls)

```python
def get_pg_sites():
	cursor.execute("select id, name, url_namespace, luid from sites")
	sites = cursor.fetchall()
	global sites_data
	sites_data = []
	
	for row in sites:
		values = [row[0], row[1], row[2] row[3]]
		record = {'site_id':values[0], 'site_name':values[1], 'url_namespace':values[2], 'site_luid':values[3]}
		sites_data.append(record)
	
	return sites_data
```

The goal of the function above is to return a list where each item is a complete database "record", formatted as a dictionary, with the field names as keys and their associated values. Use this function as a *template* for the remaining data to be retrieved, substituing the query and adding fields to the output as necessary.

### List Unused Content

Since the goal of this tool is clean up content, we need to identify which content (Workbooks and Datasources) have become stale.

```sql
select id, name, workbook_url, owner_id, project_id, site_id, site_luid, luid, date_part('day',now()-last_view_time) as days_since_last_view 
from 
(
	select w.id, w.name, w.workbook_url, w.owner_id, w.project_id, w.site_id, wkb.luid, sites.luid as site_luid, max(vs.last_view_time) as last_view_time
	from _workbooks w
	left join _users u on w.owner_id=u.id
	left join _views_stats vs on w.id=vs.views_workbook_id
	join workbooks wkb on w.id=wkb.id
	join sites on sites.id=w.site_id
	where date_part('day',now()-last_view_time) >= %s
	group by w.id, w.name, w.workbook_url, w.owner_id, w.project_id, w.site_id, wkb.luid, site.luid
) a

```

Remember where we set `days_since_viewed` in the yaml configuration? Swapping out `%s` in the query with this variable will allow you to change the yaml as needed and not have to modify this query. As with the Site details before, store the records as a list of key, value pairs.

The same process needs to be done for Datasources but the tables and fields are [different](http://onlinehelp.tableau.com/current/server/en-us/data_dictionary.html#_datasources_anchor).

### Get User Details

Once we know the content we want to examine, we need to know a little bit about the users that published it. Using the prior examples, write another function to retrieve the following:

```sql
select 
	su.friendly_name as user_friendly_name,
	su.name as system_user_name,
	su.email,
	u2.licensing_role_name,
	u2.site_id,
	u.luid as user_luid
from system_users su
join users u on su.id=u.system_user_id
join _user u2 on u.id=u2.id
where u.id = %s
```

This time, `%s` represents a substituted variable for a user_id. In this case, we'll be passing an owner_id that we got from our list of Workbooks and Datasources so we don't scan through all the users on the server.

### Get Server Schedule Details

We'll also need all of the possible schedules that exist on the server. Here's the query:

```sql
select id, luid, name,
case schedule_type 
	when 0 then 'Hourly'
	when 1 then 'Daily'
	when 2 then 'Weekly'
	when 3 then 'Monthly' end as schedule_type
from schedules
```
### Get Content Schedule Details

And last but not least, the schedules for particular pieces of content (since they can often have more than one). This query is pretty large because we need parse the `dayofweekmask` to determine which days during the week a piece of content is scheduled for.

```sql
select workbook_id, workbook_name, task_id, schedule_id, schedule_type, priority, 
day_of_week_mask_array[1] is_sat,
day_of_week_mask_array[2] is_fri,
day_of_week_mask_array[3] is_thu,
day_of_week_mask_array[4] is_wed,
day_of_week_mask_array[5] is_tue,
day_of_week_mask_array[6] is_mon,
day_of_week_mask_array[7] is_sun
from 
(
	select *, regexp_split_to_array(day_of_week_mask_binary,'') as day_of_week_mask_array
	from
	(
		select workbook_id, workbook_name, task_id, schedule_id,
		schedule_name, priority, 
		case schedule_type 
			when 0 then 'Hourly'
			when 1 then 'Daily'
			when 2 then 'Weekly'
			when 3 then 'Monthly' end as schedule_type,
		day_of_week_mask,
		cast(day_of_week_mask::bit(7) as varchar) as day_of_week_mask_binary,
		day_of_month_mask
		from
		(
			select w.id as workbook_id, w.name as workbook_name,
			w.owner_id, w.project_id, w.site_id, t.id as task_id,
			t.schedule_id, t.type, t.luid, s.name as schedule_name,
			s.priority, s.schedule_type, s.day_of_week_mask, s.day_of_month_mask
			from tasks t
			inner join _workbooks w on t.obj_id=w.id
			left join _schedules s on t.schedule_id=s.id
			where w.id = %s
		) a
	) b
) c
```

Again, substitute `%s` for a variable representing the Workbook Id. You'll need to write a similar query for Datasources, substituting the Datasource Id as needed.

## What is All of this Data Used For?

At this point you're probably asking yourself:

> "Why did I just write all of these functions and really long, complex queries?"

> ["What on Earth am I supposed to do with all of this?"](http://go.slalom.com/wehavefunwithdata)

The simple answer is that we now have the ***building blocks*** for the rest of the API calls that we want to make. Let's do a recap of what we know:

* We can now identify Workbooks and Datasources that have not been viewed/accessed for a given number of days.
* We have all of the details needed to loop through the Sites on our server and issue the proper commands via the REST API.
* We know who all of the content owners are and the email addresses associated with their server user names. This will be hepful when we need to send them notifications.
* We have the list of all schedules available on Tableau Server.
* We have the schedule details for any Workbook or Datasource on server, including their frequency of execution.

Using these data points we can start to build our **workflow** and begin managing content based on the settings we supply in our `yaml` configuration.

In my next post I'll cover how we structure the REST API calls and automate the workflow.