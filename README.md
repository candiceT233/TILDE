# TILDE
This is the official repository for the SIGIR2021 paper "TILDE: Term Independent Likelihood moDEl for Passage Re-ranking"

For details about this paper, check out our [website](http://ielab.io/publications/arvin-2021-TILDE).

TILDE now is on huggingface model hub, you can directly download and use it by:

```
from transformers import BertLMHeadModel, BertTokenizerFast

model = BertLMHeadModel.from_pretrained("ielab/TILDE")
tokenizer = BertTokenizerFast.from_pretrained('bert-base-uncased')
```

## Prepare environment and the data folder
We use [huggingface](https://huggingface.co/) implementation of BERT and [pytorch-lightning](https://www.pytorchlightning.ai/) to train TILDE. run `pip install -r requirements.txt` in the root folder to set up the libraries that will be used in this repo.

To repoduce the results presented in the paper, you need to download `collection.tar.gz` from MS MARCO passage ranking repo with this [link](https://msmarco.blob.core.windows.net/msmarcoranking/collection.tar.gz). Unzip and put `collection.tsv` into the folder `./data`.

In order to reproduce the results with minimum effort, we also provided the TREC DL2019 query file (`DL2019-queries.tsv`) in the `./data/queries/` folder, and its qrel file (`2019qrels-pass.txt`) in `./data/qrels/`. There is also a TREC style BM25 run file (`run.trec2019-bm25.res`) generated by [pyserini](https://github.com/castorini/pyserini) in `./data/runs/` folder which we will use to re-rank.

## Passage re-ranking with TILDE
TILDE uses BERT to pre-compute passage representations. Since the MS MARCO passage collection has around 8.8m passages, it will require more than 500G to store the representations of the whole collection. To quickly try out TILDE, in this example, we only pre-compute passages that we need to re-rank.

###Indexing the collection

First, run the following command from the root:

```
python3 indexing.py --run_path ./data/runs/run.trec2019-bm25.res
```
If you have a gpu with big memory, you can set `--batch_size` that suits your gpu the best.

This command will create a mini index in the folder `./data/index/` that stores representations of passages in the BM25 run file.

If you want to index the whole collection, simply run:

```
python3 indexing.py
```
###Re-rank BM25 results.
After you got the index, now you can use TILDE to re-rank BM25 results.

Let‘s first check out what is the BM25 performance on TREC DL2019 with [trec_eval](https://github.com/usnistgov/trec_eval):

```
trec_eval -m ndcg_cut.10 -m map ./data/qrels/2019qrels-pass.txt ./data/runs/run.trec2019-bm25.res
```
we get:

```
map                     all     0.3766
ndcg_cut_10             all     0.4973
```

Now run the command bellow to use TILDE to re-rank BM25 top1000 results:

```
python3 inference.py --run_path ./data/runs/run.trec2019-bm25.res --query_path ./data/queries/DL2019-queries.tsv --index_path ./data/index/passage_embeddings.pkl 
```
It will generate another run file in `./data/runs/` and also will print the query latency of the average query processing time and re-ranking time:

```
Query processing time: 0.2 ms
passage re-ranking time: 6.7 ms
```
In our case, we use an intel cpu version of Mac mini without cuda library, this means we do not use any gpu in this example. TILDE only uses 0.2ms to compute the query sparse representation and 6.7ms to re-rank 1000 passages retrieved by BM25. Note, by default, the code uses a pure query likelihood ranking setting (alpha=1).

Now let's evaluate the TILDE run:

```
trec_eval -m ndcg_cut.10 -m map ./data/qrels/2019qrels-pass.txt ./data/runs/TILDE_alpha1.res 
```
we get:

```
map                     all     0.4058
ndcg_cut_10             all     0.5791
```
This means, with only 0.2ms + 6.7ms add on BM25, TILDE can improve the performance quite a bit. If you want more improvement, you can interpolate query likelihood score with document likelihood by:

```
python3 inference.py --run_path ./data/runs/run.trec2019-bm25.res --query_path ./data/queries/DL2019-queries.tsv --index_path ./data/index/passage_embeddings.pkl --alpha 0.5
```
you will get higher query latency:

```
Query processing time: 68.0 ms
passage re-ranking time: 16.4 ms
```
This is because now TILDE has an extra step of using BERT to compute query dense representation. As a trade-off you will get higher effectiveness:

```
trec_eval -m ndcg_cut.10 -m map ./data/qrels/2019qrels-pass.txt ./data/runs/TILDE_alpha0.5.res 
```
```
map                     all     0.4204
ndcg_cut_10             all     0.6088
```

## To train TILDE
To be available soon
