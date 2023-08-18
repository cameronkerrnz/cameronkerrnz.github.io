+++ 
draft = false
date = 2023-01-25
title = "Map Git Dependencies using n-grams in Python"
description = ""
slug = ""
authors = []
tags = ["AWS", "CloudFormation", "Git", "Python", "SRE"]
categories = []
externalLink = ""
series = []
+++

> This post was originally published on the [Pythian blog](https://blog.pythian.com/).

Which option would you prefer to receive when asked to make many changes throughout a complex system you’re unfamiliar with?

Many requests like “fix _this”_ ...

... or many requests like “fix _this_, which might also be known as _that_, most likely by making a change _there_.”

Imagine you have a lot of Terraform code spread across various Git repositories for about a lot of cloud resources. You scan your cloud resources using some audit tool and have a list of things that need fixed... but now you need to figure out where the matching source is. As an SRE consultant helping organizations with Infrastructure-as-Code technologies, we often need to discover how different systems are related and trace how a complex system has been deployed.

## CloudFormation traced back to Git using Git Worktrees

In this example, Mood Stores Inc. uses CloudFormation to deploy its infrastructure. Each CloudFormation ‘stack’ is defined with a ‘template’ found in one of many (say ~100) Git repositories, each of which has several branches. We need to map the CloudFormation stack template to its template somewhere in a Git repository. It’s certainly not something you want to search for by hand when you can get the computer to do something more innovative.

Using [Git worktrees](https://git-scm.com/docs/git-worktree), we can check out multiple branches to different filesystem locations. This is handy if you want to do a recursive search through the filesystem, so perhaps I have a directory for each branch:

- repos/Mood Stores/purple-store.moodstores.com.branch.develop/
- repos/Mood Stores/purple-store.moodstores.com.branch.main/
- repos/Mood Stores/orange-store.moodstores.com.branch.develop/
- repos/Mood Stores/orange-store.moodstores.com.branch.main/

Now let’s assume that you look at a CloudFormation stack; perhaps this stack name is present in another AWS resource as a Tag, such as on an S3 bucket. The stack name might look like the following: `purple-store-mood-sandbox`. Naming conventions often don’t line up well in real-life; the ‘sandbox’ environment is equal to the Git ‘develop’ branch, and maybe ‘production’ implies the ‘main’ branch, so it can be helpful to normalize the text first. A chain of replacements is a nice easy, if not particularly scalable, way to do this.

```python
re.sub(r'[^a-z0-9]', ' ', s.lower())\
    .replace('sandbox', 'develop')\
    .replace('production', 'main')
```

With this done, you might think to use a well-known text similarity algorithm, such as the Levenshtein edit distance, although in this case, you’d be unhappy with the results because the strings are too different. Levensthein edit distance is better suited to suggesting typing corrections. Other algorithms look for similarity on a much broader scale (documents, sentences, phrases of natural language text).

## N-grams and Python

In my case, I’m much more interested in counting common substrings… although on a much smaller scale as might be done in fields such as Bioinformatics. So I turned to n-grams. Turning strings into their n-grams is pretty easy, although the code is a bit non-intuitive. An n-gram (where n is an integer, such as 4) can be considered a sliding window over a list… remember that a string is a list of characters in Python. Visually:

```plain
cameron <-- the input string
came    <-- the first 4-gram (n-gram where n=4)
 amer   <-- the second
  mero
   eron
```

The following diagram should make the processing stages of the following code easier to comprehend:

```plain
"cameron" > ['C', 'a', 'm', 'e', 'r', 'o', 'n']  >      > [('C', 'A', 'M'), >      > ["CAM", 
   n=3    > ['A', 'm', 'e', 'r', 'o', 'n']       > zip  >  ('a', 'm', 'e'), > join >  "ame", 
          > ['M', 'e', 'r', 'o', 'n']            >      >  ('m', 'e', 'r'), >      >  "mer", 
                                                        >  ...]             >      >  ...]   

                          (Capitalization is only for highlighting purposes)
```

Creating the full function in Python, returning the output as a set (like a list but with no duplicates)

```python
def ngrams(s, n):
    s = re.sub(r'[^a-z0-9]', ' ', s.lower())\
        .replace('\bsandbox\b', 'develop')\
        .replace('\bproduction\b', 'main')
    ngrams = zip(*[s[i:] for i in range(n)])
    return set([''.join(ngram) for ngram in ngrams])
```

N-grams are used for tasks such as phrase-searching and come up a lot in technology, such as text searching. In my use case, I’m interested in knowing the size of the intersection of two sets of n-grams. The larger the intersection, the more similar. If there are too few similarities, we should probably not return a result. That’s enough theory; let’s see the complete code:

```python
#!/usr/bin/env python3

import re

# For comparitive purposes only; pip install levenshtein
import Levenshtein

def ngrams(s, n):
    s = re.sub(r'[^a-z0-9]', ' ', s.lower())\
        .replace('sandbox', 'develop')\
        .replace('production', 'main')
    ngrams = zip(*[s[i:] for i in range(n)])
    return set([''.join(ngram) for ngram in ngrams])

a = 'purple-store-mood-sandbox'
# a = 'purple-store-mood-main'
# a = 'yellow-bobbidy-boo'

bs = [
    'repos/Mood Stores/purple-store.moodstores.com.branch.develop/',
    'repos/Mood Stores/purple-store.moodstores.com.branch.main/',
    'repos/Mood Stores/orange-store.moodstores.com.branch.develop/',
    'repos/Mood Stores/orange-store.moodstores.com.branch.main/',
    'repos/Mood Stores/customer-support.branch.develop/',
    'repos/Mood Stores/customer-support.branch.main/',
    'repos/Mood Stores/design.moodstores.com.branch.develop/',
    'repos/Mood Stores/design.moodstores.com.branch.main/',
]

a_grams = ngrams(a, 4)

# Show how to determine the score for `a` against each `b`

print(f"Looking to compare against {a!r}\n")

print("Scoring each option using ngrams (larger means more similar):\n")
for b in bs:
    b_grams = ngrams(b, 4)
    intersection = a_grams.intersection(b_grams)
    score = len(intersection)
    print(f"  {score:3}  {b}")

# Alternatively, let's see how we might use that to sort the list `bs` and take
# the best match, if it meets some threshold. Determining a threshold is
# a weakness, but if n is smaller we would expect more matches, and proportional to
# the length of the strings being matched.
#
# With a threshold aiming to match 0.25 of the string, then we could use
# something as simple as:  0.25 * (len(a)-n) / n

def closest_ngram_match(a, bs, n=4, t=0.5):
    threshold = t * (len(a) - n) / n
    print(f"\nThreshold is {threshold}")
    scored = [ ( len(a_grams.intersection(ngrams(b, n))), b ) for b in bs ]
    scored.sort(reverse=True, key=lambda x: x[0])
    filtered = [ x for x_score, x in scored if x_score >= threshold ]
    if len(filtered) > 0:
        return filtered[0]
    else:
        return None

closest = closest_ngram_match(a, bs)

print(f"With a threshold, best match is: {closest}")

# Compare with a more classical method of text-similarity

print(
    "\nFor reference, score each using the more typical\n"
    "Levenshtein edit-distance (smaller means more similar):\n")

def norm(s):
    out = re.sub(r'[^a-z0-9]', ' ', s.lower())\
        .replace('sandbox', 'develop')\
        .replace('production', 'main')
    return out

a_norm=norm(a)

for b in bs:
    score = Levenshtein.distance(a_norm, norm(b))
    print(f"  {score:3}  {b}")
```

Let’s see it in action. I’ve provided some sample inputs for ‘a’

```plain
Looking to compare against 'purple-store-mood-sandbox'

Scoring each option using ngrams (larger means more similar):

   20  repos/Mood Stores/purple-store.moodstores.com.branch.develop/
   15  repos/Mood Stores/purple-store.moodstores.com.branch.main/
   15  repos/Mood Stores/orange-store.moodstores.com.branch.develop/
   10  repos/Mood Stores/orange-store.moodstores.com.branch.main/
   11  repos/Mood Stores/customer-support.branch.develop/
    6  repos/Mood Stores/customer-support.branch.main/
   11  repos/Mood Stores/design.moodstores.com.branch.develop/
    6  repos/Mood Stores/design.moodstores.com.branch.main/

Threshold is 2.625
With a threshold, best match is: repos/Mood Stores/purple-store.moodstores.com.branch.develop/

For reference, score each using the more typical
Levenshtein edit-distance (smaller means more similar):

   36  repos/Mood Stores/purple-store.moodstores.com.branch.develop/
   39  repos/Mood Stores/purple-store.moodstores.com.branch.main/
   39  repos/Mood Stores/orange-store.moodstores.com.branch.develop/
   42  repos/Mood Stores/orange-store.moodstores.com.branch.main/
   32  repos/Mood Stores/customer-support.branch.develop/
   36  repos/Mood Stores/customer-support.branch.main/
   35  repos/Mood Stores/design.moodstores.com.branch.develop/
   38  repos/Mood Stores/design.moodstores.com.branch.main/
```

In the case of ‘yellow-bobbidy-boo` we wouldn’t expect to match any of the inputs. In that case, we prefer to return nothing rather than selecting the least bad. The thresholding is simple and works well enough for the current task.

## Using this Technique

Let’s finish off by imagining how we might use this in practice. You are given a list of audit findings from [Prowler](https://github.com/prowler-cloud/prowler) or similar and have a list of S3 buckets that don’t have encryption enforced:

- Get the list of AWS resources from Prowler
- Retrieve AWS tags for each resource
- Extract the [`aws:cloudformation:stack-name`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-resource-tags.html) tag
- Retrieve the Original version of the template from the CloudFormation stack to find a list of candidate files in different repositories
- Compare (using n-gram similarity) the stack name against the list of candidate directories or file names to get the most likely
- Rather than saying, “fix this,” you can say, “fix _this_, which might be better known as _that_, most likely by making a change _there_”
