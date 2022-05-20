---
layout: post
title:  "Pluralización de modelos en Rails."
date:   2016-11-14 17:36:35 -0400
categories: recommendations rails
---

TL;DR
-----

```ruby
# En vez de usar
Model.model_name.human.pluralize

# Es mejor utilizar
Model.model_name.human(count: :many)
```

La historia completa
---------------------

Rails y su internacionalización es muy útil y cómoda. Y nos invita a usar sus componentes la mayor cantidad de veces posible.

Sin embargo, podemos pecar algunas veces por omisión y usar los elementos incorrectos, en particular nos sucedió usando el inflector para obtener la versión plural de los nombres de los modelos que tenemos en nuestra aplicación.

Un ejemplo, tenemos un modelo de usuario llamado creativamente `User` y queremos obtener su versión en plural para mostrarlo en una vista de nuestros ERBs.

```erb
<h2><%= User.model_name.human.pluralize %></h2>
```

Esto nos aparecerá correctamente en inglés como **Users**.

Y si tenemos nuestro archivo de i18n correcto, como este:

```yaml
es:
  models:
    user: Usuario
```

El mensaje se leerá correctamente como **Usuarios**

Pero qué pasa cuando tengamos otro nombre que no obedezca a la misma regla que la inglesa para los plurales? Por ejemplo, si usamos la palabra *calor*, nos retornará **calors** en vez de **calores**.

Entonces, la solución que dimos fue parchar el archivo de inflexiones, inflections.rb y agregamos algunas excepciones:

```ruby
ActiveSupport::Inflector.inflections do |inflect|
  # inflect.plural /^(ox)$/i, '\1en'
  # inflect.singular /^(ox)en/i, '\1'
  inflect.irregular 'campus', 'campuses'
  inflect.irregular 'educador', 'educadores'
  # inflect.uncountable %w( fish sheep )
end
```

Pero, resulta que esa no es la manera correcta. Por qué? Porque el inflector es para un uso interno de Rails, i.e. para determinar el nombre de tablas y cosas que son necesarias para el funcionamiento por convención de Rails.

La manera correcta es usar el método `human` que vimos anteriormente y entregarle la cardinalidad al invocarlo, así:

```erb
<h3><%= User.model_name.human(count: :many) %></h3>
```

Esto funciona cuando tu archivo de I18n contiene las llaves apropiadas, por ejemplo:

```yaml
es:
  models:
    user:
      one: Usuario
      other: Usuarios
```
