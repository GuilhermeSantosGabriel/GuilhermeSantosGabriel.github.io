---
layout: post
title:  "(2/2) Análise de sentimentos no dataset LKML5Ws"
---
Guilherme Santos Gabriel

# Análise de sentimento — lista de e-mails amd-gfx

Esse é o segundo post relacionado a como os dados do dataset LKML5Ws, especificamente na lista da amd-gfx, foram analisados num processo de análise de sentimento usando a ferramenta VADER para a disciplina de Desenvolvimento de Software Livre no IME-USP.

Começamos por carregar o dataset preparado e pontuado na etapa anterior.


```python
import polars as pl
pl.Config.set_tbl_rows(-1)            # -1 displays all rows
pl.Config.set_tbl_cols(-1)            # -1 displays all columns
pl.Config.set_fmt_str_lengths(999)     # Prevents truncating text inside strings
pl.Config.set_tbl_width_chars(999)

df = pl.read_parquet("parquets/amd-gfx_list_data_SCORED.parquet")
```

### Estatísticas gerais do score

E calculamos algumas estatísticas gerais baseadas no score gerado pelo VADER. Note que essas estatísticas não ligam necessariamente para a thread, são calculadas em cima do dataset inteiro. Em geral, percebemos que o dataset é levemente positivo e com um desvio padrão considerável (0.44).


```python
overall_stats = df.select([
    pl.col("vader_score").mean().alias("mean"),
    pl.col("vader_score").median().alias("median"),
    pl.col("vader_score").std().alias("std"),
    pl.col("vader_score").min().alias("min"),
    pl.col("vader_score").max().alias("max"),
])

print(overall_stats)
```

    shape: (1, 5)
    ┌──────────┬────────┬──────────┬──────┬─────┐
    │ mean     ┆ median ┆ std      ┆ min  ┆ max │
    │ ---      ┆ ---    ┆ ---      ┆ ---  ┆ --- │
    │ f64      ┆ f64    ┆ f64      ┆ f64  ┆ f64 │
    ╞══════════╪════════╪══════════╪══════╪═════╡
    │ 0.121331 ┆ 0.0    ┆ 0.444255 ┆ -1.0 ┆ 1.0 │
    └──────────┴────────┴──────────┴──────┴─────┘


### Classificação em Positive / Neutral / Negative

Agora vamos analisar quantas mensagens positivas, neutras e negativas o dataset possui. o threshold de 0.05 é recomendado pela documentação do VADER:


```python
distribution = (
    df.with_columns(
        pl.when(pl.col("vader_score") > 0.05)
          .then(pl.lit("Positive"))
          .when(pl.col("vader_score") < -0.05)
          .then(pl.lit("Negative"))
          .otherwise(pl.lit("Neutral"))
          .alias("sentiment")
    )
    .group_by("sentiment")
    .len()
)

print(distribution)
```

    shape: (3, 2)
    ┌───────────┬───────┐
    │ sentiment ┆ len   │
    │ ---       ┆ ---   │
    │ str       ┆ u32   │
    ╞═══════════╪═══════╡
    │ Positive  ┆ 59420 │
    │ Neutral   ┆ 44268 │
    │ Negative  ┆ 31086 │
    └───────────┴───────┘


## Estatísticas gerais por thread

Na primeira análise nós podemos salvar um conjunto de estatísticas criadas por thread:


```python
thread_stats_full = (
    df.group_by("thread_id")
      .agg([
          pl.len().alias("num_messages"),
          pl.col("from").n_unique().alias("num_participants"),
          pl.col("vader_score").mean().alias("mean_sentiment"),
          pl.col("vader_score").median().alias("median_sentiment"),
          pl.col("vader_score").min().alias("worst_message"),
          pl.col("vader_score").max().alias("best_message"),
          pl.col("vader_score").std().fill_null(0).alias("sentiment_std"),
      ])
)

# Nota: renomeado para thread_stats_full (em vez de thread_stats) porque mais abaixo recalculamos uma versão mais enxuta de thread_stats — com o mesmo nome, essa versão aqui era descartada silenciosamente sem nunca ser usada.
```

## Utilitários

A função abaixo é responsável por salvar um conjunto de threads para um txt. é usada futuramente.


```python
def save_threads(df_messages, threads, filename):
    with open(filename, "w", encoding="utf-8") as f:

        for thread in threads.iter_rows(named=True):

            thread_id = thread["thread_id"]

            f.write("=" * 120 + "\n")
            f.write(f"THREAD ID      : {thread_id}\n")
            f.write(f"Mean sentiment : {thread['mean_sentiment']:.4f}\n")
            f.write(f"Messages       : {thread['num_messages']}\n")
            f.write("=" * 120 + "\n\n")

            messages = (
                df_messages
                .filter(pl.col("thread_id") == thread_id)
                .sort("date")
            )

            for i, msg in enumerate(messages.iter_rows(named=True), start=1):

                f.write("-" * 120 + "\n")
                f.write(f"EMAIL #{i}\n")
                f.write(f"Date      : {msg['date']}\n")
                f.write(f"From      : {msg['from']}\n")
                f.write(f"Score     : {msg['vader_score']:.4f}\n")
                f.write(f"Patch     : {msg['has_patch_tag']}\n")
                f.write(f"Subject   : {msg['subject']}\n")
                f.write("\n")

                f.write(msg["clean_body"] or "")
                f.write("\n\n")

```

## Perguntas

### Quais são as 10 thread mais negativas?




