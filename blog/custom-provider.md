---
title: "Mastering Terraform: Create Custom Providers and Take Control 🚀"
description: "(Because Terraform isn't just a tool - it's a challenge waiting to be conquered!)"
category: "terraform"
image: "/custom-provider.png"
date: "January 05, 2025"
author: "Yogesh Patil"
---


Are you ready to push Terraform beyond its boundaries? Let's explore the fascinating world of custom providers and unlock new automation possibilities. Here's my story of curiosity, discovery, and late-night coding chaos. By the end, you'll know how to build your own Terraform provider from scratch and apply it to unconventional resources. Ready? Let's dive in!

## Why I Wrote This Blog

Ever wondered, "What if I could create my own Terraform resource?" That question hit me mid-Terraform-module-writing-marathon. It wasn't long before curiosity dragged me into the rabbit hole of custom providers. Spoiler: It was equal parts thrilling and mildly chaotic.

I mean, Terraform is already a magical tool, but making your own resource? That's like upgrading from wizard robes to a full-blown superhero cape. This blog isn't a boring how-to; it's a fun walkthrough of how I stumbled, learned, and finally created my first custom provider.

So, grab your coffee (or energy drink of choice) and join me on this journey. Who knows? By the end, you might feel like a Terraform sorcerer ready to bend the cloud to your will. ✨

## Why To-Do with Custom Provider?

*(Or: How I Ended Up Building My Own Terraform Toy Instead of Finding One)*

Every learner's first hands-on project with a new technology often starts with something simple yet functional - like a to-do list. It's a rite of passage, a way to break the ice and understand the basics. Why should Terraform custom providers be any different?

Imagine managing your to-do list with Terraform - creating tasks as resources, updating their status, or even deleting them when they're done. It might sound unconventional, but it's the perfect way to get your hands dirty with custom providers.

Starting with a to-do list demonstrates not only how Terraform can extend beyond traditional infrastructure but also how you can creatively adapt it to manage any resource you dream of. It's the kind of project that transforms theory into practice and turns curiosity into mastery.

By the end of this journey, you'll have your to-do list managed by Terraform - and more importantly, you'll have the confidence to create custom providers for whatever resource you want to automate next. After all, every great automation starts with a single "to-do."

## How to Build a Custom Terraform Provider (Step-by-Step)

Let's jump into the nitty-gritty. Here's how I created a provider to manage a todo list stored in a JSON file.

### Step 0: Install Prerequisites

