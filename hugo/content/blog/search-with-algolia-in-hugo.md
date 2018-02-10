---
title: Search with Algolia in Hugo
description: ''
date: 2018-02-06 09:40:26 +0000
authors: []
publishdate: 2017-12-07 04:00:00 +0000
expirydate: 2030-01-01 04:00:00 +0000
textline: dfgsadfgfdgsgdhsfjgsjhhfdsafsd
headline: ''
images: []
categories: []
tags: []
cta:
  headline: ''
  textline: ''
  calls_to_action: []
private: false
weight: ''
aliases: []
menu: []
draft: true

---
For this week on [_Frontend Friday_](/tags/frontend-friday/ "frontend friday tag")_,_ we'll be covering how to set up lightning ⚡️ fast search for your Hugo site using [Algolia](https://algolia.com), the SaaS (Search as a Service 😉 ) provider.

Algolia's self-proclaimed claim-to-fame is that they are_"the most reliable platform for building search into your business,"_ and honestly, it's hard to disagree. Forestry's search is powered by Algolia _(just try searching for Algolia in the search above!)._

## Table of Contents

1. [Why Algolia?](#why-algolia)
2. [Generating Your Search Index](#generating-your-search-index)
3. [Sending Your Search Index to Algolia](#sending-your-search-index-to-algolia)
4. [Updating Your Search Index with Serverless Functions](#updating-your-search-index-with-severless-functions)
5. [Next steps](#next-steps)

## 1) Why Algolia?

There are many search solutions for static sites out there. You can roll your own search using frontend Javascript with tools like [Lunr.js](https://lunrjs.com/) or [Fuse.js](http://fusejs.io/), set up powerful open-source search technology using [ElasticSearch](https://www.elastic.co/products/elasticsearch) or [Amazon CloudSearch](https://aws.amazon.com/cloudsearch/), or SaaS solutions like [Algolia](https://algolia.com).

So the question is, _what makes Algolia so great?_

When it comes down to it, the answer comes down to two factors:

* The goal of the JAMStack is to _eliminate_ server dependencies. Why add one just for search?
* Algolia offers a very generous [_free plan_](https://www.algolia.com/pricing) and performs up to [200x faster than open-source solutions like ElasticSearch](https://www.algolia.com/doc/faq/why/what-makes-algolia-different-than-elasticsearch-or-solr/).

### How Algolia Works

Algolia provides a REST API to query and update your search indices. All input and output is provided in JSON, making it extremely easy to use in frontend Javascript.

In order to create, update, and maintain an Algolia search index, you'll need to generate a valid JSON array of all of the content in your Hugo site.

We'll do that in the next step!

## 2) Generating Your Search Index

To get started with Algolia, the very first thing you'll need to do is [sign up](https://www.algolia.com/users/sign_up). Once that is out of the way, your next step is to generate your JSON search index.

With Hugo, we'll do this using the [custom output formats](https://gohugo.io/templates/output-formats/) feature, which allows us to output an existing document in a different format (in this case, JSON).

To get started, open up the config file (usually `config.toml`) in your favorite text editor.

{{% tip %}}

Don't have a Hugo site yet? Check out our [_Up & Running With Hugo_](/blog/up-and-running-with-hugo/) series to get started with the static site generator Hugo in less than 30 minutes!

{{% /tip %}}

Here, we'll add the Hugo configuration for your custom output formats.

    [outputFormats.Algolia]
    baseName = "algolia"
    isPlainText = true
    mediaType = "application/json"
    notAlternative = true
    
    [params.algolia]
    vars = ["title", "summary", "date", "publishdate", "expirydate", "permalink"]
    params = ["categories", "tags"]

In `\\\\\\\[outputFormats.Algolia\\\\\\\]`:

* `baseName` tells the output format how to look for the Hugo layout for this output format
* `isPlainText` tells the output format to use GoLang's plain text parser for the layout, preventing some automatic HTML formatting from ruining your JSON
* `mediaType` tells the output format what kind of file to output.
* `notAlternative` tells the output format not to be included when looping over the `.AlternativeOutputFormats` [page variable](https://gohugo.io/variables/page/#page-variables).

In `\\\\\\\[params.algolia\\\\\\\]`:

* `vars` sets the [page variables](https://gohugo.io/variables/page/) in which you want included in your index.
* `params` sets the [custom page params](https://gohugo.io/variables/page/#page-level-params) in which you want included in your index.

### Creating the JSON Template

The next step is to provide Hugo with the JSON template for your custom output format. To do this, we'll create a new layout to do this.

{{% tip %}}

In the example above, we set `baseName` to `algolia`, which tells Hugo to look for list templates with the algolia in the filename. For example, `list.agolia.json`, `taxonomy.algolia.json`.

{{% /tip %}}

Copy the contents below into `layouts/_default/list.algolia.json`

    {{/* Generates a valid Algolia search index */}}
    {{- $hits := slice -}}
    {{- $validVars := $.Site.Params.algolia.vars | default slice -}}
    {{- $validParams := $.Site.Params.algolia.params | default slice -}}
    {{- range $i, $hit := where (where .Data.Pages "Params.private" "ne" "true") "Draft" "ne" "true" -}}
      {{- $dot := . -}}
      {{/* Set the hit's objectID */}}
      {{- .Scratch.SetInMap $hit.File.Path "objectID" $hit.UniqueID -}}
      {{/* Include built-in page variables */}}
      {{- range $key, $var := $hit -}}
      	{{- if and not (eq $key "Params") (in $validVars (lower $key)) -}}
        	{{- .Scratch.SetInMap $hit.File.Path $key $var -}}
        {{- end -}}
      {{- end -}}
      {{/* Include custom page params */}}
      {{- range $key, $param := $hit.Params -}}
        {{- if in $validParams (lower $key) -}}
          {{- $dot.Scratch.SetInMap $hit.File.Path $key $param -}}
        {{- end -}}
      {{- end -}}
      {{- $.Scratch.SetInMap "hits" $hit.File.Path (.Scratch.Get $hit.File.Path) -}}
    {{- end -}}
    {{- jsonify ($.Scratch.GetSortedMapValues "hits") -}}

In this layout, we loop through all of the current page's children and do the following:

* Set the objectID of the Algolia indexes document using the `.UniqueID` [page variable](https://gohugo.io/variables/page/#page-variables).
* Loop through the document's built-in variables and add specific variables to the document.
* Loop through the document's custom _Front Matter_ params and add specific params to the document.

{{% tip %}}

The above layout will only add pages that do not have `private = true` or `draft = true` in their front matter. This makes it easy to exclude results that shouldn't be index, and prevents drafted content from being included.

{{% /tip %}}

### Outputting the Index

Now that we've created our custom output format, the layout for it, and configured the variables and page-level params we want included in the index, we now have to set up the site to actually create the JSON index!

We can do this in two ways, using the `outputs` [param](https://gohugo.io/templates/output-formats/#output-formats-for-pages):

1. Using site-wide `outputs` configuration to output indexes for all content of a specific type
2. Specifying the `outputs` on per-page basis, outputting the index for a specific page.

For the purposes of this guide, we'll do the former. Open up your config file one more time, and add the following:

    [outputs]
    home = ["HTML", "RSS", "Algolia"]

This configuration tells Hugo to output the HTML document, the RSS Feed, and an Algolia index for your site's homepage, which will contain every other page on your site. (That's perfect!)

In your built site, you'll now find a file called `algolia.json` in the root, which we can use to update your index in Algolia.

## 3) Sending Your Search Index to Algolia

The next step is sending your search index to Algolia. For this article, we'll be using a great NPM package to do this: [atomic-algolia](https://www.npmjs.com/package/atomic-algolia).

{{% tip %}}

[atomic-algolia](https://www.npmjs.com/package/atomic-algolia) is an NPM package that does _atomic_ updates to an Algolia index. This means that it only updates changed records, adds new records, or deletes expired records, and does it all at once, so that your index is never out-of-sync with your website's content.

This is important, because Algolia's plans are based on _operations_ on your index, and _searches_ on the index, and this plugin ensures you use the _smallest amount of operations possible!_

{{% /tip %}}

To get started, make sure you have Node installed. If you don't, you can do so by [downloading an installer for your operating system](https://nodejs.org/en/download/).

### Create Your Index

Head over to your [Algolia app dashboard](https://www.algolia.com/manage/applications), and click _New Application._

![](/uploads/2018/02/Screen Shot 2018-02-06 at 3.33.12 PM.png)Set the application name to something memorable (i.e, your company name), and choose _Community_ as your plan.

![](/uploads/2018/02/Screen Shot 2018-02-06 at 3.38.01 PM.png)

Then, choose the region closest to you. _(In our case, that's Canada. 🇨🇦 )_

![](/uploads/2018/02/Screen Shot 2018-02-06 at 3.35.45 PM.png)You'll be redirected to the app's dashboard. Select the _Indices_ tab on the left, and then click _Add New Index._ Give it a unique name, (.ie, your site's domain), as this is what we'll use when updating the index.

![](/uploads/2018/02/Screen Shot 2018-02-06 at 3.36.18 PM.png)

Finally, select the _API Keys_ tab on the left, and copy the _Application ID_ and _Admin API Key,_ as we'll need these to update the index.

### Update Your Index

Now that you have an index, open up your terminal, navigate to your Hugo project, and run the following command:

    npm install atomic-algolia --save

This will install the atomic-algolia package to a local `node_modules` folder and make it available for use in your Hugo project.

Next, open up the newly created `package.json`, where we'll add an NPM script to update your index. Find `"scripts"`, and add the following:

    "algolia": "atomic-algolia"

Now, you can update your index by running the following command:

    ALGOLIA_APP_ID={{ YOUR_APP_ID }} ALGOLIA_ADMIN_KEY={{ YOUR_ADMIN_KEY }} ALGOLIA_INDEX={{ YOUR_INDEX NAME }} ALGOLIA_FILE_PATH={{ PATH/TO/algolia.json }} npm run algolia

{{% tip %}}

The path to the index file in the _Hugo Boilerplate_ is `dist/algolia.json`, whereas the path in a default hugo site is `public/algolia.json`

{{% /tip %}}

### Using a `.env` File

Passing in the environment variables to the NPM script each time you call it isn't ideal. That's why atomic-algolia supports a .env file.

Create a new file in the root of your Hugo project called `.env`, and add the following contents:

    ALGOLIA_APP_ID={{ YOUR_APP_ID }}
    ALGOLIA_ADMIN_KEY={{ YOUR_ADMIN_KEY }}
    ALGOLIA_INDEX={{ YOUR_INDEX NAME }}
    ALGOLIA_FILE_PATH={{ PATH/TO/algolia.json }}

Now you can update your index more simply by running:

    npm run algolia

## 4) Updating Your Search Index with Serverless Functions

Having to run the NPM script manually each time your site changes isn't ideal, especially when using services like Forestry.

That's why we've created an [open-source template](https://github.com/forestryio-templates/serverless-atomic-algolia) for creating a Serverless [Webtask Function](https://webtask.io/) that can automatically update your Algolia index each time your site is updated using web hooks.

### Setting Up

To get started, clone the [template repository](https://github.com/forestryio-templates/serverless-atomic-algolia) to your local machine by running:

    git clone https://github.com/forestryio-templates/serverless-atomic-algolia.git

{{% tip %}}

Don't have or want to use Git? Feel free to [download the template](https://github.com/forestryio-templates/serverless-atomic-algolia/archive/master.zip) as a zip instead.

{{% /tip %}}

Then, navigate to the template directory and install the dependencies:

    cd serverless-atomic-algolia
    npm install serverless -g && npm install

### Configuring

Next, if you don't already have a Webtasks profile set up, you'll need to do so. This can be done directly from the command line.

    serverless config credentials --provider webtasks

You will be asked for a phone number or email. You'll immediately receive a verification code. Enter the verification code and your profile will be entirely setup and ready to use

Next, you'll need to configure the function with your indices and Algolia app information.

First, copy `config/secrets.yml.stub` to `config/secrets.yml` and then open it up in your favorite text editor.

    ALGOLIA_APP_ID: {{ YOUR_APP_ID }}
    ALGOLIA_ADMIN_KEY: {{ YOUR_ADMIN_KEY }}
    DEBOUNCE: 0

Then, open up `config/index.js` in your favorite text editor:

    module.exports = () => {
    
      var indexes = [
    
        {
    
          name: "YOUR_INDEX_NAME",
    
          url: "PUBLIC_URL_OF_INDEX"
    
        }
    
      ]
    
      return JSON.stringify(indexes)
    
    }

Update `name` to the name of your index that you set up earlier, and `url` to `yourdomain.com/algola.json`, replacing `yourdomain.com`with your site's domain.

### Deploying the Function

Now we can deploy the function by running:

    serverless deploy

In the terminal, you'll receive an output for the success of your deployment, including the public URL for your new function. Copy that to the clipboard, as this is the URL we'll trigger with a web hook when changes are made to the site.

### Setting Up A Webhook

Finally, our last step is to set up a post-deployment web hook with your deployment tool. Set up for each tool will be different, but we'll provide setup for:

* Forestry.io

#### Forestry.io

![](/uploads/2018/01/settings-webhook.png)

Head over to the _Settings_ page of your site in Forestry, and scroll down to the _Webhook URL_ setting.

Enter the URL you received when deploying your function, and then click _Save Settings_.

Now each time Forestry finishes deploying your site, your function will be invoked to update your Algolia index.

{{% tip %}}

This set up only works when Forestry is set up to handle the build and deployment of your site. If you're using a third party CI service to build your site (like GitLab CI or Netlify), you will need to use their webhook features to trigger your function.

{{% /tip %}}

## 5) Next Steps

Now that you've successfully set up search indexing for your static site, it's time to add the actual search interface to your website!

Algolia has a fantastic library called [InstantSearch.js](https://community.algolia.com/instantsearch.js/) for implementing search on the web, and provides a [full tutorial](https://community.algolia.com/instantsearch.js/v2/getting-started.html) for implementing search from scratch.

### Next Week

TK: todo

### Last Week

TK: todo