```python
thread_stats = (
    df.group_by("thread_id")
      .agg([
          pl.len().alias("num_messages"),
          pl.col("vader_score").mean().alias("mean_sentiment"),
      ])
)

worst_threads = thread_stats.sort("mean_sentiment").head(10)

print(worst_threads)
```

    shape: (10, 3)
    ┌────────────────────────────────────────────────────────────────────────────────────────────────────────────────┬──────────────┬────────────────┐
    │ thread_id                                                                                                      ┆ num_messages ┆ mean_sentiment │
    │ ---                                                                                                            ┆ ---          ┆ ---            │
    │ str                                                                                                            ┆ u32          ┆ f64            │
    ╞════════════════════════════════════════════════════════════════════════════════════════════════════════════════╪══════════════╪════════════════╡
    │ 874le1h4by.fsf@esperi.org.uk                                                                                   ┆ 1            ┆ -0.9982        │
    │ BN6PR12MB1809AE48F3B1FD9CCB1DEFA0F7FE0-/b2+HYfkarSEx6ez0IUAagdYzm3356FpvxpqHgZTriW3zl9H0oFU5g@public.gmane.org ┆ 1            ┆ -0.9953        │
    │ 20240603084451.2569401-1-jesse.zhang@amd.com                                                                   ┆ 1            ┆ -0.9953        │
    │ 20191118122759.eTdmLbt05NUNqG-Vhmt4ds3LhjsuShz-6DLq2csm5rU@z                                                   ┆ 1            ┆ -0.9945        │
    │ 20191118082149.71s81AAlz6Q_5GoghCqvBDodpY7MH0hLX27TEANzb40@z                                                   ┆ 1            ┆ -0.994         │
    │ 0fa9086e-49c7-580e-2611-4878c92899f5-5C7GfCeVMHo@public.gmane.org                                              ┆ 1            ┆ -0.9939        │
    │ fd5b1a21-391f-078c-2928-5647be28e3fb-Re5JQEeQqe8AvxtiuMwx3w@public.gmane.org                                   ┆ 1            ┆ -0.9933        │
    │ 20200916023111.349330-1-yebin10@huawei.com                                                                     ┆ 1            ┆ -0.993         │
    │ 20230417210523.2553531-1-arnd@kernel.org                                                                       ┆ 1            ┆ -0.9917        │
    │ 20260406-5-10-clang-amdgpu-hard-float-errors-v1-1-09c4c045f848@kernel.org                                      ┆ 2            ┆ -0.99085       │
    └────────────────────────────────────────────────────────────────────────────────────────────────────────────────┴──────────────┴────────────────┘


### Quais são as threads mais positivas?


```python
best_threads = (
    thread_stats
    .sort("mean_sentiment", descending=True)
    .head(10)
)

print(best_threads)


```

    shape: (10, 3)
    ┌────────────────────────────────────────────────────────────────────────────────────────────────────────────────┬──────────────┬────────────────┐
    │ thread_id                                                                                                      ┆ num_messages ┆ mean_sentiment │
    │ ---                                                                                                            ┆ ---          ┆ ---            │
    │ str                                                                                                            ┆ u32          ┆ f64            │
    ╞════════════════════════════════════════════════════════════════════════════════════════════════════════════════╪══════════════╪════════════════╡
    │ 20190617191700.17899-1-alexander.deucher@amd.com                                                               ┆ 1            ┆ 0.9999         │
    │ 20190617192644.17988-1-alexander.deucher@amd.com                                                               ┆ 1            ┆ 0.9999         │
    │ 1490041835-11255-1-git-send-email-alexander.deucher@amd.com                                                    ┆ 1            ┆ 0.9998         │
    │ 20200601183536.1268046-1-alexander.deucher@amd.com                                                             ┆ 1            ┆ 0.9995         │
    │ 1494442068-8202-1-git-send-email-alexander.deucher@amd.com                                                     ┆ 1            ┆ 0.9995         │
    │ 20190715212437.31793-1-alexander.deucher@amd.com                                                               ┆ 1            ┆ 0.9989         │
    │ BN6PR12MB1348857756669A145968B1BBE8DF0-/b2+HYfkarQX0pEhCR5T8QdYzm3356FpvxpqHgZTriW3zl9H0oFU5g@public.gmane.org ┆ 1            ┆ 0.9973         │
    │ 46261ccd-61dd-0fc5-ba9d-eaafb40da038-ANTagKRnAhcb1SvskN2V4Q@public.gmane.org                                   ┆ 1            ┆ 0.9971         │
    │ 20180321134639.18782-1-alexander.deucher@amd.com                                                               ┆ 1            ┆ 0.997          │
    │ 20180911161842.5480-1-nicholas.kazlauskas@amd.com                                                              ┆ 1            ┆ 0.9969         │
    └────────────────────────────────────────────────────────────────────────────────────────────────────────────────┴──────────────┴────────────────┘


### Salvando os resultados:


```python
save_threads(
    df,
    worst_threads,
    "10_worst_threads.txt",
)

save_threads(
    df,
    best_threads,
    "10_best_threads.txt",
)
```

### Como são os reviews quanto ao sentimento de suas mensagens?


```python
reviewers = (
    df.group_by("from")
      .agg([
          pl.len().alias("messages"),
          pl.col("thread_id").n_unique().alias("threads"),
          pl.col("vader_score").mean().alias("mean_sentiment"),
          pl.col("vader_score").std().fill_null(0).alias("std"),
      ])
      .filter(pl.col("messages") >= 10)
)
```

### Quais os reviewers mais negativos?


