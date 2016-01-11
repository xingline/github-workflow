Github 团队协作
==============

Git 再次将 Unix 设计哲学发挥到极致，尤其是 KISS 原则（Keep It Simple and Stupid）。

使用它的人（不仅是编程人员）对它的认识，可能是这样：与其说 Git 是一个分布式版本控制工具，不如说 Git 是一个分布式文件系统。正式这样看似简单粗暴的设计让 Git 从众多版本控制工具中脱颖而出。

Git 相比其他版本控制工具有众多优势。如分布式，任何一个仓库都是一各完整的镜像。又如创建删除分支，Git 以很低的代价让使用它的人可以随意地快速创建删除分支。

本文就 Git 的分布式和分支特性，结合 Github 强大的源码分享贡献平台，探讨一种简单有效的 Github 团队协作流程，实现多环境发布、Pull Request 的流程化。

这个流程有如下两个原则：

1. 任何程序都可以追踪到对应的源码；
2. 所有代码都必须经过严格代码走查。


每个环境对应一个源码分支
--------------------

通常 Git 仓库会有一个 master 分支，这个分支的代码应该 **一直** 是最新稳定版的， **永远** 对着应生产环境发布的版本，所有生产环境应用都要从这个分支上发布，所以我们不应该在这个分支上做开发。

```
Source branch  ===>   Deploy environment
master                Production
```

我们可以创建一个 develop 分支，所有开发工作都在这个分支上进行。本地开发好，单元测试通过后，发布到开发环境和其他系统（或服务，或模块，或应用)联调测试。

（至于为什么不简写成 dev，答案是约定和对齐。大多数项目 develop 分支名都不缩写成 dev，所以我们也不应该缩写。对齐就是说 dev/master，一个缩写一个不缩写，很别扭，develop/master 看起来舒服点儿。）

```
Source branch  ===>   Deploy environment
develop               Development
master                Production
```

开发环境只是开发人员自己的环境，开发人员可以自由发布，所以经常会有各种变化，无法稳定，这对测试人员来说，是个灾难，所以说必须搭建属于测试人员自己的测试环境。这个环境相对稳定，只会定期发布，也可以做一些压力测试等。

既然有了测试环境，那么测试环境对应的源码就必须被标记，我们可以建一个 testing 分支（名字为什么这么长，不再解释了）。同 master 一样，testing 分支最新代码永远对应着测试环境。

有了 testing 分支，那么我们开发环境联调通过的代码就可以从 develop 分支定期合并到 testing 分支了，合并完成后，从 testing 分支上编译并打包，发布到测试环境，测试，联调。


```
Source branch  ===>   Deploy environment
develop               Development
testing               Testing
master                Production
```

注意，所有测试环境发布工作都要在 testing 这个分支上进行，坚决不能从 develop 上发布，这个可以在发布脚本上做严格控制。同样，生产环境发布工作也不能从 develop/testing 分支上进行。

测试环境积累一定的功能点，测试和联调通过后，就可以把 testing 分支的代码合并到 master 分支。这时 master 分支上的代码就是最新稳定版的，可以发布到生产环境了。但是发布前，必须更新 changelog，然后打 tag，以说明本次发布更新了哪些功能。打完 tag，才能发布，发布时只能从 master 分支编译大包，发布到生产环境。

```
develop ...................................
             \   \   \       \        \
testing ------o---o---o-------o--------o
                       \       \
master  ----------------o-------o
                        |       |
Tags                  1.0.1   1.0.2
```

这样说来，我们至少会有 3 个分支：develop/testing/master，一般项目基本够用，如果需要 UAT（Stage）环境，可以再添加其他分支。

这里只考虑到所有开发人员都工作在 develop 分支上这种情况，如果有需要，可以从 develop 分支上创建特性分支，开发完成后，合并到 develop 分支。


所有代码都必须经过代码走查
----------------------

Github 提供了 Fork 和 Pull request 功能，可以方便地分享代码，也可以贡献代码。

利用这些功能，团队可以很方便地完成代码审查，而不会影响效率和协作，而且 Github 的文件比较和批注非常强大，大大提高了代码审查和沟通效率。

具体流程如下：

1. 在 Github 组织（如 gotips）下创建项目仓库（如 test-git），此时项目地应该是  `https://github.com/gotips/test-git`；

2. 参与该项目的组织成员把这个项目 fork 到自己的账号下，此时项目地址是 `https://github.com/omigo/test-git`（如果仓库是私有的，fork 后，在自己的账号下依然是私有的，不会泄露）；

3. 然后在本机创建组织目录（如 go 语言需要创建 $GOPATH/src/github.com/gotips）；

4. 把自己 Github 账号下的项目 clone 到刚创建的组织目录下 `git clone git@github.com:omigo/test-git.git`（为什么不放到自己的目录下？代码是组织的，包路径也是组织的，不是自己的）；

5. 此时本地项目只有一个远程仓库快捷方式 `origin => git@github.com:omigo/test-git.git`，再添加一个组织快捷方式，执行 `git remote add gotips git@github.com:gotips/test-git.git`；

