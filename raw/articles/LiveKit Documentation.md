---
title: "LiveKit Documentation"
source: "https://docs.livekit.io/agents/?phid=019dafa3-565f-778e-b0b8-1a383f77530e"
author:
published:
created: 2026-04-21
description: "Realtime framework for voice, video, and physical AI agents."
tags:
  - "clippings"
---
## Overview

The Agents framework lets you add any Python or Node.js program to LiveKit rooms as full realtime participants. Build agents with code using the Python and Node.js SDKs, or use [LiveKit Agent Builder](https://docs.livekit.io/agents/start/builder/) to prototype and deploy agents directly in your browser without writing code. The framework provides tools and abstractions for feeding realtime media and data through an AI pipeline that works with any provider, and publishing realtime results back to the room.

Use LiveKit Cloud to start building agents right away, with managed deployment, built-in observability with transcripts and traces, and LiveKit Inference for running AI models without API keys. You can deploy your agents to [LiveKit Cloud](https://docs.livekit.io/deploy/agents/) or any [custom environment](https://docs.livekit.io/deploy/custom/deployments/) of your choice.

If you want to get your hands on the code for building an agent right away, follow the Voice AI quickstart guide or try out Agent Builder and build your first voice agent in minutes. It takes just a few minutes to build your first voice agent.

## [Voice AI quickstart](https://docs.livekit.io/agents/start/voice-ai/)

Build and deploy a simple voice assistant with Python or Node.js in less than 10 minutes.

## [LiveKit Agent Builder](https://docs.livekit.io/agents/start/builder/)

Prototype and deploy voice agents directly in your browser, without writing any code.

## [LiveKit 101: Build Production-Ready Voice AI Agents](https://www.youtube.com/playlist?list=PLWx-Xa8RhJxXuv8fu2Qz9rj2MPb4qgXir)

Build production-ready voice AI agents with LiveKit in the official video course series on YouTube.

## [Deploying to LiveKit Cloud](https://docs.livekit.io/agents/ops/deployment/)

Run your agent on LiveKit Cloud's global infrastructure.

## [GitHub repository](https://github.com/livekit/agents)

Python source code and examples for the LiveKit Agents SDK.

## [SDK reference](https://docs.livekit.io/reference/python/livekit/agents/index.html)

Python reference docs for the LiveKit Agents SDK.

### Use cases

Some applications for agents include:

- **Multimodal assistant**: Talk, text, or screen share with an AI assistant.
- **Telehealth**: Bring AI into realtime telemedicine consultations, with or without humans in the loop.
- **Call center**: Deploy AI to the front lines of customer service with inbound and outbound call support.
- **Realtime translation**: Translate conversations in realtime.
- **NPCs**: Add lifelike NPCs backed by language models instead of static scripts.
- **Robotics**: Put your robot's brain in the cloud, giving it access to the most powerful models.

The following [recipes](https://docs.livekit.io/recipes/) demonstrate some of these use cases:

## [Medical Office Triage](https://github.com/livekit-examples/python-agents-examples/tree/main/complex-agents/medical_office_triage)

Agent that triages patients based on symptoms and medical history.

## [Restaurant Agent](https://github.com/livekit/agents/blob/main/examples/voice_agents/restaurant_agent.py)

A restaurant front-of-house agent that can take orders, add items to a shared cart, and checkout.

## [Company Directory](https://docs.livekit.io/reference/recipes/company-directory/)

Build a AI company directory agent. The agent can respond to DTMF tones and voice prompts, then redirect callers.

## [Pipeline Translator](https://docs.livekit.io/reference/recipes/pipeline_translator/)

Implement translation in the processing pipeline.

### Framework overview

Your agent code operates as a stateful, realtime bridge between powerful AI models and your users. While AI models typically run in data centers with reliable connectivity, users often connect from mobile networks with varying quality.

WebRTC ensures smooth communication between agents and users, even over unstable connections. LiveKit WebRTC is used between the frontend and the agent, while the agent communicates with your backend using HTTP and WebSockets. This setup provides the benefits of WebRTC without its typical complexity.

The agents SDK includes components for handling the core challenges of realtime voice AI, such as streaming audio through an STT-LLM-TTS pipeline, reliable turn detection, handling interruptions, and LLM orchestration. It supports plugins for most major AI providers, with more continually added. The framework is fully open source and supported by an active community.

Other framework features include:

- **Voice, video, and text**: Build agents that can process realtime input and produce output in any modality.
- **Tool use**: Define tools that are compatible with any LLM, and even forward tool calls to your frontend.
- **Multi-agent handoff**: Break down complex workflows into simpler tasks.
- **Extensive integrations**: Integrate with nearly every AI provider there is for LLMs, STT, TTS, and more.
- **State-of-the-art turn detection**: Use the custom turn detection model for lifelike conversation flow.
- **Made for developers**: Build your agents in code, not configuration.
- **Production ready**: Includes built-in agent server orchestration, load balancing, and Kubernetes compatibility.
- **Open source**: The framework and entire LiveKit ecosystem are open source under the Apache 2.0 license.

### How agents connect to LiveKit

When your agent code starts, it first registers with a LiveKit server (either [self hosted](https://docs.livekit.io/transport/self-hosting/) or [LiveKit Cloud](https://cloud.livekit.io/)) to run as an "agent server" process. The agent server waits until it receives a dispatch request. To fulfill this request, the agent server boots a "job" subprocess which joins the room. By default, your agent servers are dispatched to each new room created in your LiveKit Cloud project (or self-hosted server). To learn more about agent servers, see the [Server lifecycle](https://docs.livekit.io/agents/server/lifecycle/) guide.

After your agent and user join a room, the agent and your frontend app can communicate using LiveKit WebRTC. This enables reliable and fast realtime communication in any network conditions. LiveKit also includes full support for telephony, so the user can join the call from a phone instead of a frontend app.

To learn more about how LiveKit works overall, see the [Intro to LiveKit](https://docs.livekit.io/intro/) guide.

## Key concepts

Understand these core concepts to build effective agents with the LiveKit Agents framework.

### Multimodality

Agents can communicate through multiple channels — speech and audio, text and transcriptions, and vision. Just as humans can see, hear, speak, and read, agents can process and generate content across these modalities, enabling richer, more natural interactions where they understand context from different sources.

## [Multimodality overview](https://docs.livekit.io/agents/multimodality/)

Learn how to configure agents to process speech, text, and vision.

### Logic & structure

The framework provides powerful abstractions for organizing agent behavior, including agent sessions, tasks and task groups, workflows, tools, pipeline nodes, turn detection, agent handoffs, and external data integration.

## [Logic & structure overview](https://docs.livekit.io/agents/logic/)

Learn how to structure your agent's logic and behavior.

### Agent server

Agent servers manage the lifecycle of your agents, handling dispatch, job execution, and scaling. They provide production-ready infrastructure including automatic load balancing and graceful shutdowns.

## [Agent server overview](https://docs.livekit.io/agents/server/)

Learn how agent servers manage your agents' lifecycle and deployment.

### Models

The Agents framework supports a wide range of AI models for LLMs, speech-to-text (STT), text-to-speech (TTS), realtime APIs, and virtual avatars. Use [LiveKit Inference](https://docs.livekit.io/agents/models/#inference) to access models directly through LiveKit Cloud, or use plugins to connect to a wide range of providers updated regularly.

## [Models overview](https://docs.livekit.io/agents/models/)

Explore the full list of AI models and providers available for your agents, both through LiveKit Inference and plugins.

## Getting started

Follow these guides to learn more and get started with LiveKit Agents.

## [Voice AI quickstart](https://docs.livekit.io/agents/start/voice-ai/)

Build a simple voice assistant with Python or Node.js in less than 10 minutes.

## [Recipes](https://docs.livekit.io/reference/recipes/)

A comprehensive collection of examples, guides, and recipes for LiveKit Agents.

## [Intro to LiveKit](https://docs.livekit.io/intro/)

An overview of the LiveKit ecosystem.

## [Web and mobile frontends](https://docs.livekit.io/agents/start/frontend/)

Put your agent in your pocket with a custom web or mobile app.

## [Telephony integration](https://docs.livekit.io/agents/start/telephony/)

Your agent can place and receive calls with LiveKit's SIP integration.

## [Building voice agents](https://docs.livekit.io/agents/build/)

Comprehensive documentation to build advanced voice AI apps with LiveKit.

## [Agent server lifecycle](https://docs.livekit.io/agents/server/)

Learn how to manage your agents with agent servers and jobs.

## [Deploying to production](https://docs.livekit.io/agents/ops/deployment/)

Guide to deploying your voice agent in a production environment.

## [AI models](https://docs.livekit.io/agents/models/)

Explore the full list of AI models available for LiveKit Agents.