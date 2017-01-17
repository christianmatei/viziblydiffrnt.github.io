---
layout: post
title: "Tableau's "Undocumented" API Made Easy with Python"
date: 2016-12-14
tags: [Tableau Server, APIs]
header:
  teaser: /assets/images/undocumented_teaser.jpg
---

<p align="center">
<img src="https://viziblydiffrnt.github.io/assets/images/undocumented_teaser.jpg">
</p>

After returning from this year's Tableau Conference and all the great sessions on Tableau's APIs, I couldn't wait to dive into the content the devs made available. I have been collecting ideas for automated Tableau Server tools for a while now and TC16 ignited the passion in me to put some of them into action.

I started by exploring the REST API with Python and quickly realized that there was a gap between what I wanted to build and the functionality that the API offered. While many Server actions can be scripted, simple things like modifying the schedule attached to an Extract refresh isn't possible. Why Tableau? You'll let me change the schedule for a Subscription, so why not an Extract?

The answer is probably that the devs are super busy and they just haven't gotten around to it. Hopefully this and other Extract functionality (like triggering Extract refreshes) is just a release or two away.

But what should I do in the mean time? I can't just give up. This is a core element of my app. After a quick post to the Global Tableau Developers Group, I found a tiny light at the end of the tunnel in the form of Tableau's "Undocumented" API.

## What is the "Undocumented API"?

You're probably not familiar with the Undocumented API by name but if you've ever used Tableau Server, you've used it. Put simply, the Undocumented API (hereafter referred to by its real name: the Vizportal API) is the technology that enables Tableau Server's Web Client. Any time you interact with Tableau Server, your browser is issuing calls via the Vizportal API to execute the commands you choose. Using tools like Fiddler (or by inspecting the page with Chrome or Safari) you can see the calls that your browser is making to the Vizportal API.

<p align="center">

<img src="https://viziblydiffrnt.github.io/assets/images/undocumented_api.jpg">

<i>By inspecting the page, we can see there is a call to Vizportal to "getExtractTasks"</i>
</p>

## This is Amazing! How Come I Never Knew this Existed?


From this we can clearly see that there are API methods for lots of tasks that are not part of the REST API. By leaving the inspector window open and clicking around Server, you can see exactly what calls are used to control each function. Amazing! How come I never knew this existed?

This is why the Vizportal API is known as the Undocumented API. While it's technically possible to make these same calls from your own code, it's not officially supported by Tableau. Tableau could disable or change these methods at any time, without notice so if you're going to use this consider yourself warned.

## How To Connect to the Vizportal API using Python

Even though it's not supported or officially documented, there is some material online that discusses how you can connect to Vizportal. The team at Tableau that runs their internal server has a post that describes one method but it stops just short of providing the code. I'll present a method using Python (version 2.7) but this could be done with Javascript or any other language the supports HTTP requests and RSA encryption. To log into the Vizportal API we'll need to perform the following steps:

1. Generate a Public Key that we can use to encrypt our password that will be used for the login
2. Encrypt the user's password with RSA PKCS1 encryption
3. Login to Vizportal and retain the workgroup_session_id and XSRF-TOKEN so we can submit other calls to Vizportal

*__Before I go any further I should point out that this is not a back door into Tableau Server. We are still authenticating with a credentialed user and as such, the requests we make will honor the permissions of the user that we're logging in with. If you're unable to successfully execute this, check that you're using a licensed user account with permission to execute the calls you're making.__*

### Generating the Public Key

The first step we need to do is to generate a public key and capture the modulus and exponent so we can encrypt our password:

```python
import os
import sys
import requests
import json

tab_server_url = "http://your_tableau_server_url"

def _encode_for_display(text):
     return text.encode('ascii', errors="backslashreplace").decode('utf-8')

# Establish a session so we can retain the cookies
session = requests.Session()

def generatePublicKey(): 
      payload = "{\"method\":\"generatePublicKey\",\"params\":{}}"
      endpoint = "generatePublicKey"
      url = tab_server_url + "/vizportal/api/web/v1/"+endpoint
      headers = {
      'content-type': "application/json;charset=UTF-8",
      'accept': "application/json, text/plain, */*",
      'cache-control': "no-cache"
      }
      response = session.post(url, data=payload, headers=headers)
      response_text = json.loads(_encode_for_display(response.text))
      response_values = {"keyId":response_text["result"]["keyId"], "n":response_text["result"]["key"]["n"],"e":response_text["result"]["key"]["e"]}
      return response_values
```
      
This function will return a dictionary with 3 key/value pairs that we can use for the next step.

* The keyId
* The modulus "n"
* The exponent "e"

Put this into practice like this:

```python
 # Generate a pubilc key that will be used to encrypt the user's password
 public_key = generatePublicKey()
 pk = public_key["keyId"]
```

