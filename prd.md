# Product Requirements Document

**Product Name:** Sprinter  
**Repository:** https://github.com/co0p/Sprinter  
**Owner:** co0p  
**Date:** 2026-01-11

---

## Overview

Sprinter is a sprint dashboard and monitoring tool that provides real-time visibility into sprint health, progress, and risk. It integrates with Jira (as the source of truth for sprint scope and story points) and GitHub (for pull request flow metrics) to give engineering teams actionable insights throughout the sprint lifecycle.

The dashboard surfaces key metrics, identifies bottlenecks, and highlights risk signals that may impact sprint delivery, enabling teams to take corrective action early.

---

## Problem Statement

Engineering teams running agile sprints face several challenges:

- **Limited visibility**: Sprint progress is scattered across Jira, GitHub, and team discussions, making it hard to get a holistic view
- **Reactive vs. proactive management**: Issues are often discovered too late when scope creep, blocked work, or PR bottlenecks have already impacted delivery
- **Manual tracking overhead**: Calculating burn rates, tracking scope changes, and monitoring PR flow requires manual effort and is error-prone
- **Lack of standardized risk signals**: Teams lack consistent, data-driven metrics to identify when a sprint is at risk

Sprinter addresses these challenges by consolidating sprint data from multiple sources and providing automated risk detection with clear, actionable insights.

---

## Goals / Non-Goals

### Goals

- Provide a single dashboard for sprint health monitoring
- Automatically track sprint metrics using Jira and GitHub as data sources
- Surface risk signals early with clear, explainable reasons
- Enable configurable thresholds and sprint parameters to fit different team workflows
- Support the full sprint lifecycle: draft, active, and closed sprints
- Track scope changes and provide visibility into sprint churn

### Non-Goals

- Replace Jira or GitHub as primary tools
- Provide sprint planning or estimation features
- Support CI/CD orchestration or deployment workflows (CI health is optional monitoring only)
- Multi-project portfolio management
- Time tracking or capacity planning beyond story points

---

## Personas & Use Cases

### Personas

1. **Scrum Master / Agile Coach**
   - Needs: Real-time sprint health visibility, risk identification, data for retrospectives
   - Uses dashboard to: Monitor progress, identify blockers, prepare standup discussions

2. **Engineering Manager**
   - Needs: Team performance insights, bottleneck identification, sprint predictability
   - Uses dashboard to: Track delivery trends, identify process improvements, support team

3. **Individual Contributor / Developer**
   - Needs: PR queue visibility, awareness of sprint status
   - Uses dashboard to: See review requests, understand sprint priorities, check CI health

### Use Cases

- **UC-1**: Monitor active sprint progress and compare actual vs. required burn rate
- **UC-2**: Identify PRs that need review attention (time to first review, stale PRs)
- **UC-3**: Detect scope creep by tracking committed vs. current story points
- **UC-4**: Get alerted when sprint is at risk (behind pace, high churn, blocked work)
- **UC-5**: Review historical sprint data for retrospectives and planning
- **UC-6**: Configure sprint parameters (duration, story points field, thresholds) per team

---

## Requirements

### Sprint Lifecycle Management

- **Sprint States**:
  - **Draft**: Sprint is being planned, scope not finalized
  - **Active**: Sprint is in progress, tracking metrics actively
  - **Closed**: Sprint is complete, data is historical

- **Baseline Snapshot**: When a sprint moves from draft to active, capture the committed scope (issues and story points) as the baseline

- **Scope Changes Log**: Track all changes to sprint scope after baseline:
  - Issues added (scope creep)
  - Issues removed (scope reduction)
  - Story point changes
  - Timestamp and reason (if available)

### Configurable Sprint Duration

- Support two configuration modes:
  - **Date-based**: Explicit start date and end date
  - **Duration-based**: Start date + duration in days (e.g., 2 weeks)
  
- Display days remaining in active sprints
- Support sprint timezone configuration

### Jira Integration

- **Source of Truth**: Jira is the authoritative source for:
  - Sprint scope (which issues are in the sprint)
  - Story points per issue
  - Issue status (To Do, In Progress, Done, Blocked, etc.)
  
- **Authentication**: Support Jira API token authentication
  
- **Configurable Story Points Field**: Allow configuration of custom field ID for story points (e.g., `customfield_10016`) as this varies by Jira instance
  
