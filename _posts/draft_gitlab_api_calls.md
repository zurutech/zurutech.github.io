## Introduction

<div class="blog-image-container" markdown="1">
![Intro](/images/apicharts/intro.png){:class="blog-image"}
</div>

Here we go again!

The target of this tutorial is to create Gitlab charts based on your needs, by exploiting the Gsheet functionality and Gitlab's API.

Gitlab is a powerful tool for software development, but is not designed for project management charts; this limit is circumventable
by using external plugins or by following the solution which I'm going to describe here.

My need was to build always updated QA/QC KPI charts, using some Gitlab data (like the number of bugs per sprint, or stats for severity bugs for each sprint).

The logic is to get Gitlab data through the API Calls where the query results can be post-processed automatically into a Gsheet, which includes the possibility to easily create and embed easygoing charts.


## API Overview
First of all, let's give a quick introduction to API Calls.

The APIs (Application Programming Interfaces), are software-to-software interfaces which allow different applications to talk to each other and exchange information or functionalities. 

An API call is a process where a client application submits a request to an API made available by the Server; the data request performed is sent back to the client.

The Server exposes its API using an address called **URI** (Uniform Resource Identifier), defined as the unique sequence of characters that identifies a logical or physical resource used by web technologies (source: Wikipedia).

Once you have the URI, then you can formulate the request through a **"verb"**, a command string that distinguishes a specific action you want to do with the invoked data on the server. The four most basic request verbs are:

- **GET:** To retrieve a resource;
- **POST:** To create a new resource;
- **PUT:** To edit or update an existing resource;
- **DELETE:** To delete a resource.

So, for the purpose of our target, we are going to use:

- **Google Sheet with Appscript** as our **Client**, through which we will use the **GET** verb only, as long as we want just to get data to nourish our charts;
- **Gitlab** is the **Server**, that expose its API through the **URI** " [https://gitlab.com/api](https://gitlab.example.com/api/v4)

## Gitlab API

Gitlab's API is a service that is reachable and usable only if the authorization server approves the client's request ([Git credential manager](https://docs.gitlab.com/ee/user/profile/account/two_factor_authentication.html#git-credential-manager)), which contains the security credentials for a login session and identifies the user, the user's groups, the user's privileges.

Among the various options, get a [**personal access token**](https://docs.gitlab.com/ee/user/profile/personal_access_tokens.html)  could be the easier way to integrate it in our API Calls.



## API Calls Testing: Postman (WIP)

<div class="blog-image-container" markdown="1">
![Intro](/images/apicharts/postman.png){:class="blog-image"}
</div>


## Google AppScript Script: Client API Call

```javascript
function cri0510() {
  var url = "API URL";
  var headers = {
  "Authorization": "Bearer Code",
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

[5] [Access Token](https://en.wikipedia.org/wiki/Access_token)


