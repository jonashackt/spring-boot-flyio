# spring-boot-flyio
Having a look at https://fly.io/

## Why?

In late August Heroku announced their cancellation of their free plan for Heroku Dynos, Heroku Postgres and Heroku Data for Redis: https://blog.heroku.com/next-chapter

Looking at my Heroku dashboard there's also a new warning now stating that I need to upgrade to a paid plan for my apps before November 28th.

![heroku-dashboard-cancelling-free-dynos-postgres-redis](screenshots/heroku-dashboard-cancelling-free-dynos-postgres-redis.png)

I always loved Heroku for it's simplicity and used it in a lot of my blog posts, where I used it to deploy my example projects on GitHub in a super comprehensive way. I even dedicated some posts solely to Heroku https://blog.codecentric.de/en/2019/08/spring-boot-heroku-docker-jdk11/ and some of my best voted stackoverflow answers feature Heroku as well (e.g. "Connecting to Heroku Postgres from Spring Boot" https://stackoverflow.com/a/49978310/4964553).

Finally I often introduced Heroku to my students in my lectures at University of Applied Sciences Erfurt or Bauhaus University Weimar, where all the students had a running app (in Heroku) at the end of the first lessons.


## Fly.io to the rescue - but only with Buildpack support!

So what alternatives do we have? On a occasional team Friday some weeks ago my colleague Daniel entered the discussion about Heroku alternatives with: why not use https://fly.io/ ?! Ok, I said - never heard of it. It has been announced in March 2020 https://news.ycombinator.com/item?id=22616857 as:

> fly.io is really a way to run Docker images on servers in different cities and a global router to connect users to the nearest available instance. We convert your Docker image into a root filesystem, boot tiny VMs using an Amazon project called Firecracker, and then proxy connections to it. As your app gets more traffic, we add VMs in the most popular locations.

But does it support Buildpacks? I asked him. Since Heroku is the inventor of Cloud Native Buildpacks and they belong to my standard toolbelt for around 2 years now https://blog.codecentric.de/en/2020/11/buildpacks-spring-boot/ I really don't want to miss them again. And yes, Fly.io seems to support Buildpacks https://fly.io/blog/deno-on-fly-using-buildpacks/. So why not start a small Dev Friday and have a look at fly.io in detail? My colleague Andreas started using a Go project, I opted for a Spring Boot based project and started fresh via https://start.spring.io/:

![spring-initializer](spring-initializer.png)


## HowTo

So why not start using fly.io by checking out their hands-on guide at https://fly.io/docs/hands-on/


### Install flyctl

https://fly.io/docs/hands-on/install-flyctl/

On a Mac install flyctl via brew:

```shell
brew install flyctl
```


### Signup or login to fly.io

https://fly.io/docs/hands-on/sign-up/ or https://fly.io/docs/hands-on/sign-in/

```shell
fly auth signup
```

![signup-to-flyio](screenshots/signup-to-flyio.png)



### Build example Spring Boot app with Paketo

As stated in https://fly.io/docs/hands-on/launch-app/

> Fly.io allows you to deploy any kind of app as long as it is packaged in a Docker image. That also means you can just deploy a Docker image and as it happens we have one ready to go in flyio/hellofly:latest.

That means you can use the proposed command `flyctl launch --image flyio/hellofly:latest` - but this would only launch a pre-build app based on the fly.io image flyio/hellofly https://hub.docker.com/r/flyio/hellofly

But as we want to use our own Spring Boot project at https://github.com/jonashackt/spring-boot-flyio we need to create a Docker image first. 

That's easy using Cloud Native Buildpack support in Spring Boot https://blog.codecentric.de/en/2020/11/buildpacks-spring-boot/. Via Maven there's the `spring-boot:build-image` goal. Since fly CLI `launch --image` command cannont deploy a Docker image that wasn't published to a dedicated registry before, we also need to publish our image. With Maven there would be the `-Dspring-boot.build-image.publish` parameter as stated in the docs https://docs.spring.io/spring-boot/docs/current/maven-plugin/reference/htmlsingle/#build-image.examples.publish - but we also would have to configure a `<publishRegistry>` tag inside our `pom.xml`.

