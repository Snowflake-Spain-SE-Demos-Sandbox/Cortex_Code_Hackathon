# 📺 Media & Audience Intelligence Copilot
**DB:** `MEDIA_COPILOT`

---

## FASE 0 — Setup

**[CoCo]**
```
Create a database called MEDIA_COPILOT with four schemas: BRONZE, SILVER, GOLD and AI.
Use warehouse COMPUTE_WH.
```

**[✓ SQL]**
```sql
SHOW SCHEMAS IN DATABASE MEDIA_COPILOT;
```

---

## FASE 1 — Bronze

**[CoCo]** Git Integration
```
Create a git_https_api API Integration called GIT_HACKATHON_INTEGRATION
and a Git Repository in MEDIA_COPILOT.PUBLIC called HACKATHON_REPO pointing to:
https://github.com/Snowflake-Spain-SE-Demos-Sandbox/Cortex_Code_Hackathon.git
No authentication required (public repository).
```

**[✓ SQL]**
```sql
ALTER GIT REPOSITORY MEDIA_COPILOT.PUBLIC.HACKATHON_REPO FETCH;
LIST @MEDIA_COPILOT.PUBLIC.HACKATHON_REPO/branches/main/data/media/;
```

**[CoCo]** Bronze Tables
```
In MEDIA_COPILOT.BRONZE create the following tables with a SRC VARIANT column
and load them from the Git repo using COPY INTO (FILE_FORMAT JSON, STRIP_OUTER_ARRAY=TRUE):
- RAW_AUDIENCES from data/media/audiences.json
- RAW_ENGAGEMENT from data/media/engagement_events.json
- RAW_FEEDBACK from data/media/content_feedback.json
- RAW_CAMPAIGNS from data/media/campaigns.json
```

**[✓ SQL]**
```sql
SELECT COUNT(*) FROM MEDIA_COPILOT.BRONZE.RAW_AUDIENCES; -- expected: 350
SELECT COUNT(*) FROM MEDIA_COPILOT.BRONZE.RAW_ENGAGEMENT; -- expected: 10,000
SELECT COUNT(*) FROM MEDIA_COPILOT.BRONZE.RAW_FEEDBACK; -- expected: 750
SELECT COUNT(*) FROM MEDIA_COPILOT.BRONZE.RAW_CAMPAIGNS; -- expected: 2,000
```

---

## FASE 2 — Silver (dbt)

> **Note:** Make sure you are in **Projects > Workspaces** in Snowsight before running dbt.

**[CoCo]** Init dbt
```
Create a dbt project called media_copilot connected to MEDIA_COPILOT.
models/staging/ -> schema SILVER | models/marts/ -> schema GOLD | warehouse COMPUTE_WH.
Create models/staging/sources.yml with the Bronze tables: RAW_AUDIENCES, RAW_ENGAGEMENT, RAW_FEEDBACK, RAW_CAMPAIGNS
```

**[CoCo]** Silver: STG_AUDIENCES
```
Create models/staging/stg_audiences.sql to flatten RAW_AUDIENCES.
Fields: audience profiles (audience_id, username, region, subscription_type, preferred_genre, age_group, watch_hours_monthly, status)
Materialize as table in SILVER.
```

**[CoCo]** Silver: STG_ENGAGEMENT
```
Create models/staging/stg_engagement.sql to flatten RAW_ENGAGEMENT.
Fields: engagement events (event_id, audience_id, event_type, content_id, content_type, genre, duration_seconds, platform)
Materialize as table in SILVER.
```

**[CoCo]** Silver: STG_FEEDBACK
```
Create models/staging/stg_feedback.sql to flatten RAW_FEEDBACK.
Fields: content feedback (feedback_id, audience_id, category, priority, description, sla_breached, satisfaction_score)
Materialize as table in SILVER.
```

**[CoCo]** Silver: STG_CAMPAIGNS
```
Create models/staging/stg_campaigns.sql to flatten RAW_CAMPAIGNS.
Fields: advertising campaigns (campaign_id, advertiser_name, channel, budget_eur, spend_eur, impressions, clicks, conversions, cpm_eur)
Materialize as table in SILVER.
```

**[CoCo]** Run Silver
```
dbt run --select staging
```

**[✓ SQL]**
```sql
SELECT COUNT(*) FROM MEDIA_COPILOT.SILVER.STG_AUDIENCES; -- expected: 350
SELECT COUNT(*) FROM MEDIA_COPILOT.SILVER.STG_ENGAGEMENT; -- expected: 10,000
SELECT COUNT(*) FROM MEDIA_COPILOT.SILVER.STG_FEEDBACK; -- expected: 750
SELECT COUNT(*) FROM MEDIA_COPILOT.SILVER.STG_CAMPAIGNS; -- expected: 2,000
```

