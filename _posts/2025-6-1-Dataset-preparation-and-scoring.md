---
layout: post
title:  "(1/2) Análise de sentimentos no dataset LKML5Ws"
---
Guilherme Santos Gabriel

# Preparo dos dados e scoring de sentimento — lista amd-gfx

Esse é o primeiro post relacionado a como os dados do dataset LKML5Ws, especificamente na lista da amd-gfx, foram processados e pontuados num processo de análise de sentimento usando a ferramenta VADER para a disciplina de Desenvolvimento de Software Livre no IME-USP.

## Reconstruindo as threads

O primeiro passo será reconstruir as threads de emails. Por padrão, o dataset possui apenas os emails de forma individual, sem uma divisão entre emails de uma mesma thread. Essa informação será importante na etapa seguinte de análise, e portanto nós começamos por tentar reconstruir a thread de emails usando a coluna "in_reply_to" do dataset. Essa coluna, em respostas, aponta para o "message_id" da mensagem que foi respondida e nos permite adicionar uma nova coluna de "thread_id". 


```python
import polars as pl
import re


def add_thread_ids(df: pl.DataFrame) -> pl.DataFrame:
    
    #thread_id = root message_id of the conversation.    

    #message_id -> parent message_id
    parent = dict(
        zip(
            df["message_id"].to_list(),
            df["in_reply_to"].to_list()
        )
    )

    cache = {}

    def find_root(msg_id):
        if msg_id is None:
            return None

        if msg_id in cache:
            return cache[msg_id]

        current = msg_id
        visited = set()

        while True:
            # Broken chain
            if current not in parent:
                root = current
                break

            p = parent[current]

            # Reached the beginning of the thread
            if p is None:
                root = current
                break

            # Parent missing from dataset
            if p not in parent:
                root = p
                break

            # Protect against malformed data
            if current in visited:
                root = current
                break

            visited.add(current)
            current = p

        for node in visited:
            cache[node] = root
        cache[msg_id] = root

        return root

    thread_ids = [find_root(mid) for mid in df["message_id"]]

    return df.with_columns(
        pl.Series("thread_id", thread_ids)
    )



```

## Limpando o corpo dos e-mails

Agora, a próxima etapa e até hoje um "work in progress" é a tentativa de limpar o corpo dos e-mails. A coluna que possui o corpo do email é a coluna "raw_body", mas ela não possui *exclusivamente* o corpo do email. Ela possui tudo, basicamente todo o email, com os trailers, códigos, mensagens anteriores, textos adicionados automaticamente pela mailing list (como "[AMD Official Use Only - General]") e, finalmente, a mensagem propriamente dita.

Desse modo, a tentativa de limpeza tenta remover todos esses elementos usando um conjunto de marcadores, checagem de padrões e parando em elementos finais (como código, sinalizações de fim de mensagem como "--" e outros).


```python
DIFF_MARKERS = (
    "diff --git",
    "Index:",
    "@@",
)

PATTERN = re.compile(
    r"^(Signed[- ]off[- ]by|Acked[- ]by|Reviewed[- ]by|"
    r"Suggested[- ]by|Tested[- ]by|Reported[- ]by|"
    r"Fixes|Link|Co[- ]developed[- ]by):"
)

def clean_email(body: str) -> str:
    if body is None:
        return ""

    cleaned = []

    for line in body.splitlines():

        if (line.lstrip().startswith(">") 
            or line.startswith("From:")
            or line.startswith("[AMD Official Use Only - General]")
            or line.startswith("On Mon")
            or line.startswith("On Tue")
            or line.startswith("On Wed")
            or line.startswith("On Thu")
            or line.startswith("On Fri")
            or line.startswith("On Sat")
            or line.startswith("On Sun")
            ):
            continue
        
        if PATTERN.match(line) or line.strip() == "--" or line.startswith("-----Original Message-----") or line.startswith("Sent:"):
            break

        if (
            line == "---"
            or any(line.startswith(marker) for marker in DIFF_MARKERS)
        ):
            break

        cleaned.append(line)

    text = "\n".join(cleaned)

    text = re.sub(r"\n{3,}", "\n\n", text)
    text = re.sub(r"[ \t]+", " ", text)

    return text.strip()
```

## Carregando e filtrando o dataset

Agora vamos carregar o dataset, filtrá-lo apenas por patches (para facilitar a análise, já que patches são melhor formatados que discussões e porque permitirá análises melhores na próxima etapa) e dividí-lo pelos thread_id.


```python
df = pl.read_parquet("parquets/amd-gfx_list_data.parquet")

df = df.filter(pl.col("has_patch_tag"))

df = add_thread_ids(df)

print("dataset divido em threads:")
print(df)

```

## Aplicando a limpeza ao corpo dos e-mails

A função clean_email() é responsável por limpar o email, como dito anteriormente.


```python
df = df.with_columns(
    pl.col("raw_body")
      .map_elements(clean_email, return_dtype=pl.String)
      .alias("clean_body")
)

print("raw_body limpo:")
print(df['clean_body'])

```

    shape: (134_774,)
    Series: 'clean_body' [str]
    [
    	"AMD General"
    	"<Pratik.Vishwakarma@amd.com> w…
    	"<Pratik.Vishwakarma@amd.com> w…
    	"<Pratik.Vishwakarma@amd.com> w…
    	"<Pratik.Vishwakarma@amd.com> w…
    	…
    	"Since we now raise the clocks …
    	""
    	"Was never used as far as I can…
    	"Am 21.07.2016 um 09:46 schrieb…
    	"Change-Id: If00d5b97ba9e30f9b7…
    ]


