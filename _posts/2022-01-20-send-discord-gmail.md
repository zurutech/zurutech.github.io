---
author: "lucadandria"
layout: "post"
title:  "Daily Builds: Send Discord/Gmail Notifications via GSheet"
slug:   "Daily Builds: Send Discord/Gmail Notifications via GSheet"
date:    2022-01-20 6:00:00
categories: scripting testing cd ci
image: images/notif-scripts/gappscript.png
tags: coding
---

Hey, where can I find the latest builds?

Hey, which build the QA Team is using right now?

The above questions are something that I heard plenty of times, from many people from different company’s areas.

It makes perfect sense: in a continuous deployment environment, it’s easy to be up-to-date: that’s why, considering that I use a nested Google sheet in a Google site to track the information regarding the build deployed from Gitlab CI (<a href="/coding/2020/09/29/gitlab-ci-cd-for-cross-platform-unreal-engine-4-projects">here is the post of Paolo Galeone</a>), which are ready to be manually tested (smoke testing, feature testing, regression testing, etc), be quickly informed regarding the latest available one, it could be really helpful.

At the moment, the scripts send two notifications using the Discord API and Google API.

<div class="blog-image-container" markdown="1">
![1](/images/notif-scripts/1.png){:class="blog-image"}
</div>

To achieve the result I used **Google App Script**, a JavaScript cloud scripting language that provides easy ways to automate tasks across Google products and third-party services and build web applications which allow extending Google sheet functionality. 
**Google App Script environment** includes some really useful features like a **trigger** panel.

Here below is the GSheet table portion, updated with the uploaded builds that passed the automation tests in the pipeline:

<div class="blog-image-container" markdown="1">
![2](/images/notif-scripts/2.png){:class="blog-image"}
</div>

When the builds are uploaded in the repository (currently Gdrive) after the successful execution of the automated tests via Gitlab's pipeline (as explained in the <a href="/coding/2021/02/12/unit-testing-with-unreal-engine-4"> Francesco Chiatti's post </a> ), the notifications are ready to be sent to all the stakeholders, but how? 
The idea is to handle the notifications through the flags positioned along the columns **W** (windows), **L** (Linux), **M** (MacOSx): when the flags of a specific column is set on **“true”** a consistent notification is sent with the build type and its references.


## Google Apps Script environment

Google App Script is integrated into the GSheet topbar, via the “Extension” button. The development environment is basically composed of the below sections:

![3](/images/notif-scripts/3.png)

#### Overview

In this section, there is a recap of some information retrieved from the other panels (last run, internal statistics, etc.).

<div class="blog-image-container" markdown="1">
![10](/images/notif-scripts/10.png){:class="blog-image"}
</div>


#### Editor

It’s the section where you can create scripts that contain functions and the related code; all the scripts as **.gs** as the file extension.
However, there is a debug section that allows to check the code syntax and run the scripts: all the executions performed via debugger are stored as entries in the **Executions** panel.

<div markdown="1" class="blog-image-container">
![9](/images/notif-scripts/9.png){:class="blog-image"}
</div>

#### Triggers

It’s the automatic task that is executed **when an event occurs**: I used the event **when the GSheet is updated**, in order to trigger the execution of the scripts every time the flags are set to true; An if condition inside the code allows to solicit the trigger avoiding any sendings, when the event condition is not met.

So what, the trigger is sensible to all the Gsheet updates, but the invoked functions drives the final outcome.   

<div markdown="1" class="blog-image-container">
![7](/images/notif-scripts/7.png){:class="blog-image"}
</div>

#### Executions

In this section, there are all the entries related to the function executions with some relevant information like the type (debugger, manually or triggered), the timestamp, and the outcome.

<div class="blog-image-container" markdown="1">
![8](/images/notif-scripts/8.png){:class="blog-image"}
</div>

## Discord Code Script

Differently from the Gmail script one, a **Webhook** activation from the Discord side is needed.

