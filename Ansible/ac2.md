# Lesson: Build, Share and Install Your Own Ansible Collection

**Level:** Beginner | **Time:** about 60 minutes | **You need:** a laptop, internet, a GitHub account

---

## What you will learn

By the end of this lesson you will be able to:

1. Explain what an Ansible collection is
2. Create your own collection
3. Store it on GitHub
4. Install and use it on any computer
5. Release a new version

---

## Part 0: The big idea (5 minutes)

### What is a collection?

Imagine a **toolbox**. Inside the toolbox you keep a hammer, a screwdriver and a tape measure. You can hand the whole toolbox to a friend.

An **Ansible collection** is a toolbox for automation. Inside it you can keep:

| Tool in the box | Real name | What it does |
|-----------------|-----------|--------------|
| A recipe of steps | **Role** | Does a job, like installing a web server |
| A small program | **Module** | Does one action, like creating a user |
| A text helper | **Plugin / filter** | Changes data, like making text uppercase |

### Why do we need it?

Without collections, you copy and paste files between projects. With a collection you **write once, share and reuse**, and every version is labelled so nothing breaks by surprise.

### The name pattern

Every collection has a two-part name: `namespace.name`

- `namespace` = who owns it (like your username)
- `name` = what it is called

Example: `myname.sample`

---

## Part 1: Set up your computer (10 minutes)

### Step 1: Check Python

```bash
python3 --version
```

You should see Python 3.9 or newer.

### Step 2: Install Ansible

```bash
pip install ansible-core
```

### Step 3: Check it worked

```bash
ansible --version
```

**Checkpoint:** you see a version number. If not, ask your teacher.

### Step 4: Install Git

```bash
git --version
```

If it is missing, install it from https://git-scm.com.

---

## Part 2: Create the collection (15 minutes)

### Step 5: Make a project folder

```bash
mkdir ansible-collection-sample
cd ansible-collection-sample
git init
```

**Why?** This folder will become your GitHub repository. `git init` tells Git to start tracking changes.

### Step 6: Generate the collection skeleton

```bash
ansible-galaxy collection init myname.sample --init-path /tmp/scaffold
cp -r /tmp/scaffold/myname/sample/. .
```

**Why?** Ansible creates the correct folders for us, so we do not have to remember them.

> Replace `myname` with your own name or GitHub username (small letters, no spaces).

### Step 7: Look at what was created

```bash
ls
```

You will see:

```
galaxy.yml   README.md   meta/   plugins/   roles/   docs/
```

| Item | Purpose |
|------|---------|
| `galaxy.yml` | The ID card of the collection (name, version, author) |
| `roles/` | Where roles live |
| `plugins/` | Where modules and filters live |
| `README.md` | Instructions for other people |

### Step 8: Fill in the ID card

Open `galaxy.yml` and change it to:

```yaml
namespace: myname
name: sample
version: 1.0.0
readme: README.md
authors:
  - Your Name <you@example.com>
description: My first collection
license:
  - GPL-3.0-or-later
repository: https://github.com/YOUR-USERNAME/ansible-collection-sample
build_ignore:
  - .git
  - "*.tar.gz"
```

### Step 9: Add a role (the recipe)

```bash
mkdir -p roles/hello/tasks roles/hello/defaults
```

Create `roles/hello/tasks/main.yml`:

```yaml
---
- name: Print hello message
  ansible.builtin.debug:
    msg: "{{ hello_message }}"
```

Create `roles/hello/defaults/main.yml`:

```yaml
---
hello_message: "Hello from my collection!"
```

**Explain to yourself:** the task prints a message. The message text comes from a variable, so users can change it later.

### Step 10: Add a module (the small program)

Create `plugins/modules/greet.py`:

```python
#!/usr/bin/python
from __future__ import absolute_import, division, print_function
__metaclass__ = type

DOCUMENTATION = r'''
---
module: greet
short_description: Returns a greeting
version_added: "1.0.0"
description:
  - Returns a greeting for the given name.
options:
  name:
    description: Name to greet.
    type: str
    required: true
author:
  - Your Name (@yourhandle)
'''

EXAMPLES = r'''
- myname.sample.greet:
    name: World
'''

RETURN = r'''
message:
  description: The greeting text.
  type: str
  returned: always
'''

from ansible.module_utils.basic import AnsibleModule


def main():
    module = AnsibleModule(
        argument_spec=dict(name=dict(type="str", required=True)),
        supports_check_mode=True,
    )
    module.exit_json(changed=False, message="Hello, %s!" % module.params["name"])


if __name__ == "__main__":
    main()
```

**Explain to yourself:** the module receives a `name` and sends back `Hello, <name>!`.

### Step 11: Add a `.gitignore`

Create a file named `.gitignore`:

```
*.tar.gz
__pycache__/
*.pyc
```

**Why?** It tells Git to ignore build files we do not want to upload.

**Checkpoint:** you now have a role and a module inside one collection.

---

## Part 3: Test it on your own computer (10 minutes)

### Step 12: Build the package

