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

Among the various options, get a [**Personal access token**](https://docs.gitlab.com/ee/user/profile/personal_access_tokens.html)  could be the easier way to integrate it in our API Calls.

Through the user's setting panel, Preferences section, is possible to generate the access token:

<div class="blog-image-container" markdown="1">
![Token](/images/apicharts/token.png){:class="blog-image"}
</div>

The encrypted string will be inserted directly into the script as **TOKEN KEY** and it works as bypass from the client and server, in order to get back the requested data.


## Write the API Calls

Thanks to the [Gitlab API Resources](https://docs.gitlab.com/ee/api/api_resources.html) it's easy to understand the query's syntax acknowledged by the server.

<div class="blog-image-container" markdown="1">
![Git Api issues](/images/apicharts/git_api_issues.png){:class="blog-image"}
</div>

For istance, in the above screenshot you can see the GET calls at "**Project**" level, relating to the **"issues"**

We can easily perform some API Calls Tests by using [Postman](https://web.postman.co/).

Postman allows to have your own, multiple workspaces, which allows to create a grouping of API calls.

<div class="blog-image-container" markdown="1">
![Post_workspace](/images/apicharts/post_workspace.png){:class="blog-image"}
</div>

Before creating the first API call, we need to add the token key (as explained in the previous paragraph); once you've generated the token key in Gitlab, you can add as "Bearer Token" in the "Authorization" Panel of Postman.

<div class="blog-image-container" markdown="1">
![api_auth](/images/apicharts/api_auth.png){:class="blog-image"}
</div>

Now we are ready to sent an API Call, let's try to get **all the issues in closed state**. If everything is properly set, postman will return the result in the bottom part of the UI.

<div class="blog-image-container" markdown="1">
![closed_issues](/images/apicharts/closed_issues.png){:class="blog-image"}
</div>

Over the inline way, when the queries include parameters, Postman recognize them as "KEY", and it allows the customization through the "VALUE" column.

<div class="blog-image-container" markdown="1">
![post_params](/images/apicharts/post_params.png){:class="blog-image"}
</div>

For mine target, it came to me essential the [Issue statistics](https://docs.gitlab.com/ee/api/issues_statistics.html) query.

Basically this call returns the sum data aggregation of  

<div class="blog-image-container" markdown="1">
![issue_stats](/images/apicharts/issue_stats.png){:class="blog-image"}
</div>

I let the reader to try more complex queries by following the Gitlab's documentation.

## Google AppScript Script: Client API Call

```javascript
function cri0510() {
  var url = "API CALL";
  var headers = {
  "Authorization": "TOKEN KEY"
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

Google Site is part of the Gsuite, and it's totally integrated with all the other apps; here below the result of the embedding in mine webpage:

<div class="blog-image-container" markdown="1">
![site_charts](/images/apicharts/site_charts.png){:class="blog-image"}
</div>

In this way the charts are readable from everyone in view only mode, costantly updated in background thanks to the scripts. 


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


