# Lesson: Coaching — Introduction to Spring AI

## Lesson Overview

In this session you will connect a Spring Boot application to a Large Language Model (LLM) such as OpenAI's GPT using **Spring AI**, a framework that makes the integration straightforward. You will go from a blank Spring Boot project to a working AI-powered REST endpoint in a single session, then customise the AI's behaviour with a system prompt. No prior AI experience is required.

> **Note:** This lesson uses **Spring AI 2.0** on **Spring Boot 4.x** with **Java 21**. Spring AI 2.0 requires Spring Boot 4, so create your project on the current Spring Boot 4.x offered by Spring Initializr.

## Lesson Objectives

By the end of this lesson, students will be able to:

1. **Add** Spring AI to a Spring Boot project and configure it with an API key
2. **Build** a REST endpoint that sends a user prompt to an LLM and returns the response
3. **Customise** AI behaviour by applying a system prompt

---

## Prerequisites

This lesson assumes you are comfortable with Spring Boot basics — creating a project, writing a controller, using `application.properties`, and Dependency Injection.

You will also need your own **API key** to run the code. See **api-key-setup.md** for how to create one. There are two options in that document, and either works for this lesson. Have your key saved and ready before the session.

---

## Part 1: What is Spring AI?

Spring AI is an official Spring project that makes it easy to integrate AI models — such as OpenAI, Google Gemini, and Anthropic Claude — into your Java applications. Think of it as the "Spring Data" of the AI world: instead of writing raw HTTP calls to OpenAI's API yourself, Spring AI provides ready-made Spring beans and clean abstractions so you can focus on building features rather than dealing with API plumbing.

Before Spring AI existed, integrating an LLM into a Java application meant manually writing HTTP clients, handling authentication, parsing JSON responses, and managing retries. Spring AI handles all of that for you.

**Why does this matter?**

AI features are now a standard part of enterprise applications — chatbots, intelligent search, automated summaries, and more. As Java developers, knowing how to wire an LLM into a Spring Boot backend puts you at a significant advantage. Today you will see just how few lines of code it actually takes.

Spring AI supports many providers (OpenAI, Anthropic, Google, Amazon Bedrock, Mistral, DeepSeek, Ollama, and more). In this lesson we will use **OpenAI** with the **GPT-4o-mini** model — it is fast, inexpensive, and ideal for learning.

> **Note:** Spring AI 2.0 uses each vendor's official SDK under the hood (for OpenAI, the `openai-java` SDK). This is invisible at the `ChatClient` level — the same code works regardless of provider — and it is the real SDK, not hand-rolled HTTP.

---

## Part 2: Project Setup

### Create a New Spring Boot Project

Create a new Spring Boot project using Spring Initializr (Ctrl/Cmd + Shift + P → "Spring Initializr: Create a Maven Project").

| Setting | Value |
|---|---|
| Spring Boot version | Current stable release offered by Spring Initializr (**Spring Boot 4.x**) |
| Language | Java |
| Group ID | `sg.edu.ntu` |
| Artifact ID | `spring-ai-demo` |
| Packaging | Jar |
| Java version | 21 |

For dependencies, select:
- **Spring Web**
- **Spring Boot DevTools**

We will add the Spring AI dependency manually in the next step.

> **Note:** In Spring Boot 4, the web starter appears in the generated `pom.xml` as `spring-boot-starter-webmvc` (renamed from `spring-boot-starter-web` in Spring Boot 3). Selecting "Spring Web" in Initializr adds the correct one automatically. Many online tutorials still show the old name.

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
      <version>2.0.0</version>
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
  <artifactId>spring-ai-starter-model-openai</artifactId>
