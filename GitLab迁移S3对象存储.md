# GitLab 19.x 迁移实战：从本地 MinIO 切换到 S3 兼容对象存储

## 前言

GitLab 上线一段时间后，常见的两个诉求会同时出现：

- 服务器要换（扩容、换机房、上云）
- 附件、CI 产物、LFS 大文件把本地盘吃满了

这时最自然的方案是：**把 GitLab 迁到新机器，同时把文件存储切到 S3 兼容对象存储**（MinIO、华为 OBS、阿里 OSS、Ceph RGW、AWS S3 都行）。

本文以 GitLab 19.5（Omnibus / Docker 部署）为例，完整讲清楚三件事：

1. GitLab 的数据到底哪些能放对象存储
2. S3 兼容存储怎么配（含 19.x 的 Registry 变更）
3. 迁移怎么做到不丢数据、可回滚

文中密钥、IP、桶名均为占位，替换成实际值即可。

---

## 一、先理清 GitLab 的数据边界

迁移方案的前提，是知道什么东西在哪。

| 数据 | 存储位置 | 能否放 S3 |
|---|---|---|
| Git 仓库（提交、分支、Wiki） | Gitaly 管理的本地文件系统 | ❌ |
| PostgreSQL（用户、项目、Issue、CI 变量、Pipeline、Runner） | 本地卷 | ❌ |
| Redis | 本地 | ❌ |
| CI artifacts | S3 | ✅ |
| Git LFS | S3 | ✅ |
| Web 上传附件 / 头像 | S3 | ✅ |
| Package Registry | S3 | ✅ |
| Container Registry | S3（19.x 用 `s3_v2`） | ✅ |
| 备份包 | S3 | ✅ |

**为什么 Git 仓库不能放对象存储？**

Git 依赖原子 rename、文件锁、mmap 和目录遍历。对象存储是扁平的键值语义，没有这些能力。用 s3fs 之类把桶挂成本地盘再给 Gitaly 使用，会遇到写丢失、仓库损坏。代码仓库的正确做法是本地盘、块存储或 Gitaly Cluster。

所以迁移时真正需要搬的本地数据只有：

```
git-data/repositories           Git 仓库
postgresql                      数据库
gitlab.rb + gitlab-secrets.json 配置与加密密钥
```

其余"大文件"都在对象存储里，单独迁移。

---

## 二、S3 兼容对象存储配置

### 2.1 19.x 必须使用 consolidated 写法

老版本分散式配置（`artifacts_object_store_enabled` 那一套）在 19.x 已移除。统一写法：

```ruby
gitlab_rails['object_store']['enabled'] = true
gitlab_rails['object_store']['connection'] = {
  'provider'              => 'AWS',
  'region'                => 'us-east-1',
  'aws_access_key_id'     => '<ACCESS_KEY>',
  'aws_secret_access_key' => '<SECRET_KEY>',
  'endpoint'              => 'http://s3.example.com',
  'path_style'            => true,
  'aws_signature_version' => 4
}

gitlab_rails['object_store']['objects']['artifacts']['bucket'] = 'gitlab-artifacts'
gitlab_rails['object_store']['objects']['lfs']['bucket']       = 'gitlab-lfs'
gitlab_rails['object_store']['objects']['uploads']['bucket']   = 'gitlab-uploads'
gitlab_rails['object_store']['objects']['packages']['bucket']  = 'gitlab-packages'
```

### 2.2 连接参数四要素

非 AWS 的 S3 兼容存储，以下四项必须正确：

| 参数 | 要求 | 常见错误 |
|---|---|---|
| `endpoint` | 带协议，指向 API 端口 | 写成 Console 端口，全部超时 |
| `path_style` | 一律 `true` | 不开会走 `bucket.endpoint` 虚拟主机风格，DNS 解析失败 |
| `region` | 服务端可接受的值，多数兼容存储填 `us-east-1` | 填错报 `AuthorizationHeaderMalformed` |
| `aws_signature_version` | `4` | 用 v2 报 `SignatureDoesNotMatch` |

### 2.3 桶规划

推荐一类对象一个桶：

```
gitlab-artifacts
gitlab-lfs
gitlab-uploads
gitlab-packages
gitlab-backups        # 备份必须独立
gitlab-registry       # 启用 Registry 才需要
```

不建议所有类型挤一个桶：无法按类型设置配额和生命周期，备份与业务对象混放容易误删，出问题也难定位。

如果对象存储账号没有建桶权限，只能先用单桶跑通，再找管理员拆分。

### 2.4 Container Registry：19.x 强制 `s3_v2`

