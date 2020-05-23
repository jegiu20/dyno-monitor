# Heroku Dyno Monitoring & Alerting

This project is useful for keeping a tab on heroku dyno errors and performance metrics across your different heroku apps. In its final form, the app will contain the following (hopefully):

### Error Monitoring

- :heavy_check_mark: **Generic Dyno Error** Monitoring (R13, R14 memory errors and other Rxx errors). You can configure the Rxx Error Rules from the admin interface
- :heavy_check_mark: **Web Specific Dyno Error** Monitoring (H12, H13, H18 and other Hxx errors). Configuration via admin interface
- :hourglass_flowing_sand: **Web Dyno Failed Requests** Monitoring (5xx errors)

### Alerting

- :heavy_check_mark: **Email alerts** Setup email by entering your SMTP server details and recipients. Then configure alerts from the admin dashboard
- :hourglass_flowing_sand: **SMS alerts** Configure SMS alerts by entering your Infobip SMS provider details. If you use some other service (twilio, msg91 etc) you can easily plugin your own implementation

### Actions

- :heavy_check_mark: **Restart actions** Perform basic recovery actions like dyno restart or app restart when a certain alert is breached

### Metrics Collection

For dynos that have metrics logging enabled, the following metrics will be collected
- :soon: **RAM usage** metric per dyno type. Collection and logging
- :soon: **Load %CPU** per dyno type. Collection and logging

### Status Checks

- :soon: **Web server** status checks. Specify the endpoint to check, frequency and timeout
- :soon: **Redis instance** status checks. Specify redis url, list name & threshold (for QL checking if required), frequency and timeout
- :soon: **Postgres instance** status check. Specify the DB url, table name (to check for existence if required), frequency and timeout

-----------------
## Quick Start

#### Deploy to Heroku

To quickly get started and test the app, you can deploy this application on your heroku account servers. Fill up the pre-requisite environment variables and your app should be up within 5 minutes

