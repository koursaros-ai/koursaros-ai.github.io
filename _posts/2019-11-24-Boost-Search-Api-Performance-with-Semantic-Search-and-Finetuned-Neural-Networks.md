---
layout: post
title: Boost Search API Performance (e.g. Elasticsearch) by 80% with Semantic Search and Finetuned Neural Networks
---

[Semantic search](https://towardsdatascience.com/semantic-search-fuck-yeah-e371c0f639d) has generated a lot of hype recently by promising search engines that can understand the meaning behind a search, rather than just looking for keywords. However, the only people using and building these models are **Information Retrieval (IR)** researchers and their code is not easily transferrable to a production environment with real users. The best of the models that these researchers have developed are capable of impressive language understanding, topping the leaderboard of IR competition such as [MS MARCO](http://www.msmarco.org/) and [TREC-CAR](http://trec-car.cs.unh.edu/). These semantic search methods represent a significant improvement over traditional keyword search, as much as doubling standard search performance.

While these results hold promise for improving a user's search experience, any integrated solution that utilizing semantic search needs to be built from the ground up – as anyone who’s build a search engine cant tell you, it is no small feat. We thought there must be a way to apply these models to existing search solutions! (Such as Elasticsearch) That’s why we decided to build [NBoost](https://github.com/koursaros-ai/nboost). From day one, we had an idea of how the platform should behave:

### 1. It should be easy to integrate into existing search APIs.

<p align="center">
<img src="https://github.com/koursaros-ai/nboost/raw/master/.github/overview.svg?sanitize=true" width="100%">
</p>

Our first thought was to use NBoost to increase the performance of existing Elasticsearch deployments. Elasticsearch is a scalable open source text search platform, currently relied on across many industries. In order for NBoost to have production value, we knew that there couldn’t be a large switchover cost. That’s why we decided to build a proxy. Integrating a neural model in [Elasticsearch](https://www.elastic.co/) is as easy as pointing your search requests at a different host. In order to understand how this works, we need to understand how [neural ranking](http://www.bigdatalab.ac.cn/~gjf/papers/2019/Survey_Preprint.pdf) works. Check out the [NBoost Overview](https://github.com/koursaros-ai/nboost#overview) if you're not familiar with this concept.

Running the proxy is *super* simple. All you need to do is give nboost the location of your Elasticsearch server. After you pip install nboost, you just run `nboost --uhost localhost --uport 9200` (uhost and uport are short for upstream-host and upstream-port). This will load up the default model, ` bert-base-uncased-msmarco` which was trained on Bing search results.

> Your will need to pip install nboost[tf] for the default model

### 2. The models should be plugins.

<p align="center">
<img src="https://github.com/koursaros-ai/nboost/raw/master/.github/plugin.svg?sanitize=true" width="100%">
</p>

The ability to use previously finetuned neural models is crucial. Therefore, the ability to modularly swap in new models for new tasks is a must-have. Constantly training models from scratch for domains that have previously existing models is a waste. That’s part of the reason for the [current pretrained->finetuned AI supply chain](http://hanxiao.io/2019/07/29/Generic-Neural-Elastic-Search-From-bert-as-service-and-Go-Way-Beyond/#gnes-preliminaries-breakdown-of-neural-elastic-and-search). For search domains such as medicine, travel, and politics, we are training models that can dramatically increase search engine performance. These models can then be easily hooked into the platform.

To make switching out models as easy as possible, we built a platform that is agnostic to which ML library (tf vs torch vs transformers) and model we’re trying to use. We store previously finetuned models and [host them on Google Buckets](https://console.cloud.google.com/storage/browser/_details/koursaros/albert-tiny-uncased-msmarco.tar.gz?project=blueface&organizationId=775498386689). Then, depending on the model specified by the `nboost --model_dir` switch, it loads the dependencies for that model. If you’re interested in exactly how this is implemented, check out [this article](https://colethienes.github.io/Managing-Machine-Learning-Dependencies-In-Distributed-Systems/).

### 3. It should be rippin' fast.

<p align="left">
<img src="https://github.com/koursaros-ai/nboost/raw/master/.github/rocket.svg?sanitize=true" width="50%">
</p>


In production, time is money. The longer search results take to get to the user, the more likely they are to [click away](https://www.thinkwithgoogle.com/marketing-resources/the-google-gospel-of-speed-urs-hoelzle/). Therefore, NBoost must be able to request, rank, and return search results from the search API without too much added latency. The ML libraries (which are written in Python) are the main hurdle for making it fast.

To get around this, we stuck to low-level infrastructure. The proxy buffers the socket in order to capture only the search requests, and proxy everything else through (such as miscellaneous Elasticsearch requests not having anything to do with search). This is done with the python standard socket library (which is a thin wrapper around the C-library. To parse the http requests efficiently, NBoost uses the same [C-based Http-parsing library as NodeJs](https://github.com/MagicStack/httptools). If you want to read more, read [this article](https://colethienes.github.io/Implementing-a-URL-Selective-Proxy-in-Python/).

The main tradeoff for NBoost is (almost) always speed vs accuracy. Larger models are more effective, but slower. Ranking fewer search results is faster, but yields less relevant results. In order to capture this tradeoff, [we benchmark the search boost vs query speed to compare different finetuned models](https://github.com/koursaros-ai/nboost#benchmarks). 

### 4. It should be elastic.

Currently, the main way to scale is by increasing the number of `nboost --workers`. However, we are currently developing a Helm Chart to load-balance NBoost on Kubernetes.

## Into the future

The most exciting aspect of NBoost is the reusability of finetuned models for specific search domains. Our first finetuned model was trained on [millions of Bing queries](https://github.com/nyu-dl/dl4marco-bert). We found that this model increased Elasticsearch search accuracy by 80%!  Our next model utilizes the BioBERT model to finetune a model for boosting search results in the biomedical space. Additionally, a couple of our top secret 🤫  projects include training smaller models such as [ALBERT](https://medium.com/syncedreview/googles-albert-is-a-leaner-bert-achieves-sota-on-3-nlp-benchmarks-f64466dd583) and [tiny BERT](https://medium.com/syncedreview/huaweis-tinybert-is-7x-smaller-and-9x-faster-than-bert-2fbe76f03974) for faster search results. Stay tuned!