# :snowflake: Embedding Model Eval Demo :snowflake:

## In this demo repo we will explore the following

### EMBED_AND_INDEX.ipynb
 - Read data into snowflake containing writeups of support ticket conversations
 - Use external access integrations to allow communication with api.openai.com to call an Open AI embedding model on your Snowflake data (Note - you must provide your own OAI API Key!)
 - Build a Cortex Search Service on top of the embeddings created with the OAI embedding model
 - Build a second Cortex Search Service on the raw text data, using built-in Arctic Embedding models hosted in Snowflake

### EMBEDDING_EVAL.ipynb
 - Test out Cortex Search Service and experiment with confidence score thresholds
 - Set up a class with various functions to perform RAG and instrument for Evaluation and Tracing in Snowflake AI Observability
 - Instantiate two versions of the class - one with the Cortex Search Service on the OAI embeddings and one with Arctic embeddings
 - Define these applications in Snowflake's AI Observability Framework
 - Create runs with a batch of input prompts to evaluate various aspects of these applications' performance


### Snowflake AI Observability UI
- Review and compare each application side-by-side to better understand
  - Retrieval Quality
  - Model Quality
  - Latency
  - Cost
- Consider which application would be the best candidate for production based on insights


SETUP INSTRUCTIONS

```
-- Create DB, Schema, and Warehouse
CREATE DATABASE IF NOT EXISTS EMBEDDING_EVAL_DB;
CREATE SCHEMA IF NOT EXISTS DATA;
CREATE WAREHOUSE IF NOT EXISTS MEDIUM WITH WAREHOUSE_SIZE='MEDIUM';
 
--Create network rule and api integration to install packages from pypi
CREATE OR REPLACE NETWORK RULE OAI_network_rule
 MODE = EGRESS
 TYPE = HOST_PORT
 VALUE_LIST = ('api.openai.com', 'platform.openai.com');

  -- Create external access integration on top of network rule for pypi access
CREATE OR REPLACE EXTERNAL ACCESS INTEGRATION OAI_EAI
 ALLOWED_NETWORK_RULES = (OAI_network_rule)
 ENABLED = true;

-- Create an API integration with Github
CREATE OR REPLACE API INTEGRATION GIT_INTEGRATION_EBOTWICK
   api_provider = git_https_api
   api_allowed_prefixes = ('https://github.com/sfc-gh-ebotwick')
   enabled = true
   comment='Git integration with Elliott Botwick''s repositories';

-- Create the integration with the Github demo repository
CREATE OR REPLACE GIT REPOSITORY EMBEDDING_EVAL_GIT_REPO
   ORIGIN = 'https://github.com/sfc-gh-ebotwick/embed_model_eval_demo.git' 
   API_INTEGRATION = 'GIT_INTEGRATION_EBOTWICK' 
   COMMENT = 'Github Repository ';

-- Fetch most recent files from Github repository
ALTER GIT REPOSITORY EMBEDDING_EVAL_GIT_REPO FETCH;

-- Copy Cortex Search Setup notebook into snowflake configure runtime settings
CREATE OR REPLACE NOTEBOOK EMBEDDING_EVAL_DB.DATA.EMBED_AND_INDEX
FROM '@EMBEDDING_EVAL_DB.DATA.EMBEDDING_EVAL_GIT_REPO/branches/main/' 
MAIN_FILE = 'EMBED_AND_INDEX.ipynb' QUERY_WAREHOUSE = MEDIUM
RUNTIME_NAME = 'SYSTEM$BASIC_RUNTIME' 
IDLE_AUTO_SHUTDOWN_TIME_SECONDS = 3600;

-- Copy AI Obs notebook into snowflake configure runtime settings
CREATE OR REPLACE NOTEBOOK EMBEDDING_EVAL_DB.DATA.EMBEDDING_EVALS
FROM '@EMBEDDING_EVAL_DB.DATA.EMBEDDING_EVAL_GIT_REPO/branches/main/' 
MAIN_FILE = 'EMBEDDING_EVALS.ipynb' QUERY_WAREHOUSE = MEDIUM
RUNTIME_NAME = 'SYSTEM$BASIC_RUNTIME' 
IDLE_AUTO_SHUTDOWN_TIME_SECONDS = 3600;

-- Enable external access integration for web search tool in AI Agent Noteobok
alter NOTEBOOK EMBEDDING_EVAL_DB.DATA.EMBED_AND_INDEX set EXTERNAL_ACCESS_INTEGRATIONS = ( 'OAI_EAI' );
alter NOTEBOOK EMBEDDING_EVAL_DB.DATA.EMBEDDING_EVALS set EXTERNAL_ACCESS_INTEGRATIONS = ( 'OAI_EAI' );
```
