# Overview

This project involves building a world-building application that
integrates TOML for configuration, AutoGen to manage AI agents, and
Azure for cloud deployment. The project includes designing a relational
database in Azure SQL for core world-building data, setting up a vector
database (Chroma or Pinecone) to handle AI agent queries, and
integrating image generation/recognition AI. The AI agents can retrieve
and work with this data to simulate characters, cultures, and historical
events.

# Technologies Used

-   **TOML**: Configuration and modular data storage format.

-   **AutoGen**: AI agent framework.

-   **Azure SQL Database**: Relational database for storing
    world-building data.

-   **Pinecone/Chroma**: Vector database for semantic search.

-   **LangChain**: Framework for building logical chains for AI agents.

-   **OpenAI API**: For language and image generation tasks.

-   **Azure Cognitive Services**: For image recognition and generation.

-   **Python**: Development language.

-   **Azure DevOps**: CI/CD pipelines.

# Project Components

## Database Design (TOML + Azure SQL)

The core data for world-building will be stored in TOML for modularity
and will be loaded into an Azure SQL database for querying. TOML will
store cultures, religions, characters, events, and other static data,
while Azure SQL will handle dynamic operations.

## Vector Database for AI Agents (Pinecone or Chroma)

The vector database allows AI agents to retrieve relevant data quickly.
The vector database can store embeddings for events, characters, and
cultural information.

## AI Agents with AutoGen and LangChain

AutoGen and LangChain will work together to create AI agents that
retrieve and interact with the world-building data. AutoGen manages the
AI agents, while LangChain allows for chaining multiple tasks for more
complex queries.

## Image Generation and Recognition

Using Azure Cognitive Services for image recognition and OpenAI API for
image generation, you can integrate visual elements into the
world-building process. Agents can retrieve, recognize, and generate
images based on textual descriptions or historical events.

# Step-by-Step Setup

## Azure Setup

1.  Create an Azure SQL Database to store world-building data.

2.  Set up Azure Cognitive Services for image recognition and analysis.

3.  Set up Azure DevOps for CI/CD pipelines.

## Database Design and Management

Create the schema for storing core world-building data. Use TOML for
configuration and Azure SQL for querying data.

## Setting Up AutoGen with AI Agents

Install AutoGen using Python and define agents to interact with the
database and vector store.

## Implementing the Vector Database

Set up Pinecone or Chroma for storing embeddings of world-building data.
The embeddings will be used by AI agents to perform semantic search.

## Image Generation and Recognition Integration

Integrate Azure Cognitive Services for image recognition and the OpenAI
DALL-E API for image generation. This allows agents to interact with
images, recognize symbols, or generate visual representations of events.

# Code Examples and Tools

## Python Code for Database Integration

    import pyodbc

    def connect_to_azure_sql():
        conn = pyodbc.connect(
            'DRIVER={ODBC Driver 17 for SQL Server};'
            'SERVER=tcp:<your-server-name>.database.windows.net,1433;'
            'DATABASE=<your-database-name>;'
            'UID=<your-username>;'
            'PWD=<your-password>'
        )
        return conn

## TOML Data Example

    [cultures.egypt]
    name = "Ancient Egypt"
    location = "Nile Valley"
    time_period = "3100 BCE - 30 BCE"
    notable_figures = ["Tutankhamun", "Cleopatra"]
    religions = ["Egyptian Religion"]

# Tooling for Image Generation and Recognition

## Image Generation with OpenAI API

    import openai

    openai.api_key = "your-api-key"

    def generate_image(prompt):
        response = openai.Image.create(
            prompt=prompt,
            n=1,
            size="1024x1024"
        )
        return response['data'][0]['url']

## Image Recognition with Azure Cognitive Services

    from azure.cognitiveservices.vision.computervision import ComputerVisionClient
    from msrest.authentication import CognitiveServicesCredentials

    def analyze_image(image_path):
        client = ComputerVisionClient(endpoint="https://<your-endpoint>.cognitiveservices.azure.com/",
                                      credentials=CognitiveServicesCredentials("<your-api-key>"))
        with open(image_path, "rb") as image_stream:
            result = client.describe_image_in_stream(image_stream)
        return result.captions[0].text

# Diagram

::: center
:::

# Additional Resources

-   [Azure SQL
    Documentation](https://docs.microsoft.com/en-us/azure/azure-sql/)

-   [AutoGen Documentation](https://github.com/autogen-ai/autogen)

-   [Pinecone Documentation](https://www.pinecone.io/docs/)

-   [Chroma Documentation](https://docs.trychroma.com/)
