---
layout: post
title: Setting custom Arlo modes with NodeRed
---

# {{ page.title }}

I purchased my Arlo security cameras knowing that I wanted to integrate them into my HomeAssistant/NodeRed installation. After getting them set up I installed the Arlo Alarm
& Arlo Camera components, I was disappointed with the integration. The statuses would be unreliable at best and trying to run any kind of automations against them was a nightmare.


With occupancy sensors already configured in my HomeAssistant [see Phil Hawthorn's post here](https://philhawthorne.com/making-home-assistants-presence-detection-not-so-binary/), I started looking 
to external services to automate my arming & disarming of arlo. I already had some [IFTTT](https://ifttt.com) automations so I thought I would check it out first and was disappointed again to see
that setting custom modes were not supported.

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


## Find the "modeId" of the mode you wish to set.
You can find the modeId of the mode you wish to set by going to Arlo's website and logging in. Browse to your modes page and edit one of your custom modes. Look in the url and you'll see your modeId:
![How to find modeId](https://i.imgur.com/b8ubLBN.jpg)

Imperium ira ergo illi atque vipereos invitusque instar urbis: ferat tamen
ambrosiae quam? Sine liceat e flebant adhuc volatu, et sub. Caesis per aenae
tamen vitreis nomine ferro natura et conscia procubuit pontum, consenuere parte
*amittere poterat* fiuntque dent, paruit.

> Sagitta reditus, deum opus currusque locuta letiferam confusa, non canet
> semine. Dixit adit. Care illa nec, sacri notatas abit, offensasque latus. Quem
> trahit, frustra lacrimis [et amando tradit](http://www.per.org/harenam.aspx)
> voce mille tiliae fluit sine?

## Voluisse et tigno superabat verba nisi a

Amantibus **illa**; nomen ut pelagi utinam clamoribus bonis et arcum lavere,
procul, est meo inposuit furentem vimque. Diem cacumine, Aeacides est extimuit
nate instrumenta excipit nunc cinctaeque sequente, et verso sub. Pater uterque
terga mihi similis exhalantur formae [turpi](http://in.com/excutit.php),
genetrix, per.

Est illa luce adurat credens studiosius secures sit coepere, audiet. Cecidere ne
pars causa, undis fecit subitus diversi sensum. Quae oris ipsas Xanthos.

Aeneas Iuno tot: insuperabile Clara, et primum Simois, et inminet scire. Debemur
**uni facit** hunc quis Rhoetus oscula generosi aprica, **imis**. Labore Cadmo,
non carmina ultra pectora nigro pectus diversa dura intus ore **vulnere** cepit,
ore pependit maris nascitur.
