# google-safebrowsing-docker
A docker image to host Google Safe Browsing Server: https://github.com/google/safebrowsing

The server requires a Google API key to run, which can be acquired from Google Developer Console: https://console.developers.google.com/

**Example:**  
```
docker run -p 8080:80 christiandt/google-safebrowsing -apikey "APIKEY"
```


## Google Safe Browsing API Example
A basic example of validating the test-url with a POST request:

**POST:**   
```
127.0.0.1:8080/v4/threatMatches:find
```

**BODY:**  
```
{
    "threatInfo": {
        "threatEntryTypes": ["URL"],
        "threatEntries": [{"url": "http://malware.testing.google.test/testing/malware/"}]
    }
}
```
