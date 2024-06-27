---
title: "Simple Gen AI Product Recommendation System you can POC quickly"
description: "This simple Gen AI Product Recommendation System is designed for rapid proof-of-concept (POC) development. It leverages generative AI techniques to provide product recommendations to your customers."
date: 2024-06-20T14:28:16+10:00
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

I have been working with some clients recently on proof-of-concepts (POC) to recommend products to their customers. I have found that generative AI techniques can be very effective and in this post, I will show you how to build a simple Gen AI Product Recommendation System that you can use to quickly develop a proof-of-concept (POC) for your clients.

The idea behind this post is to just use Python, a CSV file (containing customer and product performance) and Azure Open AI to build a simple Gen AI Product Recommendation System :smiley:

Again this process is just for quickly POCing the concept and will be part one of a series of posts I will be writing on this topic. Which eventually will include more advanced techniques and technologies and lead to a production-ready system using techniques like Retrieval Augmented Generation (RAG) and more :sweat_smile:

## TLDR

If you just want to see the code, you can find it [here](https://github.com/fredderf204/poc-recsys)

All you need is a CSV file with customer and product performance data and an Azure Open AI account. Then you can use the python code above to POC the concept. A sample CSV file has been included in the GitHub repo.

So as long as you can get customer and product performance out of your system, you can POC a Gen AI Product Recommendation System :+1:

## Prerequisites

There are only two prerequisites to follow along with this post;

1. An Azure subscription and access to Azure Open AI. For the Azure Open AI component I use the *text-embedding-ada-002* model to vectorise the CSV file (more details on this later) and the question. I also use the *GPT-4o* model to recommend the next best product to the customer. You can sign up for Azure Open AI [here](https://azure.microsoft.com/en-us/services/cognitive-services/openai/)

> :book: Note: You do not have to use Azure Open AI to make this work. You can sub in other AI models to perform the same functions as I have done in this post.

2. A CSV file with customer and product performance data. I have included a sample CSV file in the GitHub repo for you to use and will go into more detail about the CSV file below.

## Introduction

The python code found [here](https://github.com/fredderf204/poc-recsys) is a simple Gen AI Product Recommendation System that uses generative AI techniques to recommend products to customers. The system is designed for rapid proof-of-concept (POC) development and can be used to quickly develop a POC for your clients. The code flows as follows;

1. Prepare the data
2. Vectorise the CSV file using Azure Open AI
3. Vectorise your target customers profile and return similar customers
4. Refine the list of similar customers with customers that have good product performance
5. Recommend the next best product to the customer using Azure Open AI

## Prepare the data

The first step is to prepare the data. I used a combination of ChatGPT and RAND functions in excel to create sample data. You should use your own data for this step. The data should be in a CSV file with similar columns to the below;

```Company Name, State, Industry, Segment, Product 1 Name, Product 1 Performance, Product 2 Name, Product 2 Performance, Product 3 Name, Product 3 Performance, Product 4 Name,Product 4 Performance, Product 5 Name, Product 5 Performance, Product 6 Name, Product 6 Performance, Product 7 Name, Product 7 Performance, Product 8 Name, Product 8 Performance```

In my example, I am recommending products to companies. You can see with headers like Company Name, State, Industry, Segment I am tring to capture the profiles of companies. You can change these headers to suit your needs. I.e. if you are recommending products to individuals you may remove the Industry column an replace it with something else.

I chose to have 8 products in my example to make it more interesting. You might have more or less products, which is ok and the concept will still work. 

The performance column is a number between 0 and 100. 0 being the worst performing product and 100 being the best performing product. You can change this to suit your needs. 

> :book: Note: Your performance column could be a dollar value or a percentage or range from -100 to 10,000. It's completely up to you.

## Vectorise the CSV file using Azure Open AI

Now we have our CSV file, we need to vectorise it. This is where Azure Open AI comes in. I use the *text-embedding-ada-002* model to vectorise the CSV file. This model is a transformer-based model that can convert text into vectors.

I have being using the word vectorise pretty liberally in this post. What I really mean is to create embeddings of the text. If you are interested in digging into this deeper, please follow this link [here](https://learn.microsoft.com/en-us/azure/cosmos-db/vector-database#embeddings).

> :book: Note: I will not be covering chunking strategies in this post. But you can read more about them [here](https://learn.microsoft.com/en-us/azure/architecture/ai-ml/guide/rag/rag-chunking-phase). **In my experience when working with CSV files, you want a single customers data on one line.** As this will make the chunking strategy easier, as it becomes one embedding per line.

The *generate_embeddings* function has been taken from this sample [here](https://learn.microsoft.com/en-us/azure/ai-services/openai/tutorials/embeddings?tabs=python-new%2Ccommand-line&pivots=programming-language-python), which I found super useful to kick start this process.

If you are following along with th example, your DataFrame should now look like the below; if you uncomment the print statement at line 48 and run the code.

```bash

               Company Name State       Industry  ...                                               comb n_tokens                                             ada_v2
0            Aether Alytics   NSW     Technology  ...  Aether Alytics NSW Technology Software Widget ...       35  [0.006774003151804209, 0.014850836247205734, -...
1             Aether Dymics   QLD     Healthcare  ...  Aether Dymics QLD Healthcare Services     Widg...       31  [0.0005342101794667542, 0.023639481514692307, ...
2           Aether Innovate   VIC          Fince  ...  Aether Innovate VIC Fince Banking Widget W 93....       29  [-0.00746396416798234, -0.001040163915604353, ...
3              Aether Nexus   NSW  Manufacturing  ...  Aether Nexus NSW Manufacturing Electronics   W...       40  [0.00921355839818716, 0.013308473862707615, -0...
4   AetherCraft Innovations   QLD     Technology  ...  AetherCraft Innovations QLD Technology Service...       42  [0.004110227804630995, 0.012736016884446144, -...
..                      ...   ...            ...  ...                                                ...      ...                                                ...
95          Vertex Ventures   VIC         Retail  ...  Vertex Ventures VIC Retail Sporting Goods   Wi...       35  [0.0020639034919440746, -0.0007355211419053376...
96            Zenith Dymics   QLD     Healthcare  ...  Zenith Dymics QLD Healthcare Aged Care       W...       39  [0.011205820366740227, 0.01472207996994257, -0...
97       Zenith Innovations   VIC  Manufacturing  ...  Zenith Innovations VIC Manufacturing Electroni...       53  [0.00700162211433053, 0.006710764020681381, -0...
98          Zenith Ventures   NSW         Retail  ...  Zenith Ventures NSW Retail Supermarket   Widge...       35  [0.01493804156780243, -0.005660619121044874, -...
99            Zenith Zephyr   NSW          Fince  ...  Zenith Zephyr NSW Fince Superannuation   Widge...       52  [0.003903178498148918, 0.010286277160048485, -...
```

As you can see above, we have a a couple of new columns; *comb, n-tokens and ada_v2*.

The *comb* column is a combination of the all of the columns in the CSV, which is defined in the cols variable in line 22 of the Python code. This is the data we will use to find similar companies to our target company.

> :book: Note: I have included all of the column data in *comb* column, which you may not what to do. This is the data that will been turned into our embeddings and used to find similar company profiles. Feel free to experiment and play around which which columns make sense to define as company profile data.

The *n-tokens* column contains the number of tokens in the *comb* column. This is part of the sample code [here](https://learn.microsoft.com/en-us/azure/ai-services/openai/tutorials/embeddings?tabs=python-new%2Ccommand-line&pivots=programming-language-python) which I thought would be good to include, so we can see hoe many tokens are in each company profile.

The ada-v2 column contains the embeddings of the text in the *comb* column. This is what we will use to find similar companies to our target company.

## Vectorise your target customers profile and return similar customers

Now we have our CSV file vectorised, we can vectorise our target companies profile and return similar companies. The *search_docs* function does this. It takes the target companies profile and the vectorised CSV file and returns the most similar companies to the target company.

Again the **search_docs* function has been taken from this sample [here](https://learn.microsoft.com/en-us/azure/ai-services/openai/tutorials/embeddings?tabs=python-new%2Ccommand-line&pivots=programming-language-python), which I found super useful to kick start this process.

One thing I did change was the number of similar companies returned. I found that 4 wasn't enough to demonstrate the concept. So I increased it to 20. You can change this to suit your needs.

Our target company profile is ```Madeup Inc NSW Finance Banking Widget W 85 Widget Y 74```. So to expand on this, the name of the company is *Madeup Inc*, they are located in *NSW*, their industry is *Finance*, their segment is *Banking* and they have purchased *Widget W* and *Widget Y* with performance scores of *85* and *74* respectively.

If you are following along with the example and run the code, we have created a DataFrame called res and it should look like the below; if you uncomment the print statement at line 79 and run the code.

```bash
[20 rows x 25 columns]
              Company Name State       Industry         Segment  ...                                               comb n_tokens                                             ada_v2 similarities
94      Vertex Innovations   NSW          Fince         Banking  ...  Vertex Innovations NSW Fince Banking   Widget ...       34  [-0.013468814082443714, 0.011046885512769222, ...     0.895079
40         Nebula Innovate   QLD          Fince         Banking  ...  Nebula Innovate QLD Fince Banking Widget W 55....       43  [-0.009700409136712551, 0.00857381522655487, -...     0.880019
36                 Lu Labs    WA          Fince         Banking  ...  Lu Labs WA Fince Banking Widget W 77.0   Widge...       41  [0.001298676710575819, 0.014152105897665024, 0...     0.877526
16    Celestial Innovators   NSW     Healthcare       Aged Care  ...  Celestial Innovators NSW Healthcare Aged Care ...       36  [0.02342524565756321, 0.01131583470851183, -0....     0.873675
99           Zenith Zephyr   NSW          Fince  Superannuation  ...  Zenith Zephyr NSW Fince Superannuation   Widge...       52  [0.003903178498148918, 0.010286277160048485, -...     0.872854
93  TitanCraft Innovations   NSW          Fince  Superannuation  ...  TitanCraft Innovations NSW Fince Superannuatio...       46  [0.0011813724413514137, 0.0001086916818167083,...     0.872731
50        Nova Innovations   NSW          Fince        Services  ...  Nova Innovations NSW Fince Services   Widget X...       30  [0.008242596872150898, 0.013941271230578423, -...     0.872451
77       Stellar Solutions   NSW     Technology        Services  ...  Stellar Solutions NSW Technology Services Widg...       41  [0.009536132216453552, 0.01122966781258583, -0...     0.872061
9       Arcane Innovations   NSW     Technology        Services  ...  Arcane Innovations NSW Technology Services Wid...       34  [0.009464128874242306, 0.00965742114931345, -0...     0.869977
21      Electra Innovators   NSW     Technology        Software  ...  Electra Innovators NSW Technology Software    ...       28  [-0.005648091435432434, -0.001541689271107316,...     0.869639
98         Zenith Ventures   NSW         Retail     Supermarket  ...  Zenith Ventures NSW Retail Supermarket   Widge...       35  [0.01493804156780243, -0.005660619121044874, -...     0.869488
86     TerraNova Solutions   QLD          Fince         Banking  ...  TerraNova Solutions QLD Fince Banking Widget W...       49  [0.0007926894468255341, 0.008511657826602459, ...     0.868256
82       Synergy Solutions   VIC         Retail         Fashion  ...  Synergy Solutions VIC Retail Fashion Widget W ...       31  [1.3071230569039471e-05, -0.016084423288702965...     0.868212
66     Quantum Innovations   VIC          Fince         Banking  ...  Quantum Innovations VIC Fince Banking Widget W...       36  [-0.0017555200029164553, 0.0030148178339004517...     0.867936
2          Aether Innovate   VIC          Fince         Banking  ...  Aether Innovate VIC Fince Banking Widget W 93....       29  [-0.006697694770991802, -0.0010367195354774594...     0.866954
58          Orion Ventures   NSW     Technology        Services  ...  Orion Ventures NSW Technology Services   Widge...       35  [0.01005646027624607, -0.0012994403950870037, ...     0.865864
73     Solaris Innovations   QLD          Fince  Superannuation  ...  Solaris Innovations QLD Fince Superannuation  ...       40  [0.015805836766958237, 0.010568874888122082, 0...     0.865290
75         Solaris Synergy   VIC          Fince         Banking  ...  Solaris Synergy VIC Fince Banking   Widget X 8...       35  [0.001066106022335589, -0.00687881326302886, 0...     0.862809
25         Helios Innovate   NSW         Retail         Fashion  ...  Helios Innovate NSW Retail Fashion   Widget X ...       23  [0.008114686235785484, 0.014257645234465599, -...     0.861633
20        Electra Innovate   NSW  Manufacturing        Textiles  ...  Electra Innovate NSW Manufacturing Textiles Wi...       28  [-0.003946746699512005, 0.008698659017682076, ...     0.861492
```

As you can see above, we now have a DataFrame called res which contains 20 similar companies to our target company. The *similarities* column contains the cosine similarity between the target company and the similar companies. The higher the cosine similarity, the more similar the companies are.

## Refine the list of similar customers with customers that have good product performance

Next I want to refine this list to only include companies that have a similarities over 0.87, as I only want to find companies that are very similar to the target customer. So in you uncomment the print statement at line 85 and run the code, you will see the DataFrame res has been refined to only include companies with a similarities over 0.87.

```bash

              Company Name State    Industry         Segment  ...                                               comb n_tokens                                             ada_v2 similarities
94      Vertex Innovations   NSW       Fince         Banking  ...  Vertex Innovations NSW Fince Banking   Widget ...       34  [-0.013468814082443714, 0.011046885512769222, ...     0.895079
40         Nebula Innovate   QLD       Fince         Banking  ...  Nebula Innovate QLD Fince Banking Widget W 55....       43  [-0.009700409136712551, 0.00857381522655487, -...     0.880019
36                 Lu Labs    WA       Fince         Banking  ...  Lu Labs WA Fince Banking Widget W 77.0   Widge...       41  [0.001298676710575819, 0.014152105897665024, 0...     0.877526
16    Celestial Innovators   NSW  Healthcare       Aged Care  ...  Celestial Innovators NSW Healthcare Aged Care ...       36  [0.02342524565756321, 0.01131583470851183, -0....     0.873675
99           Zenith Zephyr   NSW       Fince  Superannuation  ...  Zenith Zephyr NSW Fince Superannuation   Widge...       52  [0.003903178498148918, 0.010286277160048485, -...     0.872854
93  TitanCraft Innovations   NSW       Fince  Superannuation  ...  TitanCraft Innovations NSW Fince Superannuatio...       46  [0.0011813724413514137, 0.0001086916818167083,...     0.872731
50        Nova Innovations   NSW       Fince        Services  ...  Nova Innovations NSW Fince Services   Widget X...       30  [0.008242596872150898, 0.013941271230578423, -...     0.872451
77       Stellar Solutions   NSW  Technology        Services  ...  Stellar Solutions NSW Technology Services Widg...       41  [0.009536132216453552, 0.01122966781258583, -0...     0.872061
```

As you can see above, we are left with 8 company profiles. 6 of these companies are in the same Industry as the target company and 3 are even in same segment. Also 6 are in the same State.

Next we again want to refine this list to only include companies that have good product performance. So I have taken a rudimentary approach to giving each of these 8 companies a score based on their product performance, simply by taking the mean of their product performance. Then I have removed all companies with a score under 70. If you uncomment the print statement at line 103 and run the code, you will see the DataFrame res has been refined to only include companies with a product performance score over 70.

```bash

              Company Name State    Industry         Segment  ... n_tokens                                             ada_v2 similarities  performance
94      Vertex Innovations   NSW       Fince         Banking  ...       34  [-0.013468814082443714, 0.011046885512769222, ...     0.895079         72.5
99           Zenith Zephyr   NSW       Fince  Superannuation  ...       52  [0.003903178498148918, 0.010286277160048485, -...     0.872854         72.0
93  TitanCraft Innovations   NSW       Fince  Superannuation  ...       46  [0.0011813724413514137, 0.0001086916818167083,...     0.872731         76.0
77       Stellar Solutions   NSW  Technology        Services  ...       41  [0.009536132216453552, 0.01122966781258583, -0...     0.872061         73.0
```

As you can see above, we are left with 4 company profiles. 3 of these companies are in the same Industry as the target company and all 4 are in the same State.

> :book: Note: Again, not sure if 4 or 6 or 10 or 20 profiles will yield the best results. But for demonstration purposes, I am happy to stick with 4. You may choose a higher number is you like :smiley:

## Recommend the next best product to the customer using Azure Open AI

Finally, we can recommend the next best product to the company using Azure Open AI. I use the *GPT-4o* model to recommend the next best product to the company. 

The way I do this is I create a system prompt that has the following instructions;

```bash
You are a assistant that recommends products to companies.
You will receive a company profile and you need to recommend the next product that the company should buy.

Below are some examples of similar companies and their products. Use the below to recommend the next products that the company should buy. 

### Example Company 1
$comp_prof_1

### Example Company 2
$comp_prof_2

### Example Company 3 
$comp_prof_3

### Example Company 4\n 
$comp_prof_4

Recommend the next one product only that the company should buy. Then on a separate line write a short concise reason why they should buy the product along with insights contained in the data.
```

I am using some good prompt engineering practises here, but the key thing is to include the company profiles in the prompt. This way the *GPT-4o* model can use the company profiles to recommend the next best product to the Madeup Inc. Also I am asking it to recommend only one product, as I am only interested in the next best product for Madeup Inc. If you would like to see all the products that the similar companies have bought, you can change this prompt to suit your needs.

And lastly, below is the recommendation that the *GPT-4o* model has made for Madeup Inc;

```bash
**Recommended Product: Widget Z**

**Reason:** Widget Z has consistently high performance in the banking segment, with performance scores such as 94 from Vertex Innovations. Adding Widget Z could bolster Madeup Inc’s product lineup and improve their overall offerings.
```

As you can see above, the *GPT-4o* model has recommended Widget Z to Madeup Inc. The reason being that Widget Z has consistently high performance in the banking segment, with performance scores such as 94 from Vertex Innovations. Adding Widget Z could bolster Madeup Inc’s product lineup and improve their overall offerings.

## Conclusion

In this post, I have shown you how to build a simple Gen AI Product Recommendation System that you can use to quickly develop a proof-of-concept (POC) for your clients. The system uses generative AI techniques to recommend products to customers and is designed for rapid POC development. The system is built using Python, a CSV file (containing customer and product performance) and Azure Open AI. The system can be used to quickly develop a POC for your clients and can be easily adapted to suit your needs.

> :book: Note: This is just a simple example to get you started. In future posts, I will be covering more advanced techniques and technologies to build a production-ready Gen AI Product Recommendation System. For example, this process stores all of the vectors and CSV data in memory. This is ok for POCs but certainly not Production or even MVPs. You would need to store this data in a vector database or search technology and use various different techniques to retrieve similar company profiles. I will cover this in a future post.

I hope you have found this post useful and that it has inspired you to build your own Gen AI Product Recommendation System. If you have any questions or comments, please feel free to leave them below. I would love to hear from you.
