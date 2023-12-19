---
title: Git Hackfest 2023
date: 2023-12-19 00:00:00 +0000
categories: [CAPTURE THE FLAG]
tags: [cybersecurity, intermediate, writeup, git]
---

Back in october 2023, I took part in the Hackfest 24h CTF which was quite intense. During this CTF, 2 git challenges were available for a decent amount of points for each of them. With my programmer background and the fact that I mastered the [Bandit challenges](https://overthewire.org/wargames/bandit/), I though I had enough knowledge to get over it so I told my team that I could easily handle them. In fact it took me a couple of hours to figure them out and shown me how creative you can be with git when it comes to hide data.

I kept only one of the git challenge on my computer -the easiest one- and erased the other not knowing I would be able to post about them later. However, the idea was quite the same.

So I downloaded the archive and found myself with this architecture:

```
.
..
.git
flag.txt
```

After realizing that the flag.txt had only one letter, I just though it was going to be easy, there is going to be maximum 5 commits and I just had to manually rollback to the version having the flag so I tried to get the number of commits:

```shell
git log --oneline | wc -l #printing the commits and piping the result through a line counter
```

```
2612
```

![Desktop View](/assets/img/2023-12-19-git-hackfest/Surprised_Pikachu.jpg)

After such a betrayal, I tried to investigate more on what was going on, doing:

```shell
git rebase -i HEAD~2611
```

And replaced all the "pick" by "edit", I could manually see each commit and how they modify the `flag.txt` file so I chained the 2 following commands over and over again:

```shell
git rebase --continue; cat flag.txt
```

Until I realized that it will not lead me anywhere and made me abort the rebase. From here I though I needed a more visual point of view on what was happening so I opened my VSCode into this folder and took advantage of the git graph extension to explore the commits.

I was going down the commits desperately looking for a useful piece of information that could get me on track until I found this:

![Desktop View](/assets/img/2023-12-19-git-hackfest/git_commit_linking.png)

I realized that each commit title was a hash, and VSCode just gave me the hint I needed: some commits have hashes of other commits. From here you can guess that if you have a commit A having the hash of the commit B, you can jump to commit B which may have the hash of the commit C to which you can jump to and so on. Then among those 2612 commits you have commits chained by their titles that can make something !

The thing is: I don't see how long is the "chain" I am talking about and there's no way for me to do that manually, so I wrote a Shell script collecting the commit chain for me (during the Hackfest, the script was really messy, I modified it, removed useless echos and added comments to make it more readable):

```shell
#!/bin/bash

function get_parent_hash(){
     # Set the commit hash from the command line argument
    local commit_hash="$1"
    # Get the parent commit hash using Git log
    local parent_commit_hash=$(git log --format=%P -n 1 "$commit_hash" 2>/dev/null)
    # Check if the commit hash exists in the Git repository
    if [ -n "$parent_commit_hash" ]; then
        echo "$parent_commit_hash"
    else
        return 1
    fi
}

function get_commit_title(){
    # Set the commit hash from the command line argument
    local commit_hash="$1"
    # Get the commit title using Git log
    local commit_title=$(git log --format=%s -n 1 "$commit_hash" 2>/dev/null)
    # Check if the commit hash exists in the Git repository
    if [ -n "$commit_title" ]; then
        echo "$commit_title"
    else
        return 1
    fi
}

function commit_exists(){
    # Set the commit hash from the command line argument
    commit_hash="$1"
    # Check if the commit hash exists in the Git repository
    if git cat-file -e "$commit_hash^{commit}" 2>/dev/null; then
        echo 0
    else
        echo 1
    fi
}

LATEST_COMMIT="3431e57c288848b15394b4c70eabaccc7c06c36e"

commit_hash=$LATEST_COMMIT
#Iterating through the commits to find the first element of the chain
while [[ $(commit_exists "$(get_commit_title "$commit_hash")") == "1" ]]; do
    commit_hash=$(get_parent_hash "$commit_hash")
done

#Getting each commit hash and concatening letters from flag.txt
flag_string=""
while [[ $(commit_exists "$(get_commit_title "$commit_hash")") == "0" ]]; do
    git checkout "$commit_hash" 2>/dev/null
    flag_string="$flag_string$(cat flag.txt)"
    commit_hash=$(get_commit_title "$commit_hash")
done

echo "$flag_string"

# Coming back to the original commit when we're done
git checkout 3431e57c288848b15394b4c70eabaccc7c06c36e 2>/dev/null
```

Which gave me some giberish:

```
SEYtYWE4ODQ0NDI5MGM5ZDExMjI3NzljZDU5MWFmMTQzMDA
```

By this time I was so tired I didn't realized what encoding it would be and I didn't even know if I was onto something or not, so I just threw the result to [CyberChef](https://gchq.github.io/CyberChef/) with its magic tool and it woke me up when I suddenly recognized the Hackfest flag format (it was base64 encoded):

```
HF-aa88444290c9d1122779cd591af14300
```

This was just another way to play around with Git !
