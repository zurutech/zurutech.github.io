## Introduction

<div class="blog-image-container" markdown="1">
![Intro](/images/apicharts/intro.png){:class="blog-image"}
</div>

Here we go again!

The target of this tutorial is to create Gitlab charts based on your needs, by exploiting the Gsheet functionality and the Gitlab's API.

Gitlab is a powerful tool for software development, but is not designed for project management and/or for charts; this limit is circumventable
by using external plugins or by following the solution which I'm going to describe here.

Mine main need was to get automatically some data from Gitlab that let me build some QA/QC KPI's like the number of bugs or user stories per sprint.

The logic is to get desired Gitlab data through the API Calls where the query results can be post-processed and automated into a Gsheet, which includes the possibility to easily create and embed easygoing charts.

In a previous post, was already reported an overview of the Google AppScript Dashboard [Notifications via Gsheet](https://blog.zuru.tech/scripting/testing/cd/ci/2022/01/20/daily-builds-send-discord-gmail-notifications-via-gsheet) where you can find its basic element like the editor, the debugger and the triggers used even in this tutorial.

## Gitlab API Calls
First of all, let's give a quick introduction to the API Calls.

The APIs (Application Programming Interfaces), are software-to-software interfaces which allow different applications to talk to each other and exchange information or functionalities. 

An API call is the process where a client application submits a request to an API made available by the Server; the data request performed is sent back to the client.

The Server exposes its API using an address called **URI** (Uniform Resource Identifier), defined as the unique sequence of characters that identifies a logical or physical resource used by web technologies (source: Wikipedia).


Once you have the URI, then you need to know how to formulate the request through a **"verb"**: The four most basic request verbs are:

- **GET:** To retrieve a resource
- **POST:** To create a new resource
- **PUT:** To edit or update an existing resource
- **DELETE:** To delete a resource

For our purposes we are going to use:

- **Google Sheet with Appscript** as our **Client**, where we perform the API Calls, using the **GET** verb only;
- **Gitlab** is the **Server**, that expose its API through the **URI** " https://gitlab.com/api"

## API Calls Testing: Postman (WIP)

## Google AppScript Script: Client API Call

Differently from the Gmail script one, a **Webhook** activation from the Discord side is needed.

Webhooks are user-defined HTTP callbacks that are triggered by specific events; whenever the trigger event occurs, the Discord API client sees the event, collects the data, and immediately sends a notification (HTTP request) to the Webhook URL specified in the application settings. These requests are known as webhooks or callbacks.

```javascript
function cri0510() {
  var url = "API URL";
  var headers = {
  "Authorization": "Autorization code to access in Gitlab",
  "Content-Type": "Application/txt"
  };
  var options = {
  "method" : "get",
  "headers" : headers
  };
  var response = UrlFetchApp.fetch(url, options);
  var fact = response.getContentText();
  var sheet = SpreadsheetApp.getActiveSpreadsheet().getSheetByName("SHEET_NAME");
  //sheet.getCurrentCell().setValue([fact]);
  sheet.getRange(2,3).setValue([fact]);
}
```

I've scheduled each functions to be triggered each hours, in order to refresh the data called by the API.

<div markdown="1" class="blog-image-container">
![triggers](/images/apicharts/triggers.png){:class="blog-image"}
</div>

#### Google Website Integration


To make the consultation of the charts more attractive and professional compared to a Simple Google Sheet, it is very easy to embed them into a Google Site.

Google Site is part of the Gsuite, and it's totally integrated with all the other apps.



<div class="blog-image-container" markdown="1">
![charts1](/images/apicharts/charts1.png){:class="blog-image"}
</div>


<div class="blog-image-container" markdown="1">
![charts2](/images/apicharts/charts2.png){:class="blog-image"}
</div>


<div class="blog-image-container" markdown="1">
![charts3](/images/apicharts/charts3.png){:class="blog-image"}
</div>


<div class="blog-image-container" markdown="1">
![stats](/images/apicharts/stats.png){:class="blog-image"}
</div>


## Potential Reuses

This process it requires a minimal technical knowledge and consequently some effort, but is really powerful, embeddable and free.

In my opinion it could be easily adopted by a PM and/or a Scrum Master, to build a costant up-to-date dashboard or to nourish their slides shown during the Agile cerimonies: a lot of time wasted getting manually the information can be avoided. 

---

**Resources**

[1] [Use the API](https://docs.gitlab.com/ee/api/)

[2] [Gitlab API Resources](https://docs.gitlab.com/ee/api/api_resources.html)

[3] [URI](https://en.wikipedia.org/wiki/Uniform_Resource_Identifier) 

[4] [Get started with Sites](https://support.google.com/a/users/answer/9310491?hl=en)


