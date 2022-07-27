## Introduction

Here we go again!

The target of this tutorial is to create Gitlab charts based on your needs, by exploiting the Gsheet functionality.

Gitlab is a powerful tool for software development, but for other stakeholders, the need to show charts could require external plugins or
the free solution which I'm going to describe in this post.

The logic is to get desired Gitlab data through the API Calls where the query results can be post-processed and automated into a Gsheet, which includes the possibility to easily create and embed charts.

I've already reported in this post an example of Gsheet automation via Google AppScript [Notifications via Gsheet](https://blog.zuru.tech/scripting/testing/cd/ci/2022/01/20/daily-builds-send-discord-gmail-notifications-via-gsheet) where you can get a short overview regarding the Coding environment and how it generally works.

#### Gitlab API Calls

In this section, there is a recap of some information retrieved from the other panels (last run, internal statistics, etc.).

<div class="blog-image-container" markdown="1">
![10](/images/notif-scripts/10.png){:class="blog-image"}
</div>


#### Goo

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

#### Chart Samples

Here the samples email received:

<div markdown="1" class="blog-image-container">
![5](/images/notif-scripts/5.png){:class="blog-image"}
</div>

<div markdown="1" class="blog-image-container">
![12](/images/notif-scripts/12.png){:class="blog-image"}
</div>

#### Embedded Samples

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

[1] [Use the API](https://docs.gitlab.com/ee/api/)

[2] [Gitlab API Resources](https://docs.gitlab.com/ee/api/api_resources.html)


