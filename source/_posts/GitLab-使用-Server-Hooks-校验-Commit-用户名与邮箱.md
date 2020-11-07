---
title: GitLab 使用 Server Hooks 校验 Commit 用户名与邮箱
tags:
  - GitLab
categories:
  - Linux
date: 2020-11-07 15:53:00
---

# 校验原因

在企业内部进行代码提交时, commit 中会存在提交者的 username 与 email

但该 username 与 email 是提交者在 Git 客户端自己设置的

如果提交者忘记设置或者设置错误, 并将 commit push 到远程服务后

当协作者需要寻找该 commit 提交者时, 错误的 username 与 email 会对协作者造成障碍

为解决这个问题, 需要在 GitLab 使用 Server Hooks 对 commit 进行校验, 只有 username 与 email 与 GitLab 中的一致才允许 push, 否则拒绝 push 并提示修改

# 准备工作

如需对 commit 的 username 与 email 进行校验, 那么需要在校验脚本中获取 push 者的 username 与 email

通过 [GitLab Server Hooks](https://docs.gitlab.com/ee/administration/server_hooks.html#environment-variables) 文档可知存在 `GL_USERNAME` 环境变量, 该变量的值为 push 者的 GitLab 的 username, 但是缺乏 email 相关环境变量

为获取 push 者的 email, 需使用 GitLab 提供的 [Users API](https://docs.gitlab.com/ee/api/users.html#list-users) 进行获取

通过 API 文档可知只有 admin 用户才返回用户 email, 所以需要先使用 admin 账号生成一个 TOKEN

这个 TOKEN 只是用来获取获取用户 email, 故创建时选择 read_user 的范围即可

# 校验用户名与邮箱 hook

GitHub 的 [platform-samples](https://github.com/github/platform-samples) 项目提供了一个 [commit-current-user-check.sh](https://github.com/github/platform-samples/blob/master/pre-receive-hooks/commit-current-user-check.sh) 的 hook, 我们可以将该脚本下载下来, 进行修改即可

以下是修改后的 `commit-current-user-check.sh` 文件

```sh
#!/usr/bin/env bash

# 访问 GitLab API 所需的 token, 需使用 admin 用户生成, scope 选择 read_user 即可
# https://docs.gitlab.com/ee/user/profile/personal_access_tokens.html
TOKEN="GitLab admin user read_user TOKEN"

# GitLab 服务的访问地址, 因为该脚本是放置在 GitLab 服务中, 所以使用本机地址即可
GITLAB_URL=http://127.0.0.1

# 使用 python 提取 json 中的 name 和 email 代码
get_name_and_email="import sys, json;
try:
  obj=json.load(sys.stdin);
  print(obj[0]['name']+':'+obj[0]['email'])
except:
    print('error')"

# 访问 GitLab Users API 获取用户显示名称与 email
# GL_USERNAME 为 GitLab 的 username, 一般为英文, 而 commit 中的 username 我们一般设置为中文
# API 返回的数据为 json 格式, 通过 python 代码进行提取显示名称与 email
# 因为显示名称为中文, 为了解决乱码问题, 配置 PYTHONIOENCODING='UTF-8'
# python 返回的格式为 name:email
# https://docs.gitlab.com/ee/api/users.html#list-users
GITLAB_NAME_EMAIL=`curl -s --header "Private-Token: ${TOKEN}" "${GITLAB_URL}/api/v4/users?username=${GL_USERNAME}" | PYTHONIOENCODING='UTF-8' python3 -c "${get_name_and_email}"`

if [ "${GITLAB_NAME_EMAIL}" == "error" ]; then
    echo "Push 异常: GitLab 获取用户信息异常, 请通知管理员进行排查"
    exit 1
fi

# 截取显示名称
GITLAB_USER_NAME=${GITLAB_NAME_EMAIL%:*}
# 截取 email
GITLAB_USER_EMAIL=${GITLAB_NAME_EMAIL#*:}

zero_commit="0000000000000000000000000000000000000000"

excludeExisting="--not --all"

while read oldrev newrev refname; do
  # branch or tag get deleted
  if [ "$newrev" = "$zero_commit" ]; then
    continue
  fi

  # Check for new branch or tag
  if [ "$oldrev" = "$zero_commit" ]; then
    span=`git rev-list $newrev $excludeExisting`
  else
    span=`git rev-list $oldrev..$newrev $excludeExisting`
  fi

  for COMMIT in $span;
   do
        AUTHOR_USER=`git log --format=%an -n 1 ${COMMIT}`
        AUTHOR_EMAIL=`git log --format=%ae -n 1 ${COMMIT}`
        COMMIT_USER=`git log --format=%cn -n 1 ${COMMIT}`
        COMMIT_EMAIL=`git log --format=%ce -n 1 ${COMMIT}`

        # 进行 username 与 email 校验
        # 在 GitHub 的示例脚本中启用了 AUTHOR_USER 与 AUTHOR_EMAIL 校验, 但是使用时可能存在 author 与 committer 不是同一个人的情况, 故注释校验 AUTHOR_USER 与 AUTHOR_EMAIL 的代码
        # 如果自己公司实际使用时不存在这种情况, 可以取消注释
        # author 与 committer 区别 https://stackoverflow.com/q/6755824

#        if [[ ${AUTHOR_USER} != ${GITLAB_USER_NAME} ]]; then
#            echo -e "Push 异常: ${COMMIT} 的 author (${AUTHOR_USER}) 不是 GitLab 中的中文名 (${GITLAB_USER_NAME})"
#            exit 20
#        fi

        if [[ ${COMMIT_USER} != ${GITLAB_USER_NAME} ]]; then
            echo -e "Push 异常: ${COMMIT} 的 committer (${COMMIT_USER}) 不是 GitLab 中的中文名 (${GITLAB_USER_NAME})"
            exit 30
        fi

#        if [[ ${AUTHOR_EMAIL} != ${GITLAB_USER_EMAIL} ]]; then
#            echo -e "Push 异常: ${COMMIT} 的 author 的邮箱 (${AUTHOR_EMAIL}) 不是 GitLab 中的邮箱 (${GITLAB_USER_EMAIL})"
#            exit 40
#        fi

        if [[ ${COMMIT_EMAIL} != ${GITLAB_USER_EMAIL} ]]; then
            echo -e "Push 异常: ${COMMIT} 的 committer 的邮箱 (${COMMIT_EMAIL}) 不是 GitLab 中的邮箱 (${GITLAB_USER_EMAIL})"
            exit 50
        fi
    done
done

exit 0
```


# 配置 hook

由该 hook 功能可知其类型应为 `pre-receive` 

我们按照 [GitLab 全局配置 Server Hook 文档](https://docs.gitlab.com/ee/administration/server_hooks.html#create-a-global-server-hook-for-all-repositories) 将 `commit-current-user-check.sh`  放在 server hook 的 `pre-receive.d` 目录下

并添加可执行权限与配置所属者为 git 用户即可

```console
chmod u+x commit-current-user-check.sh
chown git:git commit-current-user-check.sh
```

# 参考资料

[GitLab Server Hooks 文档](https://docs.gitlab.com/ee/administration/server_hooks.html)

[GitLab API 文档](https://docs.gitlab.com/ee/api/README.html)

[GitLab Personal access tokens 文档](https://docs.gitlab.com/ee/user/profile/personal_access_tokens.html)

[GitLab Users API 文档](https://docs.gitlab.com/ee/api/users.html)
