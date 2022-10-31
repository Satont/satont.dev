---
title: "Setup of a large project on a local machine"
date: 2022-10-28T20:04:44+03:00
draft: false
---

You have a mono repository with different applications in different langs. Maybe you have some system dependencies for that project. Maybe you need other versions of some dependencies.

How do you introduce a person to development on this project? How to describe dozens of dependencies needed for a simple run? How do you cut time?

Maybe you are a freak like me and you are constantly changing one system to another and you're already lazy to repeat the same steps on cd.

Note: my project uses doppler (https://doppler.com) - environment variable manager, protoc (https://developers.google.com/protocol-buffers) - protobuf compiler, golang (https://go.dev), nodejs (https://nodejs.org), taskfile (https://taskfile.dev) - convenient (for me) replacement of `Makefile`, various third party utilities (watcher to restart the application, linter `gofumpt`, linter `golines`), which must be installed as system tools through `go install` and spelled in gopath.

There may be several solutions, I have tried a couple. I'll say right away that I'm no expert, and there may actually be much simpler solutions.

1. Docker.

    With Docker, you can describe a `Dockerfile` with the required language, system dependencies, and describe a `docker-compose.yaml` file, where you have a row of databases, other services. As an example, I had `postgres`, `nats`, `minio`, `redis`.

    Great? As if it wasn't.

    ### Problems:

    * Reload on change
    
    What if you want your application to restart when files change, or even worse - you have 10+ microservices in your project that also need to be restarted individually, not all at once?

    Solution: pull the project directory into our Docker image via the `docker-compose.yaml` directive `volumes`.

    When we apply changes to a file (assuming you have nodemon or its alternatives running) our application will restart inside the container, and we have shared files.

    * Nodejs dependencies.

    We lifted all the project files, but with it the `node_modules` also pulled up. What if we want to add a new dependency to the application/microservice? Stop our entire `docker-compose.yaml` described project, including databases, etc., go to the right directory, add the dependency, restart the Docker image build, wait for the databases, services, our application to come up.

    Conclusion: docker can be handy if you have quite a small application without a bunch of third party dependencies.

2. devbox (https://github.com/jetpack-io/devbox).

    Devbox itself needs only 1 dependency, basically: the Nix Package Manager (https://nixos.org). The devbox itself is a superstructure on top of it, to simplify the interaction and configuration interface.

    Devbox allows you to install any package from the nixos repository (https://search.nixos.org/packages) without cluttering your system. 

    An example configuration that will allow you to put the above mentioned dependencies I need for a project: 

    ```json
    {
      "packages": [
        "doppler",
        "go_1_19",
        "protobuf",
        "go-protobuf",
        "go-task",
        "nodejs-18_x"
      ]
    }
    ```

    Then when you enter the project, all you have to do is write `devbox shell` in your console, and `devbox` will read the required packages, download them into the local project folder `.devbox`, add the required cli to your `PATH`, then you can use them as if they were systematically installed.

    ## Problems

    * What about databases, redis, e.t.c.?

      Here we can continue down the path of describing services in `packages`. But then you have to write configs for postgres, redis, nats, e.t.c, store files of these services too locally in the project (I, for example, stored in folder `data/postgres`, `data/redis`)

      From this problem arises the second:

    * How do I start the databases when I enter the project directory, or when I enter the `devbox shell`?

      The `devbox` itself offers this solution:

      ```json
      {
        "shell": {
          "init_hook": [
            "redis-server -c ./configs/redis.conf &",
            "postgres -c ./configs/postgres.conf" // I don't remember the full command here, so this is just an abstract example.
          ]
        }
      }
      ```

      So if you log in to a `devbox shell` this hook will be executed and the services described in this hook will run as background jobs in linux. 

      This solution works fine but maybe you have already guessed that this solution also has its BUT:

    * What to do if you want to open several terminals in a project folder?

      It is possible to write a bash script that will detect launched jobs, but this is a pain that I don't want to live and put up with.

    * How do I exit the `devbox shell` to terminate these jobs?

      `devbox` suggests this solution: add `source pre_exit_script.sh` to the end of your `init_hook`, which would have something like

      ```bash
      finish()
      {
        kill redis
        kill postgres
        echo "Services stopped, exiting."
      }
      trap finish EXIT SIGHUP
      ```

      This will require you to write `exit` twice to get this script to run, or to press `CTRL + D` twice on your keyboard.

      Ouch, looks very messy and inconvenient, is there anything you can do about it? You can, and this is solution #3!

3. Docker + Devbox = 🥰

    * Describe our `docker-compose.yaml` services (redis, postgres, e.t.c) example (https://github.com/Satont/tsuwari/blob/main/docker-compose.dev.yml)
    * Describe our `devbox.json` config with the right tools, but without bases, etc, example (https://github.com/Satont/tsuwari/blob/main/devbox.json)
    * Manually run our services once with `docker compose -f docker-compose.yaml up -d
    * Enter the `devbox shell`!
    * That's it, we can start our applications however we want. 
    For this I have a taskfile.yml (https://github.com/Satont/tsuwari/blob/main/Taskfile.yml), which by `task dev` builds internal project libraries (with cache, properly, to make it fast), runs migrations if needed, and runs all my `11` microservices at once via `nodemon` and `gow` (the Gosh equivalent of nodemon).

    If necessary, you can even use an extension for `VSCode` which will do `devbox shell` itself when you open the terminal, or write an extension for `fish`/`zsh` if you are used to keeping the terminal open separately from the code editor, or are not working in `VSCode`.

    Stop postgres, redis, e.t.c if necessary via `docker compose -f docker-compose.yaml stop`. I personally never do this, I don't spare the RAM for them.


Conclusion: By combining different tools we can speed up the process of getting a project up in a few minutes for new developers (or for me who change the system oooooo often). All you need to do is: install `Docker`, install `Devbox`.

You will not need dozens of configs for your services, no need to clutter the system with junk utilities that you may need only for one project.

P.S I apologize for the clumsiness, and in some places, the bluntness of the text. I am just getting used to writing my posts.

P.P.S Share your solutions to the problem. I heard that some people do it in `kubernetes`, but it's some kind of weird perversion for me, since I'm not familiar with kbuer.
