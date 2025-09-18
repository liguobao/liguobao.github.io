---
title: Gitlab 低版本迁移简易教程
description: 
slug: gitlab-migration-low-version
date: 2024-11-25 08:00:00+0000
# image: girl.png
categories:
  - linux
tags:
  - linux
  - Git
  - Gitlab
---

最近在折腾GitLab迁移，由于原GitLab版本实在是太低了，

同时使用的是被魔改过的，完全不太可能从老版本跨几个大版本升级到最新版本，

所以最后决定写Python脚本调用Gitlab API导出用户和分组以及项目，

接着通过Gitlab API添加到新的Gitlab中，然后把源码仓库整个文件夹拷贝到新服务器，

使用Python执行shell命令，把所有的git 裸仓库 提交到新的Gitlab中。

## 导出用户、分组、项目

```python

import requests
import json
from loguru import logger
from datetime import datetime

# GitLab 域名
gitlab_domain = "gitlab.com"
# GitLab 配置
GITLAB_URL = f'https://{domain}'  # 你的 GitLab 实例的 URL
API_TOKEN = 'xxx'  # 你的 GitLab 访问令牌

# 老版本的Auth Token在 Header
HEADERS = {'PRIVATE-TOKEN': f'{API_TOKEN}'}

# 获取所有项目的函数
def get_all_projects():
    url = f"{GITLAB_URL}/api/v4/projects"
    params = {
        "page": 1,  # 默认第一页
        "per_page": 100  # 每页最多返回100个项目
    }
    projects = []

    while True:
        response = requests.get(url, headers=HEADERS, params=params)

        if response.status_code != 200:
            logger.error(f"Error: Unable to fetch data. Status code {response.status_code}")
            break

        data = response.json()

        if not data:
            logger.info("No more projects to fetch.")
            break

        projects.extend(data)
        params["page"] += 1  # 获取下一页

        logger.info(f"Fetched {len(data)} projects from page {params['page'] - 1}.")

    return projects

# 提取所需的项目数据
def extract_project_info(projects, rep_base):
    user_project_list = [
        {
            'name': p["name"],
            'path_with_namespace': p["path_with_namespace"],
            'default_branch': p["default_branch"],
            'visibility': p["visibility"],
            'ssh_url_to_repo': p["ssh_url_to_repo"],
            'absolute_path': rep_base + p["ssh_url_to_repo"].replace(f'git@{gitlab_domain}:', ""),
            'new_ssh_url_to_repo': p["ssh_url_to_repo"].replace(f"git@{gitlab_domain}:", "git@127.0.0.1:"),
        }
        for p in projects
    ]
    return user_project_list

# 保存数据到文件
def save_to_file(data, filename="old_all_projects.json"):
    with open(filename, "w", encoding="utf-8") as f:
        json.dump(data, f, indent=4, ensure_ascii=False)
    logger.info(f"Projects data has been saved to '{filename}'.")

# 主函数
def main():
    rep_base = "/var/opt/gitlab/git-data/back_repositories/repositories/"  # 你自己的仓库路径
    logger.info("Starting the project extraction process.")

    try:
        # 获取所有项目
        projects = get_all_projects()

        if not projects:
            logger.warning("No projects found or there was an issue fetching data.")
            return

        # 提取项目信息
        user_project_list = extract_project_info(projects, rep_base)

        # 保存项目列表到文件
        save_to_file(user_project_list)

        logger.info("Process completed successfully.")
    except Exception as e:
        logger.error(f"An error occurred: {e}")

if __name__ == "__main__":
    main()

```

## 创建用户、分组、项目

