# google-safebrowsing-docker
A docker image to host Google Safe Browsing Server (https://github.com/google/safebrowsing)
The server requires a Google API key to run, which can be acquired from Google Developer Console: https://console.developers.google.com/

Example:
````docker run -p 1080:80 christiandt/google-safebrowsing -apikey "APIKEY"````
