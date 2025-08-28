# Scrum Automation Workflow Documentation

## Overview

The **Scrum Automation** workflow is an n8n automation designed to enhance project tracking for Scrum teams. It integrates Jira for issue management, GitHub for code repository insights (commits, pull requests, and deployments), OpenAI for intelligent analysis of progress, and Slack for real-time notifications.

### Purpose

This workflow automates the monitoring of sprint progress by:

- Fetching active Jira issues.
- Analyzing GitHub commits and pull requests to map them to Jira acceptance criteria.
- Estimating time spent and calculating metrics like completed points, burndown, and velocity.
- Detecting blockers and generating aggregated reports.
- Sending contextual notifications to Slack for team awareness.

It runs on a daily schedule, providing a complete overview of project health, developer activity, and potential risks.

### Key Features

- **Multi-Tool Integration**: Combines Jira, GitHub, OpenAI (GPT-4 Turbo), and Slack.
- **AI-Powered Analysis**: Uses NLP to match commits/PRs to acceptance criteria and compute progress.
- **Metrics Calculation**: Tracks story points, burndown, velocity, and estimated time.
- **Blocker Detection**: Identifies stuck issues or low progress.
- **Notifications**: Sends formatted daily reports to a specified Slack channel.

### Workflow Flow

1. **Trigger**: Activates daily via Schedule Trigger.
2. **Data Fetch**:
   - Fetch Jira Issues: Retrieves details for a specific issue (e.g., WEB_APP_01).
   - Fetch GitHub Commits: Gets recent commits from the repository.
   - Fetch PRs: Retrieves open pull requests.
3. **Data Processing**:
   - Merge Git Data: Combines commits and PRs.
   - Merge Data: Multiplexes Jira issues with Git data.
   - LLM Criteria Mapper: Uses OpenAI to analyze progress and output JSON metrics.
4. **Analytics**:
   - Time Logger: Estimates time spent based on Jira logs or commit count.
   - Analytics Aggregator: Computes aggregate metrics (total points, completed, burndown, velocity).
   - Deployment Track: Fetches GitHub workflow runs for deployment status.
   - Blocker Detector: Identifies issues with low progress or blockers.
5. **Reporting**:
   - Merge Reports: Combines analytics and blockers.
   - Slack Notifier: Sends a formatted report to Slack.

## Setup Steps

To set up and run this workflow in n8n:

1. **Import the Workflow**:

   - Log in to your n8n instance.
   - Go to the Workflows tab and click "Import from File" or "Import from URL".
   - Upload the `workflow.json` file or paste its contents.
   - Save the workflow.

2. **Configure Credentials**:

   - **Jira**: Create a "Jira Software Cloud API" credential with your Jira API token and base URL.
   - **GitHub**: Set up a "GitHub API" credential with your GitHub personal access token (scopes: repo, workflow).
   - **OpenAI**: Add an "OpenAI API" credential with your OpenAI API key.
   - **Slack**: Create a "Slack OAuth2 API" credential by authorizing n8n in your Slack workspace (scopes: chat:write, channels:read).

