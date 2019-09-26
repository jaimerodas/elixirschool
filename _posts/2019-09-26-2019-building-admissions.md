## Building Elixir School's Admissions application

If you didn't know, Elixir School has it's own Slack where contributors can gather to discuss our organization's content and projects but most importantly, support one another in our Elixir journey. When we set out to create our own Slack we wanted to address a big concern with many public Slacks: the signal to noise ratio is bad, there's just too much spam. 

> Have you contributed to an Elixir School project but not joined us on Slack? Head over to https://admissions.elixirschool.com to get your invite today!

So how can we keep our Slack public but prevent spammers from joining and do so in a way that doesn't add work to our maintainers? Our solution: required at least once contribution of any side to any one of our projects. 

Achieving this required an application that used GitHub to verify a user's eligibility. This application would come to be known as: Admissions.

> Want to skip ahead and see the final product? The code can be found at https://github.com/elixirschool/admissions.

In this post we're going to explore how Admissions works and how we achieved our goals using Elixir and Phoenix. To start let's look at the expected flow and work from there:

```mermaid
graph LR
    A["Sign in with GitHub"] --> B{"Contributor?"}
    B-- Yes -->D["Send invite via Slack API"]
    B-- No -->E["Ineligibility message"]
    D-->F["Congratulations message"]
```

In addition to telling us how the application should function, this diagram breaks the flow up into convient development task. Working from this diagram let's explore the individual tasks we'll need in order to fulfill our high level requirements:

1. Allow a user to sign in using GitHub and capture their access token. We can leverage [Ueberauth](https://github.com/ueberauth/ueberauth) and its [GitHub strategy](https://github.com/ueberauth/ueberauth_github) to do the heavy lifting for us.
2. With the user's access token use the GitHub API to see if the user has contributed to an organization's project. To avoid having to spend time writing our own GitHub API client we're going to make use of [Tentacat](https://github.com/edgurgel/tentacat).
3. Using the result of the API search, process the user's result
   1. In the event a user **is** a contributor, use the Slack API to send an invite and congratulate them.
   2. If they **have not** contributed we need to notify them of their inelibility

### Login with GitHub

Starting from a new Phoenix project (`mix phx.new admisssions`)  let's look at how to support GitHub login. For that we'll need to open our `mix.exs` file and add our new dependency: `ueberauth_github` and run `mix deps.get`:

```elixir
  defp deps do
    [
      {:gettext, "~> 0.11"},
      {:phoenix, "~> 1.4.0"},
      {:phoenix_html, "~> 2.11"},
      {:plug_cowboy, "~> 2.0"},
      {:ueberauth_github, "~> 0.7.0"},

      {:phoenix_live_reload, "~> 1.2", only: :dev}
    ]
  end
```

> We won't need to include `ueberauth` itself, as a dependency  of `ueberauth_github` it is included for us.

> Helpful tip: Did you know you can use `mix hex.info <package name>` to get the latest version? Try it!

With our application empowered with our new dependency what's left to do? Plenty! To finish our integration with Ueberauth we need to complete some subtasks:

1. Create a `AuthController` that'll handle the callback phase of the OAuth request.

2. Update our `router.ex` file to include the new route and controller.

3. Add the necessary configuration for Ueberauth to our `config/config.exs` file.

4. Setting up your application on GitHub. Here you'll also need to retrieve your `CLIENT_ID` and `CLIENT_SECRET` .

   > GitHub setup and configuration goes beyond this article. If you aren't quite sure what to do, head over to GitHub's Developer article [Authorizing OAuth Apps](https://developer.github.com/apps/building-oauth-apps/authorizing-oauth-apps/) 

Let's get started!

####  Creating our new controller

Completing our first subtask requires we create a new controller for Ueberauth that will handle the OAuth callback from GitHub in the event of successful login. The only hard requirement for our controller is that we include the Ueberauth plug:

```elixir
defmodule AdmissionsWeb.AuthController do
	use AdmissionsWeb, :controller

  plug Ueberauth
end
```

With the plug now in place we can move on to defining a function to handle our request which our case we've elected to name `callback/2`. Within this function we need to retreive the user details Ueberauth has so convienently placed into the `Plug.Conn` assigns for us. The fields we're concerned with are the user's email, GitHub nickname, and access token:

```elixir
defmodule AdmissionsWeb.AuthController do
	use AdmissionsWeb, :controller

  plug Ueberauth

  def callback(%{assigns: %{ueberauth_auth: ueberauth_auth}} = conn, _params) do
    %{credentials: %{token: token}, info: %{email: email, nickname: nickname}} = ueberauth_auth
  end
end
```

There's no need to concern ourselves _in this instance_ with a match error because all successful logins will contain the aforementioned fields.

Now that we've got what we need, we need to forward the user on to the next step in the process: determining eligibility. To ensure we've got what we need in the next step, let's put our GitHub data into the session and then redirect the user to the eligibility check:

```elixir
defmodule AdmissionsWeb.AuthController do
	use AdmissionsWeb, :controller

  plug Ueberauth

  def callback(%{assigns: %{ueberauth_auth: ueberauth_auth}} = conn, _params) do
    %{credentials: %{token: token}, info: %{email: email, nickname: nickname}} = ueberauth_auth
    
    conn
    |> put_session(:github, %{email: email, nickname: nickname, token: token})
    |> redirect(to: Routes.registrar_path(conn, :eligibility))
  end
end
```

With that in place we're done with our controller and can move on to the next subtask, updating our `router.ex`.

#### Updating Phoenix's router

Updating our router for Ueberauth is a fairly easy and straight forward change, let's open up our `router.ex` file and the following scope block to the bottom:

```elixir
scope "/auth", AdmissionsWeb do
  pipe_through :browser

  get "/github", AuthController, :request
  get "/github/callback", AuthController, :callback
end
```

We've added 2 routes but only 1 request handler, `callback/2` in our controller so what gives? Remember you remember the `plug Ueberauth` code we added to our controller? Our good friend Ueberauth takes care of the request phase of the OAuth exchange saving us the hassle.

Our router is updated and we're almost done with our integration, let's move on to our configuration.

#### Ueberauth configuration

```elixir
config :ueberauth, Ueberauth,
  providers: [
    github: {Ueberauth.Strategy.Github, [default_scope: "user:email,user:profile", send_redirect_uri: false]}
  ]

config :ueberauth, Ueberauth.Strategy.Github.OAuth,
  client_id: System.get_env("GITHUB_CLIENT_ID"),
  client_secret: System.get_env("GITHUB_CLIENT_SECRET")
```



An optional but strongly encourage configuration is to update the `oauth2` serializer to use the newer JSON library [Jason]():

```elixir
config :oauth2,
	serializers: %{
		"application/json" => Jason
	}
```

To do this we'll need to add `jason` to our `mix.exs` as we did before with `ueberauth_github`.

### Verifying contributor status

### Processing the user's request