---
title: "Silhouette+CAS"
date: 2018-02-08T08:56:48-05:00
draft: false
---

I've been using the excellent [Silhouette](https://www.silhouette.rocks/docs) authentication
library for my Play! project.  When I first started using it there was no support for 
[CAS](https://www.apereo.org/projects/cas).  I rolled my own support.  

Silhouette now supports
CAS much more directly, but none of the seed projects show it integrated in.  Part of the
difficulty of including it in a seed project is that there aren't any public social providers
such as Facebook or Google to demonstrate it.

So here's what I did to get it working with our institution's CAS server, starting with the
[Silhouette Seed Template](https://github.com/mohiva/play-silhouette-seed).

## Getting Going
1. Clone the seed project:  `git clone https://github.com/mohiva/play-silhouette-seed.git play-siloutte-cas-seed`.  
0. Note the instructions in the seed project's tutorial to update the code to run locally:
	* In the `application.conf` file, set `play.mailer.mock = true` (this was already true for me).
	* In the `SignUpController` at about line 95 set `activated = true`.  If you're using CAS, the `User` class
	will likely be modified to eliminate this field.
0. Make a quick and dirty CAS icon by duplicating and editing one of the existing icons in `public/images/providers/`.  The project will display this icon.  Clicking it will redirect to CAS.  Name your new icon `cas.png`.

## The Build File
Add the library dependency for play-silhouette-cas to `build.sbt`:  `"com.mohiva" %% "play-silhouette-cas" % "5.0.0",`.  Change the name of the project and any other personal preferences.

At one point in the protocol CAS demands that the connection to your server be over https.  So add the following:

```
javaOptions ++= Seq(
  "-Dhttp.port=9000",
  "-Dhttps.port=9443"
  )
```


## Logging
I found enabling the logging to be a very helpful step in debugging the interactions with our CAS server.
I copied the default [`logback.xml`](https://www.playframework.com/documentation/2.6.x/SettingsLogger) and 
added `<logger name="com.mohiva.play.silhouette" level="DEBUG" />`.

## Config Parameters
Add something similar to this to `silhouette.conf`:

```
  # CAS provideer
  cas.casURL="https://cas.uwaterloo.ca/cas"
  cas.redirectURL="https://localhost:9443/authenticate/cas"
  cas.protocol="CAS10"
```

The default protocol is CAS30.  Our institution is apparently using CAS10; it took some trial and error and
looking at logs to figure that out.  Also pay attention to the `casURL`.  Yours could very well be different.


## Tackling SilhouetteModule
`SilhouetteModule` is where all the heavy lifting takes place to integrate CAS into the seed project.

Add `import com.mohiva.play.silhouette.impl.providers.CasProvider`.

In `provideSocialProviderRegistry` add `casProvider: CasProvider` as a parameter and pass it to the 
`SocialProviderRegistry` constructor.

Add a `CasProvider` provider for Guice:

```scala
  /**
   * Provides the CAS provider.
   *
   */
  @Provides
  def provideCasProvider(
    httpLayer: HTTPLayer,
    configuration: Configuration
  ): CasProvider = {
    import net.ceedubs.ficus.readers.EnumerationReader._

    val settings = configuration.underlying.as[CasSettings]("silhouette.cas")
    val client = new CasClient(settings)
    new CasProvider(httpLayer, settings, client)
  }
```

Add a provision for a CAS `AuthInfoRepository` by adjusting the `provideAuthInfoRepository` definition:
```scala
  def provideAuthInfoRepository(
    passwordInfoDAO: DelegableAuthInfoDAO[PasswordInfo],
    oauth1InfoDAO: DelegableAuthInfoDAO[OAuth1Info],
    oauth2InfoDAO: DelegableAuthInfoDAO[OAuth2Info],
    openIDInfoDAO: DelegableAuthInfoDAO[OpenIDInfo],
    casInfoDAO: DelegableAuthInfoDAO[CasInfo]): AuthInfoRepository = {

    new DelegableAuthInfoRepository(passwordInfoDAO, oauth1InfoDAO, oauth2InfoDAO, openIDInfoDAO, casInfoDAO)
  }
```

Add a binding in the `configure` method:
```scala
    bind[DelegableAuthInfoDAO[CasInfo]].toInstance(new InMemoryAuthInfoDAO[CasInfo])
```

At this point `sbt ~run` should successfully compile and run the program.  Pointing a browser at
[localhost:9000](localhost:9000) should display the signin page that includes your CAS icon.  Clicking
that icon should redirect you to the CAS server.  Log in and get redirected back to the authenticate 
controller.  It finishes the authentication procedure and then redirects you to the site's home page.

Our CAS server only returns the userid, which isn't displayed on the home page.  Add the following to 
`home.scala.html` to display the userid.

```
<div class="row">
    <p class="col-md-6"><strong>userid:</strong></p><p class="col-md-6">@user.loginInfo.providerKey</p>
</div>
```
