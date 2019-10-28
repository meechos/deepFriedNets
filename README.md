# Neural network classifier for rare disease prediction
Health claims data containing information on medical diagnoses, procedures and prescriptions can be used to identify undiagnosed rare disease patients. Ensemble decision tree approaches (such as `LightGBM` and `XGBoost`) are the most common machine learning approaches for tabular data. 

To optimise the performance of these models, feature engineering and feature selection is employed. Here, we implement a **Deep Neural Network classifier** optimised for the tabular datasets used in undiagnosed patient prediction in order to compare its performance to existing ensemble decision tree approaches.

- **deepFried net:** wide and deep feed-forward approach
- **Tabby net:** deep neural network featuring decision-tree like architecture with attention mechanism
