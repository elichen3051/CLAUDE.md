# Benchmark 配對比較與統計推斷實作重點整理

這份整理的目的是作為後續與 AI（如 Codex）協作時的 **agent knowledge base**。
核心任務是：**在同一個 benchmark dataset 上，公平且可重現地比較 full-precision model 與 quantized model 的表現**，並且能對不同 decoding 設定下的 performance 做統計推斷與不確定性估計。

---

## 1. 問題定義

我們要比較的是：

* **full-precision model**
* **quantized model**

在**同一個 benchmark dataset**上的表現差異。

這些 benchmark 的 decoding 方式可能包含：

1. **greedy decoding**
2. **random sampling，每題只答一次**
3. **random sampling，每題答 n 次**

處理的任務類型可能包含：

1. **Multiple choice problem**
2. **exact match**
3. **free-form generative**

而程式在估計 performance 時，應支援：

* 是否使用 **bootstrap**
* **bootstrap 次數**
* 單一模型 performance 的區間估計
* 兩模型差值的區間估計

---

## 2. 最核心原則：統計單位應以 item-level 為主

無論 decoding 怎麼做，**統計推斷的基本單位都應該是 benchmark item（題目）**，而不是單一生成結果。

也就是說：

* dataset 有 (m) 題
* 每一題可能只生成 1 次，也可能生成 (n) 次
* 但最終統計分析時，應先把每一題整理成一個 **item-level score**
* 再對所有題目做平均、比較與 bootstrap

### 為什麼不能把每個 sampled response 當成獨立樣本？

因為同一題下的多次 sampling：

* 共享同一個題目內容
* 共享同一個 ground truth
* 共享同一模型狀態
* 彼此不是獨立 benchmark item

如果把每個 response 都當作獨立樣本，會：

* 嚴重低估不確定性
* 錯誤放大有效樣本數
* 導致過度自信的 CI / 檢定結果

---

## 3. 評估應優先使用 benchmark 已儲存的 item-level 分數

若使用的 benchmark framework 是：

* `lighteval`
* `lm-evaluation-harness (lm_eval)`

則實作上**優先使用它們已儲存的 record-level / item-level metric**，而不是重新從 prediction text 重算 correctness。

### 原因

1. 可避免重複實作 scoring logic
2. 可與原 benchmark framework 的結果完全對齊
3. 可避免 exact match normalization、MC parsing、judge 規則不一致
4. 更適合做 paired comparison 與 bootstrap

### 推薦做法

對每個 item record，直接讀：

* `metric.<metric_name>`
* 或 `__metric__.<metric_name>`

例如：

* `metric.acc`
* `metric.exact_match`
* `metric.avg@n:n=1`
* `metric.pass@1`

若 framework 已儲存這些 item-level 分數，則：

* **不要重算**
* 直接拿這些分數做統計推斷

---

## 4. 單一模型 performance 的定義

對一個模型，假設 dataset 有 (m) 題，每題 item-level score 為：

[
Y_1, Y_2, \dots, Y_m
]

則整體 performance 為：

[
\hat{\mu} = \frac{1}{m}\sum_{i=1}^m Y_i
]

其中 (Y_i) 可以是：

* binary：0 / 1
* continuous：任意浮點數

### binary 的例子

* multiple choice correctness
* exact match correctness
* pass@n after item-level aggregation
* majority vote correctness

### continuous 的例子

* partial credit
* judge score
* free-form metric
* benchmark 預先儲存的 continuous metric

---

## 5. full-precision vs quantized 的比較方式：一定要做 paired comparison

對同一個 item (i)，定義：

* (Y_i^{FP})：full-precision model 的 item-level score
* (Y_i^{Q})：quantized model 的 item-level score

則 item-level difference 定義為：

[
D_i = Y_i^{Q} - Y_i^{FP}
]

整體差異估計為：

[
\hat{\Delta} = \frac{1}{m}\sum_{i=1}^m D_i
]

### 解讀

* (\hat{\Delta} > 0)：quantized model 較好
* (\hat{\Delta} < 0)：quantized model 較差
* (\hat{\Delta} \approx 0)：兩者接近

### 為什麼一定要 paired？

因為兩模型是在**同一批題目**上測試。
同一題通常對兩模型共享相近的難度，使用 paired analysis 可以：

