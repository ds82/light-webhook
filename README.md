# light-webhook

Light webhook client.  
Receives requests from Github, Gitlab, or HTTP and executes HTTP request or Bash command.

[![Build Status](https://travis-ci.org/rpenco/light-webhook.svg?branch=master)](https://travis-ci.org/rpenco/light-webhook)
[![npm version](https://badge.fury.io/js/light-webhook.svg)](https://badge.fury.io/js/light-webhook)

## Usage

```bash

npm i -g light-webhook

light-webhook --config=./config.json
```

| option    | description                       |
|-----------|-----------------------------------|
| --config  | path to configuration file (JSON) |
| --port    | server port. Default 8080         |
| --help    | help!                             |


Create a configuration file `config.json`.

```json

{
  "webclient1": {
    "subscribe": [
          {
            "service": "github",
            "name": "my-github",
            "settings":{
              "events": ["push"],
              "secret": "helloworld"
            }
          }
      ],
      "publish": [
        {
          "service": "bash",
          "description": "Execute my local script",
          "settings": {
            "cmd": [ "echo", "'event: {{headers.x-github-event}}, url: {{body.hook.url}}'"]
          }
        }
      ]
    }
}
```

And configure [a github webhook](https://developer.github.com/webhooks/creating/) to point to `http://example.com:8080/webclient1/my-github`.  

That's all, when you `push` your changes in your git repository, `light-webhook` app is triggered and execute the `echo` bash command. 

You can now execute any action (bash, http request) from any http event (Github, Gitlab, Custom Http) ! 

## Configuration

### Generic structure

The configuration has a generic structure. 

- A *client* with an unique identifier which contains 
    - a *subscribe* list of *webhook listener* called **service receiver**
    - a *publish* list of *action to perform* called **service emitter**

```json
[
    {
      "client1": {
                           
       "subscribe": [   
           {
             "service": "", 
             "name": "", 
             "settings": {} 
           }
       ],
       "publish": [ 
         {
           "service": "", 
           "name": "", 
           "settings": {} 
         }
       ]
     }
    }
]
```


## Receivers (subscribers)

### Github

```json
{
  "service": "github",
  "name": "my-github",
  "settings":{
    "events": ["push", "merge_request"],
    "secret": "mysecret" 
  }
}
```

| option        | description                       |
|---------------|-----------------------------------|
| service       | service used : `github`           |
| name          | choose an unique service name     |
| settings      | service configuration             |
| settings.events | Accepted Github events. Use `['*']` to allow all events |
| settings.secret | Secret provided by `X-Hub-Signature` header. Or `false` to disabled it. |


### Gitlab

```json
{
  "service": "gitlab",
  "name": "my-gitlab",
  "settings":{
    "events": ["push", "merge_request"]
  }
}
```

| option        | description                       |
|---------------|-----------------------------------|
| service       | service used : `gitlab`           |
| name          | choose an unique service name     |
| settings      | service configuration             |
| settings.events | Accepted Gitlab events. Use `['*']` to allow all events |


### Custom HTTP POST

```json
 {
    "service": "http",
    "name": "my-custom-hook",
    "settings":{
     "events": ["my-event"], 
      "secret": "sha1-secret"
    }
}
```

| option        | description                       |
|---------------|-----------------------------------|
| service       | service used : `http`             |
| name          | choose an unique service name     |
| settings      | service configuration             |
| settings.events | Accepted events. Use `['*']` to allow all events |
| settings.secret | Secret provided by `X-Webhook-Signature` header. Or `false` to disabled it.|

An header `X-Webhook-Event: my-event` must be provided to match with settings events.
if `secret` is provided, an header `X-Webhook-Signature: sha1=xxxxxxxxx` must be provided where *xxxxxxxxx* is the secret cyphered.


```bash
curl -X POST \
     -H 'Content-Type: application/json' \
     -H 'X-Webhook-Event: my-event' \ 
     -H "X-Webhook-Signature: sha1=aaf4c61ddcc5e8a2dabede0f3b482cd9aea9434d" \ 
     -d '{"repository": "myrepo", "build": 1}' \
     http://localhost:8080/myclient/my-custom-hook
```

## Emitters (publishers)

### Variables

You can use **variables** from your *receiver* body and your *emitter* settings.
 
Use `{{brackets}}` to identify a variable in your configuration. For exemple:


| option        | description                       |
|---------------|-----------------------------------|
| {{body.}}     | Variable from *receiver body*     |
| {{headers.}}  | Variable from *receiver headers*  |
| {{params.}}  | Variable from *receiver query params* |


You can for example create dynamic bash command 

```json
{
  "service": "bash",
  "name": "my-script",
  "settings":{
    "cmd": ["./my-script.sh", "{{headers.x-github-event}}" , "{{body.repository.url}}"]
  }
}
```

Will execute my custom script: `./my-script.sh push https://github.com/rpenco/light-webhook`

### Bash 

You can call a local bash command.

```json
{
  "service": "bash",
  "name": "bash-cmd",
  "settings":{
    "cmd": ["echo", "'event: {{headers.x-github-event}}, body:{{body}}'"],
    "stringify": true
  }
}
```

or call a bash script

```json
{
  "service": "bash",
  "name": "my-script",
  "settings":{
    "cmd": ["/home/me/my-script.sh"]
  }
}
```

| option        | description                       |
|---------------|-----------------------------------|
| service       | service used : `bash`             |
| name          | choose an unique service name     |
| settings      | service configuration             |
| settings.cmd  | Bash command to execute.          |
| settings.stringify | If `true`, when you use a variable, if it is an object, it will be serialized. |


### Custom HTTP request

You can create an other webhook.

```json
{
  "service": "http",
  "name": "my-http-req",
  "settings":{
    "method": "GET",
    "url": "http://myotherservice.com",
    "params": {"key": "value"},
    "headers": {"key": "value"}
  }
}
```
| option        | description                       |
|---------------|-----------------------------------|
| service       | service used : `http`             |
| name          | choose an unique service name     |
| settings      | service configuration             |
| settings.method  | HTTP method.                   |
| settings.url  | Remote url to call                |
| settings.params    | `body` request in case of `post` or `put`.|
| settings.headers   | request headers.|
| settings.stringify | If `true`, when you use a variable, if it is an object, it will be serialized. |