- **Required Configuration**:
  - Jira instance URL
  - API token or credentials
  - Project key(s)
  - Story points custom field ID
  - Sprint board ID or JQL filter
  
- **Data Refresh**: Configurable polling interval (default: 5 minutes)

### GitHub Integration

- **Pull Request Metrics**: Track PR flow for work associated with sprint issues
  
- **Mapping**: Map Jira issues to GitHub PRs using Jira key convention in PR title or branch name (e.g., `PROJECT-123`)
  
- **Authentication**: Support GitHub personal access token or GitHub App
  
- **Required Configuration**:
  - GitHub repository owner and name(s)
  - Authentication token
  - PR label filters (optional)
  
- **Tracked PR Data**:
  - PR state (open, merged, closed)
  - Created timestamp
  - First review timestamp
  - Merged timestamp
  - Number of reviews
  - Reviewer list
  - CI status (optional)
  
- **Data Refresh**: Configurable polling interval (default: 5 minutes)

### CI Health (Optional)

- Track CI/CD pipeline success rate for sprint-related PRs
- Surface failing builds that may block PR merges
- Configurable: teams can opt-in to CI monitoring

---

## Metrics & Risk Signals

**Note on Edge Cases**: All metric formulas should handle division by zero gracefully. When denominators are zero (e.g., Days Elapsed = 0, Days Remaining = 0, Committed Points = 0, Total PRs = 0), the implementation should return appropriate default values (0, null, or "N/A") or skip the metric calculation until sufficient data is available.

### Story Point Metrics

Define and track the following metrics based on Jira data:

1. **Committed Points**: Story points in the baseline snapshot when sprint became active
   - Formula: `Sum of story points for all issues in baseline`

2. **Done Points**: Story points for issues marked as Done
   - Formula: `Sum of story points for issues with status = "Done" or equivalent`

3. **Remaining Points**: Story points for issues not yet Done
   - Formula: `Committed Points - Done Points + Added Points - Removed Points`

4. **Burn Rate**: Actual rate of story point completion
   - Formula: `Done Points / Days Elapsed`

5. **Required Rate**: Rate needed to complete remaining points by sprint end
   - Formula: `Remaining Points / Days Remaining`

6. **Scope Churn Ratio**: Percentage of baseline scope that has changed
   - Formula: `(|Added Points| + |Removed Points|) / Committed Points * 100`
   - Note: Uses absolute values to measure total scope change magnitude

7. **Blocked Ratio**: Percentage of current sprint scope that is blocked
   - Formula: `Blocked Points / (Committed Points + Added Points - Removed Points) * 100`
   - Note: Denominator represents total current scope

### GitHub PR Metrics

Track the following PR flow metrics:

1. **Open PR Count**: Number of open PRs associated with sprint issues
   - Formula: `Count of open PRs linked to sprint Jira issues`

2. **Time to First Review (Median)**: Median time from PR open to first review
   - Formula: `Median of (First Review Timestamp - PR Created Timestamp)` for PRs in sprint

3. **Time to First Review (P90)**: 90th percentile time to first review
   - Formula: `P90 of (First Review Timestamp - PR Created Timestamp)` for PRs in sprint

4. **Cycle Time**: Median time from PR open to merge
   - Formula: `Median of (PR Merged Timestamp - PR Created Timestamp)` for merged PRs in sprint

5. **Review Queue per Reviewer**: Number of open PRs awaiting review per reviewer
   - Formula: `Count of open PRs where reviewer is requested and has not submitted a review`
   - Note: Based on GitHub's review request feature

6. **CI Failing Rate** (if CI monitoring enabled): Percentage of PRs with failing CI checks
   - Formula: `Count of PRs with failing CI / Total PRs * 100`

### Risk Thresholds & Scoring

Define configurable thresholds with default values:

| Metric | Green (Low Risk) | Yellow (Medium Risk) | Red (High Risk) |
|--------|------------------|----------------------|-----------------|
| Required Rate vs. Burn Rate | Required ≤ Burn Rate * 1.1 | Burn Rate * 1.1 < Required ≤ Burn Rate * 1.5 | Required > Burn Rate * 1.5 |
| Scope Churn Ratio | < 10% | 10% - 25% | > 25% |
| Blocked Ratio | < 10% | 10% - 20% | > 20% |
| Time to First Review (Median) | < 4 hours | 4 - 24 hours | > 24 hours |
| Time to First Review (P90) | < 24 hours | 24 - 48 hours | > 48 hours |
| Open PR Count | < 5 | 5 - 10 | > 10 |
| CI Failing Rate | < 10% | 10% - 30% | > 30% |

