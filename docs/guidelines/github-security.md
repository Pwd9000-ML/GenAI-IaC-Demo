# Securing Your Code with GitHub at Contoso

## Understanding and Leveraging GitHub's Security Tools

At Contoso, we prioritize the security of our code and projects. GitHub, our chosen platform for version control, offers a comprehensive suite of security features designed to keep our projects safe and secure.

In this guide, we will walk you through the various features available on GitHub, from **security alerts** for **vulnerable dependencies** to **secret scanning**, to help you fortify your code at Contoso.

---

## Automating Code Security

Before we dive deeper, let's understand why securing our code is crucial. With cyber threats on the rise, robust security measures are more important than ever. Hackers often exploit vulnerabilities in code and dependencies, which might be left exposed accidentally.

GitHub's automated security features help mitigate these risks, making our projects at Contoso resilient to such threats.

You can enable and configure these security features on **_all_** repositories by navigating to the **Security & Analysis** tab in your organization's **settings**.

Alternatively, you can enable or disable security features on individual repositories by navigating to the **Security** tab in your repository's **settings**.

**Recommendation:** At Contoso, we recommend enabling these features on **_all_** repositories (both existing and new) within our organization. If absolutely necessary, you can disable specific features on a per-repository basis.

Let's take a closer look at some of these security features in more detail.

---

## 1. Security Alerts for Vulnerable Dependencies

At Contoso, we leverage GitHub's **[Dependabot Alerts](https://docs.github.com/en/code-security/dependabot/dependabot-alerts/about-dependabot-alerts?wt.mc_id=DT-MVP-5004771)** to monitor our dependencies. This feature, accessible under the repository's Security > Vulnerability alerts > Dependabot section, sends alerts whenever it detects vulnerabilities in our dependencies.

For instance, if you're using an outdated or vulnerable version of a library, **Dependabot** will notify you about the vulnerability, its severity level, and provide steps to resolve it. Depending on the severity of the issue, GitHub can also generate an automated security update to mitigate the risk, thereby enhancing the security of our code at Contoso.

---

## 2. Secret Scanning

At Contoso, we understand that in the rush of development work, it's not uncommon to accidentally commit sensitive details like **API keys** or **passwords**. GitHub's Secret Scanning feature is invaluable in such scenarios.

Suppose you unknowingly committed an **Azure Storage Account Access Key** to your repository. The **Secret Scanning** feature, once activated, would identify this and notify you or the secret provider. You can then revoke the compromised secret and generate a new one, thereby preventing any unauthorized access.

For more information, refer to the [supported secrets](https://docs.github.com/en/code-security/secret-scanning/secret-scanning-patterns#supported-secrets?wt.mc_id=DT-MVP-5004771) documentation.

---

## 3. Code Scanning

GitHub's Code Scanning feature, powered by the semantic analysis engine **CodeQL**, is a crucial security tool that scans your code for any potential vulnerabilities.

Consider a scenario where a developer unknowingly introduces a SQL injection vulnerability in their code. The **Code Scanning** feature would identify this vulnerability during its analysis, providing a description of the issue and advice for resolution. This proactive approach to threat detection allows for resolution before any damage occurs.

For more information, refer to the [supported languages and frameworks](https://docs.github.com/en/code-security/code-scanning/introduction-to-code-scanning/about-code-scanning-with-codeql#about-codeql?wt.mc_id=DT-MVP-5004771) documentation.

If your repository hosts supported CodeQL languages, you can either let GitHub automatically analyze your code using a **_default_** setting or customize an advanced configuration using a **YAML** config.

If your repository does not host supported CodeQL languages, or even if it does but also contains other languages or frameworks, you can add third-party code scanning tools to further enhance your code's security, such as:

- **[SonarCloud](https://github.com/Pwd9000-ML/terraform-azurerm-nsg-administration/actions/new?category=security&query=SonarCloud)**: A cloud-based code analysis service that automatically detects bugs, vulnerabilities, and code smells in your code.
- **[TfSec](https://github.com/Pwd9000-ML/terraform-azurerm-nsg-administration/actions/new?category=security&query=TFSEC)**: A static analysis security scanner for your Terraform code.
- **[trivy](https://github.com/Pwd9000-ML/terraform-azurerm-nsg-administration/actions/new?category=security&query=Trivy)**: Scans Docker container images for vulnerabilities in OS packages and language dependencies.

At the time of this writing, there are over 76 third-party [code scanning tools/workflows](https://github.com/Pwd9000-ML/terraform-azurerm-nsg-administration/actions/new?category=security&query=code+scanning) available for use, and the list is growing.

---

## 4. Dependabot Security/Dependency Updates

**[Dependabot Security Updates](https://docs.github.com/en/code-security/dependabot/dependabot-security-updates/about-dependabot-security-updates?wt.mc_id=DT-MVP-5004771)** is a security tool that handles your project **dependencies** by generating **alerts** for **vulnerabilities** and can also create pull requests to update them.

Dependabot Security Updates is a feature of **Dependabot**, which automates dependency updates not just for security, but also for non-security updates or out-of-date dependencies, keeping your project up to date.

For instance, if a new version of a dependency you're using is released that fixes a major security flaw, Dependabot would send an alert. It would also raise a pull request to update the dependency version in your project, keeping your project secure without requiring manual intervention.

For more information, refer to the [supported package ecosystems](https://docs.github.com/en/code-security/dependabot/dependabot-version-updates/configuration-options-for-the-dependabot.yml-file#package-ecosystem?wt.mc_id=DT-MVP-5004771) documentation.

You can also look at what dependencies are being monitored by **Dependabot** in your repository by navigating to the **Insights** tab in your repository.

---

## 5. Security Policies and Advisories at Contoso

At Contoso, we encourage developers to establish robust security policies and advisories. GitHub facilitates this by allowing anyone to report security vulnerabilities directly and privately to the maintainers.

- A **security policy** document helps contributors understand how to report a security vulnerability in your project. It's akin to creating a help page for a user who identifies a potential breach, thereby promoting responsible reporting.

- A **security advisory**, on the other hand, allows you to interact with users regarding identified vulnerabilities. For example, you could use an advisory to discuss a recently discovered flaw in your project, suggest a workaround, and preview a fix before public disclosure.

When private vulnerability reporting is enabled for a repository, security researchers will see a new button in the Advisories page of the repository. The security researcher can click this button to privately discuss, fix, and publish information about security vulnerabilities in your repository's code.

For more information, refer to **[Privately reporting a security vulnerability](https://docs.github.com/en/code-security/security-advisories/guidance-on-reporting-and-writing-information-about-vulnerabilities/privately-reporting-a-security-vulnerability?wt.mc_id=DT-MVP-5004771)**.

---

## Conclusion

GitHub's security features can significantly reduce the risk of your code being exploited. By using these tools in concert, you benefit from both proactive detection and resolution of potential vulnerabilities.

Moreover, the value of automating your code security cannot be overstated. With these automated features, you can manage vulnerabilities, dependencies, and other threats all in one place. The ability to find and fix issues before they become problematic means you can continue to write code confidently.

By harnessing the potential of GitHub's security features, you are taking a significant step towards a more secure coding environment at Contoso. Protecting your code is as crucial as writing it. Lean on GitHub's comprehensive suite of security tools and keep your projects safe and resilient.

---

### _Author_

Marcel Lupo - Follow me on: | [GitHub](https://github.com/Pwd9000-ML) | [X/Twitter](https://x.com/pwd9000) | [LinkedIn](https://www.linkedin.com/in/marcel-l-61b0a96b/)
