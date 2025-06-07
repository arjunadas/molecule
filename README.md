### дайнный пайплайн скачаивает ansible роль и проверяет её с помощью ansible molecule

ниже описаны шаги по настройке процесса

#from https://sysadmintalks.ru/ansible-gitlab-ci/

#создаём образ с установленным ansible + molecule + docker, на базе этого образа будет создан gitlab-runner, на этом раннере будет проходить процесс проверки

nano Dockerfile
```
#Download base image ubuntu 22.04
FROM ubuntu:22.04

#Configure tz-data
ENV TZ=Europe/Moscow
RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone

# Update Ubuntu Software repository
RUN apt-get -qy update \
    && apt-get install -y --no-install-recommends \
    python3 python3-pip software-properties-common git libssl-dev openssh-client ca-certificates curl
	
# Add Docker's official GPG key:
RUN install -m 0755 -d /etc/apt/keyrings && curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc && chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
RUN echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | \
  tee /etc/apt/sources.list.d/docker.list > /dev/null && apt-get update

RUN	apt-get install -y --no-install-recommends docker-ce

RUN pip3 install ansible-dev-tools \
    && python3 -m pip install --user "molecule-plugins[docker]" \
    && rm -rf /var/lib/apt/lists/* \
    && rm -Rf /usr/share/doc && rm -Rf /usr/share/man \
    && apt-get clean
	
RUN ansible-galaxy collection install ansible.posix community.docker community.general	

CMD ["/bin/bash"]
```

```
docker build -t test-yakurnov-harbor.rnd.omnichannel.ru/library/ansible:2.17 .

docker login -u="admin" -p="***" https://test-yakurnov-harbor.rnd.omnichannel.ru

docker image pull test-yakurnov-harbor.rnd.omnichannel.ru/library/ansible:2.17
```

#для запуска гилаб раннера, в докере
#идём на сервер, где у нас крутится gitlab-runner
```
docker run -d --name gitlab-runner --restart always \
  -v /srv/gitlab-runner/config:/etc/gitlab-runner \
  -v /var/run/docker.sock:/var/run/docker.sock \
  gitlab/gitlab-runner:latest
```  

#для регистрации раннера
```
docker exec -it gitlab-runner gitlab-runner register
```

#при регистрации - важно правильно указать docker-image !
```
  --url "https://test-yakurnov-gitlab.rnd.omnichannel.ru/" \
  --token "GR1348941rQZbaj8rikqfP8zMs***" \
  --executor "docker" \
  --docker-image test-yakurnov-harbor.rnd.omnichannel.ru/library/ansible:2.17 \
  --tag-list "molecule_in_docker" \
```  

#https://sysadmintalks.ru/install-gitlab-runner/
#для работы docker-in-docker, надо внести правки в конфиг файл
#правим на сервере, где у нас крутится gitlab-runner

docker exec -it gitlab-runner bash

nano /etc/gitlab-runner/config.toml

#приводим к виду
```
volumes = ["/cache", "/var/run/docker.sock:/var/run/docker.sock"]

docker restart gitlab-runner
```
