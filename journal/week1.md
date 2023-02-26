# Week 1 — App Containerization

## Required Homework

### Dockerized flask app amd react frontend

-  Dockerized Flask backend and React frontend for our project, creating Dockerfiles with necessary dependencies and configurations
-  Used Docker Compose to manage containers and ensure proper linkage
-  Tweaked frontend Dockerfile to automatically run the required number of npm installs 
-  Switched image to alpine variant effectively reducing image size from 1.2GB to ~400MB, effectively making the image more secure by reducing attack surface
- Attempted to optimize React app further by building and deploying build artifact in lean nginx container, reducing final image size to ~40MB. However, app broke and kept calling `/undefined/api/activities/home` instead of the backend app
- Spent most of the week debugging but couldn't resolve issue.

- frontend dockerfile
```Dockerfile
FROM node:16.18-alpine as builder

WORKDIR /intermediate

COPY . .

RUN npm install

FROM node:16.18-alpine

WORKDIR /app

COPY --from=builder /intermediate .

RUN npm install

EXPOSE ${PORT}

CMD ["npm", "start"]
```

- backend dockerfile
```Dockerfile

FROM python:3.10-slim-buster

WORKDIR /backend-flask

COPY requirements.txt requirements.txt

RUN pip3 install -r requirements.txt

COPY . .

ENV FLASK_ENV=development

EXPOSE $PORT

CMD [ "python3", "-m", "flask", "run", "--host=0.0.0.0", "--port=4567" ]

```


### Documented Notification endpoint for OpenAI document

-   Documented notification endpoint for OpenAI document
-   Added new endpoint to YAML file and generated Swagger interactive docs
-   Excited to contribute to OpenAI's mission and continue learning about API documentation best practices.
- [link to source](https://github.com/bxffour/aws-bootcamp-cruddur-2023/blob/main/backend-flask/openapi-3.0.yml)

### Implemented the notifications feature in the app

- Implemented notification feature on both Flask and React sides
- Created notification endpoint on Flask backend to receive and process notifications from external service
- Tested notification feature extensively and confirmed it's working as expected
- Learned a lot about integrating backend and frontend functionality and excited to continue exploring the possibilities of web development.

### Run a Dynamodb and Postgresdb container and ensured it worked

I created a local environment for DynamoDB and PostgreSQL using Docker containers. To do this I:
- Copied the code from the provided source into the docker-compose.yml
- Then I ran docker-compose up to run the containers in the terminal window which started the containers and created a local instance of the databases
- Then I run a few commands to confirm that it works properly
Initially I couldnt connect to the postgres database through the psql CLI client but I was able to solve this by specifying the `--host localhost` and `-U postgres` flags.

