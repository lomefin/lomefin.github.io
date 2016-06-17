Rails 5 está pronto a salir y quiero aprovechar de documentar el desarrollo de un microservicio de noticias con el cual queremos entregar algunas informaciones en nuestra plataforma Bambinotes.

La idea es que los roles de usuarios de Bambinotes estén suscritos a un canal de noticias con el que puedan revisar algunos detalles de ésta, y que posteriormente puedan navegar a un blog más profesional si es necesario.

El repositorio del proyecto está en GitHub: https://github.com/lomefin/nms

Asumiremos que estamos desde un computador sin Rails 5, pero con RVM y Postgres ya instalado.

Primero, instalar ruby 2.3.1 -el más reciente al momento de escribir este artículo.

[code lang="bash"]rvm use 2.3.1@nms --create[/code]

Luego instalar rails 5 (5.0.0.rc1)

```bash gem install rails --pre --no-ri --no-rdoc````

Luego que se instale todo eso, creo el nuevo proyecto llamado nms.

```bash rails new nms --database=postgresql```

Con el nuevo proyecto, y ya configurado para Postgres, debo crear las bases de datos.

```bash
createdb nms_development
createdb nms_test
```

Luego agregué Pry para tener una mejor consola y rails_12factor para integrarme con Heroku

```ruby
group :development do
  gem 'pry-rails'
end

group :production do
  gem 'rails_12factor'
end
```

Vamos a dejar todo listo para Heroku de una vez, eso significa tener un Procfile funcionando.

Procfile

[code lang="bash"]

web: bundle exec puma -t 5:5 -p ${PORT:-3000} -e ${RACK_ENV:-development}

[/code]

Ahora hacemos __bundle update__ y luego __heroku local__ y tendremos una aplicación lista para andar. En paralelo abro una nueva terminal para tener la consola andando a su vez con __rails c__.

Luego vamos a crear el objeto Post que tendrá las noticias, siempre prefiero usar la clase News, pero como es un término plural se enreda todo, aparte Post es un elemento más genérico en los ejemplos.

[code lang="bash"]
rails g model Post channel:string uuid:uuid options:json message:text title:text
published_at:datetime meta:json url:string
[/code]

Una de las novedades de Rails 5 es que el uso de rake disminuye en favor del uso de rails, por lo tanto para hacer esta migración hacemos

```bash rails db:migrate ```

Ahora a implementarle algunos scopes.

[code lang="ruby"]
class Post < ApplicationRecord
  scope :of_channel, -> (channel) { where(channel: channel) }
  scope :published, -> { where.not(published_at: nil) }
  scope :incremental, -> { order(published_at: :desc) }
end
[/code]

Listo nuestro modelo! Por ahora estaré usando el scope __:published__ que luego será reemplazado por [AASM](https://github.com/aasm/aasm). Ahora los controladores (en verdad, EL controlador)

```bash
rails g controller posts
```

El único controlador que tendremos hasta el momento será post_controller. __Nota:__ En el código habrá un controlador de administración, pero no hablaré de él aún hasta que tenga una implementación definida.

Nuestro controlador tendrá esta forma
[code lang="ruby"]

class PostsController < ApplicationController

  def index
    @posts = Post.of_channel(params[:channel]).published.incremental
  end

  def show
    @post = Post.of_channel(params[:channel]).published.find_by_uuid params[:uuid]
  end

end
[/code]

Luego configuraremos las rutas para que podamos llegar a estos objetos.

[code lang="ruby"]
Rails.application.routes.draw do
  scope ':channel' do
    resources :posts, only: [:show, :index], param: :uuid
  end
end
[/code]

Ya tenemos las rutas definidas, podemos revisarla en la consola de pry usando el comando __show-routes__.

[code]

         Prefix Verb   URI Pattern                       Controller#Action
          posts GET    /:channel/posts(.:format)         posts#index
           post GET    /:channel/posts/:uuid(.:format)   posts#show
    admin_posts GET    /admin/posts(.:format)            admin/posts#index
                POST   /admin/posts(.:format)            admin/posts#create
   new_admin_post GET    /admin/posts/new(.:format)        admin/posts#new
  edit_admin_post GET    /admin/posts/:uuid/edit(.:format) admin/posts#edit
     admin_post GET    /admin/posts/:uuid(.:format)      admin/posts#show
                PATCH  /admin/posts/:uuid(.:format)      admin/posts#update
                PUT    /admin/posts/:uuid(.:format)      admin/posts#update
                DELETE /admin/posts/:uuid(.:format)      admin/posts#destroy
[/code]

Finalmente, para tener lista la lectura de los posts por canal implementamos las vistas tanto en html como en json.

index.html.erb

[code lang="erb"]
<h1><%= params[:channel] %></h1>
<ul class="nms-post-list">
  <%= render partial: "post", collection: @posts %>
</ul>
[/code]

_post.html.erb

[code lang="erb"]
<li class="nms-post-list-item">
  <%= link_to post_path(uuid: post.uuid,channel: post.channel) do %>
    <%= post.title %>
  <% end %>
</li>
[/code]

show.html.erb

[code lang="erb"]
<article class="nms-post" data-uuid="<%= @post.uuid %>">
  <h1 class="nms-post-title"><%= @post.title %></h1>
  <p class="nms-post-header">
    <small class="nms-post-publication-date" data-value="<%= @post.published_at %>"><%= l @post.published_at %></small>
  </p>
  <p class="nms-post-message"><%= @post.message %></p>

</article>
[/code]

index.json.jbuilder

[code lang="ruby"]
json.array! @posts do |post|
  json.uuid post.uuid
  json.title post.title
  json.message post.message
end
[/code]

show.json.jbuilder

[code lang="ruby"]
json.uuid @post.uuid
json.title @post.title
json.message @post.message
json.published_at @post.published_at
json.channel @post.channel
json.meta (if @post.meta then @post.meta else {} end)
[/code]

Para hacer una prueba rápida crearemos un Post en la consola y veremos que entrega.

[code lang="ruby"]
Post.create channel: "BNS", uuid: SecureRandom.uuid, message: "Welcome to NMS, the new News Microservice", title: "Welcome to NMS", published_at: Time.zone.now
[/code]

Con esto ya podemos comenzar a ver el listado en ambos formatos y sus detalles.
Seguiremos con el sistema para agregar los posts en otro post.
