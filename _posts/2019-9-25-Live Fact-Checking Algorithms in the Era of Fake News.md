---
layout: post
title: Live Fact-Checking in the Era of Fake News
excerpt: Fake news, political deception, and rumor mills are not new phenomena, but in recent years the internet has fueled an explosion of deliberate disinformation.
---

Fake news, political deception, and rumor mills are not new phenomena, but in recent years the internet has fueled an explosion of deliberate disinformation. The spread of deceptive content is often rapid and uncontrollable. A recent study by MIT found that falsehood diffuses significantly farther, faster, deeper, and more broadly than the truth, in all categories of information. Today, any individual group can spread untrue statements and manipulate the public without being held accountable.

Simultaneously, recent advances in machine learning models, namely transformer networks such as BERT,  have pushed computer’s ability to understand natural language ever closer to human performance. Our algorithm for live fact checking lies at the nexus of this technology and recent developments in politics. 

Recent concern has stimulated research efforts to automate fact-checking. Machine learning efforts to tackle fact-checking are fueled by invaluable datasets such as the Fact Extraction and VERification (FEVER) dataset, Fake News Challenge (FNC), General Language Understanding Evaluation (GLUE), MultiGenre Natural Language Inference (MultiNLI), and Semantic Evaluation (SemEval). 

In order to tackle this challenge we first sought to prove the effectiveness of our methods by achieving state of the art accuracy on the largest of these datasets,  <a href = 'http://fever.ai'>FEVER</a>. While such datasets are notoriously biased and often fail to generalize to real-world problems, they are a useful tool to at least know your technology is in the ballpark of effective. Once achieving this ballpark effectiveness we planned to see if such a pipelines could be deployed for real-world, low-latency fact checking, potentially by supplementing wikipedia with other ground truth data sources.

## Our System