* 降低 variance
* 提高比較精度
* 避免把題目難度差異誤當成模型差異

---

## 6. 三種 decoding 模式下，item-level score 的定義方式

---

### 6.1 Greedy decoding

#### 定義

每題只生成 1 次，且 decoding deterministic。

#### item-level score

每題記錄一個分數：

[
Y_i
]

這個分數通常直接由 benchmark 的 record-level metric 提供。

#### 特性

* 最穩定
* 無 seed noise
* 最適合作為 quantization 的 primary baseline
* 最適合 paired bootstrap

#### 建議

在 quantization 研究中，greedy decoding 應保留為主要基準。

---

### 6.2 Random sampling，每題只答一次

#### 定義

每題只生成 1 次，但 decoding stochastic。

#### item-level score

對某次特定 seed 的完整 eval run，對每題仍定義：

[
Y_i^{(s)}
]

其中 (s) 是該次 run 的 random seed。

#### 特性

此時 performance 來源有兩種變異：

1. 題目抽樣變異
2. decoding randomness

#### 實作建議

支援兩種用法：

##### 模式 A：固定 seed

* 適合工程比較
* 適合 regression test
* 適合固定 protocol 下的 FP vs quantized 比較

##### 模式 B：多 seed 重複 eval

* 每個 seed 得到一個 overall score
* 再對不同 seed 的結果做摘要

但如果 item-level metric 是 benchmark 已儲存好的，且你現在比較的是**同一組固定 run**，那主要仍然是做 paired item-level analysis。

---

### 6.3 Random sampling，每題答 n 次

這種情況要先明確定義「如何從同一題的 n 次回答得到 item-level score」。

#### 常見 aggregation

---

#### A. Pass@n / Any-correct

若一題有 (n) 次回答，每次 correctness 為：

[
Z_{i1}, Z_{i2}, \dots, Z_{in} \in {0,1}
]

則 item-level score 定義為：

[
Y_i = \mathbf{1}\left(\sum_{j=1}^{n} Z_{ij} \ge 1\right)
]

表示只要 n 次中有一次答對，該題就算成功。

---

#### B. Mean-of-n accuracy

[
Y_i = \frac{1}{n}\sum_{j=1}^{n} Z_{ij}
]

表示該題在 n 次抽樣下的平均成功率。

---

#### C. Majority vote correctness

* 先對每次回答抽取 final answer
* 取出現次數最多的答案
* 與 ground truth 比對

得到：

[
Y_i \in {0,1}
]

---

#### D. Best-of-n / custom judge score

若 free-form generative 需要對 n 次回答做挑選或 judge：

[
Y_i = \text{aggregate}(score_{i1}, \dots, score_{in})
]

其中 `aggregate` 可以是：

* max
* mean
* judge-selected
* custom aggregation

---

### 實作上的關鍵原則

若 benchmark framework 已經幫你算好 item-level metric，例如：

* `metric.avg@n:n=1`
* `metric.pass@1`
* `metric.acc`

那麼應**直接使用該 item-level 分數**，不要自行重算。

---

## 7. 任務類型的處理原則

---

### 7.1 Multiple choice problem

通常 item-level score 為 binary：

[
Y_i \in {0,1}
]

若 benchmark 已存 item-level `acc`，應直接使用。

可支援的 decoding 情況：

* greedy → 單一 correctness
* sampling single → 單一 sampled correctness
* sampling n → pass@n / mean-of-n / majority vote 等已彙整後的 metric

---

### 7.2 Exact match

通常也是 binary：

[
Y_i \in {0,1}
]

若 benchmark framework 已幫你做完 normalization 與 scoring，應直接用它存的 item-level score，而不是自己重做。

---

### 7.3 Free-form generative

item-level score 可能是：

* binary correctness
* partial credit
* judge score
* rubric score
* continuous metric

因此實作時不應假設 score 一定是 0/1。
應支援：

[
Y_i \in \mathbb{R}
]

---

## 8. Bootstrap 的正確使用方式

程式應支援：

* `use_bootstrap: bool`
* `bootstrap_iters: int`
* `confidence_level: float`

### Bootstrap 的目的

1. 估計單一模型 performance 的不確定性
2. 估計兩模型差異 (\Delta) 的不確定性

---

