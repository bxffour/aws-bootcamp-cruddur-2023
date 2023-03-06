# Week 2 — Distributed Tracing

### Tracing using honeycomb.io
* To use tracing with Honeycomb.io, I created an account and obtained an API key, which can be exported as a HONEYCOMB_API_KEY environment variable or hardcoded in the application code. I chose to hardcode it instide my code
* To connect to Honeycomb, you should include the required environment variables that utilize the standardized tracing formats built on OpenTelemetry into the back-end service of your application and add them in the docker-compose file under the backend-service.
```yml
OTEL-SERVICE-NAME: 'backend-flask'
OTEL_EXPORTER_OTLP_ENDPOINT: "https://api.honeycomb.io"
OTEL_EXPORTER_OTLP_HEADERS: "x-honeycomb-team=${HONEYCOMB_API_KEY}"
```
* Then I setup the dependencies needed inside the requirements.txt file 
```
opentelemetry-api 
opentelemetry-sdk 
opentelemetry-exporter-otlp-proto-http 
opentelemetry-instrumentation-flask 
opentelemetry-instrumentation-requests
```
* In order to enable tracing and set up an exporter to send data to Honeycomb in my Flask application, I imported the necessary packages from the OpenTelemetry standardized library into my /backend-flask/app.py file. I then included the code that would allow tracing and set up the exporter for Honeycomb.
```python
from opentelemetry import trace
from opentelemetry.instrumentation.flask import FlaskInstrumentor
from opentelemetry.instrumentation.requests import RequestsInstrumentor
from opentelemetry.exporter.otlp.proto.http.trace_exporter import OTLPSpanExporter
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor

...
provider = TracerProvider()
processor = BatchSpanProcessor(OTLPSpanExporter())
provider.add_span_processor(processor)
trace.set_tracer_provider(provider)
tracer = trace.get_tracer(__name__)
```

* To initialize automatic instrumentation for my Flask application, I placed the code after the line that starts the Flask application, as shown.
```python
FlaskInstrumentor().instrument_app(app) 
RequestsInstrumentor().instrument()
```

* I started the application using the docker compose up command to get everything up and running. After hitting a few endpoints, I visited my Honeycomb account and navigated to the right environment to see the traces that were generated.

### Spans
As I worked on distributed tracing, I learned the following:

-   A span represents a logical unit of work done in completing a request or transaction, and it is a single operation within a trace.
-   To obtain a trace each time the /api/activities/home endpoint is accessed, I placed the tracer in the /backend-flask/services/home_activities.py file.
-   To obtain a span in my Flask application, I imported a tracer and utilized its span methods to create spans in my code.
-   The resulting code should look something like this: 

```python
from datetime import datetime, timedelta, timezone
from opentelemetry import trace

tracer = trace.get_tracer("home.activities") 

class HomeActivities:
	def run():
		with tracer.start_as_current_span("home-actvities-mock-data"):
		span = trace.get_current_span()
		now = datetime.now(timezone.utc).astimezone()
		span.set_attribute("app.now", now.isoformat())
		results = [{
				'uuid': '68f126b0-1ceb-4a33-88be-d90fa7109eee',
				'handle': 'Andrew Brown',
				
			...
		
		}
		]
		span.set_attribute("app.result_length", len(results))
		return results
```

### AWS X-RAY
-   To use AWS X-Ray to track traces in my applications, I need to have a sidecar container running an X-Ray daemon alongside my application.
-   The X-Ray daemon is an open-source application that collects trace data from the application and sends it to the AWS X-Ray service for analysis and visualization.
-   I can use either the X-Ray SDK or the X-Ray daemon's API to capture trace data, but in this case, I will be using the X-Ray SDK to instrument tracing for my application.
-   To start initializing AWS X-Ray, I followed the instructions provided in the aws-xray-sdk for Python documentation.
-   I added a line of code to the /backend-flask/requirements.txt file to install the necessary package using the pip command.
```
aws-xray-sdk
```