```ruby
registry['enable'] = true
registry['storage'] = {
  's3_v2' => {
    'accesskey'         => '<ACCESS_KEY>',
    'secretkey'         => '<SECRET_KEY>',
    'bucket'            => 'gitlab-registry',
    'region'            => 'us-east-1',
    'regionendpoint'    => 'http://s3.example.com',
    'pathstyle'         => true,
    'checksum_disabled' => true
  }
}
```

三个 19.x 专属问题：

| 现象 | 原因 | 处理 |
|---|---|---|
| `plaintext endpoint rule error` | `s3_v2` 要求 `regionendpoint` 带协议 | 写完整 `http://` |
| Push 镜像 400 `XAmzContentSHA256Mismatch` | 默认发送增强校验和，部分兼容实现不支持 | `checksum_disabled => true` |
| Registry 起不来 | `s3`（v1）驱动在 19.0 已移除 | 改用 `s3_v2` |

### 2.5 备份写入对象存储

```ruby
gitlab_rails['backup_upload_connection'] = {
  'provider'              => 'AWS',
  'region'                => 'us-east-1',
  'aws_access_key_id'     => '<ACCESS_KEY>',
  'aws_secret_access_key' => '<SECRET_KEY>',
  'endpoint'              => 'http://s3.example.com',
  'path_style'            => true
}
gitlab_rails['backup_upload_remote_directory'] = 'gitlab-backups'
gitlab_rails['backup_multipart_chunk_size']    = 104857600
```

### 2.6 配置前先验证，不要等 reconfigure

改配置之前，先用 SDK 把桶和权限验证一遍，避免"配完才发现桶名写错"。

```python
import boto3
from botocore.config import Config

cfg = Config(signature_version="s3v4", s3={"addressing_style": "path"})
s3 = boto3.client("s3", endpoint_url="http://s3.example.com",
                  aws_access_key_id="<AK>", aws_secret_access_key="<SK>",
                  region_name="us-east-1", config=cfg)

s3.head_bucket(Bucket="gitlab-artifacts")
s3.list_objects_v2(Bucket="gitlab-artifacts")
s3.put_object(Bucket="gitlab-artifacts", Key="_probe/x.txt", Body=b"x")
print(s3.get_object(Bucket="gitlab-artifacts", Key="_probe/x.txt")["Body"].read())
s3.delete_object(Bucket="gitlab-artifacts", Key="_probe/x.txt")
```

GitLab 上传大文件依赖 multipart，必须单独验证：

```ruby
mp   = c.create_multipart_upload(bucket: b, key: k)
part = c.upload_part(bucket: b, key: k, upload_id: mp.upload_id,
                     part_number: 1, body: "x" * 5_242_880)
c.complete_multipart_upload(bucket: b, key: k, upload_id: mp.upload_id,
  multipart_upload: { parts: [{ part_number: 1, etag: part.etag }] })
```

更贴近真实链路的做法，是在 GitLab 容器内部用自带的 `aws-sdk-s3` 验证：

```bash
docker exec gitlab /opt/gitlab/embedded/bin/ruby -e '
require "aws-sdk-s3"
c = Aws::S3::Client.new(endpoint: "http://s3.example.com",
  access_key_id: "<AK>", secret_access_key: "<SK>",
  region: "us-east-1", force_path_style: true)
puts c.list_buckets.buckets.map(&:name)
'
```

### 2.7 `proxy_download` 怎么选

```ruby
gitlab_rails['object_store']['proxy_download'] = false
```

| 值 | 行为 | 适用场景 |
|---|---|---|
| `true` | 下载经 GitLab 中转 | 客户端无法直连对象存储 |
| `false` | 302 跳转，客户端直连对象存储 | 大文件多，性能优先 |

LFS 动辄几百 MB，长期 `true` 会持续占用 GitLab 带宽。

---

## 三、迁移实施

### 3.1 前置条件

- 源机与目标机 **GitLab 版本完全一致**
- 能拿到 `gitlab-secrets.json`（没有它，备份也无法恢复）
- 目标机有独立数据盘，存放 git-data 与 PostgreSQL
- 对象存储凭据、桶、multipart、容器网络连通性验证通过
- 源站先做一次全量备份
- 明确对象迁移范围：历史 artifacts / LFS / uploads 是否全部迁移

### 3.2 阶段划分

