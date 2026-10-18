# 智慧電商數據分析平台 (E-Commerce LLM & Data Project)

> 四個圍繞電商資料的專案：**LLM 特徵工程與熱銷預測**、**語意推薦**、**時間序列營收預測**、**訂單資料 ETL Pipeline（Airflow）**。

| 專案 | 問題 | 做法 | 關鍵成果 |
|---|---|---|---|
| **[一、LLM 熱銷預測](docs/project1_llm_hot_seller.md)** | 商品文案是非結構化文字，難以直接入模 | ETL 清洗商品資料並寫入 DuckDB，以 LLM 萃取文案特徵，結合 XGBoost 預測熱銷，再由 LLM 產出中文診斷報告 | Accuracy 83.1%、ROC-AUC 0.938 |
| **[二、語意推薦](docs/project2_semantic_recommendation.md)** | 新商品、新用戶沒有歷史行為（冷啟動） | Sentence Transformer 向量 + Cosine Similarity | Precision@10 96.4% |
| **[三、時間序列營收預測](docs/project3_timeseries_forecast.md)** | 營收預測支援庫存備貨，但歷史資料只有 25 個月 | 月度聚合後比較 Linear Regression、Prophet、XGBoost（Lag 特徵） | XGBoost MAPE 0.8% |
| **[四、訂單 ETL Pipeline](docs/project4_orders_etl_airflow.md)** | 訂單資料需要可重複、可驗證、可排程的處理流程 | 模組化 ETL（清洗 → 星狀模型）+ Airflow 排程（來源檢查 → 清理 → 輸出驗證） | 處理約 104 萬筆訂單，整條 Pipeline 約 15 秒 |

## 技術棧

| 面向 | 工具 |
|---|---|
| 資料處理與 ETL | Python、Pandas、SQL、DuckDB、模組化 ETL（extract / transform / load） |
| 排程與資料品質 | Apache Airflow、輸出驗證與失敗處理 |
| LLM | Groq API（Llama-3.3-70b）、Prompt 設計、結構化 JSON 輸出 |
| 機器學習 | XGBoost、scikit-learn、Prophet、Sentence Transformer |
| BI | Power BI（資料模型就緒，Dashboard 規劃中） |

## 資料工程重點（ETL）

```mermaid
flowchart LR
    A["原始 CSV"] --> B["ETL 清洗與建模"]
    B --> C["星狀模型 六張表"]
    C --> D["Airflow 排程"]
    D --> E["來源檢查"]
    E --> F["清理"]
    F --> G["輸出驗證"]
```

| 面向 | 做法 |
|---|---|
| 清洗與建模 | 模組化 ETL（`etl_powerbi/`）：剔除取消訂單與數量、單價異常的資料，**每一類被剔除的原因各自計數並記錄**，輸出事實表與維度表共六張 |
| 資料品質 | 處理前檢查來源檔與必要欄位；處理後比對筆數與營收總額（以容許誤差比較浮點數）；重複資料列先記錄、不直接刪除 |
| 失敗處理 | 來源異常時下游 Task 自動停止，不會用壞資料覆蓋輸出；可從失敗點重跑；已設定自動重試 |
| 可重複執行 | 每次覆寫輸出（冪等）；路徑由程式位置推算，不依賴工作目錄；處理邏輯與排程分離 |
| 排程 | Airflow 每日排程，實測自動執行成功 |

處理規模：約 104 萬筆訂單，整條 Pipeline 約 15 秒。詳細說明見[專案四文件](docs/project4_orders_etl_airflow.md)。

## 專案結構

```
ecommerce-llm-project/
├── README.md
├── docs/                              各專案的詳細說明
├── 01_etl.ipynb ~ 04_explain.ipynb    專案一：LLM 熱銷預測
├── 05_semantic_recommendation.ipynb   專案二：語意推薦
├── 06_timeseries_forecast.ipynb       專案三：時間序列營收預測
├── etl_powerbi/                       專案四：清洗與星狀模型匯出
├── 08_etl_pandas.ipynb                專案四：Pandas ETL 原型
├── etl_pandas.py                      專案四：可被 Airflow 呼叫的 ETL
├── dags/ecommerce_etl.py              專案四：Airflow DAG
├── data/                              資料、資料庫與圖表
└── requirements.txt
```

## 快速開始

### 環境需求
- Python 3.11+
- Kaggle 帳號（API 自動下載資料）
- Groq 帳號（呼叫 Llama-3 模型）

### 安裝
```bash
pip install -r requirements.txt
```

### 設定環境變數
```bash
cp .env.example .env
# 編輯 .env，填入 KAGGLE_USERNAME、KAGGLE_KEY、GROQ_API_KEY
```

### 各專案如何執行
- **專案一**：依序執行 `01_etl` → `02_llm_features` → `03_train` → `04_explain`，詳見[專案一文件](docs/project1_llm_hot_seller.md#五如何執行)。
- **專案二**：`05_semantic_recommendation.ipynb` 可獨立執行。
- **專案三**：`06_timeseries_forecast.ipynb` 可獨立執行。
- **專案四**：`python -m etl_powerbi.main` 產出資料；Airflow 的安裝與啟動見[專案四文件](docs/project4_orders_etl_airflow.md#如何執行)。Airflow 使用獨立的虛擬環境，不在 `requirements.txt` 內。

## 注意事項
- **API Key 安全**：`.env` 已列入 `.gitignore`，請勿將真實的 API Key 上傳至 Git 平台。
- **資料檔案**：原始資料與大型輸出檔（例如 `fact_orders.csv`、`orders_cleaned_pandas.csv`，各約 100 MB）不適合放進 Git，請依各專案文件的步驟自行產生。
- **macOS**：訓練 XGBoost 若缺少 OpenMP，可用 `brew install libomp`；程式也內建 fallback，自動改用 sklearn 的 `GradientBoostingClassifier`。
