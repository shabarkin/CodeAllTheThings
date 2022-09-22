**Authors: [Pavel Shabarkin](https://www.linkedin.com/in/pavelshabarkin/)**

**Disclaimer: It is still under research, this guide can be extended.**

# Security Review for Meteor JS applications
- [Security Checklist](#security-checklist)
- [Review Methodology](#review-methodology)
	- [Getting Started](#getting-started)
		- [Setup](#setup)
		- [Identify Meteor JS file](#identify-meteor-js-file)
- [API keys and credentials leak](#api-keys-and-credentials-leak)
	- [Shared source code](#shared-source-code)
	- [`Meteor.settings` does not leak credentials](#meteorsettings-does-not-leak-credentials)
	- [Client-side mongo collections does not leak app creds](#client-side-mongo-collections-does-not-leak-app-creds)
- [Missing authentication](#missing-authentication)
- [Missing authorization](#missing-authorization)
	- [Collections direct interaction](#collections-direct-interaction) 
	- [Meteor methods](#meteor-methods)
	- [Meteor sub/pub](#meteor-subpub)
- [NoSQL Injections](#nosql-injections)
- [Cross Site Scripting (XSS) attacks](#cross-site-scripting-xss-attacks)
- [Links and References](#links-and-references)


# Security Checklist

- [ ]  The `insecure` or `autopublish` modes are not used
- [ ]  API keys, secrets, and credentials are not in the code base, client- and server-side code can be shared
- [ ]  API keys, secrets, and credentials are not in the `Meteor.settings` object
- [ ]  API keys, secrets, and credentials of the app are not in the public Mongo collections
- [ ]  Meteor Iron has implemented authentication and authorization mechanisms. Look for `Router\.route\(`.
- [ ]  Collections cannot be updated from the client-side, look for `\.allow\(` pattern within code base
- [ ]  Collections which defined `allow` rule, have `deny` rule for rest of the actions.
- [ ]  The application denies all updates to user profile, look for `Meteor\.users\.deny\(` within code base
- [ ]  The user IDs are not user controllable in the arguments of Meteor methods and publications, they should validate the the user’s id from this instruction only: `this\.userId`
- [ ]  The user IDs are not user controllable by hijacking the Meteor session `userId`, they should validate a user’s id from this instruction only: `this\.userId`
- [ ]  The Meteor publications had implemented authorization access control in the returned query and not within the publication function.
- [ ]  Selectors and [filter fields](http://guide.meteor.com/security.html#fields) are applied in publications before returning the data.
- [ ]  The application does not allow executing arbitrary NoSQL selectors coming from client-side, as a best practice they should not be user controlled at all.
- [ ]  The raw HTML inclusion is not used in Blaze, look for triple mustache: `{{{`
- [ ]  Set up secure [HTTP headers](https://guide.meteor.com/security.html#httpheaders) using [Helmet](https://www.npmjs.com/package/helmet), not all browsers support it so it provides an extra layer of security to users with modern browsers.

# Review Methodology

## Getting started

### Setup

When you use `Meteor.method`*,* `Meteor.call`, `Meteor.publish` and `Meteor.subscribe`, Meteor is using DDP underneath. In short, DDP is a JSON based protocol which Meteor had implemented on top of **SockJS**. It serves to create a full duplex communication between client and server where it can change data and react to its changes.

[Users and Accounts | Meteor Guide](https://guide.meteor.com/accounts.html#userid-ddp)

It is possible to find the DDP connection created by Meteor on the client-side by accessing a variable called **`_stream`**, on the Meteor global object. You need to run the following snippet in the browser console to track everything going through the DDP and easy debug and test Meteor communication between client- and server- sides:

```jsx
var oldSend = Meteor.connection._stream.send;

Meteor.connection._stream.send = function() {
  oldSend.apply(this, arguments);
  console.log(arguments[0]);
};

Meteor.connection._stream.on('message', function (msg) {
  console.log(msg);
});
```

[Hacking Meteor DDP | HackerNoon](https://hackernoon.com/hacking-meteor-ddp-9da37790b37b)

### Identify Meteor JS file

Meteor is a full-stack framework for building JavaScript applications. This means Meteor applications differ from most applications in that they include code that runs on the client, inside a web browser or Cordova mobile app, code that runs on the server, inside a Node.js container, and common code that runs in both environments.

There is 2 ways to review the source code of the Meteor JS: 

1. Open user’s browser, then developer tools and search for JS bundles
2. When a Meteor application is built for deployment with meteor build, all JavaScript files and templates are packaged and minimized into a single file. This can also be emulated with meteor run `--production`. We can see the minimized code when we look at the source of a built Meteor application.

> **Never use  `--production` flag to deploy!**
> 
> - `--production` flag is purely meant to simulate production minification, but does almost nothing else. This still watches source code files, exchanges data with package server and does a lot more than just running the app, leading to unnecessary computing resource wasting and security issues. Please don’t use `--production` flag to deploy!

Main HTML file of application may include similar JS file to load the application based on the Meteor JS: 

```jsx
<script type="text/javascript" src="https://domain.com/ab931b030c581324ksdc123d141b1244e470b6e6.js?meteor_js_resource=true"></script>
```

# API keys and credentials leak

## Shared source code

It is not rare to find hardcoded credentials, API keys, and tokens within the Meteor code base. Meteor JS may use the same JS file for server-side and client-side. However, Meteor JS has an option to hide server side code, the developers should put the server-side code into the `server/` folder, then it will not be accessible from client-side.

The extensive list of API calls prepared to verify and validate all the identified API credentials:

[https://github.com/streaak/keyhacks](https://github.com/streaak/keyhacks)

## `Meteor.settings` does not leak credentials

Another source of potential credential leak is the `Meteor.settings` object

> In most normal situations, API keys from your settings file will only be used by the server, and by default the data passed in through `--settings`
is only available on the server. However, if you put data under a special key called `public`, it will be available on the client. You might want to do this if, for example, you need to make an API call from the client and are OK with users knowing that key. Public settings will be available on the client under `Meteor.settings.public`.
> 

Open the browser console and run the `Meteor.settings` command, review for any sensitive information

## Client-side mongo collections does not leak app creds

Open the browser console and run the `Mongo.Collection.getAll()` command, then review Mongo collections for secrets, credentials, API keys, tokens and more of the same ... 

# Missing authentication

Having access to the source code (server-side code), review REST API endpoints defined by Meteor Iron, look for `Router\.route\(` pattern. Those API endpoint may miss authentication, work with, and return high-sensitive information:

**#examples:** 

```jsx
Router.route('/api/pull/:userId/', {
  where: 'server'
}).get(function() {

**// there is no middleware authentication check, so the following instructions are run without checking authentication:** 

	const userId = this.params.userId;
  let result = Meteor.users.find().fetch();

  this.response.writeHead(200, { 'Content-Type': 'application/json' });
  this.response.end(JSON.stringify(result));
}

Router.route('/api/push/:userId/:accessToken', {
  where: 'server'
}).get(function() { ... }

Router.route('/api/import/:userId/:fileId', {
  where: 'server'
}).post(function() { ... }
```

# Missing authorization

Extra important to determine what is run on the server only and what can be run on the client. The Meteor applications have the following attack surface in terms of authorization:

1. **Methods**: Any data that comes in through Method arguments needs to be validated, and Methods should not return data the user shouldn’t have access to.
2. **Publications**: Any data that comes in through publication arguments needs to be validated, and publications should not return data the user shouldn’t have access to.
3. **Served files**: You should make sure none of the source code or configuration files served to the client have secret data.

## Collections direct interaction

Since Meteor apps have architecture that puts client and server code together. In some cases, client can access Mongo Collections directly. If the Mongo Collection has an `allow` definition on the collection actions on the server-side, it will be accessible from the client side. There is several potential problems, the application can have:

**#allow rule**

The application allows performing actions on other user’s records. Any user would be able to update any record in the `OAuth` collection:

```jsx
OAuth.allow({
	update(userId, doc, fields, modifier) {
		// Can change any OAuth records.
    return userId
  },
});
```

But the ability to update any record can be stricted this way:

```jsx
OAuth.allow({
update(userId, doc, fields, modifier) {
    // Can only change your own OAuth records.
    return doc.owner === userId;
  },
```

> If you never set up any `allow` rules on a collection then all client writes to the collection will be denied, and it will only be possible to write to the collection from server-side code.
> 

However, if the Meteor app is run in `insecure` mode, and the application did not set up any `allow` or `deny` rules on a collection, then all users have full write access to the collection.

**New Meteor projects start in insecure mode by default.** To turn it off just run in your terminal: `meteor remove insecure`

**#deny rule**

The `deny` rule ensures that action on the collection is denied. When a client tries to write to a collection, the Meteor server first checks the collection’s `deny` rules. If none of them return `true` then it checks the collection’s `allow` rules. Meteor allows the write only if no `deny` rules return `true` and at least one `allow` rule returns `true`.

What means if application define at least one `allow` rule, and other rules are not redefined with `deny`, the client side will able to work with any actions on the collection.

**#profile editing** 

User profile editing should be always disabled: 

```jsx
Meteor.users.deny({
  update: function () {
      return true;
  }
});
```

Look for `\.deny\(` and `\.allow\(` and determine how the application defined rules’ logic accordingly to this paper, [https://guide.meteor.com/security.html#allow-deny](https://guide.meteor.com/security.html#allow-deny), and [https://docs.meteor.com/api/collections.html#Mongo-Collection-deny](https://docs.meteor.com/api/collections.html#Mongo-Collection-deny). 

The general recommendation to avoid using `allow` and `deny` rules, and switch to the Meteor methods instead. This will reduce the risk that all collection operations will be accessible to the client, if the app is deployed in the `insecure` mode. And it’s easier to maintain and scale the application. But those Meteor method should have a solid implementation of the authorization mechanism.

## Meteor methods

Meteor methods are remote call procedure system in other words an interface to interact with a server-side. If to compare with the REST API architecture, an Meteor methods are something like GET, POST requests. The main attack surface against Meteor methods is that applications do not have proper implementation of authorization access controls. 

Meteor methods can be defined in the following methods: 

1. **Basic method** is defined such as a general function in JavaScript, [https://guide.meteor.com/methods.html#basic](https://guide.meteor.com/methods.html#basic)
2. ****Advanced Method boilerplate**** is defined with a wrapper on the object level, [https://guide.meteor.com/methods.html#advanced-boilerplate](https://guide.meteor.com/methods.html#advanced-boilerplate)
3. ****Advanced Methods with mdg:validated-method**** is defined with a wrapper on the package level, [https://guide.meteor.com/methods.html#validated-method](https://guide.meteor.com/methods.html#validated-method)

There is several patterns to map and identify those methods. If you have the full source code you may target, grep, and look for the following patterns: 

1. Grep `Meteor\.methods\(\{`, each `function_name:` inside the `Meteor.methods` is a defined Meteor Method
2. Grep `ValidatedMethod\(\{`, each `name: ‘function.name’` inside the `ValidatedMethod` is a defined Meteor Method

If you have the shared source code and don’t have access to the hidden code located on the server, you can identify the app.js file and grep for their `Meteor\.call\(` instructions. 

There is several examples of poor implementation of authorization access controls. 

1. Never pass `userId` as an argument of the method, `this.userId` is not user controllable, it works underneath of meteor DDP and cannot be changed to arbitrary `userId`:
    
    ```jsx
    // #1: Bad! The client could pass any user ID and set someone else's name
    setName({ userId, newName }) {
      Meteor.users.update(userId, {
        $set: { name: newName }
      });
    }
    
    // #2: Good, the client can only set the name on the currently logged in user
    setName({ newName }) {
      Meteor.users.update(this.userId, {
        $set: { name: newName }
      });
    }
    ```
    
    The following rule applies for all Mongo collections, not only for the user profile collection. If the Meteor method does not check the legitimate owner of the record before executing any operation over it, the application will allow arbitrary users to work with records of other app users.
    
2. But the `Meteor.userId()` is user controlled! A user can fool the application by hijacking the `userId` of the session connection, the authorization within Meteor methods will be bypassed, because they rely on the returned value of the `Meteor.userId()` function:
    
    ```jsx
    // #1: Bad! The client could pass any user ID by hijacking the session userId and set someone else's name
    setName({ newName }) {
      Meteor.users.update(Meteor.userId(), {
        $set: { name: newName }
      });
    }
    
    // #2: Good, the client can only set the name on the currently logged in user
    setName({ newName }) {
      Meteor.users.update(this.userId, {
        $set: { name: newName }
      });
    }
    ```
    
    To hijack the `userId`, run the following command in the browser console: 
    
    ```jsx
    Meteor.connection.setUserId("USER_ID");
    ```
    

## Meteor sub/pub

The Meteor subscription and publishing mechanisms are actually the way Meteor server- and client-sides interact and pass data to each other. Through publications Meteor server makes data available to a client. The main security issue of the publishing mechanism that Meteor server can return unintended data to a malicious user.

The general rules prepared for Meteor Methods should be also applied for the publications:

1. Validate all arguments using `check` or npm `simpl-schema`.
2. Never pass the current user ID as an argument, and never rely on `Meteor.userId()` object.
3. Use rate limiting to stop people from spamming you with subscriptions.

**#exposing of secret data**

The functionality of the publications opened up additional attack vectors. All the Mongo collections have the `fields` option that restricts a collection returning only requested records. It prevents the accidental publishing of secret data.

```jsx
// #1: Bad! The application will return the OAuth access and refresh tokens to the client-side
Meteor.publish('oauth.public', function () {
  return OAuth.find({userId: {$exists: false}});
});

// #2: Good, The application will return only public OAuth information 
Meteor.publish('oauth.public', function () {
  return OAuth.find({userId: {$exists: false}}, {
    fields: {
      name: 1,
      IntegrationName: 1,
      userId: 1
    }
  });
});
```

If the `Meteor\.publish\(` functions missed `fields:` option, it may leak secret data all depends on the data stored in the collection. Review all the  `Meteor\.publish\(` functions and determine functions which return entire record or does not exclude secret data to be returned to the client.

The `fields:` option should be declared on the server-side only, it should never be user controlled!

**#authorization**

Authorization concepts of Meteor publications are similar to the Meteor methods. The `Meteor.userId()` is user controlled! A user can fool the application by hijacking the `userId` of the session connection, the authorization within Meteor publications will be bypassed, because they rely on the returned value of the `Meteor.userId()` function:

```jsx
// #1: Bad! The client could pass any user ID by hijacking the session userId and set someone else's name
Meteor.publish('oauth', function (oauthId) {

  const oauth = OAuth.findOne(oauthId);

  if (oauth.userId !== Meteor.userId()) {
    throw new Meteor.Error('oauth.unauthorized',
      'This oauth doesn\'t belong to you.');
  }

  return OAuth.find(oauthId);
});

// #2: Good, the client can only set the name on the currently logged in user
Meteor.publish('oauth', function (oauthId) {

  const oauth = OAuth.findOne(oauthId);

  if (oauth.userId !== this.userId) {
    throw new Meteor.Error('oauth.unauthorized',
      'This oauth doesn\'t belong to you.');
  }

  return OAuth.find(oauthId);
});
```

To hijack the `userId`, run the following command in the browser console: 

```jsx
Meteor.connection.setUserId("USER_ID");
```

But there is one hidden security concern that you need to know about publications. Publications are not reactive, and they only re-run when the currently logged in `userId` changes. The data returned from publications will often be dependent on the currently logged in user, and perhaps some properties about that user - whether they are an admin, whether they own a certain document, etc. 

```jsx
// #1: Bad! If the owner of the list changes, the old owner will still see it
Meteor.publish('list', function (listId) {
  check(listId, String);

  const list = Lists.findOne(listId);

  if (list.userId !== this.userId) {
    throw new Meteor.Error('list.unauthorized',
      'This list doesn\'t belong to you.');
  }

  return Lists.find(listId, {
    fields: {
      name: 1,
      incompleteCount: 1,
      userId: 1
    }
  });
});

// #2: Good! When the owner of the list changes, the old owner won't see it anymore
Meteor.publish('list', function (listId) {
  check(listId, String);

  return Lists.find({
    _id: listId,
    userId: this.userId
  }, {
    fields: {
      name: 1,
      incompleteCount: 1,
      userId: 1
    }
  });
});
```

If the changes were applied to the collection but user remained their session active, they will still be able to access the data returned by publication because the publication’s callback function is run only when the `userId` of the user changes (only the `return` instruction is re-run). So the authorization check should be implemented on the query itself:

```jsx
return Lists.find({
    _id: listId,
    userId: this.userId
  }, {
    fields: {
      name: 1,
      incompleteCount: 1,
      userId: 1
    }
  });
```

This authorization issue is limited by the time and is valid till user logs out.

# NoSQL Injections

App users should never have ability to control the “search” parameters (filters, selectors, etc ...) of the NoSQL queries within the Meteor methods and publications. Beyond authorization issues the app could be exposed to NoSQL injections.

Example of the vulnerable Meteor method which allows user to control NoSql query parameter:

```jsx
// Bad! User can perform NoSQL injection and retrieve password hashes, user's app tokens by controlling the query search. For example: the ////"token": {$regex: '^[a-z].*'} // start's with [a-z] (26 possibilities) //// NoSql query parameter will allow a user to enumerate tokens of other users:
Meteor.methods({
'users.count'({ filter }) {
	return Meteor.users.find(filter).count();
}});
```

Example of the client-side call:

```jsx
Meteor.call("users.count", {
		"username": "kertojasoo",
		"token": {$regex: '^[a-z].*'} // start's with [a-z] (26 possibilities)
		}, console.log);
```

# Cross Site Scripting (XSS) attacks

Meteor renders the content on the client-side, the application should have data sanitization in the Meteor methods to prevent basic XSS attack vectors. Meteor apps use Blaze framework for HTML rendering if user data is inserted into the raw HTML inclusion (triple mustache `{{{`), it is possible to achieve a XSS attack.

# Links and References
<details><summary>Links and References</summary><blockquote>

1. [Hacking meteor ddp](https://hackernoon.com/hacking-meteor-ddp-9da37790b37b)
2. [Guide meteor userid-ddp](https://guide.meteor.com/accounts.html#userid-ddp)
3. [Guide meteor structure](https://guide.meteor.com/structure.html)
4. [Guide meteor client-settings](https://guide.meteor.com/security.html#client-settings)
5. [Guide meteor attack-surface](https://guide.meteor.com/security.html#attack-surface)
6. [Docs meteor collections Mongo-Collection-deny](https://docs.meteor.com/api/collections.html#Mongo-Collection-deny)
7. [Guide meteor security allow-deny](https://guide.meteor.com/security.html#allow-deny)
8. [Wekan Authentication Bypass - Exploiting Common Pitfalls of MeteorJS | Offensive Security](https://www.offensive-security.com/offsec/wekan-authentication-bypass/)
</blockquote></details>
