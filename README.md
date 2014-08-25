# OAuth2Ex [![Build Status](https://secure.travis-ci.org/parroty/oauth2ex.png?branch=master "Build Status")](http://travis-ci.org/parroty/oauth2ex)


An OAuth 2.0 client library for elixir. It provides the following functionalities.
- OAuth token retrieval by communicating with OAuth 2.0 server.
- Caching the acquired token locally, and refreshing the token when the it's expired.
- HTTP client access by specifying OAuth2 access token. It uses httpoison (https://github.com/edgurgel/httpoison) as http client library.

It's pretty much work in progress yet, and APIs will likely to be change.
The `OAuth2Ex.Sample` modules contains example for several API servers, like Google, GitHub and Dropbox.

### Setup
Specify :oauth2ex in the `appliations` and `deps" section in the mix.exs.

```Elixir
def application do
  [ applications: [:logger, :oauth2ex] ]
end

defp deps do
  [
    {:oauth2ex, github: "parroty/oauth2ex"}
  ]
end
```

### Usage
The following is an example to call Google's BigQuery API.

#### Manual token retrieval using browser
An example to uses OAuth2Ex helper methods to retrieve OAuth token.

```Elixir
# Setup config parameters (retrive required parameters from OAuth 2.0 providers).
config = OAuth2Ex.config(
  id:            System.get_env("GOOGLE_API_CLIENT_ID"),
  secret:        System.get_env("GOOGLE_API_CLIENT_SECRET"),
  authorize_url: "https://accounts.google.com/o/oauth2/auth",
  token_url:     "https://accounts.google.com/o/oauth2/token",
  scope:         "https://www.googleapis.com/auth/bigquery",
  callback_url:  "urn:ietf:wg:oauth:2.0:oob",
  token_store:   %OAuth2Ex.FileStorage{file_path: System.user_home <> "/oauth2ex.google.token"}
)
# -> %OAuth2Ex.Config{authorize_url: "https://accounts.google.com/o/oauth2/auth"...

# Get authentication parameters.
IO.puts OAuth2Ex.get_authorize_url(config)
# -> https://accounts.google.com/o/oauth2/auth?client_id=1...
#    Open this url using browser and acquire code string.

# Acquire code from browser and a get access token using the code.
code = "xxx..."
token = OAuth2Ex.get_token(config, code)
# -> %OAuth2Ex.Token{access_token: "xxx.......",
#    expires_at: 1408467022, expires_in: 3600,
#    refresh_token: "yyy....",
#    token_type: "Bearer"}

# Access API server using token.
response = OAuth2Ex.HTTP.get(token, "https://www.googleapis.com/bigquery/v2/projects")
# -> %HTTPoison.Response{body: "{\n \"kind\": \"bigquery#projectList...

# Save token to a file for later use.
OAuth2Ex.Token.save(token)

# Load previously saved token from the file.
token = OAuth2Ex.Token.load(
          %OAuth2Ex.FileStorage{file_path: System.user_home <> "/oauth2ex.google.token"})

# Refresh access_token from refresh_token.
token = OAuth2Ex.refresh_token(config, token)
```

#### Automatic token retrieval using local callback server.
An example to uses local server for automating the token retrieval using OAuth2Ex.Client module.

```Elixir
# Setup config parameters (retrive required parameters from OAuth 2.0 providers).
config = OAuth2Ex.config(
  id:            System.get_env("GOOGLE_API_CLIENT_ID"),
  secret:        System.get_env("GOOGLE_API_CLIENT_SECRET"),
  authorize_url: "https://accounts.google.com/o/oauth2/auth",
  token_url:     "https://accounts.google.com/o/oauth2/token",
  scope:         "https://www.googleapis.com/auth/bigquery",
  callback_url:  "http://localhost:4000",
  token_store:   %OAuth2Ex.FileStorage{file_path: System.user_home <> "/oauth2ex.google.token"}
)
# -> %OAuth2Ex.Config{authorize_url: "https://accounts.google.com/o/oauth2/auth"...

# Retrieve token from server. It opens authorize_url using browser,
# and then waits for the callback on the local server on port 4000.
token = OAuth2Ex.Token.browse_and_retrieve!(config, receiver_port: 4000)
# -> %OAuth2Ex.Token{access_token: "..."

# Access API server using token.
response = OAuth2Ex.HTTP.get(token, "https://www.googleapis.com/bigquery/v2/projects")
# -> %HTTPoison.Response{body: "{\n \"kind\": \"bigquery#projectList...
```

#### Helper functions
`OAuthEx.Client` module provides some helper functions for token retrieval and http accessing.
- The `retrieve_token` method retrieves the OAuth token and store it locally.
    - This method-call starts up local web server with specified `:receiver_port` to listen callback from OAuth 2.0 server.
- The `project` method calls Google's BigQuery API using the pre-acquired OAuth token.

```Elixir
defmodule OAuth2Ex.Sample.Google do
  @moduledoc """
  Sample setting for Google OAuth 2.0 API.

  API: https://developers.google.com/accounts/docs/OAuth2
  """

  defmodule Client do
    @moduledoc """
    Client configuration for specifying required parameters
    for accessing OAuth 2.0 server.
    """

    use OAuth2Ex.Client

    def config do
      OAuth2Ex.config(
        id:            System.get_env("GOOGLE_API_CLIENT_ID"),
        secret:        System.get_env("GOOGLE_API_CLIENT_SECRET"),
        authorize_url: "https://accounts.google.com/o/oauth2/auth",
        token_url:     "https://accounts.google.com/o/oauth2/token",
        scope:         "https://www.googleapis.com/auth/bigquery",
        callback_url:  "http://localhost:4000",
        token_store:   %OAuth2Ex.FileStorage{
                         file_path: System.user_home <> "/oauth2ex.google.token"}
      )
    end
  end

  @doc """
  Retrieve the OAuth token from the server, and store to the file
  in the specified token_store path.
  """
  def browse_and_retrieve do
    Client.browse_and_retrieve!(receiver_port: 4000)
  end

  @doc """
  Refresh the OAuth access_token from the refresh_token, as
  Google's access token has expiration time.
  """
  def refresh_token, do: Client.refresh_token

  @doc """
  List the projects by calling Google BigQuery API - project list.
  API: https://developers.google.com/bigquery/docs/reference/v2/#Projects
  """
  def projects do
    response = OAuth2Ex.HTTP.get(
                 Client.token, "https://www.googleapis.com/bigquery/v2/projects")
    response.body |> JSEX.decode!
  end
end
```

### Config parameters
`OAuth2Ex.config` method requires the following parameters.

Parameter        | Description
---------------- | -------------
id(*)            | Client ID to identify the user to access.
secret(*)        | Client secret to authorize the token retrieval.
authorize_url(*) | Authorization url to retrieve a code to start authentication.
token_url(*)     | Token url to retrieve token.
scope            | Scope to identify the allowed scope within the provider's API. Some providers does not have one.
callback_url     | Callback url for receiving code, which is redirected from authorize_url.
token_store      | Specify a module to handle saving and loading.
auth_header      | HTTP Access header for specifying OAuth token. It defaults to "Bearer", which sends `Authorization: Bearer xxxx` header.
response_type    | Response type when accessing authorization url. It defaults to "code".
(*) indicates mandatory parameter.
