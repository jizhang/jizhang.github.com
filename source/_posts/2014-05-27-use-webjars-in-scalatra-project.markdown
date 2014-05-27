---
layout: post
title: "Use WebJars in Scalatra Project"
date: 2014-05-27 17:44
comments: true
categories: [Notes]
published: false
---

As I'm working with my first [Scalatra](http://www.scalatra.org/) project, I automatically think of using [WebJars](http://www.webjars.org/) to manage Javascript library dependencies, since it's more convenient and seems like a good practice. Though there's no [official support](http://www.webjars.org/documentation) for Scalatra framework, the installation process is not very complex. But this doesn't mean I didn't spend much time on this. I'm still a newbie to Scala, and there's only a few materials on this subject.

## Add WebJars Dependency in SBT Build File

Scalatra uses `.scala` configuration file instead of `.sbt`, so let's add dependency into `project/build.scala`. Take [Dojo](http://dojotoolkit.org/) for example.

```scala
object DwExplorerBuild extends Build {
  ...
  lazy val project = Project (
    ...
    settings = Defaults.defaultSettings ++ ScalatraPlugin.scalatraWithJRebel ++ scalateSettings ++ Seq(
      ...
      libraryDependencies ++= Seq(
        ...
        "org.webjars" % "dojo" % "1.9.3"
      ),
      ...
    )
  )
}
```

To view this dependency in Eclipse, you need to run `sbt eclipse` again. In the *Referenced Libraries* section, you can see a `dojo-1.9.3.jar`, and the library lies in `META-INF/resources/webjars/`.

## Add a Route for WebJars Resources

Find the `ProjectNameStack.scala` file and add the following line at the bottom of the trait:

```scala
trait ProjectNameStack extends ScalatraServlet with ScalateSupport {
  ...
  get("/webjars/*") {
    val resourcePath = "/META-INF/resources/webjars/" + params("splat")
    Option(getClass.getResourceAsStream(resourcePath)) match {
      case Some(inputStream) => {
        contentType = servletContext.getMimeType(resourcePath)
        IOUtil.loadBytes(inputStream)
      }
      case None => resourceNotFound()
    }
  }
}
```

**That's it!** Now you can refer to the WebJars resources in views, like this:

```ssp
#set (title)
Hello, Dojo!
#end

<div id="greeting"></div>

<script type="text/javascript" src="${uri("/webjars/dojo/1.9.3/dojo/dojo.js")}" data-dojo-config="async: true"></script>
<script type="text/javascript">
require([
    'dojo/dom',
    'dojo/dom-construct'
], function (dom, domConstruct) {
    var greetingNode = dom.byId('greeting');
    domConstruct.place('<i>Dojo!</i>', greetingNode);
});
</script>
```
