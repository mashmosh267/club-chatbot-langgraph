# Curious Minds Club Chatbot

Homework 4 · Memory That Survives a Restart

1. Upload `club_chatbot_langgraph.ipynb` to Google Colab.
2. In Supabase, open **Connect** and choose **Session pooler**. Fill in the database password privately, percent-encode special characters in the password, and set `sslmode=require` in the connection string. Use a role that can create the checkpoint tables. [Supabase connection guide](https://supabase.com/docs/guides/database/connecting-to-postgres)
3. In Colab’s **Secrets** panel, add `OPENAI_API_KEY` and `SUPABASE_DB_URL` and grant the notebook access to both. Keep their values out of cells and outputs.
4. Run the notebook in order. The direct tests explain the expected replies and the 8- and 16-message checks. If Colab requests a restart after installing packages, restart before running the remaining cells.
5. Open the Gradio link privately. Anyone with it may read saved chats and consume your OpenAI credits. Use sample questions and stop the app when finished.

## Verify a restart

Switch clubs mid-chat, then ask about the old club and check that the assistant refuses. Open a second chat, reopen the first, and verify that its messages and last submitted club are restored. In Supabase, inspect `checkpoints` (snapshots), `checkpoint_blobs` (serialized channel values), and `checkpoint_writes` (intermediate task writes).

Restart the Colab runtime, reconnect to the same database, rebuild and launch the UI, then refresh the saved list and continue the original chat. You can skip the direct tests on this restart.

## Live Colab results

The one-node graph diagram rendered, Supabase connected through PostgresSaver, and Gradio responded. After a Colab runtime restart, a saved Astronomy conversation reopened and the chatbot correctly answered a follow-up about Mars's two moons.

The last direct test still incorrectly refused “Who won the 2022 World Cup?” with Football selected. The restart test passed. The topic guard still fails this in-scope case.
