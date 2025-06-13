
# 📘 國家圖書館計畫LLM書目分類推薦系統實驗設計

## 1. 釐清研究目標與可檢驗假設
- **研究問題**：開發一套結合大型語言模型（LLM）之書目分類推薦系統，以提升國家圖書館新書編目的效率與準確度。
- **研究目標**：
  - LLM 書目分類推薦系統的分類結果優於統計式推薦系統且接近人工編目。
- **干擾項**：書籍領域分布、語料年限、語言、長度、標題資訊密度、編目人員的不同。

## 2. 資料規劃
- **來源**：國圖 Aleph MARC21 系統；資料欄位：書名、作者、出版社、分類號（089）、主題詞（699）、OWN（人工編目結果）
- **資料區間**：
  - 訓練集: 出版年份為 2007~2016（10年資料）
  - 測試集: 出版年份為 2017~2025 每年資料（逐年測試）
- **資料篩選條件**：
  - 語言為繁體中文（lang=chi）、英文 ( 以單字切分 )
  - 排除：學位論文、目錄、分篇等非一般圖書類型
- **法規與隱私**：使用不涉及個資之書目資料，符合國圖授權用途。

## 3. 實驗設計
- **A/B/C/D 實驗比較：**
  - A: 本地 LLM 模型（如 LLaMA、Mistral）+ Zero-shot Prompt template 推薦
  - B: 本地 LLM 模型（如 LLaMA、Mistral）+ Few-shot Prompt template 推薦
  - C: Fine-tuning 過的本地 LLM + RAG 強化分類推薦
  - D: BM25 + Fine-tuning 過的本地 LLM + RAG 強化分類推薦

- **指標**：
  - 分類號使用 Micro-F1
  - 主題詞使用 Macro-recall、Jaccard Coefficient
  - 輔助指標：推論速度、人工修正率、推薦可解釋性。

## 4. 資料收集與治理
- **流程**：
  - ISO2709 → MARC21 → MARCXML → PostgreSQL 清理匯入
  - 建立清理與標準化程序（斷詞、Bi-gram、欄位標準化）
- **監控與記錄**：建立清洗紀錄（缺值率、語言錯誤率、欄位不一致）

## 5. 前處理與特徵工程
- **IR 特徵**：
  - 書名（Bi-gram）
  - 作者、出版社（不斷詞，整段比對）
- **LLM 特徵建構**：
  - 建構 prompt template，格式包含書名、作者簡介、出版社、書籍摘要（若有）。
  - 建立 人工編目結果資料集供 fine-tuning 使用（含分類號、主題詞）

以下是針對 Mistral-7B 與 LLaMA-3 最佳化的分類推薦 Prompt，可直接用於 Zero-shot / Few-shot 推論：

### Prompt Template：結構化格式

```
你是一位具10年以上經驗的圖書館分類專家，熟悉《中文圖書分類法（2007）》與《中文主題詞表》。請根據以下書籍資訊，推薦最適合的分類號（5碼）與 1~3 個主題詞。

請注意：分類號需為正規格式（如 523.94），主題詞須為通用學術詞彙。若有多種可能，請選擇最能代表主題的分類與詞彙。

### 書籍資訊
- 書名：{title}
- 作者：{author}
- 出版社：{publisher}
- （可選）出版年份：{year}
- （可選）內容摘要：{summary}

### 輸出格式
```json
{
  "分類號": "XXXX.X",
  "主題詞": ["主題詞1", "主題詞2"],
  "推薦理由": "根據書名與出版社可推測本書屬於......"
}
```

## 6. 探索性資料分析（EDA）
- 分析主題詞是否出現在書名中之比例
- 分類號混淆矩陣與IoU分析 ( 比對「模型預測的分類」與「人工正確分類」之間差異的表格 )
- 出版年 vs. 類號/主題詞演變趨勢視覺化

## 7. 建模與驗證策略

| 實驗條件 | 模型 | 是否使用知識 | 輸入資料 |
|----------|------|--------------|----------|
| Zero-shot | LLaMA-7B、Mistral-7B | ✗ | Title, Author, Publisher, ISBN...等 |
| Few-shot | LLaMA-7B、Mistral-7B | △ | Title, Author, Publisher, ISBN...等 + Prompt template |
| RAG 增強 | LLaMA-7B、Mistral-7B | ✓（過去編目資料） | Title, Author, Publisher, ISBN...等 + RAG 查詢 |
| IR + LLM | BM25 + LLaMA-7B、Mistral-7B | ✓（過去編目資料） | Title, Author, Publisher, ISBN...等 + RAG 查詢 |

- 評估指標：
  - 分類號：Micro-F1
  - 主題詞：Macro-recall、Jaccard coefficient
  - 錯誤類別分析與案例比較
## 8. 評估與統計檢定
- F1、Recall、Jaccard 等指標之 paired t-test／Wilcoxon test 比較
- 每筆推薦結果之分類號、主題詞是否命中標準答案，並計算平均 confidence 分數

## 9. 可重現性與實驗管理
- 使用 Git + DVC 追蹤資料版本
- 模型記錄與參數追蹤：MLflow
- 容器化部署：Docker + Ubuntu（統計式系統）；Ollama + webUI（LLM）

## 10. 風險與倫理
- 分類偏誤檢測：統計學科偏好傾向（如歷史類集中化）
- 可解釋性：LLM 回傳分類理由供人工檢視（透過 prompt 輸出 “分類依據說明”）
- 測試 prompt injection 或 hallucination 的可能性（如分類錯誤與無關領域）

## 11. 結果詮釋與溝通
- 將結果可視化（分類分布圖、指標雷達圖）
- 針對技術、業務層分層報告
- 建議未來部署方向（如混合模型系統）

## 12. 持續迭代與部署
- 建立持續 retrain pipeline
- 測試 self-learning 模式（如 LLM 接收 feedback 自我校正）
- 探討 LLM 與 IR 混合式推薦可能性（先用 BM25 narrow，再交給 LLM 說明分類理由）[實驗D]
