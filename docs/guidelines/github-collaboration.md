# Collaboration and Contribution Guidelines at Contoso

## Demystifying Your First Open Source Contribution at Contoso

Open source projects have made a profound impact on the world of software development. They foster a collaborative ecosystem that encourages code sharing, mutual learning, and technological advances at an unprecedented pace.

As a developer at Contoso, we encourage you to explore and contribute to Contoso's open source projects hosted on **GitHub**. This guide will serve as a comprehensive resource to help you find suitable open source projects, understand their contribution guidelines, and make your first pull request on **GitHub**.  

---

## Discovering Open Source Projects

First, let's look at finding a suitable project that aligns with your skills, interests, and the tech stack you're familiar with or wish to learn. Here are some tips to help you find the right project:

**[GitHub Explore:](https://github.com/explore)** This built-in feature on GitHub lets you explore popular repositories, trending developers, topic-wise collections, and other curated sections. This wealth of content is a great starting point to discover open-source projects that match your abilities and interests.

**[Awesome Lists:](https://github.com/topics/awesome)** Awesome Lists are meticulously curated compilations that span across many GitHub repositories and topics, such as **_"Awesome Python"_**, **_"Awesome Machine Learning"_**, or **_"Awesome Docker"_**. These lists include a myriad of superbly crafted libraries and tools worthy of your attention and potential contribution.

**GitHub Search:** Use GitHub's advanced search feature to seek out projects using specific filters, such as repositories written in a particular language, those with issues labeled as **_"good first issue"_** or **_"help wanted"_**, or projects that have not been updated recently.  

---

## Contributing Guidelines at Contoso

At Contoso, we believe in the power of collaboration and open source. An essential step before making any contribution is to review the project's contribution guidelines, which will guide you throughout your contribution journey. These guidelines are typically found in a file named `CONTRIBUTING.md`, `CONTRIBUTE.md`, or `HOW_TO_CONTRIBUTE.md` in the project repository. In some cases, it might also be a part of the `README.md` file.

**The guidelines may contain:**

1. **Code of Conduct:** This outlines the expectations for participation in the community, as well as the consequences for unacceptable behaviour.

2. **What to Contribute:** Most projects clarify what they expect from contributors, be it bug reports, feature requests, or code improvements.

3. **Pull Request Process:** Step-by-step guidance on how to submit a pull request (PR) for the project.

4. **Coding Standards:** The coding conventions, like code styling and testing procedures, used in the project.

If the contributing guidelines seem unclear, do not hesitate to open an issue asking the maintainers for clarification.  

---

## Making Your First Pull Request at Contoso

Finally, let's focus on the most rewarding part: contributing to an open-source project by making a pull request. Here are the steps to follow:

**1. Fork the Repository:**  
GitHub allows users to create a personal copy of another user's project. This is known as forking a repository. Find the `'Fork'` button located at the top-right corner of the project page and click on it. Within a few seconds, a copy of the repository will appear in your account.

**2. Clone the Repository:**  
The next step is to get this remote repository on your local system for making changes. This is known as **cloning**. Use the `'git clone'` command followed by the URL of your forked repository.

```bash
# Code example
git clone "https://github.com/Pwd9000-ML/GenAI-IaC-Demo.git"

```bash
#Code example
git clone "https://github.com/Pwd9000-ML/GenAI-IaC-Demo.git"
```

**3. Create a New Branch:**  
It's a good practice to create a new branch for each new feature or bug fix you work on, to keep your contributions organised. Use the `'git branch "branch_name"'` to create a new branch, and `'git checkout "branch_name"'` to switch to it.

```bash
#Code example
git branch "branch_name"
git checkout "branch_name"
```

You can also create the new branch directly from the **GitHub website**, or your preferred **Integrated Development Environment (IDE)**, such as [VSCode](https://code.visualstudio.com) or even a [GitHub Codespace](https://docs.github.com/en/codespaces/overview?wt.mc_id=DT-MVP-5004771).

**4. Make Changes and Commit:**  
Now, dive into the code. First, open the code in your Development Environment (IDE), e.g. [VSCode](https://code.visualstudio.com/?wt.mc_id=DT-MVP-5004771). Make the necessary changes and save them. After making changes, use `'git add .'` to stage the changes, and `'git commit -m "commit message"'` to save your changes. Be sure to write a meaningful commit message, explaining the changes you made.

```bash
#Code example
git add .
git commit -m "commit message"
```

**5. Push Your Changes:**  
After committing your changes, it's time to upload them to your **forked repository** on GitHub. This process is known as **pushing**. The `'git push origin "your_branch_name"'` command will push your changes to your forked GitHub repository.

```bash
#Code example
git push origin "your_branch_name"
```

**6. Create a Pull Request:**  
Navigate back to your forked repository on GitHub. Click on `'Pull Request'` then create a new pull request. Ensure the source branch is **your** recently created **branch** and the destination branch is the **master/main** branch of the **original repository**. Summarize the changes you've made, then click `'Create a pull request'`.

Pat yourself on the back! You've just made your first pull request. The maintainers of the original repository will review your request, provide feedback or suggest changes (if required), and finally merge the changes into the original repository. You can inspect or add comments to your **Pull request (PR)** on the original repository by clicking on the `'Pull Requests'` tab.  

---

## Conclusion

Venturing into the world of open source contributions can seem like an uphill task, particularly if you are just getting started. However, with a resource like GitHub at your disposal, and with these steps in mind, the process becomes significantly more manageable and less intimidating.

Remember, contributing to open source projects extends far beyond bug fixes or feature additions. It involves learning and growing, enhancing your coding skills, understanding and participating in the dynamics of software development workflow, and above all, becoming a part of a community — all while contributing to the code that runs the world.

So, find a project that resonates with your interest, understand its guidelines, and make your first pull request. Every contribution, no matter how small, matters. The world of open source is waiting for what you have to offer. Happy coding!

---

### _Author_

Marcel Lupo - Follow me on: [GitHub](https://github.com/Pwd9000-ML) | [X/Twitter](https://x.com/pwd9000) | [LinkedIn](https://www.linkedin.com/in/marcel-l-61b0a96b/)
