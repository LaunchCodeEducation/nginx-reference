# Nginx Overview & Reference
Nginx (“engine-x”) is an incredibly performant web server analogous to a Tomcat or Apache server. Nginx may be used as an HTTP server for **serving static content**, as a **load balancer (LB)** or as a  **reverse proxy (RP)**. 

Note that Nginx is **only a web server** not an **application server**. Web servers are purely **transport layer** servers - they have no business logic and are only concerned with incoming requests and outgoing responses. 

Although web servers do not contain business logic they can contain **routing logic**. Routing logic defines how HTTP (and other supported protocols) requests should be routed. Nginx supports (relatively) simple and declarative syntax for defining its routing logic.

## Static Content
Nginx can be used to serve static content meaning content that exists in the file system Nginx is running from. This content can include HTML, CSS and JS files or other file types like images and documents. As a static server Nginx shines due to its efficiency in handling concurrent connections and ability to cache responses in its host file system.
* [NGINX Docs | Serving Static Content](https://docs.nginx.com/nginx/admin-guide/web-server/serving-static-content/)

## Load Balancer (LB)
Nginx can be used as a load balancer to distribute traffic to multiple application service targets. When Nginx is used as a load balancer a set of routing rules can be used to determine how it will distribute traffic among the child services running behind it. 
* [NGINX Docs | HTTP Load Balancing](https://docs.nginx.com/nginx/admin-guide/load-balancer/http-load-balancer/)

## Reverse Proxy (RP)
In a RP arrangement Nginx is used as the entry point, or gateway, for all incoming HTTP traffic. Rather than allowing traffic to be sent directly to an application server Nginx will receive it instead. 

It will listen on the common HTTP ports (80 and 443) for incoming traffic. The Nginx server will then relay (**proxy**) incoming requests to an application server based on any number of routing rules. 

One simple use case for using Nginx as a RP is to automatically upgrade traffic from http to https. In this configuration Nginx will listen for incoming requests on port 80 then redirect them to the same location using the https protocol.

Another common use case is to use Nginx for TLS (https) termination. Rather than burden application servers with managing TLS termination this logic can be written into Nginx. Nginx will then automatically decrypt traffic before proxying it to its target application service. It will also encrypt proxied responses before being sent back to the consumer.

Nginx can be used as a RP both within the same VM / Container as the application server(s) or as a standalone unit in its own VM / Container within the [intra]network. In the former architecture Nginx will proxy requests to local services listening on specified ports. In the latter design Nginx will proxy requests to internal / external IP addresses of its child services.
* [NGINX Docs | NGINX Reverse Proxy](https://docs.nginx.com/nginx/admin-guide/web-server/reverse-proxy/)

# Configuring Nginx
Nginx uses configuration files `.conf` that are saved in one of several locations. Typically this is `/etc/nginx/conf.d` but you should check your individual OS / installation procedure if this location is different for your environment.

Config files may be written as a single file or composed from several (narrower-scoped) config files. Config composition uses the `include` directive and makes individual configuration sections more portable, reusable and extendable. 

Despite the recommendation of the official docs, config composition should be avoided until you are comfortable with using single-file configuration. You can always write the single file first, make sure it works, then break them into a compositional approach. This ensures that errors are syntactical and not due to composition mistakes (which will cut down debugging time).

## Variables
There are a number of variables that can be used to access details of web traffic. They are all prefixed (like bash) with a `$` character. Their values are **context** dependent (more info below). 
* [Alphabetical index of variables](http://nginx.org/en/docs/varindex.html)

## Directives
Every config file is made up of **directives** that define its routing logic. Each directive is classified as either a **simple directive** or a **block directive**. Every directive has a **context** that it may be used in (including the **any context**). 
* [Alphabetical index of directives](nginx.org/en/docs/dirindex.html)

## Simple Directives
Each simple directive is a single line that
* begins with a `directive label` (defining what configuration target it modifies) 
* is followed by (at least one) space(s) and one or more `parameter(s)`

## Block Directives
Block directives are directives that group together inner block / simple directives within a set of curly `{}` braces. They begin with a `directive label` followed by an opening bracket `{`, any number of inner directives and a closing bracket `}`.
```nginx
directive_label {
	# other directives defined within
}
```
In programming terms a block directive is like a **block scope** in that it contains and shields any number of other simple / block directives contained within it. Those at the most narrow scope (innermost blocks) will **inherit** the directives defined in the scopes above it (when applicable).
When working with directives keep the following in scope notes mind:
* A directive at a higher scope (outer block) will be inherited by its child blocks (inner block(s))
	* “broad specificity” / lower precedence
* A directive at a lower scope (inner block / context) will supersede any inherited directive
	* “narrow specificity” / higher precedence

## Top-Level (Context) Directives
There are several **top-level directives** that define **context blocks** for organizing behavior associated with different aspects (contexts) of web traffic.  If a directive is defined outside of one of these TLD then it is considered to be in the **main context** or “global scope”.
* **Must be defined in the main context (they must be the outermost block)**
* main context: anything written in the open space (“global scope”) of the config file
* `http`: logic relating to HTTP traffic (**most commonly used**)
* `mail`: logic related to mail-server traffic
* `stream`: logic related to low-level TCP/UDP traffic 
* `events`: logic relating to connections (of any protocol)

```nginx
# http TLD is defined in the main context (not within another block)
http {
	# any number of directives that apply specifically to HTTP traffic
}
````

## Server Directives
The **server directive** is one of the most common directives used. Each server block defines the logic associated with a **virtual server**.  
* The directives defined within a server block will affect traffic for **whatever context they have been defined within**
* In other words a `server` block **must be defined within a TLD context**
As an example, a server block within the `http` TLD will represent a virtual server activated for HTTP traffic.
Virtual servers can use many different directives within them. Three directives that are most often used within a server block are the `listen` , `server_name` and `location` directives:
*  `listen` directive defines what IP address and / or port the server should listen on
	* can also use a registered hostname (like `localhost`)
	* [listen arguments](http://nginx.org/en/docs/http/ngx_http_core_module.html#listen)
* `server_name` defines the domain name that the server should handle requests for
	* can be a FQDN, wildcard (subdomain prefix or TLD suffix) or regular expression
	* [server_name arguments](http://nginx.org/en/docs/http/ngx_http_core_module.html#server_name)
*  `location` directive is used to contain logic for handling a specific route within its parent server block
	* [location arguments](http://nginx.org/en/docs/http/ngx_http_core_module.html#location)

```nginx
http {
	server {
		# server listens for all incoming HTTP requests on port 80
		listen 80; # other arguments can be used as well

		# server will only process requests (on port 80) for the given name(s)
		# requests with the Host header matching these domains only
		server_name www.launchcode.org launchcode.org;

		location /path {
			# logic for handling the /path route for all requests on port 80
		}
	}
}
```

## If (Conditional) Directive
The `if` directive is used to apply conditional directives (housed within the `if` block).
* **Can only be used within a server or location block**
* Accepts a `condition`  argument
	* a variable name, an equality comparison, regex match and several others (see docs below)
* [If Directive](http://nginx.org/en/docs/http/ngx_http_rewrite_module.html#if)
```nginx
...
server {
	# must be defined within a server or location block!
	if (condition) {
		# directives applied if the condition is met
	}
}
```

## Custom Variables
## Set Directive
The `set` directive can be used to declare and assign a value to a custom variable. 
* **It may only be used within a server, location or if directive block**.
* The variable name must begin with a `$` like other Nginx variables
* [Set directive](http://nginx.org/en/docs/http/ngx_http_rewrite_module.html#set)
```nginx
# must be defined within a server, location or if block!
set $variable_name value;
```

## Map Directive
The `map` directive behave like a `switch` statement for assigning the value to a custom variable based on a matching case. It takes two arguments (in order) the `source variable` and the `custom variable` before the block begins.
* **Can only be used within the http context block**
	* Can not be used in any blocks (like `server`) housed within the `http` block!
* The variable to be assigned must begin with a `$` like other Nginx variables
* Assignment within the `map` block is based on a `case_value assigned_value;` syntax
	* where the `source variable` is compared to the `case_value`
	* if they match then the `custom variable` is assigned to the `assigned value`
	* if they don’t match then the next case(s) are tested 
* A `default` match can be used as a fallback if no cases match
	* if the default case isn’t defined the `assigned_value` will be an empty string
* [Map directive](http://nginx.org/en/docs/http/ngx_http_map_module.html#map)
```nginx
# can only be declared within the HTTP block!

# generic form
map $source_variable $custom_variable_name {
	match_string assigned_value;
}

# working example (from docs)
map $http_user_agent $mobile {
    default       0;
    "~Opera Mini" 1; # ~ is using regex matching
}

# if the $http_user_agent value contains "Opera Mini" then $mobile = 1
# if it does not then the default value is assigned with $mobile = 0;
```

# Running Nginx
Nginx is typically run as a service (a daemon process) within its host machine. It has a few common commands and flags that can be used in the daemon configuration or in executing Nginx as a (foreground) terminal process. All commands begin with the `nginx` CLI program:
* **Nginx will use the default config when starting / restarting / reloading**
	* default config file is `/config/dir/path/nginx.conf`
	* the `-c /path/to/custom.conf` flag (see others below) can be used to direct the command towards using a custom config file 
* `nginx start`: start nginx 
* `nginx stop`: stop nginx
* `nginx restart`: stop / start nginx
* `nginx reload`: reload without stopping (used to apply a new config without interrupting existing connections)
* [List of CLI flags](http://nginx.org/en/docs/switches.html)
	* a useful tool is `nginx -t` which will test the default config file for any errors
		* `nginx -T` does the same but outputs the conf file to the terminal
	* can be used to test a custom config file with `nginx -t -c /path/to/custom.conf`

# Extra Bits
## Config Directories by OS
* CentOS
	* config dir: `/etc/nginx/conf.d`
	* default static dir: ``
* Ubuntu
	* config dir: ``
	* default static dir: ``
* Amazon Linux
	* config dir: ``
	* default static dir: ``
* Docker Image
	* source: [Docker Hub](https://hub.docker.com/_/nginx)
	* config dir: `/etc/nginx/conf.d`
	* default static dir: ``
## Useful Links
* [NGINX Docs | Installing NGINX Open Source](https://docs.nginx.com/nginx/admin-guide/installing-nginx/installing-nginx-open-source/)
* [How Nginx processes a request](http://nginx.org/en/docs/http/request_processing.html)
* [NGINX Docs | Configuring NGINX Plus as a Web Server](https://docs.nginx.com/nginx/admin-guide/web-server/web-server/)
* [NGINX Docs | Security Controls](https://docs.nginx.com/nginx/admin-guide/security-controls/)
* [(Downloadable) Nginx Cheat Sheet.pdf | DocDroid](https://www.docdroid.net/ooD0qnV/nginx-cheat-sheet.pdf)
* [GitHub - SimulatedGREG/nginx-cheatsheet](https://github.com/SimulatedGREG/nginx-cheatsheet#domain-name-server_name)

#backend #reference #nginx #devops