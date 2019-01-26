# MyApp

To start your Phoenix server:

  * Install dependencies with `mix deps.get`
  * Create and migrate your database with `mix ecto.create && mix ecto.migrate`
  * Start Phoenix endpoint with `mix phx.server`

Now you can visit [`localhost:4000`](http://localhost:4000) from your browser.

Ready to run in production? Please [check our deployment guides](Y).

**Cambiar el puerto**

en `config/dev.exs` agregar:
```elixir
config :my_app, MyAppWeb.Endpoint,
  http: [port: System.get_env("PORT") || 4000],
```
y ejecutar con :
    
    PORT=5000 mix phx.server
    # OR
    PORT=5000 iex -S mix phx.server

**Crear proyecto**

    mix phx.new my-app --app my_app --module MyApp --no-html --no-brunch
    mix ecto.create

agregar `:plug_cowboy, "~> 1.0"`

si corres `mix phx.server` no habra pagina de inicio

en `config/dev.ex` cambiar para poder manejar errores:
```elixir
    debug_errors:false
```
scaffoldear para crear login de usuarios(vi contextos en el libro pero no recuerdo donde):

    mix phx.gen.context Auth User users email:string:unique \
    is_active:boolean

abrir la migracion  `priv/repo/migrations/asd_create_users.exs` ,email not null y password_hash:
```elixir
defmodule MyApp.Repo.Migrations.CreateUsers do
  use Ecto.Migration

  def change do
    create table(:users) do
      add :email, :string, null: false
      add :password_hash, :string
      add :is_active, :boolean, default: false, null: false

      timestamps()
    end

    create unique_index(:users, [:email])
  end
end
```
    
y luego correr la migracion:


    mix ecto.migrate

https://lobotuerto.com/blog/building-a-json-api-in-elixir-with-phoenix/?fbclid=IwAR3FAO4SH5W-9nWHpXqAnQNKGXk0ZSQTCCoSVlj8y7pDjQV4jJnn8paUMUk

agregar libreria e instalar:
```elixir
    {:bcrypt_elixir, "~> 1.0"}
```
en `config/test.exs`:
```elixir
    config :bcrypt_elixir, :log_rounds, 4
```
Don’t add that configuration option to
config/dev.exs or config/prod.exs!
It’s only used during testing to speed up
the process by decreasing security settings in that environment.

en `lib/my_app/auth/user.ex`  agregar y pasar el password al changeset:
    
```elixir
schema "users" do
field :password, :string, virtual: true
field :password_hash, :string
end
def changeset(user, attrs) do
  user
  |> cast(attrs, [:email, :is_active, :password])
  |> validate_required([:email, :is_active, :password]
  |> unique_constraint(:email)
  |> put_password_hash()
end
defp put_password_hash(
       %Ecto.Changeset{valid?: true, changes: %{password: password}} = changeset
     ) do
  change(changeset, password_hash: Bcrypt.hash_pwd_salt(password))
end
defp put_password_hash(changeset) do
  changeset
end
```   

**Endpoints para los usuarios**

generarlo con:
  
    mix phx.gen.json Auth User users email:string password:string \
    is_active:boolean --no-context --no-schema

hay que reparar los tests por problemas entre user_path, agregar en `lib/my_app_web/router.ex` la siguiente linea:

```elixir
 resources "/users", UserController, except: [:new, :edit]
 ```

en `lib/my_app_web/views/user_view.ex`, en `def render` eliminaremos la pass para solo evaluar si esta ya logeado:
  
```elixir
 password: user.password, 
 ```
**Ejecutar**

    iex -S mix phx.server

crear usuarios:

    MyApp.Auth.create_user(%{email: "asd@asd.com", password: "qwerty"})

usando CURL:

    curl -H "Content-Type: application/json" -X POST \
    -d '{"user":{"email":"some@email.com","password":"some password"}}' \
    http://localhost:4000/api/users


**CORS**

para manejo de otras librerias y puertos, ej un server cn vue

instalar dependencia:
```elixir
{:corsica, "~> 1.0"}
```
y agregar el puerto en `lib/my_app_web/endpoint.ex`:
```elixir
 plug Corsica,
    origins: "http://localhost:8080",
    log: [rejected: :error, invalid: :warn, accepted: :debug]
```
**Verificacion**

agregar en `lib/my_app/auth/auth.ex` :
```elixir
def authenticate_user(email, password) do
    query = from(u in User, where: u.email == ^email)
    query |> Repo.one() |> verify_password(password)
  end

  defp verify_password(nil, _) do
    # Perform a dummy check to make user enumeration more difficult
    Bcrypt.no_user_verify()
    {:error, "Wrong email or password"}
  end

  defp verify_password(user, password) do
    if Bcrypt.verify_pass(password, user.password_hash) do
      {:ok, user}
    else
      {:error, "Wrong email or password"}
    end
  end
  ```
agregamos en el router `lib/my_app_web/router.ex` el endpoint para logearse:
```elixir
post "/users/sign_in", UserController, :sign_in
```

agregamos al controlador de auth `lib/my_app_web/controllers/user_controller.ex` :

```elixir
def sign_in(conn, %{"email" => email, "password" => password}) do
  case MyApp.Auth.authenticate_user(email, password) do
    {:ok, user} ->
      conn
      |> put_status(:ok)
      |> render(MyAppWeb.UserView, "sign_in.json", user: user)

    {:error, message} ->
      conn
      |> put_status(:unauthorized)
      |> render(MyAppWeb.ErrorView, "401.json", message: message)
  end
end
```

en `lib/my_app_web/views/user_view.ex`y definimos el `401.json` :
```elixir
def render("sign_in.json", %{user: user}) do
  %{
    data: %{
      user: %{
        id: user.id,
        email: user.email
      }
    }
  }
end
```
y en el `lib/my_app_web/error_view.ex` definimos el `401.json`:
```elixir
def render("401.json", %{message: message}) do
    %{errors: %{detail: message}}
  end
  ```
probamos el login desde `http://localhost:4000/api/users/sign_in`:

    curl -H "Content-Type: application/json" -X POST \
    -d '{"email":"asd@asd.com","password":"qwerty"}' \
    http://localhost:4000/api/users/sign_in -i


**Sesiones**
agregar al router:
```elixir
    plug :fetch_session
```
para almacenar la session modificamos el `lib/my_app_web/controllers/user_controller.ex`:
```elixir
 def sign_in(conn, %{"email" => email, "password" => password}) do
    case MyApp.Auth.authenticate_user(email, password) do
      {:ok, user} ->
        conn
        |> put_session(:current_user_id, user.id)
        |> put_status(:ok)
        |> render(MyAppWeb.UserView, "sign_in.json", user: user)

      {:error, message} ->
        conn
        |> delete_session(:current_user_id)
        |> put_status(:unauthorized)
        |> render(MyAppWeb.ErrorView, "401.json", message: message)
    end
  end
```

vamos a modificar el router `lib/my_app_web/router.ex` para proteger vistas por la session:
```elixir
 pipeline :api_auth do
    plug :ensure_authenticated
  end

  scope "/api", MyAppWeb do
    pipe_through :api
    post "/users/sign_in", UserController, :sign_in
  end

  scope "/api", MyAppWeb do
    pipe_through [:api, :api_auth]
    resources "/users", UserController, except: [:new, :edit]
  end

  # Plug function
  defp ensure_authenticated(conn, _opts) do
    current_user_id = get_session(conn, :current_user_id)

    if current_user_id do
      conn
    else
      conn
      |> put_status(:unauthorized)
      |> render(MyAppWeb.ErrorView, "401.json", message: "Unauthenticated user")
      |> halt()
    end
  end
```

para comprobar si funciona:

si no nos logueamos no podemos ver:
    
    curl -H "Content-Type: application/json" -X GET \
    http://localhost:4000/api/users \
    -c cookies.txt -b cookies.txt -i

nos logeamos y si repetimos el de arriba podremos verlo:

    curl -H "Content-Type: application/json" -X POST \
    -d '{"email":"asd@asd.com","password":"qwerty"}' \
    http://localhost:4000/api/users/sign_in \
    -c cookies.txt -b cookies.txt -i


To-Do:
+ Take into account a user’s is_active attribute when trying to log-in.
+ Implement an /api/users/sign_out endpoint on UserController.
+ Make it RESTy: Extract sign_in and sign_out’s functionality from UserController onto their own controller.
+ Maybe call it SessionController, where UserController.sign_in should be SessionController.create and UserController.sign_out should be SessionController.delete.
+ Implement DB support for sessions.
+ Implement an /api/me endpoint on a new MeController.
+ This can serve as a ping endpoint to verify the user is still logged in. Should return the current_user information.


