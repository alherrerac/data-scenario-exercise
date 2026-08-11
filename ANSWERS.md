# Technical Exercise: Support Data Scenario _ ANSWERS

Thanks for taking the time to work through this technical exercise. It's meant to reflect the kind of data work that comes up as a team lead. Note: nothing here requires advanced SQL, just clear thinking through a realistic situation.

**Time estimate:** 30–45 minutes.

## How to Submit

1. Fork this repo (or clone the gist).
2. Add your answers into a copy of this file (`ANSWERS.md`), with your SQL and short answers inline.
3. Submit the link to your fork (or your branch/PR) using the form link provided.

You're welcome to use whatever tools you'd normally reach for on the job. However, please be ready to walk through your reasoning afterward.

---

## Sample Schema

Assume three tables from a support ticketing system:

**tickets**
| Column | Type | Description |
|---|---|---|
| ticket_id | integer | Primary key |
| agent_id | integer | Foreign key to agents.agent_id |
| category | text | e.g. Billing, Technical, Account, Shipping |
| priority | text | Low, Medium, High, Urgent |
| status | text | open, closed, reopened |
| opened_at | timestamp | When the ticket was created |
| closed_at | timestamp | When the ticket was closed (null if still open) |
| first_response_minutes | integer | Minutes until the first agent reply |
| reopened_count | integer | Number of times the ticket was reopened after closing |

**agents**
| Column | Type | Description |
|---|---|---|
| agent_id | integer | Primary key |
| name | text | Agent name |
| team | text | Team the agent sits on |
| hire_date | date | Date the agent started |

**csat_responses**
| Column | Type | Description |
|---|---|---|
| response_id | integer | Primary key |
| ticket_id | integer | Foreign key to tickets.ticket_id |
| score | integer | 1–5 satisfaction score |
| submitted_at | timestamp | When the customer submitted the score |

---

## The Scenario

You're leading a small support team. Leadership has noticed that CSAT scores for the **Billing** category dropped 12% last month. Ticket volume in Billing stayed roughly flat over the same period, and nothing changed in staffing. You've been asked to look into it and report back.

Work through the following as you would if this landed in your inbox tomorrow.

**1. First response time by team**
Write a query returning the average `first_response_minutes` for tickets closed in the last 30 days, grouped by agent team

```
SELECT  
    a.team,  
    AVG(t.first_response_minutes) AS avg_first_response_minutes  
FROM tickets t  
JOIN agents a  
    ON t.agent_id = a.agent_id  
    WHERE t.closed_at >= CURRENT_DATE - INTERVAL '30 days'  
GROUP BY a.team;
```

**2. Agents with above-average reopen rates**
Write a query returning each agent's reopen rate (reopened tickets ÷ total tickets they handled) for agents whose rate is higher than their own team's average.

```
WITH agent_stats AS (
    SELECT 
        a.agent_id,
        a.name AS agent_name,
        a.team,
        COUNT(t.ticket_id) AS total_tickets,
        COUNT(CASE WHEN t.reopened_count > 0 THEN 1 END)::FLOAT / NULLIF(COUNT(t.ticket_id), 0) AS agent_reopen_rate
    FROM agents a
    JOIN tickets t 
        ON a.agent_id = t.agent_id
    GROUP BY a.agent_id, a.name, a.team
),
team_stats AS (
    SELECT 
        team,
        AVG(agent_reopen_rate) AS team_avg_reopen_rate
    FROM agent_stats
    GROUP BY team
)
SELECT 
    ast.agent_id,
    ast.agent_name,
    ast.team,
    ast.agent_reopen_rate,
    tst.team_avg_reopen_rate
FROM agent_stats ast
JOIN team_stats tst 
    ON ast.team = tst.team
WHERE ast.agent_reopen_rate > tst.team_avg_reopen_rate;
```

**3. CSAT trend by category**
Write a query returning the average CSAT score per category, per month, for the last 3 months.

```
SELECT 
    t.category,
    DATE_TRUNC('month', c.submitted_at) AS csat_month,
    ROUND(AVG(c.score), 2) AS avg_csat_score
FROM csat_responses c
JOIN tickets t 
    ON c.ticket_id = t.ticket_id
WHERE c.submitted_at >= CURRENT_DATE - INTERVAL '3 months'
GROUP BY 1, 2
ORDER BY t.category, csat_month DESC;
```

**4. Digging in**
Beyond the three tables above, what additional data would you want to pull to understand the Billing drop — and which of the existing tables would you start with, and why?

From the tickets table, I would query on the Billing category and the priority of tickets. Then I would correlate this data with corresponding CSAT scores for this Billing category looking for additional information. It will be important to get additional data from CSAT including customer name and reason/issue type for their scores. Additional helpful data would be to identify agents managing Billing cases opened and reopened, to rule out any specific personal situation or challenge with agents.



**5. Testing a theory**
Pick one specific theory for what might be driving the drop. State your theory, then write a query using the tables above that would help confirm or rule it out.

Maybe there is a specific issue type that occurred in the last month and that impacted customer's business operations severely. If this is the case, customers may have open tickets with higher priority than in the past and may have scored their CSAT lower as well. I would query the data as follows looking to confirm a trend in priority, and CSAT, for tickets opened in the Billing category:

```
SELECT 
    AVG(
        CASE t.priority
            WHEN 'Low' THEN 1
            WHEN 'Medium' THEN 2
            WHEN 'High' THEN 3
            WHEN 'Urgent' THEN 4
        END
    ) AS avg_priority_score,
    AVG(c.score) AS avg_csat_score
FROM tickets t
LEFT JOIN csat_responses c 
    ON t.ticket_id = c.ticket_id
WHERE t.category = 'Billing'
  AND t.opened_at >= CURRENT_DATE - INTERVAL '3 months';
```


**6. What you'd actually do**
Say your query in Question 5 showed that reopened tickets in Billing spiked, and most of the reopens trace back to one specific issue type (e.g., refund timing questions). What would you actually do with that in the next week? Be concrete. What would you say to the team, what (if anything) would you change in a process or macro, and how would you know if it worked?

I would work with internal stakeholders (Customer Success, Delivery Managers, and the refund product/service/IT owner ) to share my findings, make a root cause analysis, create a service restoration plan, and agree on a communication statement for customers. I would share this information with the team to in turn share the communication statement in their customer interactions. I would also implement a quick process to capture and flag tickets being opened for Billing, trying to avoid reopenings, while the restoration plan is implemented. I would expect to see Billing tickets with less priority, less tickets reopened and CSAT scores recovery.



**7. Reporting up and coaching down**
You need to update your own manager on this in two sentences, and separately coach one agent on it in a 1:1. How would those two conversations differ?

For my manager: Regarding the 12% drop of CSAT for Billing during the last month, I can confirm there is an specific issue with refund timing. There is a service restoration plan in place, agreed with the refund service and Customer Success teams, and we should expect Billing tickets to decrease and CSAT to recover in the following weeks.

For my agent: Coaching on looking for systemic issues, trends in volume of tickets opened and reopened, identify recurring situations within categories and related lesson learned.

The conversations would differ in the level and content of the communications. My leadership would need to know facts, business impact and response actions. Whereas agents need to understand tactics and the value their work can provide to overall business operations.

---

*We're less interested in a "correct" answer than in how you think through ambiguity *