### 8.1 單一模型 bootstrap

對 dataset 的 item indices 做有放回抽樣。

假設有 (m) 題：

1. 抽樣 (m) 個 item indices（with replacement）
2. 用這組 resampled items 重算：
   [
   \hat{\mu}^{*(b)}
   ]
3. 重複 (B) 次

最後從 bootstrap distribution 得到：

* bootstrap mean
* bootstrap std
* percentile CI（例如 95% CI）

---

### 8.2 配對差異的 bootstrap

對 paired item differences：

[
D_i = Y_i^Q - Y_i^{FP}
]

做 item-level resampling：

1. 抽樣 (m) 個 item indices（with replacement）
2. 計算：
   [
   \hat{\Delta}^{*(b)} = \frac{1}{m}\sum_{i \in \text{sample}_b} D_i
   ]
3. 重複 (B) 次

最後得到：

* bootstrap std of (\Delta)
* confidence interval of (\Delta)

---

### 8.3 Bootstrap 的正確 resampling 單位

**一定要 resample item，不是 response。**

不要做：

* 對同一題的 n 個 sampled responses 當成 iid observation resample
* 對所有生成結果攤平成大表直接 bootstrap

正確做法是：

* 對題目做 resampling
* 每題的 item-level score 由 benchmark record 或 aggregation 已定義好

---

### 8.4 Bootstrap 次數建議

可作為實作預設：

* `200`：快速測試
* `1000`：一般使用
* `5000`：需要更穩定 CI 時

---

## 9. 建議的資料對齊方式

比較 full-precision 與 quantized 時，必須先以 item-level key 對齊。

### 推薦 key

若 benchmark record 中有以下欄位，可作為 item 對齊依據：

* `doc.id`
* `doc.task_name + doc.id`
* 或明確定義的 `item_id`

### 若只有下列欄位

例如：

* `doc.task_name`
* `doc.num_samples`

則還需要能唯一識別 item 的欄位。
若框架本身無唯一 id，應在資料讀取層建立穩定 key。

---

## 10. 建議做的 validation

為避免 FP 與 quantized run 錯誤對齊，實作時應做 validation。

### 建議檢查

1. **task_name 一致**

   * FP 與 quantized 的 records 必須來自同一 task / dataset

2. **item_id 一致**

   * paired comparison 前應確認可完整對齊

3. **metric_name 存在**

   * 每個 item 都應有該 metric
   * 若缺失，應明確報錯或記錄遺失數量

4. **num_samples 合法**

   * 若有 `doc.num_samples`，可檢查它與實際 response 數量是否一致

5. **output_tokens 長度檢查**

   * 若 `model_response.output_tokens` 是 `List[List]`
   * 則可檢查：
     [
     \text{len(output_tokens)} = \text{num_samples}
     ]

---

## 11. output_tokens 的使用方式

若 benchmark record 中有：

```python
model_response.output_tokens: List[List]
```

且語意為：

* 單一 item
* 進行 n 次 sampling
* 每次 sampling 對應一串 output token ids

則可用來分析 token usage。

### 單題的 token 統計

對某個 item：

```python
per_inference_token_count = [len(x) for x in output_tokens]
```

可得到：

* 每次 inference 的 output token 數

進一步可算：

* `num_inferences`
* `per_inference_output_tokens`
* `total_output_tokens`
* `mean_output_tokens`

---

### 全 dataset 的 token 統計

可彙整得到：

* `total_output_tokens`
* `mean_output_tokens_per_item`
* `mean_output_tokens_per_inference`

---

### FP vs quantized 的 token 比較

除了 performance，也可比較：

* FP 總 output tokens
* quantized 總 output tokens
* 平均每次 inference token 長度
* score 與 token cost 的 tradeoff

例如：

* quantized model 分數略低，但 token cost 下降很多
* quantized model 分數幾乎不變，但輸出變冗長
* quantized model 在某些設定下生成顯著較短或較長

這些都值得納入分析報表。

---

## 12. 建議的實作模式：以 precomputed metric 為主

後續實作時，應優先採用以下模式。

### Primary mode：precomputed metric mode

#### 核心假設

benchmark 已為每個 item 算好分數。

#### 此模式下需要做的事只有：

