# Fly io secrets to set
```
fly secrets set -a fly-log-shipper-custom DATADOG_API_KEY=*** ACCESS_TOKEN=*** ORG="flipside"
```

If we only want data-app to be sent to datadog, set SUBJECT. Otherwise dont set it.
```
SUBJECT = 'logs.data-app.>'
````