| 阶段 | 内容 | 是否停机 |
|---|---|---|
| 0 | 目标机部署同版本 GitLab + 对象存储配置 | 否 |
| 1 | 对象迁移，保持 key 不变 | 否 |
| 2 | 停写 + 全量备份 | 是 |
| 3 | 恢复数据库与仓库 | 是 |
| 4 | 验收 | 是 |
| 5 | 切换 runner、webhook、访问入口 | 否 |
| 6 | 旧机保留回滚窗口 | — |

### 3.3 目标机部署

```bash
docker run -d --name gitlab --restart always \
  --hostname gitlab.example.com \
  -p 8929:8929 -p 22222:22 \
  -v /data/gitlab/config:/etc/gitlab \
  -v /data/gitlab/logs:/var/log/gitlab \
  -v /data/gitlab/data:/var/opt/gitlab \
  --shm-size 256m \
  gitlab/gitlab-ce:19.5.0-ce.0
```

两个细节：

- `external_url` 带端口时，容器内 nginx 监听该端口，映射必须是 `8929:8929`，写 `8929:80` 会 502
- `--shm-size 256m` 不能省，PostgreSQL 并行查询使用 `/dev/shm`

配置完成后：

```bash
docker exec gitlab gitlab-ctl reconfigure
docker exec gitlab gitlab-ctl status
```

### 3.4 对象迁移

GitLab 数据库记录的是对象的相对 key，**桶名可以变，key 不能变**：

```
artifacts: <2>/<2>/<sha256>/<date>/<job_id>/<artifact_id>/metadata.gz
lfs:       <2>/<2>/<oid>
uploads:   user/avatar/<id>/avatar.png
```

用 rclone 或任意 S3 SDK 复制，完成后双向核对对象数与总字节：

```bash
rclone copy src-bucket dst-bucket --checksum --transfers 8 --progress
rclone size src-bucket
rclone size dst-bucket
```

### 3.5 停写与备份

```bash
gitlab-ctl stop puma
gitlab-ctl stop sidekiq
```

> `gitlab-ctl stop puma sidekiq` 一次只生效一个服务，必须分开执行。
> `maintenance_mode` 只是只读页面，后台任务仍可能写库，不适合要求零写入的切换。

```bash
gitlab-backup create STRATEGY=copy
# /var/opt/gitlab/backups/<timestamp>_<version>_gitlab_backup.tar
```

### 3.6 恢复

先把 secrets 放好，再恢复：

```bash
cp gitlab-secrets.json /etc/gitlab/gitlab-secrets.json
cp <backup>.tar /var/opt/gitlab/backups/ && chown git:git /var/opt/gitlab/backups/<backup>.tar

gitlab-ctl stop puma
gitlab-ctl stop sidekiq
gitlab-backup restore BACKUP=<timestamp> force=yes
gitlab-ctl reconfigure
gitlab-ctl restart
gitlab-rake gitlab:check SANITIZE=true
```

备份包不包含：

- `gitlab.rb`
- `gitlab-secrets.json`
- 对象存储中的 artifacts / LFS / uploads / packages

### 3.7 验收清单

- [ ] 登录页 200，管理员可登录
- [ ] 用户数、项目数、组数与源站一致
- [ ] `git clone` 成功，提交历史正确
- [ ] 历史 job log / artifacts 可下载
- [ ] `git lfs pull` 成功，文件大小正常（不是指针文件）
- [ ] Web 附件、头像可显示
- [ ] CI/CD 变量存在且能解密
- [ ] Runner 记录存在且能连上
- [ ] 对象存储对象数与迁移前一致

### 3.8 切换与回滚

Runner 改指向：

```bash
sed -i 's|url = "http://old/"|url = "http://new:8929/"|' /etc/gitlab-runner/config.toml
docker restart gitlab-runner
docker exec gitlab-runner gitlab-runner verify
```

用户侧会涉及 git remote 地址变更，处理方式二选一：

- 通知全员改 remote
- 在新旧地址之间做反向代理，保持 URL 不变

回滚：

```bash
gitlab-ctl start puma
gitlab-ctl start sidekiq
```

旧机数据在切换阶段不删除，这是回滚成本最低的做法。

---

## 四、常见问题

**1. `external_url` 带端口导致 502**

容器内 nginx 监听 `external_url` 的端口，端口映射必须同端口。同时设置：

```ruby
gitlab_rails['gitlab_shell_ssh_port'] = 22222
```

否则 SSH clone URL 仍显示 22。

**2. puma 进程数按 CPU 自动放大**

32 核机器默认 `workers 32 × threads 4 = 128` 并发，内存可到 14GB。中小团队：

```ruby
puma['worker_processes'] = 8
```

