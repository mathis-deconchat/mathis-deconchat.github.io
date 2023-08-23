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

{{< alert context="info" text="For absolute beginner with Terrafomr." />}}


## Prerequisites
- Terraform CLI

## Setup 
### Structure
Create a `terraform` folder, and inside this folder create a     ` main.tf` file. 

```bash
└── terraform
   └── main.tf
```
For now, main.tf will contain all of our code. We'll split it later.

### Content
Inside `main.tf` add the following code:

```hcl
resource "local_file" "myfile" {
  content = "Hey World ! 
  filename = "foo.txt"
}
```
The `local_file` first paramater of the resource keyword indicates the type of resource we want to create. It refers to a provider. This time it's a built-in provider called `local` but it could also be an external provider.

The second parameter is the name of the resource for Terraform. It's used to reference it later in the code.

Now, we need to init the project for 2 reasosn :  
1. We must create a state of our infrastructure, by default it will be local
2. We need to download or load the appropriate provider

But  before, let make our life easier and create an alias for teh terraform command. Add the following line to your `.bashrc` or `.zshrc` file.

```bash
alias tf="terraform"
```
And then reload your shell with `source ~/.bashrc` or `source ~/.zshrc`

### Init and Providers

Then, init the project with 
```bash 
tf init
```


 You should see something like this:

```bash
Initializing the backend...
Initializing provider plugins...
# [ other things]
Terraform has been successfully initialized!
``` 

Now this is what your directory should look like:

```bash
.
└── terraform
   ├── .terraform
   ├── .terraform.lock.hcl
   └── main.tf
```

If you look deep inside `.terraform` you'll see a `providers` directory. This is where Terraform stores the providers you use.

## Main
### Plan
Now we need to plan the huge change to the infrastructure we are about to make. (Create a text file)
  
  ```bash
  tf plan
  ```

  You should see something like this:

  ```bash
    # local_file.myfile will be created
  + resource "local_file" "myfile" {
      + content              = "Hey World ! "
      + content_base64sha256 = (known after apply)
      + content_base64sha512 = (known after apply)
      + content_md5          = (known after apply)
      + content_sha1         = (known after apply)
      + content_sha256       = (known after apply)
      + content_sha512       = (known after apply)
      + directory_permission = "0777"
      + file_permission      = "0777"
      + filename             = "foo.txt"
      + id                   = (known after apply)
    }

Plan: 1 to add, 0 to change, 0 to destroy.
``` 
We see that it plan to add 1 resource, and that it will create a file called `foo.txt` with the content `Hey World !`.

### Apply
Now we can apply the changes to our infrastructure. (Create the file)

```bash
tf apply
```
Enter `Yes` when prompted to confirm the changes. We will fix this later.

You should see something like this:

```bash
local_file.myfile: Creating...
local_file.myfile: Creation complete after 0s [id=5b2e1c9a53d878a4dfb62b02fee65c39093991b1]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
``` 

Now here is what your directory should look like:

```bash
.
└── terraform
   ├── .terraform
   ├── .terraform.lock.hcl
   ├── foo.txt
   ├── main.tf
   ├── terraform.tfstate
   └── terraform.tfstate.backup
```
We see the foo.txt bu also 2 new files. `terraform.tfstate` and `terraform.tfstate.backup`. These files are used to store the state of the infrastructure. It's used to know what resources are present, and what are their parameters.

Since in our state, a file called `foo.txt` exists, Terraform won't try to create it again., you can try and re-run `tf apply` and you'll see that nothing happens and get this response : 
  
```bash
  Apply complete! Resources: 0 added, 0 changed, 0 destroyed.
```


### Destroy 
Now we can destroy the infrastructure we created. (Delete the file)
And remove the file existence from the state.

```bash
tf destroy
```

you should see something like this:

```bash
local_file.myfile: Destroying... [id=5b2e1c9a53d878a4dfb62b02fee65c39093991b1]
local_file.myfile: Destruction complete after 0s

Destroy complete! Resources: 1 destroyed.

```
### Ressource reference
You can reference a resource in the code anywhere. For example, we can create a new resource that will use the content of the file we created earlier.
Add this code to `main.tf`
```hcl
resource "local_file" "myfilereference" {
  content = local_file.myfile.content
  filename = "bar.txt"
}
```
Then run `tf plan` and `tf apply` and you'll see that a new file called `bar.txt` has been created with the same content as `foo.txt`.
The order in the code has no importance. Beforehand, teraform will create a graph of the resources and their dependencies, and then apply the changes in the right order.


### Data 
You can also use data in your code. For example, we can use the `local_file` data source to get the content of a file. Add this code to `main.tf`

Let's create a file called `terra.txt` with the content "Terraform is awesome !"
```bash 
echo "Terraform is awesome !" > terra.txt

```
And in `main.tf` add this code:
```hcl
data "local_file" "mydata" {
  filename = "terra.txt"
}
``` 
  
  Then we can use the data in a resource. Add this code to `main.tf`
  
  ```hcl
  resource "local_file" "mydataresource" {
    content = data.local_file.mydata.content
    filename = "foobar.txt"
  }
  ```
  
  Then run `tf plan` and `tf apply` and you'll see that a new file called `foobar.txt` has been created with the same content as `terra.txt`.

At the moment `main.tf` should look like this:

```hcl
resource "local_file" "myfile" {
  content = "Hey World ! "
  filename = "foo.txt"
}

resource "local_file" "myfilereference" {
  content = local_file.myfile.content
  filename = "bar.txt"
}

data "local_file" "mydata" {
  filename = "terra.txt"
}

resource "local_file" "mydataresource" {
  content = data.local_file.mydata.content
  filename = "foobar.txt"
}
```
Every block could be in a separate file, but for now, we'll keep it like this. It won't change anything.


## Variables
One of the keyconcept of Terraform is the use of variables. It allows to use the same code for different environments. For example, you can use the same code to create a VM on AWS or on GCP. You just need to change the variables. It's even more powerfull with Terragrunt and modules.

### Variable declaration
Let's create a variable called `myvar` with the value `Hello World !`. Add this code to `main.tf`

```hcl
variable "myvar" {
  type = string
  default = "Hello World !"
}
```
Then we can use it in a resource. Add this code to `main.tf`

```hcl
resource "local_file" "myvarresource" {
  content = var.myvar
  filename = "var.txt"
}
```
Then run `tf plan` and `tf apply` and you'll see that a new file called `var.txt` has been created with the same content as `myvar`.

But that's not right way to do. 
### Variable files
Create two files in the `terraform` folder. `variables.tf` and `terraform.tfvars`


```bash
touch variables.tf terraform.tfvars
```

This

```bash
└── terraform
   ├── .terraform
   ├── .terraform.lock.hcl
   ├── main.tf
   ├── terraform.tfstate
   ├── terraform.tfstate.backup
   ├── terraform.tfvars
   └── variables.tf
```

`variables.tf` will be used to "declare" variables. And `terraform.tfvars` will be used to "assign" values to variables.

Add this code to `variables.tf`

```hcl
variable "file_content" {
  type = string
  default = ""
}
```

And this code to `terraform.tfvars`

```hcl
file_content = "Hello World !"
```

Then we can use it in a resource. Add this code to `main.tf`

```hcl
resource "local_file" "myvarresource" {
  content = var.file_content
  filename = "var.txt"
}
```
Then run `tf plan` and `tf apply` and you'll see that a new file called `var.txt` has been created with the same content as `file_content`.

# References
- [Terraform documentation](https://www.terraform.io/docs/language/index.html)