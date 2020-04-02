+++
title = "Ciclo de vida do Phoenix LiveView"
date = "2020-04-01"
author = "Cairo Noleto"
cover = ""
tags = ["elixir", "phoenix", "phoenix liveview", "myelixirstatus"]
keywords = ["elixir", "phoenix", "phoenix liveview", "myelixirstatus"]
showFullContent = false
+++

Para quem não conhece, o [Phoenix LiveView](https://github.com/phoenixframework/phoenix_live_view) é uma biblioteca que funciona em cima do [Phoenix](https://www.phoenixframework.org/), framework web feito para [Elixir](https://elixir-lang.org/), e que trás todo o poder de se criar aplicações ricas em tempo real, tudo isso via server side e renderizando HTML.

A biblioteca usa todo o poder que Elixir trouxe de websockets e comunicação em tempo real para fazer com que sua página pareça uma aplicação sendo controlada pelo framework JavaScript da moda, mas que no final é só HTML + CSS e back-end.

Antes de começarmos, eu espero que você já tenha seguido o [guia de instalação](https://hexdocs.pm/phoenix_live_view/installation.html) e que já tenha se familiarizado um pouco com LiveView. Recomendo ler os links antes de avançar.

## Entendendo mais o ciclo de vida do Phoenix LiveView

Para começar, essa é uma típica LiveView, onde implementamos as funções `mount/3` e `render/1`:

```elixir
defmodule MyApp.RegisterView do
  use Phoenix.LiveView

  def mount(_params, session, socket) do
    socket =
      socket
      |> assign(:anonymous_id, session.anonymous_id)

    {:ok, socket}
  end

  def render(assigns) do
    MyApp.RegisterView.render("register.html", assigns)
  end
  # ...
end
```

O ciclo de vida começa na função `mount/3`. Quando você acessa a rota de qualquer LiveView, é essa função
que é executada. Ela recebe os parâmetros que vem do controller, a sessão atual e o LiveView socket.
A sessão são dados privados que é setado pela aplicação, geralmente no controller, quando chama o LiveView.

É na função `mount/3` que você inicializa a sua LiveView, setando todas as variáveis que você vai
precisar no seu template.

Após isso, a função `render/1` é chamada e o HTML é devolvido, como qualquer resposta HTML que é feita pelo
Phoenix.

Após a página HTML ser renderizada, o LiveView conecta com o socket, que mais uma vez chama o `mount/3` da
view, assim ficando conectado e "de olho" em qualquer evento feito pelo browser.

Quando acontece qualquer evento, ele manda para a LiveView usando a função `handle_event/3`, que vou detalhar
nos próximos parágrafos.

De uma forma simplista, é assim que funciona o ciclo de vida do Phoenix LiveView.

## Aconteceu um evento, o que acontece?

O Phoenix LiveView tem um código JavaScript que fica rodando no browser e envia todos os eventos através
do websocket para o back-end. Ele faz uma chamada pra função `handle_event/3`, passando três parêmetros
que é o nome do evento, os parâmetros que acompanham aquele evento e o socket.

```elixir
defmodule MyApp.RegisterView do
  # ...
  def handle_event("validate", params, socket) do
    socket =
      socket
      |> assign(:errors, validate_form(params))

    {:noreply, socket}
  end

  def handle_event("save", params, socket) do
    {:noreply, assign(socket, :show_hide_password, false)}
  end

  defp validate_form(%{"user_registration" => user_registration_params}) do
    if Enum.empty?(user_registration_params)
      {:noreply, assign(socket, :errors, ["Email can't be blank"])}
    else
      {:noreply, socket}
    end
  end
# ...
end
```

Nesse código eu declarei duas funções `handle_event/3`, que vai tratar o evento `validate` e o `save`.
São essas funções que você vai usar para interagir com a página HTML. Foi essa parte que mais estourou
minha cabeça 🤯

Demorou um pouco pra cair a ficha de que **tudo** é um _"template normal"_ do Phoenix. Isso significa que a
interação com a página se dá pelo velho e guerreiro `if/else` + combinações de `handle_event/3`.

Um exemplo bem legal é [esse aqui](https://github.com/tuacker/phoenix_live_view_form_steps/), que
[adaptei](https://github.com/caironoleto/phoenix-live-view-example)  para uma versão mais recente (dev!)
do Phoenix, onde já podemos criar uma nova aplicação Phoenix incluindo o Phoenix LiveView
(`mix phx.new path --live`).

Nesse exemplo, é implementado um formulário com 3 passos e ao final "salva" no banco de dados. O código
na view são vários `if/else`s e o controle do que é visto é feito na LiveView, usando a função `handle_event/3`
para cada passo desse formulário.

Aqui eu deixo o código do template para que você veja os vários `if/else`s (a organização disso entra em um
segundo post!) e também o código da LiveView, com todas as funções que manipulam esse template.

<details>
<summary>Aqui é o código da view leex</summary>

```elixir
<%= form_for @changeset, "#", [phx_change: :validate, phx_submit: :save], fn f -> %>
  <%= if @current_step == 1 do %>
    <div>
      <%= label f, :title %>
      <%= text_input f, :title, autofocus: true %>
      <%= error_tag f, :title %>
    </div>

    <div>
      <%= label f, :description %>
      <%= text_input f, :description %>
      <%= error_tag f, :description %>
    </div>

  <% else %>
    <%= hidden_input f, :title %>
    <%= hidden_input f, :description %>
  <% end %>


  <%= if @current_step == 2 do %>
    <div>
      <%= label f, :type %>

      <%= label do %>
        <%= radio_button f, :type, "thing_a" %>
        Thing A
      <% end %>

      <%= label do %>
        <%= radio_button f, :type, "thing_b" %>
        Thing B
      <% end %>

      <%= error_tag f, :type %>
   </div>
  <% else %>
    <%= hidden_input f, :type %>
  <% end %>

  <%= if @current_step == 3 do %>
    <div>
      <%= label f, :something_else %>
      <%= text_input f, :something_else, autofocus: true %>
      <%= error_tag f, :something_else %>
    </div>
  <% end %>


  <%= if @current_step > 1 do %><button phx-click="prev-step">Back</button><% end %>

  <%= if @current_step == 3 do %>
    <%= submit "Submit" %>
  <% else %>
    <button phx-click="next-step">Continue</button>
  <% end %>

<% end %>
```
</details>

<details>
<summary>Aqui está o código da LiveView, onde tem a lógica que avança os passos desse formulário</summary>

```elixir
defmodule PhoenixLiveViewExampleWeb.StepFormLive do
  alias PhoenixLiveViewExample.StepForm
  alias PhoenixLiveViewExampleWeb.StepFormView

  use PhoenixLiveViewExampleWeb, :live_view

  def mount(_params, _session, socket) do
    socket =
      socket
      |> assign(current_step: 1)
      |> assign(changeset: StepForm.changeset(%StepForm{}, %{}))

    {:ok, socket}
  end

  def render(assigns) do
    StepFormView.render("form.html", assigns)
  end

  def handle_event("prev-step", _params, socket) do
    new_step = max(socket.assigns.current_step - 1, 1)

    {:noreply, assign(socket, current_step: new_step)}
  end

  def handle_event("next-step", _params, socket) do
    current_step = socket.assigns.current_step
    changeset = socket.assigns.changeset

    step_invalid =
      case current_step do
        1 -> Enum.any?(Keyword.keys(changeset.errors), fn k -> k in [:title, :description] end)
        2 -> Enum.any?(Keyword.keys(changeset.errors), fn k -> k in [:type] end)
        _ -> true
      end

    new_step = if step_invalid, do: current_step, else: current_step + 1

    {:noreply, assign(socket, :current_step, new_step)}
  end

  def handle_event("validate", %{"step_form" => params}, socket) do
    changeset = StepForm.changeset(%StepForm{}, params) |> Map.put(:action, :insert)

    {:noreply, assign(socket, :changeset, changeset)}
  end

  def handle_event("save", %{"step_form" => params}, socket) do
    # Pretending to insert stuff if changeset is valid
    changeset = StepForm.changeset(%StepForm{}, params)

    case changeset.valid? do
      true ->
        {:noreply,
          socket
          |> put_flash(:info, "StepForm inserted => #{inspect(changeset.changes)}")
          |> redirect(to: "/")}

      false ->
        {:noreply, assign(socket, :changeset, %{changeset | action: :insert})}
    end
  end
end
```
</details>

Espero que você se divirta com o Phoenix LiveView como estou me divertindo muito! Até a próxima e XAU BRIGADO!