```python
print(
    reviewers
    .sort("mean_sentiment")
    .head(20)
)
```

    shape: (20, 5)
    ┌────────────────────────────────────────────────────────────┬──────────┬─────────┬────────────────┬──────────┐
    │ from                                                       ┆ messages ┆ threads ┆ mean_sentiment ┆ std      │
    │ ---                                                        ┆ ---      ┆ ---     ┆ ---            ┆ ---      │
    │ str                                                        ┆ u32      ┆ u32     ┆ f64            ┆ f64      │
    ╞════════════════════════════════════════════════════════════╪══════════╪═════════╪════════════════╪══════════╡
    │ Ran Sun <sunran001@208suo.com>                             ┆ 98       ┆ 98      ┆ -0.789103      ┆ 0.093508 │
    │ GuoHua Chen <chenguohua_716@163.com>                       ┆ 36       ┆ 36      ┆ -0.780494      ┆ 0.096797 │
    │ XueBing Chen <chenxb_99091@126.com>                        ┆ 11       ┆ 11      ┆ -0.762645      ┆ 0.083985 │
    │ chenxuebing <chenxb_99091@126.com>                         ┆ 36       ┆ 36      ┆ -0.730106      ┆ 0.092797 │
    │ Shuotao Xu <shuotaoxu@microsoft.com>                       ┆ 12       ┆ 1       ┆ -0.714183      ┆ 0.222594 │
    │ Maíra Canal <maira.canal@usp.br>                           ┆ 12       ┆ 2       ┆ -0.707217      ┆ 0.176326 │
    │ sunran001@208suo.com                                       ┆ 37       ┆ 33      ┆ -0.647911      ┆ 0.161286 │
    │ Zhenneng Li <lizhenneng@kylinos.cn>                        ┆ 23       ┆ 16      ┆ -0.626104      ┆ 0.485484 │
    │ Yinjie Yao <yinjie.yao@amd.com>                            ┆ 21       ┆ 1       ┆ -0.607         ┆ 0.0      │
    │ Isabella Basso <isabbasso@riseup.net>                      ┆ 19       ┆ 4       ┆ -0.565968      ┆ 0.434746 │
    │ Mikulas Patocka <mpatocka@redhat.com>                      ┆ 15       ┆ 8       ┆ -0.565707      ┆ 0.412219 │
    │ Changfeng.Zhu <changfeng.zhu-5C7GfCeVMHo@public.gmane.org> ┆ 10       ┆ 10      ┆ -0.56207       ┆ 0.545033 │
    │ Jason Yan <yanaijie@huawei.com>                            ┆ 20       ┆ 19      ┆ -0.559665      ┆ 0.238677 │
    │ Navid Emamdoost <navid.emamdoost@gmail.com>                ┆ 15       ┆ 10      ┆ -0.559427      ┆ 0.409877 │
    │ Chen Zhou <chenzhou10@huawei.com>                          ┆ 21       ┆ 7       ┆ -0.556329      ┆ 0.325242 │
    │ Trigger Huang <Trigger.Huang@amd.com>                      ┆ 16       ┆ 5       ┆ -0.553894      ┆ 0.343742 │
    │ Jiange Zhao <Jiange.Zhao@amd.com>                          ┆ 11       ┆ 11      ┆ -0.525855      ┆ 0.276796 │
    │ Zheng Bin <zhengbin13@huawei.com>                          ┆ 35       ┆ 6       ┆ -0.524129      ┆ 0.387078 │
    │ Lee Jones <lee.jones@linaro.org>                           ┆ 509      ┆ 40      ┆ -0.517551      ┆ 0.45742  │
    │ Jiapeng Chong <jiapeng.chong@linux.alibaba.com>            ┆ 164      ┆ 124     ┆ -0.51133       ┆ 0.239168 │
    └────────────────────────────────────────────────────────────┴──────────┴─────────┴────────────────┴──────────┘


### Quais os reviewers mais positivos?


```python
print(
    reviewers
    .sort("mean_sentiment", descending=True)
    .head(20)
)
```

    shape: (20, 5)
    ┌────────────────────────────────────────────────────────────────────────┬──────────┬─────────┬────────────────┬──────────┐
    │ from                                                                   ┆ messages ┆ threads ┆ mean_sentiment ┆ std      │
    │ ---                                                                    ┆ ---      ┆ ---     ┆ ---            ┆ ---      │
    │ str                                                                    ┆ u32      ┆ u32     ┆ f64            ┆ f64      │
    ╞════════════════════════════════════════════════════════════════════════╪══════════╪═════════╪════════════════╪══════════╡
    │ Martin Andrew <Andrew.Martin@amd.com>                                  ┆ 21       ┆ 16      ┆ 0.790538       ┆ 0.156321 │
    │ Kai Wasserbäch <kai-1ZKVMVCtJ2dx9oSEVPI0kiST3g8Odh+X@public.gmane.org> ┆ 16       ┆ 16      ┆ 0.7594         ┆ 0.216949 │
    │ Tao Yintian <Yintian.Tao-5C7GfCeVMHo@public.gmane.org>                 ┆ 21       ┆ 21      ┆ 0.717752       ┆ 0.291004 │
    │ Nirujogi Pratap <pnirujog@amd.com>                                     ┆ 26       ┆ 14      ┆ 0.711419       ┆ 0.197904 │
    │ Tejun Heo <tj-DgEjT+Ai2ygdnm+yROfE0A@public.gmane.org>                 ┆ 13       ┆ 13      ┆ 0.701662       ┆ 0.297069 │
    │ Paul Menzel <pmenzel@molgen.mpg.de>                                    ┆ 181      ┆ 99      ┆ 0.701312       ┆ 0.391749 │
    │ Paulo Miguel Almeida <paulo.miguel.almeida.rodenas@gmail.com>          ┆ 16       ┆ 11      ┆ 0.701169       ┆ 0.291654 │
    │ Mao David <David.Mao-5C7GfCeVMHo@public.gmane.org>                     ┆ 12       ┆ 12      ┆ 0.691125       ┆ 0.404139 │
    │ Tejun Heo <tj@kernel.org>                                              ┆ 15       ┆ 8       ┆ 0.67992        ┆ 0.250166 │
    │ Liu01 Tong (Esther) <Tong.Liu01@amd.com>                               ┆ 12       ┆ 9       ┆ 0.6776         ┆ 0.513283 │
    │ Zi Yan <ziy@nvidia.com>                                                ┆ 11       ┆ 2       ┆ 0.669618       ┆ 0.441582 │
    │ Daniel Stone <daniel@fooishbar.org>                                    ┆ 48       ┆ 27      ┆ 0.667158       ┆ 0.312724 │
    │ "Jesse.zhang@amd.com" <jesse.zhang@amd.com>                            ┆ 13       ┆ 10      ┆ 0.638515       ┆ 0.330724 │
    │ Paul Menzel <pmenzel+amd-gfx-KUpvgZVWgV9o1qOY/usvUg@public.gmane.org>  ┆ 23       ┆ 23      ┆ 0.627839       ┆ 0.383959 │
    │ Easwar Hariharan <eahariha@linux.microsoft.com>                        ┆ 77       ┆ 5       ┆ 0.623479       ┆ 0.169527 │
    │ Gary Guo <gary@garyguo.net>                                            ┆ 14       ┆ 6       ┆ 0.611471       ┆ 0.395209 │
    │ Sumit Semwal <sumit.semwal@linaro.org>                                 ┆ 15       ┆ 12      ┆ 0.60992        ┆ 0.32967  │
    │ Lancelot SIX <Lancelot.Six@amd.com>                                    ┆ 25       ┆ 17      ┆ 0.60982        ┆ 0.332292 │
    │ Shankar Uma <uma.shankar@intel.com>                                    ┆ 21       ┆ 4       ┆ 0.602733       ┆ 0.214285 │
    │ Lin Wayne <Wayne.Lin@amd.com>                                          ┆ 65       ┆ 38      ┆ 0.601003       ┆ 0.336354 │
    └────────────────────────────────────────────────────────────────────────┴──────────┴─────────┴────────────────┴──────────┘


