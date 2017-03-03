---
layout: post
title: "How to Detect your Tableau Server's REST API Version"
date: 2017-3-3
tags: [Tableau Server, APIs]
header:
  teaser: /assets/images/api_version.png
---

<p align="center">
<img src="https://viziblydiffrnt.github.io/assets/images/api_version.png">
</p>

With each release Tableau continues to improve the functionality of their REST API. This week we saw the release of Tableau 10.2 and with that came some enhancements that make working with the API easier: 

* The ability to specify the fields you want to receive from a GET request
* CORS and JSON support for Javascript and web development
* The ability to download high-resolution images of views (previously only available through tabcmd)

Of course these are only available with the latest version of the REST API (2.5 as of the release of 10.2). So what if you're running a different version of Tableau Server? How do you know which version of the API your server supports?

## Detecting the version of REST API supported by your Tableau Server

Starting with Tableau 10.1 (API version 2.4), Tableau included the endpoint ***ServerInfo*** that contains the version of the API supported by your Tableau Server in its response. It looks like this:

```
GET /api/api-version/serverinfo
```

and the response looks like this:

```xml
<tsResponse>
    <serverInfo>
        <productVersion build="build-number">product-version</productVersion>
        <restApiVersion>api-version</restApiVersion>
    </serverInfo>
</tsResponse>
```

Pretty great, right? 

If you look closely, you'll see that the call requires you to put the *api-version* in the URL. But wait...isn't that what we're trying to figure out? How can you input the API version when you don't know what it is? In fact, how can you tell what version of Tableau Server you're sending calls to?

By using the Vizportal API, that's how. 

## Detecting your Tableau Server and API version using the Vizportal API

In most cases you'll know what version of Tableau Server you're using and can use that to determine your API Version with the table below:

| Tableau Server version | REST API version | Schema version |
|---|---|---|
| 8.3, 9.0  | 2.0  | 2.0  |
| 9.0.1 and later versions of 9.0  | 2.0  | 2.0.1  |
| 9.1  | 2.0  | 2.0.1  |
| 9.2  | 2.1  | 2.1  |
| 9.3  | 2.2  | 2.2  |
| 10.0  | 2.3  | 2.3  |
| 10.1  | 2.4  | 2.4  |
| 10.2  | 2.5  | 2.5  |

But what if you're designing something that might be open-sourced to the Tableau Community or at a client site and you don't want to hard-code those values in? With a quick call to the Vizportal API and some JSON parsing you can determine what version of Tableau Server you're connecting to and map that its API Version. The code below builds on a [previous post](https://viziblydiffrnt.github.io/blog/2016/12/14/tableaus-undocumented-api-made-easy-with-python) where I showed how you can log into the Vizportal API. We'll start where that post left off.

```python

# The information we want (Tableau Server major and minor version) is contained in the response from this call
# Re-use the xsrf_token you captured from the Vizportal login

def getSessionInfo(xsrf_token):
    payload = "{\"method\":\"getSessionInfo\",\"params\":{}}"
    endpoint = "getSessionInfo"
    url = tab_server_url + "/vizportal/api/web/v1/"+endpoint
    headers = {
    'content-type': "application/json;charset=UTF-8",
    'accept': "application/json, text/plain, */*",
    'cache-control': "no-cache",
    'X-XSRF-TOKEN':xsrf_token
    }
    SessionResponse = session.post(url, data=payload, headers=headers)
    return SessionResponse

# We need to parse the JSON response from the getSessionInfo call
# and extract the major and minor version numbers

def DetectTableauVersion():
    global xsrf_token
    TabServerSession = getSessionInfo(xsrf_token).text
    TSS = json.loads(TabServerSession)
    v = TSS['result']['server']['version']['externalVersion']
    major = v['major']
    minor = v['minor']
    patch = v['patch']
    tsVersion = major+'.'+minor
    api_version = None

# Hit this endpoint to access a JSON version of the lookup table above
# Note: this endpoint is not provided by Tableau and might not be hosted in the future

    api_versions = requests.get('https://tbevdbgwch.execute-api.us-west-2.amazonaws.com/Production/versions/')
    api_lookup = json.loads(api_versions.text)
    for k,v in api_lookup['server'].iteritems():
        if k == tsVersion:
            api_version = v
    return tsVersion, api_version
    
tsVersion, api_version = DetectTableauVersion()

print "Tableau Server Version: %s" % (tsVersion)
print "API Version: %s" % (api_version)

```

Success! We now know what version of Tableau Server we're using and what version of the API to include in any REST API calls. It's a little clunky but it works if you're unsure of your server's version or if you're using a version prior to 10.1.