![alt_text](https://raw.githubusercontent.com/koursaros-ai/koursaros-ai.github.io/master/images/Screen%20Shot%202019-09-25%20at%204.00.58%20PM.png "image_tooltip")


### Training Elasticsearch

Rather than relying on hand built matching and filtering heuristics, such as the “entity linking” described by Athene (the top team by information retrieval (IR) for Fever) we were interested in building a system that could be _trained_ for optimal recall on an IR task such as that proposed by FEVER. Additionally such systems could be trained in production by incorporating user feedback.

One of our primary concerns in designing such a system was performance and scalability. This is in keeping with our overall system design for Koursaros, which relies on pub-sub architecture and a robust broker. 

We divide our search efforts into those powered primarily by machine learning, and those powered primarily by traditional text search. However, this distinction is blurred by our use of Bayesian optimization for tuning hyperparameters for text search. Our future plans include pursuing search exclusively powered by machine learning as we plan to explore a system of directly training a neural search system for document retrieval. However, such a solution will require more research and development. Notably an efficient method for KNN search in high dimensions using approximate nearest neighbors is needed to boost document recall. Such a system should be sharded and also implement multi-layer clustering, so that the entire index doesn’t need to stay in memory. 


### Ranking Potential Evidence Sentences for Evaluation

We used RoBERTA, from Facebook AI, as our pretrained transformer network, since it recently set state of the art on many text understanding tasks. We used it to train regression on pairs of claims and evidence from the search results, labeling the correct claim evidence pair from the FEVER set with a 1.0 and all others with a 0.0. Notably we concatenated the title of the wiki article to the beginning of every line from the article, providing context sometimes lost by pronouns. This system resulted in 94% line recall when the candidate evidence was ranked in descending order by score and the top 5 were taken, **compared with 87% from the Athene** team. We performed this evaluation on the test set in cases where the correct document had been. Our boost in this score can be attributed to using a larger, newer LSTM model, and the concatenation of titles to the candidate evidence prior to scoring.


### Training an Inference Model

We decided to use pre trained RoBERTA again, this time fine-tuning it on the top 1 result from our evidence retrieval and ranking pipeline and the ground truth labels from FEVER train set. This achieved **74%** label accuracy, #2 on the continuous evaluation leaderboard. Due to the discrepancy of this score and our FEVER, score which was discounted for correctly labeled examples which lacked ground truth evidence in the top 5, I believe the inference is not a bottleneck on our system, and we could get an even higher score by boosting article recall.


## Results

<table class = 'pure-table pure-table-horizontal' >
  <thead>
    <tr>
     <th>Model
     </th>
     <th>Label Accuracy
     </th>
     <th>Evidence F1
     </th>
     <th>FEVER Score
     </th>
    </tr>
  </thead>
  <tbody>
  <tr>
    <td><b>Koursaros w/ Roberta</b>
   </td>
    <td><b>73.64</b>
   </td>
   <td>52
   </td>
   <td>63
   </td>
  </tr>
  <tr>
   <td>UNC-NLP
   </td>
   <td>68.21
   </td>
   <td>52.96
   </td>
   <td>64.21
   </td>
  </tr>
  <tr>
   <td>UCL ML
   </td>
   <td>67.62
   </td>
   <td>34.97
   </td>
   <td>62.52
   </td>
  </tr>
  <tr>
   <td>Athene UKP
   </td>
   <td>65.46
   </td>
   <td>36.97
   </td>
   <td>61.58
   </td>
  </tr>
  <tr>
   <td>MIT FAKTA
   </td>
   <td>59.35
   </td>
   <td>
   </td>
   <td>
   </td>
  </tr>
  <tr>
   <td>Columbia-NLP
   </td>
   <td>57.45
   </td>
   <td>35.33
   </td>
   <td>49.06
   </td>
  </tr>
  </tbody>
</table>


There are 2 primary outcomes we sought, which we mostly met, and a third development which was an unexpected and interesting discovery:



1. **Pursue academic benchmarks**
    1. **Goal:** Reaching SoTA on the <a href = 'http://fever.ai'>FEVER benchmark</a>.
    2. **Outcome:** 0.74 label accuracy (#2 overall), 0.637 FEVER score (#11 overall), on the [perpetual evaluation scoreboard ](https://competitions.codalab.org/competitions/18814#results)
    3. While we were very happy with our pipeline’s label accuracy rank, our trained ElasticSearch evidence retriever only achieved 85% document recall on the test set, whereas state of the art set by Athene (University of Darmstadt) was 93%. Since the FEVER score is calculated by discounting examples where some evidence is missing in the top 5 from overall accuracy, we believe this to me the primary factor lowering our overall rank in this category. However, our document retriever relies on **no entity extraction or other ML**, making it more practical and faster for production. Additionally it doesn’t rely on an external API like wikimedia, which Athene pipeline does.
2. **Production-ready ML**
    1. **Goal:** Be the first team to deploy their fact-checking pipeline in a highly available low-latency production environment
    2. **Outcome:** We realized that there was no elastic microservice framework for NLP that would allow us to easily deploy our pipeline, so we built our own, [Koursaros](https://github.com/koursaros-ai/koursaros).  [GNES](https://hanxiao.github.io/2019/07/29/Generic-Neural-Elastic-Search-From-bert-as-service-and-Go-Way-Beyond/) is notably building a similar platform, but are still in the process of development, and lack a stable release. Additionally their platform specifically for neural search, and not inference. We have used Koursaros to deploy our pipeline with a frontend to Kubernetes.
3. **Using Hyperparameter Tuning with a Bayesian Optimizer to Boost Elasticsearch**
    1. This was not part of our initial plan, but after establishing a baseline system for document retrieval on elastic search, we were shocked by how dismal the results were (65% document recall compared with 93% SoTA). Elasticsearch offers the sharding scalability which we sought so we didn’t want to switch to an entirely different system
    2. We trained the BM25 scorer function on the fever.ai set using Facebook’s Ax Bayesian Optimizer, and boosted our document recall from ~**65%** to ~**85%.**
    3. Even though this system is still not state of the art, **we view this as one of the most novel results of our paper, not only because such a result has not yet been published to our knowledge, but because ElasticSearch is so widely adopted such a method has phenomenal real-world implications for production system.**



## Conclusion and Next Steps

Overall we are thrilled with these initial results, but will continue to iterate, especially on the article retrieval model. We believe that we have reached the limit of our ability to train Elasticsearch, and moving forward plan to train a vector based neural KNN search, using custom pytorch code and vector space indexing. Such methodology is notably only occasionally talked about in academic literature, but holds incredible promise. We are closely monitoring the development of [GNES](https://hanxiao.github.io/2019/07/29/Generic-Neural-Elastic-Search-From-bert-as-service-and-Go-Way-Beyond/), a project which has a similar aim to ours. 


<!-- Footnotes themselves at the bottom. -->
## Notes

[^1]:
     [http://news.mit.edu/2018/study-twitter-false-news-travels-faster-true-stories-0308](http://news.mit.edu/2018/study-twitter-false-news-travels-faster-true-stories-0308)

[^2]:
     [https://arxiv.org/abs/1803.05355](https://arxiv.org/abs/1803.05355)

[^3]:
    [ http://aclweb.org/anthology/W18-5516](http://aclweb.org/anthology/W18-5516) (University of Darmstadt)