* I added the following line of code to my `/backend-flask/app.py` file, changing the service name to match the particular service I'm working with: 
```python
from aws_xray_sdk.core import xray_recorder
from aws_xray_sdk.ext.flask.middleware import XRayMiddleware

xray_url = os.getenv("AWS_XRAY_URL")
xray_recorder.configure(service='backend-flask', dynamic_naming=xray_url)
```
* This code imports the necessary X-Ray libraries from the aws-xray-sdk for Python, allowing me to use X-Ray to trace and analyze my application.
* Also I added this line to `/backend-flask/app.py`
```python
XRayMiddleware(app, xray_recorder)
```

* to configure trace sampling in X-Ray, I added a json configuration file in `aws/json/xray.json` with the following contents:

```json
{
  "SamplingRule": {
      "RuleName": "Cruddur",
      "ResourceARN": "*",
      "Priority": 9000,
      "FixedRate": 0.1,
      "ReservoirSize": 5,
      "ServiceName": "backend-flask",
      "ServiceType": "*",
      "Host": "*",
      "HTTPMethod": "*",
      "URLPath": "*",
      "Version": 1
  }
}
```

Note that I replaced `rule_name` with an appropriate name and set the `"service_name"` to `"backend-flask"`, the name of my service.

* I went to the AWS console and navigated to the AWS X-Ray tab. I tried to launch a trace but canceled before actually launching it. This took me back to the AWS X-Ray page, where I clicked on "Configuration" and found "Groups" under it. Then, I created a log group that will keep all the logs generated from the backend service
```bash
aws xray create-group \
	--group-name "Cruddur" \
	--filter-expression "service(\"backend-flask\")"
```
* Creating a sampling rule using the AWS CLI:
```bash
aws xray create-sampling-rule --cli-input-json file://aws/json/xray.json
```

* To run X-Ray as a container alongside our backend application, I made sure to hardcode the region and have the AWS access key and secret access keys available for that environment.
```yaml
xray-daemon:
  image: "amazon/aws-xray-daemon"
  environment:
	AWS_ACCESS_KEY_ID: "${AWS_ACCESS_KEY_ID}"
	AWS_SECRET_ACCESS_KEY: "${AWS_SECRET_ACCESS_KEY}"
	AWS_REGION: "eu-west-2"
  command:
	- "xray -o -b xray-daemon:2000"
  ports:
	- 2000:2000/udp
```

* add these to the docker-compose file
```yaml
AWS_XRAY_URL: "*4567-${GITPOD_WORKSPACE_ID}.${GITPOD_WORKSPACE_CLUSTER_HOST}*"
AWS_XRAY_DAEMON_ADDRESS: "xray-daemon:2000"
```
I ran the docker-compose file to start all the containers, then I opened the AWS X-Ray console and clicked on "Traces" to view the traces that were collected by X-Ray.

### Cloudwatch logs
* To configure CloudWatch logging for my application, I added watchtower to the requirements.txt file in the /backend-flask folder and installed the requirements using the pip command. Then, I added the following code to the app.py file:

```python
import watchtower
import logging
from time import strftime

...

# Configuring Logger to Use CloudWatch
LOGGER = logging.getLogger(__name__)
LOGGER.setLevel(logging.DEBUG)
console_handler = logging.StreamHandler()
cw_handler = watchtower.CloudWatchLogHandler(log_group='cruddur')
LOGGER.addHandler(console_handler)
LOGGER.addHandler(cw_handler)
LOGGER.info("Test Log")
```

This code configures the logger to use CloudWatch for logging. I set the log level to `DEBUG`, added a `StreamHandler` for console output, and added a `CloudWatchLogHandler` for CloudWatch logging with a specified log group name of "cruddur". Finally, I added a test log message using `LOGGER.info("Test Log")`.

* I want edto log every request and error after every request, so I added the following code just before the @app.route("/api/message_groups", methods=['GET']) endpoint using this sample code:
```python
@app.after_request
def after_request(response):
    timestamp = strftime('[%Y-%b-%d %H:%M]')
    LOGGER.error('%s %s %s %s %s %s', timestamp, request.remote_addr, request.method, request.scheme, request.full_path, response.status)
    return response
```

