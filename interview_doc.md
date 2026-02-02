# GitHub Actions Viva/Interview Questions

This document is designed for a viva or technical interview session on GitHub Actions. It covers 10 key topics, with 5 questions per topic, ranging from basic concepts to advanced scenarios.

---

## Topic 1: Core Concepts & Architecture

1.  **Q:** What are the four primary components that make up a GitHub Actions workflow hierarchy?
    *   **A:** Workflow (the file) -> Job (execution unit) -> Step (individual task) -> Action (reusable command) or Shell Script.

2.  **Q:** Where must GitHub Actions workflow files be located in a repository for them to be recognized?
    *   **A:** In the `.github/workflows/` directory.

3.  **Q:** What is the difference between an "Action" and a "Workflow"?
    *   **A:** A *Workflow* is the automated process defined in a YAML file. An *Action* is a reusable piece of code (like a function) used *within* a step of a workflow.

4.  **Q:** Can a single workflow file contain multiple jobs? If so, do they run sequentially or in parallel by default?
    *   **A:** Yes, it can have multiple jobs. By default, they run in **parallel**.

5.  **Q:** What is the file extension used for workflow configuration?
    *   **A:** `.yml` or `.yaml`.

---

## Topic 2: Events & Triggers (`on:`)

6.  **Q:** How do you trigger a workflow to run only when code is pushed to the `main` branch?
    *   **A:** Use `on: push: branches: [ "main" ]`.

7.  **Q:** What is the `workflow_dispatch` event used for?
    *   **A:** It allows you to manually trigger the workflow from the GitHub UI (Actions tab), optionally with input parameters.

8.  **Q:** Can you trigger a workflow on a schedule? If so, what syntax is used?
    *   **A:** Yes, using `on: schedule: - cron: '...''`. It uses POSIX cron syntax.

9.  **Q:** How would you skip running a workflow if only documentation files (`*.md`) were changed in a push?
    *   **A:** Use `paths-ignore: ['*.md']` or `paths: ['src/**']` under the `push` event.

10. **Q:** What is the difference between `pull_request` and `pull_request_target` events?
    *   **A:** `pull_request` runs in the context of the merge commit (safe code). `pull_request_target` runs in the context of the base branch (privileged context), which is dangerous if untrusted code allows script injection.

---

## Topic 3: Runners (Hosted vs. Self-Hosted)

11. **Q:** What is a "Runner" in GitHub Actions?
    *   **A:** The server or virtual machine that executes the jobs in your workflow.

12. **Q:** Name three operating systems provided by GitHub-hosted runners.
    *   **A:** Ubuntu (Linux), Windows Server, and macOS.

13. **Q:** Why might a company choose to use Self-Hosted runners instead of GitHub-hosted ones?
    *   **A:** To access internal network resources (VPN/LAN), to use specialized hardware (GPUs), or to avoid per-minute billing costs on private repositories.

14. **Q:** What is the security risk of using Self-Hosted runners on public repositories?
    *   **A:** A malicious pull request could execute code on your server, potentially stealing secrets or persisting malware on your infrastructure.

15. **Q:** How do you specify which runner a job should execute on?
    *   **A:** Use the `runs-on:` key (e.g., `runs-on: ubuntu-latest`).

---

## Topic 4: Jobs & Dependencies

16. **Q:** How do you force Job B to wait until Job A finishes successfully before starting?
    *   **A:** Use the `needs: [JobA]` keyword in Job B's definition.

17. **Q:** What is a "Matrix Strategy" (`strategy: matrix`) used for?
    *   **A:** To automatically run the same job multiple times with different variables (e.g., testing against Node.js versions 14, 16, and 18 simultaneously).

18. **Q:** If one job in a matrix fails, what happens to the others by default?
    *   **A:** By default, GitHub cancels the remaining matrix jobs (`fail-fast: true`). You can set `fail-fast: false` to let them finish.

19. **Q:** How can you set a timeout for a job to prevent it from hanging forever?
    *   **A:** Use the `timeout-minutes:` key.

20. **Q:** Can jobs pass data to each other?
    *   **A:** Not directly via variables. They must use **artifacts** (upload/download) or **outputs** defined at the job level.

---

## Topic 5: Variables, Contexts, & Secrets

21. **Q:** How do you access a secret named `API_KEY` in your workflow?
    *   **A:** `${{ secrets.API_KEY }}`.

22. **Q:** What is the difference between `env` defined at the top level vs. inside a job?
    *   **A:** Top-level `env` is available to all jobs in the workflow. Job-level `env` is only available to steps within that specific job.

23. **Q:** What is the `GITHUB_TOKEN`?
    *   **A:** An automatic authentication token provided by GitHub for each workflow run, allowing the workflow to interact with the repository (push code, create releases, comment on PRs).

24. **Q:** What is the syntax to evaluate an expression (like checking an event name)?
    *   **A:** `${{ <expression> }}` (e.g., `${{ github.event_name == 'push' }}`).

