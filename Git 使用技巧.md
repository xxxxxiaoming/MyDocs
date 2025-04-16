# 分支合并到主干时，代码提交信息变成自己的问题

**问题描述**：有一个开发分支，req_club_team，这个分支有不同的人的提交，为什么我把这个分支合入到master之后，在master查看，在req_club_team这个分支不是我提交的代码，在master上，用Git Lens查看显示提交人都是我呢，是因为我在req_club_team用了git pull origin master吗?

这种情况很可能是因为合并时使用了 **Squash 合并**，导致提交历史被压缩成一个新的提交，而这个新提交的作者变成了执行合并操作的人（也就是你）。

## 可能的原因

1. **Squash 合并**：在 GitLab 上合并 MR 时选择了"Squash commits"选项
   - 这会将分支上的所有提交压缩为一个提交
   - 新提交的作者默认为执行合并操作的人

2. **提交历史被重写**：
   ```bash
   # 这些操作可能导致作者信息改变
   git rebase master
   git reset --soft <commit>
   git commit --amend
   ```

3. **合并方式问题**：
   - 使用 `git merge --squash` 在本地合并
   - 或在 GitLab UI 中勾选了 "Squash commits" 选项

## `git pull origin master` 的影响

单纯的 `git pull origin master` 不会改变已有提交的作者信息，但如果之后进行了以下操作：
- 解决冲突后，进行了新的提交
- 执行了 rebase 操作重排提交
- 重置后重新提交了变更

## 如何验证和修复

### 验证原因
```bash
git log --pretty=format:"%h %an %s" master   # 查看 master 的提交日志
git log --pretty=format:"%h %an %s" req_club_team  # 查看开发分支日志
```

### 未来避免此问题
- 在 GitLab 合并时，取消勾选 "Squash commits" 选项
- 使用 `git merge` 而非 `git merge --squash`
- 保持明确的分支管理策略和团队沟通



# git rebase时的conflict

**问题描述**：有时候我把一个分支，比如req_test，合入到master之后，在req_test进行git rebase master的时候，会出现奇怪的冲突，冲突的地方都是我在req_test上提交的代码，这是为什么？

您遇到的情况是Git工作流中的一个常见现象。让我解释为什么会发生这种"自己和自己冲突"的情况：

## 为什么会出现这种冲突

当您将req_test合并到master后，再执行`git rebase master`时会出现冲突，这是因为：

1. **提交重复应用**：您在req_test上的提交已通过合并进入了master
2. **变更重叠**：rebase尝试将req_test分支上的提交"重放"到更新后的master上
3. **Git困惑**：Git看到这些变更已存在于master中，无法确定如何处理相同的变更

## 图解过程

```
初始情况:
      A---B---C (req_test)
     /
D---E (master)

合并到master:
      A---B---C (req_test)
     /         \
D---E-----------F (master, 合并提交)

再执行rebase:
D---E-----------F (master)
                 \
                  A'? B'? C'? (冲突发生)
```

## 解决方案

### 1. 合并后不要再rebase

最佳实践是：
```bash
# 合并req_test到master后
git checkout req_test
git merge master  # 而不是rebase
```

### 2. 如果必须rebase

如果您确实需要rebase，可以：
```bash
# 在req_test上
git rebase -i master
# 在交互式编辑器中删除掉已经进入master的提交
```

### 3. 重新创建分支

更干净的方式是：
```bash
git checkout master
git branch -D req_test  # 删除旧分支
git checkout -b req_test  # 从master创建新的req_test分支
```

## 避免此类问题的工作流建议

1. **合并方向的一致性**：
   - 总是从master向req_test合并（更新req_test）
   - 完成req_test后，将其合并到master
   - req_test合入master后，不再使用该分支或创建新分支

2. **采用"上游优先"原则**：
   - 始终保持master是req_test的上游
   - 使用`git merge master`更新req_test
   - 避免在两个方向都进行合并操作

这样的工作流可以大大减少这类冲突发生的概率。
