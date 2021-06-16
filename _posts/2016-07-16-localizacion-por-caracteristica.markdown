---
layout: post
title:  "Localización por características."
date:   2016-07-16 22:36:35 -0400
categories: recommendations rails
---

El desarrollo de una aplicación siempre debe considerar ya la internacionalización y la localización, el no considerarlo nos limita inmediatamente sobre qué tan lejos queremos llegar.

Es por esto que Rails tiene considerado como un elemento principal la internacionalización, que lo acortan como i18n. Con este método en vez de escribir directamente el texto que quieres agregar, vas colocando etiquetas, que serán reemplazados por el texto que corresponda según el idioma en que se trabaje.

Por ejemplo, en vez de colocar un título de la siguiente manera:

	<h1>Noticias más recientes</h1>

Se puede internacionalizar así:

	<h1><%= t(".most_recent_stories") %></h1>

Esto nos colocará en un comienzo un texto que dirá "Most Recent Stories" y que si pasamos el mouse sobre él nos indicará que falta una etiqueta en el idioma.

## Pero entonces, cómo coloco el idioma?

En la carpeta config/locales/ estarán archivos yml en donde puedes crear las traducciones. Podemos considerar que hay dos archivos disponibles: en.yml y es.yml y que el título anterior esta en la página que se encuentra en views/welcome/index.html.erb

en.yml

	en:
		welcome:
			index:
				most_recent_stories: Most recent stories

es.yml

	es:
		welcome:
			index:
				most_recent_stories: Noticias más recientes

En verdad es una maravilla!

A medida que tu aplicación crece este archivo crecerá más y más. También, a medida que avanzas, podrás encontrarte con otros idiomas, o como en el caso de @familink, con distintas variedades de un idioma: El vocabulario que se usa en Chile dista mucho en algunos aspectos al de México o Argentina. Hay veces en que te gustaría aplicar un mismo vocabulario para todos, al fin y al cabo todos hablamos español supuestamente. Pero la experiencia dice que si quieres dar una buena experiencia de usuario, el usuario debe sentirse lo más cómodo con la aplicación y usar la terminología más cercana a él nos brinda una ventaja.

Entonces, empezamos a tener archivos como en-US.yml en-GB.yml es-CL.yml es-ES.yml es-AR.yml es-MX.yml y cada uno con miles de llaves y términos.

Entonces llegamos a un tema que hay que analizar: Tenemos dos idiomas con algunas variantes, cada vez que debemos hacer una nueva característica a nuestro software debemos agregar las llaves múltiples veces: dos sets de llaves en inglés y cuatro sets de llaves en español. Puede que haya un término distinto en cada idioma, pero seguramente la regla general es que la mayoría de los términos se conserva de una variante idiomática a otra.

Estamos ahora en un punto donde hacemos muchos copiados y pegados y donde hacer las mezclas de código en git se hace un poco más compleja pues varias personas pueden estar cambiando el archivo de internalización en distintos puntos y cada vez que queremos hacer un cambio que afecte a todas las variantes de un idioma debemos hacer un cambio por cada archivo del idioma. En resumen: un pequeño desastre. Este desastre se nos complejiza aún más por el hecho que hay componentes de una aplicación que reutilizamos en otra, por lo que hemos empezado a aislar los commits de los archivos de internacionalización solamente para "aislar la pelea de la mezcla".

Un ejemplo corto, para una nueva característica

en-US.yml

	en:
		plays_feed:
			add_to_calendar: Add to my calendar
			find_venue: Find closest Theater

en-GB.yml

	en:
		plays_feed:
			add_to_calendar: Add to my calendar
			find_venue: Find closes Theatre



Si ya estás permanentemente peleando con tu propio código, hay algo que se debe hacer.

Fue entonces que por primera vez me fijé en devise.es.yml

Devise en un sistema de control de acceso y el genera sus propias vistas, que vienen con etiquetas de i18n por supuesto, ellos tienen disponible una enorme cantidad de archivos de i18n para muchos idiomas y variantes, incluso usando distintas voces, dependiendo de que tan coloquial o formal quieres tu aplicación. Cómo ellos entregan entonces sus traducciones sin provocar una pesadilla en la mezcla? Con prefijos

## Prefijos por característica

devise.es.yml Tiene la siguiente estructura:

	es:
		devise:
			sessions:
				new:
					login: Ingresar

Exactamente igual que el archivo de i18n que vimos, pues YML combina todas las llaves y luego las deja disponibles.
Entonces, por qué no consideramos las características cada una como un nuevo prefijo para el idioma, reciclamos la gran mayoría de los textos y cambiamos la diferencias? Aquí podemos explotar la capacidad de hacer referencias en YAML.

Volviendo al ejemplo de los teatros

plays_feed.en.yml

	en-US:
		plays_feed: &plays_feed
			add_to_calendar: Add to calendar
			find_venue: Find closest theater

	en-GB:
		play_feed:
			<<: *plays_feed                             # Agrego los campos definidos anteriormente
			find_venue: Find closes theatre   # Sobreescribo solo los campos que difieren.

Esta metodología que estamos ahora ocupando nos ha sido de mucha utilidad, pues generalmente la gran cantidad de cambios vienen definidos por característica y en sus primeras etapas, con el tiempo el texto se consolida y no cambia mucho por lo que no tiene mucho sentido tener un archivo monolítico con todas las llaves que deba ir recibiendo cambios en secciones muy específicas.

El gran contra del método es que en caso de requerir hacer una modificación a ciertas traducciones no se tenga claro a qué componente pertenece por lo que se puede gastar algo de tiempo. Por esto recomiendo que las características más comunes si vivan en el archivo "principal" del idioma y solo los componentes adicionales tengan archivos prefijados.