```python

import requests
import json
import random
import string
from loguru import logger

# 新 GitLab 实例的 URL 和个人访问令牌
NEW_GITLAB_URL = 'https://new-gitlab.com'  # 新的 GitLab 实例 URL
API_TOKEN = 'xxxx'  # 新的 GitLab 管理员访问令牌

# 新版本的鉴权Header 用的 Authorization
HEADERS = {
    'Authorization': f'Bearer {API_TOKEN}'
}
# 随机生成密码
def generate_random_password(length=12):
    characters = string.ascii_letters + string.digits + string.punctuation
    return ''.join(random.choice(characters) for i in range(length))

# 从 JSON 文件中读取备份数据
def load_backup_data():
    with open("dump_gitlab_data.json", "r", encoding="utf-8") as f:
        return json.load(f)

# 创建用户
def create_user(username, email, password):
    url = f"{NEW_GITLAB_URL}/api/v4/users"
    data = {
        "username": username,
        "name": username,
        "email": email,
        "password": password,
        "skip_confirmation": True,
        "force_random_password": True  # 强制用户首次登录时修改密码
    }
    try:
        response = requests.post(url, headers=HEADERS, data=data)
        response.raise_for_status()
        logger.info(f"User {username} created successfully.")
        return response.json()  # 返回包含用户信息的 JSON
    except requests.RequestException as e:
        logger.error(f"Error creating user {username}: {e}")
        return None

# 创建项目
def create_project(group_id, project_name):
    url = f"{NEW_GITLAB_URL}/api/v4/projects"
    data = {
        "name": project_name,
        "namespace_id": group_id,  # 将项目放入指定的组
        "visibility": "private",
        "initialize_with_readme": False  # 默认创建空项目
    }
    try:
        response = requests.post(url, headers=HEADERS, data=data)
        response.raise_for_status()
        logger.info(f"Project {project_name} created successfully in group {group_id}.")
        return response.json()  # 返回创建的项目数据
    except requests.RequestException as e:
        logger.error(f"Error creating project {project_name} in group {group_id}: {e}")
        return None

# 创建分组
def create_group(group_name):
    url = f"{NEW_GITLAB_URL}/api/v4/groups"
    data = {
        "name": group_name,
        "path": group_name.lower().replace(" ", "_")  # 自动生成路径
    }
    try:
        response = requests.post(url, headers=HEADERS, data=data)
        response.raise_for_status()
        logger.info(f"Group {group_name} created successfully.")
        return response.json()  # 返回包含分组信息的 JSON
    except requests.RequestException as e:
        logger.error(f"Error creating group {group_name}: {e}")
        return None

# 添加用户到分组并设置权限为 master
def add_user_to_group(group_id, user_id, access_level=40):
    url = f"{NEW_GITLAB_URL}/api/v4/groups/{group_id}/members"
    data = {
        "user_id": user_id,
        "access_level": access_level  # 权限设置为 master (40)
    }
    try:
        response = requests.post(url, headers=HEADERS, data=data)
        response.raise_for_status()
        logger.info(f"User {user_id} added to group {group_id} with master access.")
    except requests.RequestException as e:
        logger.error(f"Error adding user {user_id} to group {group_id}: {e}")

# 导入用户、分组和项目
def import_data():
    logger.add("import_gitlab_entities.log", rotation="500 MB", compression="zip")

    # 读取备份数据
    backup_data = load_backup_data()

    # 存储用户名、邮箱、密码
    user_passwords = []

    # 创建用户
    for user in backup_data["users"]:
        username = user["username"]
        email = user["email"]
        password = generate_random_password()

        user_data = create_user(username, email, password)
        if user_data:
            user_id = user_data["id"]
            user_passwords.append({"username": username, "email": email, "password": password})

            # 创建用户的项目
            for project in user["projects"]:
                project_name = project["name"]
                # 查找项目所属的分组
                group_name = project["namespace"]["name"]
                # 创建项目时需要使用分组ID，首先获取分组的 ID
                group_id = None
                for group in backup_data["groups"]:
                    if group["name"] == group_name:
                        group_id = create_group(group_name)["id"]
                        break
                if group_id:
                    create_project(group_id, project_name)
                    # 创建项目后，将用户添加到分组中
                    add_user_to_group(group_id, user_id)

    # 创建分组和项目
    for group in backup_data["groups"]:
        group_name = group["name"]
        group_data = create_group(group_name)
        if group_data:
            group_id = group_data["id"]
            # 创建分组中的项目
            for project in group["projects"]:
                project_name = project["name"]
                create_project(group_id, project_name)

            # 添加对应的用户到分组，并设置为 master 权限
            for group_user in group["users"]:
                add_user_to_group(group_id, group_user["user_id"])

    # 将用户名、邮箱、密码保存到 JSON 文件
    with open("user_passwords.json", "w", encoding="utf-8") as f:
        json.dump(user_passwords, f, ensure_ascii=False, indent=4)

    logger.info("User passwords have been exported to user_passwords.json")

if __name__ == "__main__":
    import_data()

```

