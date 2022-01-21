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

It makes perfect sense: in a continuous deployment environment, it’s easy to be up-to-date: that’s why, considering that I use a nested Google sheet in a Google site to track the information regarding the build deployed from Gitlab which are started to be manually tested (smoke testing, feature testing, regression testing, etc), be quickly informed regarding the latest available one, it could be really helpful.

At the moment, the scripts send two notifications using the Discord API and Google API.

![1](/images/notif-scripts/1.png)

To reach the result, I used the  **Google App Script** environment, which allows extending Google sheet functionality using VBA-like code, a debugger, and a **trigger** feature.

Here below is the GSheet table portion, updated with the uploaded builds retrieved from the Gitlab CI (<a href="/coding/2020/09/29/gitlab-ci-cd-for-cross-platform-unreal-engine-4-projects">here is the post of Paolo Galeone</a> regarding this topic).

![2](/images/notif-scripts/2.png)


The events that trigger the sending of the notifications are basically the flags under the columns **W** (windows), **L** (Linux), **M** (MacOSx): when the flag is set on
**“true”** the trigger is invoked and the notifications are sent; otherwise, if the flags are set on **“false”**, the trigger works “silently” and will skip any sending.

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
However, there is a debug section that allows you to test the syntax in the code; all the executions performed via debugger are stored as entries in the Executions panel.

<div markdown="1" class="blog-image-container">
![9](/images/notif-scripts/9.png){:class="blog-image"}
</div>

#### Triggers

It’s the function that allows us to **perform a task when an event occurs**.
In our cases, the task is to execute the functions in the scripting code, **when the GSheet is updated**.
The if condition written in the script allows the user to run the function silently if the “sending” condition is not met.

<div markdown="1" class="blog-image-container">
![7](/images/notif-scripts/7.png){:class="blog-image"}
</div>

#### Executions

In this section, there are all the entries related to the function executions with some relevant information like the type (debugger, manually or triggered), the timestamp, and the outcome.

<div class="blog-image-container" markdown="1">
![8](/images/notif-scripts/8.png){:class="blog-image"}
</div>

## Discord Code Script

Here below is the Discord code script; the code part where the information gets retrieved from the data table is the same used for the Gmail Script: I decided to split to debug/test the single notifications.

Differently from the Gmail script one, we need to activate a **Webhook** url from the Discord side, in order to allow Gsheet to perform an API Call.

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

    // Get the value of the active cell with a dx offset (Commit ID)
    var r2 = r1.offset(0, 3);
    var offdx1 = r2.getValue();
    var r3 = r1.offset(0, 2);
    var offdx2 = r3.getValue();
    var r4 = r1.offset(0, 1);
    var offdx3 = r4.getValue();

    // Get the value of the active cell with a sx offset (Day)
    var r5 = r1.offset(0, -1);
    var offsx1 = r5.getValue();
    var r6 = r1.offset(0, -2);
    var offsx2 = r6.getValue();
    var r7 = r1.offset(0, -3);
    var offsx3 = r7.getValue();

    // Windows Selector
    if (flag == true && col == 4) {
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
        sendMessage("Download Windows N_" + offsx1 + "_22 Build -" + offdx1 + " Here: WINDOWS GDRIVE"); // Linux Selector
    } else if (flag == true && col == 5) {
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
        sendMessage("Download Linux N_" + offsx2 + "_22 Build -" + offdx2 + " Here: “LINUX GDRIVE");
    } // MacOS Selector
    else if (flag == true && col == 6) {
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
        sendMessage("Download MacOSX N_" + offsx3 + "_22 Build -" + offdx3 + " Here: “MACOSX GDRIVE");
    } else {
        console.log("Don't send")
    }
}
```
## Gmail Code Script

Here below is the Gmail code script; in this case, the Google API Calls are automatically allowed thanks to the performed acknowledge provided via your google account, using the Gsheet and App script. Thanks a lot, Google!

```javascript
function sendmail() {
    //  Select the active Sheet
    var s = SpreadsheetApp.getActiveSpreadsheet();
    // Select the active Cell
    var r1 = s.getActiveCell();
    // Get the value of the active cell (true or false)
    var flag = r1.getValue();
    // Get the column of the active cell
    var col = r1.getColumn();

    // Get the value of the active cell with a dx offset (Commit ID)
    var r2 = r1.offset(0, 3);
    var offdx1 = r2.getValue();
    var r3 = r1.offset(0, 2);
    var offdx2 = r3.getValue();
    var r4 = r1.offset(0, 1);
    var offdx3 = r4.getValue();

    // Get the value of the active cell with a sx offset (Day)
    var r5 = r1.offset(0, -1);
    var offsx1 = r5.getValue();
    var r6 = r1.offset(0, -2);
    var offsx2 = r6.getValue();
    var r7 = r1.offset(0, -3);
    var offsx3 = r7.getValue();

    if (flag == true && col == 4) { // Win Selector
        return MailApp.sendEmail({
                to: "luca@test.it",
                subject: "New Windows Build",
                htmlBody: "Download N_" + offsx1 + "_22 Build -" + offdx1 + " Here or below: WIN GDRIVE",
            });
    } else if (flag == true && col == 5) { // Linux Selector
        return MailApp.sendEmail({
            to: "luca@test.it",
            subject: "New Linux Build",
            htmlBody: "Download N_" + offsx2 + "_22 Build -" + offdx2 + " Here or below: LINUX GDRIVE",
        });
    } else if (flag == true && col == 6) { // MacOS Selector
        return MailApp.sendEmail({
            to: "luca@test.it",
            subject: "New MacOSX Build",
            htmlBody: "Download N_" + offsx3 + "_22 Build -" + offdx3 + " Here or below: MAC GDRIVE",
        });
    } else {
        console.log("Don't send")
    }
}
```
#### Discord Notifications Screenshots

The messages coming from the bot defined via webhook:

<div markdown="1" class="blog-image-container">
![4](/images/notif-scripts/4.png){:class="blog-image"}
</div>

#### Gmail Notification Screenshot

A formatted email with different receipts:

<div markdown="1" class="blog-image-container">
![5](/images/notif-scripts/5.png){:class="blog-image"}
</div>

## Potential Future Reuse

A potential improvement could be to use Gitlab API calls to retrieve data to nourish GSheet statistics and/or retrieve the builds directly from the git repository
when the CI/CD ends the daily scheduled task.
The notification scripts can be re-used in many cases, and the first feedback regarding this first implementation is really flattering.

![6](/images/notif-scripts/6.png)

---

**Resources**

[1] [Google Appscript Overview](https://developers.google.com/apps-script/overview)

[2] [MailApp Function](https://developers.google.com/apps-script/reference/mail/mail-app)

[3] [Jscript Discord Message](https://dev.to/oskarcodes/send-automated-discord-messages-through-webhooks-using-javascript-1p01)
