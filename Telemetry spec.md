# Telemetry spec

Goals:
- Collect number of requests a Passenger instance made
- Segmented by OSS/enterprise
- Segmented by version [MODIFIED]
- maybe: collect feature usage? [MODIFIED]
- OS en versie [MODIFIED]

TelemetryCollector class:
- First trigger in 0.5-3.5 hour [MODIFIED]
- Trigger before shutdown, timeout 2s
- Time till next trigger is determined by server response
	- If no server response, trigger every 6-9 hours (random to avoid thundering herd)
- On every send:
	- Iterate over all controllers, query number of requests handled, sum everything
	- Send quantity difference and time difference since last send
	- Beware of integer wraps
	- log message with INFO level, make it clear it is anonymous

Protocol:
- Endpoint: https://anontelemetry.phusionpassenger.com/v1/collect.json
- Request:

      {
        "requests_handled": <number>,
        "begin_time": <timestamp>,
        "end_time": <timestamp>,
        "edition": "oss" | "enterprise",
        "version": <version> [MODIFIED]
      }
- Response (200, 400, 422, 500):

      {
        "data_processed": <boolean>, // should client keep or throw away stats?
        ["backoff": <seconds>,] // when to send next data point?
        ["log_message": <string>] // to be logged by passenger
      }

Server:
- Tables: oss_stats, enterprise_stats
	- date (unique)
	- requests NUMERIC(32, 0)
- on receiving a data point:
	- maybe split a single data point into multiple date buckets
	- upsert date bucket, increment requests
- (future) every week, synchronize last month with google spreadsheets