### Encrypting the Password

This was the hardest part of the entire process as I couldn't find any good examples of how to do this with Python. After spending hours looking for a solution, I reached out to Zen Master Tamás Földi who graciously agreed to help me. Here's the code he shared:

```python
# You'll need to install the following modules
# I used PyCrypto which can be installed manually or using "pip install pycrypto"
import binascii
import Crypto
from Crypto.PublicKey import RSA
from Crypto.Cipher import PKCS1_v1_5
from base64 import b64decode

# Encrypt with RSA public key (it's important to use PKCS11)
def assymmetric_encrypt(val, public_key):
     modulusDecoded = long(public_key["n"], 16)
     exponentDecoded = long(public_key["e"], 16)
     keyPub = RSA.construct((modulusDecoded, exponentDecoded))
     # Generate a cypher using the PKCS1.5 standard
     cipher = PKCS1_v1_5.new(keyPub)
     return cipher.encrypt(val)
```

The objective here is to supply your password as the first argument and then input the key retrieved from the previous step as the second argument. The output will be an encrypted string that we will use to log in.

```python
# Encrypt the password used to login
encryptedPassword = assymmetric_encrypt(your_password,public_key)
```

Now that we have the encrypted password, we're ready to log in.

### Logging into Vizportal

The login combines the encryptedPassword with the keyId and the username to issue a request to Tableau Server. If successful, we need to capture the cookie from the response header and extract the workgroup_session_id and XSRF-TOKEN to use in any subsequent calls.

```python
# Make sure to change your username
tableau_username = "Your Username"

def vizportalLogin(encryptedPassword, keyId):
     encodedPassword = binascii.b2a_hex(encryptedPassword) 
     payload = "{\"method\":\"login\",\"params\":{\"username\":\"%s\", \"encryptedPassword\":\"%s\", \"keyId\":\"%s\"}}" % (tableau_username, encodedPassword,keyId)
     endpoint = "login"
     url = tab_server_url + "/vizportal/api/web/v1/"+endpoint
     headers = {
     'content-type': "application/json;charset=UTF-8",
     'accept': "application/json, text/plain, */*",
     'cache-control': "no-cache"
     }
     response = session.post(url, data=payload, headers=headers)
     return response

# Capture the response
login_response = vizportalLogin(encryptedPassword, pk)
# Parse the cookie
sc = login_response.headers["Set-Cookie"]
set_cookie = dict(item.split("=") for item in sc.split(";"))
xsrf_token, workgroup_session_id = set_cookie[" HttpOnly, XSRF-TOKEN"], set_cookie["workgroup_session_id"]
```

If you've made it this far you should be able to log in into Vizportal. Now comes for the fun part!

## Using the Vizportal API: Update Extract Schedules

As I mentioned before, it's currently not possible to change the Extract schedule associated with an Extract refresh using the REST API but it can be done with Vizportal. Before you try this, take a look again at the call being made in the browser. It will give you the format you'll need to construct the payload for the HTTP request. I had to experiment with the formatting of the payload string to get it to match but once I did it worked perfectly.

<p align="center">
<img src="https://viziblydiffrnt.github.io/assets/images/undocumented_api_setSchedules.jpg">
</p>

```python
# The task_id is the value from the "tasks" table in the Workgroup database 
# that corresponds to the Extract associated with your Workgroup/Datasource
task_id = "55"
# The schedule_id is the value from the "_schedules" table in the Workgroup database 
# for the schedule you want to change the task to use
schedule_id = "1"

# Use the xsrf_token you got from the login response cookie
def updateExtractSchedule(task_id, schedule_id, xsrf_token):
     payload = "{\"method\":\"setExtractTasksSchedule\",\"params\":{\"ids\":[\"%s\"], \"scheduleId\":\"%s\"}}" % (task_id, schedule_id)
     endpoint = "setExtractTasksSchedule"
     url = tab_server_url + "/vizportal/api/web/v1/"+endpoint
     headers = {
     'content-type': "application/json;charset=UTF-8",
     'accept': "application/json, text/plain, */*",
     'cache-control': "no-cache",
     'X-XSRF-TOKEN':xsrf_token
      }
     response = session.post(url, data=payload, headers=headers)
     print response.status_code
     return response
```

Success!

## So Why Do This?

The current version of the REST API lacks some basic functionality that would be really useful to have. If you have a Server that doesn't have a dedicated Administrator it can be hard to maintain the content and schedules as it grows. Automating these tasks can give your Tableau Server extra life and improve the experience for your users. I fully expect that more functionality will be added with each release and it's only a matter of time before the REST API becomes fully featured. In the mean time, experiment with Vizportal and see what you can come up with.

Big thanks to Chris Toomey and Tamás Földi for all of their help in figuring this out.