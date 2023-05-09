# Ruby on Rails

Authors: [Viktoriia Taran](https://www.linkedin.com/in/viktoriia-taran/), [Rostyslav Grynov](https://www.linkedin.com/in/rostyslav-grynov/)

<aside>
🤓 This guide will be constantly updated with new vulnerabilities and their patterns. Feel free to add something new.

</aside>

> Rails is a web application development framework written in the Ruby programming language. Rendering HTML templates, updating databases, sending and receiving emails, maintaining live pages via WebSockets, enqueuing jobs for asynchronous work, storing uploads in the cloud. This framework has many built-in security options, but they can be disabled by developers :)
> 

# Security Checklist

- [ ]  The source code does not rely on hard-coded credentials and secrets.
- [ ]  The application doesn’t use HTML characters escape like `raw`, `html_safe`, `content_tag` etc. The application doesn’t have security flag set to `ActiveSupport::escape_html_entities_in_json = false` , which may lead to XSS when using `to_json()` method.
- [ ]  The `protect_from_forgery` setting is defined in the controllers, the `<%= csrf_meta_tags %>` setting is defined in the HTML templates.
- [ ]  Users have no direct control over rendering ERB templates.
- [ ]  The application validates user input and doesn’t send it to the dangerous functions such as `eval`.
- [ ]  The application uses parameterization instead of concatenation when crafting SQL queries.
- [ ]  The application uses `config.force_ssl = true`. Ensure that application uses strong hash algorithm for cookies signatures in the `Rails.application.config.action_dispatch.signed_cookie_digest = "SHA256"` setting. Environments must use a random key present in `config/credentials.yml.enc` and key must be encrypted.
- [ ]  User controlled URLs are not permitted to request internal resources, and to redirect a user to third-party services.
- [ ]  The application uses `\A \z`  to define the start, and the end of the string or specify `multiline: **true`.**
- [ ]  The application doesn’t use `Marshal` library to serialize and unserialize objects.
- [ ]  The application applied a list of permitted parameters in `permit` method, which limits parameters to be modified by a user.
- [ ]  The application doesn’t have security flag set to `config.active_record.whitelist_attributes=false`, which stands for mass assignment.
- [ ]  The application does not use `URI#open` method from `open-uri` library.

# Security Review

RoR web applications follow the MVC (model, view, and controller) architecture. Most of client-side vulnerabilities appears within the view component. Views are built with ERB template engine, thus even SSTI vulnerabilities are quite common thing. All the server-side business logic is handled by the controller component. Developers should be aware about security of RoR applications, since most of the vulnerabilities appears within unique specifics regarded only to RoR applications. The next section describes common security pitfalls, and how their patterns can be identified during source code review. They usually occurs, when developers disable built-in security options or do not follow security coding practices.

## Framework architecture

RoR (Ruby on Rails) application has the following code base structure:

```bash
.
├── Dockerfile #software version, hardcoded credentials
├── Gemfile #software versions
├── Gemfile.lock #software versions
├── app
│   ├── assets # images, video etc
│   ├── controllers #all application logic located here
│   │   ├── admin_controller.rb
│   │   ├── users_controller.rb
│   ├── helpers #helper - method that is (mostly) used to share reusable code
│   │   ├── admin_helper.rb
│   ├── mailers #allows send emails from your application using mailer classes
│   │   └── user_mailer.rb
│   ├── models #Ruby class that is used to represent data 
│   │   ├── user.rb
│   └── views # HTML templates
│       ├── admin
│       │   ├── get_all_users.html.erb
├── config #app configuration, should be reviewed because developers can disable security features
│   ├── application.rb #application configuration
│   ├── boot.rb
│   ├── database.yml #database config, may contain hard-coded creds
│   ├── environment.rb
│   ├── environments
│   │   ├── development.rb #application configuration
│   ├── initializers
│   │   ├── constants.rb #hardcoded credentials
│   │   ├── filter_parameter_logging.rb #logging
│   │   ├── html_entities.rb #Enables or disables the escaping of HTML entities in JSON serialization
│   │   ├── key.rb #hardcoded credentials
│   │   ├── secret_token.rb #cookie signing
│   │   ├── session_store.rb #how session store is organized
│   ├── locales
│   │   └── en.yml #hardcoded credentials
│   ├── routes.rb #First thing to investigate, application routing
│   ├── secrets.yml #Is credentials/secrets encrypted?
│   └── secrets2.yml #Is credentials/secrets encrypted?
├── db
│   ├── schema.rb #database schema
│   └── seeds.rb #database data, may contain hard-coded creds
├── lib #extended modules
│   ├── encryption.rb #encryption
├── log #log files
├── public #static files and compiled assets
│   ├── 404.html
│   └── robots.txt
├── script
├── spec #for testing purposes
└── vendor #third-party code
```