3. **Update Node Configurations**:

   - **Fetch Jira Issues**: Update the `issueKey` to your target Jira issue (e.g., change "WEB_APP_01" to match your project). For multiple issues, switch to "search" operation and use a JQL query like `project = YOUR_PROJECT AND status = "In Progress"`.
   - **Fetch GitHub Commits** and **Fetch PRs**: Select your repository owner and name from the dropdown (ensure credentials are set).
   - **Deployment Track**: Similarly, select owner and repository.
   - **Slack Notifier**: Verify the `channelId` points to your desired Slack channel (e.g., #dailystatus).
   - **OpenAI Chat Model**: Ensure the model is set to "gpt-4-turbo-preview" (or adjust based on availability).
   - Review Code nodes (e.g., Time Logger, Analytics Aggregator) for custom fields like `customfield_10026` (story points) and adjust if your Jira setup differs.

4. **Test the Workflow**:

   - Click "Execute Workflow" manually to test.
   - Check for errors in node executions and verify the Slack message is received.
   - Activate the workflow to enable the daily schedule.

5. **Customize Schedule**:

   - In the "Daily Schedule" node, adjust the cron expression or interval to your preferred timing (e.g., every weekday at 9 AM).

6. **Deploy**:
   - Ensure n8n is running in a production environment (e.g., self-hosted or cloud) with proper error handling.

## Best Practices

- **Customization**:

  - Extend the JQL in Jira nodes to filter by sprint, assignee, or labels for more targeted tracking.
  - Modify the OpenAI prompt in "LLM Criteria Mapper" to include team-specific criteria or additional context.
  - Add branching logic (e.g., IF nodes) to handle different issue types or priorities.

- **Monitoring and Maintenance**:

  - Regularly review n8n execution logs for failures (e.g., API rate limits).
  - Set up n8n's error notifications to alert you of workflow issues.
  - Periodically update API credentials and test integrations.

- **Performance**:

  - For large repositories, limit GitHub fetches (e.g., add `perPage` or date filters).
  - Use n8n's looping for handling paginated API responses if needed.

- **Usage Tips**:
  - Encourage developers to reference Jira keys in commit messages/PR titles for accurate mapping.
  - Combine with other n8n workflows for full CI/CD automation.
  - Analyze reports weekly to adjust sprint planning.

## Security Recommendations

- **Credential Management**:

  - Store API keys and tokens in n8n's encrypted credentials store—never hardcode them in nodes.
  - Use OAuth where possible (e.g., for GitHub and Slack) to avoid long-lived tokens.

- **Least Privilege Principle**:

  - Limit GitHub token scopes to only what's needed (e.g., read:repo, read:workflow).
  - In Jira, use API tokens with read-only access if analysis doesn't require updates.
  - Restrict OpenAI API key to your n8n instance only.

- **Data Handling**:

  - Avoid logging sensitive data (e.g., disable verbose logging in production).
  - Ensure Slack notifications don't expose confidential information—sanitize outputs in Code nodes.

- **Access Control**:

  - Secure your n8n instance with authentication and HTTPS.
  - Use n8n's role-based access to limit who can edit or execute the workflow.

- **Compliance**:
  - If handling personal data, comply with GDPR or similar by minimizing stored data.
  - Regularly rotate API keys and monitor for unusual activity.

For questions or enhancements, refer to n8n documentation or contact the workflow maintainer.

## Comprehensive AI Prompt Example

For the "LLM Criteria Mapper" node, which uses OpenAI to analyze progress, here's a comprehensive prompt template you can use or adapt. This prompt is designed to provide semantic understanding, handle edge cases, and output structured JSON for downstream processing.

### Prompt Template

```
You are an expert Scrum analyst with deep knowledge of software development workflows. Your task is to evaluate Jira issue progress based on provided data.

### Input Data:
- Jira Issue Key: {{ $json.key }}
- Summary: {{ $json.fields.summary }}
- Description: {{ $json.fields.description }}
- Story Points: {{ $json.fields.customfield_10026 || 0 }} (assume 0 if not provided)
- Commits and PRs: {{ $json.gitData }} (this is a list of commit messages, PR titles, and descriptions; only consider those mentioning the issue key)

### Analysis Steps:
1. **Extract Acceptance Criteria**: Parse the Jira description to identify explicit acceptance criteria. If none are listed, infer logical criteria based on the summary and description (aim for 3-5 criteria if possible). List them clearly.
2. **Match Git Activity**: For each criterion, semantically match relevant commits or PRs that indicate work done. Consider:
   - Keywords like "fix", "implement", "add", "test".
   - Context: Does the commit/PR description align with the criterion?
   - Ignore unrelated activity.
3. **Calculate Progress**:
   - Total Criteria: Count of extracted/inferred criteria.
   - Points per Criterion: Divide total story points evenly (e.g., if 5 points and 5 criteria, each is 1 point).
   - Completed Criteria: Count how many are fully satisfied based on matches.
   - Completed Points: Sum points for completed criteria (use fractions if partial matches).
   - Pending Points: Total points - completed points.
   - Percentage: (Completed points / Total points) * 100, rounded to nearest integer.
4. **Identify Blockers and Reviews**:
   - Blockers: Look for terms like "blocked", "issue", "error" in recent activity or if no progress in days.
   - Review Needs: If PRs are open without reviews or if commits lack testing mentions.
   - Output as an array of strings, e.g., ["Needs code review", "Potential blocker: dependency issue"].
5. **Edge Cases**:
   - If no criteria can be extracted, assume progress based on commit volume (e.g., 20% per relevant commit, capped at 100%).
   - If no git data, set completed to 0.
   - Handle malformed data gracefully.

### Output Format:
Respond ONLY with valid JSON:
{
  "completedPoints": number,
  "pendingPoints": number,
  "percentage": number,
  "blockers": array of strings,
  "criteriaAnalysis": array of objects like { "criterion": string, "matchedActivity": array of strings, "status": "completed" | "partial" | "pending" }
}

Ensure the output is precise, data-driven, and unbiased.
```

### Customization Tips

- **Enhance Specificity**: Add team-specific keywords or patterns (e.g., commit conventions like "feat:" or "fix:").
- **Improve Accuracy**: Test with sample data and iterate on the prompt to reduce hallucinations.
- **Cost Optimization**: Keep the prompt concise while maintaining comprehensiveness to minimize token usage.
- **Integration**: Copy this into the "text" parameter of the LLM Criteria Mapper node, ensuring variables like {{ $json.key }} match your data structure.
