---
title: Resolving Repetitive Git Conflicts
categories:
  - Git
---

At my company's codebase, we have a couple of files that always generate merge conflicts. Repetitively solving those conflicts just seems like a terrible waste of time.

An example of one such file is our rspec coverage report file. We use this file to compare it to the one on our staging branch as a pre-push hook (using [Husky](https://github.com/typicode/husky)), and make the push fail if the coverage dropped. The file looks like this:

_.last_run.json_

```json
{
  "result": {
    "covered_percent": 27.51
  }
}
```

The only change to this file is the number of `covered_percent`, which obviously will cause a conflict whenever trying to merge.

What I would really like to do is be able to "teach" git how to merge specific files, give it some rules to follow when merging this file. In this case, prefer the version with the higher number. This might make a huge difference with this example, but I can think of a lot of uses for that.

What I ended up doing is the following:

- Merge staging into my branch, which generates the conflict.
- In my scenario, I can safely always accept my version and _overwrite_ staging, so I tell git to do that.
- Stage and commit.

You can create shell script and add it to your `.bashrc` or `.zshrc`, and you're laughing (make sure you change branch and file names to fit your needs). number_of_remaining checks if there are more conflicts other than the one we addressed. If not, we commit and push.

```bash
stagein () {
  git merge staging
  git checkout --ours coverage/.last_run.json
  git add coverage/.last_run.json
  number_of_remaining=$(git diff --name-only --diff-filter=U | wc -l)
  if [ $((number_of_remaining == 0)) ]
  then
    git commit -m "merged staging" && git push
  else
    echo "There are more conflicts to solve!"
  fi
}
```
