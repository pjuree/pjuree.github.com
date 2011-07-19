---
layout: post
title: A story of a TeX formula service in Clojure deployed on Heroku
---

# {{page.title}}

TeX is a language and typesetting program particularly well suited for typesetting mathematical formulas, see
[the Wikipedia entry](http://en.wikipedia.org/wiki/TeX) for more information.
We have had a TeX formula service implemented in Clojure using ring lying around for a while, but we had never deployed it online since that required either building a war file or starting a jetty instance at a dedicated server.
Therefore, the project had been floating dead around on github with no chance of being released into the wild... until now!
Heroku recently opened their service for Clojure projects and we decided to try out the project on the platform with great success: within one hour the service was up and running "in the cloud".

The service is deployed at [vivid-beach](http://vivid-beach-604.herokuapp.com/).
The source for the TeX service can be found at [github](http://github.com/tgk/tex-service).
If you want to deploy the apllication (or you own) we recommend [the guide available at Herokus homepage](http://devcenter.heroku.com/articles/clojure).

The rest of this post will briefly cover how the service was implemented and deployed.

## The TeX service itself



### Generating an image from a TeX formula

Luckily, there's a java library for generating images based on a TeX formula called [JLaTeXMath](
http://forge.scilab.org/index.php/p/jlatexmath/).
To code snippet below generates and returns an image containing a formula.

{% highlight clj %}
(defn render-tex [tex]
  "Render TeX formula (a String) to BufferedImage"
  (let [formula (TeXFormula. tex)
	icon (.createTeXIcon formula TeXConstants/STYLE_DISPLAY 20)
	img (BufferedImage. (.getIconWidth icon) (.getIconHeight icon) BufferedImage/TYPE_INT_ARGB)
	g2 (.createGraphics img)
	label (JLabel.)]
    (.setForeground label (Color/BLACK))
    (.paintIcon icon label g2 0 0)
    img))
{% endhighlight %}

### Returning an image response from ring

Responses for ring should contain a status, headers and a body in a map.
ring can handle an input stream body, so we only need to set the proper headers to serve an image.
The following snippet will generate an appropriate response.

{% highlight clj %}
(defn img-response [img]
  "Renders a BufferedImage to a ring response"
  (let [os (ByteArrayOutputStream.)
	res (ImageIO/write img "png" os)
	data (.toByteArray os)
	is (ByteArrayInputStream. data)]
    {:status 200
     :headers {"Content-Type" "image/png"}
     :body is}))
{% endhighlight %}

### The trimmings

The only thing remaining is to tie the components together using moustache for setting up the routes and hiccup for generating a nice frontpage.
The details can be found at [the github repository](https://github.com/tgk/tex-service/blob/master/src/tex_service/server.clj)
The most important thing is to start the jetty server using the port from a system property as set by foreman. 

{% highlight clj %}
(defn -main []
  (let [port (Integer/parseInt (System/getenv "PORT"))]
    (run-jetty main-app {:port port})))
{% endhighlight %}

## Deploying it on Heroku

Deployment is straight forward: simply create the app on the Heroku Cedar stack

    heroku create --stack cedar

deploy the code

    git push heroku master

and scale the web process

    heroku ps:scale web=1

That's it, the app can be visited by issuing

    heroku open

## Conclusion

Deploying a ring application on Heroku is simple and straightforward.
If you need additional services such as database access Heroku has a lot of possibilites.