---

## FASE 2 — Gold (dbt)

**[CoCo]** Gold: AUDIENCE_360
```
Create models/marts/audience_360.sql:
360-degree audience view: base data + total_events + watch_hours + likes + shares + total_feedback + open_feedback + avg_satisfaction + days_since_last_activity. JOIN of all Silver tables.
Materialize as table in GOLD.
```

**[CoCo]** Gold: CONTENT_PERFORMANCE
```
Create models/marts/content_performance.sql:
Content performance by content_type, genre and day: total_plays, total_duration_hours, completion_rate, like_rate, share_rate, distinct_audiences, top_platform.
Materialize as table in GOLD.
```

**[CoCo]** Gold: AUDIENCE_CHURN_RISK
```
Create models/marts/audience_churn_risk.sql:
Churn risk per audience: subscription_type + days_since_last_activity + feedback_count + avg_watch_hours + engagement_trend + churn_score (0-100).
Materialize as table in GOLD.
```

**[CoCo]** Run Gold
```
dbt run --select marts
```

**[✓ SQL]**
```sql
SELECT COUNT(*) FROM MEDIA_COPILOT.GOLD.AUDIENCE_360;
SELECT COUNT(*) FROM MEDIA_COPILOT.GOLD.CONTENT_PERFORMANCE;
SELECT COUNT(*) FROM MEDIA_COPILOT.GOLD.AUDIENCE_CHURN_RISK;
```

---

## FASE 3 — AI (Cortex COMPLETE)

**[CoCo]** Case 1: Content feedback classification
```
Write a SQL query using SNOWFLAKE.CORTEX.COMPLETE with model 'llama3.1-70b'
on table MEDIA_COPILOT.SILVER.STG_FEEDBACK.
Analyze the "description" field and return a JSON with: issue_type (video_quality|audio|subtitles|content|technical_error|account|suggestion), urgency (critical|high|medium|low), responsible_team (engineering|content|support|product|legal), experience_impact (high|medium|low), brief_summary (max 15 words)
System prompt: "You are an incident classification system for a streaming platform."
Create view MEDIA_COPILOT.AI.VW_FEEDBACK_CLASSIFICATION. Limit to 50 records.
```

**[✓ SQL]**
```sql
SELECT * FROM MEDIA_COPILOT.AI.VW_FEEDBACK_CLASSIFICATION LIMIT 3;
```

**[CoCo]** Case 2: Campaign executive summary
```
Write a SQL query using SNOWFLAKE.CORTEX.COMPLETE with model 'llama3.1-70b'
on table MEDIA_COPILOT.GOLD.CONTENT_PERFORMANCE.
Generate a 3-4 sentence executive summary in English using fields: content_type, genre, total_plays, total_duration_hours, completion_rate, like_rate, distinct_audiences
System prompt: "You are a digital media analyst. Generate a 3-4 sentence executive summary in English about this content/campaign performance."
Create view MEDIA_COPILOT.AI.VW_CAMPAIGN_EXECUTIVE_SUMMARY. Limit to 20 records.
```

**[✓ SQL]**
```sql
SELECT * FROM MEDIA_COPILOT.AI.VW_CAMPAIGN_EXECUTIVE_SUMMARY LIMIT 3;
```

**[CoCo]** Case 3: Personalized content recommendation
```
Write a SQL query using SNOWFLAKE.CORTEX.COMPLETE with model 'llama3.1-70b'
on table MEDIA_COPILOT.GOLD.AUDIENCE_CHURN_RISK.
Return a JSON with: audience_segment (superfan|regular|casual|dormant), recommended_content (genre + type), re_engagement_channel (push|email|in_app|home_banner), optimal_time, personalized_message
System prompt: "You are a digital content personalization expert. Analyze this user and return ONLY valid JSON."
Filter WHERE churn_score >= 30.
Create view MEDIA_COPILOT.AI.VW_CONTENT_RECOMMENDATIONS. Limit to 25 records.
```

**[✓ SQL]**
```sql
SELECT * FROM MEDIA_COPILOT.AI.VW_CONTENT_RECOMMENDATIONS LIMIT 3;
```

---

## FASE 4 — Semantic View

