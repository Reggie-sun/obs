
重启后如何验证 NFS 共享仍然生效”的最小检查清单。

**API 机器重启后检查**

1. 确认 NFS 服务还在

`systemctl status nfs-kernel-server --no-pager systemctl is-enabled nfs-kernel-server`

2. 确认导出规则还在

`sudo exportfs -v | grep /mnt/enterprise-rag-shared/data cat /etc/exports | grep /mnt/enterprise-rag-shared/data`

3. 确认共享源目录和文件还在

`findmnt -T /mnt/enterprise-rag-shared/data ls -l /mnt/enterprise-rag-shared/data/jobs/folder_delete/job_20260429012606_5a3b92c2.json`

这里 findmnt 在 API 机器上看到本地 ext4 是正常的，因为它是服务端，不是客户端。

**worker 机器重启后检查**

1. 确认 fstab 规则还在

`grep '/mnt/enterprise-rag-shared/data' /etc/fstab`

2. 确认 NFS 自动挂载成功

`findmnt -T /mnt/enterprise-rag-shared/data mount | grep '/mnt/enterprise-rag-shared/data'`

这里应该看到：

- source 是 192.168.10.69:/mnt/enterprise-rag-shared/data
- fstype 是 nfs4

3. 确认共享文件能看到

`ls -l /mnt/enterprise-rag-shared/data/jobs/folder_delete/job_20260429012606_5a3b92c2.json`

4. 确认 worker 容器里也能看到

`docker exec enterprise-rag-worker python -c "from backend.app.core.config import get_settings; from pathlib import Path; s=get_settings(); p=s.job_dir/'folder_delete'/'job_20260429012606_5a3b92c2.json'; print('data_dir=', s.data_dir); print('job_dir=', s.job_dir); print('exists=', p.exists())"`

期望还是：

`data_dir= /shared-data job_dir= /shared-data/jobs exists= True`

**如果 worker 机器重启后没自动挂上**

直接补救：

`sudo mount -a findmnt -T /mnt/enterprise-rag-shared/data`

如果还不行，再看日志：

`journalctl -b -u remote-fs.target --no-pager journalctl -b | grep nfs`

**最后的业务检查**

两边都起来后，补一个最小业务验证：

1. 新上传一个小文件
2. 看 job 是否从 queued 到 completed
3. 再试一次文件夹删除，看是否不再卡在 0%

本地默认make dev-local-api和make dev-local-frontend开发测试，后面复刻到服务器并上线。重点突出这两个环境不一样。写进AGENTS.md