Webhooks are user-defined HTTP callbacks that are triggered by specific events; whenever the trigger event occurs, the Discord API client sees the event, collects the data, and immediately sends a notification (HTTP request) to the Webhook URL specified in the application settings. These requests are known as webhooks or callbacks.

<div markdown="1" class="blog-image-container">
![11](/images/notif-scripts/11.png){:class="blog-image"}
</div>


```javascript
function sendiscord() {
    //  Select the active Sheet
    var s = SpreadsheetApp.getActiveSpreadsheet();
    // Select the active Cell
    var r1 = s.getActiveCell();
    // Get the value of the active cell (true or false)
    var flag = r1.getValue();
    // Get the column of the active cell
    var col = r1.getColumn();
    
     //Windows Data
     var w_day = r1.offset(0,-1).getValue(); 
     var w_com = r1.offset(0,3).getValue();

    //Linux Data
    var l_day = r1.offset(0,-2).getValue(); 
    var l_com = r1.offset(0,2).getValue();
    
    //MacOSX Data
    var m_day = r1.offset (0,-3).getValue(); 
    var m_com = r1.offset (0,1).getValue();

    // Current Month  
   var m = new Date().getMonth()+1;
   var month = ("0" + m.toString().slice(-2));

     // Current Year 
    var f_year = new Date().getFullYear().toString();
    var year = " ";
    year = f_year.substring(2,4);

    if (flag == true && col == 4) { // Windows Selector
        // POST call using Discord Webhook
        function sendMessage(message) {
            var url = "Discord_Webhook_url";
            var payload = JSON.stringify({
                content: message
            });
            var params = {
                headers: {
                    "Content-Type": "application/json"
                },
                method: "POST",
                payload: payload,
                muteHttpExceptions: true
            };
            // get the parameters url and params, and print via logger.log function
            var res = UrlFetchApp.fetch(url, params);
            Logger.log(res.getContentText());
        }
        sendMessage("Download Windows N_" + w_day + "_" + month + "_" + year + "-" + w_com + " Here : WINDOWS GDRIVE URL"); 
    } else if (flag == true && col == 5) {  // Linux Selector
        function sendMessage(message) {
            var url = "Discord_Webhook_url";
            var payload = JSON.stringify({
                content: message
            });
            var params = {
                headers: {
                    "Content-Type": "application/json"
                },
                method: "POST",
                payload: payload,
                muteHttpExceptions: true
            };
            // get the parameters url and params, and print via logger.log function
            var res = UrlFetchApp.fetch(url, params);
            Logger.log(res.getContentText());
        }
        sendMessage("Download Linux N_" + l_day + "_" + month + "_" + year + "-" + l_com + " Here: “LINUX GDRIVE URL");
    } // MacOS Selector
    else if (flag == true && col == 6) {  // Mac Selector
        function sendMessage(message) {
            var url = "Discord_Webhook_url";
            var payload = JSON.stringify({
                content: message
            });
            var params = {
                headers: {
                    "Content-Type": "application/json"
                },
                method: "POST",
                payload: payload,
                muteHttpExceptions: true
            };
            // get the parameters url and params, and print via logger.log function
            var res = UrlFetchApp.fetch(url, params);
            Logger.log(res.getContentText());
        }
        sendMessage("Download MacOSX N_" + m_day + "_" + month + "_" + year + "-" + m_com + " Here: “MACOSX GDRIVE URL");
    } else {
        console.log("Don't send")
    }
}
```
## Gmail Code Script

Here below is the Gmail code script; in this case, the Google API Callbacks are automatically allowed thanks to the acknowledgment provided by using the Google App script and the integration with the other Google Services. 

Unlike the previous code, other data such as the delta between the commits, have been added to the email format.