**[CoCo]**
```
Create a Semantic View in MEDIA_COPILOT.GOLD called SV_MEDIA_COPILOT
over the Silver and Gold tables for natural language queries.
Business context: Data covers audiences/subscribers, engagement events, content feedback and advertising campaigns. Streaming/media platform in Spain.
Include dimensions with English comments, metrics with synonyms
and AI_SQL_GENERATION instructions with the business context.
```

**[✓ SQL]**
```sql
DESCRIBE SEMANTIC VIEW MEDIA_COPILOT.GOLD.SV_MEDIA_COPILOT;
```

---

## FASE 5 — Cortex Agent

**[CoCo]**
```
Create a Cortex Agent in Snowflake Intelligence for Media, Entertainment & Advertising.
- Create the default Snowflake Intelligence object if it does not exist
- Create the SNOWFLAKE_INTELLIGENCE database and AGENTS schema
- Create agent MEDIA_COPILOT_AGENT connected to Semantic View MEDIA_COPILOT.GOLD.SV_MEDIA_COPILOT
  with orchestration model claude-4-sonnet, always responding in English
  and business context: digital content and advertising platform in Spain
- Grant PUBLIC role access to the agent and the Semantic View
```

**[SQL]**
```sql
CREATE SNOWFLAKE INTELLIGENCE IF NOT EXISTS SNOWFLAKE_INTELLIGENCE_OBJECT_DEFAULT;
GRANT USAGE ON SNOWFLAKE INTELLIGENCE SNOWFLAKE_INTELLIGENCE_OBJECT_DEFAULT TO ROLE PUBLIC;

CREATE DATABASE IF NOT EXISTS SNOWFLAKE_INTELLIGENCE;
CREATE SCHEMA IF NOT EXISTS SNOWFLAKE_INTELLIGENCE.AGENTS;

CREATE OR REPLACE AGENT SNOWFLAKE_INTELLIGENCE.AGENTS.MEDIA_COPILOT_AGENT
  COMMENT = 'Copilot for Media, Entertainment & Advertising'
  PROFILE = '{"display_name": "Media & Audience Intelligence Copilot"}' 
  FROM SPECIFICATION $$
  {
    "models": {"orchestration": "claude-4-sonnet"},
    "instructions": {
      "orchestration": "You are an assistant for digital content and advertising platform in Spain. Always respond in English.",
      "response": "Respond clearly and concisely in English. Use table format when appropriate."
    },
    "tools": [{
      "tool_spec": {
        "type": "cortex_analyst_text_to_sql",
        "name": "media_analyst",
        "description": "Data covers audiences/subscribers, engagement events, content feedback and advertising campaigns. Streaming/media platform in Spain."
      }
    }],
    "tool_resources": {
      "media_analyst": {
        "semantic_view": "MEDIA_COPILOT.GOLD.SV_MEDIA_COPILOT",
        "execution_environment": {"type": "warehouse", "warehouse": "COMPUTE_WH"},
        "query_timeout": 60
      }
    }
  }
  $$;

GRANT USAGE ON AGENT SNOWFLAKE_INTELLIGENCE.AGENTS.MEDIA_COPILOT_AGENT TO ROLE PUBLIC;
GRANT SELECT ON SEMANTIC VIEW MEDIA_COPILOT.GOLD.SV_MEDIA_COPILOT TO ROLE PUBLIC;
```

**[✓ SQL]**
```sql
SHOW AGENTS IN SCHEMA SNOWFLAKE_INTELLIGENCE.AGENTS;
```

> Open **Snowsight > AI & ML > Snowflake Intelligence** and search for the agent.

---

## Test the Agent

Sample questions to validate the agent:

- "Which users have been inactive for more than 30 days?"
- "Which content genres have the highest completion rate?"
- "Which advertising campaigns have the highest CTR this month?"
- "How many subscribers are at risk of cancellation?"

---

## FASE FINAL — Streamlit App on Snowflake

**[CoCo]**
```
Create a Streamlit in Snowflake (SiS) app in database MEDIA_COPILOT, schema AI, named MEDIA_APP.
The app should connect to Snowflake using the active session (get_active_session).
Include 4 sections using st.tabs:
1. KPIs — metric cards with st.metric with the main totals and ratios from AUDIENCE_360
2. Distribution — bar chart (st.bar_chart) by category/region from CONTENT_PERFORMANCE
3. Top 20 — interactive table (st.dataframe) sorted by risk score from AUDIENCE_CHURN_RISK
4. Trend — line chart (st.line_chart) with temporal evolution of key events
Use descriptive English titles and labels. Apply a dark theme with st.set_page_config.
```
