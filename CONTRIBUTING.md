# Guide to Contributing to Smart Contract Security Best Practices

Thank you for your interest in improving our project! This document will help you understand the contribution process and make it as effective as possible for all participants.

## Table of Contents

- [Using the Issue Tracker](#using-the-issue-tracker)
- [Content-related Issues](#content-related-issues)
- [Feature Requests](#feature-requests)
- [Pull Requests](#pull-requests)
- [Style Guidelines](#style-guidelines)
- [Review and Acceptance Process](#review-and-acceptance-process)
- [Community and Support](#community-and-support)

## Using the Issue Tracker

The issue tracker is the preferred channel for feature requests and submitting pull requests. However, there are cases where the issue tracker should not be used:

-   Please **do not** use the issue tracker for personal support requests or questions to the community. There are many Discord and Telegram communities dedicated to smart contract development where people will be happy to help you.
-   Please **do not** derail or troll issues. Stay on topic and respect the opinions of others.

## Content-related Issues

Sections on attacks and security best practices should include examples and additional reading materials. Security-related information may change over time. Content-related issues should point to areas that need additional explanations or contain outdated content.

Well-written issues that flag areas of Best Practices requiring attention are very helpful - thank you! Here are some guidelines for content-related issues:

1. **Use the GitHub issue search:** Check if the issue has already been reported. If the issue already exists, use a thumbs-up reaction to help developers prioritize.
2. **Check if the issue has been fixed:** Try checking open pull requests and development branches. The problem you want to flag may have already been fixed.
3. **Isolate the problem:** A good content-related issue should not leave others needing to ask you for additional information. Please be as detailed as possible. If content needs fixing across multiple areas, please create a separate issue for each case.

## Feature Requests

Feature requests are welcome. But before proposing, make sure your idea fits with the scope and goals of the project. It's up to _you_ to make a compelling case for the merits of additional content to support the project. Please provide as much detail and context as possible.

Remember that Best Practices does not aim to duplicate general security considerations that can be found in the project's respective documentation. Additional content should be original and point out potential pitfalls that developers may encounter when building smart contract systems.

## Pull Requests

Reasonable pull requests - fixes, improvements, new content - are fantastic help. They should be focused in scope and not contain unrelated commits. **Please ask first** before embarking on any significant pull request (e.g., adding new content, refactoring example code). Otherwise, you risk spending a lot of time working on something that the maintainers might not want to merge into the project.

Please adhere to the coding conventions used throughout the project (indentation, comments, etc.). Following this process is the best way to get your work merged:

1. [Fork](http://help.github.com/fork-a-repo/) the repository, clone your fork, and configure the remotes:
    ```bash
    # Clone your fork of the repository into the current directory
    git clone https://github.com/<your-username>/smart-contract-best-practices
    # Navigate to the newly cloned directory
    cd smart-contract-best-practices
    # Assign the original repository to a remote called "upstream"
    git remote add upstream https://github.com/ConsenSys/smart-contract-best-practices
    ```
2. If you cloned a while ago, get the latest changes from upstream:
    ```bash
    git checkout main
    git pull upstream main
    ```
3. Create a new topic branch (off the main project development branch) for your new content, change, or fix:
    ```bash
    git checkout -b <topic-branch-name>
    ```
4. Commit your changes in logical chunks. Please adhere to these [git commit message guidelines](http://tbaggery.com/2008/04/19/a-note-about-git-commit-messages.html), or your code is unlikely to be merged into the main project. Use Git's [interactive rebase](https://help.github.com/articles/interactive-rebase) feature to tidy up your commits before making them public.
5. Locally merge (or rebase) the upstream development branch into your topic branch:
    ```bash
    git pull [--rebase] upstream main
    ```
6. Push your topic branch up to your fork:
    ```bash
    git push origin <topic-branch-name>
    ```
7. [Open a Pull Request](https://help.github.com/articles/using-pull-requests/) with a clear title and description.

## Style Guidelines

To maintain a consistent writing style across the documents, a few considerations need to be taken:

1. **Succinctness:** Use precise language and don't assume knowledge beyond Solidity and Ethereum basics.
2. **Show, don't tell:** Examples speak more than a lengthy exposition. Provide code examples where relevant and raise issues or best practices in the code around them.
3. **Give credit:** When further reading is recommended or resources are available that corroborate security issues, they should contain credit and link to the original resource.
4. **Label code:** Examples containing code must be labeled as insecure or secure where relevant. Specifically, in content regarding attacks, vulnerable code should be preceded by `// INSECURE`.
5. **Markdown Formatting:** Use proper Markdown formatting to improve readability:
   - Use headings (`#`, `##`, `###`) to structure the document
   - Highlight important terms using **bold** or *italic*
   - Use code blocks (``` ```) for code examples
   - Create lists using `-` or `1.`, `2.`, etc.

## Review and Acceptance Process

After submitting a pull request, your changes will be reviewed by the project maintainers. The process typically includes:

1. **Initial review:** Checking for compliance with style guidelines and general applicability.
2. **Discussion:** Possible discussion of the proposed changes and requests for additional changes.
3. **Acceptance and merging:** After approval, your changes will be merged into the main codebase.

Please be patient during the review process and be prepared to make additional changes upon request.

## Community and Support

If you have questions about the contribution process or need help, you can:

- Join our [Gitter chat](https://gitter.im/ConsenSys/smart-contract-best-practices)
- Ask a question in the relevant GitHub Discussions section
- Follow project updates on [Twitter](https://twitter.com/ConsenSys)

We value your contribution and strive to make the process as smooth and productive as possible!