But using pack CLI https://buildpacks.io/docs/tools/pack/ directly makes that a lot easier. With pack we can re-use our login to the Docker registry we want to publish to on the command line. And as the Docker Hub introduced a rate limiting I always love to use the GitHub Container Registry in my projects https://blog.codecentric.de/en/2021/03/github-container-registry/. Be sure to have done a login to GitHub Container Registry https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry#authenticating-to-the-container-registry before running the following pack CLI command to build our Spring Boot app into a Docker image: 

```shell
pack build ghcr.io/jonashackt/spring-boot-flyio:latest \
    --builder paketobuildpacks/builder:base \
    --path . \
    --env "BP_OCI_SOURCE=https://github.com/jonashackt/spring-boot-flyio" \
    --env "BP_JVM_VERSION=17" \
    --publish
```

There are two additional parameters here: The `BP_OCI_SOURCE` as --env parameter creates the GitHub Container Registry <-> Repository link (https://paketo.io/docs/buildpacks/configuration/#applying-custom-labels) and the `BP_JVM_VERSION` 17, because we use Java 17 inside our Maven build but Paketo defaults to 11.


Having a local Docker daemon running this should bundle our project into a Dockerfile like this:

```shell
...
Saving ghcr.io/jonashackt/spring-boot-flyio:latest...
*** Images (sha256:59f2bf1f186f6d837d211c544dfbc342817ebb94f9776e918973a0ebcc2a4163):
      ghcr.io/jonashackt/spring-boot-flyio:latest
Reusing cache layer 'paketo-buildpacks/bellsoft-liberica:jdk'
Reusing cache layer 'paketo-buildpacks/syft:syft'
Reusing cache layer 'paketo-buildpacks/maven:application'
Reusing cache layer 'paketo-buildpacks/maven:cache'
Reusing cache layer 'paketo-buildpacks/maven:maven'
Reusing cache layer 'cache.sbom'
Successfully built image ghcr.io/jonashackt/spring-boot-flyio:latest
```

Having a look into the package view of our repository we should be able to see the new image published:

![github-container-registry-package](screenshots/github-container-registry-package.png)

Before we can actually use this image with fly.io, we have to make it publicly accessible. This is just needed once - and its only because the default visibility for container images on the GitHub Container Registry is private.

So head over to the package settings of your GitHub Container Registry image and scroll down to the `Danger Zone` and click on `change visibility`:

![container-image-visibility](screenshots/container-image-visibility.png)

Now we should finally have our image publicly available and should be able to deploy our Spring Boot app on fly.io using `flyctl` like this:

```shell
fly launch --image ghcr.io/jonashackt/spring-boot-flyio:latest
```


But looking into our fly.io dashboard we may see an error...



### Alternative image building: Configure Buildpacks support for Spring Boot in fly.io

As an alternative to using Paketo and pack CLI yourself you can even hand that one over to fly CLI. But as opposed to what's stated in the docs https://fly.io/docs/reference/configuration/#the-build-section we do not need to add a `buildpacks` configuration to our `fly.toml`.

Instead we simply override the generated `image` configuration inside the `fly.toml`:

```toml
[build]
  image = "ghcr.io/jonashackt/microservice-api-spring-boot:latest"
```

and simply use the `builder` tag only like this (as stated in this so answer https://stackoverflow.com/a/73688179/4964553):

```toml
[build]
  builder = "paketobuildpacks/builder:base"
```

Now fly CLI will build your app using Cloud Native Buildpacks without you requiring to issue pack CLI commands. And it even publishes the image to the fly.io Docker registry at registry.fly.io/microservice-api-spring-boot and deploy your app correctly.




### More RAM please!

Our Spring Boot app wasn't deployed successfully sadly. Having a look into the `Monitoring` of our app at https://fly.io/apps/spring-boot-flyio/monitoring we should see the problem leading to a `error unable to calculate memory configuration` error:

```shell
...
 2022-09-12T09:55:32.686 runner[effc34fc] fra [info] Starting virtual machine
2022-09-12T09:55:32.854 app[effc34fc] fra [info] Starting init (commit: 249766e)...
2022-09-12T09:55:32.870 app[effc34fc] fra [info] Preparing to run: `/cnb/process/web` as 1000
2022-09-12T09:55:32.881 app[effc34fc] fra [info] 2022/09/12 09:55:32 listening on [fdaa:0:938e:a7b:a992:effc:34fc:2]:22 (DNS: [fdaa::3]:53)
2022-09-12T09:55:32.921 app[effc34fc] fra [info] Setting Active Processor Count to 1
2022-09-12T09:55:33.006 app[effc34fc] fra [info] Calculating JVM memory based on 194456K available memory
2022-09-12T09:55:33.006 app[effc34fc] fra [info] For more information on this calculation, see https://paketo.io/docs/reference/java-reference/#memory-calculator
2022-09-12T09:55:33.006 app[effc34fc] fra [info] unable to calculate memory configuration
2022-09-12T09:55:33.006 app[effc34fc] fra [info] fixed memory regions require 637219K which is greater than 194456K available for allocation: -XX:MaxDirectMemorySize=10M, -XX:MaxMetaspaceSize=125219K, -XX:ReservedCodeCacheSize=240M, -Xss1M * 250 threads
2022-09-12T09:55:33.006 app[effc34fc] fra [info] ERROR: failed to launch: exec.d: failed to execute exec.d file at path '/layers/paketo-buildpacks_bellsoft-liberica/helper/exec.d/memory-calculator': exit status 1
2022-09-12T09:55:33.875 app[effc34fc] fra [info] Starting clean up. 
```

Our JVM-based app doesn't seem to have enough memory.

But we can fix that and give our app more memory [as stated here](https://community.fly.io/t/out-of-memory-restarts/1629/3): 

```shell
fly scale memory 1024
```

This should get us a green running Spring Boot app in the fly.io dashboard:

![fly-io-spring-boot-app-running-the-first-time](screenshots/fly-io-spring-boot-app-running-the-first-time.png)


__WARNING:__ This will get our Spring Boot app running on fly.io - but will also kick us out of the free plan we wanted in the first place when switching over from Heroku! The pricing docs tell us https://fly.io/docs/about/pricing/ that you the following for free:

> Resources included for free:

    Up to 3 shared-cpu-1x 256mb VMs 
    3GB persistent volume storage (total)
    160GB outbound data transfer 

That means upgrading the memory to `1024` will cost us $0.0000022/s or $5.70 a month. But maybe we can reduce that memory consumption back to less than 256mbs later again?



### Access our Spring Boot app on fly.io

Our Spring Boot app is now running on fly.io without errors. We can also click on the generated hostname microservice-api-spring-boot.fly.dev - but that won't open up our app.

There are two reasons for that. First we need to tell fly.io on which port our Spring Boot app want's to be accessed. The port is defined inside the [application.properties](https://github.com/jonashackt/microservice-api-spring-boot/blob/main/src/main/resources/application.properties) via `server.port=8098`.

Now we need to head over to the generated `fly.toml` in the root directory of our project and change the app's port as stated in the docs https://fly.io/docs/reference/configuration/#the-services-sections

```toml
[[services]]
  http_checks = []
  internal_port = 8098
```

Right inside the `fly.toml` we also need to delete the `force_https = true` configuration inside the `[[services.ports]]` section. No fear this won't deactivate https in any way, but will enable us to access our Spring Boot app.

```toml
  [[services.ports]]
    # force_https = true
    handlers = ["http"]
    port = 80
```

Now let's refresh our apps configuration by running:

```shell
fly deploy
```

Finally our app should now be accessible to the public:

![fly-io-accessible-spring-boot-app](screenshots/fly-io-accessible-spring-boot-app.png)

Just access it at https://microservice-api-spring-boot.fly.dev/api/hello

![browser-app-access](screenshots/browser-app-access.png)











# Links

> The fly launch command detects your Dockerfile and builds it. If you have Docker running locally, it builds it on your machine. If not, it builds it on a Fly build machine. Once your container is built, it's deployed!

https://fly.io/docs/languages-and-frameworks/dockerfile/