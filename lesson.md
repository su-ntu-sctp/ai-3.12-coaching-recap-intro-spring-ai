# Lesson: Introduction to AI Integration with Spring AI

## Lesson Overview

This lesson introduces Spring AI — a framework that makes it straightforward to connect your Spring Boot application to a Large Language Model (LLM) such as OpenAI's GPT. You will go from a blank Spring Boot project to a working AI-powered REST endpoint in a single session. No prior AI experience is required.

**Prerequisites:** Spring Boot basics (Lesson 3.11) — project setup, controllers, `application.properties`, Dependency Injection

## Lesson Objectives

By the end of this lesson, students will be able to:

1. **Add** Spring AI to a Spring Boot project and configure it with an OpenAI API key
2. **Build** a REST endpoint that sends a user prompt to an LLM and returns the response
3. **Customise** AI behaviour by applying a system prompt

---

## Part 1: What is Spring AI?

Spring AI is an official Spring project that makes it easy to integrate AI models — such as OpenAI, Google Gemini, and Anthropic Claude — into your Java applications. Think of it as the "Spring Data" of the AI world: instead of writing raw HTTP calls to OpenAI's API yourself, Spring AI provides ready-made Spring beans and clean abstractions so you can focus on building features rather than dealing with API plumbing.

Before Spring AI existed, integrating an LLM into a Java application meant manually writing HTTP clients, handling authentication, parsing JSON responses, and managing retries. Spring AI handles all of that for you.

**Why does this matter?**

By 2025, AI features are becoming a standard part of enterprise applications — chatbots, intelligent search, automated summaries, and more. As Java developers, knowing how to wire an LLM into a Spring Boot backend puts you at a significant advantage. Today you will see just how few lines of code it actually takes.

Spring AI supports many providers (OpenAI, Ollama, Azure, Google, Anthropic, and more). In this lesson we will use **OpenAI** with the **GPT-4o-mini** model — it is fast, inexpensive, and ideal for learning.

---

## Part 2: Project Setup

### Create a New Spring Boot Project

Create a new Spring Boot project using Spring Initializr (Ctrl/Cmd + Shift + P → "Spring Initializr: Create a Maven Project").

| Setting | Value |
|---|---|
| Spring Boot version | `3.4.x` (choose the latest 3.4) |
| Language | Java |
| Group ID | `sg.edu.ntu` |
| Artifact ID | `spring-ai-demo` |
| Packaging | Jar |
| Java version | 21 |

For dependencies, select:
- **Spring Web**
- **Spring Boot DevTools**

We will add the Spring AI dependency manually in the next step.

### Add the Spring AI Dependency

Open `pom.xml`. We need to add two things: the Spring AI **Bill of Materials (BOM)** for version management, and the OpenAI starter dependency.

**What is a BOM?** A Bill of Materials is a special Maven dependency that centrally manages the versions of a group of related libraries. Instead of specifying a version number on every individual Spring AI dependency you add, you declare the BOM once and all Spring AI modules automatically use compatible versions. Think of it as a "version agreement" for a whole family of libraries.

First, add the BOM inside the `<dependencyManagement>` block. If this block does not exist yet, add it before the closing `</project>` tag.

```xml
<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>org.springframework.ai</groupId>
      <artifactId>spring-ai-bom</artifactId>
      <version>1.1.0</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
  </dependencies>
</dependencyManagement>
```

The BOM ensures all Spring AI modules use compatible versions — you will not need to specify version numbers for individual Spring AI dependencies.

Next, add the OpenAI starter inside your existing `<dependencies>` block.

```xml
<dependency>
  <groupId>org.springframework.ai</groupId>
  <artifactId>spring-ai-openai-spring-boot-starter</artifactId>
</dependency>
```

Save the file. Maven will download the dependencies automatically. You will see a prompt in VS Code to reload — click **Yes**.

### Configure the API Key

Open `src/main/resources/application.properties` and add the following.

```properties
# Spring AI - OpenAI Configuration
spring.ai.openai.api-key=YOUR_API_KEY_HERE
spring.ai.openai.chat.options.model=gpt-4o-mini
spring.ai.openai.chat.options.temperature=0.7
```

