#Batch Notebooks Conversion

##About
The `automation.ts` script allows to perform batch conversion of JavaScript notebooks from the Portal to RamlScript notebooks and lunching the converted notebooks.
The automation steps are:
####Cloning the notebook configs [repository](https://github.com/KonstantinSviridov/notebook-configs).
The repository provides following files for each API:

######`path.txt` 
The file contains path of the API GitHub repository

######`config.cfg`
The file contains values prompted by the JavaScript Portal notebook. The format is:
```
{prompt message 1}={value 1}
{prompt message 2}={value 2}
...
```
######`auth.cfg`
The file contains authorization parameters. The format is
```
{
  "{API title} {API version or 'V' if no version}{parameter name}":"{parameter value}",
  ...
}
```



##Sample APIs

####FuelEconomy
No config files required.

####GeoNames
```
auth.cfg:

{
  "GeoNames Vusername" : "your-GeoNames-username"
}

```

####GMail
```
config.cfg:

Please, enter your GMail address=your.gmail.address@gmail.com
```
```
auth.cfg:

{
	"GMail v1clientId": "your client ID",
	"GMail v1clientSecret": "your client Secret",
	"GMail v1accessToken" : "valid OAuth2 access token"
}
```

####Parse
```
auth.cfg:

{
    "Parse 1X-Parse-Application-Id" : "Your application ID",
    "Parse 1X-Parse-REST-API-Key" : "REST API key of your application",
    "Parse 1X-Parse-Master-Key" : "Master key of your application"
}
```

####Slack
```
auth.cfg:

{
    "SlackaccessToken" : "Valid OAuth 2 access token"
}
```

####XKCD
No config files required.

####Zillow
```
auth.cfg:

{
  "Zillow Vzws-id" : "X1-ZWz1b7cczcioln_4pyss"
}
```