## Comparação: removendo threads outliers (corte de -0.5 a 0.5)

Agora, analisando os resultados DE FATO, podemos perceber que a maioria dos casos positivos e negativos são na realidade outliers. De certo modo faz sentido, já que ambos foram resultados nos limites do VADER. é muito difícil que naturalmente um texto de patch seja "positivo" ou "tóxico" o suficiente para atingir os limites. (Afinal, o VADER foi feito pra textos informais, muito mais carregados emocionalmente do que um simples texto do kernel linux). Assim, tivemos a ideia de realizar todos os testes anteriores para valores mais razoáveis e assim chegamos num novo limiar que varia de [-0.5, 0.5]. Realizamos todos os testes novamente:


```python
thread_sentiment = (
    df.group_by("thread_id")
      .agg([
          pl.len().alias("num_messages"),
          pl.col("vader_score").mean().alias("mean_sentiment"),
      ])
)

outlier_threads = thread_sentiment.filter(
    (pl.col("mean_sentiment") > 0.5) | (pl.col("mean_sentiment") < -0.5)
)

normal_threads = thread_sentiment.filter(
    (pl.col("mean_sentiment") >= -0.5) & (pl.col("mean_sentiment") <= 0.5)
)

print(f"Threads totais    : {thread_sentiment.height}")
print(f"Threads outliers  : {outlier_threads.height} "
      f"({outlier_threads.height / thread_sentiment.height:.1%})")
print(f"Threads 'normais' : {normal_threads.height} "
      f"({normal_threads.height / thread_sentiment.height:.1%})")

# Dataset filtrado: só as mensagens de threads consideradas "normais"
df_normal = df.filter(pl.col("thread_id").is_in(normal_threads["thread_id"]))
```

    Threads totais    : 39445
    Threads outliers  : 7370 (18.7%)
    Threads 'normais' : 32075 (81.3%)


    /tmp/ipykernel_19218/3275602532.py:24: DeprecationWarning: `is_in` with a collection of the same datatype is ambiguous and deprecated.
    Please use `implode` to return to previous behavior.
    
    See https://github.com/pola-rs/polars/issues/22149 for more information.
      df_normal = df.filter(pl.col("thread_id").is_in(normal_threads["thread_id"]))


### Estatísticas gerais (com vs. sem outliers)


```python
overall_stats_normal = df_normal.select([
    pl.col("vader_score").mean().alias("mean"),
    pl.col("vader_score").median().alias("median"),
    pl.col("vader_score").std().alias("std"),
    pl.col("vader_score").min().alias("min"),
    pl.col("vader_score").max().alias("max"),
])

print("Com outliers (corte -1 a 1, dataset completo):")
print(overall_stats)

print("Sem outliers (corte -0.5 a 0.5):")
print(overall_stats_normal)
```

    Com outliers (corte -1 a 1, dataset completo):
    shape: (1, 5)
    ┌──────────┬────────┬──────────┬──────┬─────┐
    │ mean     ┆ median ┆ std      ┆ min  ┆ max │
    │ ---      ┆ ---    ┆ ---      ┆ ---  ┆ --- │
    │ f64      ┆ f64    ┆ f64      ┆ f64  ┆ f64 │
    ╞══════════╪════════╪══════════╪══════╪═════╡
    │ 0.121331 ┆ 0.0    ┆ 0.444255 ┆ -1.0 ┆ 1.0 │
    └──────────┴────────┴──────────┴──────┴─────┘
    Sem outliers (corte -0.5 a 0.5):
    shape: (1, 5)
    ┌──────────┬────────┬──────────┬──────┬─────┐
    │ mean     ┆ median ┆ std      ┆ min  ┆ max │
    │ ---      ┆ ---    ┆ ---      ┆ ---  ┆ --- │
    │ f64      ┆ f64    ┆ f64      ┆ f64  ┆ f64 │
    ╞══════════╪════════╪══════════╪══════╪═════╡
    │ 0.099797 ┆ 0.0    ┆ 0.415733 ┆ -1.0 ┆ 1.0 │
    └──────────┴────────┴──────────┴──────┴─────┘


### Distribuição de sentimento (com vs. sem outliers)


```python
distribution_normal = (
    df_normal.with_columns(
        pl.when(pl.col("vader_score") > 0.05)
          .then(pl.lit("Positive"))
          .when(pl.col("vader_score") < -0.05)
          .then(pl.lit("Negative"))
          .otherwise(pl.lit("Neutral"))
          .alias("sentiment")
    )
    .group_by("sentiment")
    .len()
)

print("Com outliers:")
print(distribution)

print("Sem outliers:")
print(distribution_normal)
```

    Com outliers:
    shape: (3, 2)
    ┌───────────┬───────┐
    │ sentiment ┆ len   │
    │ ---       ┆ ---   │
    │ str       ┆ u32   │
    ╞═══════════╪═══════╡
    │ Positive  ┆ 59420 │
    │ Neutral   ┆ 44268 │
    │ Negative  ┆ 31086 │
    └───────────┴───────┘
    Sem outliers:
    shape: (3, 2)
    ┌───────────┬───────┐
    │ sentiment ┆ len   │
    │ ---       ┆ ---   │
    │ str       ┆ u32   │
    ╞═══════════╪═══════╡
    │ Neutral   ┆ 43949 │
    │ Positive  ┆ 50634 │
    │ Negative  ┆ 28136 │
    └───────────┴───────┘


### Quais são as 10 threads mais negativas? (sem outliers)


