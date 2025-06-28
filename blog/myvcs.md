---
title: "I Built My Own Git to Understand How Git Actually Works"
description: "Building my own Git clone (MyVCS) to understand what really happens under the hood. A fun deep-dive into version control internals and the DevOps patterns they reveal. Learn by doing, debug by crying."
category: "vcs"
image: "/myvcs.png"
date: "June 21, 2025"
author: "Yogesh Patil"
---


# MyVCS: Building My Own Git-Like Time Machine (Because Curiosity Killed the Cat, But Satisfaction Brought It Back)

[Source Code](https://github.com/yogeshp-code/MyVCS)

## Introduction: Why Build a VCS When Git Exists? (Spoiler: It's Not About the Destination, It's About the Journey)

Picture this: You're a DevOps engineer, living and breathing `git add`, `git commit`, `git push` like they're the holy trinity of your daily existence. But somewhere between your 47th `git merge conflict` of the week and your 3rd coffee cup of the morning, a dangerous thought creeps in: *"What the heck is actually happening under the hood here?"*

Is Git some kind of digital wizardry? A portal to the code dimension? Or just really, really clever engineering wrapped in a deceptively simple command-line interface?

Well, being the curious (some might say masochistic) DevOps engineer that I am, I decided to find out the hard way. Instead of just Googling "how git works internally" like a normal person, I thought: *"Hey, let's build my own version control system from scratch! What could go wrong?"*

Spoiler alert: A lot could go wrong. But also, a lot could go amazingly right.

Meet **MyVCS** – my very own, slightly quirky, definitely educational time machine for code. It's like Git's younger sibling who's trying really hard to be cool but still asks you to tie their shoelaces. And honestly? I'm pretty proud of the little guy.

## The "Aha!" Moments: Discovering Git's Secret Sauce

Building MyVCS was like taking apart a Swiss watch with a butter knife – messy, occasionally painful, but ultimately revealing. Here are the core concepts that made me go "OH, THAT'S HOW IT WORKS!"

### Snapshots > Diffs (Mind = Blown)

First revelation: Git doesn't store differences between files. It stores complete snapshots of your entire project at each commit. It's like having a photo album where each picture shows your entire messy desk, not just the one pen you moved.

Why is this brilliant? Because retrieving any version is lightning fast. No need to apply 47 patches in sequence to figure out what your code looked like last Tuesday.

### The Holy Trinity: Blobs, Trees, and Commits (Not as Religious as It Sounds)

Git's object model is beautifully simple:

1. **Blobs**: Store file content. Just raw data, no frills, no metadata. If two files have identical content, they share the same blob. It's like having one Netflix account for the whole family – efficient and economical.

2. **Trees**: Represent directories. They're like the table of contents that tells you where everything lives. Trees can point to blobs (files) or other trees (subdirectories).

3. **Commits**: The time capsules. Each commit points to a tree (your project snapshot) and remembers its parent commits. It's your project's family tree, but with better documentation.

### Hashing: The Fingerprint System That Never Lies

Every object gets a unique SHA-1 hash based on its content. Change one character? Completely different hash. It's like having a bouncer who never forgets a face and can spot a fake ID from orbit.

## MyVCS in Action: The Commands That Make the Magic Happen

Enough theory! Let's see how these concepts translate into actual, working commands:

### `myvcs init`: Setting Up the Time Machine

```bash
myvcs init
```

This creates the `.myvcs/` directory – our secret lair where all the version control magic happens. Think of it as installing the flux capacitor in your DeLorean. Nothing fancy, but absolutely essential.

**What happens behind the scenes:** Creates the directory structure (`objects/`, `refs/`, etc.) and sets up the initial `HEAD` pointer. Your project is now time-travel ready!

### `myvcs add .`: Staging for Greatness

```bash
myvcs add .
```

This is where we tell MyVCS: "Hey, I've made some changes, and I think they're worth remembering!" It reads your files, creates blob objects, and updates the staging area (index).

**The magic:** Files with identical content share the same blob object. It's like having a really smart librarian who realizes you don't need two copies of the same book.

### `myvcs commit`: Freezing Time (The Big Moment)

```bash
myvcs commit -m "Fixed the bug that was definitely not my fault"
```

This is where the time travel actually happens! MyVCS creates a tree object representing your entire project structure, then wraps it in a commit object with metadata and a pointer to the previous commit.

**Pro tip:** Write good commit messages. Future you will either thank you or curse your name. Choose wisely.

### `myvcs checkout`: Time Travel Made Easy

```bash
myvcs checkout <commit-hash>
```

Want to see what your code looked like last week? Boom! MyVCS reads the commit's tree object and restores all files to their historical state. It's like having a time machine, but for code instead of preventing your parents from meeting.

**Warning:** May cause existential crisis when you realize how much "progress" you've actually made.

### `myvcs tag`: Marking Important Moments

```bash
myvcs tag v1.0
```

Tags are like bookmarks for important commits. "This is where we shipped version 1.0!" or "This is where everything worked perfectly before Dave touched it."

### `myvcs log`: The Paper Trail

```bash
myvcs log
```

Shows the complete history of your project. It's like reading your code's autobiography, complete with all the embarrassing chapters you'd rather forget.

**Sample output:**
```
commit a1b2c3d4e5f6...
Author: You
Date: Yesterday
Message: Actually fixed the bug this time

commit f6e5d4c3b2a1...
Author: You  
Date: Day before yesterday
Message: Fixed the bug (spoiler: didn't actually fix it)
```

## The Fun Parts (And the Hair-Pulling Moments)

### What Went Right
- The satisfaction of typing `myvcs init` and watching it actually work
- Realizing that Git's complexity comes from features, not fundamental concepts
- That moment when `myvcs checkout` successfully transported me to last week's code
- Understanding why merge conflicts happen (spoiler: it's not because Git hates you)

### What Made Me Question My Life Choices
- Debugging hash mismatches at 2 AM (turns out newline characters matter, who knew?)
- Realizing I'd been thinking about trees upside down (roots go up in Git-land)
- The moment I discovered that "simple" file permissions have approximately 47 edge cases

### The "Wait, That's It?" Moments
- Branches are just files containing commit hashes
- The entire `.myvcs/` directory is surprisingly small
- Most Git "magic" is actually just really good engineering

## DevOps Takeaways: More Than Just Code Lessons

Building MyVCS wasn't just about understanding Git – it revealed fundamental principles that apply across our entire DevOps ecosystem:

### 1. Immutability: The Foundation of Reliable Systems
Every MyVCS object is immutable – once created, never changed. This principle is gold in DevOps:
- **Infrastructure as Code**: Don't patch servers, replace them
- **Container Images**: Immutable layers make deployments predictable
- **Configuration Management**: Versioned configs prevent drift
- **Audit Trails**: Immutable logs can't be tampered with

### 2. Content-Addressable Storage: The Ultimate Deduplication
MyVCS stores objects by content hash, eliminating duplication. Apply this to:
- **Artifact Repositories**: Same binaries, single storage
- **Container Registries**: Shared layers across images
- **Backup Systems**: Deduplicated backups save massive storage
- **CDN Optimization**: Cache by content hash, not URL

### 3. Graph-Based Thinking: Understanding Complex Systems
Version control history is a directed acyclic graph. This mental model helps with:
- **Microservice Dependencies**: Visualize service interactions
- **CI/CD Pipelines**: Model parallel and sequential stages
- **Infrastructure Dependencies**: Understand deployment order
- **Incident Response**: Trace cascading failures

### 4. Staging Areas: The Power of Atomic Operations
The index teaches us about atomic changes:
- **Database Migrations**: Stage changes before applying
- **Infrastructure Updates**: Blue-green deployments
- **Feature Flags**: Staged rollouts
- **Code Reviews**: Focused, reviewable changes

### 5. Lightweight References: Efficiency Through Indirection
Branches and tags are just pointers. This concept scales to:
- **Load Balancing**: Point to healthy instances
- **Service Discovery**: References, not hardcoded endpoints
- **Configuration Management**: Environment-specific pointers
- **Deployment Strategies**: Symbolic links for active versions

## The Hard Truth About Learning: Sometimes You Have to Build It to Believe It

Here's something nobody tells you about learning complex systems: **reading documentation and watching tutorials can only take you so far**. 

I could have read every Git internals blog post on the internet, watched every conference talk, and memorized every Stack Overflow answer. But until I actually tried to build something myself, until I faced the frustrating reality of "wait, why isn't this working?" at 2 AM, until I experienced that incredible dopamine rush when `myvcs init` finally worked – the knowledge just wasn't *really* mine.

Yes, this approach will backfire on you. Yes, you'll spend hours debugging issues that "shouldn't happen." Yes, your colleagues will question your sanity when you explain why you're rebuilding something that already exists perfectly well.

But here's the magic: **when you finally get that one small feature working, when you see your code actually do what you intended, the satisfaction is unlike anything else.** It's not just dopamine – it's understanding that gets written to your brain's permanent storage with hash-based integrity and no delete permissions! 😂

The bugs you fix, the design decisions you struggle with, the "aha!" moments when concepts finally click – these experiences create a depth of understanding that no amount of reading can replicate. It's the difference between knowing about swimming and actually jumping in the pool.

This isn't always the most efficient path. Sometimes you should just use the existing, battle-tested solution. But when you really need to understand something deeply, when you want to innovate or optimize in that space, there's no substitute for building it yourself.

## Conclusion: The Journey Was Worth Every Bug

Building MyVCS transformed Git from a mysterious black box into a transparent, logical system. It's like finally understanding how your favorite magic trick works – less magical, but infinitely more impressive.

The core concepts – immutable objects, content addressing, graph-based history – aren't just version control principles. They're fundamental design patterns that appear everywhere in modern software systems, from Kubernetes to blockchain to distributed databases.

So next time you run `git commit`, take a moment to appreciate the elegant dance of blobs, trees, and commits happening behind the scenes. And remember: sometimes the best way to understand something is to try building it yourself, even if your version can't handle merge conflicts and occasionally panics when it encounters a symbolic link.

MyVCS might not replace Git anytime soon (thank goodness), but it gave me something more valuable: the confidence to peek under the hood of any system and ask, "How does this really work?" And more importantly, the willingness to roll up my sleeves and find out.

Now if you'll excuse me, I need to go commit some changes. With MyVCS, obviously. Because I'm either very dedicated to this project or have Stockholm syndrome. Possibly both.

*Happy coding, time travelers!* ⏰🚀

---

**P.S.**: If you're thinking about building your own version control system, do it! It's educational, humbling, and will give you a deep appreciation for the brilliant engineers who built the tools we use every day. Just maybe don't use it for production code. Your team will thank you.