</dependency>
```

> ⚠️ **Note on the artifact name:** Spring AI renamed its starter artifacts back in the 1.0 release. The old name (`spring-ai-openai-spring-boot-starter`) no longer exists in Maven Central. The correct artifact ID — still correct in 2.0 — is `spring-ai-starter-model-openai`. Many older tutorials on the internet still use the old name; if you follow them, your build will fail.

Save the file. Maven will download the dependencies automatically. You will see a prompt in VS Code to reload — click **Yes**.

### Configure the API Key

Open `src/main/resources/application.properties`. What you add here depends on which key you created in **api-key-setup.md**. Use **one** of the two blocks below — not both.

**If you are using an OpenAI key:**

```properties
# Spring AI - OpenAI Configuration
spring.ai.openai.api-key=YOUR_API_KEY_HERE
spring.ai.openai.chat.model=gpt-4o-mini
spring.ai.openai.chat.temperature=0.7
```

**If you are using an OpenRouter key:**

```properties
# Spring AI - OpenRouter Configuration
spring.ai.openai.api-key=YOUR_API_KEY_HERE
spring.ai.openai.base-url=https://openrouter.ai/api/v1
spring.ai.openai.chat.model=openrouter/free
spring.ai.openai.embedding.model=nvidia/llama-nemotron-embed-vl-1b-v2:free
spring.ai.openai.embedding.encoding-format=float
```

Replace `YOUR_API_KEY_HERE` with your own key. If you have not created one yet, see **api-key-setup.md** — it only takes a few minutes.

> **Note:** The two `embedding` lines in the OpenRouter block are not used in this lesson. They are needed in the third Spring AI lesson, so set them now and you will not have to come back to this file.

**Why the OpenRouter block looks different**

OpenRouter speaks the same language as OpenAI, so Spring AI's OpenAI starter can talk to it without any code change. The `base-url` line simply tells your application to send its requests to OpenRouter's address instead of OpenAI's, and the model names are the ones OpenRouter uses. Everything else in this lesson — the dependency, the controller, the code you write — is identical whichever key you chose.

> ⚠️ **Property key change in Spring AI 2.0:** The model and temperature keys no longer contain an `.options` segment. In Spring AI 1.x these were `spring.ai.openai.chat.options.model` and `spring.ai.openai.chat.options.temperature`. In 2.0 they are flattened to `spring.ai.openai.chat.model` and `spring.ai.openai.chat.temperature`. The old `.options.` form still works through a deprecated alias, but use the flattened form. Nearly every online tutorial still shows the old `.options.` keys.

> ⚠️ **Important:** Never commit your API key to a public Git repository. For now, pasting it directly is fine for learning. In production, you would use environment variables or a secrets manager.

> ⚠️ **Common error:** If you see `HTTP 429`, it means you have run out of requests. On OpenAI this is a billing issue — add credits at [the billing overview](https://platform.openai.com/settings/organization/billing/overview). On OpenRouter it means you have used your free requests for the day; they reset the next day.

**What do these properties mean?**
- `api-key` — your credentials to access the service
- `base-url` — the address your application sends requests to. You only set this when using a provider other than OpenAI.
- `model` — which model answers your requests. `gpt-4o-mini` is fast and affordable; `openrouter/free` automatically picks an available free model.
- `temperature` — controls how creative/varied the responses are. `0.7` is a good balanced value. `0.0` is very deterministic; `1.0` is very creative. Note: Spring AI 2.0 no longer applies its own default temperature — it defers to the provider's default — so setting this explicitly is meaningful.

Run the application to confirm it starts without errors.

```bash
mvn spring-boot:run
```

---

## Part 3: Your First AI Endpoint

Now the interesting part. Create a new file `AiController.java` in your main package (`sg.edu.ntu`) and code along.

### Understanding ChatClient and ChatClient.Builder

Before we write the code, let's understand the two key Spring AI objects we are about to use.

**`ChatClient`** is the main Spring AI object you use to communicate with the LLM. Think of it like `JdbcTemplate` for databases — it is Spring's clean abstraction over all the raw HTTP calls, authentication, and JSON parsing that happen under the hood. You call methods on it to send prompts and receive responses.

**`ChatClient.Builder`** is a builder object that Spring AI auto-creates and registers as a bean. You do not create it yourself — Spring injects it for you. Its job is to configure and construct the `ChatClient` instance.

**Why `.build()`?** Because `ChatClient.Builder` is the factory, not the client itself. You call `.build()` once in the constructor to produce the ready-to-use `ChatClient`. From that point on, you use `chatClient` to send all your prompts.

```java
package sg.edu.ntu;

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