```python
thread_stats_normal = (
    df_normal.group_by("thread_id")
      .agg([
          pl.len().alias("num_messages"),
          pl.col("vader_score").mean().alias("mean_sentiment"),
      ])
)

worst_threads_normal = thread_stats_normal.sort("mean_sentiment").head(10)

print("Com outliers:")
print(worst_threads)

print("Sem outliers:")
print(worst_threads_normal)
```

    Com outliers:
    shape: (10, 3)
    ┌────────────────────────────────────────────────────────────────────────────────────────────────────────────────┬──────────────┬────────────────┐
    │ thread_id                                                                                                      ┆ num_messages ┆ mean_sentiment │
    │ ---                                                                                                            ┆ ---          ┆ ---            │
    │ str                                                                                                            ┆ u32          ┆ f64            │
    ╞════════════════════════════════════════════════════════════════════════════════════════════════════════════════╪══════════════╪════════════════╡
    │ 874le1h4by.fsf@esperi.org.uk                                                                                   ┆ 1            ┆ -0.9982        │
    │ BN6PR12MB1809AE48F3B1FD9CCB1DEFA0F7FE0-/b2+HYfkarSEx6ez0IUAagdYzm3356FpvxpqHgZTriW3zl9H0oFU5g@public.gmane.org ┆ 1            ┆ -0.9953        │
    │ 20240603084451.2569401-1-jesse.zhang@amd.com                                                                   ┆ 1            ┆ -0.9953        │
    │ 20191118122759.eTdmLbt05NUNqG-Vhmt4ds3LhjsuShz-6DLq2csm5rU@z                                                   ┆ 1            ┆ -0.9945        │
    │ 20191118082149.71s81AAlz6Q_5GoghCqvBDodpY7MH0hLX27TEANzb40@z                                                   ┆ 1            ┆ -0.994         │
    │ 0fa9086e-49c7-580e-2611-4878c92899f5-5C7GfCeVMHo@public.gmane.org                                              ┆ 1            ┆ -0.9939        │
    │ fd5b1a21-391f-078c-2928-5647be28e3fb-Re5JQEeQqe8AvxtiuMwx3w@public.gmane.org                                   ┆ 1            ┆ -0.9933        │
    │ 20200916023111.349330-1-yebin10@huawei.com                                                                     ┆ 1            ┆ -0.993         │
    │ 20230417210523.2553531-1-arnd@kernel.org                                                                       ┆ 1            ┆ -0.9917        │
    │ 20260406-5-10-clang-amdgpu-hard-float-errors-v1-1-09c4c045f848@kernel.org                                      ┆ 2            ┆ -0.99085       │
    └────────────────────────────────────────────────────────────────────────────────────────────────────────────────┴──────────────┴────────────────┘
    Sem outliers:
    shape: (10, 3)
    ┌─────────────────────────────────────────────────────────────────────────────────────────────┬──────────────┬────────────────┐
    │ thread_id                                                                                   ┆ num_messages ┆ mean_sentiment │
    │ ---                                                                                         ┆ ---          ┆ ---            │
    │ str                                                                                         ┆ u32          ┆ f64            │
    ╞═════════════════════════════════════════════════════════════════════════════════════════════╪══════════════╪════════════════╡
    │ 20230720032846.1980-1-xujianghui@cdjrlc.com                                                 ┆ 1            ┆ -0.4995        │
    │ 20180301102244.1684-4-christian.koenig-5C7GfCeVMHo@public.gmane.org                         ┆ 1            ┆ -0.4993        │
    │ 20230724175206.11054-1-Philip.Yang@amd.com                                                  ┆ 4            ┆ -0.49885       │
    │ CAOWid-dO5QH4wLyN_ztMaoZtLM9yzw-FEMgk3ufbh1ahHJ2vVg-JsoAwUIsXosN+BqQ9rBEUg@public.gmane.org ┆ 1            ┆ -0.4987        │
    │ 9a8d8f7f-a468-fd5b-dec5-472ce9c88483-otUistvHUpPR7s880joybQ@public.gmane.org                ┆ 1            ┆ -0.4986        │
    │ 20220413030854.31724-1-xinhui.pan@amd.com                                                   ┆ 2            ┆ -0.49855       │
    │ 20230517155642.2393980-1-srinivasan.shanmugam@amd.com                                       ┆ 2            ┆ -0.4982        │
    │ 1573112184-14195-1-git-send-email-zhexi.zhang@amd.com                                       ┆ 2            ┆ -0.4981        │
    │ 20220304065037.1050-1-Stanley.Yang@amd.com                                                  ┆ 3            ┆ -0.4976        │
    │ 20231114063647.71929-1-jose.pekkarinen@foxhound.fi                                          ┆ 2            ┆ -0.49755       │
    └─────────────────────────────────────────────────────────────────────────────────────────────┴──────────────┴────────────────┘


### Quais são as threads mais positivas? (sem outliers)


```python
best_threads_normal = (
    thread_stats_normal
    .sort("mean_sentiment", descending=True)
    .head(10)
)

print("Com outliers:")
print(best_threads)

print("Sem outliers:")
print(best_threads_normal)
```

    Com outliers:
    shape: (10, 3)
    ┌────────────────────────────────────────────────────────────────────────────────────────────────────────────────┬──────────────┬────────────────┐
    │ thread_id                                                                                                      ┆ num_messages ┆ mean_sentiment │
    │ ---                                                                                                            ┆ ---          ┆ ---            │
    │ str                                                                                                            ┆ u32          ┆ f64            │
    ╞════════════════════════════════════════════════════════════════════════════════════════════════════════════════╪══════════════╪════════════════╡
    │ 20190617191700.17899-1-alexander.deucher@amd.com                                                               ┆ 1            ┆ 0.9999         │
    │ 20190617192644.17988-1-alexander.deucher@amd.com                                                               ┆ 1            ┆ 0.9999         │
    │ 1490041835-11255-1-git-send-email-alexander.deucher@amd.com                                                    ┆ 1            ┆ 0.9998         │
    │ 20200601183536.1268046-1-alexander.deucher@amd.com                                                             ┆ 1            ┆ 0.9995         │
    │ 1494442068-8202-1-git-send-email-alexander.deucher@amd.com                                                     ┆ 1            ┆ 0.9995         │
    │ 20190715212437.31793-1-alexander.deucher@amd.com                                                               ┆ 1            ┆ 0.9989         │
    │ BN6PR12MB1348857756669A145968B1BBE8DF0-/b2+HYfkarQX0pEhCR5T8QdYzm3356FpvxpqHgZTriW3zl9H0oFU5g@public.gmane.org ┆ 1            ┆ 0.9973         │
    │ 46261ccd-61dd-0fc5-ba9d-eaafb40da038-ANTagKRnAhcb1SvskN2V4Q@public.gmane.org                                   ┆ 1            ┆ 0.9971         │
    │ 20180321134639.18782-1-alexander.deucher@amd.com                                                               ┆ 1            ┆ 0.997          │
    │ 20180911161842.5480-1-nicholas.kazlauskas@amd.com                                                              ┆ 1            ┆ 0.9969         │
    └────────────────────────────────────────────────────────────────────────────────────────────────────────────────┴──────────────┴────────────────┘
    Sem outliers:
    shape: (10, 3)
    ┌────────────────────────────────────────────────────────────────────────────────────────┬──────────────┬────────────────┐
    │ thread_id                                                                              ┆ num_messages ┆ mean_sentiment │
    │ ---                                                                                    ┆ ---          ┆ ---            │
    │ str                                                                                    ┆ u32          ┆ f64            │
    ╞════════════════════════════════════════════════════════════════════════════════════════╪══════════════╪════════════════╡
    │ 1582639073-16555-1-git-send-email-ray.huang@amd.com                                    ┆ 12           ┆ 0.499925       │
    │ 67b1875d-9f77-5fb8-bfc6-53d34c15ab16-5C7GfCeVMHo@public.gmane.org                      ┆ 1            ┆ 0.4998         │
    │ 20240305194714.1783500-1-alexander.deucher@amd.com                                     ┆ 1            ┆ 0.4997         │
    │ 20220412152057.1170235-1-lee.jones@linaro.org                                          ┆ 4            ┆ 0.499475       │
    │ 20180731071236.3707-1-tzimmermann-l3A5Bk7waGM@public.gmane.org                         ┆ 2            ┆ 0.49945        │
    │ 1533626820-7701-3-git-send-email-Jerry.Zhang-5C7GfCeVMHo@public.gmane.org              ┆ 3            ┆ 0.499367       │
    │ 20200304151727.221032-1-luben.tuikov@amd.com                                           ┆ 2            ┆ 0.49935        │
    │ a2fcbc0155a7f1eea9a6df9669dc2de75315e937.camel-pghWNbHTmq7QT0dZR+AlfA@public.gmane.org ┆ 2            ┆ 0.49935        │
    │ 20230123202646.356592-1-andrealmeid@igalia.com                                         ┆ 5            ┆ 0.49932        │
    │ 20180420101755.GA11400-wEGCiKHe2LqWVfeAwA7xHQ@public.gmane.org                         ┆ 10           ┆ 0.49922        │
    └────────────────────────────────────────────────────────────────────────────────────────┴──────────────┴────────────────┘