**Note**: When burn rate is zero (no points completed), use days-based comparison: if days elapsed exceeds the configured zero burn rate threshold (default: 20% of sprint duration) with no points done, mark as red. This threshold is configurable in the thresholds section.

**Overall Sprint Risk Score**:
- **Green**: No red signals, ≤ 1 yellow signal
- **Yellow**: 2-3 yellow signals OR 1 red signal
- **Red**: ≥ 2 red signals

**Explainable Reasons**: For each risk signal, provide clear explanation:
- Example: "Required rate (5.5 pts/day) is 60% higher than current burn rate (3.4 pts/day). At current pace, sprint will miss target by 8 points."
- Example: "Scope has changed by 22% since sprint start (3 issues added, 1 removed). This increases uncertainty."
- Example: "Median time to first review is 18 hours. PRs are sitting idle, which may cause delays."

---

## Acceptance Criteria

- [ ] Dashboard displays current sprint status with all defined metrics
- [ ] Jira integration successfully fetches sprint scope and story points using configurable field ID
- [ ] GitHub integration successfully fetches PR data and maps to Jira issues
- [ ] Sprint baseline is captured when moving from draft to active
- [ ] Scope changes are logged with timestamps
- [ ] Risk signals are calculated correctly using defined formulas
- [ ] Risk thresholds are configurable (overridable from defaults)
- [ ] Overall sprint risk score is displayed with RAG status and explainable reasons
- [ ] Dashboard supports date-based and duration-based sprint configuration
- [ ] Data refreshes at configured intervals
- [ ] Dashboard shows historical data for closed sprints
- [ ] Configuration can be provided via file or environment variables
- [ ] Dashboard is accessible via web interface

---

## Configuration

### Configuration File Format

Support configuration via YAML or JSON file:

```yaml
jira:
  instance_url: "https://your-company.atlassian.net"
  email: "user@company.com"
  api_token: "${JIRA_API_TOKEN}"
  project_key: "TEAM"
  board_id: 123
  story_points_field: "customfield_10016"
  status_mapping:
    done: ["Done", "Closed"]
    blocked: ["Blocked", "Impediment"]
  refresh_interval_seconds: 300

github:
  owner: "your-org"
  repos: ["repo1", "repo2"]
  token: "${GITHUB_TOKEN}"
  label_filters: ["sprint"]
  enable_ci_monitoring: true
  refresh_interval_seconds: 300

sprint:
  name: "Sprint 42"
  timezone: "America/Los_Angeles"
  # Option 1: Date-based (use explicit start and end dates)
  start_date: "2026-01-06"
  end_date: "2026-01-20"
  # Option 2: Duration-based (use start date + duration; mutually exclusive with Option 1)
  # start_date: "2026-01-06"
  # duration_days: 14

thresholds:
  scope_churn:
    yellow: 10
    red: 25
  blocked_ratio:
    yellow: 10
    red: 20
  time_to_first_review_hours:
    yellow: 4
    red: 24
  pr_count:
    yellow: 5
    red: 10
  zero_burn_rate_threshold_percent: 20  # Mark as red if no points done after this % of sprint elapsed

dashboard:
  port: 8080
  title: "Sprinter - Sprint Dashboard"
```

### Environment Variables

Support sensitive values via environment variables:
- `JIRA_API_TOKEN`
- `GITHUB_TOKEN`
- `SPRINTER_CONFIG_PATH`

---

## Future Enhancements

### Phase 2
- Multiple active sprints support (team of teams)
- Slack/Teams notifications for risk signals
- Burndown chart visualization
- Velocity trend analysis across sprints
- Custom JQL filters for advanced sprint queries

### Phase 3
- Predictive analytics (sprint success probability based on current metrics)
- GitHub Actions workflow integration for automated sprint reports
- Export sprint reports to PDF/CSV
- Team capacity tracking (planned vs. actual availability)
- Integration with additional tools (GitLab, Bitbucket, Azure DevOps)

### Phase 4
- Mobile responsive dashboard
- Browser extension for quick sprint status
- Sprint retrospective template generation based on metrics
- A/B testing for threshold tuning recommendations
- Machine learning for personalized risk thresholds per team