Notice how we are injecting `ChatClient.Builder` through the constructor. This is called **constructor injection** — it is another way to apply Dependency Injection, and is actually the preferred approach in modern Spring development. Instead of annotating a field with `@Autowired`, you declare the dependency as a constructor parameter and Spring automatically provides the bean when it creates the class. The end result is the same — Spring manages the object for you — but constructor injection makes dependencies more explicit and easier to test.

Now add our first endpoint inside the class.

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
- `.call()` — sends the request to the model and waits for the response
- `.content()` — extracts the response text as a plain `String`

Your complete `AiController.java` should now look like this:

```java
package sg.edu.ntu;

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

  @GetMapping("/chat")
  public String chat(@RequestParam String message) {
    return chatClient.prompt()
        .user(message)
        .call()
        .content();
  }

}
```

Run the application and test it in your browser:

```
localhost:8080/chat?message=What is Java?
```

You should see a response from the model. You have just built an AI-powered REST endpoint in a Spring Boot application. 🎉

Try a few different messages and observe the responses.

---

## Part 4: Adding a System Prompt

Right now, the AI will answer any question about any topic. In real applications, we usually want to give the AI a specific role and set of instructions — this is called a **system prompt**.

A system prompt is a behind-the-scenes instruction that you provide to the model before the user's message. It shapes how the AI responds — its tone, its focus area, and what it should or should not do.

Let's create a dedicated endpoint that uses a system prompt to turn our AI into a helpful customer support assistant for a CRM application.

Add this endpoint to your `AiController.java`:

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

Your complete `AiController.java` should now look like this:

```java
package sg.edu.ntu;

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

  @GetMapping("/chat")
  public String chat(@RequestParam String message) {
    return chatClient.prompt()
        .user(message)
        .call()
        .content();
  }

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

}
```

Test both endpoints in your browser:

```
localhost:8080/chat?message=How do I add a new customer?
```
```
localhost:8080/support?message=How do I add a new customer?
```
```
localhost:8080/support?message=What is the weather today?
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
4. Test your endpoint in the browser with at least 3 different messages and observe the responses

**Hint:** A good system prompt usually includes:
- Who the AI is (role)
- What it helps with (scope)
- How it should respond (tone/style)
- What it should not do (boundaries)

**Example — Interview Coach endpoint:**

```java
@GetMapping("/interview-coach")
public String interviewCoach(@RequestParam String message) {
  return chatClient.prompt()
      .system("You are an expert Java technical interview coach. " +
              "You help developers prepare for Java and Spring Boot job interviews. " +
              "Provide clear explanations, example interview questions, and model answers. " +
              "Keep your answers focused and practical. " +
              "If a question is not related to Java or software development interviews, politely redirect the user.")
      .user(message)
      .call()
      .content();
}
```

Test it in the browser:
```
localhost:8080/interview-coach?message=What is the difference between an interface and an abstract class?
```

---

## Summary

In this lesson you saw how Spring AI lets you add LLM capabilities to a Spring Boot application with minimal code. The key concepts to remember:

- We are on **Spring AI 2.0**, which runs on **Spring Boot 4**
- The `spring-ai-starter-model-openai` dependency + the `spring-ai-bom` (version `2.0.0`) wire everything up automatically
- The same starter works with other providers — pointing `base-url` at OpenRouter needs no code change at all
- Configuration property keys in 2.0 are **flattened** — `spring.ai.openai.chat.model`, not `...chat.options.model`
- **`ChatClient`** is your main interface for sending prompts and receiving responses — Spring AI's abstraction over the raw OpenAI API
- **`ChatClient.Builder`** is injected by Spring and used to construct the `ChatClient` via `.build()`
- `.prompt().user("...").call().content()` is the standard pattern for a simple chat call
- A **system prompt** (`.system("...")`) shapes the AI's role and behaviour before the user's message

This is just the beginning — Spring AI also supports conversation memory, file uploads, tool calling, and Retrieval Augmented Generation (RAG). These are topics you will explore further as you progress in the programme.

---

END