* I'm going to implement logging in my `home_activities.py` file. Here's what my code will look like, using this as a sample code:
```python 
from datetime import datetime, timedelta, timezone
from opentelemetry import trace

tracer = trace.get_tracer("home.activities")

class HomeActivities:
	def run(Logger):
		logger.info("Home Activities")
		with tracer.start_as_current_span("home-actvities-mock-data"):
		span = trace.get_current_span()
```

I first import the `datetime`, `trace`, and `logging` packages. I then get a tracer for my `home.activities` application using the `get_tracer` function from `opentelemetry`.

I define a `HomeActivities` class that has a `run()` method. I create a logger instance using the `getLogger` function from `logging` and set its name to `__name__`. I log an informational message with the `info()` function of my logger instance.

I then create a span using the `start_as_current_span()` method of my tracer instance. I pass a name to this method as an argument, which represents the name of the span. I also get the current span using the `get_current_span()` method of the `trace` package.

* In my Flask application's app.py file, I added an endpoint code using this sample code:
```python
@app.route("/api/activities/home", methods=['GET'])
def data_home():
	data = HomeActivities.run(logger=LOGGER)
	return data, 200
```
* To enable logging of the response(s) generated when we hit the /api/activities/home endpoint, we need to parse the Logger function and register the logs. These logs are then sent to CloudWatch logs for storage and analysis. To enable the app to log into CloudWatch, we need to provide the necessary AWS environment credentials. We can achieve this by adding the required environment variables to the docker-compose file, under the backend service section.
```yaml
AWS_DEFAULT_REGION: "${AWS_DEFAULT_REGION}"
AWS_ACCESS_KEY_ID: "${AWS_ACCESS_KEY_ID}"
AWS_SECRET_ACCESS_KEY: "${AWS_SECRET_ACCESS_KEY}"
```

### Rollar logs

* I created a Rollbar account and went through the process, selecting Flask as the SDK during the setup process. After that, I skipped the "Add Apps" section and proceeded to the "Add SDK" section. I ignored the "Setup SDK" page and instead added the following code to the /backend-flask/requirements.txt file.
```
blinker
rollbar
```
* I added the Rollbar Access Token as an environment variable in my Gitpod workspace. I navigated to the workspace settings and added the access token as a variable. This will allow me to access it in my application without hardcoding it.

```shell
export ROLLBAR_ACCESS_TOKEN="<redacted>"
gp env ROLLBAR_ACCESS_TOKEN="<redacted>"
```

* To reference the access token in the docker-compose file, I added an environment variable `ACCESS_TOKEN` under the `environment` section of the backend service in the docker-compose.yml file, and set its value to the access token needed by the application to access a specific API or service.
```yaml
ROLLBAR_ACCESS_TOKEN: "${ROLLBAR_ACCESS_TOKEN}"
```

* To initialize Rollbar in my Flask application, I can use the following code:
	-   First, I need to import the required packages, which are `os`, `rollbar`, `rollbar.contrib.flask`, and `got_request_exception` from `flask`.
	-   Then, I can initialize Rollbar in my `app.py` file using the following code:
	
```python
from flask import Flask, got_request_exception
import os
import rollbar
import rollbar.contrib.flask

...

app = Flask(__name__)

rollbar_access_token = os.getenv('ROLLBAR_ACCESS_TOKEN')

@app.before_first_request
def init_rollbar():
    """init rollbar module"""
    rollbar.init(
        # access token
        rollbar_access_token,
        # environment name
        'production',
        # server root directory, makes tracebacks prettier
        root=os.path.dirname(os.path.realpath(__file__)),
        # flask already sets up logging
        allow_logging_basic_config=False)

    # send exceptions from `app` to rollbar, using flask's signal system.
    got_request_exception.connect(rollbar.contrib.flask.report_exception, app)

```

* This code will initialize Rollbar in my Flask application and connect it to Flask's signal system, so that any exceptions thrown by the app will be reported to Rollbar. I also need to make sure to set the `ROLLBAR_ACCESS_TOKEN` environment variable in my `.env` file or in my deployment environment, so that Rollbar can authenticate my application.

### Issues
- I struggled very much with a merge conflict.
