# Implementing an RSS Feed with Node.js 

## Introduction

This tutorial explains how to write a Node.js application that connects with the CoreMedia Headless Server.
The result is a webapp that delivers an RSS feed with selected CoreMedia teaser documents.

## Prerequisites

- Make sure that you have Node.js installed (this tutorial was developed with Node.js 12.16).
- Make sure that you have npm installed.
- Make sure that you have access to a CoreMedia system that includes a Headless Server.

## Implementation Steps

### Step 1: Project Setup

For starting the development, you have to set up a new project first. For Node.js this is rather simple:

1. Create a new folder called _headless-server-rss-feed_.
2. Create a new file called _package.json_ within it, and paste the following content into it:

    ``` json
    {
      "name": "headless-server-rss-feed",
      "version": "1.0.0",
      "description": "A sample application for the CoreMedia Headless Server",
      "main": "index.js",
      "scripts": {
        "start": "node index.js",
        "test": "echo \"Error: no test specified\" && exit 1"
      },
      "repository": {
        "type": "git",
        "url": "https://github.com/CoreMedia/headless-server-tutorial"
      },
      "keywords": [],
      "author": "",
      "license": "CoreMedia Open Source License"
    }
    ```
3. Create a second file called _index.js_. This is the main executable of the example. 
Start with configuring the URL of the Headless Server. 

4. Paste the following snippet into your _index.js_ file, 
and replace the _<YOUR_HEADLESS_SERVER_BASE_URL>_ with the base URL of your Headless Server.
You can either use the preview or the live URL of the Headless Server. A CoreMedia installation comes with an overview page that contains all important URLs of the system.
If you are not sure what the Headless Server's URL is, you can also look it up there. Take the URL that points to
GraphiQL UI, and remove the _graphiql_ segment. 

```javascript
var headlessServerUrl = "https://<YOUR_HEADLESS_SERVER_BASE_URL>/";
console.log("Using headless server: " + headlessServerUrl);
```

You can now open a command line tool and execute the script with `npm start`.
 It will simply print the `Using headless server: ...` message.


### Step 2: Build a GraphQL Query for the CoreMedia Headless Server

The CoreMedia system overview page also has a _GraphiQL_ link where you can run GraphQL queries against the Headless Server in your browser:

1. Open the URL _https://<YOUR_HEADLESS_SERVER_BASE_URL>/graphiql_
2. Paste the following query example into the query section of the website, and press the execute button in the toolbar.

```
query {
  content {
    sites {
      name,
      id
    }
  }
}
```  

As a result you will get a list of all available sites on your system, including their IDs.
Note the _id_ of the site you want to use; you need it for the next query. Since you want to create an RSS feed,
you need the title, teaser text and a picture (if available) of all recently modified articles.
The following snippet shows an example query for this using the "Chef Corp. (US)" site id _abffe57734feeee_:

```
{
  content {
    search(query: "*", offset: 0, limit: 10, docTypes: ["CMArticle"], sortFields: [MODIFICATION_DATE_ASC], siteId: "abffe57734feeee") {
      numFound
      result {
        ... on CMArticle {
          creationDate, 
          title,           
          teaserText,
          remoteLink,
          picture {            
            data { 
              size ,
              contentType
            },
            uriTemplate,
            crops {
              name,
              minWidth              
            }          
          }
        }
      }
    }
  }
}
```

In this example you are using the search query endpoint of the Headless Server.
The search query limits the result to 10 items and filters for the content type _CMArticle_ and the site with ID _abffe57734feeee_.

Using the `... on CMArticle` expression, the search result expands every _CMArticle_ result, using the fields _creationDate_,
_title_, _teaserText_ and _remoteLink_.

Additionally, the picture is queried with its data. It contains the available crops, minimum sizes, mime type and size which you need for the RSS feed.

A single item of the result might look like this:

```json
  {
    "creationDate": "2020-02-04T15:56:34Z[GMT]",
    "title": "Delivery",
    "teaserText": "<div><p>We are committed to delivering and installing expert kitchen and restaurant solutions, with the personal attention you deserve.</p></div>",
    "remoteLink": "//<YOUR_HEADLESS_SERVER_BASE_URL>/blueprint/servlet/corporate/for-professionals/services/delivery-7730",
    "picture": {
      "data": {
        "size": 303789,
        "contentType": "image/jpeg"
      },
      "uriTemplate": "/caas/v1/media/7116/data/1db08142083c869d9d4e7e44d909275a/{cropName}/{width}",
      "crops": [
        {
          "name": "landscape_ratio16x9",
          "minWidth": 400
        },
        ...
      ]
    }
  }
```



