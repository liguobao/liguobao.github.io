---
title: "GitLab Low Version Migration Simple Tutorial"
description: "GitLab, Migration, Low Version"
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

Recently working on GitLab migration, since the original GitLab version is too low, and it's modified, completely impossible to upgrade from old version across several major versions to latest. So finally decided to write Python script to call GitLab API to export users, groups, and projects, then add to new GitLab via API, then copy the source repository entire folder to new server, use Python to execute shell commands, push all git bare repositories to new GitLab.

## Export Users, Groups, Projects

```python
import requests
import json
from loguru import logger
from datetime import datetime

# GitLab domain
gitlab_domain = "gitlab.com"
# GitLab config
GITLAB_URL = f'https://{domain}'  # Your GitLab instance URL
API_TOKEN = 'xxx'  # Your GitLab access token

# Old version Auth Token in Header
HEADERS = {'PRIVATE-TOKEN': f'{API_TOKEN}'}

# Function to get all projects
def get_all_projects():
    url = f"{GITLAB_URL}/api/v4/projects"
    params = {
        "page": 1,  # Default first page
        "per_page": 100  # Max 100 projects per page
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
        params["page"] += 1  # Get next page

        logger.info(f"Fetched {len(data)} projects from page {params['page'] - 1}.")

    return projects

# Extract required project data
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

# Save data to file
def save_to_file(data, filename="old_all_projects.json"):
    with open(filename, "w", encoding="utf-8") as f:
        json.dump(data, f, indent=4, ensure_ascii=False)
    logger.info(f"Projects data has been saved to '{filename}'.")

# Main function
def main():
    rep_base = "/var/opt/gitlab/git-data/back_repositories/repositories/"  # Your own repository path
    logger.info("Starting the project extraction process.")

    try:
        # Get all projects
        projects = get_all_projects()

        if not projects:
            logger.warning("No projects found or there was an issue fetching data.")
            return

        # Extract project info
        user_project_list = extract_project_info(projects, rep_base)

        # Save project list to file
        save_to_file(user_project_list)

        logger.info("Process completed successfully.")
    except Exception as e:
        logger.error(f"An error occurred: {e}")

if __name__ == "__main__":
    main()
```

## Create Users, Groups, Projects

