---
title: "Part 2 - Simple Gen AI Product Recommendation System"
description: "This simple Gen AI Product Recommendation System is designed for rapid proof-of-concept (POC) development. It leverages generative AI techniques to provide product recommendations to your customers."
date: 2024-09-05T10:49:01+10:00
image: cover.jpg
math: 
license: 
hidden: false
comments: true
draft: false
categories: [Azure, AI, Python]
tags: [Azure Open AI, Python]
keywords: [Gen AI, Recommendation System, Next Best Product, Azure Open AI, Python, AI, Azure]
---

> I am a Microsoft employee, but the views expressed here are mine and not those of my employer.

This is part 2 of an ongoing series on simple Gen AI Product Recommendation System that you can POC quickly. If you missed the first post in the series, you can find it [here](https://mfblog.au/p/simple-gen-ai-product-recommendation-system-you-can-poc-quickly/).

In this post, we will expand our simple Gen AI Product Recommendation System by increasing our data set and using cloud services to scale our solution. Lastly we will be using some more advanced prompt engineering techniques to improve the quality of our recommendations.

But...... why would you want to do this??? once you have completed this walk through, you will have a vector store setup in Azure and the ability to experiment with different searching techniques. As well as Azure Open AI setup to experiment with different prompt engineering techniques to get the best results for you project.

## TLDR

If you just want to see the code, you can find it [here](https://github.com/fredderf204/poc-recsys)

If you want to skip to the good stuff at the end of this blog post, you can find it searching techniques [here](#query-the-index) and prompt engineering techniques [here](#make-recommendations-using-azure-open-ai).

All you need is a CSV file or JSON file with customer and product performance data to use this sample. A sample CSV file has been included in the GitHub repo if you just want to run through the process.

## Pre-requisites

To follow along with this post, you will need the following already running in Azure:

1. Azure Open AI. I use the *text-embedding-3-large* model to vectorise the CSV or JSON file (more details on this later) and the question. I also use the *GPT-4o* model to recommend the next best product to the customer. You can sign up for Azure Open AI [here](https://azure.microsoft.com/en-us/services/cognitive-services/openai/)

2. Azure AI Search. I use Azure AI Search for various things in the sample, but mainly to index the JSON file, store the embeddings and search for similar customer profiles (more details on this later). You can sign up for Azure AI Search [here](https://azure.microsoft.com/en-us/services/search/).

3. Azure Blob Storage. I use Azure Blob Storage to store the CSV or JSON file.

## Introduction

In the first post, we built a simple Gen AI Product Recommendation System that used a small dataset to provide product recommendations to customers. We used a generative AI model to generate recommendations based on the customer's product performance. In this post, we will expand our dataset and use cloud services to scale our solution. We will also use some more advanced prompt engineering techniques to improve the quality of our recommendations.

The has 2 files that you need to follow in order;

1. [02-prepcsv.py](#02-prepcsvpy)
2. [02-csv.ipynb](#02-csvipynb)

Also I create a larger dataset using a combination of python and Microsoft excel. The dataset is called *test-recsys-data-5000.csv* and can be found in the *data* folder. It has 5000 rows of sample data.

## 02-prepcsv.py

This scripts does a couple of things to prepare the CSV file for indexing in Azure AI Search.

```python
import os
import pandas as pd

# Read the CSV file with specified data types
csv_input=pd.read_csv(
    os.path.join(os.getcwd(),'./data/test-recsys-data-5000.csv'),
    dtype={
        'CompanyName':str,
        'State':str,
        'Industry':str,
        'Segment':str,
        'Product 1 Name':str,'Product 1 Performance':float,
        'Product 2 Name':str,'Product 2 Performance':float,
        'Product 3 Name':str,'Product 3 Performance':float,
        'Product 4 Name':str,'Product 4 Performance':float,
        'Product 5 Name':str,'Product 5 Performance':float,
        'Product 6 Name':str,'Product 6 Performance':float,
        'Product 7 Name':str,'Product 7 Performance':float,
        'Product 8 Name':str,'Product 8 Performance':float
        }
    )
```

Lines 1 - 21 read the csv files, create a pandas dataframe and casts the columns to the correct data types.

```python
csv_input['json'] = csv_input.apply(lambda x: x.to_json(), axis=1)
```

Line 24 creates a new column called *json* and converts the row to a JSON string. The reason I do this is similar to the first post, when I have found the similar customer profiles to my target profile, I can use the JSON string as input to the generative AI model to recommend the next best product.

```python
performance_columns = [col for col in csv_input.columns if 'Performance' in col]
csv_input['performance'] = csv_input[performance_columns].mean(axis=1, skipna=True)
```

Line 27 and 28 create a new column called *performance* and calculates the mean of the performance columns. Just like in the first example, I have arbitrary numbers in the performance columns. You might have actual performance data in your dataset and a way of calculating the product portfolio performance of the customer. I simply take the mean of the performance columns to get a single number to represent the customer's product portfolio performance.

```python
cols=['State','Industry','Segment']
csv_input['prof'] = csv_input[cols].apply(lambda row: ' '.join(row.values.astype(str)), axis=1)
```

Again as in the first example, I am using the *State*, *Industry* and *Segment* columns to create a profile of the customer. I concatenate the values of these columns to create a single string that represents the customer profile. You might have another way of creating a customer profile or have more data points to create the customer profile, but this is a simple way to do it.

> The customer profile is important as is what's used to find similar customer profiles in the dataset. The more accurate the customer profile, the more chance of finding similar customer profiles.

```python
csv_input.to_json('02/data/output1.json', orient='records')
```

Finally I saved the dataframe to a JSON file. This is the file that will be uploaded to Azure Blob Storage and indexed in Azure AI Search. The reason I chose JSON is that it's easier to work with in Azure AI Search. So if you have started with JSON, you will just need to create a dataframe from that JSOn and start at line 24.

## 02-csv.ipynb

I have used an existing sample created my Microsoft and tweaked it for our needs. You can find the original sample [here](https://github.com/Azure/azure-search-vector-samples/blob/main/demo-python/code/indexers/csv.ipynb). The code flows as follows;

1. Setup Python environment
2. Upload the JSON file to Azure Blob Storage
3. Create a blob data source connector on Azure AI Search
4. Create a search index on Azure AI Search
5. Create an skillset on Azure AI Search
6. Create an indexer on Azure AI Search
7. Query the index
8. Make recommendations using Azure Open AI

I have added some additional code to the sample to make it easier to follow. The code is well commented and should be easy to follow.

## Setup Python environment

If you are not new to Python, this setup should be familiar to you. If you are new to Python, the notebook as some good detail on how to setup your Python environment using VS Code.

I can be a little tricky to figure out which environment variables to use and add to your .env file. So I have listed the ones that i used; AZURE_SEARCH_SERVICE_ENDPOINT, AZURE_SEARCH_ADMIN_KEY, AZURE_SEARCH_INDEX, BLOB_CONNECTION_STRING, BLOB_CONTAINER_NAME, AZURE_OPENAI_ENDPOINT, AZURE_OPENAI_KEY, AZURE_OPENAI_EMBEDDING_DEPLOYMENT, AZURE_OPENAI_EMBEDDING_MODEL_NAME, AZURE_OPENAI_EMBEDDING_DIMENSIONS, AZURE_OPENAI_CHATGPT_DEPLOYMENT and AZURE_OPENAI_API_VERSION.

## Upload the JSON file to Azure Blob Storage

This part of the notebook is pretty straight forward. If you have run through the 02-prepcsv.py script, you should have a JSON file called *output1.json* in the *data* folder. This code will take that file and upload it to Azure Blob Storage using the connection string and blob container name you have in your environment variables.

## Create a blob data source connector on Azure AI Search

Again, this part of the notebook is also straight forward. You will need to create a blob data source connector on Azure AI Search. The code will create the connector so it can be used in subsequent steps.

## Create a search index on Azure AI Search

OK, now for some more interesting stuff with Azure AI Search.

```python
index_client = SearchIndexClient(endpoint=endpoint, credential=credential)  
fields = [  
    SearchField(name="AzureSearch_DocumentKey",  key=True, type=SearchFieldDataType.String),
    SearchField(name="CompanyName", type=SearchFieldDataType.String, sortable=True, filterable=True, facetable=False),
    SearchField(name="performance", type=SearchFieldDataType.Double, sortable=True, filterable=True, facetable=False), 
    SearchField(name="prof", type=SearchFieldDataType.String, sortable=True, filterable=True, facetable=False), 
    SearchField(name="json", type=SearchFieldDataType.String, sortable=True, filterable=True, facetable=False), 
    SearchField(name="prof_vector", type=SearchFieldDataType.Collection(SearchFieldDataType.Single), vector_search_dimensions=azure_openai_model_dimensions, vector_search_profile_name="myHnswProfile"),
]  
```

This code snippet creates a search index client and is the configuration to create the search index. The index has 5 fields, *AzureSearch_DocumentKey*, *CompanyName*, *performance*, *prof* and *json*. The *AzureSearch_DocumentKey* is the unique key for the index and is automatically generated by the indexer. The *CompanyName* is the name of the company. The *performance* is the performance of the customer's product portfolio. The *prof* is the customer profile. The *json* is the JSON string of the customer's product performance. The *prof_vector* is the vectorised version of the customer profile.

Notice that theses fields are the same as the columns in the JSON file. This is important as the indexer will use these fields to index the JSON file.

Lastly, the *prof_vector* is something that doesn't exist yet. This will be a vectorised version of the customer profile and will be created by our indexer later in this code. So for now, just know that it will be created and will hold the vectorised version of the customer profile.

```python
# Configure the vector search configuration  
vector_search = VectorSearch(  
    algorithms=[  
        HnswAlgorithmConfiguration(name="myHnsw"),
    ],  
    profiles=[  
        VectorSearchProfile(  
            name="myHnswProfile",  
            algorithm_configuration_name="myHnsw",  
            vectorizer="myOpenAI",  
        )
    ],  
    vectorizers=[  
        AzureOpenAIVectorizer(  
            name="myOpenAI",  
            kind="azureOpenAI",  
            azure_open_ai_parameters=AzureOpenAIParameters(  
                resource_uri=azure_openai_endpoint,  
                deployment_id=azure_openai_embedding_deployment,
                model_name=azure_openai_model_name,
                api_key=azure_openai_key,
            ),
        ),  
    ],  
) 
```

This next part of the code will configure the vector search configuration. I used the default HNSW algorithm and the Azure Open AI vectoriser. If you would like to learn more about the HNSW algorithm, you can find more information [here](https://learn.microsoft.com/en-us/azure/search/vector-search-overview#nearest-neighbors-search).

```python
semantic_config = SemanticConfiguration(  
    name="my-semantic-config",  
    prioritized_fields=SemanticPrioritizedFields(
        title_field=SemanticField(field_name="CompanyName"),
        content_fields=[SemanticField(field_name="prof")]  
    ),  
)
```

Now this section of the code is where you configure the semantic configuration. The semantic configuration is used to configure semantic ranking. [Semantic ranking iterates over an initial result set, applying an L2 ranking methodology that promotes the most semantically relevant results to the top of the stack.](https://learn.microsoft.com/en-us/azure/search/semantic-how-to-configure). I have added this section if some we can play around with semantic ranking later in this code.

## Create an skillset on Azure AI Search

This part of the code creates a skillset on Azure AI Search. The skillset is used to create the vectorised version of the customer profile. The skillset uses the Azure Open AI vectoriser to create the vectorised version of the customer profile.

```python
csv_comb_embedding_skill = AzureOpenAIEmbeddingSkill(  
    description="Skill to generate comb embeddings via Azure OpenAI",  
    context="/document",  
    resource_uri=azure_openai_endpoint,  
    deployment_id=azure_openai_embedding_deployment,  
    model_name=azure_openai_model_name,
    dimensions=azure_openai_model_dimensions,
    api_key=azure_openai_key,  
    inputs=[  
        InputFieldMappingEntry(name="text", source="/document/prof"),  
    ],  
    outputs=[  
        OutputFieldMappingEntry(name="embedding", target_name="prof_vector"),  
    ],  
)

skills = [csv_comb_embedding_skill]
```

This code snippet above is the configuration for the skillset. The main parts of the configuration to look out for are the *context* - which is the path to the document in the JSON file. I just left this as the default /document. The *inputs* - which is the field in the JSON file that will be vectorised. *name* should be text and *source* should be /document/prof, as it's the prof field we want vectorised. And lastly, the *outputs* - which is the field in the search index that will hold the vectorised version of the customer profile. *name* should be embedding  and *source* should be prof_vector.

## Create an indexer on Azure AI Search

The indexer is the part of Azure AI Search that brings together the data source, the search index and the skillset. The indexer will use the data source to index the JSON file, use the skillset to create the vectorised version of the customer profile and use the search index to store the data.

```python
indexer_name = f"{index_name}-indexer"  
indexer_parameters = IndexingParameters(
        configuration=IndexingParametersConfiguration(
            parsing_mode='jsonArray',
            query_timeout=None,
            first_line_contains_headers=True))
```

This code snippet above is the configuration for the indexing parameters. The main parts of the configuration to look out for are the *parsing_mode* - which is the mode the indexer will use to parse the JSON file. I have used jsonArray as the JSON file is an array of JSON objects and this parsing mode will take each JSON object and create a document in the search index. I.e. one JSON file to many documents in the search index.

> There are other types of parsing modes out there for Azure AI Search to parse standalone files or embedded objects of various content types in data sources like Azure Blob Storage, Azure Data Lake Storage Gen2, and SharePoint. You can learn about them [here](https://learn.microsoft.com/en-us/azure/search/search-blob-metadata-properties).

```python
indexer = SearchIndexer(  
    name=indexer_name,  
    description="Indexer to index documents and generate embeddings",  
    skillset_name=skillset_name,  
    target_index_name=index_name,  
    data_source_name=data_source.name,
    parameters=indexer_parameters,
    field_mappings=[FieldMapping(source_field_name="AzureSearch_DocumentKey", target_field_name="AzureSearch_DocumentKey", mapping_function=FieldMappingFunction(name="base64Encode"))],
    output_field_mappings=[
        FieldMapping(source_field_name="/document/prof_vector", target_field_name="prof_vector"),
    ]
)  
```

Now for the search indexer parameters configuration, which is above. This is the part that ties together the data source, the search index and the skillset. The main parts of the configuration to look out for are the *skillset_name* - which is the name of the skillset that will be used to create the vectorised version of the customer profile. The *target_index_name* - which is the name of the search index that will store the data. The *data_source_name* - which is the name of the data source that will be used to index the JSON file. The *field_mappings* - which is the field mapping that will be used to map the system created *AzureSearch_DocumentKey* field to the *AzureSearch_DocumentKey* field in the search index. The *output_field_mappings* - which is the field mapping that will be used to map the *prof_vector* field in the JSON file to the *prof_vector* field in the search index.

## Query the index

Now we can have some fun!

There are different types of queries you can run on the search index which fall in the retrieval bucket, which you can read about [here](https://learn.microsoft.com/en-us/azure/search/search-query-overview#types-of-queries). The good news is that we are setup to try out full text search, vector search and hybrid search. Also we are setup to try out semantic ranking.

There is a good blog post [here](https://techcommunity.microsoft.com/t5/ai-azure-ai-services-blog/azure-ai-search-outperforming-vector-search-with-hybrid/ba-p/3929167) which goes into more details about all of this. 

But the purpose of the next part of the notebook is to return the top 20 similar customer profiles to the target customer profile. I have picked 20 for no particular reason, but feel free to try out some different numbers and see how that affects your results.

```python
target_company = '[{"CompanyName":"Madeup Inc","State":"NSW","Industry":"Finance","Segment":"Banking","Product 1 Name":"Widget W","Product 1 Performance":85,"Product 2 Name":"","Product 2 Performance":"","Product 3 Name":"Widget Y","Product 3 Performance":74}]'
company_data = json.loads(target_company)[0]
query = f"{company_data['State']} {company_data['Industry']} {company_data['Segment']}"
print(f"Query: {query}")
```

For this code snippet, I have made up a target customer profile called *target_company*. This is a JSON string that represents the target customer full information, including individual product performance. I have used the *State*, *Industry* and *Segment* columns to create the target customer profile, which I have saved as *query*. 

```python
search_client = SearchClient(endpoint, index_name, credential=credential)
vector_query = VectorizableTextQuery(text=query, k_nearest_neighbors=50, fields="prof_vector")
  
results = search_client.search(  
    search_text=query,  
    #vector_queries= [vector_query],
    select=["CompanyName", "prof", "performance", "json"],
    filter="performance ge 60",
    top=20,
    include_total_count=True
)  

print ('Total Documents Matching Query:', results.get_count())
print("-" * 50)

comp_profiles = []

for result in results:
    print(f"Score: {result['@search.score']}")  
    print(f"CompanyName: {result['CompanyName']}")  
    print(f"profile: {result['prof']}")  
    print(f"performance: {result['performance']}")
    print(f"json: {result['json']}")
    comp_profiles.append(result['json'])
    print("-" * 50)  
```

Now if you look at the code above, I have created a search client and I am only using the *search_text* parameter, which means I am doing a full text search. I have also commented out the *vector_queries* parameter, which means I am not doing a vector search (but we can add it in right after this).

> Now I have added the line *filter="performance ge 60"* to filter out any search results that do not have high product performance. This is because later in the code I want to make recommendations based on the high performing products of the customer. This is by no means a perfect solution, but it's a simple way to filter out low performing products.

Which produces the below results, which I have truncated for brevity;

```bash
Query: NSW Finance Banking
Total Documents Matching Query: 404
--------------------------------------------------
Score: 13.586723
CompanyName: Sky Robotics
profile: NSW Finance Banking
performance: 61.25
json: {"CompanyName":"Sky Robotics","State":"NSW","Industry":"Finance","Segment":"Banking","Product 1 Name":"Widget W","Product 1 Performance":75.0,"Product 2 Name":"Widget X","Product 2 Performance":87.0,"Product 3 Name":null,"Product 3 Performance":null,"Product 4 Name":"Widget Z","Product 4 Performance":21.0,"Product 5 Name":null,"Product 5 Performance":null,"Product 6 Name":null,"Product 6 Performance":null,"Product 7 Name":null,"Product 7 Performance":null,"Product 8 Name":"Gizmo D","Product 8 Performance":62.0}
--------------------------------------------------
Score: 13.586723
CompanyName: Consulting Apex
profile: NSW Finance Banking
performance: 61.0
json: {"CompanyName":"Consulting Apex","State":"NSW","Industry":"Finance","Segment":"Banking","Product 1 Name":null,"Product 1 Performance":null,"Product 2 Name":null,"Product 2 Performance":null,"Product 3 Name":"Widget Y","Product 3 Performance":58.0,"Product 4 Name":"Widget Z","Product 4 Performance":81.0,"Product 5 Name":null,"Product 5 Performance":null,"Product 6 Name":null,"Product 6 Performance":null,"Product 7 Name":"Gizmo C","Product 7 Performance":56.0,"Product 8 Name":"Gizmo D","Product 8 Performance":49.0}
--------------------------------------------------
Score: 13.266863
CompanyName: Orion Consulting
profile: NSW Finance Banking
performance: 65.0
json: {"CompanyName":"Orion Consulting","State":"NSW","Industry":"Finance","Segment":"Banking","Product 1 Name":null,"Product 1 Performance":null,"Product 2 Name":null,"Product 2 Performance":null,"Product 3 Name":null,"Product 3 Performance":null,"Product 4 Name":null,"Product 4 Performance":null,"Product 5 Name":null,"Product 5 Performance":null,"Product 6 Name":"Gizmo B","Product 6 Performance":64.0,"Product 7 Name":null,"Product 7 Performance":null,"Product 8 Name":"Gizmo D","Product 8 Performance":66.0}
--------------------------------------------------
Score: 12.660498
CompanyName: Dynamics Arcadia
profile: NSW Finance Banking
performance: 67.0
json: {"CompanyName":"Dynamics Arcadia","State":"NSW","Industry":"Finance","Segment":"Banking","Product 1 Name":"Widget W","Product 1 Performance":49.0,"Product 2 Name":null,"Product 2 Performance":null,"Product 3 Name":null,"Product 3 Performance":null,"Product 4 Name":"Widget Z","Product 4 Performance":90.0,"Product 5 Name":null,"Product 5 Performance":null,"Product 6 Name":null,"Product 6 Performance":null,"Product 7 Name":"Gizmo C","Product 7 Performance":62.0,"Product 8 Name":null,"Product 8 Performance":null}
--------------------------------------------------
Score: 12.314001
CompanyName: Nexus Hyper
profile: NSW Finance Banking
performance: 68.8333333333
json: {"CompanyName":"Nexus Hyper","State":"NSW","Industry":"Finance","Segment":"Banking","Product 1 Name":"Widget W","Product 1 Performance":49.0,"Product 2 Name":"Widget X","Product 2 Performance":85.0,"Product 3 Name":"Widget Y","Product 3 Performance":92.0,"Product 4 Name":"Widget Z","Product 4 Performance":15.0,"Product 5 Name":null,"Product 5 Performance":null,"Product 6 Name":"Gizmo B","Product 6 Performance":93.0,"Product 7 Name":"Gizmo C","Product 7 Performance":79.0,"Product 8 Name":null,"Product 8 Performance":null}
```

And as you can see, all of the top 5 results also have a profile of *NSW Finance Banking* and have a performance of over 60. And if we look at the bottom 3 matches (matched 18-20), they have veered off the profile a little bit, but you can see are still related to the target profile. This is a good sign that the search index is working as expected.

```bash
Score: 9.345702
CompanyName: Octogon Craft
profile: SA Finance Banking
performance: 79.4
json: {"CompanyName":"Octogon Craft","State":"SA","Industry":"Finance","Segment":"Banking","Product 1 Name":null,"Product 1 Performance":null,"Product 2 Name":null,"Product 2 Performance":null,"Product 3 Name":null,"Product 3 Performance":null,"Product 4 Name":"Widget Z","Product 4 Performance":87.0,"Product 5 Name":"Gizmo A","Product 5 Performance":68.0,"Product 6 Name":"Gizmo B","Product 6 Performance":94.0,"Product 7 Name":"Gizmo C","Product 7 Performance":85.0,"Product 8 Name":"Gizmo D","Product 8 Performance":63.0}
--------------------------------------------------
Score: 9.345702
CompanyName: Sea Communications
profile: VIC Finance Banking
performance: 69.0
json: {"CompanyName":"Sea Communications","State":"VIC","Industry":"Finance","Segment":"Banking","Product 1 Name":null,"Product 1 Performance":null,"Product 2 Name":"Widget X","Product 2 Performance":93.0,"Product 3 Name":"Widget Y","Product 3 Performance":78.0,"Product 4 Name":null,"Product 4 Performance":null,"Product 5 Name":"Gizmo A","Product 5 Performance":92.0,"Product 6 Name":null,"Product 6 Performance":null,"Product 7 Name":"Gizmo C","Product 7 Performance":13.0,"Product 8 Name":null,"Product 8 Performance":null}
--------------------------------------------------
Score: 9.345702
CompanyName: Square Dynamics
profile: VIC Finance Banking
performance: 63.25
json: {"CompanyName":"Square Dynamics","State":"VIC","Industry":"Finance","Segment":"Banking","Product 1 Name":null,"Product 1 Performance":null,"Product 2 Name":"Widget X","Product 2 Performance":33.0,"Product 3 Name":"Widget Y","Product 3 Performance":83.0,"Product 4 Name":null,"Product 4 Performance":null,"Product 5 Name":"Gizmo A","Product 5 Performance":43.0,"Product 6 Name":null,"Product 6 Performance":null,"Product 7 Name":null,"Product 7 Performance":null,"Product 8 Name":"Gizmo D","Product 8 Performance":94.0}
--------------------------------------------------
```

Now lets look at the vector search results :smile: but remember to uncomment the *vector_queries* parameter in the search client and comment out the *search_text* parameter. It should look like this;

```python
results = search_client.search(  
    #search_text=query,  
    vector_queries= [vector_query],
    select=["CompanyName", "prof", "performance", "json"],
    filter="performance ge 60",
    top=20,
    include_total_count=True
)  
```

And the results should look like this;

```bash
Query: NSW Finance Banking
Total Documents Matching Query: 50
--------------------------------------------------
Score: 0.9999994
CompanyName: Sky Robotics
profile: NSW Finance Banking
performance: 61.25
json: {"CompanyName":"Sky Robotics","State":"NSW","Industry":"Finance","Segment":"Banking","Product 1 Name":"Widget W","Product 1 Performance":75.0,"Product 2 Name":"Widget X","Product 2 Performance":87.0,"Product 3 Name":null,"Product 3 Performance":null,"Product 4 Name":"Widget Z","Product 4 Performance":21.0,"Product 5 Name":null,"Product 5 Performance":null,"Product 6 Name":null,"Product 6 Performance":null,"Product 7 Name":null,"Product 7 Performance":null,"Product 8 Name":"Gizmo D","Product 8 Performance":62.0}
--------------------------------------------------
Score: 0.9999994
CompanyName: Consulting Apex
profile: NSW Finance Banking
performance: 61.0
json: {"CompanyName":"Consulting Apex","State":"NSW","Industry":"Finance","Segment":"Banking","Product 1 Name":null,"Product 1 Performance":null,"Product 2 Name":null,"Product 2 Performance":null,"Product 3 Name":"Widget Y","Product 3 Performance":58.0,"Product 4 Name":"Widget Z","Product 4 Performance":81.0,"Product 5 Name":null,"Product 5 Performance":null,"Product 6 Name":null,"Product 6 Performance":null,"Product 7 Name":"Gizmo C","Product 7 Performance":56.0,"Product 8 Name":"Gizmo D","Product 8 Performance":49.0}
--------------------------------------------------
Score: 0.9999994
CompanyName: Dynamics Arcadia
profile: NSW Finance Banking
performance: 67.0
json: {"CompanyName":"Dynamics Arcadia","State":"NSW","Industry":"Finance","Segment":"Banking","Product 1 Name":"Widget W","Product 1 Performance":49.0,"Product 2 Name":null,"Product 2 Performance":null,"Product 3 Name":null,"Product 3 Performance":null,"Product 4 Name":"Widget Z","Product 4 Performance":90.0,"Product 5 Name":null,"Product 5 Performance":null,"Product 6 Name":null,"Product 6 Performance":null,"Product 7 Name":"Gizmo C","Product 7 Performance":62.0,"Product 8 Name":null,"Product 8 Performance":null}
--------------------------------------------------
Score: 0.9999994
CompanyName: Nexus Hyper
profile: NSW Finance Banking
performance: 68.8333333333
json: {"CompanyName":"Nexus Hyper","State":"NSW","Industry":"Finance","Segment":"Banking","Product 1 Name":"Widget W","Product 1 Performance":49.0,"Product 2 Name":"Widget X","Product 2 Performance":85.0,"Product 3 Name":"Widget Y","Product 3 Performance":92.0,"Product 4 Name":"Widget Z","Product 4 Performance":15.0,"Product 5 Name":null,"Product 5 Performance":null,"Product 6 Name":"Gizmo B","Product 6 Performance":93.0,"Product 7 Name":"Gizmo C","Product 7 Performance":79.0,"Product 8 Name":null,"Product 8 Performance":null}
--------------------------------------------------
Score: 0.9999994
CompanyName: Orion Consulting
profile: NSW Finance Banking
performance: 65.0
json: {"CompanyName":"Orion Consulting","State":"NSW","Industry":"Finance","Segment":"Banking","Product 1 Name":null,"Product 1 Performance":null,"Product 2 Name":null,"Product 2 Performance":null,"Product 3 Name":null,"Product 3 Performance":null,"Product 4 Name":null,"Product 4 Performance":null,"Product 5 Name":null,"Product 5 Performance":null,"Product 6 Name":"Gizmo B","Product 6 Performance":64.0,"Product 7 Name":null,"Product 7 Performance":null,"Product 8 Name":"Gizmo D","Product 8 Performance":66.0}
```

And as you can see, the results are the same as the full text search results, just in a slight different order, as all of the vectors match with a score of 0.9999994. This is a good sign that the vector search is working as expected.

Now lets look at the bottom 3 matches (matched 18-20) and see how they compare to the full text search results.

```bash
Score: 0.8401246
CompanyName: Innovations Strategies
profile: NSW Finance Software
performance: 60.1666666667
json: {"CompanyName":"Innovations Strategies","State":"NSW","Industry":"Finance","Segment":"Software","Product 1 Name":null,"Product 1 Performance":null,"Product 2 Name":"Widget X","Product 2 Performance":85.0,"Product 3 Name":"Widget Y","Product 3 Performance":31.0,"Product 4 Name":"Widget Z","Product 4 Performance":27.0,"Product 5 Name":"Gizmo A","Product 5 Performance":76.0,"Product 6 Name":null,"Product 6 Performance":null,"Product 7 Name":"Gizmo C","Product 7 Performance":70.0,"Product 8 Name":"Gizmo D","Product 8 Performance":72.0}
--------------------------------------------------
Score: 0.8401246
CompanyName: Moon Force
profile: NSW Finance Software
performance: 61.0
json: {"CompanyName":"Moon Force","State":"NSW","Industry":"Finance","Segment":"Software","Product 1 Name":null,"Product 1 Performance":null,"Product 2 Name":null,"Product 2 Performance":null,"Product 3 Name":null,"Product 3 Performance":null,"Product 4 Name":null,"Product 4 Performance":null,"Product 5 Name":"Gizmo A","Product 5 Performance":75.0,"Product 6 Name":null,"Product 6 Performance":null,"Product 7 Name":"Gizmo C","Product 7 Performance":33.0,"Product 8 Name":"Gizmo D","Product 8 Performance":75.0}
--------------------------------------------------
Score: 0.8401246
CompanyName: Aether Consulting
profile: NSW Finance Software
performance: 61.5
json: {"CompanyName":"Aether Consulting","State":"NSW","Industry":"Finance","Segment":"Software","Product 1 Name":"Widget W","Product 1 Performance":92.0,"Product 2 Name":null,"Product 2 Performance":null,"Product 3 Name":null,"Product 3 Performance":null,"Product 4 Name":"Widget Z","Product 4 Performance":40.0,"Product 5 Name":null,"Product 5 Performance":null,"Product 6 Name":null,"Product 6 Performance":null,"Product 7 Name":"Gizmo C","Product 7 Performance":70.0,"Product 8 Name":"Gizmo D","Product 8 Performance":44.0}
--------------------------------------------------
```

And as you can see, the bottom 3 matches are still related to the target profile, but are different to the full text search results.

Lastly, lets look at hybrid search results. To do this, you will need to uncomment the *vector_queries* parameter in the search client and the *search_text* parameter. And the results should look like this;

```bash
Query: NSW Finance Banking
Total Documents Matching Query: 404
--------------------------------------------------
Score: 0.03333333507180214
CompanyName: Sky Robotics
profile: NSW Finance Banking
performance: 61.25
json: {"CompanyName":"Sky Robotics","State":"NSW","Industry":"Finance","Segment":"Banking","Product 1 Name":"Widget W","Product 1 Performance":75.0,"Product 2 Name":"Widget X","Product 2 Performance":87.0,"Product 3 Name":null,"Product 3 Performance":null,"Product 4 Name":"Widget Z","Product 4 Performance":21.0,"Product 5 Name":null,"Product 5 Performance":null,"Product 6 Name":null,"Product 6 Performance":null,"Product 7 Name":null,"Product 7 Performance":null,"Product 8 Name":"Gizmo D","Product 8 Performance":62.0}
--------------------------------------------------
Score: 0.032786883413791656
CompanyName: Consulting Apex
profile: NSW Finance Banking
performance: 61.0
json: {"CompanyName":"Consulting Apex","State":"NSW","Industry":"Finance","Segment":"Banking","Product 1 Name":null,"Product 1 Performance":null,"Product 2 Name":null,"Product 2 Performance":null,"Product 3 Name":"Widget Y","Product 3 Performance":58.0,"Product 4 Name":"Widget Z","Product 4 Performance":81.0,"Product 5 Name":null,"Product 5 Performance":null,"Product 6 Name":null,"Product 6 Performance":null,"Product 7 Name":"Gizmo C","Product 7 Performance":56.0,"Product 8 Name":"Gizmo D","Product 8 Performance":49.0}
--------------------------------------------------
Score: 0.0320020467042923
CompanyName: Dynamics Arcadia
profile: NSW Finance Banking
performance: 67.0
json: {"CompanyName":"Dynamics Arcadia","State":"NSW","Industry":"Finance","Segment":"Banking","Product 1 Name":"Widget W","Product 1 Performance":49.0,"Product 2 Name":null,"Product 2 Performance":null,"Product 3 Name":null,"Product 3 Performance":null,"Product 4 Name":"Widget Z","Product 4 Performance":90.0,"Product 5 Name":null,"Product 5 Performance":null,"Product 6 Name":null,"Product 6 Performance":null,"Product 7 Name":"Gizmo C","Product 7 Performance":62.0,"Product 8 Name":null,"Product 8 Performance":null}
--------------------------------------------------
Score: 0.0317540317773819
CompanyName: Orion Consulting
profile: NSW Finance Banking
performance: 65.0
json: {"CompanyName":"Orion Consulting","State":"NSW","Industry":"Finance","Segment":"Banking","Product 1 Name":null,"Product 1 Performance":null,"Product 2 Name":null,"Product 2 Performance":null,"Product 3 Name":null,"Product 3 Performance":null,"Product 4 Name":null,"Product 4 Performance":null,"Product 5 Name":null,"Product 5 Performance":null,"Product 6 Name":"Gizmo B","Product 6 Performance":64.0,"Product 7 Name":null,"Product 7 Performance":null,"Product 8 Name":"Gizmo D","Product 8 Performance":66.0}
--------------------------------------------------
Score: 0.0314980149269104
CompanyName: Nexus Hyper
profile: NSW Finance Banking
performance: 68.8333333333
json: {"CompanyName":"Nexus Hyper","State":"NSW","Industry":"Finance","Segment":"Banking","Product 1 Name":"Widget W","Product 1 Performance":49.0,"Product 2 Name":"Widget X","Product 2 Performance":85.0,"Product 3 Name":"Widget Y","Product 3 Performance":92.0,"Product 4 Name":"Widget Z","Product 4 Performance":15.0,"Product 5 Name":null,"Product 5 Performance":null,"Product 6 Name":"Gizmo B","Product 6 Performance":93.0,"Product 7 Name":"Gizmo C","Product 7 Performance":79.0,"Product 8 Name":null,"Product 8 Performance":null}
```

As you can see all of the usual suspects are there, but the scores are different. This is because the hybrid search is using both the full text search and the vector search to return the results. This is a good sign that the hybrid search is working as expected.

> As you can see for our use case, it doesn't really matter which search type we use, as they all return very similar results. But for your use case, you might find that one search type is better than the other.

## Make recommendations using Azure Open AI

Now we are finally at the last part of the notebook. This part of the code will use Azure Open AI to make recommendations based on the high performing products of the customer. In the code above, as part of the print the results to output, we filled up an array called *comp_profiles* with the JSON strings of the top 20 similar customer profiles. We will use this array to make recommendations.

```python
### Create AOAI Client ###
chat_client = AzureOpenAI(
    api_key = azure_openai_key,  
    api_version=azure_openai_api_version,
    azure_endpoint = azure_openai_endpoint
    )

# Prepare the message content with the company profiles
message_content = "You are an assistant that recommends products to companies.\nYou will receive a company profile and you need to recommend the next product that the company should buy.\nBelow are some examples of similar companies and their products. Use the below to recommend the next products that the company should buy.\n\n"
for i, profile in enumerate(comp_profiles, start=1):
    message_content += f"### Company {i}\n{profile}\n\n"

message_content += "Recommend the next product that the company should buy using data exclusively from the above text. Take a step-by-step approach in your response, cite product performance examples from the provided data and give reasoning before sharing final answer."

# Send a chat call to generate an answer
response = chat_client.chat.completions.create(
    model=azure_openai_chat_deployment,
    messages=[
        {"role": "system", "content": message_content},
        {"role": "user", "content": target_company}
    ]
)

print(response.choices[0].message.content)
```

The things that has the most impact on the recommendations are the *message_content* and the *target_company*. The *message_content* is the system message where we are telling the Azure Open AI model what we want it to do. I have tried to use some good prompt engineering techniques including System message, Start with clear instructions, Repeat instructions at the end, Chain of thought prompting and Provide grounding context to get the best results. All of these techniques and more are explained in more detail [here](https://learn.microsoft.com/en-us/azure/ai-services/openai/concepts/advanced-prompt-engineering?pivots=programming-language-chat-completions).

The way that this piece of code connects to the previous pieces is that I am adding in those top 20 search results from our search index to the *message_content*. So I have given the Azure Open AI model some context to work with.

And this is the response I got from the Azure Open AI model;

```bash
To recommend the next product Madeup Inc should buy, let's analyze the data and consider the following factors:

1. **Current Products and Performance**:
    - Madeup Inc already has "Widget W" with a performance of 85.
    - They also have "Widget Y" with a performance of 74.

2. **Identify High-Performing Products in Similar Companies**:
    - For companies in the same industry, segment, and state (NSW, Finance, Banking), we can look at the highly performing products:
        - **"Widget Z"** has high performance across various companies:
            - Example: Sky Robotics: "Widget Z" (21); Dynamics Arcadia: "Widget Z" (90); Nexus Hyper: "Widget Z" (15); Consulting Apex: "Widget Z" (81); - with higher performance of 81 to 90
        - **"Widget X"** also shows high performance in companies where it's being effectively utilized:
            - Example: Helix Industries: "Widget X" (68); Communications Ignite: "Widget X" (76); Sea Sphere: "Widget X" (67)

3. **Look for Consistently High Performance or Complements**:
    - Based on the products already in use and their performance, adding a product with consistent high performance would benefit the overall system.
    - Considering the consistent performance of "Widget Z" around 90 in both Nexus Hyper and Diamond Titanium suggests that this could be a viable next option.

### Reasoning for Recommendation:
1. **Widget Z** has shown high performance in similar companies:
    - For example, Dynamics Arcadia has "Widget Z" performing at 90, and Consulting Apex has it performing at 81. Given that "Widget Z" performs well in the same segment and industry, it is likely to complement the existing products of Madeup Inc.
2. **Widget X** also is a viable option based on performance ranges between 67 to 76 in companies with similar profiles and segments.

### Final Recommendation:
**Next Product: "Widget Z"**
- **Reason:**
    - High performance in multiple similar companies within the same segment and industry.
    - Complements the existing high-performance products already in use by Madeup Inc.

By choosing "Widget Z," Madeup Inc is likely to enhance its system efficiency and overall performance.
```

And as you can see, the Azure Open AI model has done a good job at making recommendations based on the high performing products of the customer. It has also given a good reasoning for the recommendation and has cited product performance examples from the provided data. It looks like it was a close race between *Widget Z* and *Widget X*, but *Widget Z* won out in the end.

I do like the way that that Azure Open AI 4o has laid out it's reasoning and final recommendation. It's a good way to present the information to the user.

Also one of the interesting things with generative ai is that you can run the same code multiple times and get different results, like below;

```bash
To recommend the next product that "Madeup Inc" should buy, we'll consider the provided data and performance metrics for similar companies in the same segment (Banking) and geographical region (NSW). Here are the steps we will take:

### Step 1: Identify Relevant Companies

We'll focus on companies in the banking segment within NSW that have similar product usage. Relevant companies include:

1. **Sky Robotics**
2. **Consulting Apex**
3. **Dynamics Arcadia**
4. **Nexus Hyper**
5. **Diamond Titanium**

### Step 2: Analyze Product Performance

We'll evaluate the performance of products used by these companies to identify the next suitable product for Madeup Inc.

1. **Widget X:**
   - Sky Robotics: 87.0
   - Nexus Hyper: 85.0
   - Hexagon Ignite: 94.0
   - Consulting Apex: 58.0
   - Diamond Titanium: (No Data)
   - **Average Performance: (87+85+94+58)/4 = 81.0**

2. **Widget Z:**
   - Sky Robotics: 21.0
   - Dynamics Arcadia: 90.0
   - Nexus Hyper: 15.0
   - Consulting Apex: 81.0
   - Diamond Titanium: 87.0
   - **Average Performance: (21+90+15+81+87)/5 = 58.8**

3. **Gizmo D:**
   - Sky Robotics: 62.0
   - Consulting Apex: 49.0
   - Dynamics Arcadia: (No Data)
   - Nexus Hyper: (No Data)
   - Diamond Titanium: (No Data)
   - **Average Performance: (62+49)/2 = 55.5**

4. **Gizmo C:**
   - Dynamics Arcadia: 62.0
   - Nexus Hyper: 79.0
   - Consulting Apex: 56.0
   - Diamond Titanium: (No Data)
   - **Average Performance: (62+79+56)/3 = 65.7**

### Step 3: Determine the Best Next Product

Since Madeup Inc is in the banking segment within NSW and has used Widget W and Widget Y with reasonable performance:

- Widget W: 85.0
- Widget Y: 74.0

The statistics show that **Widget X** is the next best-performing product with a consistent high average performance of around **81.0**.

### Recommendation

**Widget X** is the most suitable next product for Madeup Inc, considering its high average performance across similar banking companies in NSW, and its successful track record in similar companies.

**Final Answer: Widget X**
```

I really like the second response more then the first. So my next job will be to try and get a repeatable response I am happy with from Azure Open AI model.

So I have changed the second half of my prompt to be *"Recommend the next product that the company should buy using data exclusively from the above text. You process should involve a step-by-step approach, which the following steps: 1. Analyze the company profile, 2. Identify Relevant Companies, 3. Calculate Average Performance, 4. Recommend two products a primary and secondary, 5. Justify primary and secondary Recommendations."*

The major changes I have made is to swap out the chain of thought technique for a break the task down technique. I have also added in a secondary recommendation, as I think that is a good idea. Lastly I added in a step to calculate the average performance of the products, to give more justification to the recommendations.

And this is the response I got from the Azure Open AI model;

```bash
To recommend the next product for Madeup Inc, I'll follow the outlined steps.

### Step 1: Analyze the company profile
**Company Name:** Madeup Inc  
**Location:** NSW  
**Industry:** Finance  
**Segment:** Banking  
**Current Products and Performance:**
- **Widget W:** 85.0
- **Widget Y:** 74.0

### Step 2: Identify Relevant Companies
Since Madeup Inc is in the Finance industry, Banking segment, located in NSW, I'll look for other NSW-based Banking companies within the dataset.

Relevant companies:
1. Sky Robotics
2. Consulting Apex
3. Nexus Hyper
4. Diamond Titanium
5. Dynamics Arcadia

### Step 3: Calculate Average Performance
Among the relevant companies, assess the performance of the products they use:

#### Widget X
- Sky Robotics: 87.0
- Nexus Hyper: 85.0
- Average: (87.0 + 85.0) / 2 = 86.0

#### Widget Z
- Sky Robotics: 21.0
- Consulting Apex: 81.0
- Nexus Hyper: 15.0
- Dynamics Arcadia: 90.0
- Diamond Titanium: 87.0
- Average: (21.0 + 81.0 + 15.0 + 90.0 + 87.0) / 5 = 58.8

#### Gizmo B
- Nexus Hyper: 93.0
- Diamond Titanium: 67.0
- Average: (93.0 + 67.0) / 2 = 80.0

#### Gizmo C
- Nexus Hyper: 79.0
- Dynamics Arcadia: 62.0
- Consulting Apex: 56.0
- Average: (79.0 + 62.0 + 56.0) / 3 = 65.7

#### Gizmo D
- Sky Robotics: 62.0
- Consulting Apex: 49.0
- Dynamics Arcadia: 0.0 (absent)
- Nexus Hyper: 0.0 (absent)
- Diamond Titanium: 0.0 (absent)
- Average: (62.0 + 49.0) / 2 = 55.5

### Step 4: Recommend Primary and Secondary Products
**Primary Recommendation:** Widget X (Average Performance: 86.0)
**Secondary Recommendation:** Gizmo B (Average Performance: 80.0)

### Step 5: Justify Recommendations
**Primary Recommendation: Widget X**
- **Reasoning:** Widget X shows a consistently high performance average (86.0) in relevant companies such as Sky Robotics and Nexus Hyper. The high average and strong performance suggest it is a reliable product for achieving effective outcomes in the Banking segment within the Finance industry.

**Secondary Recommendation: Gizmo B**
- **Reasoning:** Gizmo B also shows good performance with an average of 80.0, as seen in companies like Nexus Hyper and Diamond Titanium. This makes it a strong contender for providing additional benefits and complementing the existing high performance of Widget W and Widget Y. 

Hence, based on the performance data and relevance within the industry and geographical segment, Widget X and Gizmo B are the recommended next products for Madeup Inc.
```

I also ran the prompt again 5 more times (which we can do now we have setup our Azure environment) and got consistent results with Wiget X and Gizmo B always being recommended. So I am happy with the results.

## Conclusion

In this part two post, we have expanded our dataset and created a search index on Azure AI Search. We have also queried the search index using full text search, vector search and hybrid search and We have also made recommendations using Azure Open AI.

I hope you have found this post useful and that it has inspired you to build your own Gen AI Product Recommendation System. If you have any questions or comments, please feel free to leave them below. I would love to hear from you.