*(Because We're Not Mind Readers)*

Before we begin, ensure you have the following installed:

- Terraform
- Go (Golang): (version > 1.21.0)
- Your favorite text editor or IDE (e.g., VS Code)

**Pro Tip:** Use a version manager like `tfenv` for Terraform to easily switch between versions during testing.

### Step 1: Project Setup

*(The Boring But Necessary Part)*

Let's set up our project structure. Think of it like arranging LEGO blocks before building a masterpiece.

```bash
mkdir terraform-provider-todo  
cd terraform-provider-todo  
go mod init terraform-provider-todo  
mkdir -p internal/provider  
touch internal/provider/{provider,resource_todo}.go  
touch main.go
```

### Step 2: The Main Entry Point (main.go)

This file is the heart of our provider. It uses Terraform's Plugin SDK to serve our custom provider.

```go
package main

import (
    "github.com/hashicorp/terraform-plugin-sdk/v2/plugin"
    "terraform-provider-todo/internal/provider"
)

func main() {
    plugin.Serve(&plugin.ServeOpts{
        ProviderFunc: provider.Provider})
}
```

### Step 3: Provider Definition (internal/provider/provider.go)

Now, let's define the provider itself. This is where we specify what our provider will manage and configure the resources for Terraform to work with.

```go
package provider

import (
    "github.com/hashicorp/terraform-plugin-sdk/v2/helper/schema"
)

type providerConfig struct {
    StoragePath string
}

func Provider() *schema.Provider {
    return &schema.Provider{
        Schema: map[string]*schema.Schema{
            "storage_path": {
                Type:     schema.TypeString,
                Required: true,
            },
        },
        ResourcesMap: map[string]*schema.Resource{
            "yogesh_todo_item": resourceTodoItem(),
        },
        ConfigureFunc: providerConfigure,
    }
}

func providerConfigure(d *schema.ResourceData) (interface{}, error) {
    config := providerConfig{
        StoragePath: d.Get("storage_path").(string),
    }
    return &config, nil
}
```

### Step 4: The Todo Resource (internal/provider/resource_todo.go)

Now, let's get into the meat of it. This step is where we define the resource we want to manage, which in this case is our trusty `todo_item`. Think of this like building a small module that will allow Terraform to manipulate your todo list.

```go
package provider

import (
 "encoding/json"
 "fmt"
 "os"
 "time"
 "github.com/gofrs/flock"
 "github.com/google/uuid"
 "github.com/hashicorp/terraform-plugin-sdk/v2/helper/schema"
)

type TodoItem struct {
 ID        string    `json:"id"`
 Title     string    `json:"title"`
 Completed bool      `json:"completed"`
 CreatedAt time.Time `json:"created_at"`
}

type TodoStorage struct {
 Items []TodoItem `json:"items"`
}

func resourceTodoItem() *schema.Resource {
 return &schema.Resource{
  Create: resourceTodoItemCreate,
  Read:   resourceTodoItemRead,
  Update: resourceTodoItemUpdate,
  Delete: resourceTodoItemDelete,

  Schema: map[string]*schema.Schema{
   "title": {
    Type:     schema.TypeString,
    Required: true,
   },
   "completed": {
    Type:     schema.TypeBool,
    Optional: true,
    Default:  false,
   },
   "created_at": {
    Type:     schema.TypeString,
    Computed: true,
   },
  },
 }
}

// Helper function to load the storage with file locking
func loadStorageWithLock(path string) (*TodoStorage, error) {
 lock := flock.New(path + ".lock")
 defer lock.Unlock()

 if err := lock.Lock(); err != nil {
  return nil, fmt.Errorf("failed to acquire file lock: %w", err)
 }

 if _, err := os.Stat(path); os.IsNotExist(err) {
  return &TodoStorage{Items: []TodoItem{}}, nil
 }

 data, err := os.ReadFile(path)
 if err != nil {
  return nil, err
 }

 var storage TodoStorage
 if err := json.Unmarshal(data, &storage); err != nil {
  return nil, err
 }

 return &storage, nil
}

// Save the storage with updated data
func saveStorage(storage *TodoStorage, path string) error {
 data, err := json.MarshalIndent(storage, "", "  ")
 if err != nil {
  return err
 }

 return os.WriteFile(path, data, 0600)
}

func resourceTodoItemCreate(d *schema.ResourceData, m interface{}) error {
 config := m.(*providerConfig)
 storagePath := config.StoragePath

 if storagePath == "" {
  return fmt.Errorf("storage path is not configured")
 }

 storage, err := loadStorageWithLock(storagePath)
 if err != nil {
  return fmt.Errorf("failed to load storage: %w", err)
 }

 item := TodoItem{
  ID:        uuid.New().String(), // Using UUID for guaranteed unique ID
  Title:     d.Get("title").(string),
  Completed: d.Get("completed").(bool),
  CreatedAt: time.Now(),
 }

 storage.Items = append(storage.Items, item)
 if err := saveStorage(storage, storagePath); err != nil {
  return err
 }

 d.SetId(item.ID)
 d.Set("created_at", item.CreatedAt.Format(time.RFC3339))
 return nil
}

func resourceTodoItemRead(d *schema.ResourceData, m interface{}) error {
 config := m.(*providerConfig)
 storagePath := config.StoragePath

 if storagePath == "" {
  return fmt.Errorf("storage path is not configured")
 }

 storage, err := loadStorageWithLock(storagePath)
 if err != nil {
  return fmt.Errorf("failed to load storage: %w", err)
 }

 for _, item := range storage.Items {
  if item.ID == d.Id() {
   d.Set("title", item.Title)
   d.Set("completed", item.Completed)
   d.Set("created_at", item.CreatedAt.Format(time.RFC3339))
   return nil
  }
 }

 // Returning a more user-friendly error message
 d.SetId("")
 return fmt.Errorf("todo item with ID '%s' not found", d.Id())
}

func resourceTodoItemUpdate(d *schema.ResourceData, m interface{}) error {
 config := m.(*providerConfig)
 storagePath := config.StoragePath

 if storagePath == "" {
  return fmt.Errorf("storage path is not configured")
 }

 storage, err := loadStorageWithLock(storagePath)
 if err != nil {
  return fmt.Errorf("failed to load storage: %w", err)
 }

 // We only need the index, so item is not needed
 _, index, err := findTodoItemByID(storage, d.Id())
 if err != nil {
  return err
 }

 storage.Items[index].Title = d.Get("title").(string)
 storage.Items[index].Completed = d.Get("completed").(bool)

 if err := saveStorage(storage, storagePath); err != nil {
  return err
 }

 return nil
}

func resourceTodoItemDelete(d *schema.ResourceData, m interface{}) error {
 config := m.(*providerConfig)
 storagePath := config.StoragePath

 if storagePath == "" {
  return fmt.Errorf("storage path is not configured")
 }

 storage, err := loadStorageWithLock(storagePath)
 if err != nil {
  return fmt.Errorf("failed to load storage: %w", err)
 }

 // We only need the index, so item is not needed
 _, index, err := findTodoItemByID(storage, d.Id())
 if err != nil {
  return err
 }

 storage.Items = append(storage.Items[:index], storage.Items[index+1:]...)
 if err := saveStorage(storage, storagePath); err != nil {
  return err
 }

 return nil
}

// Helper function to find TodoItem by ID
func findTodoItemByID(storage *TodoStorage, id string) (*TodoItem, int, error) {
 for i, item := range storage.Items {
  if item.ID == id {
   return &item, i, nil
  }
 }
 return nil, -1, fmt.Errorf("todo item with ID '%s' not found", id)
}
```

### Step 5: Build and Install the Provider

Now that we've got our provider and resource definitions in place, it's time to build the provider and install it locally. This is like assembling your custom tool and setting it up for use, ready to manage your todo list.

```bash
go get github.com/google/uuid
go get github.com/gofrs/flock
go mod tidy
go build -o terraform-provider-todo

# Create a local plugin directory where Terraform will look for the provider
mkdir -p ~/.terraform.d/plugins/local.providers/local/todo/1.0.0/linux_amd64

# Copy the built provider to the local plugin directory
cp terraform-provider-todo ~/.terraform.d/plugins/local.providers/local/todo/1.0.0/linux_amd64/
```

### Step 6: Use Your Shiny New Provider!

The moment of truth has arrived! It's time to use the custom provider you've just created. Let's put it to the test and see it in action.

#### i) Create a new directory for your Terraform configuration:

```bash
mkdir todo-project
cd todo-project
```

#### ii) Create provider.tf:

In this file, we define our provider configuration:

```hcl
terraform {
  required_providers {
    todo = {
      source = "local.providers/local/todo"
      version = "1.0.0"
    }
  }
}

provider "todo" {
  storage_path = "todos.json"
}
```

This tells Terraform where to find our custom provider and the version we're using. Also, we specify the storage path for our todo list (which will be a JSON file).

#### iii) Create main.tf:

Here, we'll define a couple of todo items to manage:

```hcl
resource "yogesh_todo_item" "eternal_deployment" {
  title     = "Tackle that Terraform task my manager assigned months ago, because 'It'll be quick, trust me' 🤔"
  completed = false
}

resource "yogesh_todo_item" "blog_writing" {
  title     = "Write a blog about custom providers because my manager casually said, 'Do something out of the box,' and I thought... why not? 🧠💡"
  completed = true
}

# For those extra sprint items that keep coming
resource "yogesh_todo_item" "reality_check" {
  title     = "Remind manager that 'quick 5-minute fixes' actually take 5 hours"
  completed = false
}

# The classic one
resource "yogesh_todo_item" "documentation" {
  title     = "Update documentation (Just kidding, we all know this will never be completed)"
  completed = false
}
```

Now you can run:

```bash
terraform init
terraform plan
terraform apply -parallelism=1
```

**Note:** My app is not yet optimized to handle parallel resources, so please bear with me while I work on improving this.

And voilà! 🎉 You've just created a working Terraform provider that manages todo items in a JSON file. Is it practical? Maybe not. But is it cool? Absolutely! 😎

## What Did We Just Build?

*(Besides a Way to Procrastinate with Infrastructure as Code)*

Our todo provider:

- Stores todos in a JSON file (because who doesn't love JSON?)
- Supports creating, reading, updating, and deleting todos
- Tracks completion status
- Automatically sets creation timestamps
- Makes you feel like a DevOps superhero

Sure, it's not running critical infrastructure, but hey - it's yours, and it works!

## Common Pitfalls

*(Or: Things That Will Test Your Patience)*

- **State Conflicts:** Directly modify the JSON file, and Terraform will throw a tantrum akin to someone rearranging your perfectly organized desk. Not fun.
- **Concurrent Access:** This example doesn't handle multiple Terraform runs accessing the file simultaneously. Debugging this will make you question life choices at 2 AM.
- **Error Handling:** Let's just say our current approach to errors is… "minimalist." Definitely room for improvement here.

## Pro Tips

*(That I Learned the Hard Way)*

- **Testing is a Must:** Write tests. Future-you will thank present-you.
- **Document Everything:** Pretend you're writing for someone who thinks Terraform is a new Marvel superhero.
- **Versioning:** Breaking changes without versioning is like showing up to a meeting unprepared - awkward for everyone.

## Conclusion: From "Can I?" to "I Did!" 🎉

Congratulations - you've just built a custom Terraform provider! It might not manage the world's infrastructure, but it's uniquely yours, and that's what matters.

Let me share a little secret: this is both my strength and my weakness. When the question "Can I?" arises, I don't look for a syllabus or roadmap to follow. I dive in headfirst, knowing that the deepest learning often comes from rolling up my sleeves and getting my hands dirty. Sure, there are plenty of ways to gain knowledge, but when it comes to real impact, I find that experience teaches faster and with far greater clarity than any course could.

So, what's next? Maybe integrate a real API, add cool features like due dates, or even take a crack at automating your coffee machine. The sky's the limit when curiosity leads the way!

Remember: with great power comes great responsibility… and an irresistible urge to automate everything.

Happy terraforming, and if anyone asks why you built this, just smile and say, "Because I could!" 🚀

Found this helpful? Share your custom provider adventures in the comments!