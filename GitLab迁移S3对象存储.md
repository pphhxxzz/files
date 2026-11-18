# GitLab 19.x 迁移指南：换机器 + 对接 S3 对象存储

Docker 部署的 GitLab 19.5 从一台机器迁到另一台，文件存储从本地 MinIO 换成 S3 兼容对象存储（OBS）。数据量：几十个项目、仓库几百兆、对象存储 1G 多，停机 25 分钟。

## 数据分布：本地数据与对象存储数据

对象存储只接管文件，Git 仓库不包含在内。GitLab 的数据分两部分：

- 本地盘：Git 仓库（Gitaly 管的）、PostgreSQL、Redis
- 对象存储：CI artifacts、LFS、上传附件、Package

Git 仓库不能存放在 S3 上，因为 Git 依赖原子 rename 和文件锁，对象存储不具备这些能力。本地需要迁移的只有三项：

```
git-data/repositories
postgresql
gitlab.rb + gitlab-secrets.json
```

对象存储中的数据单独迁移，不占用停机时间。

## 配对象存储

19.x 只认合并写法，`artifacts_object_store_enabled` 已移除：

```ruby
gitlab_rails['object_store']['enabled'] = true
gitlab_rails['object_store']['connection'] = {
  'provider'              => 'AWS',
  'region'                => 'us-east-1',
  'aws_access_key_id'     => 'xxx',
  'aws_secret_access_key' => 'xxx',
  'endpoint'              => 'http://oss.example.com',
  'path_style'            => true,
  'aws_signature_version' => 4
}

gitlab_rails['object_store']['objects']['artifacts']['bucket'] = 'dataflow-gitlab-artifacts'
gitlab_rails['object_store']['objects']['lfs']['bucket']       = 'dataflow-gitlab-lfs'
gitlab_rails['object_store']['objects']['uploads']['bucket']   = 'dataflow-gitlab-uploads'
gitlab_rails['object_store']['objects']['packages']['bucket']  = 'dataflow-gitlab-packages'
```

非 AWS 的 S3 兼容存储要开 `path_style`，`endpoint` 带 `http://`，签名版本用 4。region 填服务端认可的值，多数情况 `us-east-1`。

### 桶名核对

先用 `curl` 请求端点确认连通，不带签名返回 403，说明网络和网关正常。桶列表在存储控制台查看，实际有 `datafilow-gitlab` 和 `dataflow-knowledge-base` 两个。

给定的桶名是 `dataflow-gitlab`，控制台里是 `datafilow-gitlab`，中间多一个 `i`。按给定名称配置，`reconfigure` 后所有上传都会 `NoSuchBucket`。

存储端是华为 OBS 网关（响应头有 `server: OBS`、`x-obs-version`），配置写法和 MinIO 一致，区别在权限模型。

### 建桶权限

按一类对象一个桶拆分时，`create_bucket` 返回 403。这个账号只有指定桶的读写权限，不能建新桶，需要存储管理员开通。实际开通的 5 个桶：

```
dataflow-gitlab-artifacts
dataflow-gitlab-lfs
dataflow-gitlab-uploads
dataflow-gitlab-packages
dataflow-gitlab-backups
```

备份桶单独一个，方便后面配生命周期和清理。

### Registry 的两个参数

这台 GitLab 的 Registry 是关闭的，以下配置未在本环境启用。19.x 启用 Registry 时存储类型用 `s3_v2`，另有两个参数：

```ruby
registry['storage'] = {
  's3_v2' => {
    'regionendpoint'    => 'http://oss.example.com',  # 必须带协议
    'pathstyle'         => true,
    'checksum_disabled' => true,                      # 不带这个大概率 400
  }
}
```

`regionendpoint` 缺 `http://`，启动报 `plaintext endpoint rule error`；缺 `checksum_disabled`，push 镜像报 `XAmzContentSHA256Mismatch`。

## 迁对象数据

