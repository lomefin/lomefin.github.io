---
layout: post
title:  "Microservicio de noticias en Rails 5. Parte 2"
date:   2016-06-17 13:40:35 -0400
categories: tutorials rails5
---

Esta es la continuación del tutorial, si te faltó leerla, puedes [ver la parte 1]({% post_url 2016-06-16-microservicio-de-noticias-en-rails-5-parte-1%}).

Ahora ya que podemos ver resultados en el sistema, haremos un administrador para que se puedan ingresar noticias en él.

Primero haremos el namespace de administración.

`routes.rb`
```ruby
Rails.application.routes.draw do
  namespace :admin do
    resources :posts, param: :uuid do
      member do
        put :publish
        put :unpublish
      end
    end
  end

  scope ':channel' do
    resources :posts, only: [:show, :index], param: :uuid
  end
end
```

Dos cosas importantes:
  - El namespace admin lo estoy colocando antes, pues aquí usamos un scope que sirve como variable, si lo colocamos después Rails considerará que admin es un canal.
  - Siempre estoy usando los uuid para identificar los elementos, pese a que internamente sigo trabajando con los ids. Esto es básicamente porque el soporte para conectar los modelos mediante UUID no está tan limpio aún.

Podría trabajar en el administrador con los ids en vez de los uuid, pero esto hace más fácil que si tienes un problema en la vista de lectura, puedas encontrar la noticia más rápidamente.

Ahora debemos crear el controlador `rails g controller admin/posts`

Pero antes de implementar el controlador, agregaremos otro, un BaseController, que nos permitirá colocarle reglas especiales a los controladores de Admin (por ejemplo, autenticación). Teniendo controladores base por namespace nos permitirá tener distintas reglas en cada uno de ellos, lo que puede ser muy cómodo.

`admin/base_controller.rb`

```ruby
class Admin::BaseController < ApplicationController
end
```

`admin/posts_controller.rb`

```ruby
class Admin::PostsController < Admin::BaseController
  before_action :set_post, only: [:show, :edit, :update, :destroy, :publish, :unpublish]
  def index
    @posts = Post.incremental
  end

  def show
  end

  def new
    @post = Post.new
  end

  def create
    @post = Post.create post_params
    show_post
  end

  def edit
  end

  def update
    @post = Post.update post_params
    show_post
  end

  def destroy
    @post.destroy
  end

  def publish
    @post.publish!
    show_post
  end

  def unpublish
    @post.unpublish!
    show_post
  end

  private

  def show_post
    redirect_to admin_post_path uuid: @post.uuid and return
  end

  def post_params
    params.require(:post).permit(:title, :message, :channel, :url)
  end

  def set_post
    @post = Post.find_by_uuid params[:uuid]
  end
end

```

Obviamente, con tanto escribir olvidé la estructura de Post, pero en vez de ir a revisar al schema pueda preguntar en la consola.

```bash
show-model Post

Post
  id: integer
  channel: string
  uuid: uuid
  options: json
  message: text
  title: text
  published_at: datetime
  meta: json
  aasm_state: string
  created_at: datetime
  updated_at: datetime
  url: string

```

También podemos fijarnos que el modelo de Post tiene unos metodos adicionales: __publish!__ y __unpublish!__ asi que debemos implementarlos y como estamos trabajando con UUID sería bueno asegurarnos que exista al crearse el objeto.

```ruby
class Post < ApplicationRecord
  before_validation :check_uuid
  validates :uuid, uniqueness: true, presence: true

  scope :of_channel, -> (channel) { where(channel: channel) }
  scope :published, -> { where.not(published_at: nil) }
  scope :incremental, -> { order(published_at: :desc) }

  def publish!
    update published_at: Time.zone.now
  end

  def unpublish!
    update published_at: nil
  end

  private

  def check_uuid
    return unless self.uuid.nil?
    uuid  = SecureRandom.uuid
    while self.class.where(uuid: uuid).count > 0
      uuid = SecureRandom.uuid
    end
    self.uuid = uuid
  end
end
```

Y ahora finalmente agregamos los templates.

`index.html.erb`

```erb
<h1>Posts</h1>
<ul class="nms-post-list">
  <%= render partial: "post", collection: @posts %>
</ul>
<ul class="nms-posts-actions">
  <li><%= link_to "New post", new_admin_post_path %></li>
</ul>
```

`_post.html.erb`

```erb
<li class="nms-post-list-item">
  <%= link_to admin_post_path(uuid: post.uuid) do %>
    <%= post.title %>
  <% end %>
</li>
```

`show.html.erb`

```erb
<article class="nms-post" data-uuid="<%= @post.uuid %>">
  <h1 class="nms-post-title"><%= @post.title %></h1>
  <p class="nms-post-header">
    <small class="nms-post-channel" data-value="<%= @post.channel %>">Channel: <%= @post.channel %></small>
    <% if @post.published_at %>
    <br />
    <small class="nms-post-publication-date" data-value="<%= @post.published_at %>">Published: <%= l @post.published_at %></small>
    <% end %>
  </p>
  <p class="nms-post-message"><%= @post.message %></p>
</article>
<ul class="nms-post-actions">
  <li><%= link_to 'Edit', edit_admin_post_path(uuid: @post.uuid) %></li>
  <% if @post.published_at.nil? %>
    <li><%= link_to 'Publish', publish_admin_post_path(uuid: @post.uuid), method: :put %></li>
  <% else %>
    <li><%= link_to 'Unpublish', unpublish_admin_post_path(uuid: @post.uuid), method: :put %></li>
  <% end %>
  <li><%= link_to 'All posts', admin_posts_path %></li>
</ul>
```

`_form.html.erb`

```erb
<%= form_for [:admin, post] do |p| %>
  <%= p.label :channel %>
  <%= p.text_field :channel %> <br />
  <%= p.label :title %>
  <%= p.text_field :title %> <br />
  <%= p.label :message %>
  <%= p.text_area :message %> <br />
  <%= p.label :url %>
  <%= p.text_field :url %> <br />
  <%= submit_tag  %>
<% end %>
```

`edit.html.erb`

```erb
<%= render partial: "form", locals: {post: @post} %>

<ul class="nms-edit-post-options">
  <li><%= link_to 'Back',  admin_post_path(uuid: @post.uuid) %></li>
  <li><%= link_to 'All posts', admin_posts_path %></li>
</ul>
```

`new.html.erb`

```erb
<%= render partial: "form", locals: {post: Post.new } %>

<ul class="nms-new-post-options">
  <li><%= link_to 'Back', admin_posts_path %></li>
</ul>
```

La siguiente parte del tutorial considera el aprovechar un nuevo feature de Rails 5: ActionCable, luego seguiremos con cosas más convencionales: agregarle seguridad al sistema y en la última etapa le agregaremos un poco de estilo para que no se vea tan simplón.




