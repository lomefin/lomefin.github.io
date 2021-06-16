---
layout: post
title: "ActiveRecord delegate methods"
date: 2021-04-23
categories: ruby
language: es
---

Especialmente en las vistas, ellas deberían ser lo más tontas posibles (seguir el principio de menor conocimiento).

Entonces, conocer la estructura de los modelos es una responsabilidad que no deberían tener.

Si por ejemplo tuvieran algo como credit.vehicle.identification.plate significa que debes conocer Credit Vehicle e Identification .

Ahí hay dos cosas que se pueden hacer.

Si va a haber una parte de la vista que solo hablará de vehicle, podría haber un partial. Ese partial solo conoce a vehicle.

Por ej:

`render partial: 'common/vehicle', locals: { vehicle: @credit.vehicle }`

También se pueden delegar métodos, dentro de esa vista tengo a vehicle y delego sus datos de identificación, así

    class Vehicle
      belongs_to :identification
      delegate :plate, to: identification
    end

y ahí tengo entonces una vista que trabaja solo con Vehicle y cuando quiero la patente hago `= vehicle.plate` o si lo quieren hacer más explícito

    class Vehicle
      belongs_to :identification
      delegate :plate, to: identification, prefix: true
    end

`= vehicle.identification_plate`

Más para leer: 
https://guides.rubyonrails.org/active_support_core_extensions.html#method-delegation