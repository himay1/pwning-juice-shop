# Hosting a CTF event

> In computer security, Capture the Flag (CTF) is a computer security
> competition. CTF contests are usually designed to serve as an
> educational exercise to give participants experience in securing a
> machine, as well as conducting and reacting to the sort of attacks
> found in the real world. Reverse-engineering, network sniffing,
> protocol analysis, system administration, programming, and
> cryptanalysis are all skills which have been required by prior CTF
> contests at DEF CON. There are two main styles of capture the flag
> competitions: attack/defense and jeopardy.
>
> In an attack/defense style competition, each team is given a machine
> (or a small network) to defend on an isolated network. Teams are
> scored on both their success in defending their assigned machine and
> on their success in attacking the other team's machines. Depending on
> the nature of the particular CTF game, teams may either be attempting
> to take an opponent's flag from their machine or teams may be
> attempting to plant their own flag on their opponent's machine. Two of
> the more prominent attack/defense CTF's are held every year at DEF
> CON, the largest hacker conference, and the NYU-CSAW (Cyber Security
> Awareness Week), the largest student cyber-security contest.
>
> Jeopardy-style competitions usually involve multiple categories of
> problems, each of which contains a variety of questions of different
> point values and difficulties. Teams attempt to earn the most points
> in the competition's time frame (for example 24 hours), but do not
> directly attack each other. Rather than a race, this style of game
> play encourages taking time to approach challenges and prioritizes
> quantity of correct submissions over the timing.[^1]

![OWASP Juice Shop CTF logo](../introduction/img/JuiceShopCTF_Logo.png)

OWASP Juice Shop can be run in a special configuration that allows to
use it in Capture-the-flag (CTF) events. This can add some extra
motivation and fun competition for the participants of a security
training or workshop.

## Running Juice Shop in CTF-mode

Juice Shop supports _Jeopardy-style CTFs_ by generating a unique _CTF
flag code_ for each solved challenge. These codes are not displayed by
default, but can be made visible by running the application with the
`config/ctf.yml` configuration:

```
set NODE_ENV=ctf     # on Windows
export NODE_ENV=ctf  # on Linux

npm start
```

On Linux you can also pass the `NODE_ENV` in directly in a single
command

```
NODE_ENV=ctf npm start
```

When running the application as a Docker container instead execute

```
docker run -d -p 3000:3000 -e "NODE_ENV=ctf"
```

### Overriding the `ctf.key`

Juice Shop uses the content of the provided `ctf.key` file as the secret
component of the generated CTF flag codes. If you want to make sure that
your flag codes are not the same for every hosted CTF event, you need to
override that secret key.

The simplest way to do so, is by providing an alternative secret key via
the `CTF_KEY` environment variable:

```
set CTF_KEY=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxx     # on Windows
export CTF_KEY=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxx  # on Linux
```

or when using Docker

```
docker run -d -p 3000:3000 -e "NODE_ENV=ctf" -e "CTF_KEY=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
```

## Choosing a CTF score server

:wrench:[**TODO**](https://github.com/bkimminich/pwning-juice-shop/issues/5)

## Setting up CTFd for Juice Shop

Juice Shop comes with
[the convenient `juice-shop-ctf-cli` tool](https://github.com/bkimminich/juice-shop-ctf)
to to simplify the hosting of CTFs using the open source
[CTFd](https://ctfd.io) framework. This can significantly speed up your
setup time for an event.

Apart from that, CTFd is a very well-written and stable piece of Open
Source Software, which is why OWASP Juice Shop recommends CTFd as its
preferred CTF score server!

### Generating CTFd challenges with `juice-shop-ctf-cli`

:wrench:[**TODO**](https://github.com/bkimminich/pwning-juice-shop/issues/5)

### Running CTFd

Apply the generated `insert-ctfd-challenges.sql` following the steps
describing your preferred CTFd run-mode below.

#### Default setup (including SQLite database)

1. Get CTFd with `git clone https://github.com/CTFd/CTFd.git`.
2. Perform steps 1 and 3 from
   [the CTFd installation instructions](https://github.com/CTFd/CTFd#install).
3. Use your favourite SQLite client to connect to the CTFd database and
   execute the `INSERT` statements you created.
4. Browse to your CTFd instance UI (by default <http://127.0.0.1:4000>)
   and create an admin user and CTF name

#### `docker-compose` setup (including MySQL container)

1. Setup
   [Docker host and Docker compose](https://docs.docker.com/compose/install/).
2. Follow steps 2-4 from
   [the CTFd Docker setup](https://github.com/isislab/CTFd/wiki/Deployment#docker)
   to download the source code, create containers and start them.
3. After running `docker-compose up` from previous step, you should be
   able to browse to your CTFd instance UI (`<<docker host IP>>:8000` by
   default) and create an admin user and CTF name.
4. Once you have done this, run `docker-compose down` or use `Ctrl-C` to
   shut down CTFd. Note: Unlike a usual Docker container, data will
   persist even afterwards.
5. Add the following section to the `docker-compose.yml` file and then
   run `docker-compose up` again:

   ```
   ports:
     - "3306:3306"
   ```

6. Use your favourite MySQL client to connect to the CTFd database
   (default credentials are root with no password) and execute the
   `INSERT` statements you created.
7. Browse back to your CTFd instance UI and check everything has worked
   correctly.
8. If everything has worked, do another `docker-compose down`, remove
   the ports section you added to `docker-compose.yml` and then do
   `docker-compose up` again and you are ready to go!

## Using other CTF frameworks

:wrench:[**TODO**](https://github.com/bkimminich/pwning-juice-shop/issues/5)

## Commercial use disclaimer

:warning: Bear in mind: With the increasing number of challenge
solutions (this book included) available on the Internet _it might
**not** be wise to host a professional CTF for prize money_ with OWASP
Juice Shop!

[^1]: https://en.wikipedia.org/wiki/Capture_the_flag#Computer_security