GitLab 数据库存的是对象相对 key，换桶不影响，key 必须保持不变：

```
artifacts: <2>/<2>/<sha>/<date>/<job_id>/<artifact_id>/metadata.gz
lfs:       <2>/<2>/<oid>
uploads:   user/avatar/1/avatar.png
```

用 rclone 从旧 MinIO 复制到新桶：

```bash
rclone copy minio:gitlab-artifacts obs:dataflow-gitlab-artifacts --transfers 8
rclone copy minio:gitlab-lfs       obs:dataflow-gitlab-lfs       --transfers 8
rclone copy minio:gitlab-uploads   obs:dataflow-gitlab-uploads   --transfers 8
rclone copy minio:gitlab-packages  obs:dataflow-gitlab-packages  --transfers 8
```

复制完成后核对两侧的对象数量和总字节。

## 停服迁移

### 新机启动空实例

```bash
docker run -d --name gitlab --restart always \
  --hostname 10.8.3.120 \
  -p 8929:8929 -p 22222:22 \
  -v /data/gitlab/config:/etc/gitlab \
  -v /data/gitlab/logs:/var/log/gitlab \
  -v /data/gitlab/data:/var/opt/gitlab \
  --shm-size 256m \
  gitlab/gitlab-ce:19.5.0-ce.0
```

### 端口映射使用 8929:8929

`external_url` 配的是 `http://10.8.3.120:8929`，容器内 nginx 也监听 8929，映射必须配置为 `8929:8929`。配置为 `8929:80` 时容器内没有服务监听 80，访问一直是 502。

### /dev/shm 设置为 256m

容器默认 `/dev/shm` 为 64MB，PostgreSQL 并行查询报 `could not resize shared memory segment`。官方 Docker 部署文档要求 256m。

### puma worker 设置为 8

32 核机器上 puma 默认按核数启动 32 个 worker，启动后内存占用 14G。当前数据量不需要这么多，配置设置为 8：

```ruby
puma['worker_processes'] = 8
```

reconfigure 会重启 puma，预加载约一分钟，期间访问返回 502。

### 停写与备份

新机空实例能正常登录后开始切换。`gitlab-ctl stop` 一次只生效一个服务，需分两条执行：

```bash
docker exec gitlab gitlab-ctl stop puma
docker exec gitlab gitlab-ctl stop sidekiq
```

nginx 和 postgres 保持运行：puma 停掉后 80 端口直接 502，数据库没有写入方，回滚时启动更快。

随后执行全量备份，产物约 700MB：

```bash
docker exec gitlab gitlab-backup create STRATEGY=copy
```

### gitlab-secrets.json 必须拷贝

备份包里不含这个文件。目标机使用新生成的 secrets，2FA、CI 变量、加密 token 全部无法解密。把 tar 包和 `gitlab-secrets.json` 一起复制到新机。

### 恢复

```bash
# 新机先停写
docker exec gitlab gitlab-ctl stop puma
docker exec gitlab gitlab-ctl stop sidekiq

# 放 secrets 和备份包
cp gitlab-secrets.json /data/gitlab/config/
cp 1794900926_2026_11_17_19.5.0_gitlab_backup.tar /data/gitlab/data/backups/

docker exec gitlab gitlab-backup restore BACKUP=1794900926_2026_11_17_19.5.0 force=yes
docker exec gitlab gitlab-ctl reconfigure
docker exec gitlab gitlab-ctl restart
```

日志最后一行 `Restore task is done` 表示恢复完成。重启后访问会短暂 502，puma 的 8 个 worker 预加载需要约一分钟。

## 验收

项目数、用户数、pipeline 历史、CI 变量都正常，clone 可用。附件和产物验证时遇到两个问题。

### artifacts 404：看 file_type

请求 job 的 artifacts 返回 404。查数据库，该 job 的 `file_type = 3`，是 job log 而非归档包；归档是 `file_type = 1`。这个项目历史上只保留 log，没有 archive，`/artifacts` 本来就该返回 404。

