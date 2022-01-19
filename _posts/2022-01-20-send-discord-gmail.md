---
author: "lucadandria"
layout: "post"
title:  "Daily Builds: Send Discord/Gmail Notifications via Gsheet"
slug:   "Daily Builds: Send Discord/Gmail Notifications via Gsheet"
date:    2022-01-20 8:00:00
categories: Testing, CD/CI
image: images/gappscript.PNG
tags: scripting, testing, cd, ci
---

## Daily Builds: Send Discord/Gmail Notifications via Gsheet

Hey, where can I find the latest builds?
Hey, which build the QA Team is using right now?

The above questions are something that I heard plenty of times, from many people from different company’s areas.

It makes perfect sense: in a continuous deployment environment, it’s easy to be up-to-date: that’s why, considering that I use a nested Google sheet in a Google site to track the information regarding the build deployed from Gitlab which are started to be manually tested (smoke testing, feature testing, regression testing etc), be quickly informed regarding the latest available one, it could be really helpful.

At the moment, the scripts send two notifications using the Discord API and Google API. 

![1](https://user-images.githubusercontent.com/70565462/150167036-7816763a-5c93-49b2-870b-ae06dcdac511.PNG)

To reach the result, I used **Google App Script** environment, which allows extending Google sheet functionality using VBA-like code, a debugger, and **trigger** feature. 


Here below the Gsheet tample portion, updated with the uploaded builds retrieved from the Gitlab CI (here the post of Paolo Galeone, https://blog.zuru.tech/coding/2020/09/29/gitlab-ci-cd-for-cross-platform-unreal-engine-4-projects regarding this topic).


![2](https://user-images.githubusercontent.com/70565462/150167250-78363c66-684f-40aa-9be2-dbd5e95c9be2.PNG)


The events that trigger the sending of the notifications are basically the flags under the columns **W** (windows), **L** (Linux), **M** (MacOSx): when the flag is set on 
**“true”** the trigger is invoked and the notifications are sent; otherwise, if the flags are set on **“false”**, the trigger works “silently” and will skip any sending.


## Google Apps Script environment
Google App Script is integrated into the Gsheet topbar, via the “Extension” button. The development environment is basically composed of the below sections:

![3](https://user-images.githubusercontent.com/70565462/150173157-70d028a3-e592-41f5-8439-0cd83da75665.PNG)


#### Overview
In this section there is a recap of some informations retrieved from the other panels (last run, internal statistics, etc.)

![10](https://user-images.githubusercontent.com/70565462/150177734-60cf4761-e9ba-4ce2-8a0d-a3daa6cac68b.PNG)


#### Editor
It’s the section where you can create scripts that contain functions and the related code; all the scripts as **.gs** as the file extension.
However, there is a debug section that allows you to test the syntax in the code; all the executions performed via debugger are stored as entries in the Executions panel.

![9](https://user-images.githubusercontent.com/70565462/150177904-7002adad-4a39-49e1-9300-d02fd24aaffc.PNG)


#### Triggers
It’s the function that allows us to **perform a task when an event occurs**. 
In our cases, the task is to execute the functions in the scripting code, **when the Gsheet is updated**. 
The if condition written in the script allows the user to run the function silently if the “sending” condition is not met.

![7](https://user-images.githubusercontent.com/70565462/150172228-dbc72334-6387-4ddd-aba4-a605d7594e14.PNG)


#### Executions

In this section, there are all the entries related to the function executions with some relevant information like the type (debugger, manually or triggered), the timestamp, and the outcome.

![8](https://user-images.githubusercontent.com/70565462/150172540-552a5efd-702f-4ab5-ab65-70d7675b0379.PNG)


## Discord Code Script

```java
function sendiscord(){ 
//  Select the active Sheet
var s = SpreadsheetApp.getActiveSpreadsheet();
// Select the active Cell
var r1 = s.getActiveCell();
// Get the value of the active cell (true or false)
var flag = r1.getValue();
// Get the column of the active cell 
var col = r1.getColumn();
 
// Get the value of the active cell with a dx offset (Commit ID) 
var r2 = r1.offset (0,3); var offdx1 = r2.getValue(); 
var r3 = r1.offset (0,2); var offdx2 = r3.getValue();
var r4 = r1.offset (0,1); var offdx3 = r4.getValue();
 
// Get the value of the active cell with a sx offset (Day) 
var r5 = r1.offset (0,-1); var offsx1 = r5.getValue();
var r6 = r1.offset (0,-2); var offsx2 = r6.getValue();
var r7 = r1.offset (0,-3); var offsx3 = r7.getValue();
 
// Windows Selector
   if (flag == true && col == 4){
// POST call using Discord Webhook
        function sendMessage(message) {
        var url = "Discord_Webhook_url";
        var payload = JSON.stringify({content: message});
        var params = {
          headers: {"Content-Type": "application/json"},
          method: "POST",
          payload: payload,
          muteHttpExceptions: true 
          };
         // get the parameters url and params, and print via logger.log function
        var res = UrlFetchApp.fetch(url, params);
        Logger.log(res.getContentText());
        } 
        sendMessage("Uploaded Windows N_" + offsx1 + "_22 Build -" + offdx1 + " Here or below: WINDOWS GDRIVE");  // Linux Selector
    }  else if (flag == true && col == 5) {
        function sendMessage(message) {
        var url = "Discord_Webhook_url";
        var payload = JSON.stringify({content: message});
        var params = {
          headers: {"Content-Type": "application/json"},
          method: "POST",
          payload: payload,
          muteHttpExceptions: true 
          };
         // get the parameters url and params, and print via logger.log function
        var res = UrlFetchApp.fetch(url, params);
        Logger.log(res.getContentText());
        } 
        sendMessage("Uploaded Linux N_" + offsx2 + "_22 Build -" + offdx2 + " Here or below: “LINUX GDRIVE");
    } // MacOS Selector
       else if (flag == true && col == 6) {
        function sendMessage(message) {
        var url = "Discord_Webhook_url";
        var payload = JSON.stringify({content: message});
        var params = {
          headers: {"Content-Type": "application/json"},
          method: "POST",
          payload: payload,
          muteHttpExceptions: true 
          };
        // get the parameters url and params, and print via logger.log function 
        var res = UrlFetchApp.fetch(url, params);
        Logger.log(res.getContentText());
        } 
        sendMessage("Uploaded MacOSX N_" + offsx3 + "_22 Build -" + offdx3 + " Here or below: “MACOSX GDRIVE");
    } else {
      console.log ("Don't send")
    }
  }

```
## Gmail Code Script

```java
function sendmail (){ 
//  Select the active Sheet
var s = SpreadsheetApp.getActiveSpreadsheet();
// Select the active Cell
var r1 = s.getActiveCell();
// Get the value of the active cell (true or false)
var flag = r1.getValue();
// Get the column of the active cell 
var col = r1.getColumn();
 
// Get the value of the active cell with a dx offset (Commit ID) 
var r2 = r1.offset (0,3); var offdx1 = r2.getValue(); 
var r3 = r1.offset (0,2); var offdx2 = r3.getValue();
var r4 = r1.offset (0,1); var offdx3 = r4.getValue();
 
// Get the value of the active cell with a sx offset (Day) 
var r5 = r1.offset (0,-1); var offsx1 = r5.getValue();
var r6 = r1.offset (0,-2); var offsx2 = r6.getValue();
var r7 = r1.offset (0,-3); var offsx3 = r7.getValue();
      
   if (flag == true && col == 4){ // Win Selector
        return MailApp.sendEmail({ 
          to: "luca@test.it",
          subject: "New Windows Build",
          htmlBody: "Download N_" + offsx1 + "_22 Build -" + offdx1 + " Here or below: WIN GDRIVE”});
} else if (flag == true && col == 5) { // Linux Selector
        return MailApp.sendEmail({ 
          to: "luca@test.it",
          subject: "New Linux Build",
          htmlBody: "Download N_" + offsx2 + "_22 Build -" + offdx2 + " Here or below: LINUX GDRIVE",});
    } else if (flag == true && col == 6) { // MacOS Selector
        return MailApp.sendEmail({ 
          to: "luca@test.it",
          subject: "New MacOSX Build",
          htmlBody: "Download N_" + offsx3 + "_22 Build -" + offdx3 + " Here or below: MAC GDRIVE",});
    } else {
      console.log ("Don't send")
    }
  }

```
#### Discord Notifications Screenshot

The messages coming from the bot defined via webhook:

![4](https://user-images.githubusercontent.com/70565462/150170338-6da0b260-cc63-4b3c-adec-c98fb2ecc761.PNG)

#### Gmail Notification Screenshots

A formatted email with different receipts:

![image](https://user-images.githubusercontent.com/70565462/150170654-62d964e1-7055-49d6-b951-c8573c78540f.png)

## Potential Future Reuse 

A potential improvement could be to use Gitlab API calls to retrieve data to nourish Gsheet statistics and/or retrieve the builds directly from the git repository 
when the CI/CD ends is daily scheduled task.
The notification scripts can be re-used in any cases, and the first feedback regarding this first implementation, are really flattering.

![6](https://user-images.githubusercontent.com/70565462/150171922-b0440ea5-9e0c-47dd-8e62-de8e006be7d5.PNG)


---

**Resources**

[1] [Google Appscript Overview](https://developers.google.com/apps-script/overview)

[2] [MailApp Function](https://developers.google.com/apps-script/reference/mail/mail-app)

[3] [Jscript Discord Message](https://dev.to/oskarcodes/send-automated-discord-messages-through-webhooks-using-javascript-1p01)