```bash
ansible-galaxy collection build
```

You get a file like `myname-sample-1.0.0.tar.gz`. This is your collection packed like a zip file.

### Step 13: Install it locally

```bash
ansible-galaxy collection install myname-sample-1.0.0.tar.gz --force
```

### Step 14: Confirm it is installed

```bash
ansible-galaxy collection list myname.sample
```

### Step 15: Write a test playbook

Create `test.yml`:

```yaml
---
- name: Test my collection
  hosts: localhost
  gather_facts: false
  tasks:
    - name: Run the role
      ansible.builtin.include_role:
        name: myname.sample.hello

    - name: Call the module
      myname.sample.greet:
        name: Student
      register: result

    - name: Show the result
      ansible.builtin.debug:
        msg: "{{ result.message }}"
```

### Step 16: Run it

```bash
ansible-playbook test.yml
```

**Expected output:** you see `Hello from my collection!` and `Hello, Student!`.

**Checkpoint:** if you see both messages, your collection works!

---

## Part 4: Put it on GitHub (10 minutes)

### Step 17: Create an empty repository

1. Go to https://github.com and click **New repository**
2. Name it `ansible-collection-sample`
3. Leave it **empty** (do not add a README)
4. Click **Create repository**

### Step 18: Upload your code

```bash
git add .
git commit -m "My first collection"
git branch -M main
git remote add origin https://github.com/YOUR-USERNAME/ansible-collection-sample.git
git push -u origin main
```

**Explain to yourself:**

| Command | Meaning |
|---------|---------|
| `git add .` | Choose all files to save |
| `git commit` | Save a snapshot with a message |
| `git remote add` | Connect to the GitHub repository |
| `git push` | Upload to GitHub |

**Checkpoint:** refresh the GitHub page. You should see your files.

---

## Part 5: Install it from GitHub (5 minutes)

This is the step that proves sharing works. Any student can do this on any computer.

```bash
ansible-galaxy collection install git+https://github.com/YOUR-USERNAME/ansible-collection-sample.git,main --force
```

Run the test playbook again:

```bash
ansible-playbook test.yml
```

**Challenge:** ask a classmate to install your collection using your GitHub link and run their own playbook.

---

## Part 6: Release a new version (5 minutes)

### Step 19: Make a change

Edit `roles/hello/defaults/main.yml`:

```yaml
hello_message: "Hello from version 1.0.1!"
```

### Step 20: Update the version

In `galaxy.yml` change:

```yaml
version: 1.0.1
```

### Step 21: Save and upload

```bash
git add .
git commit -m "Release 1.0.1"
git push
```

### Step 22: Tag the release

```bash
git tag v1.0.1
git push origin v1.0.1
```

**Why tag?** A tag is a permanent bookmark. Users can install exactly this version:

```bash
ansible-galaxy collection install git+https://github.com/YOUR-USERNAME/ansible-collection-sample.git,v1.0.1 --force
```

### Version numbers explained

| Change | Example |
|--------|---------|
| Small bug fix | 1.0.0 to 1.0.1 |
| New feature (still compatible) | 1.0.0 to 1.1.0 |
| Big change that may break things | 1.0.0 to 2.0.0 |

---

## Summary: the whole journey in one picture

```
Create folder  ->  Generate skeleton  ->  Add role + module
       |
       v
Build + test locally  ->  Push to GitHub  ->  Tag a version
       |
       v
Anyone installs from the GitHub link
```

---

## Practice exercises

1. **Easy:** change `hello_message` and run the playbook again.
2. **Easy:** greet your own name using the `greet` module.
3. **Medium:** add a second role named `goodbye` that prints a goodbye message.
4. **Medium:** release version `1.1.0` and install it using its tag.
5. **Hard:** add a filter plugin that turns text into uppercase (`plugins/filter/shout.py`).

---

## Quick quiz

1. What does `galaxy.yml` contain?
2. What is the difference between a role and a module?
3. Why do we use version numbers?
4. Which command packs the collection into a `.tar.gz` file?
5. How do you install a specific version from GitHub?

**Answers:**
1. The collection's name, version, author and description.
2. A role is a set of steps. A module is a small program that does one action.
3. So users can pick a stable version and updates do not break their work.
4. `ansible-galaxy collection build`
5. Add the tag after the repo URL, for example `...git,v1.0.1`.

---

## Troubleshooting for students

| Problem | Fix |
|---------|-----|
| `command not found: ansible` | Run `pip install ansible-core` again, restart the terminal |
| `galaxy.yml not found` | You are in the wrong folder, or the file is not at the repo root |
| Module not found | Use the full name: `myname.sample.greet` |
| Install shows old code | Add `--force` to the install command |
| `git push` asks for a password | Use a GitHub personal access token, or set up an SSH key |

---

## Teacher notes

- Give each student a unique namespace (their GitHub username) to avoid mix-ups.
- Do Part 3 live as a demo before students try it themselves.
- Part 5 works well as a pair activity: students install each other's collections.
- Extra time? Add GitHub Actions to build the collection automatically on every tag.