[![Deploy](https://www.herokucdn.com/deploy/button.svg)](https://heroku.com/deploy?template=https://github.com/carnot-technologies/dyno-monitor/tree/master)

- Post deployment, the app creates all the required tables and the superuser account
- It auto-detects all your heroku apps and dynos. This process can take upto a minute depending on the number of apps in your account
- You can then visit your app's admin page at https://<your-app-name>.herokuapp.com/admin/ and start adding `Hxx Error` and `Rxx Error` rules against your dynos

#### Environment Configuration

- Required environment variables
  - `HEROKU_API_KEY` - To access the logs from different heroku apps in your accounts for which alerts have been configured
  - `DJANGO_SUPERUSER_USERNAME`, `DJANGO_SUPERUSER_PASSWORD`, `DJANGO_SUPERUSER_EMAIL` - These three variables are required at the start to create a superuser. Once a superuser is created, these can be removed

- To enable emails, make sure the following environment variables are exported
  ```bash
  heroku config:set ENABLE_EMAILS=1
  heroku config:set EMAIL_HOST=email-server.abc.com --app <YOUR_APP_NAME>
  heroku config:set EMAIL_PORT=587 --app <YOUR_APP_NAME>
  heroku config:set EMAIL_HOST_PASSWORD=xxxxxxxxxxxxxxxxxxxxx --app <YOUR_APP_NAME>
  heroku config:set EMAIL_HOST_USER=xxxxxxxxxxxxxxxxxxxxx --app <YOUR_APP_NAME>
  heroku config:set SERVER_EMAIL=postmaster@abc.com --app <YOUR_APP_NAME>
  heroku config:set RECIPIENTS=admin.one@abc.com,admin.two@abc.com --app <YOUR_APP_NAME>
  ```

-----------------

## How to configure and use?

This is a python / django based application. Broadly you need the following to set it up:
- `Python 3` - we have verified it on Python 3.6
- `virtualenv` - or similar program to create and manage your virtual environment
- `postgres` - access to a small postgres server instance for data logging and storage. RAM required less than 1GB
- `redis` - a small fast-access cache for storing certain details. Less than 25 MB

### Configure the environment

- Setup the virtualenv and install all dependencies
- Much of the application is controlled by its environment variables. A few variables are necessary, others are optional
- Mandatory environment variables

### Initial setup scripts

- WIP

--------------------

## Log access and samples

### Accessing heroku logs

- Ideally the _right approach_ to consume heroku logs is to setup a [Syslog or HTTPS log drain](https://devcenter.heroku.com/articles/log-drains) and have a dedicated log server to parse the logs for you
- But our basic assumption is we will not read a lot of (typically) large logs from applications, but look at the few but highly useful logs generated under `source=heroku`
- We can get this stream of logs using heroku3 python library
```python
import heroku3
app = heroku3.from_key(<YOUR_API_KEY>).app(<YOUR_APP_NAME>)
app.stream_log(source="app", dyno="worker", lines=10)
```

### Error Codes Log

A set of errors from the platform we will aim to detect. Please see the [full set of error codes](https://devcenter.heroku.com/articles/error-codes)

- H10 - App Crashed (`source=heroku`, `dyno=router`)
  ```
  2010-10-06T21:51:12-07:00 heroku[router]: at=error code=H10 desc="App crashed" method=GET path="/" host=myapp.herokuapp.com fwd=17.17.17.17 dyno= connect= service= status=503 bytes=
  ```

- H12 - Request Timeout (`source=heroku`, `dyno=router`)
  ```
  2010-10-06T21:51:37-07:00 heroku[router]: at=error code=H12 desc="Request timeout" method=GET path="/" host=myapp.herokuapp.com fwd=17.17.17.17 dyno=web.1 connect=6ms service=30001ms status=503 bytes=0
  ```

- R14 - Memory Quota Exceeded (`source=heroku`, `dyno=worker`)
  ```
  2011-05-03T17:40:10+00:00 heroku[worker.1]: Process running mem=1028MB(103.3%)
  2011-05-03T17:40:11+00:00 heroku[worker.1]: Error R14 (Memory quota exceeded)
  ```

- R15 - Memory Quota Vastly Exceeded (`source=heroku`, `dyno=worker`)
  ```
  2011-05-03T17:40:10+00:00 heroku[worker.1]: Process running mem=1029MB(201.0%)
  2011-05-03T17:40:11+00:00 heroku[worker.1]: Error R15 (Memory quota vastly exceeded)
  2011-05-03T17:40:11+00:00 heroku[worker.1]: Stopping process with SIGKILL
  2011-05-03T17:40:12+00:00 heroku[worker.1]: Process exited
  ```

- H80 - Maintenance Mode (`source=heroku`, `dyno=router`)
  ```
  2010-10-06T21:51:07-07:00 heroku[router]: at=info code=H80 desc="Maintenance mode" method=GET path="/" host=myapp.herokuapp.com fwd=17.17.17.17 dyno= connect= service= status=503 bytes=
  ```

- H99 or R99 - Platform Error (`source=heroku`, `dyno=router`)
  ```
  2010-10-06T21:51:07-07:00 heroku[router]: at=error code=H99 desc="Platform error" method=GET path="/" host=myapp.herokuapp.com fwd=17.17.17.17 dyno= connect= service= status=503 bytes=
  ```

### General Metrics Log

- Web Request Logs for web dynos (`source=heroku`, `dyno=router`)
  ```
  2020-05-21T09:27:33.001084+00:00 heroku[router]: at=info method=POST path="/api/card/964/query" host=carnot-metabase.herokuapp.com request_id=0b27af66-8bee-4f7f-bca9-96a7b3851130 fwd="115.96.38.61" dyno=web.1 connect=1ms service=40ms status=200 bytes=1811 protocol=https
  2020-05-21T09:27:33.334719+00:00 heroku[router]: at=info method=GET path="/app/assets/img/no_results.svg" host=carnot-metabase.herokuapp.com request_id=7b451cee-5bfe-449b-af67-ee54b6c469ea fwd="115.96.38.61" dyno=web.1 connect=1ms service=12ms status=200 bytes=3274 protocol=https
  ```


- Web dyno metrics (`source=heroku`, `dyno=web`)
  ```
  2020-05-21T09:23:13.271104+00:00 heroku[web.1]: source=web.1 dyno=heroku.103402527.ea2fc07d-d807-4864-a2c3-5cac3531c914 sample#load_avg_1m=0.00 sample#load_avg_5m=0.01 sample#load_avg_15m=0.06
  2020-05-21T09:23:13.271104+00:00 heroku[web.1]: source=web.1 dyno=heroku.103402527.ea2fc07d-d807-4864-a2c3-5cac3531c914 sample#memory_total=906.01MB sample#memory_rss=835.69MB sample#memory_cache=0.29MB sample#memory_swap=70.03MB sample#memory_pgpgin=485104pages sample#memory_pgpgout=282335pages sample#memory_quota=1024.00MB
  ```

- Worker dyno metrics (`source=heroku`, `dyno=worker`)
  ```
  2020-05-21T13:37:36.303415+00:00 heroku[worker.1]: Starting process with command `python worker-manager.py`
  2020-05-21T13:37:36.970517+00:00 heroku[worker.1]: State changed from starting to up
  2020-05-21T13:37:51.999653+00:00 heroku[worker.1]: source=worker.1 dyno=heroku.80178655.00ef45ff-41d2-40c8-87c8-a1f847fe2576 sample#memory_total=407.75MB sample#memory_rss=392.41MB sample#memory_cache=15.34MB sample#memory_swap=0.00MB sample#memory_pgpgin=129484pages sample#memory_pgpgout=27656pages sample#memory_quota=512.00MB
  2020-05-21T13:38:13.88587+00:00 heroku[worker.1]: source=worker.1 dyno=heroku.80178655.00ef45ff-41d2-40c8-87c8-a1f847fe2576 sample#memory_total=409.36MB sample#memory_rss=394.02MB sample#memory_cache=15.34MB sample#memory_swap=0.00MB sample#memory_pgpgin=129906pages sample#memory_pgpgout=27666pages sample#memory_quota=512.00MB
  2020-05-21T13:38:35.665206+00:00 heroku[worker.1]: source=worker.1 dyno=heroku.80178655.00ef45ff-41d2-40c8-87c8-a1f847fe2576 sample#load_avg_1m=0.30
  2020-05-21T13:38:35.665308+00:00 heroku[worker.1]: source=worker.1 dyno=heroku.80178655.00ef45ff-41d2-40c8-87c8-a1f847fe2576 sample#memory_total=409.54MB sample#memory_rss=394.20MB sample#memory_cache=15.34MB sample#memory_swap=0.00MB sample#memory_pgpgin=130055pages sample#memory_pgpgout=27769pages sample#memory_quota=512.00MB
  ```

- Heroku Postgres Health logs (`source=app`, `dyno=heroku-postgres`)
  ```
  2020-05-21T09:31:00+00:00 app[heroku-postgres]: source=HEROKU_POSTGRESQL_ROSE addon=postgresql-clear-67109 sample#current_transaction=2445712089 sample#db_size=37066259591bytes sample#tables=150 sample#active-connections=84 sample#waiting-connections=0 sample#index-cache-hit-rate=0.98701 sample#table-cache-hit-rate=0.96767 sample#load-avg-1m=0.06 sample#load-avg-5m=0.115 sample#load-avg-15m=0.165 sample#read-iops=17.711 sample#write-iops=6.7273 sample#tmp-disk-used=33849344 sample#tmp-disk-available=72944943104 sample#memory-total=4044972kB sample#memory-free=87596kB sample#memory-cached=3350760kB sample#memory-postgres=375912kB     
  2020-05-21T09:30:00+00:00 app[heroku-postgres]: source=HEROKU_POSTGRESQL_RED addon=postgresql-tapered-27885 sample#current_transaction=867971978 sample#db_size=33737964679bytes sample#tables=190 sample#active-connections=12 sample#waiting-connections=0 sample#index-cache-hit-rate=0.99551 sample#table-cache-hit-rate=0.62029 sample#load-avg-1m=0.145 sample#load-avg-5m=0.105 sample#load-avg-15m=0.115 sample#read-iops=0.016949 sample#write-iops=0.10169 sample#tmp-disk-used=33849344 sample#tmp-disk-available=72944943104 sample#memory-total=4044980kB sample#memory-free=399344kB sample#memory-cached=3297112kB sample#memory-postgres=68888kB
  ```

- Actual Postgres Instance logs (`source=app`, `dyno=postgres`)
  ```
  2020-05-21T09:33:12+00:00 app[postgres.26875]: [RED] [1846262-3]  sql_error_code = 23503 STATEMENT:  COMMIT
  2020-05-21T09:33:12+00:00 app[postgres.26875]: [RED] [1846263-1]  sql_error_code = 23503 ERROR:  insert or update on table "devices_devicelatestdata" violates foreign key constraint "devices_devicelatest_device_fk_id_c404589b_fk_devices_d"
  2020-05-21T09:33:12+00:00 app[postgres.26875]: [RED] [1846263-2]  sql_error_code = 23503 DETAIL:  Key (device_fk_id)=(5188) is not present in table "devices_device".
  ```