### Salvando os resultados (sem outliers)


```python
save_threads(
    df_normal,
    worst_threads_normal,
    "10_worst_threads_sem_outliers.txt",
)

save_threads(
    df_normal,
    best_threads_normal,
    "10_best_threads_sem_outliers.txt",
)
```

### Reviewers mais negativos e mais positivos (sem outliers)


```python
reviewers_normal = (
    df_normal.group_by("from")
      .agg([
          pl.len().alias("messages"),
          pl.col("thread_id").n_unique().alias("threads"),
          pl.col("vader_score").mean().alias("mean_sentiment"),
          pl.col("vader_score").std().fill_null(0).alias("std"),
      ])
      .filter(pl.col("messages") >= 1)
)

print("Mais negativos - com outliers:")
print(reviewers.sort("mean_sentiment").head(20))

print("Mais negativos - sem outliers:")
print(reviewers_normal.sort("mean_sentiment").head(20))
```

    Mais negativos - com outliers:
    shape: (20, 5)
    ┌────────────────────────────────────────────────────────────┬──────────┬─────────┬────────────────┬──────────┐
    │ from                                                       ┆ messages ┆ threads ┆ mean_sentiment ┆ std      │
    │ ---                                                        ┆ ---      ┆ ---     ┆ ---            ┆ ---      │
    │ str                                                        ┆ u32      ┆ u32     ┆ f64            ┆ f64      │
    ╞════════════════════════════════════════════════════════════╪══════════╪═════════╪════════════════╪══════════╡
    │ Ran Sun <sunran001@208suo.com>                             ┆ 98       ┆ 98      ┆ -0.789103      ┆ 0.093508 │
    │ GuoHua Chen <chenguohua_716@163.com>                       ┆ 36       ┆ 36      ┆ -0.780494      ┆ 0.096797 │
    │ XueBing Chen <chenxb_99091@126.com>                        ┆ 11       ┆ 11      ┆ -0.762645      ┆ 0.083985 │
    │ chenxuebing <chenxb_99091@126.com>                         ┆ 36       ┆ 36      ┆ -0.730106      ┆ 0.092797 │
    │ Shuotao Xu <shuotaoxu@microsoft.com>                       ┆ 12       ┆ 1       ┆ -0.714183      ┆ 0.222594 │
    │ Maíra Canal <maira.canal@usp.br>                           ┆ 12       ┆ 2       ┆ -0.707217      ┆ 0.176326 │
    │ sunran001@208suo.com                                       ┆ 37       ┆ 33      ┆ -0.647911      ┆ 0.161286 │
    │ Zhenneng Li <lizhenneng@kylinos.cn>                        ┆ 23       ┆ 16      ┆ -0.626104      ┆ 0.485484 │
    │ Yinjie Yao <yinjie.yao@amd.com>                            ┆ 21       ┆ 1       ┆ -0.607         ┆ 0.0      │
    │ Isabella Basso <isabbasso@riseup.net>                      ┆ 19       ┆ 4       ┆ -0.565968      ┆ 0.434746 │
    │ Mikulas Patocka <mpatocka@redhat.com>                      ┆ 15       ┆ 8       ┆ -0.565707      ┆ 0.412219 │
    │ Changfeng.Zhu <changfeng.zhu-5C7GfCeVMHo@public.gmane.org> ┆ 10       ┆ 10      ┆ -0.56207       ┆ 0.545033 │
    │ Jason Yan <yanaijie@huawei.com>                            ┆ 20       ┆ 19      ┆ -0.559665      ┆ 0.238677 │
    │ Navid Emamdoost <navid.emamdoost@gmail.com>                ┆ 15       ┆ 10      ┆ -0.559427      ┆ 0.409877 │
    │ Chen Zhou <chenzhou10@huawei.com>                          ┆ 21       ┆ 7       ┆ -0.556329      ┆ 0.325242 │
    │ Trigger Huang <Trigger.Huang@amd.com>                      ┆ 16       ┆ 5       ┆ -0.553894      ┆ 0.343742 │
    │ Jiange Zhao <Jiange.Zhao@amd.com>                          ┆ 11       ┆ 11      ┆ -0.525855      ┆ 0.276796 │
    │ Zheng Bin <zhengbin13@huawei.com>                          ┆ 35       ┆ 6       ┆ -0.524129      ┆ 0.387078 │
    │ Lee Jones <lee.jones@linaro.org>                           ┆ 509      ┆ 40      ┆ -0.517551      ┆ 0.45742  │
    │ Jiapeng Chong <jiapeng.chong@linux.alibaba.com>            ┆ 164      ┆ 124     ┆ -0.51133       ┆ 0.239168 │
    └────────────────────────────────────────────────────────────┴──────────┴─────────┴────────────────┴──────────┘
    Mais negativos - sem outliers:
    shape: (20, 5)
    ┌───────────────────────────────────────────────────────────────────┬──────────┬─────────┬────────────────┬──────────┐
    │ from                                                              ┆ messages ┆ threads ┆ mean_sentiment ┆ std      │
    │ ---                                                               ┆ ---      ┆ ---     ┆ ---            ┆ ---      │
    │ str                                                               ┆ u32      ┆ u32     ┆ f64            ┆ f64      │
    ╞═══════════════════════════════════════════════════════════════════╪══════════╪═════════╪════════════════╪══════════╡
    │ kbuild test robot via dri-devel <dri-devel@lists.freedesktop.org> ┆ 1        ┆ 1       ┆ -0.9999        ┆ 0.0      │
    │ Michael D Labriola <michael.d.labriola@gmail.com>                 ┆ 1        ┆ 1       ┆ -0.9762        ┆ 0.0      │
    │ ozeng <ozeng-5C7GfCeVMHo@public.gmane.org>                        ┆ 1        ┆ 1       ┆ -0.9739        ┆ 0.0      │
    │ Michael Bommarito <michael.bommarito@gmail.com>                   ┆ 1        ┆ 1       ┆ -0.9732        ┆ 0.0      │
    │ Jérôme Glisse <jglisse-H+wXaHxf7aLQT0dZR+AlfA@public.gmane.org>   ┆ 1        ┆ 1       ┆ -0.9675        ┆ 0.0      │
    │ Liu01 Tong <Tong.Liu01@amd.com>                                   ┆ 3        ┆ 3       ┆ -0.9644        ┆ 0.021997 │
    │ David Baum <davidbaum461@gmail.com>                               ┆ 1        ┆ 1       ┆ -0.9565        ┆ 0.0      │
    │ Amir Shetaia <amir.shetaia@amd.com>                               ┆ 1        ┆ 1       ┆ -0.9509        ┆ 0.0      │
    │ Mario Limonciello <superm1@gmail.com>                             ┆ 1        ┆ 1       ┆ -0.9501        ┆ 0.0      │
    │ xurui <xurui@kylinos.cn>                                          ┆ 2        ┆ 2       ┆ -0.9452        ┆ 0.0      │
    │ R SUNDAR <prosunofficial@gmail.com>                               ┆ 1        ┆ 1       ┆ -0.9413        ┆ 0.0      │
    │ Swarup Laxman Kotiaklapudi <swarupkotikalapudi@gmail.com>         ┆ 1        ┆ 1       ┆ -0.9413        ┆ 0.0      │
    │ ts8060 <ts8060@my.bristol.ac.uk>                                  ┆ 1        ┆ 1       ┆ -0.9386        ┆ 0.0      │
    │ Dixit Ashutosh <ashutosh.dixit@intel.com>                         ┆ 1        ┆ 1       ┆ -0.9349        ┆ 0.0      │
    │ Guangshuo Li <lgs201920130244@gmail.com>                          ┆ 2        ┆ 2       ┆ -0.92785       ┆ 0.045467 │
    │ Siyang Liu <Security@tencent.com>                                 ┆ 1        ┆ 1       ┆ -0.9274        ┆ 0.0      │
    │ Ahmed Elmetwally <en22ue@gmail.com>                               ┆ 1        ┆ 1       ┆ -0.9246        ┆ 0.0      │
    │ Ken Xue <Ken.Xue@amd.com>                                         ┆ 1        ┆ 1       ┆ -0.9062        ┆ 0.0      │
    │ Ziyi Guo <n7l8m4@u.northwestern.edu>                              ┆ 1        ┆ 1       ┆ -0.9055        ┆ 0.0      │
    │ Daniel Kurtz <djkurtz-F7+t8E8rja9g9hUCZPvPmw@public.gmane.org>    ┆ 1        ┆ 1       ┆ -0.8989        ┆ 0.0      │
    └───────────────────────────────────────────────────────────────────┴──────────┴─────────┴────────────────┴──────────┘



