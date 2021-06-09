# SonarQube

## docker 加载镜像

```bash
docker image load -i sonarqube.tar
docker image load -i postgres.tar
```

## docker-compose 命令

路径切换到docker-compose.yml文件所在上下文

```bash
docker-compose up -d   # 创建并启动
docker-compose ps      # 查看启动状态
docker-compose logs -f # 查看日志
docker-compose stop    # 停止
docker-compose restart # 重启
```

docker-compose.yml

```yaml
version: "3"

services:
  sonarqube:
    image: docker.repo.swiftpass.xyz/devops/sonarqube:8.6.0-ee
    restart: always
    depends_on:
      - db
    environment:
      SONAR_JDBC_URL: jdbc:postgresql://db:5432/sonar
      SONAR_JDBC_USERNAME: sonar
      SONAR_JDBC_PASSWORD: sonar@2018
    volumes:
      - /home/admin/workspace/sonarqube2/data:/opt/sonarqube/data
      - /home/admin/workspace/sonarqube2/extensions:/opt/sonarqube/extensions
      - /home/admin/workspace/sonarqube2/logs:/opt/sonarqube/logs
      - /home/admin/workspace/sonarqube2/temp:/opt/sonarqube/temp
    ports:
      - "19000:9000"
  db:
    image: postgres:12
    restart: always
    environment:
      POSTGRES_USER: sonar
      POSTGRES_PASSWORD: sonar@2018
    volumes:
      - /home/admin/workspace/sonarqube2/pg:/var/lib/postgresql
      - /home/admin/workspace/sonarqube2/pg_data:/var/lib/postgresql/data

#volumes:
  # sonarqube_data:
  # sonarqube_extensions:
  # sonarqube_logs:
  # sonarqube_temp:
  # postgresql:
  # postgresql_data:
```