25. **Q:** How do you mask a value in the logs so it doesn't show up in plain text?
    *   **A:** You access it via `${{ secrets... }}` which masks automatically, or use the `::add-mask::` workflow command for generated values.

---

## Topic 6: Actions Composition

26. **Q:** What is the `uses:` keyword for?
    *   **A:** To reference and execute a pre-built Action (e.g., `uses: actions/checkout@v3`).

27. **Q:** How do you pass input parameters to an Action (like a version number)?
    *   **A:** Use the `with:` keyword.

28. **Q:** What are the three types of custom Actions you can create?
    *   **A:** JavaScript Actions, Docker Container Actions, and Composite Actions.

29. **Q:** Which action is almost always the first step in a workflow, and why?
    *   **A:** `actions/checkout`. Because the runner starts empty; you need to check out your repository code to build or test it.

30. **Q:** What format is a "Composite Action" written in?
    *   **A:** It is written efficiently in a single `action.yml` file using shell steps, without needing a Dockerfile or compiled JS.

---

## Topic 7: Artifacts & Caching

31. **Q:** How do you persist a build file (like a `.jar` or binary) so it can be used after the workflow finishes?
    *   **A:** Use the `actions/upload-artifact` action.

32. **Q:** What is the primary purpose of `actions/cache`?
    *   **A:** To store dependencies (like `node_modules` or `.m2` repository) between runs to speed up build times.

33. **Q:** How does the cache action decide if it needs to restore or save a new cache?
    *   **A:** It uses a `key`. If the key matches an existing cache, it restores. If keys change (e.g., `package-lock.json` hash changes), it saves a new cache.

34. **Q:** Can Artifacts be shared between different jobs in the same workflow?
    *   **A:** Yes. Job A uploads the artifact, and Job B (which `needs: JobA`) downloads it.

35. **Q:** Are artifacts stored forever?
    *   **A:** No, they have a default retention period (usually 90 days), which can be customized.

---

## Topic 8: Security & Permissions

36. **Q:** How can you verify that a workflow is referencing a secure version of a third-party Action?
    *   **A:** Pin the Action to a specific commit SHA (e.g., `@a1b2c3d`) instead of a tag (`@v1`) which can be overwritten.

37. **Q:** What is "OIDC" (OpenID Connect) in the context of GitHub Actions and AWS/Azure?
    *   **A:** It allows GitHub Actions to authenticate with Cloud Providers without storing long-lived separate secrets (Access Keys) in GitHub. It uses short-lived tokens.

38. **Q:** How do you restrict the `GITHUB_TOKEN` permissions to read-only?
    *   **A:** Define `permissions: read-all` or specific scopes (e.g., `contents: read`) at the top of the workflow file.

39. **Q:** Can you approve a deployment manually before it proceeds to Production?
    *   **A:** Yes, by using **Environments** and configuring "Required Reviewers" in the repository settings.

40. **Q:** What happens if a workflow attempts to print a secret to the console?
    *   **A:** GitHub Actions detects the secret string and automatically redacts it, replacing it with `***`.

---

## Topic 9: Reusable Workflows

41. **Q:** What trigger event allows a workflow to be called by other workflows?
    *   **A:** `on: workflow_call`.

42. **Q:** Why use Reusable Workflows instead of copying YAML files?
    *   **A:** Standardization (DRY principle). You can maintain one central deployment logic and have 50 repositories reuse it.

43. **Q:** Can a caller workflow pass secrets to the called workflow?
    *   **A:** Yes, using the `secrets:` keyword when invoking the reusable workflow (`secrets: inherit` usually).

44. **Q:** Where can reusable workflows be stored?
    *   **A:** In the same repository, or in a public/internal repository that is accessible to the caller.

45. **Q:** Does a reusable workflow run in the caller's context or its own context?
    *   **A:** It runs in the context of the **caller** repository (e.g., it checks out the caller's code).

---

## Topic 10: Monitoring & Debugging

46. **Q:** How do you enable "Step Debug Logging" to see detailed verbose output?
    *   **A:** Set a repository secret named `ACTIONS_STEP_DEBUG` to `true`.

47. **Q:** What is a "Status Check" on a Pull Request?
    *   **A:** A requirement that a specific workflow job must succeed (green checkmark) before the PR can be merged.

48. **Q:** How can you generate a Markdown summary of the build results on the Actions summary page?
    *   **A:** Write to the `$GITHUB_STEP_SUMMARY` environment file (e.g., `echo "### Results" >> $GITHUB_STEP_SUMMARY`).

49. **Q:** If a step fails, how do you force the next step to run anyway (e.g., to upload logs)?
    *   **A:** Use the conditional `if: always()` or `if: failure()` on the subsequent step.

50. **Q:** Can you ssh into a runner to debug a live failure?
    *   **A:** Not natively by default, but you can use third-party actions (like `tmate`) to establish a debug SSH session during the build.