reconfigure 后 puma 重启，预加载期间约 1 分钟 502，属正常现象。

**3. 备份恢复后 2FA / CI 变量失效**

secrets 未替换。恢复前必须用源站的 `gitlab-secrets.json` 覆盖目标机文件。

**4. 对象存储部分功能异常**

典型表现是 artifacts 正常、LFS 403。逐桶验证，确认桶名、key 前缀和权限，而不是整体配置重写。

**5. 删除大量项目后的残留**

- Gitaly 留下孤儿仓库，磁盘 `.git` 数量与数据库对不上
- Sidekiq dead queue 堆积删除通知邮件
- PostgreSQL 文件大小不下降

孤儿仓库可按数据库 `projects.disk_path` 对比清理；dead queue 可以直接清空失败任务；数据库空间问题见第 7 条。

**6. 延迟删除配置在 19.x 已改名**

旧版的 `delayed_project_deletion` / `delayed_group_deletion` 已移除，现在统一为 `deletion_adjourned_period`，默认 30 天，最小 1 天，不能设为 0。

API 层面，顶层组的 `permanently_remove` 参数只对子组有效；项目需要"先标记删除，再永久删除"两步。

**7. PostgreSQL 删除数据后不释放磁盘**

MVCC 机制下，DELETE 只是把行标记为 dead tuple。普通 VACUUM 让空间可复用但不缩小文件，只有 `VACUUM FULL` 才会归还操作系统：

```sql
VACUUM FULL VERBOSE;
```

该操作持有排他锁，需要停机窗口，并要求约等于表大小的临时空间。

另外 autovacuum 的触发阈值是 `50 + 20% × 表行数`，小表删几十行根本不会触发。

**8. 内网 Registry 的拉取与推送权限可能不同**

部分私有 Registry 允许匿名 pull，只有 push 需要登录。判断方法：

```bash
docker pull registry.example.com/repo:not-exist
```

返回 401 说明需要认证，返回 not found 说明匿名可拉。

**9. Runner 的 helper 镜像必须本地可用**

Runner 每个 job 都会使用 `gitlab-runner-helper:<arch>-<version>`。内网无法访问 GitLab 官方 Registry 时，需要 `docker save/load` 同步，且镜像名不能修改。

**10. `pull_policy = if-not-present` 会复用旧镜像**

CI 使用固定 tag（如 `latest`）时，本地有同名镜像就不会重新拉取。要么使用唯一 tag（commit sha），要么改为 `always`。

**11. 备份包在本地持续累积**

每次全量备份大小约等于 PostgreSQL + 仓库体积。配置定时任务时必须同时设置保留策略：

```bash
# 每天 2 点，本地保留 3 天
0 2 * * * docker exec gitlab gitlab-backup create CRON=1
find /data/gitlab/data/backups -name '*_gitlab_backup.tar' -mtime +3 -delete
```

对象存储侧建议加 30 天生命周期规则。

---

## 五、容量规划

| 对象 | 增长来源 | 说明 |
|---|---|---|
| git-data | 项目数 × 代码体积 | 本地盘，增长慢 |
| PostgreSQL | Issue / MR / Pipeline 记录 | 本地盘，增长慢 |
| artifacts | 构建频率 × 产物大小 × 保留天数 | 最大变量 |
| lfs | 设计稿、模型、二进制文件 | 无自动过期，需人工清理 |
| uploads | 附件、头像 | 量小 |
| backups | 全量大小 × 保留份数 | 注意累积 |

artifacts 年增长估算：

```
项目数 × 每次产物大小 × 每日构建次数 × 保留天数
```

300 个项目 × 20MB × 5 次/天 × 90 天 = 2.7TB。

因此 artifacts 必须配置过期策略，否则配额再大也会用尽：

```yaml
artifacts:
  expire_in: 30 days
```

---

## 六、总结

1. Git 仓库与 PostgreSQL 始终在本地盘，对象存储只接管"文件"。
2. S3 兼容配置的核心是四项：`endpoint` 带协议、`path_style=true`、`signature v4`、`region` 正确；19.x Registry 使用 `s3_v2` 并关闭增强校验和。
3. 迁移顺序：对象先行（不停机）→ 停写 → 备份 → 恢复 → 验收 → 切换 → 保留回滚窗口。
4. 两条不能错的红线：`gitlab-secrets.json` 与对象 key。
5. 备份必须做到位：本地保留若干份、对象存储长期保留、定期演练恢复。

按这个流程执行，一个中小规模 GitLab（数十项目、GB 级数据）的迁移停机时间可以控制在 30 分钟以内。