## 提交Git 裸仓库

```python

import os
import subprocess
import json
from loguru import logger

# 读取配置的 JSON 文件
def load_config(file_path):
    with open(file_path, 'r', encoding='utf-8') as f:
        return json.load(f)

# 将操作结果写入到 JSON 文件
def write_result_to_json(results, output_file):
    with open(output_file, 'w+', encoding='utf-8') as f:
        json.dump(results, f, indent=4, ensure_ascii=False)

# 推送裸仓库到新远端
def push_bare_repo_to_remote(project, results):
    absolute_path = project["absolute_path"]
    new_ssh_url_to_repo = project["new_ssh_url_to_repo"]

    # 检查仓库路径是否合法
    if not os.path.isdir(absolute_path) or not os.path.exists(os.path.join(absolute_path, "HEAD")):
        logger.error(f"Invalid bare repository path: {absolute_path}")
        results.append({
            "project": project,
            "status": "failed",
            "message": f"Invalid bare repository path: {absolute_path}"
        })
        return

    try:
        # 进入原始仓库目录
        logger.info(f"Accessing local bare repository: {absolute_path}")
        os.chdir(absolute_path)

        # 确保是裸仓库
        subprocess.run(["git", "rev-parse", "--is-bare-repository"], check=True)

        # 设置新的远端地址
        logger.info(f"Setting new remote URL: {new_ssh_url_to_repo}")
        subprocess.run(["git", "remote", "add", "new-origin", new_ssh_url_to_repo], check=True)

        # 推送所有分支到新远端
        logger.info("Pushing all branches to the new remote repository")
        subprocess.run(["git", "push", "new-origin", "--all","-f"], check=True)

        # 推送所有标签到新远端
        logger.info("Pushing all tags to the new remote repository")
        subprocess.run(["git", "push", "new-origin", "--tags"], check=True)

        results.append({
            "project": project,
            "status": "success",
            "message": f"Successfully pushed to {new_ssh_url_to_repo}"
        })

    except subprocess.CalledProcessError as e:
        logger.error(f"Error during git operations: {e}")
        results.append({
            "project": project,
            "status": "failed",
            "message": f"Error during git operations: {str(e)}"
        })
    except Exception as e:
        logger.exception("Unexpected error occurred")
        results.append({
            "project": project,
            "status": "failed",
            "message": f"Unexpected error: {str(e)}"
        })
    finally:
        # 移除新远端配置以保持原始仓库清洁
        try:
            subprocess.run(["git", "remote", "remove", "new-origin"], check=True)
        except subprocess.CalledProcessError:
            logger.warning("Failed to remove new-origin remote, please check manually.")

# 主函数
def main():
    config_file = 'old_all_projects.json'
    output_file = 'old_all_projects_result_2024.json'
    projects = load_config(config_file)
    logger.add("all_projects_operation_log_2024.log", rotation="500 MB", level="INFO", compression="zip")
    results = []

    for project in projects:
        logger.info(f"Processing project: {project['name']}")
        push_bare_repo_to_remote(project, results)

    write_result_to_json(results, output_file)
    logger.info(f"Operation results saved to {output_file}")

if __name__ == "__main__":
    main()

```