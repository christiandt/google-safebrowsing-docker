# google-safebrowsing-docker
A docker image to host Google Safe Browsing Server: https://github.com/google/safebrowsing

The server requires a Google API key to run, which can be acquired from Google Developer Console: https://console.developers.google.com/

**Example:**  
```
docker run -p 8080:80 christiandt/google-safebrowsing -apikey "APIKEY"
```


## Google Safe Browsing API Extremely Simple Example
A basic example of validating the test-url with a POST request:

**POST:**   
```
127.0.0.1:8080/v4/threatMatches:find

{
    "threatInfo": {
        "threatEntryTypes": ["URL"],
        "threatEntries": [{"url": "http://malware.testing.google.test/testing/malware/"}]
    }
}
```

# From The Official Documentation

## Command sbserver is an application for serving URL lookups via a simple API.

In order to abstract away the complexities of the Safe Browsing API v4, the
sbserver application can be used to serve a subset of API v4 over HTTP.
This subset is intentionally small so that it would be easy to implement by
a client. It is intended for sbserver to either be running locally on a
client's machine or within the same local network. That way it can handle
most local API calls before resorting to making an API call to the actual
Safe Browsing API over the internet.

Usage of sbserver looks something like this:

	             _________________
	            |                 |
	            |  Safe Browsing  |
	            |  API v4 servers |
	            |_________________|
	                     |
	            ~ ~ ~ ~ ~ ~ ~ ~ ~ ~
	               The Internet
	            ~ ~ ~ ~ ~ ~ ~ ~ ~ ~
	                     |
	              _______V_______
	             |               |
	             |   SBServer    |
	      +------|  Application  |------+
	      |      |_______________|      |
	      |              |              |
	 _____V_____    _____V_____    _____V_____
	|           |  |           |  |           |
	|  Client1  |  |  Client2  |  |  Client3  |
	|___________|  |___________|  |___________|

In theory, each client could directly use the Go SafeBrowser implementation,
but there are situations where that is not desirable. For example, the client
may not be using the language Go, or there may be multiple clients in the
same machine or local network that would like to share a local database
and cache. The sbserver was designed to address these issues.

The sbserver application is technically a proxy since it is itself actually
an API v4 client. It connects the Safe Browsing API servers using an API key
and maintains a local database and cache. However, it is also a server since
it re-serves a subset of the API v4 endpoints. These endpoints are minimal
in that they do not require each client to maintain state between calls.

The assumption is that communication between SBServer and Client1, Client2,
and Client3 is inexpensive, since they are within the same machine or same
local network. Thus, the sbserver can efficiently satisfy some requests
without talking to the global Safe Browsing servers since it has a
potentially larger cache. Furthermore, it can multiplex multiple requests to
the Safe Browsing servers on fewer TCP connections, reducing the cost for
comparatively more expensive internet transfers.

By default, the sbserver listens on localhost:8080 and serves the following

    API endpoints:
	    /v4/threatMatches:find
	    /v4/threatLists
	    /status
	    /r


### Endpoint: /v4/threatMatches:find

This is a lightweight implementation of the API v4 threatMatches endpoint.
Essentially, it takes in a list of URLs, and returns a list of threat matches
for those URLs. Unlike the Safe Browsing API, it does not require an API key.

Example usage:

	# Send request to server:
	$ curl \
	  -H "Content-Type: application/json" \
	  -X POST -d '{
	      "threatInfo": {
	          "threatTypes":      ["UNWANTED_SOFTWARE", "MALWARE"],
	          "platformTypes":    ["ANY_PLATFORM"],
	          "threatEntryTypes": ["URL"],
	          "threatEntries": [
	              {"url": "google.com"},
	              {"url": "bad1url.org"},
	              {"url": "bad2url.org"}
	          ]
	      }
	  }' \
	  localhost:8080/v4/threatMatches:find

	# Receive response from server:
	{
	    "matches": [{
	        "threat":          {"url": "bad1url.org"},
	        "platformType":    "ANY_PLATFORM",
	        "threatType":      "UNWANTED_SOFTWARE",
	        "threatEntryType": "URL"
	    }, {
	        "threat":          {"url": "bad2url.org"},
	        "platformType":    "ANY_PLATFORM",
	        "threatType":      "UNWANTED_SOFTWARE",
	        "threatEntryType": "URL"
	    }, {
	        "threat":          {"url": "bad2url.org"},
	        "platformType":    "ANY_PLATFORM",
	        "threatType":      "MALWARE",
	        "threatEntryType": "URL"
	    }]
	}


### Endpoint: /v4/threatLists

The endpoint returns a list of the threat lists that the sbserver is
currently subscribed to. The threats returned by the earlier threatMatches
API call may only be one of these types.

Example usage:

	# Send request to server:
	$ curl -X GET localhost:8080/v4/threatLists

	# Receive response from server:
	{
	    "threatLists": [{
	        "threatType":      "MALWARE"
	        "platformType":    "ANY_PLATFORM",
	        "threatEntryType": "URL",
	    }, {
	        "threatType":      "SOCIAL_ENGINEERING",
	        "platformType":    "ANY_PLATFORM"
	        "threatEntryType": "URL",
	    }, {
	        "threatType":      "UNWANTED_SOFTWARE"
	        "platformType":    "ANY_PLATFORM",
	        "threatEntryType": "URL",
	    }]
	}


### Endpoint: /status

The status endpoint allows a client to obtain some statistical information
regarding the health of sbserver. It can be used to determine how many
requests were satisfied locally by sbserver alone and how many requests
were forwarded to the Safe Browsing API servers.

Example usage:

	$ curl localhost:8080/status
	{
	    "Stats" : {
	        "QueriesByDatabase" : 132,
	        "QueriesByCache" : 31,
	        "QueriesByAPI" : 6,
	        "QueriesFail" : 0,
	    },
	    "Error" : ""
	}


### Endpoint: /r

The redirector endpoint allows a client to pass in a query URL.
If the URL is safe, the client is automatically redirected to the target.
If the URL is unsafe, then an interstitial warning page is shown instead.

Example usage:

	$ curl -i localhost:8080/r?url=http:google.com
	HTTP/1.1 302 Found
	Location: http:google.com

	$ curl -i localhost:8080/r?url=http:bad1url.org
	HTTP/1.1 200 OK
	Date: Wed, 13 Apr 2016 21:29:33 GMT
	Content-Length: 1783
	Content-Type: text/html; charset=utf-8

	<!-- Warning interstitial page shown -->
	...
