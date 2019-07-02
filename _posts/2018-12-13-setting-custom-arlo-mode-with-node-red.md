---
layout: post
title: Setting custom Arlo modes with NodeRed
tags: homeassistant arlo nodered smarthome
image: camera.jpg
---

## Update: 5/2019
I am slowly migrated from an http based flow over to full integration with Arlo into homeassistant using [hass-aarlo](https://github.com/twrecked/hass-aarlo). This is a full websocket driving homeassistant integration and has been working pretty well with my system. The HTTP/Node-red solution below still works however moving forward, I think this integration will be a more elegant solution.

## Intro
I purchased my Arlo security cameras knowing that I wanted to integrate them into my HomeAssistant/NodeRed installation. After getting them set up I installed the Arlo Alarm
& Arlo Camera components, I was disappointed with the integration. The statuses would be unreliable at best and trying to run any kind of automations against them was a nightmare.


With occupancy sensors already configured in my HomeAssistant [see Phil Hawthorn's post here](https://philhawthorne.com/making-home-assistants-presence-detection-not-so-binary/), I started looking 
to external services to automate my arming & disarming of arlo. I already had some [IFTTT](https://ifttt.com) automations so I thought I would check it out first and was disappointed again to see
that setting custom modes was not supported.

I figured I wasn't the only one who was frustrated with Arlo, I started looking into the Arlo API itself. While not documented by Arlo, Roberto Gallea [had a great post](https://www.robertogallea.com/blog/netgear-arlo-api) documenting all the endpoints
for me to use via NodeRed. 

A great way to explore this API is to use [Postman](https://www.getpostman.com/).

## 1. Authenticate against Arlo API
You'll be able to authenticate to the Arlo website by making the following call:
```
curl -H "Content-Type: application/json;charset=UTF-8" -d '{"email":"YOUR_EMAIL","password":"YOUR_PASSWORD"}' -X POST "https://arlo.netgear.com/hmsweb/login/v2"
```
Which should return a document similar to 

```json
{
    "data": {
        "userId": "YOUR_USER_ID",
        "email": "YOUR_EMAIL",
        "token": "YOUR_TOKEN",
        "paymentId": "YOUR_PAYMENT_ID",
        "authenticated": 1504597425,
        "accountStatus": "registered",
        "serialNumber": "YOUR_SERIAL_NUMBER",
        "countryCode": "IT",
        "tocUpdate": false,
        "policyUpdate": false,
        "validEmail": true,
        "arlo": true,
        "dateCreated": 1477059159622
    },
    "success": true
}
```

What we need in this response is the `token`

## 2. Retrieve DeviceId & xCloudId
After retrieving your token, you'll need to the the deviceId of your base station (or whatever other device) you'll want to change the mode on. You can do this by making the following request:
```
curl -H "Authorization: YOUR_TOKEN" -X GET "https://arlo.netgear.com/hmsweb/users/devices"
```

This request will return a response similar to the following

```json
{
    "data": [
        {
            "userId": "AB12-2345-6787878",
            "deviceId": "0123456789ABC",
            "parentId": "CBA9876543210",
            "uniqueId": "AB12-2345-6787878_48B5647F83E96",
            "deviceType": "camera",
            "deviceName": "Your camera name",
            "lastModified": 1504596539327,
            "xCloudId": "35CK-1005-210-4644171",
            "lastImageUploaded": "true",
            "userRole": "OWNER",
            "displayOrder": 1,
            "presignedLastImageUrl": "https://arlolastimage-z1.s3.amazonaws.com/......./....",
            "presignedSnapshotUrl": "https://arlos3-prod-z1.s3.amazonaws.com/.../.....",
            "presignedFullFrameSnapshotUrl": "https://arlos3-prod-z1.s3.amazonaws.com/..../.....",
            "mediaObjectCount": 0,
            "state": "provisioned",
            "modelId": "VMC3030",
            "interfaceVersion": "i000",
            "interfaceSchemaVer": "2",
            "owner": {
                "firstName": "OwnerFirstName",
                "lastName": "OwnerLastName",
                "ownerId": "3ACD-458-4578956"
            },
            "properties": {
                "modelId": "VMC3030",
                "olsonTimeZone": "Europe/Amsterdam",
                "hwVersion": "H7"
            }
        },        
        {
            "userId": "CD12-2345-6787878",
            "deviceId": "0123456789CDE",
            "uniqueId": "CD12-2345-6787878_48B5647F83E96",
            "deviceType": "basestation",
            "deviceName": "YourBaseStationName",
            "lastModified": 1504596539327,
            "xCloudId": "35CK-1005-210-4644171",
            "userRole": "OWNER",
            "displayOrder": 2,
            "mediaObjectCount": 0,
            "state": "provisioned",
            "modelId": "VMB3010",
            "interfaceVersion": "i001",
            "interfaceSchemaVer": "2",
            "owner": {
                "firstName": "OwnerFirstName",
                "lastName": "OwnerLastName",
            "properties": {
                "modelId": "VMB3010",
                "olsonTimeZone": "Europe/Amsterdam",
                "hwVersion": "VMB3010r2"
            }
        }
    ],
    "success": true
}
```    

From this response you'll need `deviceId` and `xCloudId`


## 3. Find the "modeId" of the mode you wish to set.
You can find the modeId of the mode you wish to set by going to Arlo's website and logging in. Browse to your modes page and edit one of your custom modes. Look in the url and you'll see your modeId:
<img src="https://i.imgur.com/b8ubLBN.jpg" class='img-fluid'>

## 4. Make request to change device mode.

Using all the information we've gathered, we can now make a `POST` to Arlo's api, to change our mode with the following request:
```
curl -H "Content-Type: application/json;charset=UTF-8" -H "Authorization: YOUR_TOKEN" -H "xcloudid: DEVICE_XCLOUDID" -d "JSON_OBJECT" -X POST "https://arlo.netgear.com/hmsweb/users/devices/notify/DEVICE_ID"
```

## 5. Implement in NodeRed
You can use the `function` node and the `http request` node to make this call.
First you will need to drop a function node in your flow and set the following payload variables replacing text in brackets `$$` with values for your request:


Variable | Value
--- | ---
$FROM$ | A random string. I used one to identify my home assistant instance
$DEVICEID$ | The device id you wish to change modes on
$TRANSACTIONID$ | A random string
$MODEID$ | The modeID you wish to set your device to received in step 3. "mode0" for the default "disarmed" and "mode1" for the default "armed"
$TOKEN$ | The token you received in step 1.
$XCLOUDID$ | The xCloudId for your device recieved in step 2.



```
msg.payload = '{"from":"$FROM$","to":"$DEVICEID$","action":"set","resource":"modes","transId":"$TRANSACTIONID$","publishResponse":true,"properties": {"active":"$MODEID$"}}';
msg.headers = {};
msg.headers['Content-Type'] = 'application/json';
msg.headers['Authorization'] = '$TOKEN$';
msg.headers['xcloudId'] = '$XCLOUDID$';
return msg;
```

Next you will need to drop an `http request` node into your flow and enter the the url to the arlo service, filling in `$DEVICEID$` with your deviceid:
```
https://arlo.netgear.com/hmsweb/users/devices/notify/$DEVICEID$/
```

<img src="https://i.imgur.com/S6Iv03b.jpg" class='img-fluid'>

## That's it.

When this flow run, your Arlo device should change modes.