### Step 3: Execute GraphQL Query

Next, query the data using the Node.js program. For requesting the data, you have to install
the _node-fetch_ module, which supports promises and is, therefore, a little bit more comfortable than the standard HTTP post
provided by Node.js:

```
npm install node-fetch
```

Once you have installed _node-fetch_, add the following statement at the beginning of your _index.js_:

```
var fetch = require("node-fetch");
```

You can now implement the fetch call. Note that GraphQL queries are _POST_ requests with MIME type _application/json_.

```javascript
function fetchFeedData() {
  return fetch(headlessServerUrl + 'graphql', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Accept': 'application/json',
    },
    body: JSON.stringify({
      query: "{\n" +
              "  content {\n" +
              "    search(query: \"*\", offset: 0, limit: 10, docTypes: [\"CMArticle\"], sortFields: [MODIFICATION_DATE_ASC], siteId: \"abffe57734feeee\") {\n" +
              "      numFound\n" +
              "      result {\n" +
              "        ... on CMArticle {\n" +
              "          creationDate, \n" +
              "          title,           \n" +
              "          teaserText,\n" +
              "          remoteLink,\n" +
              "          picture {            \n" +
              "            data { \n" +
              "              size ,\n" +
              "              contentType\n" +
              "            },\n" +
              "            uriTemplate,\n" +
              "            crops {\n" +
              "              name,\n" +
              "              minWidth              \n" +
              "            },            \n" +
              "          }\n" +
              "        }\n" +
              "      }\n" +
              "    }\n" +
              "  }\n" +
              "}"
    })
  })
          .then(r => r.json())
          .then(data => {
            console.log('data returned:', data.data.content.search.result.length);
            return data.data.content.search.result;
          });
}

fetchFeedData();
```

Execute the program with _npm start_. You should see the following output (assuming you have at least 10 _CMArticle_ documents for your site):

```
> headless-server-rss-feed@1.0.0 start C:\workspace\headless-server-tutorial\headless-server-rss-feed
> node index.js

data returned: 10
```





### Step 4: Convert Response JSON to RSS Feed XML

For creating an RSS feed, you need to convert the response JSON to XML.
Use the library _rss_ (https://www.npmjs.com/package/rss) for this:

```
npm install rss
```

After adding the RSS library at the beginning of your _index.js_

```
var RSS = require('rss');
```

you can write the function that converts the JSON object into RSS XML.
Append the following snippet to the end of your _index.js_:

```javascript
function toFeedXML(data) {
  //create the feed itself
  var feed = new RSS({
    title: 'Headless RSS Feed',
    description: 'RSS feed build from CoreMedia Headless Server data',
    feed_url: 'http://localhost:8080',
    language: 'en'
  });

  //iterate over the article records and create feed items
  for (let i = 0; i < data.length; i++) {
    var entry = data[i];
    feed.item({
      title:  entry.title,
      description: entry.teaserText,
      date: entry.creationDate.substr(0, 19),
      url: 'https:' + entry.remoteLink,
      enclosure: {
        'url'  : formatPictureUrl(entry.picture),
        'type' : 'image/jpeg'
      }
    });
  }

  //return the RSS XML
  return feed.xml();
}

//helper for formatting image URLs
function formatPictureUrl(picture) {
  let url = picture.uriTemplate.replace('{cropName}', picture.crops[0].name).replace('{width}', picture.crops[0].minWidth);
  return headlessServerUrl + url;
}
```

The helper function, _formatPictureUrl_, ensures that the crop variant with its minimum width is formatted correctly.
For the feed, simply use the first variant.

### Step 5: Implementing a Server

As a final step, you need a webserver that serves the RSS feed XML. In Node.js, this is pretty simple to write.
First, you need the Node.js _http_ package. Add the following line at the beginning of your _index.js_:

```javascript
var http = require('http');
```

Then create and start a server that serves the generated XML:

```javascript
http.createServer(function (req, res) {
  fetchFeedData().then(data => {
    let xml = toFeedXML(data);
    res.writeHead(200, {'Content-Type': 'application/rss+xml'});
    res.end(xml);
  });
}).listen(8081);
console.log("Started HTTP Server (http://localhost:8081)");
```

After these changes, your RSS feed is finished. You can restart the server with `npm start` and invoke the logged URL in your browser.
Assuming you have an RSS reader plugin installed in your browser, the result might look similar to this:

![Example Feed](../images/feed.png "Example Feed")
 
