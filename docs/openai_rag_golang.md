> **This guide is influenced by [this walkthrough](https://docs.leapfrog.ai/docs/dev-with-lfai-guide/dev_guide/) of using the OpenAI SDK with LeapfrogAI**. This guide uses the official [openai-go](https://github.com/openai/openai-go) API library for use with Golang to create a simple demonstration of RAG using OpenAI. This library is in Alpha so these instructions may need updating over time. This guide also does not leverage LeapfrogAI and uses the actual OpenAI API.

## Basic Usage of OpenAI in Go

### OpenAI API Reference

The best place to look for help with using the OpenAI Golang library is to refer to the [package reference](https://pkg.go.dev/github.com/openai/openai-go#section-documentation) for `openai-go`, so it is recommended to return to this reference when understanding how specific API functionality works.

### Requirements

It's required to use Go version `1.18+`.

You'll need to install the library:

```bash
go get github.com/openai/openai-go
```

You will also need an [OpenAI API Key](https://help.openai.com/en/articles/4936850-where-do-i-find-my-openai-api-key).

### Creating the Client

Now that you have your API key, you can create an OpenAI client in a Go package:

```go
package main

import (
    "github.com/openai/openai-go"
)

// API key (recommended that this is set as an environment variable)
const OPENAI_API_KEY = "api-key" // insert actual API key here

func main() {
    // Create an openai client
    client := openai.NewClient(
        option.WithAPIKey(OPENAI_API_KEY),
    )
...
```

### Running Chat Completions

Now that you have a client created, you can utilize it to handle basic chat completion requests:

```go
... // Using the same code from above

    // Create a context for the API
    ctx := context.Background()

    question := "Please tell me a fun fact about frogs."

    print("> ")
    println(question)
    println()

    // Make a completion request
    completion, err := client.Chat.Completions.New(ctx, openai.ChatCompletionNewParams{
        Messages: openai.F([]openai.ChatCompletionMessageParamUnion{
            openai.UserMessage(question),
        }),
        Seed:  openai.Int(1), // set a seed for more deterministic sampling
        Model: openai.String(openai.ChatModelGPT4o), // use any OpenAI model you like
    })
    if err != nil {
        panic(err)
    }

    println(completion.Choices[0].Message.Content)

}
```

This is just a basic example; check out the [chat completion reference](https://github.com/openai/openai-go/blob/main/api.md#chat) for more options!

## Building a RAG Pipeline using Assistants

Now that we've seen a basic example, let's leverage OpenAI assistants to handle a more complex task: [**Retrieval Augmented Generation (RAG)**](https://blogs.nvidia.com/blog/what-is-retrieval-augmented-generation/).

We'll break this example down into a few steps:

### Requirements

Referencing the [Basic Usage](#basic-usage-of-the-openai-sdk-with-leapfrogai) section, you'll need:

- An OpenAI API key
- An OpenAI Client

### Create a Vector Store

A [vector database](https://www.pinecone.io/learn/vector-database/) is a fundamental piece of RAG-enabled systems. Vector databases store vectorized representations of our documents, which aids in being able to properly search the vector store for the data we need via [similarity search](https://www.pinecone.io/learn/what-is-similarity-search/). Creating a vector store instance is the first step to building a RAG pipeline.

Assuming you've created an OpenAI client as detailed above, create a vector store to contain our vectorized files:

```go
// Create a vector store
vector_store, err := client.Beta.VectorStores.New(ctx, openai.BetaVectorStoreNewParams{
    Name: openai.String("RAG Demo Vector Store"),
    ExpiresAfter: openai.F(openai.BetaVectorStoreNewParamsExpiresAfter{
        Anchor: openai.F(openai.BetaVectorStoreNewParamsExpiresAfterAnchorLastActiveAt),
        Days:   openai.F(int64(1)),
    }),
})
if err != nil {
    panic(err)
}
```

### Upload a file

Now that you have a vector store, let's add some documents. For a simple example, let's assume you have two text files with the following contents:

**doc_1.txt**

```text
Joseph has a pet frog named Milo.
```

**doc_2.txt**

```text
Milo's birthday is on October 7th.
```

Create these documents so you can add them to the vector store:

```go
// Upload files
docs := []string{"doc_1.txt", "doc_2.txt"}
for _, doc := range docs {
    file, err := os.Open(doc)
    if err != nil {
        panic(err)
    }
    defer file.Close()

    doc_params := openai.FileNewParams{
        File:    openai.F[io.Reader](file),
        Purpose: openai.F(openai.FilePurposeAssistants),
    }
    file_id, err := client.Files.New(ctx, doc_params)
    if err != nil {
        panic(err)
    }
    _, err = client.Beta.VectorStores.Files.New(ctx, vector_store.ID, openai.BetaVectorStoreFileNewParams{
        FileID: openai.F(string(file_id.ID)),
    })
    if err != nil {
        panic(err)
    }
    file.Close()
    println(fmt.Sprintf("File added to vector store: %v", doc))
}
```

When you upload files to a vector store, this creates a `VectorStoreFile` object. You can record these for later usage, but for now they aren't needed for simple chatting with your documents.

### Create an Assistant

[OpenAI Assistants](https://platform.openai.com/docs/assistants/overview) carry specific instructions and can reference specific tools to add functionality to your workflows. In this case, we'll add the ability for this assistant to search files in our vector store using the `file_search` tool. This `file_search` tool is attached our vector store as a resource so that the assistant can only query the vector store we specify.

```go
// Create an assistant
const INSTRUCTIONS = `You are a helpful AI bot that answers questions for a user. Keep your response short and direct.
    You may receive a set of context and a question that will relate to the context.
    Do not give information outside the document or repeat your findings.`

assistant, err := client.Beta.Assistants.New(ctx, openai.BetaAssistantNewParams{
    Name:         openai.String("Frog Buddy"),
    Instructions: openai.String(INSTRUCTIONS),
    Tools: openai.F([]openai.AssistantToolUnionParam{
        openai.FileSearchToolParam{Type: openai.F(openai.FileSearchToolTypeFileSearch)},
    }),
    ToolResources: openai.F(openai.BetaAssistantNewParamsToolResources{
        FileSearch: openai.F(openai.BetaAssistantNewParamsToolResourcesFileSearch{
            VectorStoreIDs: openai.F([]string{vector_store.ID}),
        }),
    }),
    Model: openai.String(openai.ChatModelGPT4o),
})
if err != nil {
    panic(err)
}
```

### Create a Thread and User Message

Now that we have an assistant that is able to pull context from our vector store, let's query the assistant. This is done with the usage of threads and runs. A thread in OpenAI is a session between an Assistant and a user, and a run is an invocation of an assistant on said thread. See the [assistants overview](https://platform.openai.com/docs/assistants/overview) for more info.

We'll make a query specific to the information in the documents we've uploaded:

```go
// Create a thread
thread, err := client.Beta.Threads.New(ctx, openai.BetaThreadNewParams{})
if err != nil {
    panic(err)
}

// Create a message in the thread
request_message, err := client.Beta.Threads.Messages.New(ctx, thread.ID, openai.BetaThreadMessageNewParams{
    Role: openai.F(openai.BetaThreadMessageNewParamsRoleUser),
    Content: openai.F([]openai.MessageContentPartParamUnion{
        openai.TextContentBlockParam{
            Type: openai.F(openai.TextContentBlockParamTypeText),
            Text: openai.String("When is the birthday of Joseph's pet frog?"),
        },
    }),
})
if err != nil {
    panic(err)
}
```

The thread we've created will track the interactions between the User and the Assistant. The request message we've created is the question that we, the user, is asking to the assistant to answer. However, the assistant doesn't do anything with this User message until we create a run for it.

You'll also notice that both documents are needed in order to answer the question we're asking. One contains the actual birthday date, while the other contains the relationship information between Joseph and Milo the frog. This is one of the reasons LLMs are utilized when extracting information from documents; they can integrate specific pieces of information across multiple sources.

### Run the Inquiry get a Response

We'll now create a run in order to evaluate our request:

```go
// Create a run
run, err := client.Beta.Threads.Runs.New(ctx, thread.ID, openai.BetaThreadRunNewParams{
    AssistantID: openai.String(assistant.ID),
    Include:     openai.F([]openai.RunStepInclude{openai.RunStepIncludeStepDetailsToolCallsFileSearchResultsContent}),
})
if err != nil {
    panic(err)
}

println("Waiting for Run to complete...")
for run.Status != openai.RunStatusCompleted {
    run, err = client.Beta.Threads.Runs.Get(ctx, thread.ID, run.ID)
    if err != nil {
        panic(err)
    }
    println("Run status: ", run.Status)
    time.Sleep(2 * time.Second)
}
println("Run Completed!")
```

After the run is created, it gets queued depending on resource availability. To make sure we don't get ahead of the run before it finishes, we wait until the run is completed.

### View the Response

With the run executed, you can now list the messages associated with that run to get the response to our query.

```go
// List messages
messages, err := client.Beta.Threads.Messages.List(ctx, thread.ID, openai.BetaThreadMessageListParams{
    RunID: openai.F(string(run.ID)),
})
if err != nil {
    panic(err)
}

println()
println(">", request_message.Content[0].Text.Value)
for messages != nil {
    for _, message := range messages.Data {
        println(message.Content[0].Text.Value)
    }
    messages, err = messages.GetNextPage()
    if err != nil {
        panic(err)
    }
}
```

The output will look something like this:

```text
Joseph's pet frog, Milo, has a birthday on October 7th. 【4:0†doc_2.txt】 【4:0†doc_1.txt】
```

As you can see, our "Frog Buddy" assistant was able to recieve the contextual information it needed in order to know how to answer the query. You'll also notice that the attached annotations correspond to the files we uploaded earlier, so we know we're pulling our information from the right place!

This just scratches the surface of what you can create with the OpenAI SDK. This may be a simple example that doesn't necessarily require the added overhead of RAG, but when you need to search for information hidden in hundreds or thousands of documents, you may not be able to hand your LLM all the data at once, which is where RAG really comes in handy.

As a reminder, the [OpenAI API Reference](https://platform.openai.com/docs/api-reference/introduction) has lots of information on using OpenAI!
