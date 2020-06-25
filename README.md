# Hands-on with Service Mesh with Consul Connect

## 👩‍💻 ⬅️ 🐻💍💍 (Getting your bearings)


Two cloud instances have been provisioned for you: `frontend` and `data-api`.
You can log in as user `playground`, with password `JunePanda` 🐼.

**frontend** hosts a webapp (`frontend.py`) that listens for incoming connections from the internet, and makes API calls to the other application: **data-api** also hosts a webapp (`data.py`) that exposes a REST API to manage users and sessions, and persist them on disk (`data-api.db`, an sqlite3 DB).  
Upon startup, `data-api` registers itself as a service with Consul; `frontend` queries Consul on startup for the address and port of a service called "data-api".

Please take a few minutes to explore these:

* navigate to http://\<frontend\>/ and play around with it;
* navigate to the Consul UI on either http://\<frontend\>:8500/ or http://\<data-api\>:8500/ and explore the nodes & services;
  * what port is data-api listening on? (remember this as `<data-api-port>` for the step below);
  * (note that the IP address shown is a private one within the AWS cloud);
* see if you can establish a connection to `data-api`: http://\<data-api\>:\<data-api-port\>/admin/list_all ;
  * you should get a JSON-encoded error message back; this means the connection works;

`frontend` basically follows the same approach: it connects to Consul (a Consul agent runs on every node, listening on well-known port 8500), looks up the service it wants to connect to by name (`data-api`), then initiates a direct connection to that IP address and port.

We'll have a look at how this works in the next section.

> When done with the section, please spam the YouTube comments with: 🐻

---

## 1) Service Registration  
  * Log into the `data-api` instance (using SSH or WeTTY)  
  * Have a look at the bottom of the file `/opt/dpg/data-api.py`
    * e.g. type `less /opt/dpg/data-api.py`, `[Enter]` then `Shift + G`

    You should see the following lines:  
    ```
    ...
    port = 5001
    c = consul.Consul()
    c.agent.service.register('data-api', port=port)
    ...
    ```

`data-api` connects to its local Consul agent using the default settings (HTTP on 127.0.0.1:8500), then registers a service called `data-api` (itself) on a given port. By default, the IP private address of the agent will be used—which is useful in our case as we don't have to implement logic to find out our own IP address inside `data-api.py`.

This is how the service `data-api` comes to show up in the Consul UI.

> When done with the section, please spam the YouTube comments with: 🥅

---

## 2) Service Discovery
  * Now, log into the `frontend` instance instead
  * Have a look at the bottom of the file `/opt/dpg/frontend.py`
    * e.g. type `less /opt/dpg/frontend.py`, `[Enter]` then `Shift + G`

    You should see the following lines:  
    ```
    ...
    c = consul.Consul()
    da = c.catalog.service('data-api')
    da = da[1][0]
    # TO DO: implement SSL!
    BACKEND_URL = f'http://{da["Address"]}:{da["ServicePort"]}'
    ...
    ```

