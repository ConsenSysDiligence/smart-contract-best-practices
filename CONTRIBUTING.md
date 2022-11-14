# Contributing to the Smart Contract Security Best Practices

Please take a moment to review this document to make the contribution
process easy and effective for everyone involved. Following these guidelines
helps to communicate that you respect the maintainers' time in managing and
developing this open-source project. They should reciprocate that
respect in addressing your issue or assessing pull requests.

## Using the Issue Tracker

The issue tracker is the preferred channel for feature requests and submitting
pull requests. There are some cases where the issue tracker should not be used:

-   Please **do not** use the issue tracker for personal support requests to ask
    the community for help. There are many Discord and Telegram communities
    around Smart Contract development where people will be more than happy to
    help.
-   Please **do not** derail or troll issues. Keep the discussion on topic and
    respect the opinions of others.

## Content-related Issues

Sections on attacks and security best practices should include examples and
further reading. Security-related information is bound to change over time,
however. Content-related issues should flag areas that need more explanations
or contain outdated content.

A well-written issue flagging areas of the Best Practices that need attention
are very beneficial - thank you! Some guidelines for content-related issues:

1. **Use the GitHub issue search:** Check if the issue has already been
   reported. If the issue is already present, use a thumbs-up reaction to
   help the developers prioritize.
2. **Check if the issue has been fixed:** Try checking out open pull requests
    and development branches. The problem you are looking to flag might already
    be fixed.
3. **Isolate the problem:** An excellent content-related issue shouldn't leave
    others needing to chase you up for more information. Please try to be as
    detailed as possible. If content needs fixing across multiple areas,
    please flag one issue for each case.

## Feature Requests

Feature requests are welcome. But take a moment to find out whether your idea
fits with the scope and aims of the project. It's up to _you_ to make a solid
case to convince the maintainers of the merits of an additional piece of content.
Please provide as much detail and context as possible. Keep in mind that the
Best Practices do not aim to mirror general security considerations that can
be found in the project's respective documentation. Additional content to the
repository should be original and flag potential pitfalls developers
might encounter when building Smart Contract systems.

## Pull Requests

Reasonable pull requests - patches, improvements, new content - are a fantastic help.
They should remain focused in scope and avoid containing unrelated commits.
**Please ask first** before embarking on any significant pull request (e.g.
adding new content, refactoring example code). Otherwise, you risk spending a
lot of time working on something that the maintainers might not want to merge
into the project. Please adhere to the coding conventions used throughout the
project (indentation, comments, etc.). Adhering to the following this process
is the best way to get your work merged:

1. [Fork](http://help.github.com/fork-a-repo/) the repo, clone your fork,
   and configure the remotes:
    ```bash
    # Clone your fork of the repo into the current directory
    git clone https://github.com/<your-username>/<repo-name>
    # Navigate to the newly cloned directory
    cd <repo-name>
    # Assign the original repo to a remote called "upstream"
    git remote add upstream https://github.com/<upsteam-owner>/<repo-name>
    ```
2. If you cloned a while ago, get the latest changes from upstream:
    ```bash
    git checkout <dev-branch>
    git pull upstream <dev-branch>
    ```
3. Create a new topic branch (off the main project development branch) to
   contain your new content, change, or fix:
    ```bash
    git checkout -b <topic-branch-name>
    ```
4. Commit your changes in logical chunks. Please adhere to these [git commit
   message guidelines](http://tbaggery.com/2008/04/19/a-note-about-git-commit-messages.html)
   or your code is unlikely to be merged into the main project. Use Git's
   [interactive rebase](https://help.github.com/articles/interactive-rebase)
   feature to tidy up your commits before making them public.
5. Locally merge (or rebase) the upstream development branch into your topic branch:
    ```bash
    git pull [--rebase] upstream <dev-branch>
    ```
6. Push your topic branch up to your fork:
    ```bash
    git push origin <topic-branch-name>
    ```
7. [Open a Pull Request](https://help.github.com/articles/using-pull-requests/)
   with a clear title and description.

## Style Guidelines

To maintain a consistent writing style across the documents, a few
considerations need to be taken.

1. **Succinctness:** Use precise language and don't assume knowledge beyond
    Solidity and Ethereum basics.
2. **Show, don't tell:** Examples speak more than a lengthy exposition.
    Provide code examples where relevant and raise issues or best practices
    in the code around them.
3. **Give credit:** When further reading is recommended or resources are
    available that corroborate security issues, they should contain credit
    and link to the original resource.
4. **Label code:** Examples containing code must be labeled as insecure or
    secure where relevant. Specifically, in content regarding attacks,
    vulnerable code should be preceded by `// INSECURE`.
