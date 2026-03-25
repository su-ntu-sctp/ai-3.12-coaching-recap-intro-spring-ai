# [3.12] Coaching: Recap on REST API Development with Spring Boot + Introduction to Spring AI

## Lesson Overview

![Spring AI](./assets/images/infographic-3.12-spring-ai-intro.png)

## Dependencies

- [Lesson](./lesson.md) / [Slide Deck](./slides.md)

## Lesson Objectives

By the end of this lesson, students will be able to:

* **Demonstrate** understanding of REST API development with Spring Boot by building a working endpoint from scratch
* **Add** Spring AI to a Spring Boot project and configure it with an OpenAI API key
* **Build** a REST endpoint that sends a user prompt to an LLM and returns the response
* **Customise** AI behaviour by applying a system prompt

## Lesson Plan

| Duration | What | How or Why |
|---|---|---|
| 60 min | Revision: Lesson 3.11 — REST API with Spring Boot | Structured recap of project setup, `application.properties`, controllers, and DI — followed by a quick hands-on activity and open Q&A |
| 10 min | Part 1: What is Spring AI? | Contextual intro — why Spring AI exists and where it fits in the Java ecosystem |
| 20 min | Part 2: Project Setup | New Spring Boot project, add Spring AI BOM + OpenAI starter, configure API key in `application.properties` |
| 20 min | Part 3: First AI Endpoint | Code-along — build `/chat` endpoint using `ChatClient`; test live with real prompts |
| 15 min | Part 4: System Prompt | Add `/support` endpoint with a system prompt; compare behaviour with and without it |
| 20 min | Activity — Build a Themed AI Assistant | Students build their own themed endpoint with a custom system prompt |
| 15 min | Wrap-up + Q&A | Recap key concepts; preview what's next |
| **160 min** | **Total** | |