1. 讀取 item-level metric
2. 用 item-level metric 算單一模型 performance
3. 對齊 FP / quantized item
4. 做 paired difference
5. 做 bootstrap
6. 統計 output_tokens

### 這個模式下不需要做的事

* 不需要重算 exact match
* 不需要重算 MC correctness
* 不需要重新從 text 解析答案
* 不需要自行重現 benchmark scoring

---

## 13. 建議的框架輸入參數

```python
metric_name: str
use_bootstrap: bool
bootstrap_iters: int
confidence_level: float
validate_task_name: bool
validate_num_samples: bool
validate_output_tokens: bool
```

若要支援讀取不同框架的輸出，可再加：

```python
field_map: dict | None
```

用來對應欄位名稱，例如：

* `doc`
* `metric`
* `model_response.output_tokens`
* `item_id`

---

## 14. 建議的框架輸出內容

---

### 單一模型結果

```json
{
  "metric_name": "avg@n:n=1",
  "point_estimate": 0.4333,
  "n_items": 60,
  "bootstrap": {
    "enabled": true,
    "iters": 1000,
    "std": 0.061,
    "ci_lower": 0.316,
    "ci_upper": 0.550
  },
  "token_usage": {
    "total_output_tokens": 12345,
    "mean_output_tokens_per_item": 205.75,
    "mean_output_tokens_per_inference": 19.12
  }
}
```

---

### 兩模型比較結果

```json
{
  "metric_name": "avg@n:n=1",
  "fp_score": 0.4833,
  "quant_score": 0.4333,
  "delta_quant_minus_fp": -0.0500,
  "n_items": 60,
  "bootstrap": {
    "enabled": true,
    "iters": 1000,
    "std_delta": 0.028,
    "ci_lower": -0.106,
    "ci_upper": 0.004
  },
  "token_usage": {
    "fp_total_output_tokens": 13200,
    "quant_total_output_tokens": 12150,
    "delta_total_output_tokens": -1050,
    "fp_mean_output_tokens_per_inference": 20.4,
    "quant_mean_output_tokens_per_inference": 18.8
  }
}
```

---

## 15. 對 paired statistical inference 的實作結論

以下原則可視為之後實作 paired test / paired bootstrap 的具體依據：

### 原則 1

**比較 full-precision 與 quantized model 時，一律以 item-level paired comparison 為主。**

---

### 原則 2

**若 benchmark 已儲存 item-level metric，應直接使用，不重算。**

---

### 原則 3

**統計推斷的單位是 item，不是 response。**

---

### 原則 4

**sampling n 次時，必須先定義 item-level aggregation，再做整體統計。**

---

### 原則 5

**bootstrap 應在 item-level 進行 resampling。**

---

### 原則 6

**greedy decoding 是最穩定的 baseline，特別適合作為 quantization 研究中的 primary evaluation。**

---

### 原則 7

**random sampling 的 performance 可能同時受到題目數有限與 decoding randomness 影響，因此應避免把單次隨機 run 的結果過度解讀。**

---

### 原則 8

**token usage 應獨立統計，並可與 performance 一起做 tradeoff 分析。**

---

## 16. 可直接給 Codex / agent 的精簡需求摘要

以下內容可直接作為工程需求：

### 需求摘要

請實作一個 benchmark evaluation framework，用來比較 full-precision model 與 quantized model 在同一個 dataset 上的表現。

#### 必要要求

1. 讀取 benchmark framework（如 lighteval / lm_eval）已儲存的 item-level metric
2. 不重算 correctness
3. 以 item 為統計單位
4. 支援 greedy、sampling single、sampling n 的評估結果比較
5. 支援 multiple choice、exact match、free-form generative，只要 benchmark 已存 item-level score 即可
6. 比較 FP 與 quantized 時，使用 paired item comparison
7. 計算：

   * 單一模型 performance
   * quantized - full-precision 的 performance delta
8. 支援可選 bootstrap：

   * `use_bootstrap`
   * `bootstrap_iters`
9. bootstrap 應以 item-level resampling 為主
10. 若有 `model_response.output_tokens: List[List]`，需統計：

* 每題每次 inference 的 token 數
* dataset 總 output tokens
* 每次 inference 平均 token 數
* FP vs quantized token usage 差異

11. 支援 validation：

* task_name
* item_id 對齊
* num_samples
* output_tokens 長度

---