[Understanding your Rails Application Structure [Beginners Guide] | HackerNoon](https://hackernoon.com/understanding-your-rails-application-structure-r8w32xj)

[Getting Started with Rails - Ruby on Rails Guides](https://guides.rubyonrails.org/getting_started.html#creating-the-blog-application)

### Sensitive files

```markdown
/config/database.yml -  May contain production credentials.
/config/initializers/secret_token.rb - Contains a secret used to hash session cookies.
/db/seeds.rb - May contain seed data including bootstrap admin user.
/db/development.sqlite3 -  May contain real data.
```

## MVC architecture

Routes, controllers, actions, views and models are typical pieces of a web application that follows the [MVC (Model-View-Controller)](https://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93controller) pattern. MVC is a design pattern that divides the responsibilities of an application to make it easier to reason about. Rails follows this design pattern by convention.

### Routes

We recommend starting the security review with the app routing, because it allows mapping routes with its handlers to understand the API structure of the application. Router determines what kind of controller and action should actually execute the code. App routing is described within `/config/routes.rb` file. 

The code base below would process users’ request, if they will request the `https://app.domain.com/patients/<id>` URL. When RoR server receives user’s request, it knows that it should execute code within the controller `patients` and action `show`.

```ruby
get '/patients/:id', to: 'patients#show'
```

We could also find something like this:

1. Declare all of the common routes for a given resourceful controller like `index`, `show`, `new`, `edit`, `create`, `update`, and `destroy` actions.
    
    ```ruby
    resources :photos
    ```
    
2. Limit the actions which can be used for the resourceful controller.
    
    ```ruby
    resources :comments, only: [:show, :edit, :update, :destroy]
    ```
    
3. Organise the resources under the namespace “admin” and navigate to them like /admin/posts or /admin/comments.
    
    ```ruby
    namespace "admin" do
      resources :posts, :comments
    end
    ```
    

Follow the documentation for other possible routes mapping:

[ActionDispatch::Routing::Mapper](https://api.rubyonrails.org/v7.0.3.1/classes/ActionDispatch/Routing/Mapper.html)

[Rails Routing from the Outside In - Ruby on Rails Guides](https://guides.rubyonrails.org/routing.html)

### Controllers & Actions

The controllers is an actual application business logic, each controller can be found within the `app/controllers/..` folder. Example:

```bash
├── app
│   ├── controllers #all application logic located here
│   │   ├── admin_controller.rb
```

Example of simple controller. A controller is a Ruby class which inherits base `ApplicationController` class and all its methods. The `<` sing is an inheritance.

```ruby
class ClientsController < ApplicationController
  def new
  end
end
```

As an example, if a user goes to `/clients/new` path in your application to add a new client, Rails will create an instance of `ClientsController` controller and execute its `new` method. What’s worth to mention is that empty method from the example above would work. Because Rails would render the `new.html.erb` view by default, unless the method defines different logic. By creating a new `Client`, the `new` method can make a `@client` instance variable accessible in the view:

```ruby
def new
  @client = Client.new
end
```

[Action Controller Overview - Ruby on Rails Guides](https://guides.rubyonrails.org/action_controller_overview.html)

[Action Controller Overview - Ruby on Rails Guides](https://guides.rubyonrails.org/action_controller_overview.html#methods-and-actions)

### Views

Views are stored within the following `app/views/[controller]/[view_name].html.erb` pattern path. View is a simple HTML page managed by ERB template engine, which displays values returned from the controller.

```ruby
<h1>Articles</h1>
<ul>
  <% @articles.each do |article| %>
    <li>
      <%= article.title %>
    </li>
  <% end %>
</ul>
```

The methodology for mapping routes to views is the following:

1. Controller’s action has the same name in the routes file as a view name.
    1. If the `resources` specified, a view name corresponds to a controller’s action.  For example, `POST` method calls `create` action and app/views/[controller]/create.html.erb view is displayed. 
2. `Render` method is used in the controller’s code or template itself to display a view.

[Layouts and Rendering in Rails - Ruby on Rails Guides](https://guides.rubyonrails.org/layouts_and_rendering.html)

### Models

A model is a Ruby class that is used to represent data. Additionally, models can interact with the application's database through a feature of Rails called Active Record.

Example of User model located under `app/models`:

```ruby
class User < ApplicationRecord
  validates :name, presence: true
end
```

Model is used to describe object and use it through the application.

[Active Record Basics - Ruby on Rails Guides](https://guides.rubyonrails.org/active_record_basics.html)

## XSS

By default, RoR escapes HTML entities, but it proposes a couple of methods to disable HTML characters escape like `raw`, `html_safe`, `content_tag` etc.

****Unescaped variable enters template engine in Ruby code****

- `html = "<div>#{name}</div>".html_safe`
- `content_tag :p, "Hello, #{name}”`
- `raw @user.name`
- `config.active_support.escape_html_entities_in_json = false` It leads to XSS, when `Hash#to_json()` used.

****Bypassing the template engine****

- `ERB.new("<div>#{@user.name}</div>").result`
- `render inline: "<div>#{@user.name}</div>”`
- `render text: "<div>#{@user.name}</div>”`

****Templates: Variable explicitly unescape****

- `<%= name.html_safe %>`
- `<%= content_tag :p, "Hello, #{name}" %>`
- `<%= raw @user.name =>`
- `<%== @user.name %>`

****Templates: Variable in dangerous location****

- `<div class=<%= classes %></div>`
- `<a href="<%= link %>"></a>`
- `<%= link_to "Here", @link %>`
- `<script>var name = <%= name %>;</script>`

[XSS prevention for Ruby on Rails | Semgrep](https://semgrep.dev/docs/cheat-sheets/rails-xss/)

## Command injection

![https://i.stack.imgur.com/1Vuvp.png](https://i.stack.imgur.com/1Vuvp.png)

List of command injection sinks:

```markdown
eval("ruby code here")
system("os command here")
`ls -al /` # (backticks contain os command)
exec("os command here")
spawn("os command here")
open("| os command here")
URI#open from open-uri
Process.exec("os command here")
Process.spawn("os command here")
IO.binread("| os command here")
IO.binwrite("| os command here", "foo")
IO.foreach("| os command here") {}
IO.popen("os command here")
IO.read("| os command here")
IO.readlines("| os command here")
IO.write("| os command here", "foo")
syscall
%x() %x %x{} %x-os-
popen<n>
exec
Open3.popen3()
fork()
PTY.spawn()
constantize
```

## SQL injection

Concatenation of user input with SQL query parameter will lead to SQL injection. Example:

```ruby
User.where("name = '#{params[:name]'") # SQL Injection!
```

To mitigate the risk of this kind of SQL injection the application should use parametrization, there are 2 examples of those:

```ruby
User.where(["name = ?", "#{params[:name]}"])
```

```ruby
User.where({ name: params[:name] })
```

In those 2 examples there is no direct operations with SQL query, column `name` is set explicitly to the `name` key.

### Potentially vulnerable methods

**Calculate Methods**

`Calculate` function takes an operation and a `column` name to calculate an amount or find a maximum/minimum/average value. The `column` value is a user controlled parameter:

```ruby
**Order**.calculate(:sum, params[:column])
```

Since parametrization and sanitization are missing user can manipulate a column value and achieve a limited SQL injection. An attacker could calculate values from the other tables:

```ruby
params[:column] = "age) FROM users WHERE name = 'Bob';"
```

The final SQL query sums an age of users named Bob from the users table:

```sql
**Query
SELECT** **SUM**(age) **FROM** users **WHERE** name = 'Bob';) **FROM** "orders"
**Result
7**6
```

**Delete By Method**

The `delete_all` method takes the same kind of conditions arguments as `where`.
The argument can be a string, an array, or a hash of conditions. Strings will not be escaped at all. Use an array or hash to parameterize arguments.

```ruby

**User**.delete_by("id = **#{**params[:id]**}**")
```

This example bypasses any conditions and deletes all users.

```ruby
params[:id] = "1) OR 1=1--"
```

The final SQL query deletes all users and result returns an amount of deleted users:

```sql
Query
DELETE FROM "users" WHERE (id = 1) OR 1=1--)

Result
6
```

**Destroy By Method** 

The `destroy_by` method takes the same kind of conditions arguments as `where`.
The argument can be a string, an array, or a hash of conditions. Strings will not be escaped at all. Use an array or hash to safely parameterize arguments.

```ruby
**User**.destroy_by(["id = ? AND admin = '**#{**params[:admin]**}**", params[:id]])
```

This example bypasses any conditions and deletes all users.

```ruby
params[:admin] = "') OR 1=1--'"

```

The final SQL query destroys all users and result returns an amount of destroyed users:

```sql
Query
DELETE FROM "users" WHERE "users"."id" = ?
```

[Rails SQL Injection Guide: Examples andPrevention](https://www.stackhawk.com/blog/sql-injection-prevention-rails/)

[Rails SQL Injection Examples](https://rails-sqli.org/)

## SSTI

Occurs when an attacker could directly manipulate with ERB template like this:

- `ERB.new("<div>#{@user.name}</div>").result`

## Sessions

### Session Hijacking

*Stealing a user's session ID allows an attacker to use the web application in the victim's name.*

Here are some ways to hijack a session, and their countermeasures:

- Sniff the cookie in an insecure network. That’s why the connection must be secure. In Rails 3.1 and later, this could be accomplished by always forcing SSL connection in your application config file: `config.force_ssl = true`
- Instead of stealing a cookie unknown to the attacker, they fix a user's session identifier (in the cookie) known to them.

### Session Storage

Rails uses `ActionDispatch::Session::CookieStore` as the default session storage.

Rails `CookieStore` saves the session hash in a cookie on the client-side. The server retrieves the session hash from the cookie and eliminates the need for a session ID. That will greatly increase the speed of the application, but it is a controversial storage option and you have to think about the security implications and storage limitations of it:

- Cookies expiration must be in place
- No sensitive information must be in cookies
- The application must invalidate old session cookies to avoid malicious reuse
- Cookies must be encrypted. Rails encrypts cookies by default

The `CookieStore` uses the [encrypted](https://api.rubyonrails.org/v7.0.3/classes/ActionDispatch/Cookies/ChainedCookieJars.html#method-i-encrypted) cookie jar to provide a secure, encrypted location to store session data. 

The encryption key for cookies, is derived from the `secret_key_base` configuration value.

Secrets must be long and random(`bin/rails secret` must be used to get new unique secrets).

Different salt values for encrypted and signed cookies must be used. 

Environments must use a random key present in `config/credentials.yml.enc`, shown here in its decrypted state: `secret_key_base: 492f...`

### Signed Cookies Configurations

Example of specifying the digest used for signed cookies:

`Rails.application.config.action_dispatch.signed_cookie_digest = "SHA256"`

Strong cryptographic algorithm must be in place.

### Replay Attacks for CookieStore Sessions

If cookie stores sensitive information such as credit balance an attacker can reuse old cookies this bigger balance and abuse the system.

No sensitive information must be in cookies or a nonce (random value) must be in place to prevent replay attacks. A nonce is valid only once, and the server has to keep track of all the valid nonces. 

### Session Fixation

![https://guides.rubyonrails.org/images/security/session_fixation.png](https://guides.rubyonrails.org/images/security/session_fixation.png)

Attack focuses on fixing a user's session ID known to the attacker, and forcing the user's browser into using this ID. It is therefore not necessary for an attacker to steal the session ID afterwards.

The most effective countermeasure is to *issue a new session identifier* and declare the old one invalid after a successful login. That way, an attacker cannot use the fixed session identifier. This is a good countermeasure against session hijacking, as well. Here is how to create a new session in Rails:

`reset_session`

[Devise](https://rubygems.org/gems/devise) gem for user management, it will automatically expire sessions on sign in and sign out for you. 

[Securing Rails Applications - Ruby on Rails Guides](https://guides.rubyonrails.org/security.html#sessions)

## CSRF

By default, Rails includes an [unobtrusive scripting adapter](https://github.com/rails/rails-ujs), which adds a header called `X-CSRF-Token` with the security token on every non-GET and non-HEAD Ajax call. Without this header, non-GET and non-HEAD Ajax requests won't be accepted by Rails.

- `protect_from_forgery` in application_controller
- `<%= csrf_meta_tags %>` in template

## SSRF

To find SSRF vulnerabilities during source code review first libraries used by the application must be identified, next corresponding dangerous functions using next check-list must be found:

1. `net/http` library:
    1. `require 'net/http'`
    2. `Net::HTTP.get()`  or `Net::HTTP.get_print()` or `Net::HTTP.get_responce()`- make request
    `Net::HTTP.start(uri.hostname, uri.port)` - creates connection to host
    `Net::HTTP::Get.new uri` - `Get` request within connection
    
    Redirect bypass - look for `Net::HTTPRedirection` and decide whether redirects happens.
    
2. `OpenURI` library:
    1. `URI.open()`
3. `httparty` library:
    1. `require 'httparty'`
    2. `HTTParty.get` -  make request
4. `http` library:
    1. `require 'http'`
    2. `HTTP.get` or `HTTP.follow.get` - make request
    `HTTP.persistent` - persistent session
    
    Redirect bypass -  `HTTP.follow.get` will perform location redirection.
    
5. `faraday` library:
    1. `require 'faraday'`
    2. `Faraday.get` or `HTTP.follow.get` - make request
    `Faraday.new` - request creation
    
    Redirect bypass - `.response :follow_redirects` 
    
6. `Httpx` library:
    1. `require "httpx"`
    2. `HTTPX.get` or `HTTPX.post`
7. `rest-client` library:
    1. `require 'rest-client'`
    2. `RestClient.get`, `RestClient.post`, `RestClient::Request.execute`, `RestClient.delete`,`RestClient::Resource.new`

8. `Excon` library:

1. `require 'excon'`
2. `Excon.get`, `Excon.new`, `Excon.post`

9. `Typhoeus` library:

1. `require 'typhoeus'`
2. `Typhoeus::Request.new`, `Typhoeus::Hydra.hydra`
1. `Curb` library:
    1. `require 'curb'`
    2. `Curl.get`, `Curl.post`,`Curl::Easy.perform`,`Curl::Easy.new`,`Curl::Easy.http_post`

In addition, all methods and functions that can make HTTP requests should be analyzed to determine if they use user input and how this might affect the requests.

## Redirects

Surprisingly, but most of tested RoR apps have open HTTP redirects vulnerabilities, why? RoR framework has special methods for redirection like `redirect_to`, `redirect_back`  and developers do not validate the domain before redirecting a user.

- `redirect_to params[:to]`
- `redirect_back`, `redirect_back_or_to` -> takes URL from Referrer and redirects a user to it

Comment:

Starting from Rails 7.0, open redirect protection `raise_on_open_redirects` takes place in the file `config/initializers/new_framework_defaults_7_0.rb`. Thus, to allow open redirect, it should be explicitly allowed like (or enabled globally):

- `redirect_to "https://rubyonrails.org", allow_other_host: true`

## Business logic

### RegExp

Very funny thing, but common regexp rules do not work in RoR, but why? RoR string seems to be multi-line, thus `/^https?:\/\/[^\n]+$/i` will check only the first line leaving the others. It can be a sink for many other vulnerabilities like SQLi, command injections, XSS etc. The proper way to check string with regexp is `/\Ahttps?:\/\/[^\n]+\z/i`  (`\A \z`) or enable `multiline: **true`.**

`\A` - start of the line

`\z` - end of the line

### Mass assignment

It’s a quite common vulnerability and it takes place, when a developer rewrites default action like `create`, `update` and does not validate parameters to accept.

```ruby
def create
    user = User.new(user_params)

def user_params
    params.require(:user).permit!
end
```

In addition, a developer can add this parameter to `attr_protected`, thus mass assignment won’t work there. Otherwise, it can be done globally via flag `config.active_record.whitelist_attributes = true`, where a specific file is created attributes available for mass-assignment.

### HEAD bypass

RoR router treats HEAD method as GET. For example, the routes file contains `get 'app/index'` and HTTP requests can be made like `GET app/index` and `HEAD app/index`. So Rails router does not distinguish HEAD and GET methods, but the controllers can do that. If the controller checks for GET method, so HEAD method can be used to bypass the checks:

```ruby
if request.get?
  #do something
else (HEAD method goes there)
  #grant permission
end
```

[Bypassing GitHub's OAuth flow](https://blog.teddykatz.com/2019/11/05/github-oauth-bypass.html)

## Insecure deserialization

Ruby uses the `Marshal` library to serialize and unserialize objects. Marshalling is an unsafe way to deserialize the provided data because the `Marshal.load()` method can deserialize any class loaded into the Ruby process, which can lead to remote code execution (RCE) attacks.

- `Marshal.load()`
- `ActiveSupport::MessageVerifier.new(key)` - By default, Marshall serializer is used, if no other is specified
- `Rails.application.message_verifier()` - For this particular case, there is no option to specify a serializer

Message verifier is used to calculate a signature and prevent the tampering of transmit data. To exploit the insecure deserialization and achieve RCE, an attacker would need to obtain a key used for signature calculation.

## Misconfigurations

This group will include RoR built-in flags, which can be disabled by developers and pose a treat for applications:

- `config.active_record.whitelist_attributes=false` → mass assignment
- `ActiveSupport::escape_html_entities_in_json = false` → XSS when to_json()

# References

[Ruby on Rails - OWASP Cheat Sheet Series](https://cheatsheetseries.owasp.org/cheatsheets/Ruby_on_Rails_Cheat_Sheet.html)

[Securing Rails Applications - Ruby on Rails Guides](https://guides.rubyonrails.org/security.html)

[Brakeman](https://brakemanscanner.org/)

[GitHub - brunofacca/zen-rails-security-checklist: Checklist of security precautions for Ruby on Rails applications.](https://github.com/brunofacca/zen-rails-security-checklist#the-checklist)
