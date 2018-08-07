# Setup Local docker based concourse

check out [concourse/concourse-docker](https://github.com/concourse/concourse-docker) for up-to-date instructions

I used the instructions to [get concourse-quickstart up and running with 2 containers](https://github.com/concourse/concourse-docker#docker-run) with the following commands:

- This starts a postgres database named 'atc' with a login `pg/pg` in a docker container named `concorse-db` on a docker network called `concourse-net`

  ```bash
  docker run \
    --detach \
    -h concourse-postgres \
    -p 5432:5432 \
    --net=concourse-net \
    --name=concourse-db \
    -e POSTGRES_USER=pg \
    -e POSTGRES_PASSWORD=pg \
    -e POSTGRES_DB=atc \
    postgres
  ```

- This starts the concourse processes using the postgres from the container we just created. take special note of:
  - the docker network `concourse-net`.  Both containers need to be on the same network so this container can find a host named `concourse-db`
  - the `--add-local-user` and `--main-team-local-user` flags.  This defines the credentials we will use to access the Web UI.  Ensure that if you change this username you change it in both places.  In This case we are logging in using `concourse/password`
  - the `-p` flag which is what port we are exposing on our host machine to access concourse i.e. [http://localhost:8080/](http://localhost:8080)
```bash
docker run \
  --detach --privileged \
  -h concourse \
  -p 8080:8080 \
  --net=concourse-net \
  --name concourse \
  concourse/concourse quickstart \
  --add-local-user=concourse:password \
  --main-team-local-user=concourse \
  --worker-garden-dns-server 8.8.8.8 \
  --postgres-user=pg \
  --postgres-password=pg \
  --postgres-host=concourse-db
```

## Setup the fly cli
1. Download the `fly` cli from the concourse ui
    - go to [http://localhost:8080]
    - Follow the prompts on the page to download the cli for your platform
1. Make the downloaded fly cli executable
    ```bash
    chmod 755 ${HOME}/Downloads/fly
    ```
1. Move to someplace on your `${PATH}`
    ```bash
    mv ${HOME}/Downloads/fly /usr/local/bin/
    ```
1. Login to your local concourse
    ```bash
    fly -t local login \
      -c http://127.0.0.1:8080 \
      --username=concourse \
      --password=password
    ```

# Create 'local' Git remote and ssh key to access
### (Optional, only if you don't want to put a bunch of noise on GitHub)
1. create local bare repo
    ```bash
    mkdir -p ${HOME}/localRepos/hello-concourse.git
    pushd !$
      git init --bare
    popd

    ```

1. Add new `local` remote to project
    ```bash
    pushd ${HOME}/workspace/hello-concourse
      git remote add local ${HOME}/localRepos/hello-concourse.git
      git checkout master
      git push local HEAD:master
      git branch -u local/master
    popd

    ```

1. Generate some ssh keys and grant access to your host machine
    ```bash
    mkdir -p ${HOME}/.ssh
    pushd !$
      ssh-keygen -t rsa -b 1024 -C "for push-to-master demo" -f id_rsa.push-to-master.demo -N ''
      cat id_rsa.push-to-master.demo.pub >> ${HOME}/.ssh/authorized_keys
      chmod 600 ${HOME}/.ssh/authorized_keys
    popd

    ```
1. Verify remote access to local machine
    ```bash
    ssh-add -D #clear out ssh-agent
    ssh-add ${HOME}/.ssh/id_rsa.push-to-master.demo
    ssh ${USER}@127.0.0.1 git --git-dir ${HOME}/localRepos/hello-concourse.git br -avv
    ```

    you should see somethings simplar to:
    ```bash
    * master 7bda0eb Initial Commit
    ```

# Create a `secrets.yml` file to hold pipeline credentials. (We won't want to commit to Git)
1. obtain your current IP address.  I use `ifconfig` and look for the `en0:` entry then look for the `inet` field but your machine may have different networking

1. Add a `git-repo` variable to the secrets.yml (replace `[ip-address]` with your machines IP address)
    ```bash
    echo "git-repo: ${USER}@[ip-address]:${HOME}/localRepos/hello-concourse.git" > ${HOME}/Desktop/hello-concourse.secrets.yml
    ```

1. Add a `git-private-key` variable to the secrets.yml
   1. capture the private key contents
      ```bash
      cat ${HOME}/.ssh/id_rsa.push-to-master.demo | pbcopy
      ```
      this puts the contents of the Private Key we created earlier into your `OSX paste buffer`.

   1. edit the secrets file we created earlier and ensure there is an entry that looks like:
      ```yml
      git-private-key: |
        -----BEGIN RSA PRIVATE KEY-----
        ...
        -----END RSA PRIVATE KEY-----
      ```
      Take special note of the whitespace around the private key contents.
      
      1. after the colon there is exactly one space then the pipe symbol then a new line e.g. ` |`
      1. the content of the private key are on the lines immediately following the key name
      1. each line of the contents of the private key are indented

   1. save the secrets file for later use.


# Other useful things for demo
- > command to apply commits onto master
  ```bash
  git cherry-pick demo/push-to-master~8
  ```

- > command to break the build (note will not create a `commit`)
  ```bash
  git apply patches/break.patch
  ```

- > command to fix the build (note will not create a `commit`)
  ```bash
  git apply patches/fix.patch
  ```

- > command to create/update `main` pipeline
  ```bash
  fly -t local set-pipeline \
    -p main \
    -c ci/pipeline.yml \
    -l ${HOME}/Desktop/hello-concourse.secrets.yml
  ```

- > command to create/update `push-to-master` pipeline called '`mobius`'
  ```bash
  fly -t local set-pipeline \
    -p mobius \
    -c ci/push-to-master.yml \
    -l ${HOME}/Desktop/hello-concourse.secrets.yml \
    -v git-branch=changes/mobius
  ```

- > command to force a check in the pipeline
  ```bash
  fly -t local check-resource -r main/master
  fly -t local check-resource -r mobius/changes/mobius
  ```