```python
print("Mais positivos - com outliers:")
print(reviewers.sort("mean_sentiment", descending=True).head(20))

print("Mais positivos - sem outliers:")
print(reviewers_normal.sort("mean_sentiment", descending=True).head(20))
```

    Mais positivos - com outliers:
    shape: (20, 5)
    ┌────────────────────────────────────────────────────────────────────────┬──────────┬─────────┬────────────────┬──────────┐
    │ from                                                                   ┆ messages ┆ threads ┆ mean_sentiment ┆ std      │
    │ ---                                                                    ┆ ---      ┆ ---     ┆ ---            ┆ ---      │
    │ str                                                                    ┆ u32      ┆ u32     ┆ f64            ┆ f64      │
    ╞════════════════════════════════════════════════════════════════════════╪══════════╪═════════╪════════════════╪══════════╡
    │ Martin Andrew <Andrew.Martin@amd.com>                                  ┆ 21       ┆ 16      ┆ 0.790538       ┆ 0.156321 │
    │ Kai Wasserbäch <kai-1ZKVMVCtJ2dx9oSEVPI0kiST3g8Odh+X@public.gmane.org> ┆ 16       ┆ 16      ┆ 0.7594         ┆ 0.216949 │
    │ Tao Yintian <Yintian.Tao-5C7GfCeVMHo@public.gmane.org>                 ┆ 21       ┆ 21      ┆ 0.717752       ┆ 0.291004 │
    │ Nirujogi Pratap <pnirujog@amd.com>                                     ┆ 26       ┆ 14      ┆ 0.711419       ┆ 0.197904 │
    │ Tejun Heo <tj-DgEjT+Ai2ygdnm+yROfE0A@public.gmane.org>                 ┆ 13       ┆ 13      ┆ 0.701662       ┆ 0.297069 │
    │ Paul Menzel <pmenzel@molgen.mpg.de>                                    ┆ 181      ┆ 99      ┆ 0.701312       ┆ 0.391749 │
    │ Paulo Miguel Almeida <paulo.miguel.almeida.rodenas@gmail.com>          ┆ 16       ┆ 11      ┆ 0.701169       ┆ 0.291654 │
    │ Mao David <David.Mao-5C7GfCeVMHo@public.gmane.org>                     ┆ 12       ┆ 12      ┆ 0.691125       ┆ 0.404139 │
    │ Tejun Heo <tj@kernel.org>                                              ┆ 15       ┆ 8       ┆ 0.67992        ┆ 0.250166 │
    │ Liu01 Tong (Esther) <Tong.Liu01@amd.com>                               ┆ 12       ┆ 9       ┆ 0.6776         ┆ 0.513283 │
    │ Zi Yan <ziy@nvidia.com>                                                ┆ 11       ┆ 2       ┆ 0.669618       ┆ 0.441582 │
    │ Daniel Stone <daniel@fooishbar.org>                                    ┆ 48       ┆ 27      ┆ 0.667158       ┆ 0.312724 │
    │ "Jesse.zhang@amd.com" <jesse.zhang@amd.com>                            ┆ 13       ┆ 10      ┆ 0.638515       ┆ 0.330724 │
    │ Paul Menzel <pmenzel+amd-gfx-KUpvgZVWgV9o1qOY/usvUg@public.gmane.org>  ┆ 23       ┆ 23      ┆ 0.627839       ┆ 0.383959 │
    │ Easwar Hariharan <eahariha@linux.microsoft.com>                        ┆ 77       ┆ 5       ┆ 0.623479       ┆ 0.169527 │
    │ Gary Guo <gary@garyguo.net>                                            ┆ 14       ┆ 6       ┆ 0.611471       ┆ 0.395209 │
    │ Sumit Semwal <sumit.semwal@linaro.org>                                 ┆ 15       ┆ 12      ┆ 0.60992        ┆ 0.32967  │
    │ Lancelot SIX <Lancelot.Six@amd.com>                                    ┆ 25       ┆ 17      ┆ 0.60982        ┆ 0.332292 │
    │ Shankar Uma <uma.shankar@intel.com>                                    ┆ 21       ┆ 4       ┆ 0.602733       ┆ 0.214285 │
    │ Lin Wayne <Wayne.Lin@amd.com>                                          ┆ 65       ┆ 38      ┆ 0.601003       ┆ 0.336354 │
    └────────────────────────────────────────────────────────────────────────┴──────────┴─────────┴────────────────┴──────────┘
    Mais positivos - sem outliers:
    shape: (20, 5)
    ┌─────────────────────────────────────────────────────────────────────────────┬──────────┬─────────┬────────────────┬──────────┐
    │ from                                                                        ┆ messages ┆ threads ┆ mean_sentiment ┆ std      │
    │ ---                                                                         ┆ ---      ┆ ---     ┆ ---            ┆ ---      │
    │ str                                                                         ┆ u32      ┆ u32     ┆ f64            ┆ f64      │
    ╞═════════════════════════════════════════════════════════════════════════════╪══════════╪═════════╪════════════════╪══════════╡
    │ Cihangir Akturk <cakturk-Re5JQEeQqe8AvxtiuMwx3w@public.gmane.org>           ┆ 2        ┆ 2       ┆ 0.99785        ┆ 0.000071 │
    │ Andreas Messer <andi@bastelmap.de>                                          ┆ 1        ┆ 1       ┆ 0.976          ┆ 0.0      │
    │ Dan Carpenter via dri-devel <dri-devel@lists.freedesktop.org>               ┆ 1        ┆ 1       ┆ 0.9732         ┆ 0.0      │
    │ Amit Kachhap <Amit.Kachhap@arm.com>                                         ┆ 1        ┆ 1       ┆ 0.9699         ┆ 0.0      │
    │ Mario Kleiner via dri-devel <dri-devel@lists.freedesktop.org>               ┆ 1        ┆ 1       ┆ 0.9549         ┆ 0.0      │
    │ Daniel Latypov <dlatypov@google.com>                                        ┆ 1        ┆ 1       ┆ 0.9546         ┆ 0.0      │
    │ ydirson@free.fr                                                             ┆ 1        ┆ 1       ┆ 0.9506         ┆ 0.0      │
    │ Sebin Sebastian <mailmesebin00@gmail.com>                                   ┆ 1        ┆ 1       ┆ 0.9477         ┆ 0.0      │
    │ patchwork-bot+netdevbpf@kernel.org                                          ┆ 1        ┆ 1       ┆ 0.9422         ┆ 0.0      │
    │ liviu.dudau@arm.com <liviu.dudau@arm.com>                                   ┆ 1        ┆ 1       ┆ 0.9412         ┆ 0.0      │
    │ Jeff Cook <jeff@jeffcook.io>                                                ┆ 1        ┆ 1       ┆ 0.9374         ┆ 0.0      │
    │ Andrew Worsley <amworsley@gmail.com>                                        ┆ 1        ┆ 1       ┆ 0.9311         ┆ 0.0      │
    │ taskboxtester@gmail.com                                                     ┆ 1        ┆ 1       ┆ 0.9265         ┆ 0.0      │
    │ Vivek Das Mohapatra <vivek@collabora.com>                                   ┆ 2        ┆ 1       ┆ 0.9186         ┆ 0.0      │
    │ Welty Brian <brian.welty@intel.com>                                         ┆ 1        ┆ 1       ┆ 0.9095         ┆ 0.0      │
    │ Akash Goel <akash.goel@arm.com>                                             ┆ 3        ┆ 1       ┆ 0.908567       ┆ 0.056407 │
    │ Liu HaoPing (Alan) <HaoPing.liu@amd.com>                                    ┆ 1        ┆ 1       ┆ 0.9062         ┆ 0.0      │
    │ Tamminen Eero T <eero.t.tamminen@intel.com>                                 ┆ 1        ┆ 1       ┆ 0.9042         ┆ 0.0      │
    │ Linus Torvalds <torvalds-de/tnXTf+JLsfHDXvbKv3WD2FQJk+8+b@public.gmane.org> ┆ 1        ┆ 1       ┆ 0.9032         ┆ 0.0      │
    │ Zhang Tiantian (Celine) <Tiantian.Zhang@amd.com>                            ┆ 1        ┆ 1       ┆ 0.9022         ┆ 0.0      │
    └─────────────────────────────────────────────────────────────────────────────┴──────────┴─────────┴────────────────┴──────────┘

