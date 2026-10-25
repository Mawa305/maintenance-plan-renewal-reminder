# maintenance-plan-renewal-reminder
Daily n8n workflow that reminds HVAC maintenance plan customers 30, 14, and 7 days before expiry. Writes each reminder with OpenAI, sends it by Gmail once per stage, tracks status in Supabase, and alerts on Telegram.
