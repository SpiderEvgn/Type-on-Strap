1. Build -> Execute shell

```
#!/bin/bash +x
set -e

cd $WORKSPACE
cp config/database.yml.example config/database.yml
echo -e "\033[34mStart building test image\033[0m"
docker-compose build app_test

# prepare database
COMMAND="rails db:drop db:create db:migrate"
echo -e "\033[34mRunning: $COMMAND\033[0m"
unbuffer docker-compose run app_test $COMMAND

COMMAND="rspec spec/models/allergy_spec.rb"
echo -e "\033[34mRunning: $COMMAND\033[0m"
unbuffer docker-compose run app_test $COMMAND

---- release ---

#!/bin/bash +x
set -e

cd $WORKSPACE
cp config/database.yml.example config/database.yml
echo -e "\033[34mStart building test image\033[0m"
docker-compose build app_test

# prepare database
COMMAND="rails db:drop db:create db:migrate"
echo -e "\033[34mRunning: $COMMAND\033[0m"
unbuffer docker-compose run app_test $COMMAND

COMMAND="rspec"
echo -e "\033[34mRunning: $COMMAND\033[0m"
unbuffer docker-compose run app_test $COMMAND

echo -e "\033[34mStart building prod image\033[0m"
cp ~/projects/templar/master.key config/
docker-compose build app_prod

echo -e "\033[34mStart pushing prod image to Docker Registry\033[0m"
origin_br=$GIT_BRANCH
release=${origin_br: 7}
echo $release
docker tag templar-prod_app_prod 10.250.2.12:5000/templar-app-prod:$release
docker push 10.250.2.12:5000/templar-app-prod:$release

echo -e "\033[34mStart deploying prod image to app server\033[0m"
ssh -A user01@10.250.2.11 -tt << remotessh
docker pull 10.250.2.12:5000/templar-app-prod:$release

# start prod database, just for the first time
# docker run -d -v templar_dbdata:/var/lib/postgresql/data --name postgres_templar postgres

# remove old prod image
docker stop templar-app-prod
docker rm templar-app-prod

# deploy by start the new prod image
docker run -d --env-file /var/www/templar/env.production -v /var/www/templar/storage:/app/storage -p 3000:3000 --link postgres_templar:db --name templar-app-prod 10.250.2.12:5000/templar-app-prod:$release rails s

# copy public assets to where nginx can serve
rm -rf /var/www/templar/public
docker cp templar-app-prod:/app/public /var/www/templar/public

exit
remotessh
```

2. Post build -> Post build task

```
echo -e "\033[34mStart docker cleaning job\033[0m"
docker-compose down
docker system prune --force
```

3. develop flow：

  1. master 是可正常运行的生产状态，不允许直接 commit 和 push，只接受来自 develop/hotfix 的 pr

  2. develop 是开发的核心分支，保持最新的开发版本进度，不允许直接 commit 和 push，所有日常开发都基于 develop 创建 feature 分支然后合并进 develop

  3. 当 develop 开发的到一定程度提交 pr 到 master，打上标签版本号

  4. 当 master 到一定程度后创建新的 release 分支，成为正式的发布版本，部署到生产环境

  5. 当 release 版本发现 bug 需要紧急修复，创建 hotfix 分支，再提 pr 合并到 master，然后再次运行从 master 到 release 的新版本生产发布流程，上一代 release 版本退役

  6. 最后根据 bug 实际情况决定是否需要再合并进 develop，还是通过别的方式在 feature 中用更好的方式修复


4. deploy

生产环境用什么样的结构？nginx 装在本地还是用 docker？

目前的架构，nginx 在本地，项目也在本地，通过 .:/app 的 volume 方式在 docker 起服务。如此就不能打包 productin 的 image，必须把文件留在 host。于是，部署的步骤就是到 app server 的 /var/www/templar 下载最新项目，然后 compile 编译前端，然后 docker restart 重启服务

第二种方式，nginx 和 app 都使用 docker image，全部 docker 化。如此，在 CI/CD 的流程中，就直接编译好前端，处理好细节打包成 production image。于是，部署的步骤就是到 app server 直接 docker pull templar-prod:release-version 然后停掉旧的启动新的。
  - 这种方式有几个问题：
    1. 因为每个 prod image 的 tag 都是 release 版本，所以如何获取旧的 app 版本然后停掉？image 名字必须包含版本号吗？留待考虑
    
    2. 单个的 prod image 部署，那如何编写 docker-compose.yml 文件？该怎么知道 app service 的最新（版本）名字？
      - 倒是可以不用 compose，直接 docker run $release 然后用 --link 去单独起 db、redis 等服务