Replace `YOUR_API_KEY_HERE` with your actual OpenAI API key. You can find or create one at [https://platform.openai.com/api-keys](https://platform.openai.com/api-keys).

> ⚠️ **Important:** Never commit your API key to a public Git repository. For now, pasting it directly is fine for learning. In production, you would use environment variables or a secrets manager.

**What do these properties mean?**
- `api-key` — your credentials to access the OpenAI API
- `model` — `gpt-4o-mini` is a fast and affordable model, perfect for development
- `temperature` — controls how creative/varied the responses are. `0.7` is a good balanced value. `0.0` is very deterministic; `1.0` is very creative.

Run the application to confirm it starts without errors.

```bash
mvn spring-boot:run
```

---

## Part 3: Your First AI Endpoint

Now the interesting part. Create a new file `AiController.java` in your main package and code along.

```java
import org.springframework.ai.chat.client.ChatClient;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class AiController {

  private final ChatClient chatClient;

  public AiController(ChatClient.Builder chatClientBuilder) {
    this.chatClient = chatClientBuilder.build();
  }

}
```

Notice how we are injecting `ChatClient.Builder` through the constructor. This is called **constructor injection** — it is another way to apply Dependency Injection, and is actually the preferred approach in modern Spring development. Instead of annotating a field with `@Autowired`, you declare the dependency as a constructor parameter and Spring automatically provides the bean when it creates the class. The end result is the same — Spring manages the object for you — but constructor injection makes dependencies more explicit and easier to test. We then call `.build()` to get our `ChatClient` instance, ready to send prompts to the model.

Now add our first endpoint.

```java
@GetMapping("/chat")
public String chat(@RequestParam String message) {
  return chatClient.prompt()
      .user(message)
      .call()
      .content();
}
```

Let's break down what this does:
- `chatClient.prompt()` — starts building a prompt to send to the model
- `.user(message)` — sets the user's message (what the user is asking)
- `.call()` — sends the request to OpenAI and waits for the response
- `.content()` — extracts the response text as a plain `String`

Run the application and test it.

```bash
curl "localhost:8080/chat?message=What is Java?"
```

Or open your browser at:
```
localhost:8080/chat?message=What is Java?
```

You should see a response from GPT-4o-mini. You have just built an AI-powered REST endpoint in a Spring Boot application. 🎉

Try a few different messages and observe the responses.

---

## Part 4: Adding a System Prompt

Right now, the AI will answer any question about any topic. In real applications, we usually want to give the AI a specific role and set of instructions — this is called a **system prompt**.

A system prompt is a behind-the-scenes instruction that you provide to the model before the user's message. It shapes how the AI responds — its tone, its focus area, and what it should or should not do.

Let's create a dedicated endpoint that uses a system prompt to turn our AI into a helpful customer support assistant for the `simple-crm` project from last lesson.

```java
@GetMapping("/support")
public String support(@RequestParam String message) {
  return chatClient.prompt()
      .system("You are a friendly and professional customer support assistant for a CRM software company. " +
              "You help users with questions about managing customers, contacts, and sales pipelines. " +
              "Keep your answers concise and practical. " +
              "If a question is not related to CRM or customer management, politely redirect the user.")
      .user(message)
      .call()
      .content();
}
```

Run the application and test both endpoints.

```bash
# Should give a focused CRM-related answer
curl "localhost:8080/support?message=How do I add a new customer?"

# Should politely redirect
curl "localhost:8080/support?message=What is the weather today?"
```

Compare the responses from `/chat` and `/support` for the same question. Notice how the system prompt fundamentally changes the AI's behaviour — same model, same infrastructure, completely different personality and scope.

This is the core power of system prompts: with a single block of text, you can transform a general-purpose LLM into a specialised assistant for your application.

---

## 🧑‍💻 Activity **(20 minutes)**

Build your own themed AI assistant endpoint. Pick one of the following themes, or come up with your own:

- 🛍️ **Product Recommender** — helps users find the right product based on their needs
- 📚 **Study Buddy** — helps students understand Java concepts in simple terms
- 🍽️ **Recipe Suggester** — suggests recipes based on ingredients the user has
- 💼 **Interview Coach** — helps developers prepare for Java technical interviews

Your task:
1. Create a new `@GetMapping` endpoint in `AiController.java` with a path of your choice
2. Write a system prompt that gives the AI a clear role and personality for your theme
3. Accept a `message` query parameter from the user
4. Test your endpoint with at least 3 different messages and observe the responses

**Hint:** A good system prompt usually includes:
- Who the AI is (role)
- What it helps with (scope)
- How it should respond (tone/style)
- What it should not do (boundaries)

---

## Summary

In this lesson you saw how Spring AI lets you add LLM capabilities to a Spring Boot application with minimal code. The key concepts to remember:

- The `spring-ai-openai-spring-boot-starter` dependency + BOM wires everything up automatically
- `ChatClient` is your main interface for sending prompts and receiving responses
- `ChatClient.Builder` is injected by Spring — the same DI pattern you already know
- `.prompt().user("...").call().content()` is the standard pattern for a simple chat call
- A **system prompt** (`.system("...")`) shapes the AI's role and behaviour before the user's message

This is just the beginning — Spring AI also supports conversation memory, file uploads, function calling, and Retrieval Augmented Generation (RAG). These are topics you will explore further as you progress in the programme.

---

END