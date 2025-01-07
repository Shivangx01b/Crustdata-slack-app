<h1 align="center">
  <br>
  <a href=""><img src="https://github.com/Shivangx01b/Crustdata-slack-app/blob/main/static/this.PNG" alt="" width="2000px;"></a>
  <br>
  <a href="https://twitter.com/intent/follow?screen_name=shivangx01b"><img src="https://img.shields.io/twitter/follow/shivangx01b?style=flat-square"></a>
</h1>

# Crustdata Build Challenge: Level 3 (Slack Integration)

# Slack Backend: https://crustdata-slack-app.onrender.com

## Overview

This stage of the challenge involves integrating the chatbot with Slack to function as a Slack bot. The bot will:
- Work on specific Slack channels.
- Respond only to specific users.
- Draft a response for each message based on Crustdata API knowledge.

---

## Implementation

### Slack Bot Integration
1. **Slack Client Setup**:
   - The Slack bot is integrated using the `slack_sdk` library.
   - The bot listens to events and responds to messages in specified channels.

   ```python
   from slack_sdk import WebClient
   from slack_sdk.errors import SlackApiError

   client = WebClient(token=os.getenv("SLACK_BOT_TOKEN"))
   ```

2. **API Event Handling**:
   - A FastAPI application handles incoming events from Slack.
   - The `/slack/events` endpoint processes messages and commands.

   ```python
   @app.post("/slack/events")
   async def slack_events(request: Request):
       data = await request.json()

       if "challenge" in data:
           return JSONResponse(content={"challenge": data["challenge"]})

       event = data.get("event", {})
       if event.get("type") == "message" and not event.get("bot_id"):
           user_id = event.get("user")
           channel_id = event.get("channel")
           text = event.get("text")

           if text.startswith("/api_help"):
               response = generate_response(text)
               try:
                   client.chat_postMessage(channel=channel_id, text=response)
               except SlackApiError as e:
                   print(f"Error posting message: {e.response['error']}")

       return JSONResponse(content={"status": "ok"})
   ```

3. **Command Handling**:
   - Commands like `/jointhis` and `/info` allow users to interact with the bot.
   - Background tasks process these commands asynchronously.

   ```python
   @app.post("/slack/command")
   async def slack_command(request: Request, background_tasks: BackgroundTasks):
       data = await request.form()
       command = data.get("command")
       text = data.get("text")
       user_id = data.get("user_id")
       channel_id = data.get("channel_id")

       if command == "/jointhis":
           background_tasks.add_task(handle_join_channel, channel_id, user_id)
           return JSONResponse(
               content={
                   "response_type": "ephemeral",
                   "text": "Attempting to join the channel... Please wait."
               }
           )

       if command == "/info":
           background_tasks.add_task(handle_info_command, channel_id, user_id, text)
           return JSONResponse(
               content={
                   "response_type": "ephemeral",
                   "text": "Processing your request..."
               }
           )
   ```

4. **Joining Channels**:
   - The bot can join a channel upon receiving a `/jointhis` command.

   ```python
   async def handle_join_channel(channel_id: str, user_id: str):
       try:
           client.conversations_join(channel=channel_id)
           client.chat_postEphemeral(
               channel=channel_id,
               user=user_id,
               text=f"The bot has successfully joined this channel."
           )
       except SlackApiError as e:
           print(f"Error joining channel: {e.response['error']}")
   ```

5. **Drafting Responses**:
   - User messages are processed by the `generate_response` function, which uses LangChain to retrieve and formulate responses.

   ```python
   def generate_response(prompt):
       res = retrieval_chain.invoke({
           "input": prompt + ". Please provide detailed information about the APIs."
       })
       return res["answer"]
   ```

### LangChain Integration
- The bot uses LangChain's retrieval-based QA system to generate responses.
- A FAISS index (`faiss_index`) stores and retrieves knowledge about Crustdata APIs.

```python
new_vector_store = FAISS.load_local(
    "faiss_index", OpenAIEmbeddings(), allow_dangerous_deserialization=True
)
retriever = new_vector_store.as_retriever()
retrieval_chain = create_retrieval_chain(
    history_aware_retriever, combine_docs_chain
)
```

---

## Deployment

1. **Environment Variables**:
   - Set `SLACK_BOT_TOKEN` in the environment for Slack integration.

2. **Run the App**:
   - Start the FastAPI server: `uvicorn app:app --reload`.

3. **Connect to Slack**:
   - Configure the Slack app in your workspace.
   - Set the event subscription URL to point to `/slack/events`.
   - Set the command for /info and /jointhis at slack slash command