`frontend` also connects to its local Consul agent, and then _queries_ the Consul catalog for a given service by name (`data-api`). (Don't worry about the part with `da[1][0]`—that is just due to how the Python SDK presents the results.)

As we can see, `frontend` will subsequently initiate HTTP connections directly to the `data-api` service. The mechanism known as _Service Discovery_ does not take care of or dictate _how_ the connection is made.

As we can see, a developer has done the respobnsible thing and added a comment inidcating the need for encryption.  
Technically compliant, the best kind of compliant.

As we have tried above, the `data-api` service is directly accessible to anyone (with network connectivity. For demonstration purposes, the firewalls have been left wide open for this DevOps Playground. This is not recommended as a Production setup. Another best-practice for Production hardening is not to get your best-practices from a 1h Tech-meetup.)

> When done with the section, please spam the YouTube comments with: 🏒

---

## 3) Service Mesh

### To be on the receiving end...

In the remainder of this DevOps Playground, we will change the existing setup to replace Consul's _Service Discovery_ with its _Service Mesh_. In Consul, this is also known as _Consul Connect_.

Let's start by registering a Service Mesh _Proxy_ for `data-api`:  
* Log into the `data-api` instance;
* Become root—it'll be easier;
  * `sudo -i`, `[Enter]`, `JunePanda`, `[Enter]`
* Create a new JSON file in `/etc/consul.d`
  * e.g. `vim /etc/consul.d/data-api-proxy.json`
  * if you prefer Nano, please simply disconnect and you have hereby forfeited any pizzas you may have won;
* set the following content:
```
{
  "service":
    {
        "name": "data-api-proxy",
        "kind": "connect-proxy",
        "port": 20000,
        "proxy":
        {
            "destination_service_name": "data-api",
            "destination_service_id": "data-api",
            "local_service_address": "127.0.0.1",
            "local_service_port": 5001
        }
    }
}
```
* run `consul validate /etc/consul.d` to check for configuration errors  
* run `consul reload` to make Consul take the new configuration file into account

This step tells Consul that there _is_ a Consul Connect Proxy (this is currently a lie), which listens on port 20000 and serves as a proxy for the service `data-api` which listens on port 5001 of localhost. (Using a different approach, a lot of these values can be omitted, but it's good for our understanding to see them stated explicitly.)

ℹ️If you have a peek at the Consul UI again now, you should see that the `data-api` service mentions the connect proxy.

The next step is to repent for our lie, and actually create said proxy we promised to Consul.

* Create a new systemd service file called `/etc/systemd/system/data-api-proxy.service`
  * e.g. `vim /etc/systemd/system/data-api-proxy.service`
* set the following content:
```
[Unit]
Description=My Miscellaneous Service
After=network.target

[Service]
Type=simple
User=consul
ExecStart=/usr/local/bin/consul connect proxy -sidecar-for data-api
Restart=always

[Install]
WantedBy=multi-user.target
```

* run: `systemctl daemon-reload`
* run: `systemctl start data-api-proxy.service`

This starts a new process using the already-installed Consul binary (which incorporates many functionalities: Consul server, Consul client, Consul CLI, and Consul Connect Proxy). The bnary is given the instructon to be a `-sidecar-for data-api`, and can then simply read its own configuration from the local Consul agent—it's that simple!

> When done with the section, please spam the YouTube comments with: 🤹‍♂️

--- 

### ... and dishing it out

Next, we want to create a similar setup on the `frontend` instance.  
We'll do it a bit differently, to showcase the various configuration methods for Consul services & Consul Connect proxies.

Let's start by registering the service `frontend` which hithereto Consul was unaware of:  
* Log into the `frontend` instance;
* Become root—it'll be easier;
  * `sudo -i`, `[Enter]`, `JunePanda`, `[Enter]`
* Create a new JSON file in `/etc/consul.d/frontend.json`
  * e.g. `vim /etc/consul.d/frontend.json`
* set the following content:

```
{
  "service": {
    "name": "frontend",
    "port": 5000,
    "connect":
    {
      "sidecar_service":
      {     
        "proxy":    
        {           
          "upstreams":      
          [                 
            {                       
              "destination_name": "data-api",
              "local_bind_port": 12345      
            }                       
          ]                 
        }           
      }     
    }
  }
}
```

* run `consul validate /etc/consul.d` to check for configuration errors  
* run `consul reload` to make Consul take the new configuration file into account

Next, like before, let's actually create the Consul Connect proxy.

* Create a new systemd service file called `/etc/systemd/system/frontend-proxy.service`
  * e.g. `vim /etc/systemd/system/frontend-proxy.service`
* set the following content:

```
[Unit]
Description=My Miscellaneous Service
After=network.target

[Service]
Type=simple
User=consul
ExecStart=/usr/local/bin/consul connect proxy -sidecar-for frontend
Restart=always

[Install]
WantedBy=multi-user.target
```
* run: `systemctl daemon-reload`
* run: `systemctl start data-api-proxy.service`

At this point, we have started and configured both Proxies. We should see three services in the Consul ui: `Consul`, `frontend`, and `data-api`.

Let's do a quick manual test to see if the proxy connectivity works as expected:

* log in / switch your your terminal for the `frontend` instance;  
* run `curl http://127.0.0.1:12345/admin/list_all`

We're opening a connection to the Consul Connect Proxy on the local instance. Port `12345` is the port we have defined as one of the `upstreams` above, along with `"destination_name": "data-api"`. We therefore expect this to be equivalent to the /admin/list_all endpoint we opened in our browser at the beginning of the hands-on session (http://\<data-api\>:5001/admin/list_all)—but is it?  
(Note that the direct connection to `data-api`, e.g. from our browser, still works—and why wouldn't it? We'll change that soon.)

> When done with the section, please spam the YouTube comments with: 🛰

---

### The Moment of Truth\*
(\* "Truth" Trademark Registration pending, 2020)

Now that we have our secure tunnel set up from the `frontend` to the `data-api` instances, let's reconfigure `frontend` to make use of it!

* Log into the `frontend` instance;
* Become root—it'll be easier;
  * `sudo -i`, `[Enter]`, `JunePanda`, `[Enter]`
* Edit the bottom of `frontend.py`
  * e.g. `vim /opt/dpg/frontend.py`, then `shift+G`
* change the following content:
```
if __name__ == '__main__':
    c = consul.Consul()
    da = c.catalog.service('data-api')
    da = da[1][0]
    # TO DO: implement SSL!
    BACKEND_URL = f'http://{da["Address"]}:{da["ServicePort"]}'
    app.run(host='0.0.0.0', port=5000, debug=True)
```
to:
```
if __name__ == '__main__':
    # c = consul.Consul()
    # da = c.catalog.service('data-api')
    # da = da[1][0]
    # # TO DO: implement SSL!
    # BACKEND_URL = f'http://{da["Address"]}:{da["ServicePort"]}'
    BACKEND_URL = 'http://127.0.0.1:12345'
    app.run(host='0.0.0.0', port=5000, debug=True)

```

We don't need to implement the Consul logic anymore—our app can now be completely Consul-unaware!  
Furthermore, we have changed the `BACKEND_URL` to an endpoint on the local host, and therefore don't need to worry about the lack of encryption anymore either.

* Restart `frontend` for these change sto take effect;
  * (as root) `systemctl restart frontend.service`

Refresh the `frontend` webapp in your browser and play around with it to make sure it still works. (Regsitering, Loggin in, and Doing all require a connection to `data-api`, so if one of them works, we can rest assured that the connectivity between `frontend` and `data-api` works.)

> When done with the section, please spam the YouTube comments with: 💿

---

### Hardening 

As noted, it is still possibly to directly connect to `data-api`. Let's change that!

* Log into the `data-api` instance;
* Become root—it'll be easier;
  * `sudo -i`, `[Enter]`, `JunePanda`, `[Enter]`
* Edit the last line of `data.py`
  * e.g. `vim /opt/dpg/data.py`, then `shift+G`
* change the last line:
```
    app.run(host='0.0.0.0', port=port, debug=False)
```
to:
```
    app.run(host='127.0.0.1', port=port, debug=False)
```

* Restart `data-api` for these change sto take effect;
  * (as root) `systemctl restart data-api.service`

The app should still be working (do a quick smoke test!), but we should be unable to establish a connection directly to `data-api`:  
* log in / switch your your terminal for the `frontend` instance;  
  * run `curl http://127.0.0.1:12345/admin/list_all`
* refresh the browser tab with the URL ending in `:5000/admin/list_all`

These should both now fail!

> When done with the section, please spam the YouTube comments with: 💪

#### Extra credit:

Can you get through to `data-api` by connecting diretcly to its Consul Connect Proxy, which is listening on `data-api`'s public and private IPs on port `20000`? Why / why not? What's the error message? What would you need?

---

### Good Intentions?

As mentioned in the presentation, we can set up **intentions** as an equivalent to firewall rules.  
Let's explore this:

* navigate to the Consul UI on either http://\<frontend\>:8500/ or http://\<data-api\>:8500/ ;
* go to `Intentions` in the navbar at the top ;
* let's start by creating a default _deny_ rule:  
  * click the blue `Create` button ;
  * set `Source Service` to `* (All Services)` ;
  * set `Destination Service` to `* (All Services)` ;
  * select `Deny`
  * click `Save`
* now let's add a rule to allow traffic from `frontend` to `data-api`:
  * click the blue `Create` button ;
  * set `Source Service` to `frontend` ;
  * set `Destination Service` to `data-api` ;
  * select `Allow`
  * click `Save`

Let's make sure the app still works. What does the cowsay?

Now, remove the `Allow` rule (or change it to `Deny`) and try again.

> When done with the section, please spam the YouTube comments with: 🧠

---