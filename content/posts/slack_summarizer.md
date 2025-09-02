+++
title = 'A Slack Summarizer to Overcome Information Overload 🤖'
date = 2025-08-25T07:07:07+01:00
draft = false
disableComments = true
+++

You’re back from a well-deserved two-week vacation. 🌴 After the initial “welcome back” messages, a cold wave of anxiety hits you: your Slack sidebar is a sea of red notifications. 99+ unread messages. Dozens of channels. You try to catch up, but it’s like drinking from a firehose.
This was the problem I faced. I knew there had to be a better way.

### The Solution: An Intelligent, Action-Oriented Summarizer

My goal was to create a Slack Summarizer bot that could get to the heart of the matter. This wasn’t just about condensing text; it was about providing useful, actionable context. The bot was designed to be your personal assistant for Slack, helping you quickly catch up on channels or specific threads with a simple slash command.
At its core, the bot was powered by the latest advancements in AI models, supporting both Gemini and Claude. This allowed users to select their preferred model and use their own API key, providing flexibility and control over the quality and style of the summary.

The bot’s features are:

* **Thread Summarization:** Get a concise summary of a long-running thread with a simple `/summarize_thread` command.

* **Channel Catch-Up:** Get a summary of the most important discussions in a channel by mentioning the bot.

* **Create Tickets against Problems:** Configure the Summarizer to ask for ticket creation when showing summaries and also added an API for creating a ticket on the Shortcut app.

### From Insights to Action 🚀

The most exciting part of this project was adding a feature that transformed insights into action. After generating a summary, the bot would present a follow-up asking, “Do you want to create a ticket for this?” If you clicked “Yes,” it would automatically create a ticket in our project management app, Shortcut, with all the relevant details from the conversation summary.

This feature was a game-changer. It allowed removing the friction of creating tickets by having to fill out the ticket form again and again.

### The Build: A Glimpse Under the Hood 🛠️

Building the Slack Summarizer was an interesting technical challenge. The bot was written in Python, using the Slack API to listen for commands and post responses. I used the uvicorn web server for local development and exposed it with ngrok for testing. The real magic happened in the backend, where we integrated with the Gemini and Claude APIs, and a custom integration was built for Shortcut's API to handle ticket creation. The entire application was containerized using Docker and deployed to a Kubernetes cluster.

### The Art of Prompt Engineering 🎨

A critical part of this project was the extensive prompt engineering. I had defined everything from the structure of the input, including all the Slack metadata like `channel_id` and `user_id`, to the specific requirements for the output.

The goal was to make the bot smart enough to handle situations where a problem was discussed but no resolution was found in the context. In those cases, the prompt would tell the AI to add a new block to its output, asking the user directly if they wanted to create a ticket. This proactive approach—complete with "Yes/No" buttons—made the bot incredibly useful, as it didn't just provide information; it helped turn conversations into action items.

### Using the Bot: Commands and Configuration ⚙️

Once the bot is installed and configured on your Slack workspace, using it is incredibly simple. We designed it around two core commands to make it as intuitive as possible:

* `/configure`: This is your one-time setup command. By typing this in any channel or personal chat where the bot has been invited, a modal window appears. This is where you can select your preferred AI model (Gemini or Claude) and enter your API key to get started.

* `/summarize_chat`: This is your command to summarize a chat. The input is an integer that refers to the number of days.

* `/summarize_thread`: This is your command to summarize a thread. The input is a thread link that you will provide.

You can see a live demonstration of the bot in action and how to use these commands by watching this video: Slack Summarizer Overview

### The Impact: Faster, Smarter, and More Productive 📈

Today, we have great tools like Slack’s built-in AI. But this project was a reminder of how a small team can solve a major productivity problem with a bit of creativity and technical know-how. The Slack Summarizer saved time as well as allowed us to be up to date with the latest discussions happening in different channels.

### Future Work and Improvements 🔮

While the bot is already a powerful tool, I have a few ideas for where to take it next. For now, the only things on the roadmap are adding support for more AI models, specifically OpenAI. I'd also love to integrate streaming support. I'm not sure if Slack's current API allows for true streaming, but if it does in the future, it's definitely something I'll look into to make the bot's responses even more dynamic.


### See the Summarizer in Action 🎬
<!--  -->
[![Watch the video](https://img.youtube.com/vi/9z5ZY3KUkro/0.jpg)](https://youtu.be/9z5ZY3KUkro)