## Conferência manual

Essa etapa foi simplesmente para gerar arquivos de saída, confirmando se o resultado estava válido até então.


```python
from pathlib import Path
import polars as pl

OUTPUT_FILE = "first_30_threads.txt"

# Get the first 30 thread IDs
first_threads = (
    df
    .select("thread_id")
    .unique()
    .head(30)
    .to_series()
    .to_list()
)

with open(OUTPUT_FILE, "w", encoding="utf-8") as f:

    for thread_num, thread_id in enumerate(first_threads, start=1):

        f.write("=" * 100 + "\n")
        f.write(f"THREAD {thread_num}\n")
        f.write(f"THREAD ID: {thread_id}\n")
        f.write("=" * 100 + "\n\n")

        thread = (
            df
            .filter(pl.col("thread_id") == thread_id)
            .sort("date")
        )

        for email_num, row in enumerate(thread.iter_rows(named=True), start=1):

            f.write("-" * 100 + "\n")
            f.write(f"EMAIL #{email_num}\n")
            f.write("-" * 100 + "\n")

            f.write(f"Message-ID : {row['message_id']}\n")
            f.write(f"In-Reply-To: {row['in_reply_to']}\n")
            f.write(f"From       : {row['from']}\n")
            f.write(f"To         : {row['to']}\n")
            f.write(f"Date       : {row['date']}\n")
            f.write(f"Subject    : {row['subject']}\n")
            f.write(f"Patch?     : {row['has_patch_tag']}\n")
            f.write("\n")

            f.write("BODY\n")
            f.write("~" * 100 + "\n")
            f.write(str(row['clean_body']))
            f.write("\n")
            f.write("~" * 100 + "\n\n")

        f.write("\n\n")

print(f"Saved to {OUTPUT_FILE}")
```

    Saved to first_15_threads.txt


## Calculando o score de sentimento

A ferramenta utilizada foi o VADER, que é comumente usado em análise de sentimentos, principalmente por ser leve, já que não é baseado em transformers ou técnicas mais pesadas e sim em um conjunto de regras que pontuam os tokens como negativos e positivos. Quando tentamos outros modelos (como o RoBERTa, muito famoso também) baseados em técnicas mais robustas, o tempo de inferência foi muito maior do que o razoável e permitido no hardware que tínhamos disponível. Assim, seguimos com o VADER, que fornece uma pontuação que varia entre [-1, 1] dependendo do quão negativo ou positivo o conjunto de texto foi.



```python
import polars as pl
from vaderSentiment.vaderSentiment import SentimentIntensityAnalyzer

vader = SentimentIntensityAnalyzer()

def vader_score(text: str) -> float:
    if not text or text.isspace():
        return 0.0

    return vader.polarity_scores(text)["compound"]

```

    /home/gui/main_venv/lib/python3.10/site-packages/tqdm/auto.py:21: TqdmWarning: IProgress not found. Please update jupyter and ipywidgets. See https://ipywidgets.readthedocs.io/en/stable/user_install.html
      from .autonotebook import tqdm as notebook_tqdm
    Warning: You are sending unauthenticated requests to the HF Hub. Please set a HF_TOKEN to enable higher rate limits and faster downloads.
    Loading weights: 100%|██████████| 201/201 [00:00<00:00, 30811.17it/s]
    [transformers] [1mRobertaForSequenceClassification LOAD REPORT[0m from: cardiffnlp/twitter-roberta-base-sentiment-latest
    Key                         | Status     |  | 
    ----------------------------+------------+--+-
    roberta.pooler.dense.weight | UNEXPECTED |  | 
    roberta.pooler.dense.bias   | UNEXPECTED |  | 
    
    Notes:
    - UNEXPECTED:	can be ignored when loading from different task/architecture; not ok if you expect identical arch.


Por fim, aplicamos o score gerado pelo vader em cada linha e salvamos o resultado em um .parquet novo e usado na etapa seguinte!


```python

texts = (
    df["clean_body"]
    .fill_null("")
    .to_list()
)

print("setting up vader score")
vader_scores = [vader_score(t) for t in texts]

df = df.with_columns([
    pl.Series("vader_score", vader_scores),
])

print(
    df.select(
        "clean_body",
        "vader_score",
    ).head()
)

print(df['vader_score'])
df.write_parquet("amd-gfx_list_data_SCORED.parquet")

```

    setting up vader score
    shape: (5, 2)
    ┌─────────────────────────────────┬─────────────┐
    │ clean_body                      ┆ vader_score │
    │ ---                             ┆ ---         │
    │ str                             ┆ f64         │
    ╞═════════════════════════════════╪═════════════╡
    │ AMD General                     ┆ 0.0         │
    │ <Pratik.Vishwakarma@amd.com> w… ┆ 0.0         │
    │ <Pratik.Vishwakarma@amd.com> w… ┆ 0.0         │
    │ <Pratik.Vishwakarma@amd.com> w… ┆ 0.0         │
    │ <Pratik.Vishwakarma@amd.com> w… ┆ 0.0         │
    └─────────────────────────────────┴─────────────┘
    shape: (134_774,)
    Series: 'vader_score' [f64]
    [
    	0.0
    	0.0
    	0.0
    	0.0
    	0.0
    	…
    	-0.25
    	0.0
    	0.0
    	0.3947
    	0.0
    ]


### Conclusões

Agora temos um dataset minimamente preparado para análise. O próximo post é sobre a análise que aplicá-mos (e como pretendemos melhorá-la no futuro!)
