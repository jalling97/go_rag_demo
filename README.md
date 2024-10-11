# go_rag_demo
A quick demonstration of Retrieval Augmented Generation using golang and OpenAI.

This demo utilizes the [openai-go](https://github.com/openai/openai-go/) library.

## Guide
This repository contains a [guide](docs/openai_rag_golang.md) of basic usage of the OpenAI-API using golang, including chat completions as well as an example of Retrieval Augmented Generation (RAG). The guide contains descriptions and explanations of each step, as well as links to helpful resources to understand more about the OpenAI API and RAG.

To follow along with the guide, you will need an [OpenAI API Key](https://help.openai.com/en/articles/4936850-where-do-i-find-my-openai-api-key).

## RAG Demo Code
The code snippets detailed in the guide pertaining to RAG can also be found in the [demo](demo/) folder of this repository, if you prefer to see all the pieces in action.

### Requirements

- Go version `1.18+`
- OpenAI API Key set to `OPENAI_API_KEY` as an environment variable

### Running the Demo

Execute the following via the command line:

```bash
# clone the repository
git clone https://github.com/jalling97/go_rag_demo.git

# go to the demo code
cd demo/

# run the demo
go run rag.go

# OR build as an executable 
go build
./demo
```

