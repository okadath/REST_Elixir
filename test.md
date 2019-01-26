
despues arreglamos los tests:

    mix test

agregamos en `test/my_app/auth/auth_test.exs:` el password:
  

```elixir
defmodule MyApp.AuthTest do
  # ...
  describe "users" do
    @valid_attrs %{email: "some email", is_active: true,password: "alguna pass"}
    @update_attrs %{email: "some updated email", is_active: false, password: "alguna pass actualizada"}
    @invalid_attrs %{email: nil, is_active: nil, password: nil}
    # ...
    test "list_users/0 returns all users" do
      # ...changed
      assert Auth.list_users() == [%User{user | password: nil}]
    end

    test "get_user!/1 returns the user with given id" do
      # ...changed
      assert Auth.get_user!(user.id) == %User{user | password: nil}
    end

    test "create_user/1 with valid data creates a user" do
      # ...added
      assert Bcrypt.verify_pass("some password", user.password_hash)
    end
    # ...
    test "update_user/2 with valid data updates the user" do
      # ...added
      assert Bcrypt.verify_pass("some updated password", user.password_hash)
    end

    test "update_user/2 with invalid data returns error changeset" do
      # ...changed and added
      assert %User{user | password: nil} == Auth.get_user!(user.id)
      assert Bcrypt.verify_pass("some password", user.password_hash)
    end
  end
end
```

y en los tests en `test/my_app_web/controllers/user_controller_test.exs` remover las pass en su assert en create y update 

para los errores en los tests, en `lib/my_app_web/controllers/fallback_controller.ex` agregar una nueva call/2:
```elixir
  def call(conn, {:error, %Ecto.Changeset{}}) do
    conn
    |> put_status(:unprocessable_entity)
    |> put_view(MyAppWeb.ErrorView)
    |> render(:"422")
  end
```
