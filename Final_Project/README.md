Reproducibility Instructions: 

1. Select the Kaggle dataset and the public API relevant to the project.
2. Proceed with the Batch Ingest (from Kaggle).
3. Store raw data in GCS, load curated tables into BigQuery (document schema & partitioning).
4. Provide one data quality check and one explanation of the transformation logic.
5. Proceed with Streaming Ingest (from API).
6. Create a Cloud Function (2nd gen) producer that calls a public API periodically.
7. Publish payloads to Pub/Sub as normalized JSON.
8. Build a Pipeline (Pub/Sub → BigQuery) for writing real-time data to a streaming BigQuery table.
9. Validate with queries showing near real-time rows (timestamps).
10. Proceed with Analytics and Modeling.
11. Split the analysis by cryptocurrencies (symbols).
12. Create the target variable if it does not already exist.
13. Combine both batch and streaming data into a single BigQuery table.
14. Build a baseline logistic regression model.
15. Calculate evaluation metrics for the baseline model.
16. Develop a feature-engineered logistic regression model. This model will consist of the original features from the baseline model and will have newly calculated features to improve model performance.
17. Calculate evaluation metrics for the feature-engineered model.
18. Compare the baseline and feature-engineered models to pick the best one.
19. Generate data visualizations using Plotly for both batch and streaming data to show the relevant KPIs.
20. Replicate these KPIs into the Looker Studio Dashboard as well.
21. Conduct a DIVE Analysis showing how prompts evolved, where they failed, and how you validated them.
22. Complete Individual Analysis and upload your notebook to the shared GitHub repository.
23. Complete Individual Contribution MD, consisting of the exact tasks performed and the lessons learned.
24. Contribute to team deliverables such as the Looker Studio Dashboard, Architecture README, Live Demo, and Governance Note.
25. Upload all files to the shared GitHub repository.
26. Complete and submit the Final Project to BrightSpace. 

Dataset References: 

1. Kaggle Dataset: https://www.kaggle.com/datasets/sudalairajkumar/cryptocurrencypricehistory
2. API: https://api.coinlore.net/api/tickers/ 
