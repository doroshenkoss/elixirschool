---
version: 1.3.0
title: Basics
---

Ecto é um projeto oficial do Elixir que fornece uma camada de banco de dados e linguagem integrada para consultas. Com Ecto podemos criar *migrations*, definir schemas, inserir e atualizar registos, e fazer consultas.

{% include toc.html %}

## Instalação

Crie um novo app com uma supervision tree:

```shell
$ mix new example_app --sup
$ cd example_app
```

Para começar precisamos incluir Ecto e um adaptador de banco de dados no `mix.exs` do nosso projeto. Você pode encontrar uma lista de adaptadores de banco de dados suportados na seção [*Usage*](https://github.com/elixir-lang/ecto/blob/master/README.md#usage) do README do Ecto. Para o nosso exemplo iremos usar o PostgreSQL:

```elixir
defp deps do
  [{:ecto, "~> 2.2"}, {:postgrex, ">= 0.0.0"}]
end
```

Então nós vamos baixar nossas dependências usando

```shell
$ mix deps.get
```

### Repositório

Finalmente precisamos criar o repositório do nosso projeto, a camada de banco de dados. Isto pode ser feito rodando a tarefa `mix ecto.gen.repo -r ExampleApp.Repo`, falaremos sobre tarefas mix no Ecto mais para frente. O Repositório pode ser encontrado no arquivo `lib/<nome_do_projecto>/repo.ex`:

```elixir
defmodule ExampleApp.Repo do
  use Ecto.Repo, otp_app: :example_app
end
```

### Supervisor

Uma vez criado o nosso Repositório, precisamos configurar nossa árvore de supervisor, que normalmente é encontrada em `lib/<nome_do_projecto>.ex`. Adicione o Repo à lista `children`:

```elixir
defmodule ExampleApp.Application do
  use Application

  def start(_type, _args) do
    children = [
      ExampleApp.Repo
    ]

    opts = [strategy: :one_for_one, name: ExampleApp.Supervisor]
    Supervisor.start_link(children, opts)
  end
end
```

Para mais informações sobre supervisores, consulte a lição [Supervisores OTP](../../advanced/otp-supervisors).

### Configuração

Para configurar o Ecto precisamos adicionar uma seção no nosso `config/config.exs`. Aqui iremos especificar o repositório, o adaptador, o banco de dados e as informações de acesso ao banco de dados:

```elixir
config :example_app, ExampleApp.Repo,
  adapter: Ecto.Adapters.Postgres,
  database: "example_app",
  username: "postgres",
  password: "postgres",
  hostname: "localhost"
```

## Tarefas Mix

Ecto inclui uma série de tarefas mix úteis para trabalhar com o nosso banco de dados:

```shell
mix ecto.create         # Cria um banco de dados para o repositório
mix ecto.drop           # Elimina o banco de dados do repositório
mix ecto.gen.migration  # Gera uma nova *migration* para o repositório
mix ecto.gen.repo       # Gera um novo repositório
mix ecto.migrate        # Roda as migrations em cima do repositório
mix ecto.rollback       # Reverte migrations a partir de um repositório
```

## Migrations

A melhor forma de criar migrations é usando a tarefa `mix ecto.gen.migration <nome_da_migration>`. Se você está familiarizado com ActiveRecord, isto irá parecer familiar.

Vamos começar dando uma olhada numa migration para uma tabela *users*:

```elixir
defmodule ExampleApp.Repo.Migrations.CreateUser do
  use Ecto.Migration

  def change do
    create table(:users) do
      add(:username, :string, unique: true)
      add(:encrypted_password, :string, null: false)
      add(:email, :string)
      add(:confirmed, :boolean, default: false)

      timestamps
    end

    create(unique_index(:users, [:username], name: :unique_usernames))
  end
end
```

Por padrão Ecto cria uma chave primária `id` auto incrementada. Aqui estamos usando o callback padrão `change/0` mas Ecto também suporta `up/0` e `down/0` no caso de precisar um controle mais granular.

Como você deve ter adivinhado, adicionando `timestamps` na sua migration irá criar e gerir os campos `inserted_at` e `updated_at` por você.

Para aplicar as alterações definidas na nossa migration, rode `mix ecto.migrate`.

Para mais informações dê uma olhada a seção [Ecto.Migration](http://hexdocs.pm/ecto/Ecto.Migration.html#content) da documentação.

## Schemas

Agora que temos nossa migration podemos continuar para o schema. Schema é um módulo, que define mapeamentos para uma tabela do banco de dados e seus campos, funções auxiliares, e nossos *changesets*. Iremos falar mais sobre *changesets* nas próximas seções.

Por agora vamos dar uma olhada em como o schema para nossa migration se parece:

```elixir
defmodule ExampleApp.User do
  use Ecto.Schema
  import Ecto.Changeset

  schema "users" do
    field(:username, :string)
    field(:encrypted_password, :string)
    field(:email, :string)
    field(:confirmed, :boolean, default: false)
    field(:password, :string, virtual: true)
    field(:password_confirmation, :string, virtual: true)

    timestamps
  end

  @required_fields ~w(username encrypted_password email)
  @optional_fields ~w()

  def changeset(user, params \\ :empty) do
    user
    |> cast(params, @required_fields ++ @optional_fields)
    |> validate_required(@required_fields)
    |> unique_constraint(:username)
  end
end
```

O esquema que definimos representa de perto o que especificamos na nossa *migration*. Além dos campos para o nosso banco de dados, estamos também incluindo dois campos virtuais. Campos virtuais não são armazenados no banco de dados mas podem ser úteis em casos de validação. Veremos os campos virtuais em ação na seção [Changesets](#changesets).

## Consultas

Antes de poder consultar o nosso repositório, precisamos importar a *Query API*, mas por enquanto precisamos importar apenas `from/2`:

```elixir
import Ecto.Query, only: [from: 2]
```

A documentação oficial pode ser encontrada em [Ecto.Query](http://hexdocs.pm/ecto/Ecto.Query.html).

### O Básico

Ecto fornece uma excelente DSL<sup>(domain-specific language)</sup> de consulta que nos permite expressar consultas de forma muito clara. Para encontrar os usernames de todas as contas confirmadas poderíamos usar algo como este:

```elixir
alias ExampleApp.{Repo, User}

query =
  from(
    u in User,
    where: u.confirmed == true,
    select: u.username
  )

Repo.all(query)
```

Além do `all/2`, Repo fornece uma série de callbacks incluindo `one/2`, `get/3`, `insert/2`, e `delete/2`. Uma lista completa de callbacks pode ser encontrada em [Ecto.Repo#callbacks](http://hexdocs.pm/ecto/Ecto.Repo.html#callbacks).

### Count

Se nós queremos contar o números de usuários que tem uma conta confirmada, podemos usar `count/1`:

```elixir
query =
  from(
    u in User,
    where: u.confirmed == true,
    select: count(u.id)
  )
```

A função `count/2` também existe para contar os valores distintos de um dado entrada:

```elixir
query =
  from(
    u in User,
    where: u.confirmed == true,
    select: count(u.id, :distinct)
  )
```

### Group By

Para agrupar usernames por estado de confirmação podemos incluir a opção `group_by`:

```elixir
query =
  from(
    u in User,
    group_by: u.confirmed,
    select: [u.confirmed, count(u.id)]
  )

Repo.all(query)
```

### Order By

Ordenar usuários pela data de criação:

```elixir
query =
  from(
    u in User,
    order_by: u.inserted_at,
    select: [u.username, u.inserted_at]
  )

Repo.all(query)
```

Para ordenar por `DESC`:

```elixir
query =
  from(
    u in User,
    order_by: [desc: u.inserted_at],
    select: [u.username, u.inserted_at]
  )
```

### Joins

Assumindo que temos um perfil associado ao nosso usuário, vamos encontrar todos os perfis de contas confirmadas:

```elixir
query =
  from(
    p in Profile,
    join: u in assoc(p, :user),
    where: u.confirmed == true
  )
```

### Fragmentos

Às vezes a Query API não é suficiente, por exemplo, quando precisamos de funções específicas para banco de dados. A função `fragment/1` existe para esta finalidade:

```elixir
query =
  from(
    u in User,
    where: fragment("downcase(?)", u.username) == ^username,
    select: u
  )
```

Outros exemplos de consultas podem ser encontradas na descrição do módulo [Ecto.Query.API](http://hexdocs.pm/ecto/Ecto.Query.API.html).

## Changesets

Na seção anterior aprendemos como recuperar dados. Mas então como inserir e atualizá-los? Para isso precisamos de *Changesets*.

Changesets cuidam da filtragem, validação, manutenção das *constraints* quando alteramos um schema.

Para este exemplo iremos nos focar no *changeset* para criação de conta de usuário. Para começar precisamos atualizar o nosso schema:

```elixir
defmodule ExampleApp.User do
  use Ecto.Schema
  import Ecto.Changeset
  import Comeonin.Bcrypt, only: [hashpwsalt: 1]

  schema "users" do
    field(:username, :string)
    field(:encrypted_password, :string)
    field(:email, :string)
    field(:confirmed, :boolean, default: false)
    field(:password, :string, virtual: true)
    field(:password_confirmation, :string, virtual: true)

    timestamps
  end

  @required_fields ~w(username email password password_confirmation)
  @optional_fields ~w()

  def changeset(user, params \\ :empty) do
    user
    |> cast(params, @required_fields, @optional_fields)
    |> validate_length(:password, min: 8)
    |> validate_password_confirmation()
    |> unique_constraint(:username, name: :email)
    |> put_change(:encrypted_password, hashpwsalt(params[:password]))
  end

  defp validate_password_confirmation(changeset) do
    case get_change(changeset, :password_confirmation) do
      nil ->
        password_incorrect_error(changeset)

      confirmation ->
        password = get_field(changeset, :password)
        if confirmation == password, do: changeset, else: password_mismatch_error(changeset)
    end
  end

  defp password_mismatch_error(changeset) do
    add_error(changeset, :password_confirmation, "Passwords does not match")
  end

  defp password_incorrect_error(changeset) do
    add_error(changeset, :password, "is not valid")
  end
end
```

Melhoramos nossa função `changeset/2` e adicionamos três novas funções auxiliares: `validate_password_confirmation/1`, `password_mismatch_error/1` e `password_incorrect_error/1`.

Como o próprio nome sugere, `changeset/2` cria para nós um novo *changeset*. Nele usamos `cast/3` para converter nossos parâmetros para um *changeset* a partir de um conjunto de campos obrigatórios e opcionais. Então nós validamos a presença dos campos obrigatórios. A seguir validamos o tamanho da senha do *changeset*, a correspondência da confirmação da senha usando a nossa propria função, e a unicidade do nome de usuário. Por último, atualizamos nosso campo `password` no banco de dados. Para tal usamos `put_change/3` para atualizar um valor no *changeset*.

Usar `User.changeset/2` é relativamente simples:

```elixir
alias ExampleApp.{User,Repo}

pw = "passwords should be hard"
changeset = User.changeset(%User{}, %{username: "doomspork",
                    email: "sean@seancallan.com",
                    password: pw,
                    password_confirmation: pw})

case Repo.insert(changeset) do
  {:ok, record}       -> # Inserted with success
  {:error, changeset} -> # Something went wrong
end
```

É isso aí! Agora você está pronto para guardar alguns dados.