6. 前面所有工作都是准备环境，现在可以编码了。写完一个独立逻辑，单元测试通过后，提交一次到本地仓库。同时，不定期地推送到自己的 Github 仓库；

7. 直到独立的功能点完成，所有代码都在自己的 Github 仓库后，就可以提交 Pull request，请其他伙伴帮忙审查代码了。到自己的 Github 账号下，点击【New pull request】，选择分支，点击【Create pull request】，填写清晰明确的注释，再次点击【Create pull request】。提交 Pull Request 时可以选择 Labels、Assignee 等给定标记、指定由谁来做 Code Review；

8. 当被人邀请做 Code Review 时，我们会得到一个 Pull Request 链接，如 `https://github.com/gotips/test-git/pull/1`。进入后，可以看到提交人的注释和每次的提交日志，【Files changed】页面详细列出了所有的文件变更，并和修改前对比，发现问题，可以点击代码，在该行下写注释，提交人会收到通知，修改，然后再提交。如果审查没问题，就可以点击【Merge pull request】，完成合并。这样代码才到算回到组织仓库中，开发才算完成，可以测试了。

```
Github workflow:

                +------------+             +------------+                     +--------------+
                |   Local    |  git push   |   Github   |    pull request     |    Github    |
Developer       | Repository |  -------->  | Repository |     --------->      |  Repository  |
  1..n          |   1..n     |             |    1..n    |                     | Organization |
                +------------+             +------------+                     +--------------+
                      ^                                                              |
                      |                       git pull                               |
                      +--------------------------------------------------------------+

```


注意：

1. 只能 Push 到 Github 个人仓库，不能 Push 到组织仓库，可以从组织仓库 Pull 代码` git pull gotips develop`（如果觉得不安全，可以使用 `git fetch`）；
2. 提交 Pull Request 之前，需要先从 **组织** 仓库上拉取代码，保证代码时最新的，避免 Pull Request 代码合并时发生冲突，造成合并困难；
3. 如果发生了冲突，最好是让提交人先更新代码，自己解决冲突，然后提交，再次 Code Review，轻松合并；
4. 一个 Pull Request 修改的代码，最好不要超过 200 行，差不多是一天的工作量，太多，Code Review 看不过来，大多情况下草草了事，质量无法保证；
5. 代码要符合规范，统一格式，避免琐碎问题影响 Code Review 心情；
5. Code Review 是一个学习的机会，但更多是监督与合作，提交人和合并人共同对代码负责，代码有问题，两人共同承担；
7. 最好不要强制谁走查谁的代码，应该由提交人邀请，这样才能合作更友好。


分支管理与 Pull Request 的结合
---------------------------

上面分别探讨了分支管理和 Pull Request，下面再说说，如何把两者结合起来。

我们的原则依然不变：只能在 develop 分支做开发（可能也有其他特性分支），每次提交必须经过严格审查，每个分支对应一个发布环境。

为了方便，一般我们团队协作中，会把 develop 分支设置为默认分支。

在组织下创建仓库后，建好 3 个分支 master/testing/develop，然后到 Settings -> Branches 下面 选择 develop 为默认分支。这样所有人 fork 的项目也会默认 develop 分支，clone 也是 develop 分支，避免分支切换和忘记切换分支带来的麻烦。开发都在 develop 分支，Pull Request 也在 develop 分支，merge 也是。

然后就是测试环境发布了，测试环境对应 testing 分支，应该指定一人专门管理测试环境发布。所有开发人员先从组织仓库 develop 分支拉取最新（需要发布到测试环境的版本），和本地代码 develop 分支合并后，再把 develop 分支合并到 testing 分支，然后提交代码到 Github 个人仓库 testing 分支下，提交 pull request 给测试发布管理者。测试发布管理者合并后，从组织仓库更新 testing 代码，并从 testing 分支发布到测试环境。

如果分支不多，并且团队协作良好，合并分支的工作也可以由测试发布管理者一个人完成，以简化流程。

同样指定一位生产发布管理者，测试通过后，测试发布管理者就可以把 testing 分支合并到 master 分支，提交到 Github 个人仓库 master 分支，提交 Pull Request 给生产发布管理者。生产发布管理者合并代码，打 tag，完成生产环境发布。

ChangeLog 问题，什么时候更新 ChangeLog 呢？

我们如果开发时就已经知道版本号，那么开发时就可以更新，如果发布时才知道版本号，那么发布管理者可以通过 git log 了解本次发布有哪些更新，编写 ChangeLog。当然不排除有更好的 ChangeLog 更新办法，或者有更好的 Git 协作流程。

注意：无论是分支管理，还是 Pull Request，还是编写 ChangeLog，都需要 git log 清晰明确，粒度要细，至少不能丢掉功能点，不能把两个功能点一次 commit，log   上只写一个功能点，更加不能出现无效 log，比如 优化、修复 bug、更新了一些文档、调整算法等等。