改为读取 job log：

```bash
curl -H "PRIVATE-TOKEN: xxx" \
  http://new:8929/api/v4/projects/1/jobs/412/trace
```

返回 200，日志内容完整，对象存储读路径正常。

### LFS 文件很小：缺少 pull

`git clone` 后 TTF 文件只有几百字节。原因是未安装 smudge 或未执行 pull。执行 `git lfs pull` 后再对比，文件 4.9MB，文件头正常。

### 切换 runner

Runner 是 Docker executor，挂在宿主机上，通过 `/var/run/docker.sock` 调用宿主 daemon 执行 job。可直接修改旧机 runner 的地址：

```bash
sed -i 's|url = "http://10.0.184.74/"|url = "http://10.8.3.120:8929/"|' \
  /srv/gitlab-runner/config/config.toml
docker restart gitlab-runner
docker exec gitlab-runner gitlab-runner verify
```

token 是 `glrt-` 开头，恢复数据库后自动有效，无需重新注册。之后在新机上重新启动 runner，旧机停用。新 runner 需要三个前提：宿主机 `daemon.json` 配置 `insecure-registries`、`gitlab-runner-helper` 镜像本地已有（内网访问不到 GitLab 官方 registry，需 `docker save` 导入）、CI 默认镜像本地已有或可从内网 registry 拉取。

`concurrent` 从 1 调整为 4，避免多项目排队。内网 registry 的 pull 匿名可用，push 需要登录；判断方法是请求不存在的 tag，返回 401 需要认证，返回 not found 则匿名可拉。

## 清理旧数据

迁移前清理了一批测试用户和项目，有四个问题。

### 延迟删除

19.x 移除了 `delayed_project_deletion` / `delayed_group_deletion`，改为 `deletion_adjourned_period`，默认 30 天，最小 1，不能设 0。

立即删除需要两步：先 DELETE 标记，再带 `permanently_remove=true` 永久删除。顶层组不使用该参数。

### 孤儿仓库

项目从数据库删除后，Gitaly 的仓库目录仍留在磁盘上，`.git` 数量与项目数不一致。19.5 已移除 `gitlab-rake gitlab:cleanup:orphan_repos`，需要自行比对：从数据库取全部 `projects.disk_path`，与磁盘 `@hashed/*/*/*.git` 做差集，删除多余目录。

### Sidekiq dead queue

删项目时排队的通知邮件在 job 执行时项目已不存在，报 `Couldn't find Project with 'id'=...`，进入 dead set，累计 200 多条。清理即可，不影响功能。

### PostgreSQL 文件不缩小

删除几十个项目后数据库文件大小不变。MVCC 下 DELETE 只把行标记为 dead tuple，普通 VACUUM 允许空间复用但不缩文件，只有 `VACUUM FULL` 会归还操作系统空间，但它需要排他锁，得停机执行。当前数据量未执行。

## 结果

| 项 | 结果 |
|---|---|
| 停机时间 | 约 25 分钟 |
| 数据量 | 仓库 113MB / 数据库 467MB / 对象 289 个 140MB |
| 对象迁移 | 停机前完成，几分钟 |
| 回滚 | 旧机保留，启动两个服务即可 |

新机备份已配置，`backup_upload` 指向 OBS 备份桶，执行一次确认可上传。

## 几条经验

1. 先核对桶名，控制台的桶列表能直接看出拼写差异。
2. `gitlab-secrets.json` 与备份包一起迁移，备份包中没有该文件。
3. 对象 key 不能改，桶名可以换。
4. `--shm-size 256m` 和端口映射是 Docker 部署的两个易错点。
5. 删除操作注意 19.x 的延迟删除配置项，API 分标记和永久删除两步。
6. 验证按数据类型区分：job log 和 artifacts archive 不同，LFS 需 pull 后才能确认真实文件。
