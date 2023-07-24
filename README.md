# Google Cloud SQL Auth Proxy add to Heroku by buildpack - latest version v2.6.0

This heroku buildpack adds the Google Cloud SQL proxy to your app to enable
connections to SQL instances in the GCP.

## Prerequisite

- Google service account with the Cloud SQL/Cloud SQL client role
- JSON key for the service account
- Google Cloud SQL instance set up with a postgresql user able to connect
  to the database

## Install

Add the proxy to your buildpacks. It's important that this buildpack should
not be the last one in the list, as that's used by heroku to determine your
startup processes. (--index=1)

heroku buildpacks:add --index=1 https://github.com/serenadecrimson/google-cloud-sql-auth-proxy-heroku

Add the GCP JSON credentials as `GSP_CREDENTIALS` env variable to you app.


Set the instance the proxy should connect to with the `GSP_INSTANCES` and local port by `GSP_PORT` env variables. 
Examples below
```
GSP_INSTANCES=sacred-tenure-123456:europe-west3:db-name
GSP_PORT=5430
```
You can get the **instance connection name** from the Cloud SQL console overview.

Set the connection string for your DB library to
`postgres://<username>:<password>@localhost:<local port>/<database-name>`

Start the proxy before your app tries to connect to the database by e.g. adding
`bin/run_cloud_sql_proxy` to the `.profile` in the root of your project.

# Changes on 2023-07-24

Flush cache if you already used old version of buildpack from "emartech" on your heroku app 

**Flush the Heroku buildpack CACHE_DIR**

Use heroku-repo plugin purge_cache command with your appname
```
heroku plugins:install heroku-repo
heroku repo:purge_cache -a appname
```

1) Support latest version Google Cloud SQL Auth Proxy v2.6.0
2) Retry exec on fail, using ss command.
3) Log each run of run_cloud_sql_proxy to /tmp folder with date time file name
4) Tested on Heroku-22 stack ( Ubuntu 22.04 )

# Manual run sql_proxy.sh
```
#!/bin/sh
TIMENOW=$(date +'%Y-%m-%d %H:%M:%S')
echo "-----> Starting SQL proxy $TIMENOW on port $GSP_PORT"

printf "%s" "$GSP_CREDENTIALS" > "/app/google/credentials.json"
exec /app/google/bin/cloud_sql_proxy $GSP_INSTANCES --credentials-file /app/google/credentials.json --port $GSP_PORT &
# pause for 5 seconds
sleep 5

# check if proxy is running on local host
ISRUNNING="$(ss -ltH src :$GSP_PORT)"
echo "-----> Status $ISRUNNING"
# -z check on empty string
if [ -z "$ISRUNNING" ]; then
    echo "-----> Restarting SQL proxy"
    exec /app/google/bin/cloud_sql_proxy $GSP_INSTANCES--credentials-file /app/google/credentials.json --port $GSP_PORT &
else
    echo "-----> SQL proxy is running"
fi
```

# More information

Check if Google Cloud SQL Auth Proxy is running on 5430 port
```
ss -ltH src :5430
```

Redirects all stdout and stderr to a virtual device called null to discards data
```
&> /dev/null
```

How to make your file as executable (example for sql_proxy.sh)
```
chmod +x sql_proxy.sh
```

Rebuilded to use in "Cron To Go" addon

# Credit to

Forked from [original buildpack](https://github.com/emartech/heroku-buildpack-cloud-sql-proxy)
