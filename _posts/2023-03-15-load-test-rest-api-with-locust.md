---
title: Use Locust to load test a REST API service 
tags: "testing" "locust"
---

#### Question

Use Locust to load test a REST service with following info:
- need login first to get secCode, userCode and sessionId, share them to all users
- post request to `/v1/prt/getList` with above info
- post to `/v1/prt/getAba` with above info plus countryCode

#### Answer

Sure! Here's an example script using Locust to load test a REST service with the given information:

```python
from locust import HttpUser, task, between

class MyUser(HttpUser):
    wait_time = between(1, 5)

    def on_start(self):
        # login to get secCode, userCode, and sessionId
        response = self.client.post("/login", {"username": "my_username", "password": "my_password"})
        self.secCode = response.json()["secCode"]
        self.userCode = response.json()["userCode"]
        self.sessionId = response.json()["sessionId"]
        self.token = response.json()["token"]

    @task
    def get_list(self):
        # make a POST request to /v1/prt/getList with secCode, userCode, sessionId, and XBA-Token header
        headers = {"XBA-Token": self.token}
        data = {"secCode": self.secCode, "userCode": self.userCode, "sessionId": self.sessionId}
        self.client.post("/v1/prt/getList", headers=headers, data=data)

    @task
    def get_aba(self):
        # make a POST request to /v1/prt/getAba with secCode, userCode, sessionId, countryCode, and XBA-Token header
        headers = {"XBA-Token": self.token}
        data = {"secCode": self.secCode, "userCode": self.userCode, "sessionId": self.sessionId, "countryCode": "US"}
        self.client.post("/v1/prt/getAba", headers=headers, data=data)
```

This script defines a `MyUser` class that inherits from `HttpUser` and sets a wait time between 1 and 5 seconds. The `on_start` method logs in to the REST service and saves the `secCode`, `userCode`, and `sessionId` for future requests. The `get_list` task makes a POST request to `/v1/prt/getList` with the saved login information, and the `get_aba` task makes a POST request to `/v1/prt/getAba` with the saved login information and a hard-coded `countryCode` value of `"US"`. The script also includes an `XBA-Token` header with the saved token value in each request. You can replace `"my_username"` and `"my_password"` with actual login credentials, and the `"US"` value for `countryCode` can be replaced with a valid country code for your REST service.
