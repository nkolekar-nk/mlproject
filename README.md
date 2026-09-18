Student Performance Predictor

An end-to-end machine learning project that predicts a student's math score from their background and their reading and writing scores — taken from exploratory notebooks all the way to a deployable Flask web app.

The point of the project is the pipeline, not the model: data ingestion, transformation, model selection, and serving are separate components with their own configs, logging, and error handling, so any stage can be rerun or replaced on its own.

What it predicts
	
Target	math_score (0–100)
Features	gender, race/ethnicity, parental level of education, lunch type, test preparation course, reading score, writing score
Dataset	1,000 student records (notebook/data/stud.csv)
Problem type	Regression
Results

Seven regressors are trained and compared; the one with the highest R² on the test set is saved to artifacts/model.pkl. On an 80/20 split (random_state=42):

Model	R²	MAE
Linear Regression	0.880	4.21
Gradient Boosting	0.872	4.30
CatBoost	0.861	4.46
XGBoost	0.857	4.58
Random Forest	0.853	4.64
AdaBoost	0.851	4.70
Decision Tree	0.742	6.36

Linear regression wins, which makes sense: reading and writing scores are close to linearly related to math score, and the boosted models mostly fit noise on top of that. Predictions land within about 4 points of the true score on average.

How it works
notebook/data/stud.csv
        │
        ▼
  Data Ingestion          reads the raw CSV, writes train.csv / test.csv (80/20 split)
        │
        ▼
  Data Transformation     numeric: median impute → standard scale
        │                 categorical: mode impute → one-hot → scale
        │                 saves the fitted preprocessor to artifacts/preprocessor.pkl
        ▼
  Model Trainer           grid-searches 7 regressors, scores each by R²,
        │                 saves the best to artifacts/model.pkl
        ▼
  Predict Pipeline        loads preprocessor + model, transforms one record, predicts
        │
        ▼
  Flask app               HTML form → prediction on the page

A model scoring below R² 0.6 raises an exception rather than being saved, so a bad training run fails loudly instead of shipping a weak model.

Project structure
├── notebook/
│   ├── 1 . EDA STUDENT PERFORMANCE.ipynb   exploratory analysis
│   ├── 2. MODEL TRAINING.ipynb             model experiments
│   └── data/stud.csv                       raw dataset
├── src/
│   ├── components/
│   │   ├── data_ingestion.py               load, split, persist
│   │   ├── data_transformation.py          preprocessing pipelines
│   │   └── model_trainer.py                train, tune, select, save
│   ├── pipeline/
│   │   ├── train_pipeline.py
│   │   └── predict_pipeline.py             inference for a single record
│   ├── exception.py                        custom exception with file and line context
│   ├── logger.py                           timestamped run logs
│   └── utils.py                            save/load objects, model evaluation
├── artifacts/                              generated: data splits, preprocessor, model
├── templates/                              index.html, home.html
├── application.py                          Flask entry point
├── .ebextensions/python.config             AWS Elastic Beanstalk WSGI config
├── requirements.txt
└── setup.py
Running it
bash
git clone https://github.com/Nisha2306/mlproject.git
cd mlproject

python -m venv venv
venv\Scripts\activate        # Windows
# source venv/bin/activate   # macOS / Linux

pip install -r requirements.txt

Train the pipeline end to end — this regenerates everything in artifacts/:

bash
python -m src.components.data_ingestion

Start the web app:

bash
python application.py

Then open http://127.0.0.1:5000/predictdata, fill in the form, and the predicted math score appears on the page.

Deployment

.ebextensions/python.config points AWS Elastic Beanstalk at the WSGI callable in application.py, so the repository can be deployed as-is to an Elastic Beanstalk Python environment.

Built with

Python · scikit-learn · CatBoost · XGBoost · pandas · NumPy · Flask · AWS Elastic Beanstalk

Notes

File paths in the components are currently written in Windows form (notebook\data\stud.csv), so training is run on Windows as written. Switching them to os.path.join would make the pipeline portable across operating systems.