```python
import requests
import json
import random
import string
from loguru import logger

# New GitLab instance URL and personal access token
NEW_GITLAB_URL = 'https://new-gitlab.com'  # New GitLab instance URL
API_TOKEN = 'xxxx'  # New GitLab admin access token

# New version auth header uses Authorization
HEADERS = {
    'Authorization': f'Bearer {API_TOKEN}'
}

# Generate random password
def generate_random_password(length=12):
    characters = string.ascii_letters + string.digits + string.punctuation
    return ''.join(random.choice(characters) for i in range(length))

# Load backup data from JSON file
def load_backup_data():
    with open("dump_gitlab_data.json", "r", encoding="utf-8") as f:
        return json.load(f)

# Create user
def create_user(username, email, password):
    url = f"{NEW_GITLAB_URL}/api/v4/users"
    data = {
        "username": username,
        "name": username,
        "email": email,
        "password": password,
        "skip_confirmation": True,
        "force_random_password": True  # Force user to change password on first login
    }
    try:
        response = requests.post(url, headers=HEADERS, data=data)
        response.raise_for_status()
        logger.info(f"User {username} created successfully.")
        return response.json()  # Return JSON with user info
    except requests.RequestException as e:
        logger.error(f"Error creating user {username}: {e}")
        return None

# Create project
def create_project(group_id, project_name):
    url = f"{NEW_GITLAB_URL}/api/v4/projects"
    data = {
        "name": project_name,
        "namespace_id": group_id,  # Put project in specified group
        "visibility": "private",
        "initialize_with_readme": False  # Create empty project by default
    }
    try:
        response = requests.post(url, headers=HEADERS, data=data)
        response.raise_for_status()
        logger.info(f"Project {project_name} created successfully in group {group_id}.")
        return response.json()  # Return created project data
    except requests.RequestException as e:
        logger.error(f"Error creating project {project_name} in group {group_id}: {e}")
        return None

# Create group
def create_group(group_name):
    url = f"{NEW_GITLAB_URL}/api/v4/groups"
    data = {
        "name": group_name,
        "path": group_name.lower().replace(" ", "_")  # Auto generate path
    }
    try:
        response = requests.post(url, headers=HEADERS, data=data)
        response.raise_for_status()
        logger.info(f"Group {group_name} created successfully.")
        return response.json()  # Return JSON with group info
    except requests.RequestException as e:
        logger.error(f"Error creating group {group_name}: {e}")
        return None

# Add user to group and set permission to master
def add_user_to_group(group_id, user_id, access_level=40):
    url = f"{NEW_GITLAB_URL}/api/v4/groups/{group_id}/members"
    data = {
        "user_id": user_id,
        "access_level": access_level  # Set permission to master (40)
    }
    try:
        response = requests.post(url, headers=HEADERS, data=data)
        response.raise_for_status()
        logger.info(f"User {user_id} added to group {group_id} with master access.")
    except requests.RequestException as e:
        logger.error(f"Error adding user {user_id} to group {group_id}: {e}")

# Import users, groups, and projects
def import_data():
    logger.add("import_gitlab_entities.log", rotation="500 MB", compression="zip")

    # Load backup data
    backup_data = load_backup_data()

    # Store username, email, password
    user_passwords = []

    # Create users
    for user in backup_data["users"]:
        username = user["username"]
        email = user["email"]
        password = generate_random_password()

        user_data = create_user(username, email, password)
        if user_data:
            user_id = user_data["id"]
            user_passwords.append({"username": username, "email": email, "password": password})

            # Create user's projects
            for project in user["projects"]:
                project_name = project["name"]
                # Find project's group
                group_name = project["namespace"]["name"]
                # When creating project, need group ID, first get group's ID
                group_id = None
                for group in backup_data["groups"]:
                    if group["name"] == group_name:
                        group_id = create_group(group_name)["id"]
                        break
                if group_id:
                    create_project(group_id, project_name)
                    # After creating project, add user to group
                    add_user_to_group(group_id, user_id)

    # Create groups and projects
    for group in backup_data["groups"]:
        group_name = group["name"]
        group_data = create_group(group_name)
        if group_data:
            group_id = group_data["id"]
            # Create projects in group
            for project in group["projects"]:
                project_name = project["name"]
                create_project(group_id, project_name)

            # Add corresponding users to group, set to master permission
            for group_user in group["users"]:
                add_user_to_group(group_id, group_user["user_id"])

    # Save username, email, password to JSON file
    with open("user_passwords.json", "w", encoding="utf-8") as f:
        json.dump(user_passwords, f, ensure_ascii=False, indent=4)

    logger.info("User passwords have been exported to user_passwords.json")

if __name__ == "__main__":
    import_data()
```

## Push Git Bare Repositories

```python
import os
import subprocess
import json
from loguru import logger

# Load config JSON file
def load_config(file_path):
    with open(file_path, 'r', encoding='utf-8') as f:
        return json.load(f)

# Write operation results to JSON file
def write_result_to_json(results, output_file):
    with open(output_file, 'w+', encoding='utf-8') as f:
        json.dump(results, f, indent=4, ensure_ascii=False)

# Push bare repo to new remote
def push_bare_repo_to_remote(project, results):
    absolute_path = project["absolute_path"]
    new_ssh_url_to_repo = project["new_ssh_url_to_repo"]

    # Check if repo path is valid
    if not os.path.isdir(absolute_path) or not os.path.exists(os.path.join(absolute_path, "HEAD")):
        logger.error(f"Invalid bare repository path: {absolute_path}")
        results.append({
            "project": project,
            "status": "failed",
            "message": f"Invalid bare repository path: {absolute_path}"
        })
        return

    try:
        # Enter original repo directory
        logger.info(f"Accessing local bare repository: {absolute_path}")
        os.chdir(absolute_path)

        # Ensure it's bare repo
        subprocess.run(["git", "rev-parse", "--is-bare-repository"], check=True)

        # Set new remote URL
        logger.info(f"Setting new remote URL: {new_ssh_url_to_repo}")
        subprocess.run(["git", "remote", "add", "new-origin", new_ssh_url_to_repo], check=True)

        # Push all branches to new remote
        logger.info("Pushing all branches to the new remote repository")
        subprocess.run(["git", "push", "new-origin", "--all","-f"], check=True)

        # Push all tags to new remote
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
        # Remove new remote config to keep original repo clean
        try:
            subprocess.run(["git", "remote", "remove", "new-origin"], check=True)
        except subprocess.CalledProcessError:
            logger.warning("Failed to remove new-origin remote, please check manually.")

# Main function
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
