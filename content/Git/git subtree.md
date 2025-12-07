---
tags:
  - git
---
`tldr git subtree` shows:

>   Add a Git repository as a subtree and squash the commits together:
> 
>     git subtree add --prefix path/to/directory --squash repository_url branch_name
> 
>   Update subtree repository to its latest commit:
> 
>     git subtree pull --prefix path/to/directory repository_url branch_name
> 
>   Merge recent changes up to the latest subtree commit into the subtree:
> 
>     git subtree merge --prefix path/to/directory --squash repository_url branch_name
> 
>   Push commits to a subtree repository:
> 
>     git subtree push --prefix path/to/directory repository_url branch_name
> 
>   Extract a new project history from the history of a subtree:
> 
>     git subtree split --prefix path/to/directory repository_url --branch branch_name

使用场景: 
- 同一个 repo 有两个 active 的 branches，不想频繁地来回切换（due to untracked files, stashed changes, etc.）
- 用 subtree 实现类似于 submodule 的子模块功能，但更方便

在使用场景上与 [[git submodule]] 有相似之处，但有人建议应该尽量更多地使用 subtree 而不是 submodule，有几点理由：
- `git subtree` is user-oblivious. 
- `git subtree` does not add extra metadata in repo, as opposed to `.gitsubmodule`.

我个人的理解是，如果是自己项目中逻辑上的子项目，并且子项目的重要程度不至于为它做单独托管，用 subtree 确实更简便。其他情况，例如有外部项目依赖，subtree 并不能替代 submodule。