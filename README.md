# Proxy GPT

A GPT backend powered by OpenAI

## Overview

This is a GPT backend for the OpenAI API. All requests are forwarded using a shared key. The requests are logged in a database for 
filtering, auditing, etc. It's a standard Go program that can be deployed anywhere using sqlite or a postgres database for persistence.

## Features

- OpenAI shared proxy
- User authentication
- 1:1 and 1:N Chat API
- SQLite or Postgres storage
- In-memory or Redis caching
- Proxy request and event log
- Prompt context cache forwarding

## Usage

- [Install](#install)
- [API](#api)
- [Auth](#auth)
- [Admin](#admin)
- [Database](#database)
- [Redis](#redis)
- [User API](#user-api)
- [Chat API](#chat-api)
- [Team API](#team-api)
- [Endpoints](#endpoints)
- [TODO](#todo)

### Install

Built as a Go binary

```
go build -o proxy-gpt ./main.go
```

### API

Requires an API key as env var `API_KEY` for access to OpenAI.

```
API_KEY=xxx proxy-gpt
```

Runs on 8080, proxies `/v1/*` to OpenAI verbatim

```
curl http://localhost:8080/v1/models
```

See [OpenAI API reference](https://platform.openai.com/docs/api-reference/completions) for details

#### Azure OpenAI

To use Azure's OpenAI service, provide the `API_URL` environment variable in addition to the `API_KEY`

```
API_URL=https://YOUR_RESOURCE_NAME.openai.azure.com/openai/deployments/YOUR_DEPLOYMENT_NAME/completions?api-version=2022-12-01
```

See [Azure OpenAI Reference](https://learn.microsoft.com/en-us/azure/cognitive-services/openai/reference)

#### Custom URL

To use a custom url that supports the OpenAI API specify the `API_URL` as mentioned above for azure

e.g custom local mocked LLM proxy

```
API_URL=http://localhost:9090
```

### Auth

The API supports session based authentication using a `sess` cookie header or `Authorization: Bearer $TOKEN` header.

Example of an API call using authentication header

```
curl -H 'Authorization: Bearer ZDU0Nzg5ZTctMzRkMy00ZmNlLTkyYTgtZTQwYzIxZDE1YWJm' \
http://localhost:8080/v1/models
```

Example of a cookie based call

```
curl --cookie 'sess=ZDU0Nzg5ZTctMzRkMy00ZmNlLTkyYTgtZTQwYzIxZDE1YWJm' \
http://localhost:8080/v1/models
```

See the [User API](#user-api) section for more on signup, login, etc.

### Admin

A user admin command line is available in `cmd/admin`

It requires access to the database via the `--database` flag (if postgres is used)

List users

```
admin list
``` 

Get user by id

```
admin user user-1
```

Reset password

```
admin reset foobar Password1
```

### Database

Events are stored in a database. Sqlite is used by default (local proxy.db file).

To use Postgres specify the URL as `DB_ADDRESS` env var

```
DB_ADDRESS=postgresql://user:pass@localhost:5432/proxy
```

#### Privileges

Requires the following privileges assuming user is `asim`

```
create database proxy;
create user gptio with encrypted password 'foobar';
grant all privileges on database proxy to gptio;
GRANT ALL ON SCHEMA public to gptio;
```

#### Tables

- chats - stores the chat history
- chat_users - users in the chat by id
- Events - proxy events/requests/login/etc
- messages - message history within chats
- teams - all the team information
- team_members - team members by id
- users - user login information
- sessions - current login sessions

### Redis

Context is cached in memory by default for up-to 10 prior prompts. This can be modified by request to `/chat/prompt` with 
the `context` field set to an integer above 0. The cache is built from the database of prior messages if no context is in 
memory. 

Redis can be used as an alternative persistent cache. This will enable horizontally scaling the proxy alongside the use of 
the external database like postgres. To do so specify the `REDIS_ADDRESS` env var with the connection string.

```
REDIS_ADDRESS=redis://localhost:6379
```

### User API

Signup and authentication is handled via cookies or token based header

#### Endpoints

The following endpoints are used

- `/user/signup` - register a user with username/password fields
- `/user/login` -  login with username/password fields
- `/user/logout` - call with `sess` cookie header set

#### Signup

A user can register via `/user/signup` endpoint

```
curl http://localhost:8080/user/signup \
-d "username=asim&password=bazbar"
```

Fields to set

- `username` - valid username
- `password` - valid password
- `first_name` - first name of user (optional)
- `last_name` - last name of user (optional)
- `redirect_url` - where to send after successful signup (optional)

#### Login

Login via `/user/login` with post form data. Will set the cookie `sess` with an opaque token and return as json

```
curl -vv http://localhost:8080/user/login \
-d "username=asim&password=bazbar"
```

Fields to set

- `username` - valid username
- `password` - valid password
- `redirect_url` - where to send after successful signup (optional)

#### Logout

Logout via `/user/logout` with `sess` cookie header set

```
curl --cookie "sess=YWEzZTlkYTUtZWRhNi00ODY3LWIyNzYtZGFhNGRhMmRlNmEx" \
http://localhost:8080/user/logout
```

Alternatively using a token

```
curl -XPOST http://localhost:8080/user/logout \
-d '{"token": "YWEzZTlkYTUtZWRhNi00ODY3LWIyNzYtZGFhNGRhMmRlNmEx"}'
```

#### Sessions

Based on this login session calls to the `/v1/*` endpoint can be made via a `sess` cookie set in the header. 
Stored in browser cookies or via `curl --cookie` or it can be used via the `Authorization: Bearer $TOKEN` header.

Example of API call

```
curl -H 'Authorization: Bearer ZDU0Nzg5ZTctMzRkMy00ZmNlLTkyYTgtZTQwYzIxZDE1YWJm' \
http://localhost:8080/v1/models
```

Example of a cookie based call

```
curl --cookie 'sess=ZDU0Nzg5ZTctMzRkMy00ZmNlLTkyYTgtZTQwYzIxZDE1YWJm' \
http://localhost:8080/v1/models
```

You can otherwise specify the username:token in the URL as basic auth.

## Chat API

The chat API is a slim layer on top of OpenAI endpoints to store conversations locally. 
It takes standard POST requests and returns JSON responses.

- `/chat/create` - creates a new chat (returns the chat id as `id`)
- `/chat/delete` - deletes a chat, takes `id` param (returns nil response)
- `/chat/index` - lists all chats for a given user (returns `chats` as an array)
- `/chat/read` - provides chat history, takes `id` as param (returns `chat` and `messages` array)
- `/chat/prompt` - make a request using `prompt` command and `id` (returns `reply` text and store in db)
- `/chat/stream` - stream via SSE or websockets using `ChatID` and `token` as params`
- `/chat/user/add` with `chat_id` and `user_id`
- `/chat/user/remove` with `chat_id` and `user_id`


### Create the chat

Create a chat and specify the model as `gpt-3` or `gpt-4`

```
curl http://localhost:8080/chat/create \
-d "name=foobar&model=gpt-3"
```

### Add Users

To add users to the chat

```
curl http://locahost:8080/chat/user/add \
-d "chat_id=chat-1&user_id=user-1"
```

### Send a message

Send a message to the chat

```
curl http://localhost:8080/chat/prompt \
-d "chat_id=chat-1&prompt=tell+me+about+spain"
```

The request will be made inline and response provided

### Stream messages

To stream messages asynchonrously specify `stream=bool` to the `/chat/prompt` endpoint. 

Messages will be streamed over the `/chat/stream` endpoint which you can separately 
subscribe to using server sent events or websockets.

```
Example TODO
```

Specify `id` for the chat ID you want to stream from. Messages will be received in the 
format below. Where `partial` is set to true, this is the partial response of separated 
words from the model. When this is set to false, the full message will be present.

Stream essage format

```
{
  "message": {"id": "uuid", "prompt": "your prompt", "reply": "words ..." },
  "partial": true
}
```

### Off the record

Send messages to the chat which are not sent to the AI but used as context 
later with the `otr=bool` flag to `/chat/prompt`.

## Team API

The team API enables you to create organisations that have their own members and chats.

### Create a team

```
curl http://localhost:8080/team/create \
-d "name=foobar&description=my+awesome+team"
```

### Add a member

```
curl http://localhost:8080/team/members/add \
-d "team_id=team-1&user_ids=user-1"
```

### Create a team chat

```
curl http://localhost:8080/chat/create \
-d "name=foobar&team_id=team-1"
```

## Endpoints

A full list of API endpoints

```
// chat api
"/chat/create":      ChatCreate,
"/chat/read":        ChatRead,
"/chat/update":      ChatUpdate,
"/chat/delete":      ChatDelete,
"/chat/prompt":      ChatPrompt,
"/chat/index":       ChatIndex,
"/chat/stream":      ChatStream,
"/chat/user/add":    ChatUserAdd,
"/chat/user/remove": ChatUserRemove,

// team api
"/team/create":         TeamCreate,
"/team/delete":         TeamDelete,
"/team/read":           TeamRead,
"/team/update":         TeamUpdate,
"/team/index":          TeamIndex,
"/team/members":        TeamMembers,
"/team/members/add":    TeamMembersAdd,
"/team/members/remove": TeamMembersRemove,

// user api
"/user/signup":          UserSignup,
"/user/login":           UserLogin,
"/user/logout":          UserLogout,
"/user/read":            UserRead,
"/user/update":          UserUpdate,
"/user/session":         UserSession,
"/user/password/update": UserPasswordUpdate,
```

### Request Format

The API itself supports two formats, either POST form encoded data, or application/json

In the event you send post data, the request looks something like

```
curl http://localhost:8080/user/signup \
-d "username=foo&password=bar"
```

In the case you are using JSON then something like the following

```
curl http://localhost:8080/user/signup \
-H 'Content-Type: application/json'
-d '{"username": "foo", "password": "bar"}'
```
#### Response Format

Responses are all of the format `application/json`

## TODO

- Generate API Docs using [Swag](https://github.com/swaggo/swag)
- Saving prompts for sharing/reuse
- Example web app or api usage
- Basic SDKs for js, go, etc
- More documentation!!!
