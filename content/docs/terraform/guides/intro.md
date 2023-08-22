---
title: "Intro"
date: 2023-08-21T20:45:00+02:00
draft: false
icon: "code_blocks"
weight: 100

---

# Goal

A minimalist approach to how Terraform works. Useless for anyone who already knows how to use Terraform. Usefull to understand a few key concepts.
<br>
The goal is to manipulate local files. I've choose a local environment because it allows to understand  that Terraform isn't juste a cloud provider tool.

## Prerequisites
- Terraform CLI

## Setup 
### Structure
Create a `terraform` folder, and inside this folder create a     ` main.tf` file. 

```bash.
└── terraform
   └── main.tf
```
For now, main.tf will contain all of our code. We'll split it later.