```javascript
function sendmail_t(){ 
var s = SpreadsheetApp.getActiveSpreadsheet();
var r1 = s.getActiveCell();
var flag = r1.getValue();
var col = r1.getColumn();

//Windows Data

var w_day = r1.offset(0,-1).getValue(); 
var w_com = r1.offset(0,3).getValue();

//Linux Data

var l_day = r1.offset(0,-2).getValue(); 
var l_com = r1.offset(0,2).getValue();

//MacOSX Data

var m_day = r1.offset (0,-3).getValue(); 
var m_com = r1.offset (0,1).getValue();


   // Current Month  
   var m = new Date().getMonth()+1;
   var month = ("0" + m.toString().slice(-2));
  
  // Current Year 
   var f_year = new Date().getFullYear().toString();
   var year = " ";
   year = f_year.substring(2,4);
      
   if (flag == true && col == 4){
        w_delta = r1.offset(0,4).getRichTextValue().getLinkUrl();
        return MailApp.sendEmail({ 
          to: "luca.d@zuru.tech",
          noReply: true,
          subject: "New Windows Build",
          htmlBody: "Download Windows N_" + w_day + "_" + month + "_" + year + "-" + w_com + " Here or below: WIN GDRIVE URL <br>" + "<br> Delta:" + w_delta + "<br> <br> QA Board: https://sites.google.com/zuru.tech/qaboard/home <br>" + "<br> <img src='https://blog.zuru.tech/images/logo_zuru.png'> <br>",});
} else if (flag == true && col == 5) {
        var l_delta = r1.offset(0,3).getRichTextValue().getLinkUrl();
        return MailApp.sendEmail({ 
          to: "luca.d@zuru.tech",
          noReply: true,
          subject: "New Linux Build",
          htmlBody: "Download Linux N_" + l_day + "_" + month + "_" + year + "-" + l_com + " Here or below: LINUX GDRIVE URL <br>" + "<br> Delta:" + l_delta + "<br> <br> QA Board: https://sites.google.com/zuru.tech/qaboard/home <br>" + "<br> <img src='https://blog.zuru.tech/images/logo_zuru.png'> <br>",});
    } else if (flag == true && col == 6) {
      var m_delta = r1.offset (0,2).getRichTextValue().getLinkUrl();
        return MailApp.sendEmail({ 
          to: "luca.d@zuru.tech",
          noReply: true,
          subject: "New MacOSX Build",
          htmlBody: "Download MacOSX N_" + m_day + "_" + month + "_" + year + "-" + m_com + " Here or below: MAC GDRIVE URL <br>" + "<br> Delta:" + m_delta + "<br> <br> QA Board: https://sites.google.com/zuru.tech/qaboard/home <br>" + "<br> <img src='https://blog.zuru.tech/images/logo_zuru.png'> <br>",});
    } else {
      console.log ("Don't send")
    }
  }
```
#### Discord Notifications Screenshots

The messages coming from the bot defined via webhook:

<div markdown="1" class="blog-image-container">
![4](/images/notif-scripts/4.png){:class="blog-image"}
</div>

#### Gmail Notification Screenshot

Here the samples email received:

<div markdown="1" class="blog-image-container">
![5](/images/notif-scripts/5.png){:class="blog-image"}
</div>

<div markdown="1" class="blog-image-container">
![12](/images/notif-scripts/12.png){:class="blog-image"}
</div>

<div markdown="1" class="blog-image-container">
![13](/images/notif-scripts/13.png){:class="blog-image"}
</div>

## Potential Future Reuse

A potential improvement could be to use Gitlab API calls to retrieve data to nourish GSheet statistics and/or retrieve the builds directly from the git repository
when the CI/CD ends the daily scheduled task.
The notification scripts can be re-used in many cases, and the first feedback regarding this first implementation is really flattering.

<div class="blog-image-container" markdown="1">
![6](/images/notif-scripts/6.png){:class="blog-image"}
</div>

---

**Resources**

[1] [Google Appscript Overview](https://developers.google.com/apps-script/overview)

[2] [MailApp Function](https://developers.google.com/apps-script/reference/mail/mail-app)

[3] [Jscript Discord Message](https://dev.to/oskarcodes/send-automated-discord-messages-through-webhooks-using-javascript-1p01)
