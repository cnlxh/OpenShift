```textile
作者：李晓辉

微信：Lxh_Chat

邮箱：xiaohui_li@foxmail.com
```

# 准备软件仓库

```bash
curl -s https://packages.gitlab.com/install/repositories/gitlab/gitlab-ce/script.rpm.sh | sudo bash
```

# 安装Gitlab

```bash
yum install gitlab-ce -y
```

# 配置Gitlab

```bash
vim /etc/gitlab/gitlab.rb
```

配置访问域名，本内容大约在32行左右

```bash
external_url "http://content.cluster1.xiaohui.cn:5000"
```

配置代码仓库默认存储位置，本内容大约在629行左右

```bash
mkdir /gitcode
```

```bash
git_data_dirs({
  "default" => {
    "path" => "/gitcode"
   }
})
```

定义ROOT密码并完成配置，这里也可以不定义密码，会自动生成复杂的密码

```bash
 export GITLAB_ROOT_PASSWORD=Steven0608
 gitlab-ctl reconfigure
```

如果没有定义ROOT密码，一定要注意观察最后执行的结果，用户名为root，密码在/etc/gitlab/initial_root_password文件中，一定保存好，并重置密码，这个文件24小时后会删除

```bash
firewall-cmd --add-port=5000/tcp --permanent
firewall-cmd --reload
```

配置完成之后，就可以打开页面进行工作了

登录后，点击New Project---Create blank project，输入project name，在Project URL中，选择合适的namespace，例如root，并输入project slug，选择可见级别，我这里选择了public，并点击Create Project

# 克隆仓库到本地

在点击Create Project之后，会看到我们新建的Project，点击Clone，并复制Clone with HTTP中的代码

在需要此仓库的机器上安装git客户端

```bash
yum install git -y
```

克隆仓库到本地

在执行了克隆之后，当前路径就会有一个和仓库同名的文件夹

```bash
git clone http://content.cluster1.xiaohui.cn:5000/root/openshift.git
```

# 提交代码到仓库

此处我们在本地工作树中新建一个文件

```bash
cd openshift
echo hello world > index.html
```

在Git提交的时候，我们需要标明我们是谁，邮箱是多少，执行下方命令完成设置

```bash
git config --global user.name root
git config --global user.email root@content.cluster1.xiaohui.cn
```

查看本地工作树是否有文件还未加到暂存区，此时提示我们有一个index.html还未被跟踪

```bash
[root@content openshift]# git status
On branch main

No commits yet

Untracked files:
  (use "git add <file>..." to include in what will be committed)
        index.html

nothing added to commit but untracked files present (use "git add" to track)
```

添加文件到暂存区，注意git add 后面有一个英文句号，表示当前路径，你可以直接用文件名代替这个句号，再次查询，我们已经有一个新文件在暂存区，不合适还可以执行它建议的命令从暂存区删除

```bash
[root@content openshift]# git add .
[root@content openshift]# git status
On branch main

No commits yet

Changes to be committed:
  (use "git rm --cached <file>..." to unstage)
        new file:   index.html
```

将本地工作树提交到本地仓库

```bash
git commit -m 'create index.html'
```

将本地仓库的文件推送到远程仓库，在此过程中，请输入你的用户名和密码

```bash
[root@content openshift]# git push
Username for 'http://content.cluster1.xiaohui.cn:5000': root
Password for 'http://root@content.cluster1.xiaohui.cn:5000':
Enumerating objects: 3, done.
Counting objects: 100% (3/3), done.
Writing objects: 100% (3/3), 233 bytes | 233.00 KiB/s, done.
Total 3 (delta 0), reused 0 (delta 0), pack-reused 0
To http://content.cluster1.xiaohui.cn:5000/root/openshift.git
 * [new branch]      main -> main
```
